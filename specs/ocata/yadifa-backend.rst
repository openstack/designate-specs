..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode

..

=========================================
Integrate yadifa dns backend to designate
=========================================

https://blueprints.launchpad.net/designate/+spec/yadifa-dns-backend



Problem description
===================

There are users or env that may want to use YADIFA as backend which is not supported by designate
as of now.

Proposed change
===============

YADIFA is a lightweight authoritative Name Server with DNSSEC capabilities.
It has a simple configuration syntax and can handle more queries per second while
maintaining one of the lowest memory footprints in the industry.
YADIFA also has one of the fastest zone file load times ever recorded on a name server.

Functionality:
    . authoritative name server
    . DNS UPDATE
    . DNS NOTIFY
    . AXFR
    . IXFR
    . full featured client (yadifa), which can be used to control the server
    . key management, including a tool to generate dnssec keys
    . multi-master support
    . support for other network models
    . detect and configure hyperthreading
    . support for openssl 1.1.0 API

Use Cases:
    (1) Alternative for BIND and NSD for public TLD slave.
        - Clean Implementation
        - High query rate and portable.
        - RFC compliant:
        -  . Authoritative
        -  . DNSSEC support
        -  . AXFR/IXFR (master and slave)

    (2) Dynamic updates
        - Dynamic updated(including continuous signing)
        - Generic, extensible storage backend.

    (3) Zone management
        - Dynamic zone provisioning.

    (4) Recursive Nameserver
        - Recursion
        - Validation


API Changes
-----------
N/A

Central Changes
---------------
N/A

Storage Changes
---------------
N/A

Other Changes
-------------
Add driver ‘impl_yadifa’ to designate.backends using v2 api .
Authentication is done via the keystoneclient auth / session using the provided
credentials in the backend options.

Add designate devstack plugin 'backend-yadifa' to configure yadifa as backend
while installing designate using devstack. Also add docs related to installation
and configuration of yadifa dns backend.

Alternatives
------------
N/A

Implementation
--------------
N/A

Assignee(s)
-----------

Primary assignee:
  Sonu kumar https://launchpad.net/~sonu-bhumca11

IRC Nick Name:
  sonuk

Milestones
----------
Target Milestone for completion:

Work Items
----------
    . Create YADIFA backend
    . Create YADIFA backend docs
    . Create Devstack backend plugin for YADIFA
    . Create experimental gate job
