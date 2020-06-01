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

======================
 Designate Split View
======================

https://blueprints.launchpad.net/designate/+spec/split-view
https://bugs.launchpad.net/designate/+bug/1875939

Split view is an important feature most DNS servers have support for
based on the need to split internal, external view of the IPs and hostnames,
most of the companies have this feature on their existing DNS structers and
adding support for it Designate will make it easier for the user to integrate
Designate into the current user systems

The below links provide insight on the split view feature and use cases:
* http://techgenix.com/you_need_to_create_a_split_dns/
* https://en.wikipedia.org/wiki/Split-horizon_DNS

Problem description
===================

The current implementation of Designate does not support the split view, users that
want to implement this must either patch Designate code or somehow do it in the backend of 
Designate (bind, powerdns, ....)


Proposed change
===============

Introduce a new type of zone (split_view) in Designate. when a zone with that type is created
Designate will do the following:

* Create the same zone twice in different views (internal, external), this will require
  the pool.yaml to be configured with two pools seperated with attributes internal/external
  Note: the user may use view parameter in bind or define two instances of powerdns
  to acheive that
* Create two TSIG keys one for each zone and attach the TSIG to the right zone
* Provide AXFR based on TSIG key so if the TSIG sent is exteranl give the external view
  the user should provide a regex for external/internal view that match the IPs that
  should be included in each view, so if the IP match internal regex specified by
  user and the AXFR request is signed with internal TSIG the IP will be included
  in zone AXFR response 
* Also notify requests should be signed with the right TSIG (internal, external)
  the user should provide a regex for internal/external that match the IPs that should be
  Included/Excluded from each view and based on that the TSIG key can be selected

Backend should be configured as follows:

* All AXFR request should be signed with TSIG, so the AXFR can return the right view
* Most of Designate backend agents listed in the link below
  https://docs.openstack.org/designate/pike/contributor/backends/index.html
  have support for TSIG AXFR/split_view (initial development could be for bind, powerdns)

  [bind]
  https://www.slashroot.in/how-to-configure-split-horizon-dns-in-bind
  https://bind9.readthedocs.io/en/latest/advanced.html#instructing-the-server-to-use-a-key
  [powerdns]
  https://www.frank.be/implementing-bind-views-with-powerdns/
  https://doc.powerdns.com/authoritative/tsig.html#provisioning-signed-notification-and-axfr-requests
  [djbdns]
  djbdns does not support TSIG based on the link below
  https://en.wikipedia.org/wiki/Djbdns
  https://www.fefe.de/djbdns/#splithorizon
  [gdnsd]
  gdnsd not sure if it support split view/tsig AXFR
  [infoblox]
  https://docs.infoblox.com/display/NAG8/Chapter+18+DNS+Views
  https://docs.infoblox.com/display/NAG8/Enabling+Zone+Transfers
  [knote]
  https://www.knot-dns.cz/docs/2.6/html/configuration.html
  https://knot-resolver.readthedocs.io/en/v2.2.0/modules.html#views-and-acls
  [MSDNS]
  https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-gssa/c0c6ffdd-a094-40b1-bbb9-bc4e5a58804f
  https://docs.microsoft.com/en-us/windows-server/networking/dns/deploy/split-brain-dns-deployment 



Storage Changes
---------------
The Designate zones table should be updated to accept the new zone type split_view
type enum('PRIMARY','SECONDARY') to type enum('PRIMARY','SECONDARY','SPLIT_VIEW')


Assignee(s)
-----------
hamza alqtaishat
