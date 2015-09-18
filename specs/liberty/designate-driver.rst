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
Designate to Designate driver
=============================

https://blueprints.launchpad.net/designate/+spec/d2d-driver

Add a driver that uses a Designate service and SECONDARY zones like we do
with Dyn / Akamai.


Problem description
===================

We want to be able to use a instance of a Designate service to act as the
target for our zones.

Proposed change
===============

Create a driver that utilizes the SECONDARY zones feature of Designate.

API Changes
-----------

N/A

Central Changes
---------------

N/A

Storage Changes
---------------

N/A


Other Changes
-------------

Add driver 'designate' to designate.backends using v2 api and secondary zones.
Authentication is done via the keystoneclient auth / session using the
provided credentials in the backend options.

Alternatives
------------

N/A

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  endre-karlson <endre.karlson at hp dot com>

Milestones
----------

Target Milestone for completion:
  Liberty-3

Work Items
----------

- Create designate.backend.impl_designate with associated unit tests and
  devstack support

Dependencies
============

N/A
