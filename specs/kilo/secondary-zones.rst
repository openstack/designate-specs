..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode

..

.. _secondary_zones:

===============
Secondary Zones
===============

Outline support for Secondary Zones concept in Designate.

Terminology
===========

+----------+---------------------------------------------+
| Term     | Meaning                                     |
+==========+=============================================+
| axfr     | Full Zone Update (AXFR)                     |
+----------+---------------------------------------------+
| zone     | Seconday Zone - Zone that transfers it's    |
|          | state / data from a given set of masters    |
+----------+---------------------------------------------+

Problem description
===================

We want to support Secondary Zones in order to lay the grounds for higher level
features like incoming (A|I)XFR, Federation and others (will be separate specs).

Proposed change
===============

We introduce a new concept of a Secondary zone within Designate that has
external Masters. In term this means that Designate will be acting as a Slave.

In MiniDNS we add support for inbound AXFR and NOTIFY.

In terms of Pools a Secondary Zone would be placed equally as a Primary

API Changes
-----------

API endpoints for recordsets should disallow any Create, Update, Delete action.

*status* fields will be the same as with Primary Zones.

New Zone parameters
-------------------

+----------------+----------------------------+-----------+---------+-------------------------+
| Parameter      | Type                       | Required  | Default | Description             |
+================+============================+===========+=========+=========================+
| type           | ENUM(PRIMARY, SECONDARY)   | No        | false   | Primary or Secondary    |
+----------------+----------------------------+-----------+---------+-------------------------+
| masters        | List                       | No        | None    | Zone Masters            |
+----------------+----------------------------+-----------+---------+-------------------------+
| transferred_at | Datetime                   | -         | None    | Last successful transfer|
+----------------+----------------------------+-----------+---------+-------------------------+


Create Secondary Zone
^^^^^^^^^^^^^^^^^^^^^

When creating a new zone the *managed_resource_email* is used as the initial *email*.

The fields *version* will be *1* since it's not yet transferred and *transferred_at* as *null*.

.. code-block:: http

        POST /v2/zones HTTP/1.1
        Host: 127.0.0.1:9001
        Accept: application/json
        Content-Type: application/json

        {
          "zone": {
            "name": "example.org.",
            "description": "This is an example zone.",
            "masters": ["10.0.0.1", "10.0.0.2:5354"],
            "type": "SECONDARY",
          }
        }

.. code-block:: http

        HTTP/1.1 201 Created
        Content-Type: application/json

        {
          "zone": {
            "id": "a86dba58-0043-4cc6-a1bb-69d5e86f3ca3",
            "pool_id": "572ba08c-d929-4c70-8e42-03824bb24ca2",
            "project_id": "4335d1f0-f793-11e2-b778-0800200c9a66",
            "name": "example.org.",
            "email": "managed@foo.co",
            "serial": 1404757531,
            "status": "ACTIVE",
            "description": "This is an example zone.",
            "masters": ["10.0.0.1", "10.0.0.2:5354"],
            "type": "SECONDARY",
            "transferred_at": null,
            "version": 1,
            "created_at": "2014-07-07T18:25:31.275934",
            "updated_at": null,
            "links": {
              "self": "https://127.0.0.1:9001/v2/zones/a86dba58-0043-4cc6-a1bb-69d5e86f3ca3"
            }
          }
        }


Get Secondary Zone
^^^^^^^^^^^^^^^^^^

Retrieves a secondary zone with the specified ID.

Example of GET on a untransferred zone:

.. code-block:: http


        GET /v2/zones/a86dba58-0043-4cc6-a1bb-69d5e86f3ca3 HTTP/1.1
        Host: 127.0.0.1:9001
        Accept: application/json
        Content-Type: application/json

.. code-block:: http

        HTTP/1.1 200 OK
        Vary: Accept
        Content-Type: application/json

        {
          "zone": {
            "id": "a86dba58-0043-4cc6-a1bb-69d5e86f3ca3",
            "pool_id": "572ba08c-d929-4c70-8e42-03824bb24ca2",
            "project_id": "4335d1f0-f793-11e2-b778-0800200c9a66",
            "name": "example.org.",
            "email": "managed@foo.co",
            "serial": 1404757531,
            "status": "ACTIVE",
            "description": "This is an example zone.",
            "masters": ["10.0.0.1", "10.0.0.2:5354"],
            "type": "SECONDARY",
            "transferred_at": null,
            "version": 1,
            "created_at": "2014-07-07T18:25:31.275934",
            "updated_at": null,
            "links": {
              "self": "https://127.0.0.1:9001/v2/zones/a86dba58-0043-4cc6-a1bb-69d5e86f3ca3"
            }
          }
        }


List Secondary Zones
^^^^^^^^^^^^^^^^^^^^

To filter on zone type do type=<PRIMARY|SECONDARY> as query parameters.

Below there is examples of a Zone that's not transferred yet and one that is.

.. code-block:: http

        GET /v2/zones?type=SECONDARY HTTP/1.1
        Host: 127.0.0.1:9001
        Accept: application/json
        Content-Type: application/json

.. code-block:: http

        HTTP/1.1 200 OK
        Vary: Accept
        Content-Type: application/json

        {
          "zones": [{
            "id": "a86dba58-0043-4cc6-a1bb-69d5e86f3ca3",
            "pool_id": "572ba08c-d929-4c70-8e42-03824bb24ca2",
            "project_id": "4335d1f0-f793-11e2-b778-0800200c9a66",
            "name": "example.org.",
            "email": "managed@foo.co",
            "serial": 2014120100,
            "status": "ACTIVE",
            "description": "This is an example zone.",
            "masters": ["10.0.0.1", "10.0.0.2:5354"],
            "type": "SECONDARY",
            "transferred_at": "2014-07-07T18:25:31.275934",
            "version": 2,
            "created_at": "2014-07-07T18:25:31.275934",
            "updated_at": "2014-07-07T18:25:31.275934",
            "links": {
              "self": "https://127.0.0.1:9001/v2/zones/a86dba58-0043-4cc6-a1bb-69d5e86f3ca3"
            }
          }, {
            "id": "fdd7b0dc-52a3-491e-829f-41d18e1d3ada",
            "pool_id": "572ba08c-d929-4c70-8e42-03824bb24ca2",
            "project_id": "4335d1f0-f793-11e2-b778-0800200c9a66",
            "name": "example.net.",
            "email": "managed@foo.co",
            "serial": 1404756682,
            "status": "ACTIVE",
            "description": "This is another example zone.",
            "masters": ["10.0.0.1", "10.0.0.2:5354"],
            "type": "SECONDARY",
            "transferred_at": null,
            "version": 1,
            "created_at": "2014-07-07T18:22:08.287743",
            "updated_at": null,
            "links": {
              "self": "https://127.0.0.1:9001/v2/zones/fdd7b0dc-52a3-491e-829f-41d18e1d3ada"
            }
          }],
          "links": {
            "self": "https://127.0.0.1:9001/v2/zones"
          }
        }


Update a Secondary Zone
^^^^^^^^^^^^^^^^^^^^^^^

Changes the specified attribute(s) for an existing zone.


In the example below, we update one of the masters to 10.0.0.3.

NOTE: In terms of a Secondary Zone only the following fields below are
editable.

+-------------+--------------------------+
| Parameter   |  Description             |
+=============+==========================+
| description | Description of Zone      |
+-------------+--------------------------+
| masters     | Master servers           |
+-------------+--------------------------+


.. code-block:: http

        PATCH /v2/zones/a86dba58-0043-4cc6-a1bb-69d5e86f3ca3 HTTP/1.1
        Host: 127.0.0.1:9001
        Accept: application/json
        Content-Type: application/json

        {
          "zone": {
            "masters": ["10.0.0.1", 10.0.0.3:1053"]
          }
        }

.. code-block:: http

        HTTP/1.1 200 OK
        Content-Type: application/json

        {
          "zone": {
            "id": "a86dba58-0043-4cc6-a1bb-69d5e86f3ca3",
            "pool_id": "572ba08c-d929-4c70-8e42-03824bb24ca2",
            "project_id": "4335d1f0-f793-11e2-b778-0800200c9a66",
            "name": "example.org.",
            "serial": 0,
            "status": "ACTIVE",
            "description": "This is an example zone.",
            "masters": ["10.0.0.1", "10.0.0.3:1053"],
            "type": "SECONDARY",
            "transferred_at": "2014-07-07T18:25:31.275934",
            "version": 1,
            "created_at": "2014-07-07T18:25:31.275934",
            "updated_at": null,
            "links": {
              "self": "https://127.0.0.1:9001/v2/zones/a86dba58-0043-4cc6-a1bb-69d5e86f3ca3"
            }
          }
        }


Delete Secondary Zone
^^^^^^^^^^^^^^^^^^^^^

.. code-block:: http

        DELETE /v2/zones/a86dba58-0043-4cc6-a1bb-69d5e86f3ca3 HTTP/1.1
        Host: 127.0.0.1:9001

.. code-block:: http

        HTTP/1.1 204 No Content


Central Changes
---------------

Disallow changing of a zone from PRIMARY <> SECONDARY.

Add a periodic task that loops over secondary zones looking at their `transferred_at`
and does a call to MDNS to trigger a new AXFR to keep the zone updated.

Storage Changes
---------------

Modify Table - domains
^^^^^^^^^^^^^^^^^^^^^^

+----------------+--------------------------+------------------------+---------+-------------------------+--------+
| Column         | Type                     | Nullable?              | Unique? | Notes                   | Action |
+================+==========================+========================+=========+=========================+========+
| type           | EMUM(PRIMARY, SECONDARY) | No                     | No      | Zone type               | add    |
+----------------+--------------------------+------------------------+---------+-------------------------+--------+
| transferred_at | DATETIME                 | Yes (Not transferred)  | No      | Last transfer at        | add    |
+----------------+--------------------------+------------------------+---------+-------------------------+--------+

New Table - domain_attributes
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A new table to store any metadata / attributes that doesn't need to be on the
on the domains table.

An index across domain_id, key, value.

+---------------+--------------------------+----------------------+---------+--------------------------------------+--------+
| Column        | Type                     | Nullable?            | Unique? | Notes                                | Action |
+===============+==========================+======================+=========+======================================+========+
| id            | UUID                     | no                   | Yes     | ID for this attribute                | add    |
+---------------+--------------------------+----------------------+---------+--------------------------------------+--------+
| domain_id     | FK to Domain UUID        | no                   | No      | Domain ID this attribute belongs to  | add    |
+---------------+--------------------------+----------------------+---------+--------------------------------------+--------+
| key           | ENUM(masters)            | no                   | No      | Zone type                            | add    |
+---------------+--------------------------+----------------------+---------+--------------------------------------+--------+
| value         | VARCHAR                  | no                   | No      | Master servers for Zone              | add    |
+---------------+--------------------------+----------------------+---------+--------------------------------------+--------+


MiniDNS Changes
---------------

Zone Creation
^^^^^^^^^^^^^

When a zone is created currently a notification is sent to mdns, we'll plugin
here and do a AXFR if zone.type is SECONDARY.


NOTIFY
^^^^^^

We need to change __call__ to pass NOTIFY down to self._handle_notify().

1. Receives a NOTIFY
2. Query the SOURCE of the NOTIFY for SOA
3. Compare response serial vs local if it doesn't match then continue to next step.
4. Do a AXFR towards the server that sent the NOTIFY.
5. Call dnsutils.from_dnspython to get Domain a'la Designate version
6. Call Central to method with the data from #5 to update the domain.


New - RequestHandler._handle_notify(context, request)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Handles a Notification and eventually hands off to do a AXFR.

+-----------------+---------------------------------+--------------+
| **Parameter**   | **Description**                 | **Required** |
+=================+=================================+==============+
| *context*       | Security context information.   | Yes          |
+-----------------+---------------------------------+--------------+
| *request*       | The DNS request                 | Yes          |
+-----------------+---------------------------------+--------------+


New - Service.zone_sync(context, zone, master_addr=None)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A method utilized by any method in MDNS that needs to do a AXFR.

+-----------------+---------------------------------+--------------+
| **Parameter**   | **Description**                 | **Required** |
+=================+=================================+==============+
| *context*       | Security context information.   | Yes          |
+-----------------+---------------------------------+--------------+
| *zone*          | A objects.Domain object         | Yes          |
+-----------------+---------------------------------+--------------+
| *master_addr*   | Address to use for the AXFR     | No           |
+-----------------+---------------------------------+--------------+


Future work
===========

The below is out of scope for this.

* A task for /zones/<id>/tasks or so should be added in the future
  to allow a "forced" AXFR via the API.
* A task to switch from a SECONDARY > PRIMARY.


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

* Add new columns to storage
* Extend Zones API to allow CRUD for Secondary Zones
* Extend Central logic to check if it's a Master / Secondary Zone.


Dependencies
============

- :ref:`zone_import_refactor`