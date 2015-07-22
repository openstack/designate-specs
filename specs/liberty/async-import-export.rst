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

===============================
Asynchronous Zone Import/Export
===============================

https://blueprints.launchpad.net/designate/+spec/async-import-export

Problem description
===================

Large zone imports have the potential to take a lot longer than the
typically allowed time for API responses. Parsing large zone files, and inserting
the necessary records scales linearly with the amount of records that need to be
imported.

Proposed change
===============

The API for Zone imports needs to be changed to be asynchronous. This would involve
a new database table for zone import "statuses", and a more robust system for managing
the parsing and insertion of the zone data. Exporting zones will also return to the v2 API.

API Changes
-----------

GET /v2/zones/tasks/imports
^^^^^^^^^^^^^^^^^^^^^^^^^^^

This allows the user to view the statuses of their zone import requests. If the import has
completed successfully, they can get the zone id and a link to the zone.

.. code-block:: http

    GET /v2/zones/tasks/import/cddda8f0-f558-11e3-a3ac-0800200c9a66 HTTP/1.1
    Accept: application/json

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8
    Location: /v2/zones/tasks/import/cddda8f0-f558-11e3-a3ac-0800200c9a66

    {
      "imports": [
          {
            "id": "cddda8f0-f558-11e3-a3ac-0800200c9a66",
            "zone_id": "agqm44f0-s638-15e3-f4d3-1893572c9a67",
            "status": "SUCCESS",
            "links":{
                "self": "http://127.0.0.1:9001/v2/zones/tasks/import/cddda8f0-f558-11e3-a3ac-0800200c9a66",
                "href": "http://127.0.0.1:9001/v2/zones/agqm44f0-s638-15e3-f4d3-1893572c9a67"
          },
          {
            "id": "addda8f0-f558-11e3-a3ac-0800200c9a66",
            "zone_id": "qgqm44f0-s638-15e3-f4d3-1893572c9a67",
            "status": "SUCCESS",
            "links":{
                "self": "http://127.0.0.1:9001/v2/zones/tasks/import/addda8f0-f558-11e3-a3ac-0800200c9a66",
                "href": "http://127.0.0.1:9001/v2/zones/qgqm44f0-s638-15e3-f4d3-1893572c9a67"
          }
      ]
    }

POST /v2/zones/tasks/imports
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This creates a request to import a zone. The zone data is passed via the request body,
and the Content-Type header must be set to 'text/dns'.

This returns an id for the import request, for the user to query.

.. code-block:: http

    POST /v2/zones/tasks/import HTTP/1.1
    Accept: application/json
    Content-Type: text/dns

    $ORIGIN example.com.
    example.com. 42 IN SOA ns.example.com. nsadmin.example.com. 42 42 42 42 42
    example.com. 42 IN NS ns.example.com.
    example.com. 42 IN MX 10 mail.example.com.
    ns.example.com. 42 IN A 10.0.0.1
    mail.example.com. 42 IN A 10.0.0.2

    HTTP/1.1 201 Accepted
    Content-Type: application/json; charset=UTF-8
    Location: /v2/zones/tasks/import/cddda8f0-f558-11e3-a3ac-0800200c9a66

    {
        "id": "cddda8f0-f558-11e3-a3ac-0800200c9a66",
        "zone_id": null,
        "status": "PENDING",
        "links":{
            "self": "http://127.0.0.1:9001/v2/zones/tasks/import/cddda8f0-f558-11e3-a3ac-0800200c9a66"
        }
    }

GET /v2/zones/tasks/imports/<id>
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This allows the user to view the status of their zone import request. If the import has
completed successfully, they can get the zone id and a link to the zone.

.. code-block:: http

    GET /v2/zones/tasks/import/cddda8f0-f558-11e3-a3ac-0800200c9a66 HTTP/1.1
    Accept: application/json

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8
    Location: /v2/zones/tasks/import/cddda8f0-f558-11e3-a3ac-0800200c9a66

    {
        "id": "cddda8f0-f558-11e3-a3ac-0800200c9a66",
        "zone_id": "agqm44f0-s638-15e3-f4d3-1893572c9a67",
        "status": "SUCCESS",
        "links":{
            "self": "http://127.0.0.1:9001/v2/zones/tasks/import/cddda8f0-f558-11e3-a3ac-0800200c9a66",
            "href": "http://127.0.0.1:9001/v2/zones/agqm44f0-s638-15e3-f4d3-1893572c9a67"
        }
    }

GET /v2/zones/<id>
^^^^^^^^^^^^^^^^^^

This request, with the header "Accept:text/dns" exports the zone in DNS zonefile format to the user.

.. code-block:: http

    GET /v2/zones/cddda8f0-f558-11e3-a3ac-0800200c9a66 HTTP/1.1
    Accept: text/dns

    HTTP/1.1 200 OK
    Content-Type: text/dns; charset=UTF-8
    Location: /v2/zones/cddda8f0-f558-11e3-a3ac-0800200c9a66

    HTTP/1.1 200 OK
    Content-Type: text/dns

    $ORIGIN example.com.
    $TTL 42
    example.com. IN SOA ns.designate.com. nsadmin.example.com. (
        1394213803 ; serial
        3600 ; refresh
        600 ; retry
        86400 ; expire
        3600 ; minimum
    )
    example.com. IN NS ns.designate.com.
    example.com.  IN MX 10 mail.example.com.
    ns.example.com.  IN A  10.0.0.1
    mail.example.com.  IN A  10.0.0.2

Central Changes
---------------

create_import_domain(body)
^^^^^^^^^^^^^^^^^^^^^^^^^^

+---------------+-----------------------------------------------+--------------+
| **Parameter** | **Description**                               | **Required** |
+===============+===============================================+==============+
| *body*        | Unserialized request data from the API request| Yes          |
+---------------+-----------------------------------------------+--------------+

1. Create an entry in the zone_imports table to track the request with status
PENDING.
2. Kicks off a thread to _import_domain with the request body.
3. Returns a zone_import object.

get_import_domain(id)
^^^^^^^^^^^^^^^^^^^^^

+---------------+-----------------------------------------------+--------------+
| **Parameter** | **Description**                               | **Required** |
+===============+===============================================+==============+
| *import_id*   | The id of the zone import                     | Yes          |
+---------------+-----------------------------------------------+--------------+

1. Calls storage.find_import to get a specific zone_import record from the
zone_imports table.
2. Returns the zone_import object

find_import_domains(context)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

+---------------+-----------------------------------------------+--------------+
| **Parameter** | **Description**                               | **Required** |
+===============+===============================================+==============+
| *context*     | Context to be passed to storage.find_imports  | Yes          |
+---------------+-----------------------------------------------+--------------+

1. Calls off to the storage.find_imports to find all the zone imports for a
tenant.
2. Returns the zone_import_list object

delete_import_domain(id)
^^^^^^^^^^^^^^^^^^^^^^^^

+---------------+-----------------------------------------------+--------------+
| **Parameter** | **Description**                               | **Required** |
+===============+===============================================+==============+
| *import_id*   | The id of the zone import                     | Yes          |
+---------------+-----------------------------------------------+--------------+

1. Calls off to storage.delete_zone_import to delete the zone import record from
the zone_imports table.

_import_domain(body)
""""""""""""""""""""

+---------------+-----------------------------------------------+--------------+
| **Parameter** | **Description**                               | **Required** |
+===============+===============================================+==============+
| *body*        | Unserialized request data from the API request| Yes          |
+---------------+-----------------------------------------------+--------------+
| *zone_import* | zone_import object from the original request  | Yes          |
+---------------+-----------------------------------------------+--------------+


1. Tries to use dnspython's from_text method to convert the zone into a dnspython
object. If that doesn't work, updates the zone_import object in storage to ERROR
to indicate that the zone could not be imported.
2. Converts the dnspython zone to a Designate domain object.
3. Calls central.create_domain with the converted object.
4. Takes the return object's id and updates the zone_import object in storage
with the new zone_id and updates the status of the import to COMPLETE, or similar.


Storage Changes
---------------

A new table for zone import records will be added, along with the boilerplate
get, set, delete, update methods.


New Table - zone_imports
^^^^^^^^^^^^^^^^^^^^^^^^

+---------+---------+-----------+---------+-------------------------------------+
| Row     | Type    | Nullable? | Unique? | Notes                               |
+=========+=========+===========+=========+=====================================+
| id      | uuid    | No        | Yes     | Primary key                         |
+---------+---------+-----------+---------+-------------------------------------+
| zone_id | uuid    | Yes       | Yes     | Zone id when the import completes   |
+---------+---------+-----------+---------+-------------------------------------+
| status  | ENUM    | Yes       | No      | One of [COMPLETE, PENDING, ERROR]   |
+---------+---------+-----------+---------+-------------------------------------+
| message | VARCHAR | Yes       | No      | A message letting the user know why |
|         |         |           |         | their import failed. For example:   |
|         |         |           |         | "Malformed zonefile", "Complete"    |
+---------+---------+-----------+---------+-------------------------------------+

Other Changes
-------------

New DesignateObjects for the Imports will have be created, ZoneImport, ZoneImportList.

Exporting zones will also return to the v2 API, but should remain relatively unchanged.

Alternatives
------------

Instead of creating a new zone_import table and object, it may be possible
to create an empty domain object and call central._create_domain_in_storage
and make an empty domain that is then updated with the result of the import.
Only then could the call to the pool manager be made.

This might make the code simpler. But provides very little in the way of
letting the user know that their import failed.
You could add a status like MALFORMED_ZONEFILE or something, but that
would still require the user to delete the zone before they tried again.
Unless you soft-delete the zone when it fails and modify the default
find_domains criterion to find zones that have been deleted only if they
have that status.

Assignee(s)
-----------

Primary assignee:
  tim-simmons-t

Milestones
----------

Target Milestone for completion:
  Liberty-1

Work Items
----------

- Implement the database table, migration.
- Implement the storage methods.
- Implement the central methods.
- Fix the API to use the new code.
- Move the APIs back into the v2 namespace
