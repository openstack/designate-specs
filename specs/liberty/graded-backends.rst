..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode

================
 Graded Backends
================

https://blueprints.launchpad.net/designate/+spec/graded-backends

Problem description
===================

Currently we have quite a few backends, that are not tested, or actively maintained.
We should be informing users of what backends are most actively maintained and tested

Proposed change
===============

A graduated level of grading for backends:

+--------------------+--------------------+----------------------------------------------------------------------------------------------------------------------------------+
| Grade              | Variations         | Description                                                                                                                      |
+====================+====================+==================================================================================================================================+
| Integrated         | None               | Tested on every commit by the OpenStack CI Infrastructure, and maintained by designate developers as a reference backend         |
+--------------------+--------------------+----------------------------------------------------------------------------------------------------------------------------------+
| Master Compatible  | In Tree / External | Tested on every commit by 3rd party testing, and has a person or group dedicated to maintaining compatibility on a regular basis |
+--------------------+--------------------+----------------------------------------------------------------------------------------------------------------------------------+
| Release Compatible | In Tree / External | Not necessarily tested on every commit, but has a maintainer committed to ensuring compatibility for each release                |
+--------------------+--------------------+----------------------------------------------------------------------------------------------------------------------------------+
| Untested           | In Tree / External | All other backends in the designate repository                                                                                   |
+--------------------+--------------------+----------------------------------------------------------------------------------------------------------------------------------+
| Failing            | In Tree / External | Backends that were previously "\* Compatible", but tests are now failing on a regular basis.                                     |
+--------------------+--------------------+----------------------------------------------------------------------------------------------------------------------------------+
| Known Broken       | In Tree / External | Backends that do not work, and have been broken with no sign of any fixes                                                        |
+--------------------+--------------------+----------------------------------------------------------------------------------------------------------------------------------+


The current backends would be currently fall into the following pattern:

+------------------+------------------------------+-------+
| Backend          | Grade                        | Notes |
+==================+==============================+=======+
| PowerDNS + MySQL | Integrated                   |       |
+------------------+------------------------------+-------+
| Bind             | Integrated                   |       |
+------------------+------------------------------+-------+
| Akamai           | Release Compatible - In Tree |       |
+------------------+------------------------------+-------+
| DynECT           | Release Compatible - In Tree |       |
+------------------+------------------------------+-------+
| Agent            | Untested - In Tree           |       |
+------------------+------------------------------+-------+
| PowerDNS + pgSQL | Untested - In Tree           |       |
+------------------+------------------------------+-------+
| Microsoft DNS    | Untested - External          |       |
+------------------+------------------------------+-------+

This will also include the creation of a support matrix like `Nova`_

.. _Nova: http://docs.openstack.org/developer/nova/support-matrix.html

This info should be maintained along with the list of current driver maintainers
responsible for the "Non Integrated" backends. The upkeep of this list will fall on the PTL or his/her delegate

Should a backend's grade be in dispute, it falls
on the current project PTL to make the final decision
after listening to all sides concerns.


Alternatives
------------

Continue on as is

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  grahamhayes

Milestones
----------

Target Milestone for completion:
  Liberty-1

Work Items
----------



Dependencies
============

- None
