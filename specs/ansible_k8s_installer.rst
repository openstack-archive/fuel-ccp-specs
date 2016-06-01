=====================================
Ansible-based Kubernetes Installer
=====================================

https://mirantis.jira.com/browse/MCP-614

We have decided to pursue deployment of Kubernetes with an Ansible-based
tool that scales well.

Ansible has a large community behind it and is simple to interact with. This
document proposes to use an established Kubernetes deployment utility, called
Kargo, as the basis of the installer.

Problem description
===================

This document highlights what is important for the installer to have and its
expected outcomes.

This document does not define the underlay which will provision bare metal
hosts (or VMs). It will merely set assumptions for how a prepared host should
be presented.

All installers require either input data or discovery capabilities.
In this case, there will be an underlay system to provision and configure
networking for k8s nodes. The underlay needs to expose metadata to a deployer
in order to proceed with a Kubernetes installation that is adapted for the
given environment.

Use Cases
---------

The installer will be used at all levels, by developers and deployers. CI
testing of Kubernetes and related components will run this installer tool to
test for regressions. QA will use this tool to ensure full deployment scenarios
are bug-free.

All levels of deployment should be the resulting case, but full HA is not in
scope of this spec implementation. Eventually this will apply to live
production scenarios and testing for scale/load/longevity.

It needs to work cleanly and avoid fault scenarios.

Kargo itself can be run with Vagrant and kargo-cli will deploy a heat stack on
OpenStack and also to AWS/GCE environments. Vagrant deployment from a given
metadata YAML, however, is out of scope for this spec.

Proposed change
===============

Kargo has been selected as an initial deployment utility. It requires very
little input for the Ansible inventory at this stage, although our
requirements will likely become more complex in the future.

Kargo is a community-supported, Apache-licensed, open source tool with Ansible
playbooks for deploying Kubernetes. It supports a variety of operating systems
and can source its Kubernetes components from an arbitrary source (but only in
tarball format).

A simple Kargo deploy wrapper will be put in place for installing k8s on top of
Ubuntu Xenial (and Debian) nodes. The initial tool will be a shell script that
performs simple SSH commands to prepare hosts and call Kargo deployment.

Provisioned nodes should contain the following conditions:

* Nodes contain at least 1 configured network interface.
* If calico requires a separate bridge/interface, it will be set up by the
  underlay.
* Nodes have /etc/hosts configured and corresponds to provided inventory data
  (Kargo will write if it doesn't exist already)
* SSH access is available via key or password
* Effective partitioning on the system to segregate Docker and logs from root
  and /var

Additionally, the node performing deployment must be provided a YAML or set of
ENV variables for node IPs and node hostnames must be present to generate
Ansible inventory. Mandatory parameters include:

* Node IPs
* Node hostnames
* Node SSH credentials

Optionally, the provided YAML may contain the following configurable items:

* kube password
* kube network CIDR
* kube network plugin
* kube proxy type
* Docker daemon options
* upstream DNS list
* DNS domain
* component versions
* binary download URLs (and checksums) for kubernetes, etcd, calico

.. note:: See http://bit.ly/1Y4nhPT for all download options.



Alternatives
------------

We could invest in sophisticated SSL setups, external load balancers, or
customizing which network plugin to use. Additionally, external Ceph
configuration may be necessary. These are out of scope, but should be
considered in the underlay story.

There are smaller Ansible-based Kubernetes installers available, but they have
much smaller communities and typically are geared toward Fedora/CentOS. This
includes the in-tree kubernetes Ansible playbook. We could and probably should
contribute here, but it will have a high initial cost for us.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Matthew Mosesohn <raytrac3r>

Work Items
----------

* Kargo deploy wrapper (initially written in Bash, later in Python and/or
  Ansible)
* Implement Kargo deployment in CI
* Requirements specification for CCP underlay
* Add support for deploying custom k8s/etcd/calico binaries
* Obtain QA acceptance criteria


Dependencies
============

* This project requires an underlay tool, but we will rely on fuel-devops and
  packer-built images until underlay installer is ready.


Testing
=======

While Ansible does not support unit testing, it is possible to include
assertions to test functionality after deployment.

There will be CI coverage for all commits to ensure a "standardized" deployment
works.

Documentation Impact
====================

TBD

References
==========

https://github.com/kubespray/kargo

https://github.com/kubespray/kargo-cli

https://github.com/openstack/fuel-devops

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Iteration 12-1
     - Introduced
