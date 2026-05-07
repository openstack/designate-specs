..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode

=====================================================
 TSIG Signing for Worker Poll and Notify Messages
=====================================================

https://bugs.launchpad.net/designate/+bug/2142614

Designate's worker service sends unsigned DNS NOTIFY and SOA poll queries to
backend nameservers. In split-horizon DNS deployments using BIND views, this
causes zone management failures because BIND uses TSIG keys to route requests
to the correct view.


Problem description
===================

When Designate manages zones across multiple BIND views (split-horizon DNS),
each view is typically configured to accept queries and notifications based
on match criteria such as source IP (match-clients), destination IP
(match-destinations), or TSIG key. One option is to use TSIG keys, where
BIND determines which view a DNS message belongs to by verifying the
signature.

Currently, Designate's MDNS service supports TSIG for AXFR (zone transfer)
responses — BIND can successfully pull zone data from MDNS using TSIG-signed
transfers. However, the worker service sends two types of DNS messages that
are completely unsigned:

1. **NOTIFY messages**: After creating or updating a zone on a backend target,
   the worker sends a DNS NOTIFY to tell the nameserver to initiate an AXFR.
   Without TSIG, BIND either rejects the NOTIFY or routes it to the default
   ("any") view if configured, which may be the wrong view.

2. **SOA poll queries**: After a zone change, the worker polls nameservers by
   sending SOA queries to verify the zone serial has been updated. Without
   TSIG, the SOA query hits the wrong view, returns the wrong serial (or
   NXDOMAIN), and the zone transitions to ERROR status.

This means that while Designate can technically create zones in BIND backends
with view-based split-horizon, the zones inevitably transition to ERROR
because the worker cannot properly notify or verify them.

The infrastructure to support TSIG signing partially exists:

- The ``TsigKey`` object model stores TSIG key name, algorithm, and secret
- The ``PoolTarget`` object has a ``tsigkey_id`` field
- MDNS already uses TSIG for signing AXFR responses
- dnspython provides ``Message.use_tsig()`` for signing outgoing queries

However, ``PoolNameserver`` (used for polling) has no TSIG reference, and the
worker's ``dnsutils.notify()`` / ``dnsutils.soa_query()`` functions have no
TSIG support.


Proposed change
===============

Add TSIG signing support to the worker's NOTIFY and SOA poll messages, keyed
by the pool target and nameserver configuration.


dnsutils Changes
----------------

Extend ``prepare_dns_message()``, ``notify()``, ``soa_query()``, and
``send_dns_message()`` to optionally accept TSIG parameters and sign
outgoing DNS messages.

When a TSIG key is provided, the DNS message is signed using dnspython's
``Message.use_tsig()`` before being sent. This applies to both UDP and TCP
transport.

The TSIG parameters (key name, secret, algorithm) are resolved from the
``TsigKey`` object referenced by the pool target or nameserver's
``tsigkey_id``.


Worker Changes
--------------

**SendNotify**: The ``SendNotify`` task already receives the pool ``target``
object. When ``target.tsigkey_id`` is set, the task looks up the ``TsigKey``
from storage and passes the TSIG parameters to ``dnsutils.notify()``.

**PollForZone**: The ``PollForZone`` task receives a ``PoolNameserver``.
With the addition of ``tsigkey_id`` to ``PoolNameserver`` (see below), the
task looks up the ``TsigKey`` from storage and passes the TSIG parameters
to ``dnsutils.soa_query()``.

**ZonePoller**: Passes the context to ``PollForZone`` so it can look up TSIG
keys from storage.


API Changes
-----------

None. TSIG keys are configured via pool YAML and the existing TSIG key API.


Storage Changes
---------------

Add ``tsigkey_id`` column to the ``pool_nameservers`` table.

Altered Table - pool_nameservers
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

+------------+---------+-----------+---------+
| Row        | Type    | Nullable? | Unique? |
+============+=========+===========+=========+
| tsigkey_id | uuid    | Yes       | No      |
+------------+---------+-----------+---------+

This mirrors the existing ``tsigkey_id`` column on ``pool_targets``.

A database migration will add this nullable column with a default of NULL,
which preserves backwards compatibility — existing deployments without TSIG
continue to work with unsigned queries.


Pool Configuration Changes
--------------------------

The pool YAML schema is extended to allow ``tsigkey_id`` on nameservers,
consistent with the existing support on targets:

.. code-block:: yaml

    - name: internal-view-pool
      description: Pool for BIND internal view

      nameservers:
        - host: 192.0.2.1
          port: 53
          tsigkey_id: 4a1a4e28-2b4e-4b2a-8b0a-3c4d5e6f7a8b

      targets:
        - type: bind9
          description: BIND9 internal view
          tsigkey_id: 4a1a4e28-2b4e-4b2a-8b0a-3c4d5e6f7a8b

          masters:
            - host: 192.0.2.10
              port: 5354

          options:
            host: 192.0.2.1
            port: 53
            rndc_host: 192.0.2.1
            rndc_port: 953
            rndc_key_file: /etc/bind/rndc.key

    - name: external-view-pool
      description: Pool for BIND external view

      nameservers:
        - host: 192.0.2.1
          port: 53
          tsigkey_id: 9b8c7d6e-5f4a-3b2c-1d0e-a9b8c7d6e5f4

      targets:
        - type: bind9
          description: BIND9 external view
          tsigkey_id: 9b8c7d6e-5f4a-3b2c-1d0e-a9b8c7d6e5f4

          masters:
            - host: 192.0.2.10
              port: 5354

          options:
            host: 192.0.2.1
            port: 53
            rndc_host: 192.0.2.1
            rndc_port: 953
            rndc_key_file: /etc/bind/rndc.key

In this configuration, both pools point to the same physical BIND server
(192.0.2.1), but use different TSIG keys to route to different views. The
nameserver and target for the same view typically share the same TSIG key.

The TSIG keys must be created beforehand via the Designate TSIG key API with
scope ``POOL`` and the corresponding pool's resource_id.


Other Changes
-------------

**PoolNameserver object**: Add ``tsigkey_id`` field (UUIDFields, nullable).

**Pool update command**: ``designate-manage pool update`` already reads pool
YAML and updates pool objects. It must handle the new ``tsigkey_id`` field on
nameservers, same as it already does for targets.


Alternatives
------------

1. **TSIG key on pool (not target/nameserver)**: Use a single TSIG key per
   pool instead of per-target and per-nameserver. Simpler, but doesn't work
   when a pool has multiple targets or nameservers that each require their
   own key (e.g., multiple BIND servers with distinct TSIG keys). Keeping
   TSIG per-target and per-nameserver is consistent with the existing
   target-level ``tsigkey_id`` and gives operators full control.

2. **Derive nameserver TSIG from target TSIG**: Instead of adding
   ``tsigkey_id`` to nameservers, infer the TSIG key from the target that
   shares the same host. This avoids a schema change but creates a fragile
   implicit coupling — the pool data model has no explicit relationship
   between targets and nameservers, so matching would require comparing
   hosts across different schemas (nameserver ``host`` vs. target
   ``options.host``), with the risk of ambiguous or missing matches.

3. **Configuration file options**: Add TSIG key name/secret/algorithm as
   worker configuration options (e.g., ``[service:worker] tsig_key_name``).
   This would apply a single TSIG key to all poll/notify messages globally,
   which doesn't work for multi-pool split-horizon deployments where each pool
   needs a different key.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Omer Schwartz (oschwart)


Milestones
----------

Target Milestone for completion:
  2026.2 (Hibiscus)


Work Items
----------

- Add ``tsigkey_id`` to ``PoolNameserver`` object model
- Database migration: add ``tsigkey_id`` column to ``pool_nameservers`` table
- Extend ``dnsutils.notify()`` and ``dnsutils.soa_query()`` to accept and
  apply TSIG parameters
- Update ``SendNotify`` to look up and use TSIG key from target
- Update ``PollForZone`` / ``ZonePoller`` to look up and use TSIG key from
  nameserver
- Update ``designate-manage pool update`` to handle nameserver ``tsigkey_id``
- Unit and functional tests
- Documentation: pool configuration guide for split-horizon with TSIG


Dependencies
============

None. All required libraries (dnspython TSIG support) and infrastructure
(TsigKey model, PoolTarget.tsigkey_id) already exist.


Upgrade Implications
====================

A database migration adds the nullable ``tsigkey_id`` column to
``pool_nameservers``. Existing deployments are unaffected — the column
defaults to NULL, and when no TSIG key is configured, the worker continues
to send unsigned messages as it does today. No changes to existing pool YAML
files are required.
