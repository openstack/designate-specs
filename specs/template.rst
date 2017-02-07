..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. The filename in the git
  repository should match the launchpad URL, for example a URL of
  https://blueprints.launchpad.net/designate/+spec/awesome-thing should be named
  awesome-thing.rst .  Please do not delete any of the sections in this
  template.  If you have nothing to say for a whole section, just write: None
  For help with syntax, see http://sphinx-doc.org/rest.html
  To test out your formatting, see http://www.tele3.cz/jbar/rest/rest.html

=============================
 The title of your blueprint
=============================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/designate/+spec/example

Introduction paragraph -- why are we doing anything?

If your specification proposes any changes to the Designate REST API such
as changing parameters which can be returned or accepted, or even
the semantics of what happens when a client calls into the API, then
you should add the APIImpact flag to the commit message. Specifications with
the APIImpact flag can be found with the following query::

https://review.openstack.org/#/q/status:open+project:openstack/designate-specs+message:apiimpact,n,z


Problem description
===================

A detailed description of the problem.

Proposed change
===============

Here is where you cover the change you propose to make in detail. How do you
propose to solve this problem?

If this is one part of a larger effort make it clear where this piece ends. In
other words, what's the scope of this effort?

Include where in the designate tree hierarchy this will reside.

API Changes
-----------

Include API Changes here. If you are adding endpoints / add major modifications
please ensure you have examples for calls / results - eg:

POST /v2/doohickey
^^^^^^^^^^^^^^^^^^

This creates a doohicky.

It returns an ID and the doohickey

.. code-block:: http

    POST /v2/doohickey HTTP/1.1
    Accept: application/json
    Content-Type: application/json

    {
        "doohickey":{
            "foo":"bar"
        }
    }

    HTTP/1.1 201 Created
    Content-Type: application/json; charset=UTF-8
    Location: /v2/doohickey/cddda8f0-f558-11e3-a3ac-0800200c9a66

    {
        "doohickey":{
            "id":"cddda8f0-f558-11e3-a3ac-0800200c9a66",
            "foo":"bar",
            "links":{
                "self" : "/v2/doohickey/cddda8f0-f558-11e3-a3ac-0800200c9a66"
            }
        }
    }

It may be useful to add a table with the parameters, and a info about them

+-----------+--------------------------------+----------+
| Parameter | Description                    | Required |
+===========+================================+==========+
| foo       | the foo value for the doohicky | Yes      |
+-----------+--------------------------------+----------+

Central Changes
---------------

Any changes to the central service

Storage Changes
---------------

Any changes to the DB. This should be a table (if creating a new table)
eg:


New Table - DooHickey
^^^^^^^^^^^^^^^^^^^^^

+-----+---------+-----------+---------+
| Row | Type    | Nullable? | Unique? |
+=====+=========+===========+=========+
| id  | uuid    | No        | Yes     |
+-----+---------+-----------+---------+
| foo | VARCHAR | No        | No      |
+-----+---------+-----------+---------+

Other Changes
-------------

Any other changes to Designate, broken down by which sub system is being
changed

Alternatives
------------

This is an optional section, where it does apply we'd just like a demonstration
that some thought has been put into why the proposed approach is the best one.

Implementation
==============

Assignee(s)
-----------

Who is leading the writing of the code? Or is this a blueprint where you're
throwing it out there to see who picks it up?

If more than one person is working on the implementation, please designate the
primary author and contact.

Primary assignee:
  <launchpad-id or None>

Can optionally can list additional ids if they intend on doing
substantial implementation work on this blueprint.

Milestones
----------

Target Milestone for completion:
  Juno-1

Work Items
----------

Work items or tasks -- break the feature up into the things that need to be
done to implement it. Those parts might end up being done by different people,
but we're mostly trying to understand the timeline for implementation.


Dependencies
============

- Include specific references to specs and/or blueprints in designate, or in other
  projects, that this one either depends on or is related to.

- Does this feature require any new library dependencies or code otherwise not
  included in OpenStack? Or does it depend on a specific version of library?

Upgrade Implications
====================

Does the spec introduce a change for those running the current, or an older
version of Designate? If so, describe the change(s).
