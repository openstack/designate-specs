..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode


============================================
Use Keystone Client session / discovery code
============================================

BP: https://blueprints.launchpad.net/python-designateclient/+spec/switch-to-keystone-session

Problem description
===================

Make the CLI use the Session pattern which should be better then the current
approach that's based on a hook / taking the token from ksclient.auth_token
after ksclient.authenticate() is called.

Proposed change
===============

V1 client code shouldn't change except have added capabilities.

We would end up having the following parameters for the designateclient.v1.Client object:

===================  =======================
Name                 Description
===================  =======================
username             Username (v2/v3)
user_id              User's ID (v3)
user_domain_id       User's Domain ID (v3)
user_domain_name     User's Domain Name (v3)
password             Password (v2/v3)
tenant_name          Tenant Name (v2)
tenant_id            Tenant ID (v2)
project_name         Project Name (v3)
project_id           Project ID (v3)
project_domain_name  Project Domain Name (v3)
project_domain_id    Project Domain ID (v3)
auth_url             Auth URL w/wo auth version in it
                     (It will be discovered by ks.discover if not present)
token                Existing authentication Token (v2/v3)
endpoint_type        Endpoint type (v2/v3)
service_type         Service type (v2/v3)
insecure             Require valid SSL certs
cacert               CA Cert to use
cert                 SSL Cert to use
===================  =======================


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  endre-karlson

Milestones
----------

Target Milestone for completion:
  Kilo-1

Work Items
----------

N/A

Dependencies
============

- python-keystoneclent v0.11.+