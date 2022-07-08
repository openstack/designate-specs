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

======================
 Catalog Zones
======================

 DNS Catalog Zones is a method in which a catalog of zones is
 represented as a regular DNS zone, and transferred using DNS zone
 transfers. As zones are added to or removed from the catalog zone, the
 changes are propagated to the secondary nameservers in the normal way.
 The secondary nameservers then add/remove/modify the zones they serve
 in accordance with the changes to the zone.

(https://zones.cat/)


Problem description
===================

Currently Designate does not provide a list of zones for secondary
nameservers to consume and to be provisioned from. Rather Designate
manually creates, modifies and deleted every zone one by one via a
call to a management tooling like BIND's `rndc`. This mechanism has
scaling issues for large number of domains being added and maintained
on secondary server and a slow convergence in case of new servers being
added to the server pool.


Proposed change
===============
Support for providing `catalog zones` should be added to Designate.

In essence `catalog zones` provide easy provisioning capabilities of
lists of zones on secondary servers via AXFR from a special zone,
the `catalog`.

The below links explain the catalog zone feature, use cases and
data model:

* https://zones.cat/
* https://kb.isc.org/docs/aa-01401
* https://datatracker.ietf.org/doc/draft-ietf-dnsop-dns-catalog-zones/


There already is support in BIND9, Knot and other major DNS servers like
PowerDNS or NSD to consume `catalog zones`, all of them are backends
Designate supports:

* BIND (>=9.11) https://bind9.readthedocs.io/en/latest/chapter6.html#catalog-zones
* Knot (>=3.0) https://www.knot-dns.cz/docs/3.0/html/configuration.html#catalog-zones
* NSD https://github.com/IETF-Hackathon/NSDCatZ
* PowerDNS https://github.com/PowerDNS/powercatz/

Apart from improving on performance and resilience of the provisioning
in general, supporting `catalog zones` also enables Designate to support
other backend implementationsas secondary DNS servers without the
requirement for dedicated drivers.


API Changes
-----------

None


Central Changes
---------------

1. The Pool object (and maybe adapters) will need to updated to include the new
   pool settings required for creating a catalog zone. See the `Other Changes`_
   section for details of these settings.
2. Central must be updated to increment the pool catalog zone serial number
   when a zone is created, updated, or deleted in the associated pool. It must
   also trigger a NOTIFY to the nameservers in the affected pool.
3. Central and/or the workers must be updated to not directly create the zones
   on the backend DNS servers if the pool has a catalog zone configured.


Storage Changes
---------------

1. The zone database schema will need to be updated to add a new "type" of
   CATALOG to indicate that this zone is a catalog zone.
2. The existing zone "version" field can be used to track the catalog zone
   "Schema Version" which is required to be present in a TXT record for the
   zone. As of this spec, the version must be "2".
3. The catalog zone will not have an associated tenant_id.
4. A new storage query will need to be created that merges the zone SOA and NS
   records with a the list of zone PTR records for the catalog.

   a. The zone records returned for the catalog will use the zone ID as the
      "unique ID" for the zone PTR record in the catalog.

5. Storage create_pool and update_pool will need to be updated to create the
   catalog zone for the pool when the proper settings are included in the
   pools.yaml. See the `Other Changes`_ section for details of these settings.
6. Storage create_pool and update_pool will need to be updated to create or
   update the catalog zone TSIG key record in the database when one is
   provided in the pools.yaml "catalog_zone_tsig_key" setting.
7. Storage create_pool and update_pool will need to be updated to capture the
   catalog zone attributes in the existing pool_attributes table. See the
   `Other Changes`_ section for details of these settings.


Other Changes
-------------

1. The pool definition will next to be extended to add an optional section
   "catalog_zone".

   a. A "catalog_zone_fqdn" key/value will be added below the "catalog_zone"
      parent. This will be a required field if "catalog_zone" is specified. It
      will be validated to ensure it is a valid Fully Qualified Domain Name
      (fqdn).
   b. A "catalog_zone_refresh" key/value will be added below the "catalog_zone"
      parent. This is an optional setting that will use a default of 60
      (seconds) if the parameter is not specified. This value will be validated
      to be in the range of 0 to 2147483647 as with other zones.
   c. A "catalog_zone_tsig_key" key/value will be added below the "catalog_zone"
      parent. This setting is optional, but will be highly recommended in the
      documentation for this feature. It will be validated the same way TSIG
      keys are validated in the API.
   d. A "catalog_zone_tsig_algorithm" key/value will be added below the
      "catalog_zone" parent. This setting must be specified if a
      "catalog_zone_tsig_key" is specified. It will be validated the same way
      the TSIG algorithm is validated in the API.

2. Documentation must be created for setting up catalog zones.

   a. It must include a security warning and guide to adding a TSIG key to the
      catalog zone.

3. When the catalog zone is created by a pools.yaml pool creation, only one
   NS record will be created for the zone. It must contain "invalid." as the
   NSDNAME as recommended in the catalog zone specification.
4. The catalog zone SOA record fields: RETRY and EXPIRE will be defined the
   same way other Designate zones are defined.
5. Tests must be updated to cover this new capability.

   a. This should include tests that validate that non-admin users cannot see
      the catalog zones.

6. A release note must be created for the feature.

Other Notes
-----------

1. This spec does not include creating "masters" records in the catalog zone.
   This extension does not seem to be finalized at this time and it does not
   currently support alternate port numbers. This could be added at a future
   time.
2. This spec does not include allowing backend drivers to configure the catalog
   zones on the backend DNS servers. This could be added at a future time.
3. The catalog zone TSIG key is included in the pools.yaml as opposed to
   created via the API to ensure the zone is protected as soon as possible and
   to facilitate future patches that may create the catalog zone consumer on
   the backend DNS servers automatically.


Alternatives
------------

The primary alternative to implementing "catalog zones" is the current
mechanism used by the Designate worker to manage zones on the backend name
servers.
Examples of this are: for BIND, rndc zone management; for PowerDNS, the
HTTP API.

Providing the means to allow backends to pull a list of their zones
first of all provides an alternative or addition to the explicit
configuration of backends by the worker and their drivers.

While there is the Designate API available to fetch zone data for other
means of provisioning backends, the catalog zone provides a common and
agreed interface to make this data available via AXFR and to various DNS
servers.

There is no real alterative to providing a catalog zone, as it is a
distinct data model and protocol to be then provided by Designate.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Christian Rohmann https://launchpad.net/~christian-rohmann

IRC Nick Name:
  crohmann


Milestones
----------


Work Items
----------


Dependencies
============


Upgrade Implications
====================
