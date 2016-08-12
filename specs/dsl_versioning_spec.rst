==============
DSL Versioning
==============

The goal of this spec is creating versioning mechanism for DSL


Problem description
===================

The problem is that our DSL may be changed. Some modifications are
backward compatible, some of them are not. And we may have situation when
service file written for old DSL version will be deployed using new fuel-ccp
tool which does not support that old version of DSL.

Use Cases
---------

#. User tries to deploy services with newer version than fuel-ccp DSL version,
   services files are incompatible with fuel-ccp tool.
#. User tries to deploy services with older or equal version than fuel-ccp DSL
   version, services files are compatible with fuel-ccp tool
#. User tries to deploy services with older version than fuel-ccp DSL
   version, services files are incompatible with fuel-ccp tool

Proposed change
===============

#. Introduce semver-based DSL versioning in fuel-ccp tool. Version is a number
   MAJOR.MINOR.PATCH. This version will be defined somewhere in
   `fuel-ccp repo`. Each service definition will contain minimal DSL
   version that required for successful deploy.
   You should increment:
   #. MAJOR version when you make incompatible changes in DSL;
   #. MINOR version when you make backwards-compatible changes in DSL;
   #. PATCH version when you make fixes that do not change DSL, but affect
      processing flow (will be rarely used).
#  Check version compatibility during "fuel-ccp deploy" call according next
   rules:
   #. if fuel-ccp's DSL version is less than service's DSL version -
      they are incompatible - error will be printed, deployment will be
      aborted;
   #. if MAJOR part of these versions are different - they are incompatible -
      error will be printed, deployment will be aborted;
   #. otherwise they are compatible and deploy should be continued.


Alternatives
------------

#. Leave as is, without versioning
#. Use some existing solution like https://www.python.org/dev/peps/pep-0440/
#. Invent some another versioning approach

Implementation
==============
#. Add versions to all service's repos
#. Add version to fuel-ccp repo
#. Add logic to fuel-ccp deploy for handling version validation

Assignee(s)
-----------
apavlov

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
