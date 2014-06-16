..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. The filename in the git
  repository should match the launchpad URL, for example a URL of
  https://blueprints.launchpad.net/designate/+spec/awesome-thing should be named
  awesome-thing.rst .  Please do not delete any of the sections in this
  template.  If you have nothing to say for a whole section, just write: None
  For help with syntax, see http://sphinx-doc.org/rest.html
  To test out your formatting, see http://www.tele3.cz/jbar/rest/rest.html

=============================
 Server Pools
=============================

https://blueprints.launchpad.net/designate/+spec/server-pools

Problem description
===================

Server Pools are required for a few different scenarios:

Private Pools
-------------

This allows users to have 'private' DNS servers. These servers would typically
allow non standard TLDs (.dev , .local etc), and may not have the same level of
blacklist restrictions. They would be aimed at people with Neutron Networking,
and VPC style set ups, where access to the DNS server would come from trusted
networks (E.G. in-cloud - owned instances, and onsite resources connect by VPN)

This would allow customers to set DNS entries for internal servers, on domains
that would not be available on the public pools, and have them accessible by
internal users

Distribution
------------

Having multiple public pools with the same capabilities, would allow the
scheduler to distribute zones across multiple infrastructures.

As part of the pools change we are also changing how the servers API works,
to allow for more fine grained control of servers, their capabilities and
backends

Features / Premium Systems
--------------------------

By using scheduler hints, we can mark different pools as having different
capabilities - such as GeoIP / Round Robin DNS / Anycast.

This allows operators to run different DNS infrastructures as required.
For example this allow users to have some zones on pools with GeoIP, and pay a
premium for this feature, while having the rest of their zones on a cheaper
'standard' tier.

Proposed change
===============

Terminology
-----------

The terms for different parts of pool manager can be confusing, so the following list should be use when referring to components of Server Pools

+-------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-----------------------+
| Term                    | Meaning                                                                                                                                                                                                                                    | Also known as         |
+=========================+============================================================================================================================================================================================================================================+=======================+
| Server Pool             | group of DNS servers                                                                                                                                                                                                                       |                       |
+-------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-----------------------+
| DNS Zone                | DNS Zone (aka subzone.domain.com.) more commonly referred to as a domain                                                                                                                                                                   |                       |
+-------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-----------------------+
| Zone                    | See DNS Zone                                                                                                                                                                                                                               |                       |
+-------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-----------------------+
| Presentation DNS Server | Customer Facing DNS server (used by people to resolve zones owned by designate)                                                                                                                                                            |                       |
+-------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-----------------------+
| Name Server             | A FQDN (or IP, but usually a FQDN) that is used to populate the NS Records of a Designate Managed Zone. Each Pool will have a set of Name Servers, which users then delegate to from their registrar                                       | nameserver, ns record |
+-------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-----------------------+
| Backend                 | A driver that allows Designate to control a particular type of DNS Server software (BIND, PowerDNS etc)                                                                                                                                    |                       |
+-------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-----------------------+
| MiniDNS                 | A designate service that is used to send notifies to servers that need to be updated (with new information about Zones), and serves AFXR requests with the new information. There is usually a shared set of MiniDNS servers for all pools |                       |
+-------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-----------------------+
| FQDN                    | Fully Qualified Domain Name - a DNS entry that has both a hostname section (i.e. ns1. ) and a zone section (including the trailing '.') (i.e. example.com. )                                                                               |                       |
+-------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-----------------------+


This will allow us to split domains between different groups of servers.
This will have a fairly massive impact across the whole system - and will
require changes to nearly every part of Designate.

We will add 2 major pieces to Designate:

Pool Manager
------------

This is a service that will be responsible for notifying DNS servers that
changes have occured. This will take the backends implemented for MiniDNS, and
load them.

It will take the place of the miniDNS notifier section, and will be responsible
for checking async operations.

Scheduler
---------

This service will be resonsible for assigning zones to pools.

The scheduler will assign a zone to a pool based on hints in the request, or if
there is no hints in the request, any pool that has capacity.

Initially for pools there a single 'default pool' defined in the config, and overrides will be allowed via the 'hints' section of a zone create API request


Flow of information with Server Pools
-------------------------------------

Single Frames available [1]_

`Overview Image`_

.. _Overview Image: https://wiki.openstack.org/w/images/a/a7/Designate-MiniDNS-Pools.gif

https://wiki.openstack.org/w/images/a/a7/Designate-MiniDNS-Pools.gif
[2]_

We have discusses this in person twice now - once in the Icehouse mid-cycle,
and once in Atlanta at the design summit.

`Etherpad`_

.. _Etherpad: https://etherpad.openstack.org/p/juno-design-summit-designate-session-2





Milestones
----------

Target Milestone for completion:
  Kilo-2 [3]_

Work Items
----------

+------------------------------+------------------------------------------------------------------+----------+-----------+-------+
| Work Item                    | Assignee                                                         | Priority | Milestone | Notes |
+==============================+==================================================================+==========+===========+=======+
| server-pools-storage         | https://launchpad.net/~rjrjr / https://launchpad.net/~darshan104 | High     |           |       |
+------------------------------+------------------------------------------------------------------+----------+-----------+-------+
| server-pool-manager          | https://launchpad.net/~rjrjr / https://launchpad.net/~darshan104 | High     |           |       |
+------------------------------+------------------------------------------------------------------+----------+-----------+-------+
| server-pools-minidns-support | https://launchpad.net/~vinod-mang                                | High     |           |       |
+------------------------------+------------------------------------------------------------------+----------+-----------+-------+
| server-pools-api             | https://launchpad.net/~betsy-luzader                             | High     |           |       |
+------------------------------+------------------------------------------------------------------+----------+-----------+-------+




Dependencies
============

* `MiniDNS`_


.. _MiniDNS: https://wiki.openstack.org/wiki/Designate/Blueprints/MiniDNS

Footnotes
=========

.. [1] https://imgur.com/a/CawLd
.. [2] https://wiki.openstack.org/w/images/a/a7/Designate-MiniDNS-Pools.gif
.. [3] https://wiki.openstack.org/wiki/Juno_Release_Schedule
