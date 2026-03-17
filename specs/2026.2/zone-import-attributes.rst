..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
 Zone Attributes Support for Zone Import
==========================================

https://bugs.launchpad.net/designate/+bug/1690184

Zone import currently does not support specifying zone attributes. This means
zones imported via ``POST /v2/zones/tasks/imports`` are always scheduled to the
default pool. With the deprecation of pool management APIs, zone attributes are
the primary mechanism for controlling pool assignment during zone creation. Zone
import should support the same attributes that zone creation does, so that
imported zones can be scheduled to the correct pool.


Problem description
===================

When creating a zone via ``POST /v2/zones``, users can specify ``attributes``
in the JSON body to influence pool scheduling. For example:

.. code-block:: json

    {
        "name": "example.com.",
        "email": "admin@example.com",
        "attributes": {"pool_id": "794ccc2c-d751-44fe-b57f-8894c9f5c842"}
    }

The pool scheduler uses these attributes to match the zone to the appropriate
pool. However, zone import (``POST /v2/zones/tasks/imports``) only accepts a
``text/dns`` body containing the raw zonefile. There is no way to pass zone
attributes, so imported zones always land in the default pool.

This is a significant limitation for operators managing multiple pools. The
available workarounds are cumbersome: one option is to import each zone into
the default pool and then use ``zone pool move`` to relocate each zone
individually to the correct pool — a tedious per-zone process. Another option
is to skip import entirely, create each zone via ``POST /v2/zones`` with the
desired attributes, and then populate it with records through the API — which
defeats the purpose of having a zonefile-based import. With pool management
APIs deprecated, this gap becomes even more critical, as attributes are now the
standard way to control pool assignment.


Proposed change
===============

Extend the zone import endpoint to optionally accept ``application/json`` as a
content type, in addition to the existing ``text/dns``. When the request body
is JSON, it may include both the zonefile content and zone attributes:

.. code-block:: json

    {
        "zonefile": "$ORIGIN example.com.\n$TTL 3600\n...",
        "attributes": {"pool_id": "794ccc2c-d751-44fe-b57f-8894c9f5c842"}
    }

The attributes are passed through to the zone object before creation, using
the same code path as ``POST /v2/zones``. This means pool scheduling,
validation, and attribute storage all work identically to normal zone creation.

When no attributes are provided, or when the request uses ``text/dns``, the
behavior is unchanged — the zone is scheduled to the default pool as it is
today. This makes the change fully backwards compatible.


API Changes
-----------

POST /v2/zones/tasks/imports
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The endpoint currently accepts only ``text/dns``. This change adds
``application/json`` as an accepted content type.

**Existing behavior (unchanged):**

Request:

.. code-block:: text

    POST /v2/zones/tasks/imports HTTP/1.1
    Content-Type: text/dns

    $ORIGIN example.com.
    $TTL 3600
    example.com. IN SOA ns1.example.com. admin.example.com. (
        2024101401 3600 900 1209600 300 )
    example.com. IN NS ns1.example.com.
    example.com. IN A 192.0.2.1

Response:

.. code-block:: text

    HTTP/1.1 202 Accepted
    Content-Type: application/json

    {
        "id": "074e2e30-8748-11e4-a4a2-08002783c8f3",
        "status": "PENDING",
        "message": null,
        "zone_id": null,
        "links": {
            "self": "/v2/zones/tasks/imports/074e2e30-8748-11e4-a4a2-08002783c8f3"
        }
    }

**New behavior (with attributes):**

Request:

.. code-block:: text

    POST /v2/zones/tasks/imports HTTP/1.1
    Content-Type: application/json

    {
        "zonefile": "$ORIGIN example.com.\n$TTL 3600\nexample.com. IN SOA ...",
        "attributes": {
            "pool_id": "794ccc2c-d751-44fe-b57f-8894c9f5c842"
        }
    }

Response:

.. code-block:: text

    HTTP/1.1 202 Accepted
    Content-Type: application/json

    {
        "id": "074e2e30-8748-11e4-a4a2-08002783c8f3",
        "status": "PENDING",
        "message": null,
        "zone_id": null,
        "links": {
            "self": "/v2/zones/tasks/imports/074e2e30-8748-11e4-a4a2-08002783c8f3"
        }
    }

**JSON body parameters:**

+------------+---------------------------------------------+----------+
| Parameter  | Description                                 | Required |
+============+=============================================+==========+
| zonefile   | The zonefile content as a string (same      | Yes      |
|            | format as the text/dns body).               |          |
+------------+---------------------------------------------+----------+
| attributes | Zone attributes as key-value pairs. Accepts | No       |
|            | the same attributes as zone creation (e.g.  |          |
|            | pool_id, scope).                            |          |
+------------+---------------------------------------------+----------+

**Error cases:**

- ``application/json`` body missing ``zonefile`` field: ``400 Bad Request``
- Invalid attributes (e.g. referencing non-existent pool): handled during zone
  creation, reported in the zone import status as ``ERROR``


Central Changes
---------------

The ``create_zone_import`` method in central will accept an optional
``zone_attributes`` dict parameter. This is passed through to
``_import_zone``, which sets the attributes on the zone object before calling
``create_zone``.

The attributes are parsed using the existing ``ZoneAttributeListAPIv2Adapter``,
the same adapter used by zone creation. This ensures consistent validation and
behavior.


Storage Changes
---------------

None. Zone attributes are already stored via the existing zone creation flow.


Other Changes
-------------

**python-designateclient**

The ``openstack zone import create`` command will accept an optional
``--attributes`` flag using the same ``key:value`` format as
``openstack zone create``:

.. code-block:: bash

    openstack zone import create /path/to/zonefile \
        --attributes pool_id:794ccc2c-d751-44fe-b57f-8894c9f5c842

When ``--attributes`` is provided, the client sends the request as
``application/json`` with the zonefile content and attributes in the body.
When ``--attributes`` is omitted, the client sends ``text/dns`` as it does
today.

**openstacksdk**

Update the ``ZoneImport`` resource in openstacksdk to support the new
``application/json`` content type and zone attributes.

**designate-tempest-plugin**

API tests covering:

- Zone import with attributes (verify pool assignment)
- Zone import with ``application/json`` content type
- Zone import without attributes (backwards compatibility)
- Zone import with invalid attributes (error handling)

**api-ref**

Update the zone import API reference documentation to describe the new
``application/json`` content type and the ``attributes`` parameter.


Alternatives
------------

Several alternatives were considered:

1. **Custom HTTP header (X-Designate-Zone-Attributes)**: Pass attributes as a
   custom header. This was prototyped in
   https://review.opendev.org/c/openstack/designate/+/932323 but rejected
   because headers are not easily discoverable in API documentation, require
   fragile string-to-JSON parsing, and need CORS configuration changes.

2. **Query parameters**: Pass attributes as query parameters (e.g.
   ``?attributes=pool_id:uuid``). Simple for a single attribute but becomes
   unwieldy with many attributes and is inconsistent with how zone creation
   accepts them.

3. **Two-step process**: Import the zone, then update attributes via PATCH.
   Not viable because pool scheduling happens at creation time — the zone
   would already be assigned to the wrong pool.

The JSON body approach was chosen because it reuses the exact same attribute
format as zone creation, scales to any number of attributes, and is easy to
document and discover.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Omer Schwartz (oschwart)


Milestones
----------

Target Milestone for completion:
  2026.2 (Hibiscus)


Work Items
----------

- Designate: extend zone import endpoint to accept ``application/json``
- Designate: pass attributes through central to zone creation
- Designate: unit and functional tests
- python-designateclient: add ``--attributes`` flag to ``zone import create``
- openstacksdk: update ``ZoneImport`` resource to support attributes
- designate-tempest-plugin: API tests for zone import with attributes
- api-ref: document new content type and attributes parameter


Dependencies
============

None. This feature builds entirely on existing zone attribute and pool
scheduling infrastructure.


Upgrade Implications
====================

None. The change is backwards compatible. Existing ``text/dns`` requests
continue to work unchanged. The new ``application/json`` content type is
opt-in.
