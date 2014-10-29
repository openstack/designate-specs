..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode

..

.. _zone_import_refactor:


====================
Zone Import Refactor
====================

Migrate Zone import code from designate.api.v2.controllers.zones into a new
designate.dnsutils module and move functionality into Central.


Terminology
===========

+----------+---------------------------------------------+
| Term     | Meaning                                     |
+==========+=============================================+
| zone     | DNS Zone (also refered to as a DNS domain)  |
+----------+---------------------------------------------+


Problem description
===================

Currently the Import functionality parses the Zone that is POST'ed in the API
and runs the creation of the Zone with its Records from the API side. There is
no transactional guarantee of a cleanup if the creation of anything fails.

Also there's a given value of having these methods outside of the API, it can
be used in any component outside of the API (for example AXFR support).


Proposed change
===============

We introduce a new module called designate.dnsutils that contains functions
for parsing Zones / Records into dnspython object and dnspython objects to
DesignateObjects.

Code for parsing a Zone file is already present in the
designate.api.v2.controller.ZoneController and can be easily extracted.


Central Changes
---------------

We need a new method that takes a Domain and a RecordSetList object, then
puts this into Designate accordingly.

It should have the ability to either synchronize a existing zone or at a later
point in time a new zone with it's records.


New module dnsutils
-------------------

dnsutils.from_dnspython_zone(dnspython_zone)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Takes a dnspython_zone and creates a Domain object also runs
records_to_recordset_list() on each record in the zone.

+------------------+--------------------+--------------+
| **Parameter**    | **Description**    | **Required** |
+==================+====================+==============+
| *dnspython_zone* | DNS Python zone    | Yes          |
+------------------+--------------------+--------------+

Return value
""""""""""""

Domain with a recordset relation attribute


dnsutils.dnspyrecords_to_recordsetlist(dnspython_records)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Convert a dnspython_zone.nodes structure to a RecordSetList, Recordsets,
RecordList and Records.

+----------------------+---------------------------------+--------------+
| **Parameter**        | **Description**                 | **Required** |
+======================+=================================+==============+
| *dnspython_records*  | DNS Python Record               | Yes          |
+----------------------+---------------------------------+--------------+

Return value
""""""""""""

designate.objects.recordset.RecordSet with a record attribute of
designate.objects.record.RecordList from the given dnspython node / record.


dnsutils.dnspythonrecord_to_recordset(rname, rdataset)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

+-----------------+---------------------------------+--------------+
| **Parameter**   | **Description**                 | **Required** |
+=================+=================================+==============+
| *rname*         | dnspy node name / key           | Yes          |
+-----------------+---------------------------------+--------------+
| *rdataset*      | dnspy rr dataset                | Yes          |
+-----------------+---------------------------------+--------------+

Return value
""""""""""""

designate.objects.recordset.RecordSet with a record attribute of
designate.objects.record.RecordList from the given dnspython node / record.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <https://launchpad.net/~endre-karlson>

Milestones
----------

Target Milestone for completion:
  Kilo-1

Work Items
----------

* Extract code from Zones controller into new module "dnsutils" / new central
  method.
* Change parse_zonefile to call dnsutils.parse_zonefile() and call Central's
  new method with the results of this.