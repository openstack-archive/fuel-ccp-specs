.. _intitial-priorities:

==========================
NextGen Initial Priorities
==========================

List of themes and corresponding specs that should be covered as the first
step.


+------------------------------+---------------------------+
| Priority                     | Spec Driver               |
+==============================+===========================+
| `Dev env`_                   | `Artur Zarzycki`_         |
+------------------------------+---------------------------+
| `CI/CD pipeline arch`_       | `TBD`_                    |
+------------------------------+---------------------------+
| `CI jobs`_                   | `TBD`_                    |
+------------------------------+---------------------------+
| `Repos break down`_          | `Michal Rostecki`_        |
+------------------------------+---------------------------+
| `Docker layers & versions`_  | `Kirill Proskurin`_       |
+------------------------------+---------------------------+
| `Docker build arch`_         | `Oleksandr Mogylchenko`_  |
+------------------------------+---------------------------+
| `Multiple envs per Mesos`_   | `TBD`_                    |
+------------------------------+---------------------------+
| `LMA integration`_           | `Eric Lemoine`_           |
+------------------------------+---------------------------+

.. _Artur Zarzycki: https://directory.mirantis.com/#/profile/azarzycki
.. _Michal Rostecki: https://directory.mirantis.com/#/profile/mrostecki
.. _Kirill Proskurin: https://directory.mirantis.com/#/profile/kproskurin
.. _Oleksandr Mogylchenko: https://directory.mirantis.com/#/profile/amogylchenko
.. _Eric Lemoine: https://directory.mirantis.com/#/profile/elemoine
.. _Ruslan Kamaldinov: https://directory.mirantis.com/#/profile/rkamaldinov
.. _Sergey Lukjanov: https://directory.mirantis.com/#/profile/slukjanov
.. _TBD: #

Dev env
-------

We need to have a spec to cover the following aspects of the Dev Env:

* Installer - we should have a single reusable installer that will be used for
  both dev env and at least small labs installation.
* Should be deisgned to work together with Vagrant while supporting manually
  created VMs and/or OpenStack cloud to provision them.

Current vision of how we are going to approach is desribed in `NextGen CI/CD
Proposal`_ document. So, it should be used as at least as a base for spec(s).

Here we need to have very clear short-term and long-term plans that will
include intention to have both single node (AIO) and multinode dev envs.

Additional info re installer could be found in `NextGen Installation Proposal`.

Contact persons for more details:

* `Ruslan Kamaldinov`_


CI/CD pipeline arch
-------------------

CI/CD pipeline should be described in terms of used components (Zuul, Jenkins,
Nodepool, Artifactory, Statis files, Logs, ELK, etc.) and approache of building
everything from source code (project-config repo). The repo structure, used
URLs, Mirantis clouds and HW planned to use should be described, etc.

Current vision of how we are going to approach is desribed in `NextGen CI/CD
Proposal`_ document. So, it should be used as at least as a base for spec(s).

More info about CI architecture could be found in `Wiki - CI Design`.

Contact persons for more details:

* `Ruslan Kamaldinov`_


CI jobs
-------

List of CI jobs to run for all components and their dependencies that will
cover our needs to run style checks, unit tests, functional and integration
tests for all system components and OpenStack components.

Current vision of how we are going to approach is desribed in `NextGen CI/CD
Proposal`_ document. So, it should be used as at least as a base for spec(s).

Contact persons for more details:

* `Ruslan Kamaldinov`_


Repos break down
----------------

We need to cover the repositories breakdown from a two repos we have now
(Kolla and Kolla-Mesos) to component specific repositories with initial
structure for each of this repositories. It should additionally include
prerequisites and requirements for separation.

Current vision of how we are going to approach is desribed in `NextGen CI/CD
Proposal`_ document. So, it should be used as at least as a base for spec(s).

Contact persons for more details:

* `Ruslan Kamaldinov`_
* `Sergey Lukjanov`_


Docker layers & versions
------------------------

We need to describe which layering strategy we'll use for the NextGen project
as well as naming and versioning strategies for the generated docker images.
We need to keep in mind that docker could be replaced with some other
containerization tooling, so, when possible, the approaches should be more
generic that specific to docker itself.

Current vision of how we are going to approach is desribed in `NextGen CI/CD
Proposal`_ document. So, it should be used as at least as a base for spec(s).

Contact persons for more details:

* `Ruslan Kamaldinov`_
* `Sergey Lukjanov`_


Docker build arch
-----------------

We need to describe how we'll be storing or generating Dockerfiles for building
container images instead of just mixing different configs and types in a single
Jinja2 template. There are some existing tools like Packer for doint it.

We need to keep in mind that docker could be replaced with some other
containerization tooling, so, when possible, the approaches should be more
generic that specific to docker itself. Because of that, we need to think about
possibility to implement own yaml-based DSL to store lists of the packages to
be installed and generating Dockerfile from them to build docker images and
make it pluggable to support rkt images generating as well.

Current vision of how we are going to approach is desribed in `NextGen CI/CD
Proposal`_ document. So, it should be used as at least as a base for spec(s).

Contact persons for more details:

* `Ruslan Kamaldinov`_
* `Sergey Lukjanov`_


Multiple envs per Mesos
-----------------------

We need to have an ability to deploy multiple OpenStack clouds per single
Mesos stack cluster. The main reason for this requirement is to enable more
effective HW usage for CI/CD and for the development environments.

There is already bunch of potential issues and ideas collected in different
google docs and etherpads, so, all of them should be aggregated into a single
spec that will answer the question - what should be fixed / implemented to
enable multiple OpenStack clusters deployment on top of single Mesos cluster.

So far, it sounds like we need to:

* switch to port mapping + LB / service discovery
* impl somehow compute / network node pinning

LMA integration
---------------

We need to have a short-term plan to integrate minimal logging and monitoring
into the NextGen project as well as the long-term plan to integrate
Stacklight for providing the full logging, monitoring and alerting stack.
The main task in this theme is to cover the short-term plans with concrete
solution and implementation design.

The first thing to cover now is to collect logs across the NextGen cluster
including how we'll be pushing/pulling logs from containers and which
storage will be used for storing and accessing them.

Plan is to have two specs:

* general logging/monitoring/alerting spec discussing the whole architecture.
  This will provide the highlevel picture.
* spec focusing on central logging (with Heka and Elasticsearch). This will be
  a rewrite of the `Upstream LMA spec for Kolla`.


.. _NextGen CI/CD Proposal: https://docs.google.com/document/d/1rtuINpfTvUFc1lJLuMK5LIzGKWwdh8uRUTvOLKnXuPM/edit
.. _NextGen Installation Proposal: https://docs.google.com/document/d/1TZ2X3QhmJ4bjl4bcFyLxBBbDMpl3hV6mZdq3gmD-rPo/edit
.. _Wiki - CI Design: https://mirantis.jira.com/wiki/display/NG/CI+design
.. _Upstream LMA spec for Kolla: https://github.com/openstack/kolla/blob/master/specs/logging-with-heka.rst
