..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode

=============================
New Agent
=============================

This spec describes a proposed service for Designate, tentatively called the
"Agent," not to be confused with the current component with the same name.

Terminology
===========

+----------+---------------------------------------------+
| Term     | Meaning                                     |
+==========+=============================================+
| AXFR     | A DNS zone transfer                         |
+----------+---------------------------------------------+
| NOTIFY   | A DNS protocol to inform of a zone update   |
+----------+---------------------------------------------+
| OPCODE   | Part of a DNS message that informs action   |
|          | http://bit.ly/1pfG8Zp                       |
+----------+---------------------------------------------+
| Backend  | Different code paths for various DNS servers|
+----------+---------------------------------------------+
| MiniDNS  | Designate component in charge of sending    |
|          | NOTIFYs, and answering AXFRs.               |
+----------+---------------------------------------------+

Problem description
===================

MiniDNS provides a very elegant solution for many DNS installations. Having a
Designate component as the DNS master enables Designate to have a great amount
of control over the DNS server, and makes certain processes much easier to
implement across backends.

However, some deployments may not be able to use MiniDNS as a true DNS Master,
and would rather keep the current style of backends in some form with the added
benefits of MiniDNS. They might need a more specialized type of interaction
between Designate and the DNS server, making Designate more flexible.
Additionally, there are potential problems with having MiniDNS and it's database
being a single point of failure for a DNS infrastructure.

Proposed change
===============

An agent deployed on the Master DNS server that interacts with MiniDNS solves
these problems. A plugable backend that enables users to use any backend with
MiniDNS, as well as the ability to isolate MiniDNS from being a master in
large deployments are much simpler with an Agent.

The Agent would be a standalone service that acts as a sort of mirror to
MiniDNS. It would receive AXFR/IXFR, NOTIFYs, and other notifications and
perform changes to the DNS server through a plugin-style backend.

Pool Manager Changes
--------------------

A new backend for the Pool Manager would be created for the Agent.
It should send fire-and-forget DNS messages with classes and RRTYPE intended for
private use, shown here: http://bit.ly/1soCZTffor on Creates/Deletes, and call
into MiniDNS as it would normally for updates.

Agent Changes
-------------

The agent service will need to be created (possibly supplanting the old "agent")

The structure will be very similar to MiniDNS. A service (TCP, rather than RPC)
will listen for TCP and UDP traffic. NOTIFYs and AXFRs will be handled, along
with regular DNS queries (via pass through to the real DNS server), and the
special DNS CLASSES and RRDTYPEs that are chosen to signify Creates and Deletes.
DNS traffic will result in calls into the configured backend. A special OPCODE,
14 will be used for all non-standard actions like Creating and Deleting zones.

Configuration
^^^^^^^^^^^^^

+--------------------------+-------------+--------------+--------------------------------------------------------------------------------------------------------+
| **Parameter**            | **Default** | **Required** | **Notes**                                                                                              |
+==========================+=============+==============+========================================================================================================+
| *backend_driver*         | 'None'      | Yes          | The name of the backend driver to use with the Agent                                                   |
+--------------------------+-------------+--------------+--------------------------------------------------------------------------------------------------------+
| *workers*                | None        | No           | The number of worker processes to spawn                                                                |
+--------------------------+-------------+--------------+--------------------------------------------------------------------------------------------------------+
| *masters*                | []          | Yes          | List of IP/Ports of Masters the Agent will query for AXFRs and SOA queries                             |
+--------------------------+-------------+--------------+--------------------------------------------------------------------------------------------------------+
| *allow-notify*           | []          | Yes          | List of IP/Ports of Masters allowed to NOTIFY/AXFR with the Agent                                      |
+--------------------------+-------------+--------------+--------------------------------------------------------------------------------------------------------+
| *host*                   | '0.0.0.0'   | Yes          | Bind host for the Agent                                                                                |
+--------------------------+-------------+--------------+--------------------------------------------------------------------------------------------------------+
| *port*                   | 5354        | Yes          | Port to bind to                                                                                        |
+--------------------------+-------------+--------------+--------------------------------------------------------------------------------------------------------+
| *tcp_backlog*            | 100         | Yes          | TCP backlog for the Agent                                                                              |
+--------------------------+-------------+--------------+--------------------------------------------------------------------------------------------------------+

There would also be sub-configuration sections for the different backends, using
the form [agent:backend_name]. These would be mostly ported from the current
backends.

Service
^^^^^^^

Initialization of the service, and basic handling of TCP/UDP traffic. Does some
validation on the traffic coming in, making sure it is valid before sending on
to the core application logic. This should be mostly taken from
designate/mdns/service.py
There could be some optimization, since the Agent will be talking only to mdns.

Middleware
^^^^^^^^^^

A thin middleware that sits between the TCP message receiving, and the handler.
Allows for contexts or other metadata to be attached to a request before it goes
to a handler.

Handler
^^^^^^^

The handler will take DNS requests in, and take the appropriate action. This
will be very similar to mdns/handler.py

_handle_update(request, op)
"""""""""""""""""""""""""""

+---------------+-----------------------------------------------+--------------+
| **Parameter** | **Description**                               | **Required** |
+===============+===============================================+==============+
| *request*     | Serialized request data from the DNS request  | Yes          |
+---------------+-----------------------------------------------+--------------+
| *op*          | Whether an update or create is happening      | Yes          |
+---------------+-----------------------------------------------+--------------+

1. Get the name from the request
2. Make sure the requester is in the allowed list of notifiers
3. Call the backend's find_domain to see if the message is for a zone on the DNS
   server. If not, throw the message away
4. Query the requesting mdns (SOA), per the RFC (????)
5. Call out to the Agent's AXFR module asynchronously which will perform the
   AXFR, get the zone and call backend's create or update domain, based on the
   action.

_handle_delete(request)
"""""""""""""""""""""""

+---------------+-----------------------------------------------+--------------+
| **Parameter** | **Description**                               | **Required** |
+===============+===============================================+==============+
| *request*     | Serialized request data from the DNS request  | Yes          |
+---------------+-----------------------------------------------+--------------+

1. Get the name from the request
2. Make sure the requester is in the allowed list of notifiers
3. Call out to the backend's delete_domain

_handle_record_query(request)
"""""""""""""""""""""""""""""

+---------------+-----------------------------------------------+--------------+
| **Parameter** | **Description**                               | **Required** |
+===============+===============================================+==============+
| *request*     | Serialized request data from the DNS request  | Yes          |
+---------------+-----------------------------------------------+--------------+

1. Repackage the request, send it along to the local DNS server
2. Return the results

This is a possible way for the Agent to answer DNS queries when MiniDNS is
polling for changes

AXFR
^^^^

The AXFR module will send the AXFR query to one of the masters specified in the
config for the zone name that was passed in. Based on the type of query that
called for the AXFR, a backend call will then be made with the information
gathered from the AXFR.

_do_axfr(zone_name, action)
"""""""""""""""""""""""""""

+---------------+-----------------------------------------------+--------------+
| **Parameter** | **Description**                               | **Required** |
+===============+===============================================+==============+
| *zone_name*   | The zone name to ask for the AXFR with        | Yes          |
+---------------+-----------------------------------------------+--------------+
| *new_domain*  | Boolean value to inform AXFR if the zone is a | No           |
|               | new zone or not. Default is False             |              |
+---------------+-----------------------------------------------+--------------+

1. Pick a master from the config file
2. Send the AXFR query
3. Parse the response into a designate domain object
4. Call the backend for the specified action (Create, Update)

.. note:: Eventually this module could be renamed "Transfer" and do IXFRs.

Backend
^^^^^^^

The Backend module will house a base plugin, and a variety of plugins with a
similar interface to the Pool Manager. The intent is to take the zone changes
gleaned from an AXFR (or IXFR) and apply them to the DNS server. The manner of
accomplishing this will vary widely for each DNS server, you might be editing a
flat file, or making database calls, or some other method. The following methods
would compose the base plugin.

find_zone(zone_name)
""""""""""""""""""""

+---------------+-----------------------------------------------+--------------+
| **Parameter** | **Description**                               | **Required** |
+===============+===============================================+==============+
| *zone_name*   | The zone name to be searched for              | Yes          |
+---------------+-----------------------------------------------+--------------+

When a NOTIFY comes in for a zone, the Agent must first check to make sure that
the zone is valid for the DNS server. Otherwise, it might do an AXFR and try to
update a zone that doesn't exist on the server. It's possible that this check
could be incorporated in to the update_zone function, but it seems more
efficient to do it here.

create_zone(domain)
"""""""""""""""""""

+---------------+-----------------------------------------------+--------------+
| **Parameter** | **Description**                               | **Required** |
+===============+===============================================+==============+
| *domain*      | A Designate domain object to create on the    | Yes          |
|               | backend server, including it's recordsets     |              |
+---------------+-----------------------------------------------+--------------+

Take the appropriate measure to create the zone on the DNS server. This object
will hold all the necessary information to serve the zone.

update_zone(domain)
"""""""""""""""""""

+---------------+-----------------------------------------------+--------------+
| **Parameter** | **Description**                               | **Required** |
+===============+===============================================+==============+
| *domain*      | A Designate domain object to update on the    | Yes          |
|               | backend server, including it's recordsets     |              |
+---------------+-----------------------------------------------+--------------+

Take the appropriate measure to update the zone on the DNS server. This has the
potential to be very similar to the create_zone logic if there is no way to
discern the differences between this object, and the zone on the server.

delete_zone(zone_name)
""""""""""""""""""""""

+---------------+-----------------------------------------------+--------------+
| **Parameter** | **Description**                               | **Required** |
+===============+===============================================+==============+
| *zone_name*   | The zone name identified for deletion         | Yes          |
+---------------+-----------------------------------------------+--------------+

Take the appropriate measure to delete the zone on the DNS server, and ideally
all of it's subresources.

Other Changes
-------------

This should fit into the Designate pattern well. To the Pool Manager and
MiniDNS, the Agent is the same as any other DNS server. It's possible that
something in MiniDNS could be supplemented to use with the Agent, but it
shouldn't be needed.

A lot of the code that MiniDNS uses for the actual DNS protocol stuff will
be reused in some form in the Agent. Development of the agent would be a good
time to port the commonalities into the dnsutils module of Designate.

Benefits
========

- Configurable backends that can do the work required for different DNS servers
  (RNDC, Database addition) that don’t connect directly to the database
- A deployment with the benefits of MiniDNS, while keeping a traditional
  Master/Slave DNS setup is possible
- Less DNS servers must be managed by Designate
- Less chatter between MiniDNS and the database because SOA refreshes and
  other queries can be handled by a master
- MiniDNS becomes less vital to a deployment, MiniDNS/Database issues are
  isolated from the DNS infrastructure
- An agent with direct control of the DNS servers adds the benefit of doing
  things outside of pure DNS protocol with a DNS server, periodic syncs, etc are
  made easier

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  tim-simmons-t

Milestones
----------

Target Milestone for completion:
  Kilo

Work Items
----------

- Create the Agent service
- Add support for receiving NOTIFYs
- Add support for receiving AXFRs
- Decide and implement receiving messages with non-standard CLASS/RRDATA
- Add a base class backend that is called for different operations
- Port some of the existing backends, or add some new ones
