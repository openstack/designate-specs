..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode

..

====================
Server Pools Manager
====================

https://blueprints.launchpad.net/designate/+spec/server-pools-service

This specification outlines the Pool Manager, Central, backend driver,
and storage changes needed to support the new Pool Manager service.

Problem description
===================

Coordinating DNS operations across many different backends is difficult,
especially when there is a great number of DNS servers.  A Pool Manager
service is needed to manage the changes from the Designate database to
the many DNS servers.  A Pool Manager will also track the status of those
changes.  When this specification is implemented, a Pool Manager will
be used to manage a pool with multiple DNS servers, even if those DNS
servers are of different types.

Proposed change
===============

API Changes
-----------

None

Pool Manager Changes
--------------------

A new Designate service, called designate-pool-manager, will be created.
This is the Pool Manager.  The Pool Manager will get its configuration
from the configuration file when it is instantiated.

The configuration section is called **[service:pool_manager]**.  The options
for this section are:

+--------------------------+-------------+--------------+--------------------------------------------------------------------------------------------------------+
| **Parameter**            | **Default** | **Required** | **Notes**                                                                                              |
+==========================+=============+==============+========================================================================================================+
| *pool_name*              | 'default'   | Yes          | The pool name of the pool managed by this instance of the Pool Manager                                 |
+--------------------------+-------------+--------------+--------------------------------------------------------------------------------------------------------+
| *threshold_percentage*   | 100         | Yes          | The percentage of servers requiring a successful update for a domain change to be considered active    |
+--------------------------+-------------+--------------+--------------------------------------------------------------------------------------------------------+
| *poll_timeout*           | 30          | Yes          | The time to wait for a NOTIFY response from a name server                                              |
+--------------------------+-------------+--------------+--------------------------------------------------------------------------------------------------------+
| *poll_retry_interval*    | 2           | Yes          | The time between retrying to send a NOTIFY request and waiting for a NOTIFY response                   |
+--------------------------+-------------+--------------+--------------------------------------------------------------------------------------------------------+
| *poll_max_retries*       | 3           | Yes          | The maximum number of times minidns will retry sending a NOTIFY request and wait for a NOTIFY response |
+--------------------------+-------------+--------------+--------------------------------------------------------------------------------------------------------+
| *periodic_sync_interval* | 120         | Yes          | The time between sychronizing the servers with Storage                                                 |
+--------------------------+-------------+--------------+--------------------------------------------------------------------------------------------------------+

The Pool Manager will contain a map of the servers to instantiated
backend drivers.  The backend driver will not be responsible for reading
the configuration information as the Pool Manager will read the
global backend driver and server specific backend driver sections
from the configuration file and pass the backend driver configuration
to the backend driver for instantiation.  This map will be created when
the Pool Manager is instantiated.  Please refer to the Backend Driver
Changes section in the Storage Pools - Storage specification for more
information concerning the global backend driver and server specific
backend driver sections.

The methods in the base class for the Pool Manager service include:

create_domain(context, domain)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

+---------------+-------------------------------+--------------+
| **Parameter** | **Description**               | **Required** |
+===============+===============================+==============+
| *context*     | Security context information. | Yes          |
+---------------+-------------------------------+--------------+
| *domain*      | The designate domain object.  | Yes          |
+---------------+-------------------------------+--------------+

Return Value
""""""""""""

No return value.

Design Considerations
"""""""""""""""""""""

Loop through each server in the pool and call the backend driver to create
the domain.  For each call to the backend driver, the status is stored in the
pool_manager_status table with an action of 'CREATE' and a second row is
created with an action of 'UPDATE'.  Successful creations have a status of
'SUCCESS' and failed creations have a status of 'ERROR'.  The 'UPDATE' action
row has no initial status.  Check to see if a consensus exists using the
pool_manager_status table.  Consensus exists if the number of servers for the
domain with a successful creation exceed the *threshold_percentage*.  If
consensus exists, the Central **update_status** method is called using the
serial number used when creating the domain and a status of 'SUCCESS'.  If
consensus does not exist, the Central **update_status** method is called
using the serial number used when creating the domain and a status of 'ERROR'.

Cast vs. Call
"""""""""""""
This is an RPC cast.  Communication about the status of the domain
creation will be handled using the Central **update_status** method.

delete_domain(context, domain)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

+---------------+-------------------------------+--------------+
| **Parameter** | **Description**               | **Required** |
+===============+===============================+==============+
| *context*     | Security context information. | Yes          |
+---------------+-------------------------------+--------------+
| *domain*      | The designate domain object.  | Yes          |
+---------------+-------------------------------+--------------+

Return Value
""""""""""""

No return value.

Design Considerations
"""""""""""""""""""""

Loop through each server in the pool and call the backend driver to delete
the domain.  For each call to the backend driver, the status is stored in the
pool_manager_status table with an action of 'DELETE'.  Successful deletions
have a status of 'SUCCESS' and failed deletions have a status of 'ERROR'.
Check to see if a consensus exists using the pool_manager_status table.
Consensus exists if the number of servers for the domain with a successful
deletion exceed the *threshold_percentage*.  If consensus exists, the
Central **update_status** method is called using the serial number used when
deleting the domain and a status of 'SUCCESS'.  If consensus does not exist,
the Central **update_status** method is called using the serial number
used when creating the domain and a status of 'ERROR'.

Cast vs. Call
"""""""""""""
This is an RPC cast.  Communication about the status of the domain
deletion will be handled using the Central **update_status** method.

update_domain(context, domain)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

+---------------+-------------------------------+--------------+
| **Parameter** | **Description**               | **Required** |
+===============+===============================+==============+
| *context*     | Security context information. | Yes          |
+---------------+-------------------------------+--------------+
| *domain*      | The designate domain object.  | Yes          |
+---------------+-------------------------------+--------------+

Return Value
""""""""""""

No return value.

Design Considerations
"""""""""""""""""""""

Loop through each server in the pool and call the minidns
**notify_zone_changed** method.  Loop through each server again and call
the minidns **poll_for_serial_number** method.

Cast vs. Call
"""""""""""""
This is an RPC cast.  Communication about the status of the domain update
will be handled using the Central **update_status** method which is
called by the the Pool Manager **update_status** method.  The minidns
**poll_for_serial_number** method invokes the Pool Manager
**update_status** method when it completes.

update_status(context, domain, name_server, status, serial_number)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

+-----------------+-----------------------------------------------------------------+--------------+
| **Parameter**   | **Description**                                                 | **Required** |
+=================+=================================================================+==============+
| *context*       | Security context information.                                   | Yes          |
+-----------------+-----------------------------------------------------------------+--------------+
| *domain*        | The designate domain object.                                    | Yes          |
+-----------------+-----------------------------------------------------------------+--------------+
| *name_server*   | The name server for which this serial number is applicable.     | Yes          |
+-----------------+-----------------------------------------------------------------+--------------+
| *status*        | The status, 'SUCCESS' or 'ERROR'.                               | Yes          |
+-----------------+-----------------------------------------------------------------+--------------+
| *serial_number* | The serial number received from the name server for the domain. | Yes          |
+-----------------+-----------------------------------------------------------------+--------------+

Return Value
""""""""""""

No return value.

Design Considerations
"""""""""""""""""""""

Reads the existing serial number from the pool_manager_status table for the
server and domain.  If the new serial number > the existing serial number,
update the row and check to see if a consensus exists using the
pool_manager_status table.  Consensus exists if the number of servers for
the domain with a serial number > the existing serial number exceed the
*threshold_percentage*.  Servers are discounted from participating in the
consensus starting with the servers with the lowest serial numbers until the
minimum number of servers needed to achieve consensus based on the
*threshold_percentage* is realized.  If the existing serial number < all the
serial numbers for the remaining servers, the Central **update_status** method
is called using the lowest (consensus) serial number for those remaining
servers and a status of 'SUCCESS'.

If > 100 - *threshold_percentage* servers for the domain have a status of
'ERROR', the Central **update_status** method is called using the lowest
serial number greater than the consensus serial number (calculated above) and
a status of 'ERROR'.

Cast vs. Call
"""""""""""""
This is an RPC cast.

periodic_sync()
^^^^^^^^^^^^^^^

Return Value
""""""""""""

No return value.

Design Considerations
"""""""""""""""""""""

This method is a thread that is created when Pool Manager is instantiated.
The intent of this thread is to read the pool_manager_status table and
perform failed create, delete, and updates operations.  Additionally, the
thread will call the minidns **poll_for_serial_number** method for each
domain and server to ensure the server is synchronized with Storage.

Every *period_sync_interval*, this thread will perform the following
operations:

Read the pool_manager_status table looking for 'CREATE' actions that
have a status of 'ERROR' grouping by domains and ordering by the row
create time.  Check to see if a consensus already exists for the domain
creation.  Loop through each servers with a failed creation, using the
backend driver to attempt creation.  If consensus does not already exist,
check for consensus and call the Central **update_status** if consensus
is achieved.

Read the pool_manager_status table looking for 'DELETE' actions that
have a status of 'ERROR' grouping by domains and ordering by the row
create time.  Check to see if a consensus already exists for the domain
deletion.  Loop through each servers with a failed deletion, using the
backend driver to attempt deletion.  If consensus does not already exist,
check for consensus and call the Central **update_status** if consensus
is achieved.

For each domain in the pool, read the domain's serial number from Storage.
Loop through each server in the pool and read the pool_manager_status
table looking for 'UPDATE' actions for the domain that have a serial number
< the domain's serial number and call the minidns **notify_zone_changed**
method.

Finally, for each domain in the pool, read the domain's serial number
from Storage.  Loop through each server in the pool and call the minidns
**poll_for_serial_number** method.

Central Changes
---------------

The Central service will be updated to use the Pool Manager instead of the
backend driver.  Additionally, the default_pool_name option will be removed
from the **[service:central]** section of the Designate configuration.

All domains will be 'PENDING' status initially and calls to the Central
**update_status** method by the Pool Manager will change the status.

When creating, updating, or deleting records, records will have the serial
number field set to the new serial number of the domain.  The task will be
'ADD', 'DELETE', or 'UPDATE' corresponding to the operation.  The status
will be 'PENDING'.

Valid record states are:

+----------+------------+
| **task** | **status** |
+==========+============+
| 'ADD'    | 'PENDING'  |
+----------+------------+
| 'ADD'    | 'ERROR'    |
+----------+------------+
| 'DELETE' | 'PENDING'  |
+----------+------------+
| 'DELETE' | 'ERROR'    |
+----------+------------+
| 'UPDATE' | 'PENDING'  |
+----------+------------+
| 'UPDATE' | 'ERROR'    |
+----------+------------+
| 'NONE'   | 'ACTIVE'   |
+----------+------------+
| 'NONE'   | 'DELETED'  |
+----------+------------+

Affected code in the Central service will be updated appropriately to align
with these states.

The new method needed to update the status of domains and records is:

update_status(context, domain, status, serial_number)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

+-----------------+---------------------------------------------+--------------+
| **Parameter**   | **Description**                             | **Required** |
+=================+=============================================+==============+
| *context*       | Security context information.               | Yes          |
+-----------------+---------------------------------------------+--------------+
| *domain*        | The designate domain object.                | Yes          |
+-----------------+---------------------------------------------+--------------+
| *status*        | The status, 'SUCCESS' or 'ERROR'.           | Yes          |
+-----------------+---------------------------------------------+--------------+
| *serial_number* | The consensus serial number for the domain. | Yes          |
+-----------------+---------------------------------------------+--------------+

Return Value
""""""""""""

No return value.

Design Considerations
"""""""""""""""""""""

If the status is 'SUCCESS':

Check the status of the domain and if it has a status of 'PENDING' or 'ERROR',
set the status to 'ACTIVE'.

Check the status of records for the domain.  If they have a task of
'ADD' or 'UPDATE' and a status of 'PENDING' or 'ERROR', set the task
to 'NONE' and the status to 'ACTIVE' if the consensus serial number >= serial
number field.

Check the status of records for the domain.  If they have a task of
'DELETE' and a status of 'PENDING' or 'ERROR', set the task to 'NONE' and
the status to 'DELETED' if the consensus serial number >= serial number field.

If the status is 'ERROR':

Check the status of the domain and if it has a status of 'PENDING', set the
status to 'ERROR'.

Check the status of records for the domain.  If they have a status of
'PENDING', set the status to 'ERROR' if the consensus serial number >=
serial number field.

Cast vs. Call
"""""""""""""
This is an RPC call.

Backend Driver Changes
----------------------

The backend driver will now be instantiated with information provided by
the Pool Manager as explained in the Pool Manager Changes section.  This is
necessary because of server specific backend driver configurations.

The backend driver will continue to support the same configuration options
they currently do, only the section names will change by adding a wildcard
qualifier for the server.  For example, the backend driver section for
PowerDNS will now be **[backend:powerdns:*]**.  This syntax will denote the
global configuration for the backend driver.  This is done to allow for
server specific backend driver configurations.

The new server specific backend driver section in the configuration will be
**[backend:powerdns:<uuid>]** where uuid is a universally unique identifier.

The options for this section are:

+---------------+-------------+--------------+-----------------------------------------------+
| **Parameter** | **Default** | **Required** | **Notes**                                     |
+===============+=============+==============+===============================================+
| *host*        | None        | Yes          | The host name or IP address of the DNS server |
+---------------+-------------+--------------+-----------------------------------------------+
| *port*        | 53          | Yes          | The port of the DNS server                    |
+---------------+-------------+--------------+-----------------------------------------------+
| *tsig_key*    | None        | Yes          | The TSIG key for the DNS server               |
+---------------+-------------+--------------+-----------------------------------------------+

In addition to the above options, the server specific backend driver section
will support the same options as the backend driver global section.  If
those options are not included in the server specific backend driver section,
the server configuration will default to using the global configuration
option.  These server specific backend driver sections will support
different backends in the same pool.

The server object will be implemented.  The server object encapsulates the
server specific backend driver section in the configuration.

The following methods will not be used in the backend driver:

  * create_tsigkey(tsigkey)
  * update_tsigkey(tsigkey)
  * delete_tsigkey(tsigkey)

This is due to the only provisioner supported initially being the 'unmanaged'
provisioner.  Those methods will be used for future provisioners.

Storage Changes
---------------

A new table for the Pool Manager status will be needed.  Additionally, the
domains and records tables will be modified to support pools.  Domains
and records will be 'PENDING' status initially.  A new status 'ERROR' will
be possible for domains and records.  Finally, a record can also be
'DELETE_PENDING' and 'DELETE_ERROR'.

New Table - pool_manager_status
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

+---------------+----------------------------+-----------+---------+---------------------------------+
| Column        | Type                       | Nullable? | Unique? |  Notes                          |
+===============+============================+===========+=========+=================================+
| id            | CHAR(32)                   | No        | Yes     | PK                              |
+---------------+----------------------------+-----------+---------+---------------------------------+
| updated_at    | DATETIME                   | No        | No      | UTC time of last update         |
+---------------+----------------------------+-----------+---------+---------------------------------+
| server_id     | VARCHAR(32)                | No        | No      | Server ID                       |
+---------------+----------------------------+-----------+---------+---------------------------------+
| domain_id     | CHAR(32)                   | No        | No      | FK to ID on domains table       |
+---------------+----------------------------+-----------+---------+---------------------------------+
| status        | 'SUCCESS','ERROR'          | Yes       | No      | Status                          |
+---------------+----------------------------+-----------+---------+---------------------------------+
| serial_number | INT(11)                    | No        | No      | Serial number at time of status |
+---------------+----------------------------+-----------+---------+---------------------------------+
| action        | 'CREATE','DELETE','UPDATE' | No        | No      | Action resulting in status      |
+---------------+----------------------------+-----------+---------+---------------------------------+

Modify Table - domains
^^^^^^^^^^^^^^^^^^^^^^

+--------+--------------------------------------+-----------+---------+-----------+---------------+--------+
| Column | Type                                 | Nullable? | Unique? | Default   | Notes         | Action |
+========+======================================+===========+=========+===========+===============+========+
| status | 'ACTIVE','PENDING','DELETED','ERROR' | No        | No      | 'PENDING' | Record status | update |
+--------+--------------------------------------+-----------+---------+-----------+---------------+--------+

Modify Table - records
^^^^^^^^^^^^^^^^^^^^^^

+---------------+--------------------------------------+-----------+---------+-----------+----------------------------+--------+
| Column        | Type                                 | Nullable? | Unique? | Default   | Notes                      | Action |
+===============+======================================+===========+=========+===========+============================+========+
| serial_number | INT(11)                              | No        | No      |           | Used for the record status | add    |
+---------------+--------------------------------------+-----------+---------+-----------+----------------------------+--------+
| task          | 'ADD','DELETE','UPDATE','NONE'       | No        | No      | 'ADD'     | Record operation task      | add    |
+---------------+--------------------------------------+-----------+---------+-----------+----------------------------+--------+
| status        | 'ACTIVE','PENDING','DELETED','ERROR' | No        | No      | 'PENDING' | Record status              | update |
+---------------+--------------------------------------+-----------+---------+-----------+----------------------------+--------+

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

Primary assignee:
  https://launchpad.net/~rjrjr

Additional assignee:
  https://launchpad.net/~darshan104

Milestones
----------

Target Milestone for completion:
  Kilo-1

Work Items
----------

* Pool Manager changes
* Central changes
* Backend driver changes
* Storage changes

Dependencies
============

This specification relies on the Server Pools - Storage specification.
This specification relies on the Server Pools - MiniDNS Support specification.
