..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode

..

==========================
Zone and Record TotalCount
==========================

https://blueprints.launchpad.net/designate/+spec/zone-and-record-totalcount

Problem description
===================

When a user makes a request for a list of zones or records, provide the
total count of zones or records that match the given query. The count will
ideally be provided when a user makes a request for something like /zones
or /records (to limit the need to make multiple queries).


Proposed change
===============

API Changes
-----------

This change would not add any new endpoints, but it would modify the data
presented by some existing endpoints.

The following examples demonstrate the total_entries value for a generic
collection of "resources". After implementation, this feature will be
supported by collections of zones and recordsets.

GET /v2/resources
^^^^^^^^^^^^^^^^^

**Example Request:**

.. sourcecode:: http

    GET /v2/resources HTTP/1.1
    Accept: application/json
    Content-Type: application/json

**Example Response:**

.. sourcecode:: http

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "resources": [{
        "id": "fdd7b0dc-52a3-491e-829f-41d18e1d3ada",
        "created_at": "2014-06-23T18:39:32.000000",
        "....": "...."
      }, {
        "id": "a86dba58-0043-4cc6-a1bb-69d5e86f3ca3",
        "created_at": "2014-07-08T20:28:19.000000",
        "....": "...."
      }, {
        "id": "460f7531-e381-4773-aff3-06a12fad096d",
        "created_at": "2014-06-04t19:09:17.000000",
        "....": "...."
      }, {
        "id": "40ced622-fc70-498d-9f28-3d3021b19685",
        "created_at": "2014-07-08T16:47:32.000000",
        "....": "...."
      }],
      "links": {
        "self": "https://dns.provider.com/v2/resources?sort_key=id&sort_dir=desc"
      },
      "meta": {
        "total_entries": 4
      }
    }

Central Changes
---------------

A PagedListObjectMixin class will be added that will support the metadata
associated with list pagination. This class will be added as a superclass
of the ZonesList and RecordsList classes.

Storage Changes
---------------

Any calls to find_domains will also internally call count_domains and add
this count to the DomainsList object that is returned.

Any calls to find_records will also internally call count_records and add
this count to the RecordsList object that is returned.

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

Primary assignee: jordan-cazamias

Milestones
----------

Target Milestone for completion:
  Juno-2

Work Items
----------

* Agree on spec for API formatting
* Implement changes

Dependencies
============

https://review.openstack.org/#/c/105021/

