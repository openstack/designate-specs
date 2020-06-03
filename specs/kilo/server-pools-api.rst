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

================
Server Pools API
================

https://blueprints.launchpad.net/designate/+spec/server-pools-api

This spec outlines the API changes needed for Server Pools. It is part of a
larger group of specs describing the design of Designate Server Pools. The
main blueprint is located
`here <https://blueprints.launchpad.net/designate/+spec/server-pools>`_.

Problem description
===================

Server Pools will need to be created, read, updated and deleted. An API needs
to exist in order to carry out these operations.

Proposed change
===============

This spec defines in detail the API changes needed for Server Pools.

API Changes
-----------

API Details: Create/List/Patch/Delete Pools
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
.. tip:: These should all be restricted to admins for general usage.

+--------+------------------+------------------------------------------------+
| Verb   | Resource         | Description                                    |
+========+==================+================================================+
| GET    | /pools           | Returns all pools                              |
+--------+------------------+------------------------------------------------+
| GET    | /pools/{pool_id} | Returns a specific pool with additional details|
+--------+------------------+------------------------------------------------+
| POST   | /pools           | Creates a new pool                             |
+--------+------------------+------------------------------------------------+
| PATCH  | /pools/{pool_id} | Updates the pool with data in the request      |
+--------+------------------+------------------------------------------------+
| DELETE | /pools/{pool_id} | Deletes the pool                               |
+--------+------------------+------------------------------------------------+

GET /v2/pools
^^^^^^^^^^^^^

When no id is specified, all pools are returned. No body is provided in the
request.

.. code-block::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8
    Location: /v2/pools

    {
        "pools": [
            {
                "name": "US Pool 1",
                "id": "cf1fafc0-f2f0-11e3-ac10-0800200c9a66",
                "links": {
                    "self" : "/v2/pools/cf1fafc0-f2f0-11e3-ac10-0800200c9a66"
                },
                "created_at": "2014-09-09T19:34:21.819615",
                "updated_at": null
            },
            {
                "name": "US Pool 2",
                "id": "cddda8f0-f558-11e3-a3ac-0800200c9a66",
                "links": {
                    "self" : "/v2/pools/cddda8f0-f558-11e3-a3ac-0800200c9a66"
                },
                "created_at": "2014-09-09T19:34:21.819615",
                "updated_at": null
            }
        ],
        "links": {
            "self": "/v2/pools"
        }
    }


GET /v2/pools/{pool_id}
^^^^^^^^^^^^^^^^^^^^^^^

When an id is provided in the url, the specific pool is returned with more
detail. No body is provided in the request.

.. code-block::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8
    Location: /v2/pools/cf1fafc0-f2f0-11e3-ac10-0800200c9a66

    {
        "pool": {
            "name": "US Pool 1",
            "id": "cf1fafc0-f2f0-11e3-ac10-0800200c9a66",
            "attributes": {
                "geo-ip": true,
                "anycast": false,
                "scope": public,
            },
            "pool_servers": [
                "1.2.3.4:53",
                "5.4.3.2:53"
            ],
            "nameservers": [
                {"priority": 1, "hostname": "ns1.example.com."},
                {"priority": 2, "hostname": "ns2.example.com."}
            ],
            "project_id": "<uuid>",
            "provisioner": "unmanaged",
            "created_at": "2014-09-04T19:34:20.819723",
            "updated_at": null
        },
        "links": {
            "self": "/v2/pools/cf1fafc0-f2f0-11e3-ac10-0800200c9a66"
        }
    }


POST /v2/pools/
^^^^^^^^^^^^^^^
When a new Pool is created, the user must supply the name and scope. Initially,
only "public" scope is supported. The other values shown here are
optional. If the name is the same as an existing Pool, the return
code will be 409 Conflict.

**Request**

.. code-block::

    HTTP/1.1
    Content-Type: application/json; charset=UTF-8
    Location: /v2/pools/

    {
        "pool": {
            "name": "US Pool 3",
            "attributes": {
                "scope": public
            },
            "pool_servers": [
                "10.11.12.13:53",
                "13.12.11.10:53"
            ],
            "nameservers": [
                {"priority": 1, "hostname": "ns1.example.com."}
                {"priority": 2, "hostname": "ns2.example.com."}
            ]
        }
    }

**Response**

.. code-block::

    HTTP/1.1 201 Created
    Content-Type: application/json; charset=UTF-8
    Location: /v2/pools/

    {
        "pool": {
            "name": "US Pool 3",
            "id": "cf1fafc0-f2f0-11e3-ac10-0800200c81a5",
            "attributes": {
                "scope": public,
            }
            "pool_servers": [
                "10.11.12.13:53",
                "13.12.11.10:53"
            ],
            "nameservers": [
                {"priority": 1, "hostname": "ns1.example.com."},
                {"priority": 2, "hostname": "ns2.example.com."}
            ],
            "created_at": "2014-09-04T19:34:20.819723",
            "updated_at": null
        },
        "links": {
            "self": "/v2/pools/cf1fafc0-f2f0-11e3-ac10-0800200c81a5"
        }
    }


PATCH /v2/pools/{pool_id}
^^^^^^^^^^^^^^^^^^^^^^^^^
To modify a Pool, a PATCH request is submitted. If it is successful, a 200 is
returned.

**Request**

.. code-block::

    HTTP/1.1
    Content-Type: application/json; charset=UTF-8
    Location: /v2/pools/cf1fafc0-f2f0-11e3-ac10-0800200c81a5

    {
        "pool": {
            "pool_servers": [
                "4.5.6.7:53"
            ]
        }
    }

**Response**

.. code-block::

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8
    Location: /v2/pools/cf1fafc0-f2f0-11e3-ac10-0800200c81a5

    {
        "pool": {
            "name": "US Pool 3",
            "id": "cf1fafc0-f2f0-11e3-ac10-0800200c81a5",
            "attributes": {
                "scope": public,
            },
            "pool_servers": [
                "10.11.12.13:53",
                "13.12.11.10:53",
                "4.5.6.7:53"
            ],
            "nameservers": [
                {"priority": 1, "hostname": "ns1.example.com."},
                {"priority": 2, "hostname": "ns2.example.com."}
            ],
            "created_at": "2014-09-04T19:34:20.819723",
            "updated_at": "2014-09-10T19:33:10.819555"
        },
        "links": {
            "self": "/v2/pools/cf1fafc0-f2f0-11e3-ac10-0800200c81a5"
        }
    }

DELETE /v2/pools/{pool_id}
^^^^^^^^^^^^^^^^^^^^^^^^^^
When deleting a Pool, the user must supply the id in the url. The request body
and return body are empty. A 204 is returned

    HTTP/1.1 204 No Content
    Content-Type: application/json; charset=UTF-8
    Location: /v2/pools/cf1fafc0-f2f0-11e3-ac10-0800200c81a5


Central Changes
---------------

The pool calls will have to be added to central to allow the CRUD of Server
Pools.

Storage Changes
---------------

The pool calls will have to be added to storage. The new tables are described
in a different spec.


Implementation
==============

Assignee(s)
-----------
Primary assignee:
  https://launchpad.net/~betsy-luzader


Milestones
----------

Target Milestone for completion:
  Kilo

Work Items
----------

* Create pools controller and view
* Add the calls to Central
* Add the calls to Storage
* Write tests



Dependencies
============

* Server Pools Storage: https://review.openstack.org/#/c/113447/
* Server Pools Service: https://review.openstack.org/#/c/113462/
* Server Pools MiniDNS Support: https://review.openstack.org/#/c/112688/
