==========================================================
General Logging, Monitoring, Alerting architecture for MCP
==========================================================

[No Jira Epic for this spec]

This specification describes the general Logging, Monitoring, Alerting
architecture for MCP (Mirantis Cloud Platform).

Problem description
===================

Logging, Monitoring and Alerting are key aspects which need to be taken into
account from the very beginning of the MCP project.

This specification just describes the general architecture for Logging,
Monitoring and Alerting. Details on the different parts will be provided with
more specific specifications.

In the rest of the document we will use LMA to refer to Logging, Monitoring and
Alerting.

Use Cases
---------

The final goal is to provide tools to help OpenStack Operator diagnose and
troubleshoot problems.

Proposed change
===============

We propose to add LMA components to MCP. The proposed software and architecture
are based on the current Fuel StackLight product (composed of four Fuel
plugins), with adjustements and improvements to meet the requirement of MCP
(Mirantis Cloud Platform).

General Architecture
--------------------

The following diagram describes the general architecture::

        OpenStack nodes
    +-------------------+
    | +-------------------+
    | |            +----+ |
    | | Logs+-+  +-+Snap| |             +-------------+
    | |       |  | +----+ |             |             |
    | |      +v--v+       |      +------>Elasticsearch|
    | |      |Heka+--------------+      |             |
    +-+      +----+       |      |      +-------------+
      +-------------------+      |
                                 |      +-------------+
                                 |      |             |
                                 +------+InfluxDB     |
        k8s master node          |      |             |
    +-------------------+        |      +-------------+
    | +-------------------+      |
    | |            +----+ |      |      +-------------+
    | | Logs+-+  +-+Snap| |   +--+      |             |
    | |       |  | +----+ |   |  +------>Nagios       |
    | |      +v--v+       |   |         |             |
    | |      |Heka+-----------+         +-------------+
    +-+      +----+       |
      +-------------------+


The boxes on the top-left corner of the diagram represent the nodes where the
OpenStack services run. The boxes on the bottom-left corner of the diagram
represent the the nodes where the Kubernetes infrastructure services run. The
boxes on the right of the diagram represent the nodes where the LMA backends
are run.

Each node runs two services: Heka and Snap. Although it is not depicted in the
diagram Heka and Snap also run on the backend nodes, where we also want to
collect logs and telemetry data.

`Snap`_ is the telemetry framework created by Intel that we will use in MCP for
collecting telemetry data (CPU usage, etc.). The current StackLight product
uses Collectd instead of Snap, so this is an area where StackLight and MCP will
differ. The telemetry data collected by Snap will be sent to Heka.

`Heka`_ is a stream processing software created and maintained by Mozilla. We
will use Heka for collecting logs and notifications, deriving new metrics from
the telemetry data received from Snap, and sending the results to
Elasticsearch, InfluxDB and Nagios.

`Elasticsearch`_ will be used for indexing logs and notifications. And
`Kibana`_ will be used for visualizing the data indexed in Elasticsearch.
Default Kibana dashboards will be shipped in MCP.

`InfluxDB`_ is a database optimized for time-series.  It will be used for
storing the telemetry data. And `Grafana`_ will be used for visualizing the
telemetry data stored in InfluxDB. Default Grafana dashboards will be shipped
in MCP.

`Nagios`_ is a feature-full monitoring software. In MCP we may use it for
handling status messages sent by Heka and reporting on the current status of
nodes and services. For that Nagios's `Passive Checks`_ would be used. We've
been looking at alternatives such as `Sensu`_ and `Icinga`_ , but until now we
haven't found something with the level of functionality of Nagios. Another
alternative is to just rely on Heka's `Alert module`_ and `SMTPOutput plugin`_
for notifications. Whether Nagios will be used or not in MCP will be discussed
with a more specific specification. It is also to be noted that Alerting should
be an optional part of the monitoring sytem in MCP.

.. _Snap: https://github.com/intelsdi-x/snap
.. _Heka: http://hekad.readthedocs.org/
.. _Elasticsearch: https://www.elastic.co/products/elasticsearch
.. _Kibana: https://www.elastic.co/products/kibana
.. _InfluxDB: https://influxdata.com/
.. _Grafana: http://grafana.org/
.. _Nagios: https://www.nagios.org/
.. _Passive Checks: https://assets.nagios.com/downloads/nagioscore/docs/nagioscore/3/en/passivechecks.html
.. _Sensu: https://sensuapp.org/
.. _Icinga: https://www.icinga.org/
.. _Alert module: http://hekad.readthedocs.io/en/latest/sandbox/index.html#alert-module
.. _SMTPOutput plugin: http://hekad.readthedocs.io/en/latest/config/outputs/smtp.html

Kubernetes Logging and Monitoring
---------------------------------

Kubernetes comes with its own monitoring and logging stack, so the question of
what we will use and not use of this stack should be raised. This section
discusses that.

Monitoring
~~~~~~~~~~

`Kubernetes Monitoring`_ uses `Heapster`_. Heapster runs as a pod on a node of
the Kubernetes cluster. Heapster gets container statistics by querying the
cluster nodes' Kubelets. The Kubelet itself fetches the data from cAdvisor.
Heapster groups the information by pod and sends the data to a backend for
storage and visualization (InfluxDB is supported).

Collecting container and pod statistics is necessary for MCP, but it's not
sufficient. For example, we also want to collect OpenStack services, to be able
to report on the health of the OpenStack services that run on the cluster.
Also, Heapster does not currently support any form of alerting.

The proposal is to use Snap en each node (see the previous section). Snap
already includes `plugins for OpenStack`_. For container statistics the `Docker
plugin`_ may be used, and, if necessary, a Kubernetes/Kubelet-specific Snap
plugin may be developed.

Relying on Snap on each node, instead of a centralized Heapster instance, will
also result in a more scalable solution.

However, it is to be noted that `Kubernetes Autoscaling`_ currently requires
Heapster. This means that Heapster must be used if the Autoscaling
functionality is required for MCP. But in that case, no storage backend should
be set in the Heapster configuration, as Heapster will just be used for the
Autoscaling functionality.

.. _Kubernetes Monitoring: http://kubernetes.io/docs/user-guide/monitoring/
.. _Heapster: https://github.com/kubernetes/heapster
.. _plugins for OpenStack: https://github.com/intelsdi-x?utf8=%E2%9C%93&query=snap-plugin-collector
.. _Docker plugin: https://github.com/intelsdi-x/snap-plugin-collector-docker
.. _Kubernetes Autoscaling: http://kubernetes.io/docs/user-guide/horizontal-pod-autoscaling/

Logging
~~~~~~~

`Kubernetes Logging`_ relies on Fluentd, with a Fluentd agent running on each
node. That agent collects container logs (through the Docker Engine running on
the node) and sends them to Google Cloud Logging or Elasticsearch (the backend
used is pecified through the ``KUBE_LOGGING_DESTINATION`` variable).

The main problem with this solution is our inability to act on the logs before
they're stored into Elasticsearch. For instance we want to be able to monitor
tho logs, to be able to detect spikes of errors. We also want to be able to
derive metrics from logs, such as HTTP response time metrics. Also, we may want
to use Kafka in the future (see below). In summary, Kubernetes Logging does not
provide us with the flexibility we need.

Our proposal is to use Heka instead of Fluentd. The benefits are:

* Flexibility (e.g. use Kafka between Heka and Elasticsearch in the future).
* Be able to collect logs from services that can't log to stdout.
* Team's experience on using Heka and running it in production.
* Re-use all the Heka plugins we've developed (parsers for OpenStack logs, log
  monitoring filters, etc.).

.. _Kubernetes Logging: http://kubernetes.io/docs/getting-started-guides/logging/

Use Kafka
---------

Another component that we're considering introducing is `Apache Kafka`_. Kafka
will sit between Heka and the backends, and it will be used as a robust and
scalable messaging system for the communications between the Heka instances and
the backends. Heka has the capability of buffering messaging, but we believe
that Kafka would allow for a more robust and resilient system. We may make
Kafka optional, but highly recommended for medium and large clusters.

The following diagram depicts the architecture when Kafka is used:

        OpenStack nodes
    +-------------------+
    | +-------------------+       Kafka cluster
    | |            +----+ |         +-------+             +-------------+
    | | Logs+-+  +-+Snap| |         |       |             |             |
    | |       |  | +----+ |      +--+Kafka  +--+    +----->Elasticsearch|
    | |      +v--v+       |      |  |       |  |    |     |             |
    | |      |Heka+-------------->  +-------+  +----+     +-------------+
    +-+      +----+       |      |             |
      +-------------------+      |  +-------+  |          +-------------+
                                 |  |       |  |          |             |
                                 +--+Kafka  +--+      +--->InfluxDB     |
                                 |  |       |  |      |   |             |
        k8s master nodes         |  +-------+  +------+   +-------------+
    +-------------------+        |             |
    | +-------------------+   +-->  +-------+  +----+     +-------------+
    | |            +----+ |   |  |  |       |  |    |     |             |
    | | Logs+-+  +-+Snap| |   |  +--+Kafka  +--+    +----->Nagios       |
    | |       |  | +----+ |   |     |       |             |             |
    | |      +v--v+       |   |     +-------+             +-------------+
    | |      |Heka+-----------+
    +-+      +----+       |
      +-------------------+

The Heka instances running on the OpenStack and Kubernetes nodes are Kafka
producers. Although not depicted on the diagram Heka instances will also
probably be used as Kafka consumers between the Kafka cluster and the backends.
We will need to run performance tests to determine if Heka will be able to keep
up with the load when used as a Kafka consumer.

A specific specification will be written for the introduction of Kafka.

.. _Apache Kafka: https://kafka.apache.org/

Packaging and deployment
------------------------

All the services participating to the LMA architecture will run in Docker
containers, following the MCP approach to packaging and service execution.

Relying on `Kubernetes Daemon Sets`_ for deploying Heka and Snap on all the
cluster nodes sounds like a good approach. The Kubernetes doc even mentions
logstash and collectd as a good candidates for running as Daemon Sets.

.. _Kubernetes Daemon Sets: http://kubernetes.io/docs/admin/daemons/

Alternatives
------------

The possible alternatives will be discussed in more specific specifications.

Implementation
==============

The implementation will be described in more specific specifications.

Assignee(s)
-----------

Primary assignee:
  elemoine (elemoine@mirantis.com)

Other contributors:
  obourdon (obourdon@mirantis.com)

Work Items
----------

Other specification documents will be written:

* Logging with Heka
* Logs storage and analytics with Elasticsearch and Kibana
* Monitoring with Snap
* Metrics storage and analytics with InfluxDB and Grafana
* Alerting in MCP
* Introducing Kafka to the MCP Monitoring stack

Dependencies
============

None.

Testing
=======

The testing strategy will be described in more specific specifications.

Documentation Impact
====================

The MCP monitoring system will be documented.

References
==========

None.

History
=======

None.
