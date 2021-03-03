..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode

===============
 Alias Records
===============

https://blueprints.launchpad.net/designate/+spec/alias-records

ALIAS records (aka ANAME's, root CNAME's, root domain redirect, zone
apex CNAME's) are a relatively new "addition" to DNS. These pseudo
Resource Records (RR) are used to bypass restrictions placed within
the DNS protocol that forbid the presence of a CNAME RR next to any
other RR. This prevents some modern usage of DNS, for example, many
CDNs or other static-content hosting sites like GitHub Pages will
require users to CNAME e.g. "www.example.org." towards their frontend
load balancers. A CNAME is used to ensure the provider can easily make
infrastructure changes without requiring every customer to implement
DNS changes in-sync with their deployment schedule.

As this is not a true feature within the DNS protocol, support for
ALIAS RR's must be implemented entirely within the end users
authoritative DNS infrastructure.


Problem description
===================

DNS RFCs do not support a CNAME record set for the root domain:

.. code-block:: text

  example.com IN CNAME example.github.com

As a result, this capability is not widely supported by DNS backends.



Proposed change
===============

Designate will expose a new "ALIAS" record set type via API; the
record set will be flattened on a periodic basis into an "A" or "AAAA"
record set for propagation to DNS backends. This solution is
backend-independent and complies with RFCs implementations.

Acceptance Criteria
-------------------
- An end user can create an "ALIAS" record set in the API and see that
  record set through all other API requests (e.g. get record sets,
  export zone, import zone)
- All "ALIAS" records will be flattened to "A/AAAA" record sets for
  all backends.
- An IP change to example.github.com will result in an update the IP
  address of the flattened "A" or "AAAA" record sets available on the
  backend.
- Operators can configure a custom caching name sever to use for
  resolution and flattening of ALIAS record sets.
- Operators can set the polling interval for the flattening process.

.. note:: Not recommended for use with CDN implementations.


API Changes
-----------

RRSets with the "visible" value (described below) set to "mdns" will
be filtered from view within the API.

Additionally, ALIAS RRSets will not be visible in the V1 API. Instead,
a dynamically generated TXT record will be included to indicate the
existence of the ALIAS record that is only editable via the V2
API.


**Example ALIAS record set creation request:**

.. code-block:: http

        POST /v2/zones/2150b1bf-dee2-4221-9d85-11f7886fb15f/recordsets HTTP/1.1
        Host: 127.0.0.1:9001
        Accept: application/json
        Content-Type: application/json

        {
          "name" : "example.org.",
          "description" : "This is an example ALIAS record set.",
          "type" : "ALIAS",
          "ttl" : 3600,
          "records" : [
            "example.github.com."
          ]
        }


Flatten Task
~~~~~~~~~~~~

The process of "flattening" the ALIAS record can be triggered by the
user. For example, if a user starts getting error messages the
resolution isn't working, the user can make an API request to force an
update.

**Example Manual Flatten Request:**

.. code-block:: http

   POST /v2/zones/<zone_id>/recordsets/<recordset_id>/tasks/flatten HTTP/1.1
   Host: 127.0.0.1:9001
   Accept: application/json


The request body should be empty. Also, it is recommended the user
configure rate limits on this endpoint accordingly. This rate limiting
is outside the scope of this spec.


Exporting
~~~~~~~~~

When a zone is exported as a zone file, the export should contain two
lines, the ALIAS record, commented out, and the most recent A record.

For example:

::

   ; example.com. IN ALIAS example.org.
   example.com.  IN  A     192.0.2.1

This allows the user to have a valid zone file to import, while
providing visibility that the A record was generated from an ALIAS
record.


Central Changes
---------------

Central's create and update RecordSet methods will be updated to call
a new RPC method implemented on the Zone Manager service to trigger
immediate ALIAS flattening when ALIAS RRSets are created /
updated. If the resolution fails, the record will be placed in the
`ERROR` status.

Central's delete RecordSet method will be updated to remove the
associated flattened `A` and `AAAA` RecordSets - the mechanism for
identifying these related RRSets is detailed in Zone Manager Changes.

Additionally, ALIAS RRSets will be treated somewhat similarly to
CNAMEs. It will be invalid to place an ALIAS RRSet next to an A or
AAAA RecordSet. The recordset placement validation will be updated to
handle this case.


MiniDNS Changes
---------------
MiniDNS will be updated to display RRSets with the "visible" field
(described below) set to 'mdns' or 'all', ensuring no attempt is made
to include non-RFC compliant RRSets within `AXFRs`.


Zone Manager Changes
--------------------

First, for each zone it manages, `designate-zone-manager` service will
find any ALIAS recordsets associated with the zone. The zone manager
will then issue a successful A/AAAA DNS query to get the IP
addresses associated with the ALIAS target. Designate's database
will be updated to the "real" values, incrementing the SOA serial to
trigger an AXFR towards the public facing nameservers.

If this query fails and the ALIAS target can't be resolved, the
A/AAAA records will not be updated and the ALIAS RR will be put in
STALE status.

The interval at which ALIAS records are flattened will be
configurable, defaulting to 1800 seconds (30 minutes).

Finally, a new RPC method will be implemented to trigger immediate
ALIAS flattening for a specific Zone, allowing for newly created ALIAS
RR's to be usable before the first periodic interval is hit. This RPC
method will also be utilized by the ALIAS RRset flattening task
endpoint.


Storage Changes
---------------

Recordsets: update column "type"
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Add "ALIAS" to "type".


Recordsets: New Column
~~~~~~~~~~~~~~~~~~~~~~

+---------+----------------------------+------+-----+---------+---------+
| Field   | Type                       | Null | Key | Default | Extra   |
+---------+----------------------------+------+-----+---------+---------+
| visible | enum('all', 'api', 'mdns') | NO   | MUL | 'all'   |         |
+---------+----------------------------+------+-----+---------+---------+


Sample data
-----------

For the example scenario described above, corresponding database values are the
following (columns omitted for brevity):


Recordsets Table
~~~~~~~~~~~~~~~~
+------+-----------+---------------------+-----------+-------+---------+
| id   | domain_id | name                | tenant_id | type  | visible |
+------+-----------+---------------------+-----------+-------+---------+
| 1230 | 768       | example.github.com. | 391       | ALIAS | 'api'   |
+------+-----------+---------------------+-----------+-------+---------+
| 0100 | 768       | example.org.        | 391       | A     | 'mdns'  |
+------+-----------+---------------------+-----------+-------+---------+


Records Table
~~~~~~~~~~~~~
+------+--------------+--------------+---------+-----------------------+---------------------+
| id   | recordset_id | data         | managed | managed_resource_type | managed_resource_id |
+------+--------------+--------------+---------+-----------------------+---------------------+
| 3211 | 1230         | example.org. | 0       | NULL                  | NULL                |
+------+--------------+--------------+---------+-----------------------+---------------------+
| 3212 | 0100         | 192.168.1.10 | 1       | ALIAS                 | 3211                |
+------+--------------+--------------+---------+-----------------------+---------------------+


Alternatives
------------

Some DNS backends support flattening processes (e.g. PowerDNS). An
alternative implementation is to create a new record set type called
"ALIAS" that integrates with each respective backend's implementation.


Implementation
==============

**Assignee(s)**

Primary assignee:
  Eric Larson <eric.larson@rackspace.com>

**Milestones**

Target Milestone for completion:
  Mitaka-1

**Work Items**

* Implement support for hiding RRSets in the API and MiniDNS
* Implement the periodic ALIAS flattening within the `designate-zone-manager`
  service
* Update Central's CUD RecordSet methods for ALIAS support


Dependencies
============

None
