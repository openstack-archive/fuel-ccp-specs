=================================================
Make possible to rebuild only needed images in CI
=================================================

CI should be improved to build only necessary images, because
image building is expensive operation

Problem description
===================
Currently our CI image building works in the next way:
 #. build all images from specified repositories on 'gate' stage
    (current patch's repository and repositories with required base images)
 #. build all images from all repositories on 'post' stage again and push
    then to the docker registry

Here are a few problems which possibly could be improved
 #. For building images on 'gate' stage we have to specify correct set of
    repositories with base images.
 #. We don't use our docker registry for pulling base images
 #. We rebuild and push all image set for each 'post' job

Use Cases
---------
#. Developer sends new patch set for one of docker image project. We need build
   and push only images from target repository. Also for keeping consistency
   we have to re-build and re-push all available child images.
#. Developer sends new patch set to the fuel-ccp tool itself. It shouldn't
   affect images in normal case and we don't need to re-build and re-push all
   images, just ensure that tool works fine

Proposed change
===============
#. Create separate jenkins job for rebuilding all images for registry
   (re)initialization
#. Always fetch all repositories to ability to build image dependency tree
#. Fix fuel-cpp tool to make it able to use pulling from docker registry base
   images to skip its building when we send new patch for some child docker
   image project
#. Specify docker image set to be built for each project to avoid building not
   needed images
#. use keep_image_tree_consistency = True option to re-build all set of child
   images
#. build only some test docker image or just single base image for fuel-cpp
   project's job to ensure that tool is able to build some reference image

Alternatives
------------
#. we may skip using docker registry for pulling base images because
   probably it might cause some problems for repeatable building
#. we may fetch only required subset of the project's git repositories
   but in this case we need to get image dependency tree somehow. So we need
   store it in configuration which will complicate it more or use some extra
   storage like db or probably new git repo with image configuration and fetch
   it first to know what we need to fetch more.

Implementation
==============
Proposed change required fixing fuel-ccp code and project-config jobs

Assignee(s)
-----------

Primary assignee:
   dukhlov

Other contributors:
   None

Work Items
----------

#. fix fuel-ccp tool
#. fix project-config's jobs description

Dependencies
============
None


Testing
=======
We need to test a lot of corner cases because we work with image tree
structure: jobs for root image, for leaf, for medium node.

Also we use external registry and it is reasonable to test what
happens when all images are present in registry, registry is empty,
partial empty.


Documentation Impact
====================
None

References
==========
None


History
=======
None