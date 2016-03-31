=====================================
Docker layers & versions
=====================================

We need to define a way to layer docker images and how to tag them.
Right now we decided what we should be focused on manageability of the layers,
not on the size.

Problem description
===================

Layers
------
Each Docker image references a list of read-only layers that represent
filesystem differences. Layers are stacked on top of each other to form a
the base for a container’s root filesystem.

When you create a new container, you add a new, thin, writable layer on top of
the underlying stack. This layer is often called the “container layer”. All
changes made to the running container - such as writing new files, modifying
existing files, and deleting files - are written to this thin writable
container layer.

More details `here <https://docs.docker.com/engine/userguide/storagedriver/
imagesandcontainers/>`__

So, our goal it to create good layering system which will allow us to re-use
same(common) layers for most containers. This allows us to save space and fix
common problems for all containers in one place.

Versions
--------
The container itself should be considered as an immutable package of some kind.
And so we need to mark them with some versions, to be able to understand what
code this container contains.

Use Cases
---------

Both layers and versions are base objects for creating and running containers,
so without this, we can't even create the container, so this will affect every
user involved.

Proposed change
===============

Layers
------
My proposal architecture for a layering is based on Nextgen one, with few
differences:

======================    ====================================
Container type            Description
======================    ====================================
base                      Linux OS base
tools                     Common Linux packages and DSAs
common-libs               Common Python and modules
dependencies-[service]    All dependencies for a Service
[service]                 Component executable code only
======================    ====================================

So if we take Keystone as an example:

::


    debian
    debian-mos
    debian-mos-ms
    debian-mos-ms-keystone-base
    debian-mos-ms-keystone

.. Note:: "ms" stands for "micro services"

And if we take a mysql as an example:

::


   debian
   debian-mos
   debian-mos-mysql

This layering system has several benefits:

- We could edit each type of layer independently from each other.
- We keep the good balance between manageability and size of the layers.
- Each layer serves to one simple purpose which is easy to understand.

Differences from the `NG document <https://docs.google.com/document/d/
1rtuINpfTvUFc1lJLuMK5LIzGKWwdh8uRUTvOLKnXuPM>`_:

- NG document proposes more complex layering for OpenStack services, like
  creating that kind of layers

::


    nova-base
    nova-api-base
    nova-api

Versions
--------

Full image name in docker constructed from 3 objects:

1. Registry address
2. Image name
3. Image tag

So it could look like:
`172.20.8.52:6555/kolla-team/centos-binary-base:latest`

=============================  =====================
172.20.8.52:6555               Registry address
kolla-team                     Organisation\\username
centos-binary-base             Image name
latest                         Image tag
=============================  =====================

Organisation\\username part is a bit confusing. It makes sense in terms of
private repository, but if the repository is public, it's more like a "group"
in which images are contained.

So, my proposal is to use this Organisation\\username part as a group. So it
will be constructed from this parts:

- `registry-addr`
- `<os_release-version>-<mos-version>-<additional-version>`
- `image-name`
- `<major-version>.<minor-version>-p<patch-version>`

For keystone container it will be:

`1.1.1.1:5000/debian8.3-mos9.0-stable/debian-mos-ms-keystone:1.0-p5`

=======================  =====================
1.1.1.1:5000             Registry address
debian8.3-mos9.0-stable  Image group
debian-mos-ms-keystone   Image name
1.0                      Major & minor version
p5                       Patch level 5
=======================  =====================

Version for CI builds
---------------------

For the CI builds we need to add something uniq to version. When developers
make changes that trigger rebuilding of container image(s) in CI (e.g. change
in keystone source code) we need to be able to match the resulting image with
these changes (e.g. to run deployment tests using this exact set of images).
To achieve this we propose to add Gerrit "change number" (GCR) to the version
string.

Unlike git commit hash that is only unique within 1 project, GCR is unique
within Gerrit instance that can host multiple projects and is also
human-readable (and stores useful information, e.g. it's trivial to build
Gerrit URL from GCR and review related change).
To make this versioning even more granular we can additionally attach Gerrit
"patchset number" (GPR) to distinquish multiple builds for the same CR.

Examples:

- `debian-mos-ms-keystone:1.0-p5-19026` (just the GCR)
- `debian-mos-ms-keystone:1.0-p5-19026_2` (GCR_GPR)

Alternatives
------------

Currently, we have 2 alternatives for layering:

1. **Kolla layering**

Pros:

- Compatibility with upstream.
- Ability to re-use some of their work.

Cons:

- Kolla layers a bit different from our layering vision here at Mirantis. For
  example we want to use additional layer for different usefull tools, which is
  separated from the base layer and the common libs layer.
- Could be really hard or impossible to convince community to change something
  in layering if we will need to.

2. **NG document proposal**

Pros:

- A bit more fine tuned layering system.

Cons:

- I think this will lead to confusion and not really help us to solve any
  problems, since it doesn't really help us to maintain size\managability
  balance, since this layers are tiny and used only for one type of service
  right now.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  `Proskurin Kirill <https://launchpad.net/~kproskurin>`__

Other contributors:
  `Proskurin Kirill <https://launchpad.net/~kproskurin>`__
  `Zawadzki Marek <https://launchpad.net/~mzawadzki-f>`__

Work Items
----------

None


Dependencies
============

None

Testing
=======

None

Documentation Impact
====================

This policy needs to be added to some kind of future documentation about
container creation.

References
==========

None

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Mitaka
     - Introduced
