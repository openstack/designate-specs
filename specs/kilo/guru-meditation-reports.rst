..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode

================================================
Introduce Guru Meditation Reports into Designate
================================================

https://blueprints.launchpad.net/designate/+spec/guru-meditation-reports

Problem description
===================

Guru Meditation Reports enables Designate service to print a lot of detail
debug information about service running thread, configuration and greenlet
stack straces when receiving USR1 signal.

Add Guru Meditation Reports into Designate will help us debugging problems
easier especially when dealing with bugs root in deadlocks between eventlet
greenthreads.

Guru Meditation Reports is alreadly implemented in oslo-incubator and used in
Nova.

Proposed change
===============

Pick Guru Meditation Reports implementation from oslo-incubator and add the
USR1 signal handler.

All Designate services will support Guru Meditation Reports, including:

* designate-central
* designate-api
* designate-mdns
* designate-agent
* designate-pool-manager
* designate-sink
* designate-manage

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

Designate service processes will receive USR1 signal and print Guru Meditation
Reports.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  stanzgy

Milestones
----------

Target Milestone for completion:
  Kilo-3

Work Items
----------

Work items are listed in Proposed change section.

Documentation Impact
====================

Describe how to generate and view a Guru Meditation Reports.

References
==========

* GMR wiki: https://wiki.openstack.org/wiki/GuruMeditationReport
* GMR in Nova: http://docs.openstack.org/developer/nova/devref/gmr.html
* Nova GMR User Manual: http://docs.openstack.org/admin-guide-cloud/content/section_compute-GuruMed-reports.html
