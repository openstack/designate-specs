..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode

============================
 Expose recordsets endpoint
============================

https://blueprints.launchpad.net/designate/+spec/expose-recordsets-api


Problem description
===================

Currently the ``/recordsets`` endpoint is a sub-resource of ``/zones``. But there is a need
to list all the records for a tenant in a single API call, so that doing filtering across
all the records under a tenant will be much easier.


Proposed change
===============

Designate will have a new API endpoint ``/v2/recordsets``.

* A single recordset can be retrieved via a GET call to ``/v2/recordsets/{recordset_id}``,
  which reponds a 301 and redirects to the canonical location
  ``/v2/zones/{zone_id}/recordsets/{recordset_id}``. The "self" link in the response body
  points to that location as well.
* All recordsets across all the zones owned by a tenant can be listed via a GET call to
  ``/v2/recordsets``, response will be paginated in this case if necessary.
* Filtering on all recordsets under a tenant will be supported, for example
  ``/v2/recordsets?name=foo``.


API Changes
-----------

API changes will be mainly about exposing the new API endpoint in controllers.

**Example single recordset retrieval request:**

.. code-block:: http

  GET /v2/recordsets/f7b10e9b-0cae-4a91-b162-562bc6096648 HTTP/1.1
  Host: 127.0.0.1:9001
  Accept: application/json
  Content-Type: application/json

  {
    "description": "This is an example recordset.",
    "links": {
        "self": "https://127.0.0.1:9001/v2/zones/2150b1bf-dee2-4221-9d85-11f7886fb15f/recordsets/f7b10e9b-0cae-4a91-b162-562bc6096648"
    },
    "updated_at": null,
    "records": [
        "10.1.0.2"
    ],
    "ttl": 3600,
    "id": "f7b10e9b-0cae-4a91-b162-562bc6096648",
    "name": "www.example.org.",
    "zone_id": "2150b1bf-dee2-4221-9d85-11f7886fb15f",
    "zone_name": "example.org.",
    "created_at": "2014-10-24T19:59:44.000000",
    "version": 1,
    "type": "A"
  }

**Example all recordset of a tenant retrieval request:**

.. code-block:: http

  GET /v2/recordsets HTTP/1.1
  Host: 127.0.0.1:9001
  Accept: application/json
  Content-Type: application/json

  {
    "recordsets": [
        {
            "description": null,
            "links": {
                "self": "https://127.0.0.1:9001/v2/zones/2150b1bf-dee2-4221-9d85-11f7886fb15f/recordsets/65ee6b49-bb4c-4e52-9799-31330c94161f"
            },
            "updated_at": null,
            "records": [
                "ns1.devstack.org."
            ],
            "action": "NONE",
            "ttl": null,
            "status": "ACTIVE",
            "id": "65ee6b49-bb4c-4e52-9799-31330c94161f",
            "name": "example.org.",
            "zone_id": "2150b1bf-dee2-4221-9d85-11f7886fb15f",
            "zone_name": "example.org.",
            "created_at": "2014-10-24T19:59:11.000000",
            "version": 1,
            "type": "NS"
        },
        {
            "description": null,
            "links": {
                "self": "https://127.0.0.1:9001/v2/zones/2150b1bf-dee2-4221-9d85-11f7886fb15f/recordsets/14500cf9-bdff-48f6-b06b-5fc7491ffd9e"
            },
            "updated_at": "2014-10-24T19:59:46.000000",
            "records": [
                "ns1.devstack.org. jli.ex.com. 1458666091 3502 600 86400 3600"
            ],
            "action": "NONE",
            "ttl": null,
            "status": "ACTIVE",
            "id": "14500cf9-bdff-48f6-b06b-5fc7491ffd9e",
            "name": "example.org.",
            "zone_id": "2150b1bf-dee2-4221-9d85-11f7886fb15f",
            "zone_name": "example.org.",
            "created_at": "2014-10-24T19:59:12.000000",
            "version": 1,
            "type": "SOA"
        },
        {
            "name": "jjli.com.",
            "id": "12caacfd-f0fc-4bcb-aa24-c42769897822",
            "type": "SOA",
            "zone_name": "jjli.com.",
            "action": "NONE",
            "ttl": null,
            "status": "ACTIVE",
            "description": null,
            "links": {
                "self": "http://127.0.0.1:9001/v2/zones/b8d7eaf1-e5c7-4b15-be6e-4b2809f47ec3/recordsets/12caacfd-f0fc-4bcb-aa24-c42769897822"
            },
            "created_at": "2016-03-22T16:12:35.000000",
            "updated_at": "2016-03-22T17:01:31.000000",
            "records": [
                "ns1.devstack.org. jli.ex.com. 1458666091 3502 600 86400 3600"
            ],
            "zone_id": "b8d7eaf1-e5c7-4b15-be6e-4b2809f47ec3",
            "version": 2
        },
        {
            "name": "jjli.com.",
            "id": "f39c51d1-ec2c-48a8-b9f7-877d56b7b82a",
            "type": "NS",
            "zone_name": "jjli.com.",
            "action": "NONE",
            "ttl": null,
            "status": "ACTIVE",
            "description": null,
            "links": {
                "self": "http://127.0.0.1:9001/v2/zones/b8d7eaf1-e5c7-4b15-be6e-4b2809f47ec3/recordsets/f39c51d1-ec2c-48a8-b9f7-877d56b7b82a"
            },
            "created_at": "2016-03-22T16:12:35.000000",
            "updated_at": null,
            "records": [
                "ns1.devstack.org."
            ],
            "zone_id": "b8d7eaf1-e5c7-4b15-be6e-4b2809f47ec3",
            "version": 1
         },
    ],
    "metadata": {
      "total_count": 4
    },
    "links": {
        "self": "https://127.0.0.1:9001/v2/recordsets"
    }
  }

**Example recordset filtering request:**

.. code-block:: http

  GET /v2/recordsets?data=192.168* HTTP/1.1
  Host: 127.0.0.1:9001
  Accept: application/json
  Content-Type: application/json

  {
    "metadata": {
      "total_count": 2
    },
    "links": {
      "self": "http://127.0.0.1:9001/v2/recordsets?data=192.168%2A"
    },
    "recordsets": [
      {
        "name": "ohoh.uyudbbgxdf.com.",
        "id": "a48588c5-5093-4585-b0fc-3e399d169c01",
        "type": "A",
        "zone_name": "uyudbbgxdf.com.",
        "action": "NONE",
        "ttl": null,
        "status": "ACTIVE",
        "description": null,
        "links": {
          "self": "http://127.0.0.1:9001/v2/zones/601a25f0-5c4d-4058-8d9c-e6a78f5ffbb8/recordsets/a48588c5-5093-4585-b0fc-3e399d169c01"
        },
        "created_at": "2016-04-04T20:11:08.000000",
        "updated_at": null,
        "records": [
          "192.168.0.1"
        ],
        "zone_id": "601a25f0-5c4d-4058-8d9c-e6a78f5ffbb8",
        "version": 1
      },
      {
        "name": "jli-1.uyudbbgxdf.com.",
        "id": "f2c7a0f6-8ec7-4d14-b8ec-2a55a8129160",
        "type": "A",
        "zone_name": "uyudbbgxdf.com.",
        "action": "NONE",
        "ttl": null,
        "status": "ACTIVE",
        "description": null,
        "links": {
          "self": "http://127.0.0.1:9001/v2/zones/601a25f0-5c4d-4058-8d9c-e6a78f5ffbb8/recordsets/f2c7a0f6-8ec7-4d14-b8ec-2a55a8129160"
        },
        "created_at": "2016-04-04T22:21:03.000000",
        "updated_at": null,
        "records": [
          "192.168.6.6"
        ],
        "zone_id": "601a25f0-5c4d-4058-8d9c-e6a78f5ffbb8",
        "version": 1
      }
    ]
  }

Central Changes
---------------

Central changes will include changing functions for finding recordsets from storage
in central.service to support corresponding calls from api layer.

Storage Changes
---------------
Corresponding changes to support the API change.

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
newton-1

Work Items
----------

* Make code changes to api, central and storage
* Add unit and functional tests.

Dependencies
============
None
