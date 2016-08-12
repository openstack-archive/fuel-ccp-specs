=====================================================
Application Definition Language (ADL) Versioning Tool
=====================================================

The goal of this spec is creating versioning mechanism for ADL


Problem description
===================

The problem is that our ADL may be changed. Some modifications are
backward compatible, some of them are not. And we may have situation when
service file written for old ADL version will be deployed using new fuel-ccp
tool which does not support that old version of ADL.

Use Cases
---------

#. User trys to deploy services with newer version then fuel-ccp ADL version,
   services files are incompatible with fuel-ccp tool.
#. User try to deploy services with older or equal version then fuel-ccp ADL
   version, services files are compatible with fuel-ccp tool
#. User try to deploy services with older version then fuel-ccp ADL
   version, services files are incompatible with fuel-ccp tool

Proposed change
===============

#. Introduce two part ADL versioning in fuel-ccp tool. If major version is
   changed this means that change is not backward compatible
#. Keep ADL version for each services repository (inside service.yaml) to
   store version of ADL used for writing services of current repository.
#  Check version compatibility during "fuel-ccp deploy" call according next
   rules:
   #. if fuel-ccp's ADL version is less then service's ADL version -
      they are incompatible
   #. if major part of these versions are different - they are incompatible
   #. otherwise they are compatible and deploy should be continued.


Alternatives
------------

Leave as is of invent some another versioning approach

Implementation
==============
#. Add versions to all service's repos
#. Add version to fuel-ccp repo
#. Add logic to fuel-ccp deploy for handling vesion validation

Assignee(s)
-----------
dukhlov

Work Items
----------
#. implement patch for fuel-ccp
#. implement patches for service's repos

Dependencies
============
None


Testing
=======

Unit testing for fuel-ccp version validation logic


Documentation Impact
====================

Describe version bump rules

References
==========

None

History
=======

None
