..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode

=============================
Bulk zone update throttling
=============================

https://blueprints.launchpad.net/designate/+spec/notify-throttling

Implement a mechanism to throttle the delivery of NOTIFY transactions when
a large number of zones are updated at the same time.


Problem description
===================

If a large number of zones are updated in a short time this will generate a
consequently large amount NOTIFY transaction to be sent to the nameservers
with no delay leading to a burst of incoming AXFR requests.
This might impact on bottlenecks in MiniDNS and the storage layer in terms of
CPU, I/O or network bandwidth.

A typical trigger is the update of an NS record in a Pool containing many zones.

The autonomous refreshing of zones performed by resolvers can also trigger a
similar burst of AXFR. This can happen on recently started resolvers, where the
refresh timers can share the same values across many zones.

Related to bug https://bugs.launchpad.net/designate/+bug/1498462

Proposed change
===============

Implement a mechanism for enqueuing and delayed delivery of notify transactions
at a configurable throttle speed.

Also, implement staggering of zone refresh requests by randomizing the refresh
interval.

API Changes
-----------

Expose the count of zones flagged for delayed notify in the Admin
API as "/reports/counts/zones_pending_notify".

Central Changes
---------------

Implement support for a new database column "pending_notify" and set it to
True every time a Pool NS record is updated.

Storage Changes
---------------

Add an new boolean database column "pending_notify" on Zones.
Implement a migration script to add the column to existing databases,
defaulting to False. In future, the column might default to True.

Other Changes
-------------

Implement a Task in Zone Manager to periodically fetch a set of zones that need
to receive a Notify starting with the oldest in term of last update time.
The task frequency and the maximum set size can be configured to throttle the
amount of outgoing Notify.
Zone Manager will reset the "pending_notify" flag once done.

Alternatives
------------

N/A

Implementation
==============

The throttling queue is implemented as a new database column containing a
boolean flag. See Central Changes and Storage Changes.

Also, new zones will be created with an uniformly random refresh time between a minimum and a maximum value.


Design considerations
---------------------

The throttling queue could be implemented outside of the database:
- No need to create an extra database column
- No increased database I/O

We propose using the database for the following reasons:
- Zone Manager is the best candidate to handle the delayed Notify. Currently there are no ways for Central to send a list of Zones to Zone Manager other than through the database
- The queue can support delayed Notify for changes other than Pool NS record updates
- Ability to monitor the queue size and ETA to inform the user and for debugging
- A persistent queue can survive Zone Manager unhandled exceptions or restarts
- The increased database load is negligible compared to the existing traffic

Risk analysis
-------------

- Zone Manager fails to run the Notify delivery task. The nameservers will eventually refresh the zone anyways. Impact: slow update propagation. Mitigation: expose the notification queue length to the user through Admin API and by logging.
- A big notification queue takes a considerable time to be handled. Impact: potentially prevents more urgent changes to be delivered quickly. Mitigation: encourage users to configure the throttling parameters; Provide sensible default values. Implementing a concept of notification priority seems unnecessary.

Assignee(s)
-----------

Primary assignee:
  Federico Ceratto https://launchpad.net/~federico-ceratto

Milestones
----------

Target Milestone for completion:
  Liberty-3

Work Items
----------

- Implement refresh time staggering
- Implement Notify throttling
- Add throttle parameters to configuration files
- Document throttling mechanism
- Write unit and functional tests
- Test throttling and staggering on devstack

Dependencies
============

N/A
