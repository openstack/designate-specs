..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode

..

==========================================
Get the Zone transfer accepts list
==========================================

https://blueprints.launchpad.net/designate/+spec/zone-transfer-accept-list

Currently listing of all the accepted zone transfer is not supported by the designate API.

This bp adds the support request that will enable users to view the list of all the accepted zone
transfers.

Problem description
===================

A Zone Transfer is the term used to refer to the transferring of ownership of a zone from one tenant
say A over to another tenant B.

The /zones/tasks/transfer_accepts API provides a way to list all the zone transfer accepts that will
enable users to view all the accepted zone ownership transfers.

Proposed change
===============


API Changes
-----------

Expose the list of accepted zones transfer requests in the v2
API as "/zones/tasks/transfer_accepts".


GET /v2/zones/tasks/transfer_accepts
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This generates the list of accepted zone transfer requests


**Example Request:**

.. code-block:: http

    GET  /v2/zones/tasks/transfer_accepts HTTP/1.1
    Host: 127.0.0.1
    Accept: application/json
    Content-Type: application/json

**Example Response:**

.. code-block:: http

    HTTP/1.1 200 OK
    Content-Length: 649
    Content-Type: application/json; charset=UTF-8

        {
               "transfer_accepts": [
                   {
                      "status": "COMPLETE",
                      "zone_id": "417fb114-df97-46e5-ba34-6f5cee925e49",
                      "links": {
                           "self": "http://127.0.0.1:9001/v2/zones/tasks/transfer_accepts/bf92e7c8-8e21-40c7-8076-8285b035b1ad",
                           "zone": "http://127.0.0.1:9001/v2/zones/417fb114-df97-46e5-ba34-6f5cee925e49"
                      },
                      "created_at": "2016-02-04T04:01:18.000000",
                      "updated_at": "2016-02-04T04:01:18.000000",
                      "key": null,
                      "project_id": "0469230b787b4154922c2a4a4b5fbcaa",
                      "id": "bf92e7c8-8e21-40c7-8076-8285b035b1ad",
                      "zone_transfer_request_id": "17bf5ba6-053f-4f1e-87c8-a2f8c3585d8f"
                   }
               ]
               "links": {
                   "self": "http://127.0.0.1:9001/v2/zones/tasks/transfer_accepts"
               },
               "metadata": {
                    "total_count": 1
               },
        }


Central Changes
---------------

None

Storage Changes
---------------

None

Other Changes
-------------
Cli Impact:

A new cli will also  be added that will enable users to list the status of
all the accepted zone transfer i.e.

   openstack zone transfer accept list [-h]
                                       [-f {csv,json,table,value,yaml}]
                                       [-c COLUMN]
                                       [--max-width <integer>]
                                       [--noindent]
                                       [--quote {all,minimal,none,nonnumeric}]

   List accepted zone transfers

   optional arguments:
     -h, --help            show this help message and exit

   output formatters:
     output formatter options

     -f {csv,json,table,value,yaml}, --format {csv,json,table,value,yaml}
                        the output format, defaults to table

     -c COLUMN, --column COLUMN
                        specify the column(s) to include, can be repeated

   table formatter:
     --max-width <integer>
                        Maximum display width, 0 to disable

   json formatter:
     --noindent            whether to disable indenting the JSON

   CSV Formatter:
     --quote {all,minimal,none,nonnumeric}
                        when to include quotes, defaults to nonnumeric


Alternatives
------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Sonu kumar https://launchpad.net/~sonu-bhumca11

IRC Nick Name:
  sonuk

Milestones
----------

Target Milestone for completion:
  Newton

Work Items
----------

* Add API changes
* Implement CLI changes
* Add the documentation for the same

Dependencies
============

Reference:
  https://bugs.launchpad.net/python-designateclient/+bug/1499539
