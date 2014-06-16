..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode

=========================================
 Zone Ownership Transfer Between projects
=========================================

https://blueprints.launchpad.net/designate/+spec/zone-migration-between-tenants

For many projects, their organizational needs and structure can change. These
changes result in the need to move a zone from one project to another without
interruption to domain name resolution.

Problem description
===================

Allow for a zone, and its associated recordsets and records to be moved from
one project to another.

This will remove the original owning projects ability to modify the zone.

This could also allow a project to create a sub domain, and transfer the
ownership to another project. (ie. the IT team of example.net allows developers
to manage records in dev-env.example.net)

We allow both scoped (limited to a defined project), and unscoped (anyone with
the ID and access key can accept a request)

Terminology
===========

+----------+--------------------------------------------------------------------------------------------------------------+
| Term     | Meaning                                                                                                      |
+==========+==============================================================================================================+
| zone     | DNS Zone (also refered to as a DNS domain)                                                                   |
+----------+--------------------------------------------------------------------------------------------------------------+
| project  | OpenStack keystone project (also refered to as tenant)                                                       |
+----------+--------------------------------------------------------------------------------------------------------------+
| transfer | method of transfering the ownership from one project to another (**NOT** traditional DNS AFXR zone transfer) |
+----------+--------------------------------------------------------------------------------------------------------------+

Proposed change
===============

New API Endpoint
----------------

+-------------------+------------------------------------------------------+-------------------------------+
| Parameter         | Description                                          | Required                      |
+===================+======================================================+===============================+
| zone_id           | ID of the DNS Zone to be tranfered                   | Yes                           |
+-------------------+------------------------------------------------------+-------------------------------+
| description       | Description of the transfer                          | No                            |
+-------------------+------------------------------------------------------+-------------------------------+
| target_project_id | Restrict this share request to the specified project | No - defaults to all projects |
+-------------------+------------------------------------------------------+-------------------------------+

POST /v2/zones/<zone-id>/tasks/transfer-requests
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This creates a transfer request for a zone owned by the requesting project.

It returns an ID and an access key, which should be communicated out of band to
the reciepient (via email / IM etc). They will need both of these to accept the
transfer

.. important:: There should only be one active transfer request per zone at any
   one point in time

.. code-block:: http

    POST /v2/zones/c11ae7e0-f558-11e3-a3ac-0800200c9a66/tasks/transfer-requests HTTP/1.1
    Accept: application/json
    Content-Type: application/json

    {
        "transfer_request":{
            "description":"Transfer to Developers",
            "target_project_id":"88cbc4c7-1dee-40be-804c-ecf86962198c"
        }
    }

.. code-block:: http

    HTTP/1.1 201 Created
    Content-Type: application/json; charset=UTF-8
    Location: /v2/zones/c11ae7e0-f558-11e3-a3ac-0800200c9a66/tasks/transfers/cddda8f0-f558-11e3-a3ac-0800200c9a66

    {
        "transfer_request":{
            "id":"cddda8f0-f558-11e3-a3ac-0800200c9a66",
            "key":"tSzpOAoUYXhuKDyugHV4",
            "target_project_id":"88cbc4c7-1dee-40be-804c-ecf86962198c",
            "description":"Transfer dev-env.example.net to Developers",
            "status":"PENDING",
            "links":{
                "self" : "/v2/zones/tasks/transfer-requests/cddda8f0-f558-11e3-a3ac-0800200c9a66"
            }
        }
    }

This creates the tranfer request, which can be accepted by the recipent as
detailed below.

GET /v2/zones/<zone-id>/tasks/transfer-requests
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This shows the outstanding transfer **FROM** the requesting project for this
zone.

.. code-block:: http

    GET /v2/zones/c11ae7e0-f558-11e3-a3ac-0800200c9a66/tasks/transfer-requests HTTP/1.1

.. code-block:: http

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

    {
        "transfer_requests":[
            {
                "transfer_request":{
                    "id":"cddda8f0-f558-11e3-a3ac-0800200c9a66",
                    "zone_id":"c11ae7e0-f558-11e3-a3ac-0800200c9a66",
                    "target_project_id":"88cbc4c7-1dee-40be-804c-ecf86962198c",
                    "description":"Transfer to Developers",
                    "status":"PENDING",
                    "links":{
                            "self" : "/v2/zones/tasks/transfer-requests/cddda8f0-f558-11e3-a3ac-0800200c9a66"
                    }
                }
            }
        ]
    }


GET /v2/zones/tasks/transfer-requests
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This shows the transfer requests for zones currently owned by the **CALLING**
users project

.. code-block:: http

    GET /v2/zones/tasks/transfer-requests HTTP/1.1

.. code-block:: http

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

    {
        "transfer-requests":[
            {
                "transfer_request":{
                    "id":"cddda8f0-f558-11e3-a3ac-0800200c9a66",
                    "zone_id":"c11ae7e0-f558-11e3-a3ac-0800200c9a66",
                    "target_project_id":"88cbc4c7-1dee-40be-804c-ecf86962198c",
                    "description":"Transfer to Developers",
                    "status":"PENDING",
                    "links":{
                        "self" : "/v2/zones/tasks/transfer-requests/cddda8f0-f558-11e3-a3ac-0800200c9a66"
                    }
                }
            },
            {
                "transfer_request":{
                    "id":"102e64fb-8fae-4c5e-9246-96a642822e03",
                    "zone_id":"05e94130-6356-4687-b5ac-36374f99bf2d",
                    "description":"Transfer *.www.example.net to web team",
                    "status":"PENDING",
                    "links":{
                        "self": "/v2/zones/tasks/transfer-requests/102e64fb-8fae-4c5e-9246-96a642822e03"
                    }
                }
            }
        ]
    }


GET /v2/zones/tasks/transfer-request/<transfer-id>
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This shows the transfer details. If the 'target_project_id' was set, this will
be restricted to that project only.

The "key" will only be shown to the project that the owns the zone being
transfered.

.. code-block:: http

    GET /v2/zones/tasks/transfer-requests/cddda8f0-f558-11e3-a3ac-0800200c9a66 HTTP/1.1

.. code-block:: http

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

    {
        "transfer_request":{
            "id":"cddda8f0-f558-11e3-a3ac-0800200c9a66",
            "key":"tSzpOAoUYXhuKDyugHV4",
            "target_project_id":"88cbc4c7-1dee-40be-804c-ecf86962198c",
            "zone_id":"c11ae7e0-f558-11e3-a3ac-0800200c9a66",
            "description":"Transfer dev-env.example.net to Developers",
            "status":"PENDING",
            "links":{
                "self": "/v2/zones/tasks/transfer-requests/cddda8f0-f558-11e3-a3ac-0800200c9a66"
            }
        }
    }


POST /v2/zones/tasks/transfer-accept/
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This is how a transfer gets accepted. The accepting project does a POST request
with the key to the tranfer URI. If successful, the API returns a loction
header to the user, with the API location of the zone.

.. code-block:: http

    POST /v2/zones/tasks/transfer-accept HTTP/1.1
    Accept: application/json
    Content-Type: application/json

    {
        "transfer_accept":{
            "key":"tSzpOAoUYXhuKDyugHV4",
            "transfer_request_id":"cddda8f0-f558-11e3-a3ac-0800200c9a66"
        }
    }


.. code-block:: http

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

    {
        "transfer_accept": {
            "id": "1c914bb0-f580-11e3-a3ac-0800200c9a66",
            "zone_id": "c11ae7e0-f558-11e3-a3ac-0800200c9a66",
            "status": "COMPLETE",
            "links": {
                "self": "/v2/zones/tasks/transfer-accept/1c914bb0-f580-11e3-a3ac-0800200c9a66",
                "zone": "/v2/zones/c11ae7e0-f558-11e3-a3ac-0800200c9a66"
        }
    }

A further GET on the transfer-request entity will show the following

.. code-block:: http

    GET /v2/zones/tasks/transfer-request/<transfer-id> HTTP/1.1
    Accept: application/json

.. code-block:: http

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8

    {
        "transfer_request":{
            "id":"cddda8f0-f558-11e3-a3ac-0800200c9a66",
            "key":"tSzpOAoUYXhuKDyugHV4",
            "target_project_id":"88cbc4c7-1dee-40be-804c-ecf86962198c",
            "zone_id":"c11ae7e0-f558-11e3-a3ac-0800200c9a66",
            "description":"Transfer dev-env.example.net to Developers",
            "status":"COMPLETE",
            "links":{
                "self": "/v2/zones/<zone-id>/tasks/transfer-requests/cddda8f0-f558-11e3-a3ac-0800200c9a66"
            }
        }
    }


DELETE /v2/zones/<zone-id>/tasks/transfer-requests/<request-id>
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This allows a user to cancel a tranfer request. This can only be performed
before the tranfer has happened, and **DOES NOT** cause a tranfer to roll back.


.. code-block:: http

    DELETE /v2/zones/tasks/transfer-requests/cddda8f0-f558-11e3-a3ac-0800200c9a66 HTTP/1.1

.. code-block:: http

    HTTP/1.1 204 No Content

Central Changes
---------------

New Central methods to allow for the CRUD of Transfer Requests.
The Central methods should use currently in place update method for changing
the project ownership of the referenced zone

Storage Changes
---------------

New Table - ZoneTransferRequestTasks
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

+-------------------+--------------------------------------+-----------+---------+--------------------------------------------------+
| Row               | Type                                 | Nullable? | Unique? | Notes                                            |
+===================+======================================+===========+=========+==================================================+
| id                | uuid                                 | No        | Yes     |                                                  |
+-------------------+--------------------------------------+-----------+---------+--------------------------------------------------+
| zone_id           | uuid                                 | No        | Yes     |                                                  |
+-------------------+--------------------------------------+-----------+---------+--------------------------------------------------+
| key               | VARCHAR                              | No        | No      |                                                  |
+-------------------+--------------------------------------+-----------+---------+--------------------------------------------------+
| description       | VARCHAR                              | Yes       | No      |                                                  |
+-------------------+--------------------------------------+-----------+---------+--------------------------------------------------+
| project_id        | uuid                                 | No        | No      | The project that created the ZoneTransferRequest |
+-------------------+--------------------------------------+-----------+---------+--------------------------------------------------+
| target_project_id | uuid                                 | Yes       | No      |                                                  |
+-------------------+--------------------------------------+-----------+---------+--------------------------------------------------+
| status            | ENUM(PENDING,COMPLETE,DELETED,ERROR) | No        | No      |                                                  |
+-------------------+--------------------------------------+-----------+---------+--------------------------------------------------+

New Table - ZoneTransferAcceptTasks
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

+---------------------+----------------------+-----------+---------+---------------------------------------------------------+
| Row                 | Type                 | Nullable? | Unique? | Notes                                                   |
+=====================+======================+===========+=========+=========================================================+
| id                  | uuid                 | No        | Yes     |                                                         |
+---------------------+----------------------+-----------+---------+---------------------------------------------------------+
| zone_id             | uuid                 | No        | Yes     |                                                         |
+---------------------+----------------------+-----------+---------+---------------------------------------------------------+
| project_id          | uuid                 | No        | No      | The ID of the project that accepted the tranfer request |
+---------------------+----------------------+-----------+---------+---------------------------------------------------------+
| transfer_request_id | uuid                 | No        | Yes     |                                                         |
+---------------------+----------------------+-----------+---------+---------------------------------------------------------+
| status              | ENUM(COMPLETE,ERROR) | No        | No      |                                                         |
+---------------------+----------------------+-----------+---------+---------------------------------------------------------+


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  https://launchpad.net/~grahamhayes

Milestones
----------

Target Milestone for completion:
  Juno-2

Work Items
==========

* Tasks Sub API
* CRUD for the two tasks (Request & Accept)
* Moving Zone to new tenant


Dependencies
============

- None

References
==========

* `Cinder Volume Transfer`_
* `Glance Functional API Discussion`_

.. _Glance Functional API Discussion: http://lists.openstack.org/pipermail/openstack-dev/2014-May/036416.html
.. _Cinder Volume Transfer: http://docs.openstack.org/user-guide/content/cli_manage_volumes.html#cli_transfer_volumes
