..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode


=================
V2 API - Bindings
=================

BP: https://blueprints.launchpad.net/python-designateclient/+spec/v2-api-bindings

Provide bindings for the V2 API using the Session based approach used for V1.

Problem description
===================

Provide bindings for the V2 API. (This does not include the shell commands)

Proposed change
===============

We will provide a designateclient.v2.client.Client object which would load the other Managers (not much unalike the Controllers or so in V1).

Manager - Zone
--------------

List
^^^^

List Zones.

.. code-block:: python

    client.zones.list()


Get
^^^

Get a Zone.

.. code-block:: python

    client.zones.get(zone)

Create
^^^^^^

Create a Zone.

.. code-block:: python

    client.zones.create(name)

Update
^^^^^^

Update a Zone.

.. code-block:: python

    client.zones.update(zone)

Delete
^^^^^^

Delete a Zone.

.. code-block:: python

    client.zones.delete(zone)


Manager - RecordSets
--------------------

List
^^^^

List Recordsets in a Zone.

.. code-block:: python

    client.recordsets.list(zone)

Get
^^^

Get a RecordSet

.. code-block:: python

    client.recordsets.get(zone, recordset)

Create
^^^^^^

Create a RecordSet in a Zone.

.. code-block:: python

    client.recordsets.create(zone, name, data)

Update
^^^^^^

Update a RecordSet.

.. code-block:: python

    client.recoredsets.update(zone, recordset, data)

Delete
^^^^^^

Delete a RecordSet.

.. code-block:: python

    client.recoredsets.delete(zone, recordset)


Manager - TLDs
--------------

List
^^^^

List TLDs.

.. code-block:: python

    client.tlds.list()

Get
^^^

Get a TLD

.. code-block:: python

    client.tlds.get(tld)

Create
^^^^^^

Create a TLDs.

.. code-block:: python

    client.tlds.create(name, data)

Update
^^^^^^

Update a TLD.

.. code-block:: python

    client.tlds.update(tld, data)

Delete
^^^^^^

Delete a TLD.

.. code-block:: python

    client.tlds.delete(tld)


Manager - Blacklists
--------------------

List
^^^^

List Blacklists.

.. code-block:: python

    client.blacklists.list()

Get
^^^

Get a Blacklist.

.. code-block:: python

    client.blacklists.get(blacklist)

Create
^^^^^^

Create a Blacklist.

.. code-block:: python

    client.blacklists.create(name, pattern)

Update
^^^^^^

Update a Blacklist.

.. code-block:: python

    client.blacklists.update(blacklist, pattern)

Delete
^^^^^^

Delete a Blacklist.

.. code-block:: python

    client.blacklists.delete(blacklist)


Manager - FloatingIPs
---------------------

List
^^^^

.. code-block:: python

    client.floatingips.list()

Get
^^^

.. code-block:: python

     client.floatingips.get(region, floatingip_id)

Set
^^^

Set a PTR record for a FloatingIP.

.. code-block:: python

    client.floatingips.set(region, floatingip_id, record)

Unset
^^^^^

Unset a PTR record for a FloatingIP.

.. code-block:: python

    client.floatingips.unset(region, floatingip_id)

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  endre-karlson

Milestones
----------

Target Milestone for completion:
  Juno-2

Work Items
----------

N/A

Dependencies
============

- switch-to-keystone-session