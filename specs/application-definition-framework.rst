===================================================================
Application/service definition framework and external configuration
===================================================================

https://mirantis.jira.com/browse/MCP-253

This specification proposes the creation of application definition framework
for deploying containers using Kubernetes.

Problem description
===================

We should have a framework that will allow us to define
applications/services/roles and generate k8s objects from them.

Use Cases
---------

The application definition framework will allow us to define services in
declarative form to run them in containers on top of k8s.

Proposed change
===============

Containers per pod
------------------

Initially only one container will be created per pod to simplify their
management, to use current and future affinity features [1] and because in
general we don't want all containers of the same service to be launched on one
node. Multiple-container support will be added later.

RC vs Deployments
-----------------

ReplicationControllers can be used to provide desired number of pods.

Alternative is to use Deployment objects, which provide declarative and
server-side rolling updates, however rolling update for ReplicationController
is a client-side magic. Also Deployment objects have maxUnavailable and
maxSurge parameters and store revision history that allows to perform
rollbacks.

The only problem is that Deployment object is not present in current stable
api (it is currently in beta one). We will most likely migrate from RC to
Deployments once they will be released. But probably it's worth using
Deployments from the beginning because seems like there are no open proposals
that could change already existing api. In addition, in ``kubectl run`` command
ReplicationControllers are already replaced with Deployments.

The proposition is to use Deployments for that.
Also DaemonSets will be used for some particular cases. For example, for the
compute nodes and for monitoring agents.

Service objects
---------------

For balancing between pods of the same service and for internal communication,
services with default ``ClusterIP`` type will be created.

ConfigMap objects
-----------------

ConfigMap objects will be used to store configuration files, configuration
parameters (jinja variables and jinja macros). In these case one `general` and
one or several service-related ConfigMap objects will need to be defined.

Job objects
-----------

Job objects will be used to perform commands with type `single`. We need
to control their execution order as far as Workflow object is not
implemented [4]. Execution order will be controlled via dependencies stored
in etcd.

Pods/Containers readiness/health checking
-----------------------------------------

Low-level checks are enabled by default (Kubelet asks Docker daemon if the
container process is still running). For application-specific checks k8s
Probes will be used. Both livenessProbes and readinessProbes will be defined.

In case we only want to check endpoint accessibility, ``httpGet`` action will
be used. If anything else need to be checked, we should use ``exec`` action.
Several actions can't be combined together, so endpoint accessibility check
need to be performed with ``exec`` action by ourselves in this case. [10]

Namespaces, resources quotas and limits
---------------------------------------

Namespaces will be used to separate resources of one deployment from another.
A Namespace object will be created for each deployment.

ResourceQuotas will be used to define hard resource usage limits that a
Namespace may consume.
A ResourceQuota object will be created for each namespace and will limit
the total cpu and memory requests of containers.

Limits will be used to define default constraints on the amount of
resources a container can consume in a Namespace.
Limits will be defined as part of container spec definition.

k8s python-client selection
---------------------------

Several options like pykube[6], kubernetes-py[7], libcloud[8] were explored,
but all of them either outdated or no longer being contributed or not
convenient for our cases.

So, probably the best option is to use client generated with swagger for k8s
api, like this one [9]. Moreover if we will come up with idea to use not stable
api (like v1beta1), client for this api could be generated with swagger as
well (all other client don't support not stable versions and even after
release nobody know when(if) they will be updated).

Commands
--------

There are two types of commands will be supported: `single` and `always`.

*Single* command will be executed on single container per microservice.
For such kind of commands k8s Job objects will be used.
Job will use the same image for containers as a service does.
*Always* command will be executed on each container of microservice.
Such kind of commands will be executed inside the container with start script.

Status of `single` commands will be stored in etcd in
`/<env_id>/openstack/roles/<microservice-name>/commands/<command_name>` key.
Status of `always` command will not be stored.

These commands are combined in two groups: `pre` and `post`.
`pre` commands are executed before the daemon process is started, `post` -
after it's started.

Start script
------------

`Start script` will be implemented and stored in `nextgen/ms-ext-config`
repository.
`Start script` will be executed inside a container.
`Start script` will be uploaded to ConfigMap and will be executed from it.

Its purposes:

* render service configs with jinja variables, move them to the right place
  and set right permissions
* wait for the dependency commands completion
* launch pre and post commands with type `always`
* launch daemon/job command

`Start script` should support signals that will be handled and sent to daemon
process. The following signals should be handled:

* SIGHUP
* SIGINT
* SIGTERM

Templating and rendering
------------------------

Configuration files for services will be stored in ConfigMaps in Jinja2 format.
Configuration files will be rendered inside a container with jinja variables
and macros. Rendering will be provided by start script.

The same approach as for kolla-mesos can be used to initially store the
parameters. There can be one file with general service-independent parameters
and one file per service with service-related defaults, which can be overridden
in the general file.

Each service will expose some set of jinja macros, that will be used during
rendering. Macros for each service will be located in these service repository.

When all parameters and macros will be aggregated, the `global` ConfigMap which
is include all of them will be created. These ConfigMap will be attached to
each container as a volume.

Passwords can be generated and uploaded to secrets instead. However
secrets are not secured somehow yet [3].

Service definition
------------------

Both k8s objects definition, configuration and commands workflow will be
defined within one yaml file per service. These yaml file will be located in
each service repository.

Based on these yaml, k8s objects (that described above) will be generated +
workflow definition for start script that will be executed inside containers
will be created (both for deployments/jobs/daemonsets/etc).

Example of service definition
-----------------------------

.. sourcecode:: yaml

  service:
      name: keystone
      container:
          privileged: false
          cpu: 100
          ram: 500
      ports:
          - keystone_public_port
          - keystone_admin_port
      probes:
          readiness: check_readiness.sh
          liveness: check_liveness.sh
      pre:
          - name: create_db
            dependencies:
                - mariadb
                - somethingelse
            files:
                - keystone_conf
                - config_name
            type: single
            command: some command
            user: keystone
      post:
          - name: post_command
            ...
      daemon:
          files:
            - keystone-conf
          command: daemon.sh

  files:
      keystone-conf:
          path: /etc/keystone/keystone.conf
          content: keystone.conf.j2
          perm: "0600"
          user: keystone

Example of workflow definition
------------------------------

This is what start script consumes via configmap (this file is generated by
framework and should not be defined):

.. sourcecode:: yaml

workflow:
  daemon:
      command: daemon.sh
      user: keystone
  dependencies:
      - keystone-db-create
      - keystone-db-sync
      - keystone-db-bootstrap
  files:
      - name: keystone-conf
        path: /etc/keystone/keystone.conf
        perm: "0600"
        user: keystone
  name: keystone
  pre:
      - command: pre_command
        user: keystone
  post:
      - command: post_command
        user: keystone

If daemon should be executed inside a container, "daemon:" key will be present.
If it is a job container, "job:" key will be present.

Workflow
--------

1. fetch repositories with application definitions
2. fetch repository with start script
3. find all parameters and jinja macros and create "global" ConfigMap which
   will have all defined parameters and start script.
4. create service-unrelated objects like Namespace and ResourceQuota
5. find all application definitions and process them one by one
   (ConfigMaps should be created before other k8s objects for the particular
   service)


Alternatives
------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  apavlov-n
  sreshetniak

Other contributors:
  slukjanov

Work Items
----------

* application definition file parser should be implementated
* methods for creation of k8s objects should be implemented
* start script should be implemented
* existing services should be adopted for new app definition framework

Dependencies
============

* k8s python client
* Jinja2 library


Testing
=======

* syntax checks for application definitions (YAML files)
* checks that services deployed from application definitions work correctly


Documentation Impact
====================

Documentation on how to add a new service with description of all needed
steps should be written. This documentation should cover the format of
application definition.

Also service deployment workflow should be documented.

References
==========

[1] https://mirantis.jira.com/wiki/display/NG/k8s+application+scheduling
[3] https://github.com/kubernetes/kubernetes/issues/12742
[4] https://github.com/kubernetes/kubernetes/blob/master/docs/proposals/workflow.md
[5] https://review.fuel-infra.org/#/c/18959/17/specs/repositories-split.rst
[6] https://github.com/kelproject/pykube
[7] https://github.com/mnubo/kubernetes-py
[8] https://libcloud.apache.org/blog/2016/02/05/libcloud-containers-example.html
[9] https://github.com/openstack/python-k8sclient
[10] https://mirantis.jira.com/wiki/display/NG/k8s+containers+probing


History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Mitaka
     - Introduced
