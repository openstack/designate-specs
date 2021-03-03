..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode

============================
A Worker Model for Designate
============================

Thesis: Gratuitous unnecessary complexity exists within the Designate
code, and in the operation of Designate as a service. Making Designate
a producer-worker type of project will vastly simplify the development
and operation, and align it more to the true nature of the service it
provides (DNS).


Problem description
===================

``designate-pool-manager`` does a reasonably good job at pushing Create/Delete
changes out to nameservers, but the process gets a bit less shiny after that.

- Polling to see if state is live is done via asynchronous and synchronous RPC
  with another component.
- Cache usage is a mess (storing mutliple keys, keeping one key around forever,
  storing a different key for each type of operation).
- Periodic Sync/Recovery are very unreliable as the number of changes grows.
- The update_status and logic for calculating consensus is heavy-handed, too
  eager, and too complex.
- The state machine is very foggy, and the logic for updating status that gets
  pushed into central is obfuscated.
- Pool Managers are tied to one pool

``designate-zone-manager`` does a good job at executing periodic timers for the
zones it manages. However:

- One zone-manager process is responsible for a certain set of zones, if the
  operations for that set of zones gets heavy, a single zone manager process
  could become overwhelmed.
- We rely on ``tooz`` to manage the extremely delicate task of ensuring
  balance and coverage of all zones by zone manager processes.
- Certain work (export) that's in the critical path of operations has already
  crept into the component that wasn't really meant for that. As a substitute
  for proper workers, the zone-manager is looking like the current answer.

``designate-mdns`` is a DNS server written in Python. It works well for small
amounts of traffic, but as traffic grows, we may realize that we need it to be
more specialized, as a DNS server written in Python should be. The logic for
sending NOTIFYs and polling of changes seems less likely to belong in mdns in
the future. If those bits were removed, ``designate-mdns`` could be rewritten
to make use of a better tool for the problem.


Proposed change
===============

A change to the underlying architecture of executing actual work on DNS servers
and the running of other tasks. Essentially, removing
``designate-pool-manager`` and ``designate-zone-manager``, replacing them with
``designate-worker`` and ``designate-producer`` (names up for debate) and
removing certain logic from ``designate-mdns``. All of the actual "work" would
be put in the scalable ``designate-worker`` process, which has work produced
by the API/Central, and ``designate-producer``. ``designate-mdns`` gets back
to it's roots, and only answers AXFRs. Callbacks over queues that don't involve
the API are eliminated, simplifying the code in all components that deal with
DNS servers.

**No changes** to the API or Database are required, with minimal changes to
``designate-central``.

To the end user, the results of this change would be relatively simple:

- Scalability limited only by DNS servers, datastores, and queues. If at any
  point Designate starts to slow down in some aspect, unless there's an issue
  with those services listed above, the problem can be solved by throwing more
  Designate processes at the problem.
- Fault tolerance. One or more Designate processes dying would be inivisible
  in almost every case. An operator wouldn't have to fear a certain process
  dying, because no one process handles responsiblities others will not, as long
  as there is a minimal amount of redundancy.
- Simplicity. The Designate architecture becomes much cleaner and easier to
  understand. The questions of "what is the difference between pool-manager and
  zone-manager" and "what's synchronous and asynchronous" become less muddy,
  or disappear altogether.
- Operationally, this opens the door for simpler scenarios for small
  deployments where a customer need only scale a couple components (even on
  the same machine) to get more performance. You could even go so far as to
  deploy app nodes that only shared datastore (db, cache) and had their own
  designate components and queues.


Architectural Changes
---------------------

These are the services that would remain present:

-  ``designate-api`` - To receive JSON and parse it for Designate
-  ``designate-central`` - To do validation/storage of zone/record data and
   send CRUD tasks to ``designate-worker``.
-  ``designate-mdns`` - To **only** perfrom AXFRs from Designate's database
-  ``designate-worker`` - Any and all tasks that Designate needs to
   produce state on nameservers
-  ``designate-producer`` - To run periodic/timed jobs, and produce work
   for ``designate-worker`` that is out the normal path of API operations.
   For example: Periodic recovery.

Other necessary components:

-  Queue - Usually RabbitMQ
-  Database - Usually MySQL
-  Cache - (encouraged, although not necessary) Memcached, MySQL, Redis
-  A ``tooz`` backend (Zookeeper, Memcahed, Redis)

Services/components that are no longer required:

-  ``designate-pool-manager``
-  ``designate-zone-manager``


Designate Worker
----------------

The scope of ``designate-worker``'s duties are essentially any and all tasks
that Designate needs to take action to perform. For example:

-  Create, Update, and Delete zones on pool targets via backend plugins
-  Poll that the change is live
-  Update a cache with the serial number for a zone/target
-  Emit zone-exists events for billing
-  Flatten Alias Records
-  Clean up deleted zones
-  Importing/Exporting zones
-  Many more

The service essentially exposes a vast RPCAPI that contains ``tasks``.

An important difference to Designate's current model is that all
of these tasks do not call back. They are all fire-and-forget tasks
that will be shoved on a queue and await worker action.

``tasks`` are essentially functions, that given relatively simple input, make
the desired income happen on either nameservers, or the Designate database.

Cache
~~~~~

The cache performs a similar function to the current pool manager cache
now.

It will store state for each different type of task that a worker can use
to decide if it needs to continue with a ``task`` received from the queue, or
simply drop it and move on to the next task.

This varies by task, some are relatively simple, knowing whether to perform
a zone update to a certain serial number is knowable by seeing the serial
number of a zone on each target in a pool. For DNSSEC zone signing, a key
would probably be placed to indicate that a certain worker was working on
resigning a zone, as it's a more long-running process.

In the absence of such a cache, each worker will act naive and try to complete
each task it receives.

Tasks
~~~~~

Each task will be idempotent, to the degree that it is possible.

As mentioned in the ``Cache`` section, to a certain degree, tasks could be
able to know if they need to complete work based on information in the cache.

But they should also make an effort to not duplicate work, for instance,
if it's trying to delete a zone that's already gone, it should interpret
the zone being gone as a sign that the delete is successful and move on.

On the whole these tasks would simply be lifted from where they currently
exist in the code, and wouldn't change all that much.

A slight change might be that during the course of the task, we may recheck
that the work that is being undertaken still needs to be done.

As an example:
An API customer creates many recordsets very quickly. The work being dispatched
to ``designate-worker`` processes would go to a lot of different places, and one of
the first updates to actually reach a nameserver might contain all the changes
necessary to bring the zone up-to-date. The other tasks being worked should
check before they send their NOTIFY that the state is still behind, and check
again after they've sent their NOTIFY, but before they've began polling, so
that they can cut down on unnecessary work for themselves, and the nameservers.

You could get even smarter about the markers that you drop in a cache for these
tasks. For example, on a zone update, you could drop a key in the cache of the
nature ``zoneupdate-foo.com.``, and other if other zoneupdate tasks for the same
zone see that key, they could know to throw away their job and move on.

designate-mdns changes
~~~~~~~~~~~~~~~~~~~~~~

The partioning of certain elements that Designate had previously disappears.
The worker service will send DNS queries, it will do cpu-bound tasks, but it
will be one place to scale. It should be possible to have an extremely robust
Designate architecture by simply scaling these workers.

``designate-mdns`` will have it's entire RPCAPI transferred to
``designate-worker``. This will vastly simplify the amount of work it needs
to do while it sits in the critical path of providing zone transfers to
nameservers Designate manages.

As a side-note, this would make this service much easier to optimize, or even
rewrite in a faster programming language.

Designate Producer
------------------

``designate-producer`` is the place where jobs that produce tasks that
are outside of the normal path of API operations and operate on
some kind of timer live.

The **key** difference to the ``zone-manager`` service, is that this service
simply generates work to be done, rather than actually doing the work.
``designate-producer`` simply decides what needs to be done, and sends RPC
messages on the queue to ``designate-worker`` to actually perform the work.

As we've grown Designate, we've seen the need
for this grow vastly, and even more so in the future.

-  Deleted zone purging
-  Refreshing Secondary Zones
-  Emitting zone exists tasks and other billing events
-  DNSSEC signing of zones
-  Alias record flattening

We could move the ``periodic_sync`` and ``periodic_recovery`` tasks from the
Pool Manager to this service.

The ``periodic_sync`` and ``periodic_recovery`` tasks in the Pool Manager have
been a constant struggle to maintain and get right. This is due to a lot of
factors.

Making the generation of ``tasks`` by periodic processes the job of only one
Designate component simplifies the architecture, and allows to solve the
problems it presents one time, one way, and generally do one thing well.

Timers
~~~~~~

This service would essentially be a group of timers that wake up on a cadence
and create work to be put on the queue for ``designate-worker`` processes to
pick up.

The overhead is relatively low here, as we're not actually doing the work, but
more just scheduling the work to be done. This way we can focus on the
unexpectedly difficult problem of dividing up the production of work that these
processes will put on the queue.

Dividing work
`````````````

To explain more clearly, the biggest problem we have in this service is making
it fault-tolerant, but not duplicating work for ``designate-worker`` processes
to do. This was solved before by ``tooz`` using the zone shards in the Designate
database in ``designate-zone-manager`` and it seems to work well.

``designate-worker`` processes, as described above, will do a certain amount of
optimization so that they don't duplicate work. But if we generate too much
cruft, those processes will be bogged down just by the task of seeing if they
need to do work. So we should work to minimize the amount of duplicate work we
produce.

Queue Priority
--------------

One potential complication of this implementation is that, as the number of
timers and tasks that are out of Designate's critical path of implementation
grow, they may get in the way of ``designate-worker`` processes doing the tasks
that are most important, namely CRUD of zones and records.

We propose having queues/exchanges for each type of task, this would be an
optimal way to monitor the health of different types of tasks, and isolate the
sometimes long-running tasks that periodic timers will produce from the
relatively quicker, and more important CRUD operations. The algorithm for
choosing tasks from the various options could be customized by a particular
process if desired. But a good general default would be to handle CRUD
operations from ``designate-central`` first. Or use a weighted random
choice algorithm, with the critical-path CRUD operations having higher weights.


Work Items
----------

- Stand up a ``designate-worker`` service
- Migrate CRUD zone operations to ``designate-worker``, reworking the cache
  implementation.
- Stand up ``designate-producer`` service
- Migrate Pool Manager periodic tasks to ``designate-producer``, with small
  modifications to ensure they simply generate work for ``designate-worker``
- Move ``designate-mdns``' NOTIFYing and polling to ``designate-worker``
- Fix up the ``update_status`` logic in ``designate-central``
- Migrate all tasks from ``zone-manager`` to a split of ``designate-worker``
  and ``designate-producer``, where ``producer`` creates the work on the queue
  and ``worker`` executes it. Ensuring scalable logic for distributed work
  production using cache or some other method in ``designate-producer``
- Deprecate ``pool-manager`` and ``zone-manager``
- Profit!!!


Upgrade Implications
--------------------

Upgrading to the next release with this change would introduce some operational
changes. Mostly around the services that need to be deployed. The deployment
need not be a cutover, deploying Newton Designate will work with or without the
worker. This is because of a variety of compatibility measures taken:

- ``designate-central`` will have a configurable "zone api" that it can swap
  between ``designate-pool-manager`` and ``designate-worker``. If the worker
  process is enabled, central can send c/u/d zones events to the worker
  instead of the pool manager.
- ``designate-worker``'s ability to send NOTIFYs and Poll DNS servers can
  replace a portion of ``designate-mdns``' responsibilities. For certain
  DNS servers, it's theorized that they won't behave well if a NOTIFY comes
  from a different server than it's master that it zone transfers from.
  For this reason, ``designate-worker``'s ability to send NOTIFYs is a
  configurable element. Since the worker calls into a backend plugin to update
  zones, the NOTIFY via mdns logic in those backends can remain, and if the
  operator so chooses, the NOTIFY task in the worker can no-op. This works both
  ways. An operator can also choose to have the MiniDNS notify calls noop, and
  allow them to be completed by the worker process.
- For those who choose to firewall all DNS traffic between Designate and DNS
  servers, it will be safest to deploy ``designate-worker`` processes in
  close proximity to ``designate-mdns`` processes. So that the DNS polling that
  ``designate-worker`` does can be completed, where ``designate-mdns`` used
  to do it.
- As periodic-type processes are migrated to ``designate-producer`` and
  ``designate-worker``, they can be marked as "worker tasks" in
  ``designate-zone-manager``, that can be turned off behind a configuration
  flag.

The process for upgrading to the worker model code, after deploying Newton
could look something like this:

1. (To account for firewalling dns traffic) Start ``designate-worker``
   processes operating on the same IPs as ``designate-mdns`` processes via
   proximity or proxy. The default configuration will still allow NOTIFYs
   and DNS polling to occur via ``designate-mdns``, and all other operations
   to work in ``designate-pool-manager``. No traffic will reach the worker.
2. Toggle configuration values ``designate-worker::enabled`` and
   ``designate-worker::notify`` and restart ``designate-worker``.
3. Restart ``designate-central`` and ``designate-mdns`` processes so that mdns
   NOTIFY calls no-op, and central starts to use the worker instead of
   ``designate-pool-manager``.
4. Toggle the ``designate-zone-manager::worker-tasks`` config flag and restart
   ``designate-zone-manager`` so that it hands off periodic tasks to the
   producer/worker.
5. Start the ``designate-producer`` process so that the worker starts doing
   recovery and other periodic tasks.
6. Stop the ``designate-pool-manger`` processes, and if all processes are
   migrated out of ``designate-zone-manager``, that as well.


Milestones
----------

Target Milestone for completion:
  Newton

Author(s)
---------

Tim Simmons https://launchpad.net/~timsim

Paul Glass https://launchpad.net/~pnglass

Eric Larson https://launchpad.net/~eric-larson
