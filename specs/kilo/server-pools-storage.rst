..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode

..

====================
Server Pools Storage
====================

https://blueprints.launchpad.net/designate/+spec/server-pools-storage

This specification outlines the Central, Storage, and compatibility changes
needed to support server pools.

Problem description
===================

For server pools, a new object is required to encapsulate pools.  When
this specification is implemented, the pool object will used by the backend
drivers for getting the name servers as a compatibility change.  Central
will continue to call the backend drivers.  Additional server pools
specifications will address using the entire pool object.  This is the
foundation for those other specifications.

Proposed change
===============

When the database is first initialized, a pool named 'default' will be
created.  This is the default pool.  The default pool will have no
also-notify hosts and no name servers.  New methods will be added to
Central to create, find, get, update, and delete pools and these methods
are described in the Central Changes section.  The database changes needed
to support pools are described in the Storage Changes section.  Finally,
name servers can be added to the default pool by using the V1 servers
endpoint described in the Compatibility Changes section.

API Changes
-----------

None

Central Changes
---------------

The **[service:central]** section of the Designate configuration will have a
new option:

+---------------------+-------------+--------------+-----------------------------------+
| **Parameter**       | **Default** | **Required** | **Notes**                         |
+=====================+=============+==============+===================================+
| *default_pool_name* | 'default'   | Yes          | The pool name of the default pool |
+---------------------+-------------+--------------+-----------------------------------+

This value needs to be set to the pool name of the default pool which is
'default'.

The pool object will be implemented.

The new methods implemented in the Central service for the
pool object are:

create_pool(context, pool)
^^^^^^^^^^^^^^^^^^^^^^^^^^

+---------------+-------------------------------+--------------+
| **Parameter** | **Description**               | **Required** |
+===============+===============================+==============+
| *context*     | Security context information. | Yes          |
+---------------+-------------------------------+--------------+
| *pool*        | The designate pool object.    | Yes          |
+---------------+-------------------------------+--------------+

Return Value
""""""""""""

The created pool.

find_pools(context, criterion, marker, limit, sort_key, sort_dir)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

+---------------+-------------------------------------------------------------------+--------------+
| **Parameter** | **Description**                                                   | **Required** |
+===============+===================================================================+==============+
| *context*     | Security context information.                                     | Yes          |
+---------------+-------------------------------------------------------------------+--------------+
| *criterion*   | The search criterion.                                             | No           |
+---------------+-------------------------------------------------------------------+--------------+
| *marker*      | Resource ID from which after the requested page will start after. | No           |
+---------------+-------------------------------------------------------------------+--------------+
| *limit*       | Integer limit of objects of the page size after the marker.       | No           |
+---------------+-------------------------------------------------------------------+--------------+
| *sort_key*    | Key from which to sort after.                                     | No           |
+---------------+-------------------------------------------------------------------+--------------+
| *sort_dir*    | Direction to sort after using sort_key.                           | No           |
+---------------+-------------------------------------------------------------------+--------------+

Return Value
""""""""""""

The found pools.

get_pool(context, pool_id)
^^^^^^^^^^^^^^^^^^^^^^^^^^

+---------------+-------------------------------+--------------+
| **Parameter** | **Description**               | **Required** |
+===============+===============================+==============+
| *context*     | Security context information. | Yes          |
+---------------+-------------------------------+--------------+
| *pool_id*     | The pool ID.                  | Yes          |
+---------------+-------------------------------+--------------+

Return Value
""""""""""""

The pool requested.

update_pool(context, pool)
^^^^^^^^^^^^^^^^^^^^^^^^^^

+---------------+-------------------------------+--------------+
| **Parameter** | **Description**               | **Required** |
+===============+===============================+==============+
| *context*     | Security context information. | Yes          |
+---------------+-------------------------------+--------------+
| *pool*        | The designate pool object.    | Yes          |
+---------------+-------------------------------+--------------+

Return Value
""""""""""""

The updated pool.

delete_pool(context, pool_id)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

+---------------+-------------------------------+--------------+
| **Parameter** | **Description**               | **Required** |
+===============+===============================+==============+
| *context*     | Security context information. | Yes          |
+---------------+-------------------------------+--------------+
| *pool_id*     | The pool ID.                  | Yes          |
+---------------+-------------------------------+--------------+

Return Value
""""""""""""

No return value.

Storage Changes
---------------

A new table for pools will be needed.  Another new table for the pool
attributes will be used to store the also-notify and name server values in
addition to supporting future enhancements to server pools.  The domains
table will be modified to support pools.

New Table - pools
^^^^^^^^^^^^^^^^^

+-------------+--------------+-----------+---------+-----------------------------------------------------+
| Column      | Type         | Nullable? | Unique? | Notes                                               |
+=============+==============+===========+=========+=====================================================+
| id          | CHAR(32)     | No        | Yes     | PK                                                  |
+-------------+--------------+-----------+---------+-----------------------------------------------------+
| created_at  | DATETIME     | Yes       | No      | UTC time of creation                                |
+-------------+--------------+-----------+---------+-----------------------------------------------------+
| updated_at  | DATETIME     | Yes       | No      | UTC time of last update                             |
+-------------+--------------+-----------+---------+-----------------------------------------------------+
| version     | INT(11)      | No        | No      | Designate API version                               |
+-------------+--------------+-----------+---------+-----------------------------------------------------+
| name        | VARCHAR(50)  | No        | Yes     | Server pool name                                    |
+-------------+--------------+-----------+---------+-----------------------------------------------------+
| description | VARCHAR(255) | Yes       | No      | Server pool description                             |
+-------------+--------------+-----------+---------+-----------------------------------------------------+
| project_id  | VARCHAR(36)  | No        | No      | Tenant ID that created the pool                     |
+-------------+--------------+-----------+---------+-----------------------------------------------------+
| provisioner | ENUM         | No        | No      | Only 'unmanaged' provisioner is supported currently |
+-------------+--------------+-----------+---------+-----------------------------------------------------+

New Table - pool_attributes
^^^^^^^^^^^^^^^^^^^^^^^^^^^

+------------+--------------+-----------+---------+--------------------------------------------------------+
| Column     | Type         | Nullable? | Unique? | Notes                                                  |
+============+==============+===========+=========+========================================================+
| id         | CHAR(32)     | No        | Yes     | PK                                                     |
+------------+--------------+-----------+---------+--------------------------------------------------------+
| key        | VARCHAR(255) | No        | No      | Pool attribute key (also-notify, name server, etc.)    |
+------------+--------------+-----------+---------+--------------------------------------------------------+
| value      | VARCHAR(255) | No        | No      | Pool attribute value                                   |
+------------+--------------+-----------+---------+--------------------------------------------------------+
| pool_id    | CHAR(32)     | No        | No      | FK to ID on pools table                                |
+------------+--------------+-----------+---------+--------------------------------------------------------+

Modify Table - domains
^^^^^^^^^^^^^^^^^^^^^^

+---------+----------+-----------+---------+-------------------------+--------+
| Column  | Type     | Nullable? | Unique? | Notes                   | Action |
+=========+==========+===========+=========+=========================+========+
| pool_id | CHAR(32) | No        | Yes     | FK to ID on pools table | add    |
+---------+----------+-----------+---------+-------------------------+--------+

Compatibility Changes
---------------------

When adding a name server using the V1 servers endpoint, a pool_attributes
table entry will be created for the name server.  The columns and values used
will be:

+-----------------------+-----------------------+
| Column                | Value                 |
+=======================+=======================+
| key                   | 'name_server'         |
+-----------------------+-----------------------+
| value                 | <FQDN of name server> |
+-----------------------+-----------------------+
| pool_id               | <default pool id>     |
+-----------------------+-----------------------+

When creating a domain using the V1 domains endpoint, the domains table entry
will include the pool ID of the default pool.

The backend drivers will be modified to use the name servers defined for the
default pool in the pool_attributes table instead of the servers table.

All Central methods for servers will be removed.

The servers table will be removed and all Storage methods for servers will
be removed.

The existing server object will be removed.

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
* Central changes
* Storage changes
* Compatibility changes

Dependencies
============

None
