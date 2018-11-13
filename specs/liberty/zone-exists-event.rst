..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode

===================
 Zone Exists Event
===================

https://blueprints.launchpad.net/designate/+spec/zone-exists-event

Problem description
===================

Designate emits several events for use by Ceilometer for Metering & Billing
purposes. Currently, these events are sent for zone/rrset/etc create, update
and delete actions. As these events are emitted over RabbitMQ, we cannot
guarantee delivery. In order to ensure accurate billing for customers, we need
to provide a mechanism for identifying "lost" events. This is particularity
important for Zone create/delete events, as per-Zone/per-Timeperiod is the
most common billing unit.

The accepted standard solution to the loss of create/delete events is for each
service to emit a periodic "exists" event, this allows Ceilometer to, for
example, identify that N exists events in a row were not received, surmising
that a "delete" event was missed.

In other services like Nova there is a clear "owner" for each core resource
(e.g. an instance is owned by the compute node it resides on), this allows
each nova-compute instance to periodically emit a `compute.instance.exists`
event for a small number of instances. Designate has no such clear owner,
and many thousands of zones may belong to a single pool (the smallest
grouping we currently have available). This poses a problem, in that emitting
100's of thousands of events, or more, each hour may be too demanding for the
active pool manager.

Proposed change
===============

We will introduce a new service `designate-zone-manager` which will handle all
periodic tasks relating to the zones it is responsible for. Initially, this
will only be the periodic `dns.domain.exists` event, but over time we will add
additional tasks, for example, polling of Secondary zones at their refresh
intervals.

A concept of "zone shards" will be introduced, where every zone will be
allocated to a shard based on the first three characters of the zones UUID.
This will provide for 4,096 distinct shards to be distributed over the set of
available `designate-zone-manager` processes, ensuring that no single
`designate-zone-manager` is responsible for more zones than it can reasonably
handle. With 5 million zones, each shard should contain approx 1.2k zones.

Finally, distribution of shards to available `designate-zone-manager`
processes will be handled by the `OpenStack Tooz
<https://github.com/openstack/tooz>`_ library. Tooz provides the building
blocks required to implement a "Partitioner", using it's Group Membership
APIs to divvy up the available shards between active workers, including
dynamically re-allocating based on membership changes.

API Changes
-----------

None

Central Changes
---------------

None

Storage Changes
---------------

A new column will be added to the domains, recordsets and records tables, this
column will be populated by the storage driver with the integer representation
of the first 3 characters of the UUID, giving a whole number with a value
between 0 and 4095. We add the shard to the RecordSets/Records tables now,
as we know it will be used in a future blueprint (ALIAS records).

Domains Table Additions
^^^^^^^^^^^^^^^^^^^^^^^

+-------+--------+-----------+---------+
| Row   | Type   | Nullable? | Unique? |
+=======+========+===========+=========+
| shard | int(2) | No        | No      |
+-------+--------+-----------+---------+

RecordSets/Records Table Additions
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

+--------------+--------+-----------+---------+
| Row          | Type   | Nullable? | Unique? |
+==============+========+===========+=========+
| domain_shard | int(2) | No        | No      |
+--------------+--------+-----------+---------+

Other Changes
-------------

A new service, `designate-zone-manager` will be introduced. This service will
extend the Designate base `Service` class, but will not include the typical
`RPCService` mixin. For this initial use case, there is no requirement for
an RPC API.

This service will use the existing oslo-incubator ThreadGroup.add_timer()
methods for scheduling tasks, and the tooz library for group membership.

The timer interval will be exposed as a configuration value, defaulting to
3600 seconds. The interval will be independant of any other timers introduced
into this service in the future.

Group membership will be implemented as a `Service` mixin, similar to
`RPCService`, allowing for it's inclusion in other services such as
`designate-pool-manager`.

The `dns.domain.exists` event format will be identical to the domain
create/update/delete event format, with the only difference being the event
name.

Finally, the list of Zones each `designate-zone-manager` is responsible for
will be gathered at the start of each timer interval, in batches of a
configurable size, based on the range of shard's allocated.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Endre Karlson

Milestones
----------

Target Milestone for completion:
  Liberty-2

Work Items
----------

* Implement "Partitioner" based on Tooz
* Implement `GroupMembership` mixin (Name TBD)
* Create new `designate-zone-manager` service
* Implement periodic exists event


Dependencies
============

- Requires the OpenStack Tooz library be added to our requirements
- Requires infrastructure for the OpenStack Tooz library (memcache, redis,
  or zookeeper)
