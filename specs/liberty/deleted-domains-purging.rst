This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode

=============================
 Deleted domains purging
=============================

https://blueprints.launchpad.net/designate/+spec/deleted-domains-purging

Implementation of a mechanism to routinely purge deleted domains from the
database.


Problem description
===================

Once deleted, domains are not removed immediately from the database, mostly for
billing reasons. They are flagged as deleted in the "deleted" database column
and the "deleted_at" column is populated with a timestamp.

Currently, deleted domains are purged manually by running SQL queries against
the database.
The purging process impacts the database performance and requires manual
intervention.

Proposed change
===============

This change is to implement an automated purging mechanism for deleted domains.

It also provides "purge_domain" and "purge_domains" calls to purge domains using an
arbitrary criterion.


API Changes
-----------

Central API bumps to version 5.3 with the addition of the "purge_domains" call.

Zone Manager Changes
--------------------

Implement purging as a periodic task in Zone Manager using a "domain_purge"
plugin. The task will select a group of domains and send a RPC call to Central.
Central will run a query against the database to purge any deleted domain if
needed and log the number of purged domains.

Configuration parameters:

Purging run frequency.
  Default: hourly. Users might want to run it frequently to minimize the cycle duration.

Per-cycle purging limit.
  Default: 100.

Time threshold.
  Default: 7 days.

Configuration example:

.. code-block:: cfg

    # Deleted domains purging
    #------------------------
    [zone_manager:domain_purge]
    # How frequently to purge deleted domains, in seconds
    #run_interval = 3600  # 1h
    # How many deleted domains to be purged on each run
    #limit = 100
    # How old deleted domains should be to be purged, in seconds (deleted_at column)
    #time_threshold = 604800  # 7 days


Central Changes
---------------

Implement purge_domains() method.

Storage Changes
---------------

Implement purge_domain(), purge_domains() method.

Update the _delete() method to allow forced hard delete.

Other Changes
-------------

Add a new task to setup.cfg and a new purge_domains policy item to policy.json

Add some test utility functions.

Alternatives
------------

1) No change: keeping the current manual process in place.
2) Deleting+purging the domains immediately.
3) Purging the domains on a record creation/deletion event.
4) Purging deleted domains with a single SQL query.

Further improvements
--------------------

Domains for purging are queried with a column comparison "deleted_at > $date"

Query performance can be improved by indexing the "deleted_at" column as a
B-Tree (supported by InnoDB and MyISAM):
https://dev.mysql.com/doc/refman/5.0/en/index-btree-hash.html



Implementation
==============

Assignee(s)
-----------

Primary assignee:
  https://launchpad.net/~federico-ceratto

Milestones
----------

Target Milestone for completion:
  Liberty-3

Work Items
----------

* Implement purging
* Update documentation
* Implement benchmarks

Dependencies
============

TODO
