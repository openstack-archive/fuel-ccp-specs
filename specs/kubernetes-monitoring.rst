=====================
Kubernetes Monitoring
=====================

Problem statement
=================

We need to collect statistical data for Containers deployed by Kubernetes. And
we also need to collect data for higher-level objects such as Pods and
Namespaces. This is necessary for creating Kubernetes-related graphs in Grafana
and alarms.

Proposed change
===============

We propose to rely on Kubelet as the telemetry source, itself relying on
cAdvisor. But instead of using Heapster as a central collector agent, we
propose an architecture where metric collectors are distributed across the
cluster.

More specifically we propose to use Hindsight as the metric collector on each
cluster node, and have each Hindsight instance get container statistics by
polling the local kubelet "stats" API.

Each Hindsight instance will do aggregations at the host level. In the future
we will want to do aggregations at the cluster level, but this is beyond the
scope of this specification and our first implementation.

This is a high-level architecture diagram::

        k8s node #1
    +----------------+
    |                |
    |    +---------+ |
    |    |Hindsight+-------------+
    |    +---^-----+ |           |
    |        |       |           |
    |        |poll   |           |
    |        |       |           |
    | +------++      |           |
    | |kubelet|      |           |     +-----------------+
    | |API    |      |           |     |                 |
    | +-------+      |           |     |                 |
    |                |           |     |                 |
    +----------------+           |     |                 |
                                 |     |    +--------+   |
                                 +---------->InfluxDB|   |
                                 |     |    +--------+   |
        k8s node #2              |     |                 |
    +----------------+           |     |                 |
    |                |           |     |                 |
    |    +---------+ |           |     |                 |
    |    |Hindsight+-------------+     |                 |
    |    +---^-----+ |                 +-----------------+
    |        |       |
    |        |poll   |
    |        |       |
    | +------++      |
    | |kubelet|      |
    | |API    |      |
    | +-------+      |
    |                |
    +----------------+

The kubelet "stats" are available at the URL
http://<node-IP>:10255/stats/summary. (See `example
<https://gist.github.com/elemoine/f3f60118594550bc38f49ecae693c9af>`_).

So to be able to use the "stats" API Hindsight needs to know the IP of the node
it runs on. One problem is that Kubernetes doesn’t currently allow Pod
containers to know the IP of the node they run on. This limitation is addressed
with Pull Requests [1] [2] that are still open at time of this writing. We will
work with the Mirantis Kubernetes team to ensure the inclusion of required
patches.

Accesses to the kubelet "stats" API are not controlled in an way, so no
authentication mechanism is needed for Hindsight to be able to poll that API.

* [1] https://github.com/kubernetes/kubernetes/pull/27880
* [2] https://github.com/kubernetes/kubernetes/pull/25957

Metrics
~~~~~~~

Three types of metrics will be collected:

*  "count" metrics – counters, e.g. number of pods
*  "resource usage" metrics
*  "check" metrics

Note: "hostname" is not listed as a dimension in the following tables.
"hostname" is implicit. This is because "hostname" is not a strictly
a dimension, but a standard field of the StackLight data model.

"count" metrics
+++++++++++++++

+---------------------------+----------------------------------------------------------------------------------------------------------------------------------------------+------------------------------------------------------------------------+
| Metric name               | Description                                                                                                                                  | Dimensions                                                             |
+---------------------------+----------------------------------------------------------------------------------------------------------------------------------------------+------------------------------------------------------------------------+
| k8s\_containers\_count    | The number of running containers on the Kubernetes node. This includes both Kubernetes containers (running in pods) and system containers.   |                                                                        |
+---------------------------+----------------------------------------------------------------------------------------------------------------------------------------------+------------------------------------------------------------------------+
| k8s\_pods\_count          | The number of pods on the Kubernetes node, in a given namespace.                                                                             | * "namespace" (string) The namespace. Should be a non-empty string.    |
+---------------------------+----------------------------------------------------------------------------------------------------------------------------------------------+------------------------------------------------------------------------+
| k8s\_pods\_count\_total   | The number of pods on the Kubernetes node.                                                                                                   |                                                                        |
+---------------------------+----------------------------------------------------------------------------------------------------------------------------------------------+------------------------------------------------------------------------+

For example, the k8s\_pod\_count metric will be used in a Grafana graph to
display the number of pods for an hostname and a namespace selected by the
Grafana user.

"resource usage" metrics
++++++++++++++++++++++++

+----------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------+
| Metric name                            | Description                                                                                                                                                                                                 | Dimensions                                                                                                          |
+----------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------+
| k8s\_pod\_cpu\_usage                   | The percentage of CPU time consumed by a given pod.                                                                                                                                                         | * "pod\_name" (string) The pod name. Should be a non-empty string.                                                  |
|                                        |                                                                                                                                                                                                             |                                                                                                                     |
|                                        | The way the CPU percentage is calculated is provided in the next section.                                                                                                                                   | * "namespace" (string) The namespace of the pod. Should be a non-empty string.                                      |
|                                        |                                                                                                                                                                                                             |                                                                                                                     |
|                                        |                                                                                                                                                                                                             | * "labels" (string) The labels set on the pod. The format is "key1:val1,key2:val2" (as in the Heapster metrics).    |
+----------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------+
| k8s\_namespace\_cpu\_usage             | The percentage of CPU time consumed by all the pods of a given namespace.                                                                                                                                   | * "namespace" (string) The namespace of the pod. Should be a non-empty string.                                      |
+----------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------+
| k8s\_pods\_cpu\_usage                  | The percentage of CPU time consumed by all the pods on the Kubernetes node.                                                                                                                                 |                                                                                                                     |
+----------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------+
| k8s\_containers\_cpu\_usage            | The percentage of CPU time consumed by all the containers on the Kubernetes node. This includes both Kubernetes containers (running in pods) and system containers.                                         |                                                                                                                     |
+----------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------+
| k8s\_pod\_memory\_usag                 | The memory used by a given pod (in Bytes).                                                                                                                                                                  | * "pod\_name" (string) The pod name. Should be a non-empty string.                                                  |
|                                        |                                                                                                                                                                                                             |                                                                                                                     |
|                                        |                                                                                                                                                                                                             | * "namespace" (string) The namespace of the pod. Should be a non-empty string.                                      |
|                                        |                                                                                                                                                                                                             |                                                                                                                     |
|                                        |                                                                                                                                                                                                             | * "labels" (string) The labels set on the pod. The format is "key1:val1,key2:val2" (as in the Heapster metrics).    |
+----------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------+
| k8s\_namespace\_memory\_usage          | The memory used by all the pods of a given namespace (in Bytes).                                                                                                                                            | * "namespace" (string) The namespace of the pod. Should be a non-empty string.                                      |
+----------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------+
| k8s\_pods\_memory\_usage               | The memory used by all the pods on the Kubernetes node.                                                                                                                                                     |                                                                                                                     |
+----------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------+
| k8s\_containers\_memory\_usage         | The memory used by all the containers on the Kubernetes nodes. This includes both Kubernetes containers (running in pods) and system containers.                                                            |                                                                                                                     |
+----------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------+
| k8s\_pod\_major\_page\_faults          | The number of major page faults per second for a given pod.                                                                                                                                                 | * "pod\_name" (string) The pod name. Should be a non-empty string.                                                  |
|                                        |                                                                                                                                                                                                             |                                                                                                                     |
|                                        |                                                                                                                                                                                                             | * "namespace" (string) The namespace of the pod. Should be a non-empty string.                                      |
|                                        |                                                                                                                                                                                                             |                                                                                                                     |
|                                        |                                                                                                                                                                                                             | * "labels" (string) The labels set on the pod. The format is "key1:val1,key2:val2" (as in the Heapster metrics).    |
+----------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------+
| k8s\_namespace\_major\_page\_faults    | The number of major page faults per second for all the pods of a given namespace.                                                                                                                           | * "namespace" (string) The namespace of the pod. Should be a non-empty string.                                      |
+----------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------+
| k8s\_pods\_major\_page\_faults         | The number of major page faults per second for all the pods on the Kubernetes node.                                                                                                                         |                                                                                                                     |
+----------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------+
| k8s\_containers\_major\_page\_faults   | The number of major page faults per second for all the containers on the Kubernetes nodes. This includes both Kubernetes containers (running in pods) and system containers.                                |                                                                                                                     |
+----------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------+
| k8s\_pod\_page\_faults                 | The number of minor page faults per second for a given pod.                                                                                                                                                 | * "pod\_name" (string) The pod name. Should be a non-empty string.                                                  |
|                                        |                                                                                                                                                                                                             |                                                                                                                     |
|                                        |                                                                                                                                                                                                             | * "pod\_uid" (string) The pod UId. Should be a non-empty string.                                                    |
|                                        |                                                                                                                                                                                                             |                                                                                                                     |
|                                        |                                                                                                                                                                                                             | * "namespace" (string) The namespace of the pod. Should be a non-empty string.                                      |
|                                        |                                                                                                                                                                                                             |                                                                                                                     |
|                                        |                                                                                                                                                                                                             | * "labels" (string) The labels set on the pod. The format is "key1:val1,key2:val2" (as in the Heapster metrics).    |
+----------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------+
| k8s\_namespace\_page\_faults           | The number of minor page faults per second for all the pods of a given namespace.                                                                                                                           | * "namespace" (string) The namespace of the pod. Should be a non-empty string.                                      |
+----------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------+
| k8s\_pods\_page\_faults                | The number of minor page faults per second of all the pods on the Kubernetes node.                                                                                                                          |                                                                                                                     |
+----------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------+
| k8s\_containers\_page\_faults          | The number of minor page faults per second for all the containers on the Kubernetes nodes. This includes both Kubernetes containers (running in pods) and system containers.                                |                                                                                                                     |
+----------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------+
| k8s\_pod\_[rx\|tx]\_bytes              | The number of bytes per second received/sent over the network for a given pod.                                                                                                                              | * "pod\_name" (string) The pod name. Should be a non-empty string.                                                  |
|                                        |                                                                                                                                                                                                             |                                                                                                                     |
|                                        |                                                                                                                                                                                                             | * "namespace" (string) The namespace of the pod. Should be a non-empty string.                                      |
|                                        |                                                                                                                                                                                                             |                                                                                                                     |
|                                        |                                                                                                                                                                                                             | * "labels" (string) The labels set on the pod. The format is "key1:val1,key2:val2" (as in the Heapster metrics).    |
+----------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------+
| k8s\_namespace\_[rx\|tx]\_bytes        | The number of bytes per second received/sent over the network for all the pods of a given namespace.                                                                                                        | * "namespace" (string) The namespace of the pod. Should be a non-empty string.                                      |
+----------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------+
| k8s\_pods\_[rx\|tx]\_bytes             | The number of bytes per second received/sent over the network for all the pods on the Kubernetes node.                                                                                                      |                                                                                                                     |
+----------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------+
| k8s\_containers\_[rx\|tx]\_bytes       | The number of bytes per second received/sent over the network for all the containers on the Kubernetes node. This includes both Kubernetes containers (running in pods) and system containers.              |                                                                                                                     |
+----------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------+
| k8s\_pod\_[rx\|tx]\_errors             | The number of errors per second while receiving/sending over the network for a given pod.                                                                                                                   | * "pod\_name" (string) The pod name. Should be a non-empty string.                                                  |
|                                        |                                                                                                                                                                                                             |                                                                                                                     |
|                                        |                                                                                                                                                                                                             | * "namespace" (string) The namespace of the pod. Should be a non-empty string.                                      |
|                                        |                                                                                                                                                                                                             |                                                                                                                     |
|                                        |                                                                                                                                                                                                             | * "labels" (string) The labels set on the pod. The format is "key1:val1,key2:val2" (as in the Heapster metrics).    |
+----------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------+
| k8s\_namespace\_[rx\|tx]\_errors       | The number of errors per second while receiving/sending over the network for all the pods of a given namespace.                                                                                             | * "namespace" (string) The namespace of the pod. Should be a non-empty string.                                      |
+----------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------+
| k8s\_pods\_[rx\|tx]\_errors            | The number of errors per second while receiving/sending over the network for all the pods on the Kubernetes node.                                                                                           |                                                                                                                     |
+----------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------+
| k8s\_containers\_[rx\|tx]\_errors      | The number of errors per second while receiving/sending over the network for all the containers on the Kubernetes node. This includes both Kubernetes containers (running in pods) and system containers.   |                                                                                                                     |
+----------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------------------+

"check" metrics
+++++++++++++++

+---------------+----------------------------------------------------------------------------------------------------------------------------+--------------+
| Metric name   | Description                                                                                                                | Dimensions   |
+---------------+----------------------------------------------------------------------------------------------------------------------------+--------------+
| k8s\_check    | Expresses the success or failure of the metric collection itself. The value of the metric is 0 (failure) or 1 (success).   |              |
+---------------+----------------------------------------------------------------------------------------------------------------------------+--------------+

Hindsight Input Plugin
~~~~~~~~~~~~~~~~~~~~~~

We will develop an Hindsight Input Plugin for the polling of the Kubelet APIs.
This plugin will be kubernetes.lua.

The "mode of operation" of that plugin will be `Polling
<https://github.com/mozilla-services/lua_sandbox/blob/master/docs/heka/input.md#polling>`_.
In this mode of operation Hindsight calls the plugin’s process\_message
function every ticker interval, which is set in the plugin’s configuration file
using the ticker\_interval parameter.

The process\_message function will send requests to kubelet’s stats" API. The
responses will be processed and messages will be injected into the Hindsight
pipeline. The format of messages will respect the `StackLight metric data mode
<http://lma-developer-guide.readthedocs.io/en/latest/metrics.html>`_.

This is an example of a message::

    :Uuid: 64727217-9546-4941-BCEB-75EDC2076
    :Timestamp: 2016-06-15 06:55:04.415659520 +0000 UTC
    :Type: metric
    :Logger: kubelet
    :Severity: 6
    :Pid: 135
    :Hostname: node1
    :Fields:
    | name:name type:0 representation: value:k8s\_pods\_count
    | name:value type:3 representation: value:3
    | name:timestamp type:2 representation: value:1.4659737043588e+18
    | name:hostname type:0 representation: value:node1
    | name:phase type:0 representation: value:running
    | name:namespace type:0 representation: value:kube-system
    | name:dimensions type:0 representation: value:phase,namespace

The plugin will use be based on the standard `LuaSocket
<http://w3.impa.br/~diego/software/luasocket/http.html>`_ for the querying of
the kubelet APIs.

Polling
~~~~~~~

The Hindsight plugin will query the "kubelet" stats API every ticker
interval, which will be set to 10 seconds.

The CPU usage in percentage will be calculated as follows:

(usageCoreNanoSeconds(tn) - usageCoreNanoSeconds(tn+1)) \* 100 / (tn -
tn+1)

where usageCoreNanoSeconds is the cumulative CPU usage (for all CPUs/cores) in
nanoseconds since the creation of the object. See `link
<https://github.com/kubernetes/heapster/blob/32d5425fbcfe3cebc4b86b432011444d6bd2377f/vendor/k8s.io/kubernetes/pkg/kubelet/api/v1alpha1/stats/types.go#L133-L134>`_.

The same sort of calculation will be used for other "rate" metrics, such as
k8s\_pod\_rx\_bytes.

Grafana Dashboard
~~~~~~~~~~~~~~~~~

A new "Kubernetes" Grafana dashboard will be created. This dashboard
will include graphs for the metrics provided further up in this
specification.

For example, the "Kubernetes" dashboard will include a graph that
displays the CPU usage of a namespace on a node. For that the user will
select the hostname and namespace using Templating-based dropdowns.

Useful Links
~~~~~~~~~~~~

* Heapster metrics: https://github.com/kubernetes/heapster/blob/master/docs/storage-schema.md
* Use of "stats/summary" API in Heapster: https://github.com/kubernetes/heapster/blob/a861ef2b55f6a786e2ef12d958932849b6266e2e/metrics/sources/summary/summary.go
* Kubedash screenshots: https://github.com/kubernetes/kubedash/tree/master/screenshots
* Kubernetes Performance Measurements and Roadmap: http://blog.kubernetes.io/2015/09/kubernetes-performance-measurements-and.html
* How to monitor Docker resource metrics (by Datadog): https://www.datadoghq.com/blog/how-to-monitor-docker-resource-metrics
* Top Docker Metrics to Watch (by sematext): https://sematext.com/blog/2016/06/28/top-docker-metrics-to-watch
* Example of kubelet "stats/summary" response: https://gist.github.com/elemoine/f3f60118594550bc38f49ecae693c9af

Alternatives
~~~~~~~~~~~~

None

Data model impact
~~~~~~~~~~~~~~~~~

None

REST API impact
~~~~~~~~~~~~~~~

None

Upgrade impact
~~~~~~~~~~~~~~

None

Security impact
~~~~~~~~~~~~~~~

None

Notifications impact
~~~~~~~~~~~~~~~~~~~~

None

Other end user impact
~~~~~~~~~~~~~~~~~~~~~

None

Performance Impact
~~~~~~~~~~~~~~~~~~

None

Assignee(s)
~~~~~~~~~~~

* Project Owner: Patrick Petit <ppetit@mirantis.com>
* Feature Lead: Eric Lemoine <elemoine@mirantis.com>
* Primary assignee: Eric Lemoine <elemoine@mirantis.com>

Other contributors:

* Olivier Bourdon <obourdon@mirantis.com>

Mandatory design review:

* Patrick Petit <ppetit@mirantis.com>
* Simon Pasquier <spasquier@mirantis.com>

Work Items
~~~~~~~~~~

*  Develop a "kubernetes" input plugin for Hindsight
*  Create a "kubernetes" dashboard in Grafana

Dependencies
~~~~~~~~~~~~

None

Testing
~~~~~~~

Functional tests to be created. In particular we need to verify that we
get the data we need from the kubelet "stats" API. This means running
Hindsight on a Kubernetes node and checking that Hindsight does collect
all the metrics provided in this specification.

Acceptance criteria
~~~~~~~~~~~~~~~~~~~

1. A new "Kubernetes" dashboard in available in Grafana.
2. This new dashboard includes graphs for the metrics included in this
   specification.

Documentation Impact
~~~~~~~~~~~~~~~~~~~~

None
