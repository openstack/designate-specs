..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode

=========================
 Designate Zone Pool Move
=========================

https://blueprints.launchpad.net/designate/+spec/zone-move

Zone move is an important feature that possibly eliminates the problem observed
with zone export-import.


Problem description
===================

Assume, zone is currently located in pool A, but has to move to pool B. (for example
pool B is another, new DNS provider).

Currently, Designate only allows you to create zone export (will be a Bind-format
single file with all the records of the zone) and zone import. So, a zone can be
exported from one pool; deleted from pool A; and imported into another pool B.

* The problem is that the zone has to be deleted in this process and imported.
* Large zone imports (20-30k records) can take a couple of hours as Designate
  creates all records one by one.

Therefore, it is problematic to move a zone between the pools/backends without any
hacks. At the point when the zone is deleted, but not imported, any DNS queries to
Designate-MDNS from the backend will not work.


Proposed change
===============

Introduce a new admin-only command in designate ``zone pool move``. The command works
as below:

**openstack zone pool move zone_id_or_name --pool_id=AN_ID_OF_POOL_B**

* The designate service should update zone pool_id property similar to other zone
  properties. The updated pool_id will be reflected in the designate DB. If the
  command is invoked without pool_id paramter, the designate reschedule the zone
  and move it to a different pool if appropriate. For example, user can move
  their zone up to a higher tier of service by updating an attribute and then
  calling move will cause attribute scheduler to select appropriate pool.
* After DB update, designate will create copy of zone (can also be called as clone)
  except it should not create new db entry for the clone zone. The clone zone will
  be created on target pool backend servers i.e. pool B.
* The zone transfer(AXFR/IXFR) will happen and the zone on pool B gets synced with
  the designate DB.
* At this point zone still exists in pool A. It can be removed after the administrator
  has changed the settings manually at domain registrar. This is a manual process.

The above proposed change eliminates the zone export as well as zone delete steps.
Thus, speed up the import of large zone to the target pool.


API Changes
-----------

The zone move will be triggered via the V2 API.

**Example zone pool move request:**

.. code-block:: http

        POST http://192.168.1.47/dns/v2/zones/52f42b9e-c48b-43e2-af01-180e8ed33cd0/tasks/pool_move HTTP/1.1
        Host: 127.0.0.1:9001
        Accept: application/json
        Content-Type: application/json

        {
          "pool_id": "794ccc2c-d751-44fe-b57f-8894c9f5c842"
        }


Central Changes
---------------

Central will update the pool_id of the zone for which move is requested. The updated
pool_id will be written to DB. Central will then clone the zone and then create new clone
zone in target pool backend servers.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Kiran Pawar (kpdev in Gerrit)


Dependencies
============

None
