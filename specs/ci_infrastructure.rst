==================
CI Infrastructure
==================

This spec captures the work necessary to provision CI infrastructure for the
project. This spec covers following topics:

* Underlying hardware and virtualized infrastructure
* Source code repositories
* CI tools - Jenkins, Zuul, Nodepool, etc
* Methods of deployment of CI tools
* Configuration of CI infrastructure

Problem description
===================

There are several goals for CI infrastructure:

* Reproducibility - it should be a matter of a few actions to roll out new
  infrastructure if the old one is gone.
* Re-use existing tools - reuse whatever is available from MOS Infra and
  upstream OpenStack Infra.
* Dogfooding - we should run every infrastructure service on top of OpenStack
  in the same way upstream OpenStack Infra is running on top of public clouds.
  Possibly we should use MCP, then MOS and Mirantis cloud.

Use cases
---------


Proposed change
===============

Services
--------

Setup infrastructure and automation for provisioning of the following
components:

1. Jenkins
   Jenkins will run CI jobs

2. Nodepool and VM-based workers
   Nodepool will provision single-use worker machines for Jenkins on infra OpenStack
   cloud. Following types of workers will be provisioned:
   * Build machine
     ** With Docker engine pre-installed
     ** Python dependencies pre-installed
     ** Everything else needed to build and publish Docker images

3. Hardware Jenkins workers
   These servers will be used for deployment jobs (each job can spawn multiple
   VMs e.g. multinode K8s cluster).

3. Zuul
   Zuul will launch Jenkins jobs based on events from Gerrit

4. LMA (Logging, Monitoring, Alerting) services and tools:

The LMA subsystem will consist of various components:
4.1. Static log storage
     Will be used to publish logs from automated CI jobs

4.2. Advanced logging and searching services:
This part of LMA is very important but not critical for operation of CI.
It will be covered under separate research including the following technologies:
* ELK stack (ElasticSearch, Logstash, Kibana), if possible integrated with
Jenkins using existing solution. The idea is to send logs from CI jobs to ELK
for further analysis.
* ElasticRecheck: if possible to be based on upstream solution
  http://docs.openstack.org/infra/system-config/elastic-recheck.html
* Stacklight
  https://launchpad.net/lma-toolchain

5. Artifact repository.
   This repository will serve several purposes:
   * Storage for Docker images
   * Local cache for DEB packages
   * frozen repositories (pypi, Deb, Docker) to prevent dynamic updates from
   public repos
   For internal development any repository can be used (e.g. simple Docker
   registry) in the beginning of development, later on more complex solutions
   will be evaluated (e.g. Artifactory, Nexus).

Infrastructure
--------------

* https://mirantis.jira.com/wiki/display/NG/Lab+environment

Alternatives
------------


Implementation
==============


Assignee(s)
-----------
Primary assignee:
  Mateusz Matuszkowiak

Work items
----------

* deploy HW workers
* create VMs
* configure networking
* deploy all services from Puppet manifest
* configure related repositories
* create sample jobs

Services are documented here:
https://mirantis.jira.com/wiki/display/NG/CI+design


Dependencies
============


Testing
=======

Basic tests of CI infrastructure should be done by running sample jobs
that execute "hello-world" type of shell scripts on selected nodes (bare metal,
VM or on K8s cluster).

Documentation impact
====================


References
==========

.. _Jenkins: http://docs.openstack.org/infra/system-config/jenkins.html

.. _Zuul: http://docs.openstack.org/infra/zuul/

.. _Nodepool: http://docs.openstack.org/infra/nodepool/

History
=======
