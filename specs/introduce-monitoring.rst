=====================
Add Monitoring to MCP
=====================

This specification describes the initial work that will be done for adding
monitoring capabilities to MCP. The goal is to lay out the architecture for
collecting, storing and visualizing basic system metrics (CPU, memory, etc.).

Problem description
===================

Monitoring is a core aspect of MCP. The goal of the work discussed in this
specification is to introduce first elements of Monitoring in MCP.

Use Cases
---------

This specification covers the following:

* Collecting data with Snap
* Processing the data with Hindsight
* Storing the data into InfluxDB
* Visualizing the data with Grafana

This specification focuses on Monitoring, aspects related to Alarming are out
of scope.

Proposed change
===============

Collecting data with Snap
-------------------------

We will use `Snap`_ for collecting monitoring data. Snap will run on every
cluster node. At this stage only system statistics will be collected, i.e. CPU,
memory, disk, network, etc. The list of collected metrics will depend on what
is available in Snap.

The data collected by Snap will then published to Hindsight (described in the
next section). For that we will use Snap's `Heka publisher plugin`_.

.. _Snap: http://intelsdi-x.github.io/snap/
.. _Heka publisher plugin: https://github.com/intelsdi-x/snap-plugin-publisher-heka

Snap in Docker
--------------

Snap will run in a Docker container, so a ``Dockerfile`` will be written for
Snap in MCP.

Some Snap plugins aren't currently compatible with running into Docker
containers. We've created GitHub Issues for these incompatibilities:

* https://github.com/intelsdi-x/snap-plugin-collector-cpu/issues/14
* https://github.com/intelsdi-x/snap-plugin-collector-processes/issues/11
* https://github.com/intelsdi-x/snap-plugin-collector-meminfo/issues/11
* https://github.com/intelsdi-x/snap-plugin-collector-disk/issues/6
* https://github.com/intelsdi-x/snap-plugin-collector-df/issues/8
* https://github.com/intelsdi-x/snap-plugin-collector-load/issues/8
* https://github.com/intelsdi-x/snap-plugin-collector-iostat/issues/9
* https://github.com/intelsdi-x/snap-plugin-collector-swap/issues/7

We will address these imcompatibilities with Pull Requests to Snap. Plugins
like the `df` and `iostat` use Unix commands internally (the `iostat` plugin
uses the `iostat` command for example). And in most cases these Unix commands
cannot be parameterized to read from another directory than `/proc`. This means
that these plugins will need to be eventually rewritten.

Snap is in active development. We ourselves need to change Snap and create Pull
Requests. So it is important that we are able to build Snap from sources, as
opposed to depending on binary releases created by the Snap team.

Docker Hub includes `official images for Go (golang)`_, which are based on
`buildpack-deps`_ images. We're not going to rely of these images. Instead we
will rely on MCP's ``ms-debian-base`` image, and we will ourselves install the
build tools we need for building Snap. In this way we can remove the build
tools, and thereby minimize the weight of Docker images. The final Snap image
will just include the Snap binaries required for running Snap. The Go compiler
will for example not be present in the final image.

.. _official images for Go (golang): https://hub.docker.com/r/library/golang/
.. _buildpack-deps: https://hub.docker.com/_/buildpack-deps/

Processing the data with Hindsight
----------------------------------

The data collected with Snap will be published to `Hindsight`_. Like Snap,
Hindsight will run on each cluster node, and Snap will publish the collected
data to the Hindsight instance running on the same node (in the same ``Pod``).

Hindsight is a rewrite of Heka in C. It was created to address some issues
found in Heka (mainly performance issues). Hindsight is compatible with Heka,
in the sense that Lua plugins that work in Heka also work in Hindsight. And
Hindsight supports Heka's Protobuf format.

For the communication between Snap and Hindsight we will use TCP and Heka's
Protobuf format, which Hindsight supports. On the Snap side the Heka publisher
plugin will be used. On the Hindsight side we will use the `Heka TCP Input
Plugin`_. The plugin will listen on port ``5565``, and decode the Protobuf
messages sent by Snap. The resulting messages injected into the Hindsight
pipeline will then be grouped and sent to InfluxDB by batch. For that an
Hindsight Output Plugin will be developed. That plugin will reuse some `Lua
code`_ from StackLight.

It is to be noted that Hindsight just acts as a "passthru" here. In other
words, for the basic needs described in this specification, we could do without
Hindsight and have Snap directly publish the metrics to InfluxDB (through
Snap's InfluxDB publisher plugin). But in the future we will use Hindsight for
evaluating alarms, deriving metrics from logs, etc. So it is important to use
Hindsight from the beginning.

.. _Hindsight: https://github.com/trink/hindsight/
.. _Heka TCP Input Plugin: https://github.com/mozilla-services/lua_sandbox/blob/master/sandboxes/heka/input/heka_tcp.lua
.. _Lua code: https://github.com/openstack/fuel-plugin-lma-collector/blob/master/deployment_scripts/puppet/modules/lma_collector/files/plugins/filters/influxdb_accumulator.lua

Snap and Hindsight deployment
-----------------------------

Snap and Hindsight will be deployed by Kubernetes on every Kubernetes minion.
We will rely on a ``DaemonSet`` for that. In this way, Kubernetes will start
Snap and Hindsight on any new node added to the cluster.

Snap and Hindsight will run in separate Containers part of the same ``Pod``.
Only one ``DaemonSet`` will be required.

At this stage we won't use `Snap Tribes`_. We indeed don't need that level of
sophistication for now, because all the Snap instances will be identical: they
will run the same plugins and tasks.

The Snap configuration will be stored in ``ConfigMap``, and we will rely on
Snap's plugin/task `auto_discovery_path`_ functionality for Snap to load the
plugins and create the tasks at start-up time. Currently the
``auto_discovery_path`` functionality only works for the loading of plugins. We
will extend it to also work for the creation of tasks.

Likewise, the Hindsight configuration will also be stored in ``ConfigMap``.

.. _Snap Tribes: https://github.com/intelsdi-x/snap/blob/master/docs/TRIBE.md
.. _auto_discovery_path: https://github.com/intelsdi-x/snap/blob/master/docs/SNAPD_CONFIGURATION.md#snapd-control-configurations

Git repositories
----------------

As discussed previously Snap and Hindsight will run in the same ``Pod``, and
they will be deployed by the same ``DaemonSet``. This means that the
``DaemonSet`` spec file will be common to Snap and Hindsight. For that reason
we will use just one Git repository: ``ms-lma`` or ``ms-stacklight``.

Storing the data into InfluxDB
------------------------------

As described in the previous section Hindsight will send the data to InfluxDB
for storing.

.. note:: We will investigate using Cassandra in the future. But we will start
  with InfluxDB, because we know how to run and operate InfluxDB.

InfluxDB deployment
-------------------

InfluxDB will be deployed by Kubernetes. At this point we will not run InfluxDB
in cluster mode. We will use a ``ReplicaSet`` (in a ``Deployment``) with one
replica.

Storage is another issue. To simplify the problem we will dedicate a node to
InfluxDB (using a node label). InfluxDB will run on that node and it will not
run on any other node. At this point we will use an ``emptyDir`` or
``hostPath`` Kubernetes volume on a local disk for the data storage. In the
future we may use LVM, depending on the outcome of our `LVM driver for
Volumes`_ work.

For Hindsight and Grafana to be able to access InfluxDB a Kubernetes Service
will be created for InfluxDB. The ``ClusterIP`` service type will be used.

.. note:: Using Ceph RDB for the data storage is not an option. However we know
  from experience that a local SSD is required to get decent performances.

.. note:: In the future we will need to support the case of a remote InfluxDB
  backend deployed outside the Kubernetes cluster. This means that it will be
  possible to configure Hindsight to use a different InfluxDB endpoint.

.. note:: For deploying InfluxDB/Grafana with Kubernetes we can get inspiration
  from Heapster. See https://github.com/kubernetes/heapster/tree/master/deploy/kube-config/influxdb.

.. _LVM driver for Volumes: https://mirantis.jira.com/browse/MCP-692

Grafana
-------

Grafana will run in a Docker container. And Grafana will be deployed by
Kubernetes, through a dedicated ``Pod`` and a dedicated ``ReplicaSet``.

The number of replicas will be set to one, ensuring that there will be at most
one running Grafana instance at a given time. In the future we will be able to
scale Grafana by using more replicas, but we don't need that level of
sophistication for the moment.

Grafana needs a database to store users and dashboards. By default Grafana uses
SQLite. To simplify the deployment we will use SQLite and an ``emptyDir``
Kubernetes volume. This means that any custom settings will be lost if Grafana
is restarted on another node. In the future we could rely on an RDB or LVM
volume to avoid that problem. We may also consider using another DBMS than
SQLite.

Grafana will be pre-configured with a default data source (connected to the
InfluxDB instance) and default dashboards. Adding data sources and dashboards
to Grafana is done using the Grafana HTTP API. The configuration operations
will be done in the ``start.sh`` script that starts the Grafana service. See
https://github.com/grafana/grafana-docker/blob/master/run.sh for an example.

The operator will need to access the Grafana UI. For that we will create
a Kubernetes Service for Grafana. We will use a `NodePort Service`_ for the
moment, but we will probably need to rely on a Load Balancer in the future.
This depends on what will be used in MCP.

.. _NodePort Service_: http://kubernetes.io/docs/user-guide/services/#type-nodeport

Users and Passwords
-------------------

Both InfluxDB and Grafana require creating users and passwords. We will use
`Kubernetes Secrets`_ for that.

.. _Kubernetes Secrets: http://kubernetes.io/docs/user-guide/secrets/

Alternatives
------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Éric Lemoine

Other contributors:
  Olivier Bourdon

Work Items
----------

* Create a ``Dockerfile`` for Snap
* Create a ``Dockerfile`` for Heka
* Create a ``Dockerfile`` for InfluxDB
* Create a ``Dockerfile`` for Grafana
* Create ``DaemonSet`` and ``Pod`` definitions for Snap/Heka
* Create ``ReplicaSet`` for InfluxDB/Grafana
* Create ``ConfigMap`` for Snap and Heka configurations
* Extend Snap to support the auto-discovery of tasks
* Make Snap plugins compatible with Docker

Dependencies
============

* Working developement environment
* Working CI

Testing
=======

We will develop functional tests to verify that our Snap/Heka pipeline
works as expected. This can be done with a mock Snap collector plugin, and
checking that the output of the pipeline is as expected.

Documentation Impact
====================

We will document how to set up Monitoring in MCP.

References
==========

None.
