=================
Logging with Heka
=================

[No Jira Epic for this spec]

This specification describes the logging system to implement in Mirantis Cloud
Platform (MCP). It is complementary to the *General Logging, Monitoring,
Alerting architecture for MCP* specification.

That system is based on `Heka`_, `Elasticsearch`_ and `Kibana`_.

.. _Heka: http://hekad.readthedocs.org/
.. _Elasticsearch: https://www.elastic.co/products/elasticsearch
.. _Kibana: https://www.elastic.co/products/kibana

Problem description
===================

It is important that MCP comes with a robust and scalable logging system. This
specification describes the logging system we want to implement in MCP. It is
based on StackLight's logging solution for MOS.

Use Cases
---------

The target for the logging system is the Operator of Kubernetes cluster. The
Operator will use Kibana to view and search logs, with dashboards providing
statistics views of the logs.

We also want to be able to derive metrics from logs and monitor logs to detect
spike of errors for example. The solution described in this specification makes
this possible, but the details of log monitoring will be covered with
a separate specification.

The deployment and configuration of Elasticsearch and Kibana through Kubernetes
will also be described with a separate specification.

Proposed change
===============

Architecture
------------

This is the architecture::

      Cluster nodes
    +---------------+                        +----------------+
    | +---------------+                      | +----------------+
    | | +---------------+                    | | +----------------+
    | | |               |                    | | |                |
    | | | Logs+-+       |                    | | |                |
    | | |       |       |                    | | |                |
    | | |       |       |                    | | | Elasticsearch  |
    | | |    +--v-+     |                    | | |                |
    | | |    |Heka+--------------------------> | |                |
    +-+ |    +----+     |                    +-+ |                |
      +-+               |                      +-+                |
        +---------------+                        +----------------+

In this architecture Heka runs on every node of the Kubernetes cluster. It runs
in a dedicated container, referred to as the *Heka container* in the rest of
this document.

Each Heka instance reads and processes the logs local to the node it runs on,
and sends these logs to Elasticsearch for indexing. Elasticsearch may be
distributed on multiple nodes for resiliency and scalability, but this topic is
outside the scope of that specification.

Heka, written in Go, is fast and has a small footprint, making it possible to
run it on every node of the cluster and effectively distribute the log
processing load.

Another important aspect is flow control and avoiding the loss of log messages
in case of overload. Heka’s filter and output plugins, and the Elasticsearch
output plugin in particular, support the use of a disk based message queue.
This message queue allows plugins to reprocess messages from the queue when
downstream servers (Elasticsearch) are down or cannot keep up with the data
flow.

Rely on Docker Logging
----------------------

Based on `discussions`_ with the Mirantis architects and experience gained with
the Kolla project the plan is to rely on `Docker Logging`_ and Heka's
`DockerLogInput plugin`_.

Since the `Kolla logging specification`_ was written the support for Docker
Logging has improved in Heka. More specifically Heka is now able to collect
logs that were created while Heka wasn't running.

Things to note:

* When ``DockerLogInput`` is used there is no way to differentiate log messages
  for containers producing multiple log streams – containers running multiple
  processes/agents for example. So some other technique will have to be used
  for containers producing multiple log streams. One technique involves using
  log files and Docker volumes, which is the technique currently used in Kolla.
  Another technique involves having services use Syslog and have Heka act as
  a Syslog server for these services.

* We will also probably encounter services that cannot be configured to log to
  ``stdout``. So again, we will have to resort to using some other technique
  for these services. Log files or Syslog can be used, as described previously.

* Past experiments have shown that the OpenStack logs written to ``stdout`` are
  visible to neither Heka nor ``docker logs``.  This problem does not exist
  when ``stderr`` is used rather than ``stdout``.  The cause of this problem is
  currently unknown.

* ``DockerLogInput`` relies on Docker's `Get container logs endpoint`_, which
  works only for containers with the ``json-file`` or ``journald`` logging
  drivers. This means the Docker daemon cannot be configured with another
  logging driver than ``json-file`` or ``journald``.

* If the ``json-file`` logging driver is used then the ``max-size`` and
  ``max-file`` options should be set, for containers logs to be rolled over as
  appropriate. These options are not set by default in Ubuntu (in neither 14.04
  nor 16.04).

.. _discussions: https://docs.google.com/document/d/15QYIX_cggbDH2wAJ6-7xUfmyZ3Izy_MOasVACutwqkE
.. _Docker Logging: https://docs.docker.com/engine/admin/logging/overview/
.. _DockerLogInput plugin: http://hekad.readthedocs.org/en/v0.10.0/config/inputs/docker_log.html
.. _Kolla logging specification: https://github.com/openstack/kolla/blob/master/specs/logging-with-heka.rst
.. _Get container logs endpoint: https://docs.docker.com/engine/reference/api/docker_remote_api_v1.20/#get-container-logs

Read Python Tracebacks
----------------------

In case of exceptions the OpenStack services log Python Tracebacks as multiple
log messages. If no special care is taken then the Python Tracebacks will be
indexed as separate documents in Elasticsearch, and displayed as distinct log
entries in Kibana, making them hard to read.  To address that issue we will use
a custom Heka decoder, which will be responsible for coalescing the log lines
making up a Python Traceback into one message.

Collect system logs
-------------------

In addition to container logs it is important to collect system logs as well.
For that we propose to mount the host's ``/var/log`` directory into the Heka
container (as ``/var/log-host/``), and configure Heka to get logs from standard
log files located in that directory (e.g. ``kern.log``, ``auth.log``,
``messages``).

Create an ``heka`` user
-----------------------

For security reasons an ``heka`` user will be created in the Heka container and
the ``hekad`` daemon will run under that user.

Deployment
----------

Following the MCP approach to packaging and service execution the Heka daemon
will run in a container. We plan to rely on Kubernetes's `Daemon Sets`_
functionality for deploying Heka on all the Kubernetes nodes.

We also want Heka to be deployed on the Kubernetes master node. For that the
Kubernetes master node should also be a minion server, where Kubernetes may
deploy containers.

.. _Daemon Sets: http://kubernetes.io/docs/admin/daemons/

Security impact
---------------

The security impact is minor, as Heka will not expose any network port to the
outside. Also, Heka's "dynamic sandboxes" functionality will be disabled,
eliminating the risk of injecting malicious code into the Heka pipeline.

Performance Impact
------------------

The ``hekad`` daemon will run in a container on each cluster node. And we have
assessed that Heka is lightweight enough to run on every node. See the
`Introduction of Heka in Kolla`_ email sent to the openstack-dev mailing list
for a discussion on comparison between Heka and Logstash. Also, a possible
option would be to constrain the resources associated to the Heka container.

.. _Introduction of Heka in Kolla: http://lists.openstack.org/pipermail/openstack-dev/2016-January/083751.html

Alternatives
------------

An alternative to this proposal involves relying on Kubernetes Logging, i.e.
use Kubernetes's native logging system. Some `research`_ has been done on
Kubernetes Logging. The conclusion to this research is that Kubernetes Logging
is not flexible enough, making it impossible to implement features such as
log monitoring in the future.

.. _research: https://mirantis.jira.com/wiki/display/NG/k8s+LMA+approaches

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Éric Lemoine (elemoine)

Work Items
----------

1. Create an Heka Docker image
2. Create some general Heka configuration
3. Deploy Heka through Kubernetes
4. Collect OpenStack logs
5. Collect other services' logs (RabbitMQ, MySQL...)
6. Collect Kubernetes logs
7. Send logs to Elasticsearch

Testing
=======

We will add functional tests that verify that the Heka chain works for all the
service and system logs Heka collects. These tests will be executed as part of
the gating process.

Documentation Impact
====================

None.

References
==========

None.
