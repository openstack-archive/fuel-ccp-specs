=====================================
Container build architecture
=====================================

We need to describe how we’ll be storing or generating container specifications

Problem description
===================

Docker proved itself to be production ready and is widely accepted as a ready-to-use
Linux container toolbox.

There are at least 3 different container specifications at the moment, all of
them at different state of production readiness, community acceptance and
interoperability:

* DockerSpec_;
* ACI_;
* OCI_;

It should also be noted that Docker is a ready to use tool, where specification
is provided more as a documentation, rather than a motivation for future changes.
Both ACI and OCF are specifications:

* RKT is an implementation of ACI on Linux, and Jetpack implements ACI
  specification on FreeBSD;
* runC_ is an implementation of OCI_ specification on Linux;

Docker, on the other hand, relies on compatibility layer to run on FreeBSD.

Current efforts to bring ACI support to Docker have not yet succeeded (see
https://github.com/docker/docker/issues/9538,
https://github.com/docker/docker/pull/10776,
and https://github.com/docker/docker/issues/10777)

Starting from `1.11 release
<https://github.com/docker/docker/releases/tag/v1.11.0>_` Docker now support
OCI_ runtime specification and allows natively running containers defined in
that format. The drawback here is that OCI_ image format is not defined yet,
and runC_ does not support Docker layers.

Use Cases
---------

Containers are especially useful in micro-services architectures, where full
isolation of application environment is usually an overkill. Containers
provide an easy and declarative way to maintain environment for services.

Proposed change
===============

All containers should use Docker as target platform, and Debian as base OS.
Container configuration should be shipped in Jinja2 template format, which is
made in accordance to Docker format (e.g. Dockerfile). For example:

.. code::

  FROM {{ namespace }}/{{ image_prefix }}base:{{ tag }}
  MAINTAINER {{ maintainer }}

  RUN apt-get install -y --no-install-recommends \
          cron \
          logrotate \
      && apt-get clean

  {% endif %}

  COPY extend_start.sh /usr/local/bin/extend_start
  RUN chmod 755 /usr/local/bin/extend_start

  {{ include_footer }}

Full documentation about templating engine can be found here:

http://jinja.pocoo.org/docs/dev/

Dockerfiles should not contain exact package versions:

  * this simplifies maintenance tasks and removed the burden from the
    development team;

  * avoids breaking dependencies which are maintained by upstream developers;

  * brings the risk of fetching broken upstream packages - this should be
    mitigated either by having local mirrors, or by additional tests or by
    conbination of those two;

See Dependencies_ for full list of specs which define exact layering scheme and
format of the repositories.

Each commit entering build system will flow according to the sheme below:

+-------------+     +------------------------------------------------+
|    GIT      |     |    Jenkins                                     |
|             |     |                                                |
|Microservices|     |                                                |
|    repo     +--+--->Monitor Changes+------------>New commit        |
|             |  ^  |                                   +            |
| Application |  |  |                                   |            |
|    repo     |  |  |                                   |            |
+-------------+  |  |                                   |            |
                 |  |                                   v            |
+-------------+  |  |  Success<--------------------+Run spec         |
| Artifactory |  |  |      +                          tests          |
|             |  |  |      |                                         |
|  Images of  +--+  |      |                                         |
| lower layers|     |      |                                         |
|             |     |      v                                         |
+-------------+     | Build Docker<-------------------+              |
                    |    image                        |              |
                    |                                 |              |
                    |                                 +              |
                    | Run integration          Trigger rebuild       |
                    |    tests              of all dependent images  |
                    |                                 ^              |
                    |                                 |              |
                    |     Push                        |              |
                    |  to registry+-------------------+              |
                    |                                                |
                    +------------------------------------------------+

CI/CD build artifacts should follow certain rules:

* artifacts built from master branch should be kept for a long period of time
  for investigations;

* artifacts built from other branches should be rotated on a weekly basis with
  a possibility to mark certain build as an exception and keep it for a longer
  period of time;

Alternatives
------------

* implement own DSL with compiler capable of producing Dockefile/ACI/OCI spec

  * additional burden on the team;
  * Docker, ACI and OCI are not 100% compatible in terms of features, so DSL
    will only implement some subset of features;

* incorporate `Packer.io <https://www.packer.io/>`_, which allows building
  VM images with any available provisioner like Puppet or Salt

  * this approach moves the logic of VM configuration from declarative
    Dockerfile to Puppet/Salt/whatever, making it yet another logic element;
  * does not really provide a way to move to ACI because of
    https://github.com/mitchellh/packer/issues/1905

* migrate to OCI runtime spec:

  * allows to reuse existing tools for testing configurations, like
    https://github.com/huawei-openlab/oct
  * OCI image format is not formalized yet, and runC_ does not support layers;

* keep container image configuration in Docker format, but migrate runtime to
  runC_:

  * allows to keep well known configuration format and layers;
  * requires additional convertion before starting containers;
  * Docker immutable images feature will not be preserved;

* use Ubuntu as Debian-compatible OS as a base:

  * Ubuntu's copyright policy is unsuitable not only for Mirantis, but for
    rest of the world as well, see:
    https://mjg59.dreamwidth.org/39913.html

Implementation
==============

Concrete implementation may differ in details. For example, Zuul might be used
for launching new jobs instead of Jenkins events. This does not affect the spec
and depends on the CI/CD engineer.

Assignee(s)
-----------

Primary assignee:
  TBD

Work Items
----------

#. Configure jenkins using the above schema for workflow

#. Setup spec tests using rspec

#. Configure integration tests and write tests

Dependencies
============

#. `Docker layers & versions`

#. `Repositories split`

#. Jenkins

#. Docker registry

Testing
=======

Tests should be executed on each commit to ensure green CI/CD pipeline. Minimal
set of tests should ensure that:

* Docker logic stays valid;

* integration with other components is not broken;

Exact tooling is out of scope of this spec, but I believe the following
projects might be good candidates:

* https://github.com/mizzy/serverspec (with
  https://github.com/swipely/docker-api);

* https://github.com/huawei-openlab/oct (Docker support is under development,
  but project's aim is to provide integrated test pipeline for any container
  specification available);

In addition to spec testing (where we do not have image yet), integration
testing should be implemented as well. This layer will run tests on environment
with running images, e.g. it should ensure that all applications inside
containers are configured correctly:

  * application is listening on the port according to the configuration;

  * if TLS/SSL is configured, application correctly negotiates secured
    connections;

  * application correctly loads configuration and applies it (exact
    configuration management is out of scope of this spec though);

  * application replies on API requests according to specification;

  * regression testing, which will guarantee that new changes do not break old
    behavior;

  * application is able to work in k8s environment according to its pod
    definition;

Documentation Impact
====================

Jenkins workflow should be described in details after all specs are finalized
and merged, and resulting system is implemented.

References
==========

.. _DockerSpec: https://github.com/docker/docker/blob/master/image/spec/v1.md

.. _ACI: https://github.com/appc/spec/blob/master/spec/aci.md

.. _OCI: https://github.com/opencontainers/specs

.. _runC: https://github.com/opencontainers/runc

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Mitaka
     - Introduced
