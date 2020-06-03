..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode

..

=============
Filtering API
=============

Discussion Needed
=================

Update:
-------

For now, wildcards will not be restricted, though limitations may have to be added
due to scaling concerns. Also, to address a bug with wildcard matching
(https://bugs.launchpad.net/designate/+bug/1335902) and to conform to standards,
the wildcard character has been changed from '%' to '*'.

Previous Discussion:
--------------------

http://paste.openstack.org/show/85431/


Problem description
===================

https://blueprints.launchpad.net/designate/+spec/example

Filtering provides the ability to qualify the result set returned by a query to the designate api.
It will ultimately be available on all collections - zones, record sets, rdata of record sets, and pools.

Filtering will be controlled using query parameters which match the name of the attribute being filtered.
It is *not* required that all attributes are available as filter targets, but the majority will be.

Filters are either an exact match or wildcard match, which is specified by the presence of the reserved
wildcard character ('*').

If the filtering request is successful, the resources that pass the filter criteria are returned,
as well as links for retrieving more details.

Pagination of results will take advantage of proposed Designate pagination
(https://blueprints.launchpad.net/designate/+spec/pagination)

Proposed change
===============

Filtering vs. Search: A Clarification
-------------------------------------

NOTE: Filtering and searching are two completely different features and will be addressed separately.
Search involves the ability to compile a list of results from storage, possibly drawing from many
different places (for example, finding all of a tenant's A records with a certain IP address). Filtering
only involves further restricting the standard queries that are offered by the API (for example, /zones,
/zones/{id}/recordsets, etc.).

You can find the wiki for search `here <https://wiki.openstack.org/wiki/Designate/Blueprints/Search_API>`_

API Changes
-----------

* Basic filtering for:

  * Blacklists: pattern
  * Recordsets: name, type, ttl, data
  * TLDs: name
  * Zones: name

* Wildcard search using SQL pattern matching

The examples below demonstrate the types of calls typically made with filtering.  These calls apply
to all of the aforementioned filtering parameters.

+------+----------------------------------------------+----------------------+
| Verb | Resource                                     | Description          |
+======+==============================================+======================+
| GET  | zones/{id}/recordsets/{id}/recordsets?       | From the specified   |
|      |   name=www.example.com                       | recordset, return    |
|      |                                              | any record with a    |
|      |                                              | name that matches    |
|      |                                              | the specified filter |
|      |                                              | (“name”) exactly.    |
+------+----------------------------------------------+----------------------+
| GET  | zones/{id}/recordsets/{id}/recordsets?       | From the specified   |
|      |    name=www.test*                            | recordset, return    |
|      |                                              | any record with a    |
|      |                                              | name that BEGINS     |
|      |                                              | with the specified   |
|      |                                              | “name” attribute.    |
+------+----------------------------------------------+----------------------+
| GET  | zones/{id}/recordsets/{id}/recordsets?       | From the specified   |
|      |    name=*.com                                | recordset, return    |
|      |                                              | any record with a    |
|      |                                              | name that ENDS       |
|      |                                              | with the specified   |
|      |                                              | “name” attribute.    |
+------+----------------------------------------------+----------------------+
| GET  | zones/{id}/recordsets/{id}/recordsets?       | From the specified   |
|      |    name=www.*.com                            | recordset, return    |
|      |                                              | any record with a    |
|      |                                              | name that contains   |
|      |                                              | the specified name   |
|      |                                              | data on both sides   |
|      |                                              | of the wildcard.     |
+------+----------------------------------------------+----------------------+


GET zones/{zone-id}/recordsets?name=example*
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This call is for a privileged user filtering recordsets by name, for one tenant, using wildcard matching.

Request:

.. code-block::

  GET zones/{zone-id}/recordsets?name=example*
  Host: dns.provider.com
  Accept: application/json

Response:

.. code-block::

    {
        "recordsets": [
            {
                "description": null,
                "links": {
                    "self": "http://192.168.33.8:9001/v2/zones/a4e29ed3-d7a4-4e4d-945d-ce64678d3b94/recordsets/948a0233-abdf-434e-a003-c5d682daf0ea"
                },
                "updated_at": null,
                "records": [],
                "ttl": null,
                "id": "948a0233-abdf-434e-a003-c5d682daf0ea",
                "name": "example.com.",
                "zone_id": "a4e29ed3-d7a4-4e4d-945d-ce64678d3b94",
                "created_at": "2014-07-08T20:28:27.000000",
                "priority": [],
                "version": 1,
                "type": "NS"
            },
            {
                "description": null,
                "links": {
                    "self": "http://192.168.33.8:9001/v2/zones/a4e29ed3-d7a4-4e4d-945d-ce64678d3b94/recordsets/7da6119a-8b41-4f09-a2b7-a44ed2c9ebd0"
                },
                "updated_at": null,
                "records": [
                    "mail2.example.org.",
                    "mail.example.org."
                ],
                "ttl": null,
                "id": "7da6119a-8b41-4f09-a2b7-a44ed2c9ebd0",
                "name": "example.org.",
                "zone_id": "a4e29ed3-d7a4-4e4d-945d-ce64678d3b94",
                "created_at": "2014-07-08T20:28:28.000000",
                "priority": [
                    20,
                    10
                ],
                "version": 1,
                "type": "MX"
            }
        ],
        "links": {
            "self": "http://192.168.33.8:9001/v2/zones/a4e29ed3-d7a4-4e4d-945d-ce64678d3b94/recordsets?name=example%2A"
        }
    }


GET zones/{zone-id}/recordsets?name=example.org.
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This call is for customers filtering recordsets by name, using exact matching.

Request:

.. code-block::

    GET zones/{zone-id}/recordsets?name=www.example.org.
    Host: dns.provider.com
    Accept: application/json

Response:

.. code-block::

    {
        "recordsets": [
            {
                "description": null,
                "links": {
                    "self": "http://192.168.33.8:9001/v2/zones/a4e29ed3-d7a4-4e4d-945d-ce64678d3b94/recordsets/7da6119a-8b41-4f09-a2b7-a44ed2c9ebd0"
                },
                "updated_at": null,
                "records": [
                    "mail2.example.org.",
                    "mail.example.org."
                ],
                "ttl": null,
                "id": "7da6119a-8b41-4f09-a2b7-a44ed2c9ebd0",
                "name": "example.org.",
                "zone_id": "a4e29ed3-d7a4-4e4d-945d-ce64678d3b94",
                "created_at": "2014-07-08T20:28:28.000000",
                "priority": [
                    20,
                    10
                ],
                "version": 1,
                "type": "MX"
            }
        ],
        "links": {
            "self": "http://192.168.33.8:9001/v2/zones/a4e29ed3-d7a4-4e4d-945d-ce64678d3b94/recordsets?name=example%2A"
        }
    }


GET zones/{zone-id}/recordsets?data=1.2.3.*
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This call is for customers who wish to find all recordsets in a zone that contains
one or more records with the matching data value (with a wildcard applied).
The data parameter can be used in conjunction with other parameters.

Request:

.. code-block::

    GET zones/{zone-id}/recordsets?data=1.2.3.*
    Host: dns.provider.com
    Accept: application/json

Response:

.. code-block::

    {
        "recordsets": [
            {
                "description": null,
                "links": {
                    "self": "http://192.168.33.8:9001/v2/zones/a4e29ed3-d7a4-4e4d-945d-ce64678d3b94/recordsets/077ce2b4-1d52-4c3f-8c70-07dfddeed5cc"
                },
                "updated_at": null,
                "records": [
                    "1.2.3.4"
                ],
                "ttl": null,
                "id": "077ce2b4-1d52-4c3f-8c70-07dfddeed5cc",
                "name": "dns2.example.com.",
                "zone_id": "a4e29ed3-d7a4-4e4d-945d-ce64678d3b94",
                "created_at": "2014-07-08T20:28:26.000000",
                "priority": [
                    null
                ],
                "version": 1,
                "type": "A"
            },
            {
                "description": null,
                "links": {
                    "self": "http://192.168.33.8:9001/v2/zones/a4e29ed3-d7a4-4e4d-945d-ce64678d3b94/recordsets/758004fe-4f0f-4756-a8f3-ee01ca5db8a2"
                },
                "updated_at": null,
                "records": [
                    "1.2.3.10"
                ],
                "ttl": null,
                "id": "758004fe-4f0f-4756-a8f3-ee01ca5db8a2",
                "name": "mail2.example.com.",
                "zone_id": "a4e29ed3-d7a4-4e4d-945d-ce64678d3b94",
                "created_at": "2014-07-08T20:28:20.000000",
                "priority": [
                    null
                ],
                "version": 1,
                "type": "A"
            }
        ],
        "links": {
            "self": "http://192.168.33.8:9001/v2/zones/a4e29ed3-d7a4-4e4d-945d-ce64678d3b94/recordsets?data=1.2.3.*"
        }
    }

Central Changes
---------------
Some restrictions on the usages of wildcard filtering may be applied in the future.

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
None

Milestones
----------
None

Work Items
----------

The API changes mentioned above have essentially been completed. Examples of potential
future changes include:
* Restrictions on wildcard search (currently wildcard searching is unrestricted)
* Adding more attributes that can be filtered


Dependencies
============
None
