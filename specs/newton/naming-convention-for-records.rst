..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode


==============================
Naming Conventions for Records
==============================

https://blueprints.launchpad.net/designate/+spec/different-format-for-ipv4-and-ipv6

Naming Conventions for records
Right now we have a single format to define FQDN for records created
through designate sink.

Problem description
===================

At present there is only one format we can define for both IPv4 and IPv6 ips.
If a user want to have different naming convention for records created through
Sink service, we do not have a way to do that since there is only one option
in designate.conf where we define the FQDN format.

Proposed change
===============

In designate.conf we should give an option under the Sink configuration where
we can define two different formats for IPv4 and IPv6. Similarly we need to make
appropriate changes in sink code under designate/notification_handler/ for
base.py, nova.py and neutron.py

The  new format options in designate.conf would be "formatv4" and "formatv6".

To help users from older versions, we will still support "format" which will
work for IPv4 addresses. "format" or "formatv4" can be used interchangebaly
but if both of them are defined then "formatv4" will be accepted along with
a Warning message.

SINK Changes
------------

Base.py, nova.py and neutron.py needs to be changed to get the format type from
designate.conf. The new formats that we need to get are formatv4 and formatv6 which will
now be implemented in designate.conf.

API Changes
-----------

None

Central Changes
---------------

None

Storage Changes
---------------

None

Other Changes
-------------

None

Alternatives
------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Kumar Acharya<ma501v>

Milestones
----------

Target Milestone for completion:
  Mitaka-3

Work Items
----------

Define the format for IPV4 and IPv6 FQDN and Implement it in designate.conf
Code changes required in designate

Dependencies
============

formatv4 and formatv6 should be implemented in designate.conf as new format types
