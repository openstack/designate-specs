..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode

..

====================================
 Support for server pools in Minidns
====================================

https://blueprints.launchpad.net/designate/+spec/server-pools-minidns-support

This spec outlines the apis that need to be added to MiniDNS for supporting
pool manager.

Problem description
===================


#. **notify_zone_changed:** The pool manager needs to notify pool servers when a
   zone changes. It calls into mindns to notify the pool servers of the zone change.
#. **poll_for_serial_number:** When a zone is changed, the change is marked as
   *PENDING* in central.  It will be marked as *ACTIVE* when a certain number of
   pool servers have the change.  Server pools would use mindns to check the
   pool servers for the presence of the change.

Proposed change
===============

There would be 2 calls added to minidns to support the features needed by pool manager.

Mindns Call Changes
===================

notify_zone_changed(context, domain, destination, timeout, retry_interval, max_retries)
---------------------------------------------------------------------------------------
+----------------+-----------------------------------------------------------+------------+
| **Parameter**  | **Description**                                           |**Required**|
+================+===========================================================+============+
| *context*      | The user context.                                         | Yes        |
+----------------+-----------------------------------------------------------+------------+
| *domain*       | The designate domain object.                              | Yes        |
+----------------+-----------------------------------------------------------+------------+
| *destination*  | The recipient of the NOTIFY message. The format is        |            |
|                | "ip:[port]".  If there is no port, port 53 is used.       | Yes        |
+----------------+-----------------------------------------------------------+------------+
| *timeout*      | The time to wait for a NOTIFY response from *destination*.| Yes        |
+----------------+-----------------------------------------------------------+------------+
|*retry_interval*| The time between retries.                                 | Yes        |
+----------------+-----------------------------------------------------------+------------+
| *max_retries*  | The maximum number of retries mindns would do for a NOTIFY| Yes        |
|                | response. After this many retries, mindns drops the NOTIFY|            |
|                | message.                                                  |            |
+----------------+-----------------------------------------------------------+------------+

Design Considerations
^^^^^^^^^^^^^^^^^^^^^
Mindns uses *domain.name* to construct a NOTIFY message and sends it to
*destination*.  If a response is not received within *timeout* interval, then
after a time interval of *retry_interval*, the NOTIFY message is resent to *destination*.
After trying for *max_retries*, the call completes.  All times are in seconds.

Cast vs Call
""""""""""""
This is an RPC cast rather than an RPC call.  The pool manager need not wait
for the NOTIFY response.  The reasoning here is that in the worst case after the
refresh interval, the zone on the *destination* would be refreshed.  Also the
status would be correctly updated elsewhere so this need not be a call.

Single Call vs Multiple Calls
"""""""""""""""""""""""""""""
Pool Manager sends multiple requests to (potentially multiple instances of) minidns
with a single *destination* in each call.  The reasoning is that the pool manager
has the global knowledge of the number of pool servers and NOTIFY messages that
need to be sent.  It is in a better position to decide the approach to parallelizing
the messages that need to be sent.  The minidns on the other hand is only concerned
with the dns protocol.  That said BIND sends a zone transfer request to the server
that sends the NOTIFY message.  So it makes sense to have a minidns server associated
with the same set of pool servers.

Parameter sources/validations
"""""""""""""""""""""""""""""
The *timeout*, *retry_interval* and *max_retries* would possibly be specified in
the config file in the pool manager section for the initial version.  Irrespective
of where the pool manager gets its data, it passes these as parameters to minidns.
It makes sense for pool manager to track these parameters rather than minidns.

Minidns does not do any validation of the *timeout*, *retry_interval* and *max_retries*
parameters.


poll_for_serial_number(context, domain, destination, timeout, retry_interval, max_retries)
------------------------------------------------------------------------------------------
+----------------+-----------------------------------------------------------+------------+
| **Parameter**  | **Description**                                           |**Required**|
+================+===========================================================+============+
| *context*      | The user context.                                         | Yes        |
+----------------+-----------------------------------------------------------+------------+
| *domain*       | The designate domain object.                              | Yes        |
+----------------+-----------------------------------------------------------+------------+
| *destination*  | The server to check for an updated serial number. The     |            |
|                | format is "ip:[port]".  If there is no port, port 53 is   | Yes        |
|                | used.                                                     |            |
+----------------+-----------------------------------------------------------+------------+
| *timeout*      | The time to wait for a SOA response from *destination*.   | Yes        |
+----------------+-----------------------------------------------------------+------------+
|*retry_interval*| The time between retries.                                 | Yes        |
+----------------+-----------------------------------------------------------+------------+
| *max_retries*  | The maximum number of retries mindns would do for an      | Yes        |
|                | expected serial number. After this many retries, mindns   |            |
|                | returns an ERROR.                                         |            |
+----------------+-----------------------------------------------------------+------------+

Design Considerations
^^^^^^^^^^^^^^^^^^^^^
Mindns uses *domain.name* to construct a SOA request and sends it to *destination*.
The serial number in the SOA response (hereafter referred to as *actual_serial*)
is compared to *domain.serial* (hereafter referred to as *expected_serial*).

Cast vs Call
""""""""""""
This would be a cast.  Pool Manager would provide a function - *update_status*
for minidns to inform of the result.  For more discussions on this see the
`designate meeting log notes on 03-Sep-2014
<http://eavesdrop.openstack.org/meetings/designate/2014/designate.2014-09-03-17.02.log.html>`_

.. _return_values_poll_for_serial:

Return Values
"""""""""""""
The return status from this function is sent to the pool manager using the pool
manager call back function - *update_status*.  The values sent are summarized
below.  We actually could send just the serial number without a status too.

+-------------------------------------+---------------------------------------------------------------+
|         **Case**                    | **Action**                                                    |
+=====================================+===============================================================+
| *actual_serial* = *expected_serial* | Send *SUCCESS* and *actual_serial*.                           |
+-------------------------------------+---------------------------------------------------------------+
| *actual_serial* > *expected_serial* | Send *SUCCESS* and *actual_serial*.                           |
+-------------------------------------+---------------------------------------------------------------+
| *actual_serial* < *expected_serial* | poll again after a time interval of *retry_interval* elapses. |
|                                     | Continue this until we get the *expected_serial* or for       |
|                                     | *max_retries*. Send *ERROR* and *actual_serial* after         |
|                                     | *max_retries*                                                 |
+-------------------------------------+---------------------------------------------------------------+
| timeout for *max_retries*           | Send *ERROR* and *None* for serial number.                    |
+-------------------------------------+---------------------------------------------------------------+

Pool Manager Dependencies
=========================
Pool Manager would provide an api - *update_status* for minidns to call in to
inform the status of the poll_for_serial_number call.  The :ref:`return_values_poll_for_serial`
section of *poll_for_serial_number* summarizes the values that would be sent with *update_status*
As noted above, we need not send a separate status, sending the actual_serial (or None)
should be enough.

update_status(context, domain, destination, status, actual_serial)
------------------------------------------------------------------
+------------------------+------------------------------------------------------+------------+
| **Parameter**          | **Description**                                      |**Required**|
+========================+======================================================+============+
| *context*              | The user context passed in the corresponding .       | Yes        |
|                        | *poll_for_serial_number* call                        |            |
+------------------------+------------------------------------------------------+------------+
| *domain*               | The designate domain object. *domain.serial* has the |            |
|                        | *expected_serial* number. This object is the same as | Yes        |
|                        | the one sent in the corresponding                    |            |
|                        | *poll_for_serial_number* call                        |            |
+------------------------+------------------------------------------------------+------------+
| *destination*          | The *destination* parameter from the corresponding   |            |
|                        | *poll_for_serial_number* call                        | Yes        |
+------------------------+------------------------------------------------------+------------+
| *status*               | The status would be one of *SUCCESS*, *ERROR*.       | Yes        |
+------------------------+------------------------------------------------------+------------+
| *actual_serial*        | The actual serial number received from *destination* | Yes        |
|                        | or *None* in case of a timeout.                      |            |
+------------------------+------------------------------------------------------+------------+

Storage Changes
===============

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <vinod-mang>


Milestones
----------

Target Milestone for completion:
  Kilo

Work Items
----------
* Api addition to mini-dns - notify_zone_changed
* Api addition to mini-dns - poll_for_serial_number
* Api to pool manager (minidns would call this api) - update_status

Dependencies
============
Pool Manager needs to add *update_status* for minidns to inform the status of
*poll_for_serial_number*.

