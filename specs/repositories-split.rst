==========================================
Repositories split
==========================================

MCP project is going to use splitted repositories for Dockerfiles and
config files per each OpenStack service and infra service.


Problem description
===================

MCP project wants to move out from using Kolla images and it's aiming to
provide its own repositories for Dockerfiles and configuration files

The layout of Kolla repository, which contains all the Dockerfiles and config
files is not satysfying NextGen needs.

Pros for splitting repositories:

* Possibility to engage engineers from teams to take some responsibility on
  containerizing each component.
* Similarity to the Puppet, Chef and Ansible repositories in upstream.

Cons:

* CI will have to handle cross-repository dependencies. That may be hard
  especially in case of fixing the issues which have to be fixed in the
  configuration of multiple OpenStack components at once, because CI will become
  green after accepting a stream of patches in many repos.
* Release management and stable branch maintenance will be harder, because
  the whole Microservices project will contain ~38 repositories.

Use Cases
---------

Proposed change
===============

The "nextgen" organization should be created on Gerrit with following git
repositories:

* nextgen/microservices - the builder of Docker images and CLI tool for pushing
  configuration in etcd and deploying OpenStack on Kubernetes
* nextgen/ms-ext-config - start script for containers which fetches config
  from etcd/ZooKeeper
* nextgen/ms-debian-base - the layer for base Linux packages
* nextgen/ms-openstack-base - the base layer for OpenStack services, containing
  common Python modules
* nextgen/ms-aodh-api
* nextgen/ms-aodh-base
* nextgen/ms-aodh-evaluator
* nextgen/ms-aodh-expirer
* nextgen/ms-aodh-listener
* nextgen/ms-aodh-notifier
* nextgen/ms-ceilometer-api
* nextgen/ms-ceilometer-base
* nextgen/ms-ceilometer-central
* nextgen/ms-ceilometer-collector
* nextgen/ms-ceilometer-compute
* nextgen/ms-ceilometer-notification
* nextgen/ms-ceph-base
* nextgen/ms-ceph-mon
* nextgen/ms-ceph-osd
* nextgen/ms-ceph-rgw
* nextgen/ms-cinder-api
* nextgen/ms-cinder-backup
* nextgen/ms-cinder-base
* nextgen/ms-cinder-rpcbind
* nextgen/ms-cinder-scheduler
* nextgen/ms-cinder-volume
* nextgen/ms-designate-api
* nextgen/ms-designate-backend-bind9
* nextgen/ms-designate-base
* nextgen/ms-designate-central
* nextgen/ms-designate-mdns
* nextgen/ms-designate-poolmanager
* nextgen/ms-designate-sink
* nextgen/ms-elasticsearch
* nextgen/ms-glance-api
* nextgen/ms-glance-base
* nextgen/ms-glance-registry
* nextgen/ms-heat-api
* nextgen/ms-heat-api-cfn
* nextgen/ms-heat-base
* nextgen/ms-heat-engine
* nextgen/ms-heka
* nextgen/ms-horizon
* nextgen/ms-ironic-api
* nextgen/ms-ironic-base
* nextgen/ms-ironic-conductor
* nextgen/ms-ironic-inspector
* nextgen/ms-ironic-pxe
* nextgen/ms-keystone
* nextgen/ms-kibana
* nextgen/ms-toolbox
* nextgen/ms-magnum-api
* nextgen/ms-magnum-base
* nextgen/ms-magnum-conductor
* nextgen/ms-manila-api
* nextgen/ms-manila-base
* nextgen/ms-manila-scheduler
* nextgen/ms-manila-share
* nextgen/ms-mariadb
* nextgen/ms-memcached
* nextgen/ms-mistral-api
* nextgen/ms-mistral-base
* nextgen/ms-mistral-engine
* nextgen/ms-mistral-executor
* nextgen/ms-mongodb
* nextgen/ms-murano-api
* nextgen/ms-murano-base
* nextgen/ms-murano-engine
* nextgen/ms-neutron-base
* nextgen/ms-neutron-dhcp-agent
* nextgen/ms-neutron-l3-agent
* nextgen/ms-neutron-linuxbridge-agent
* nextgen/ms-neutron-metadata-agent
* nextgen/ms-neutron-openvswitch-agent
* nextgen/ms-neutron-server
* nextgen/ms-nova-api
* nextgen/ms-nova-base
* nextgen/ms-nova-compute-ironic
* nextgen/ms-nova-compute
* nextgen/ms-nova-conductor
* nextgen/ms-nova-consoleauth
* nextgen/ms-nova-libvirt
* nextgen/ms-nova-network
* nextgen/ms-nova-novncproxy
* nextgen/ms-nova-scheduler
* nextgen/ms-nova-spicehtml5proxy
* nextgen/ms-openvswitch-base
* nextgen/ms-openvswitch-db-server
* nextgen/ms-openvswitch-vswitchd
* nextgen/ms-rabbitmq
* nextgen/ms-sahara-api
* nextgen/ms-sahara-base
* nextgen/ms-sahara-engine
* nextgen/ms-swift-account
* nextgen/ms-swift-base
* nextgen/ms-swift-container
* nextgen/ms-swift-object
* nextgen/ms-swift-proxy-server
* nextgen/ms-swift-rsyncd
* nextgen/ms-tempest
* nextgen/ms-trove-api
* nextgen/ms-trove-base
* nextgen/ms-trove-conductor
* nextgen/ms-trove-guestagent
* nextgen/ms-trove-taskmanager
* nextgen/ms-zaqar
* nextgen/dev-env - Vagrantfile for creating the development environment
* nextgen/puppet-deployment - Puppet mianifests for deploying Kubernetes and all
  infra aroud it
* nextgen/project-config - config of NextGen/Microservices repositories

Repositories structure
----------------------

nextgen/microservices
~~~~~~~~~~~~~~~

This repository will be an usual Python package, made from the OpenStack
cookiecutter template[1].

nextgen/ms-ext-config
~~~~~~~~~~~~~~~~~~~~~

This repository will be an usual Python package as well.

nextgen/ms-*
~~~~~~~~~~~~

The structure of this repository will look like:

* *docker/*
  * *image.json.j2 / Dockerfile.j2* - template of appc definition or Dockerfile
  (TBD), rendered and consumed by microservices CLI
* *config/*
  * *<service_name>.conf.j2* - template of config file
  * *<service_name>-some-other-config-file.conf.j2* - template of config file
* *service/*
  * *<service_name>.yaml.j2* - service definition for Kubernetes

nextgen/dev-env
~~~~~~~~~~~~~~~

The structure of this repository will look like:

* *Vagrantfile* - main Vagrantfile
* *Vagrantfile.custom.example* - example config file for Vagrant
* *provision.sh* - provisioning script

Alternatives
------------

Kolla community is working on spec about running the kolla-kubernetes project.[2]
The only difference between this project and our planned Microservices project is
a single git repository for all Dockerfiles.

Data model impact
-----------------

There is no data model impact.

REST API impact
---------------

There is no REST API impact.

Security impact
---------------

There is no security impact.

Notifications impact
--------------------

There is no notification impact.

Other end user impact
---------------------

There shoudn't be impact on the end user. The experience which currently is
comming from kolla-build and kolla-mesos utilities shouldn't be changed from
user's point of view and we can even achieve a compatibility of configuration
and usage if we want.

However, in addition to the backwards compatibility, we are going to provide
all needed tooling to be able to easy deploy and plug-in any set of services
just by using nextgen/microservies tool.

Performance Impact
------------------

There will be two options regarding the service repositories.

First option is to clone all needed repositories. That will decrease performance,
but not in critical way.

Second option is to specify already cloned repositories. That will not change
anything in performance.

Other deployer impact
---------------------

Deployer will have to be aware of using the multiple git repositories when
deploying NG OpenStack and Kubernetes infra.

Developer impact
----------------

There will be an impact on developer's experience:

* CI will have to use many repositories and check whether the deployment from
  the current Code Review + the trunk of the other NextGen repositories is
  working.
* In case of cross-repository failures, developers should care about making
  appropriate dependencies by Depends-On clause in commits as well as handling
  and contributing the changes in the other repositories if needed.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Michal Rostecki

Work Items
----------

* Create the repositories for all OpenStack big tent services, infra services
  around them and base layers in fuel-infra Gerrit. The name of these
  repositories should follow the following pattern: "ms-<service_name>".
* Move kolla-build tool from openstack/kolla repository to ng-builder repository.
* Move kolla-mesos tool from openstack/kolla-mesos repository to nextgen/microservices
  repository.
* Make changes to build and mesos deployment tools to fetch the external git
  repositories.
* Move the Dockerfiles from openstack/kolla repository to the appropriate
  NextGen repositories.
* Move the configuration files from openstack/kolla and/or openstack/kolla-mesos
  to the appropriate NextGen repositories.


Dependencies
============

* Specs about Docker build architecture [3] and layers [4].


Testing
=======

The testing scenarios for NextGen will be similar to what was planned
during our work on kolla-mesos. But of course it should include change
from Mesos to Kubernetes and repositories splitup.

Testing scenarios for kolla-mesos were described on Mirantis wiki[5].
We need to re-write them when implementing this bleuprint.

However, there will be some infrastructure impact due to running tests
separately for each repo.


Documentation Impact
====================

Only parts of documentation describing the kolla-build and kolla-mesos CLI-s
can be preserved with the small changes. Every other part of documentation
should be rewritten from scratch.


References
==========

* https://docs.google.com/document/d/1rtuINpfTvUFc1lJLuMK5LIzGKWwdh8uRUTvOLKnXuPM/edit
* [1] https://github.com/openstack-dev/cookiecutter
* [2] https://review.openstack.org/#/c/304182/
* [3] https://review.fuel-infra.org/#/c/18867/
* [4] https://review.fuel-infra.org/#/c/19028/
* [5] https://mirantis.jira.com/wiki/display/KOLLA/CI+jobs
