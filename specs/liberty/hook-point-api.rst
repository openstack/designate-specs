..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode

=================
 Hook Points API
=================

`Hook Point API Blueprint
<https://blueprints.launchpad.net/designate/+spec/hook-point-api>`_

Designate, by design, is required to interface with legacy DNS
applications. Organizations will typically have other systems that
must interface with DNS and vice versa. The hook points API provides a
generic way to add organization specific code that adds no overhead in
the default use case and does not allow changes to the underlying APIs
when code is hooked in to a method or function call.


Problem description
===================

Currently, in order to support organization specific functionality, it
is required to either:

 1. Maintain a fork of the Designate code base
 2. Monkey patch the code via a module
 3. Treat designate as a black box and use HTTP level tools to add org
    specific code.

While maintaining a fork can be a reasonable solution, if there are
large changes, it can be very difficult to merge upstream
changes. Monkey patching forces the same level of knowledge regarding
the code base, yet it is tricky to get right as small code changes can
silently fail due to import changes. Lastly, treating Designate as a
black box will often require org specific code to reimplement aspects
of Designate due to limited level of granularity.

The Hook Points API provides a practical supported means of injecting
code, while avoiding the pitfalls mentioned above.

Proposed change
===============

The Hook Points API provides a decorator to define a function or
method as a hook point.

.. code-block:: python

    @hookpoints.hook_point('pool_manager_create_domain')
    def create_domain(self, context, domain):
        ...

In the above case the hook point is a **named** hook point called
`pool_manager_create_domain`. When a name is not provided, the hook
point uses the path of the module and method.

.. code-block:: python

   @hookpoints.hook_point()
   def create_domain(self, context, domain):
        ...

The name is the combination of function's module and name. ::

  '%s.%s' % (func.__module__, func.__name__)

If no hook point is defined, the original function is returned
as-is.

In order to define a hook point, a package must be installed
that provides a `designate.hook_point` entry point.

.. code-block:: python

   from setuptools import setup


   setup(
       name='raxdns',
       entry_points: {
           'designate.hook_point': [
               'pool_manager_create_domain = raxdns.hooks.pool_manager:create_domain'
           ]
       }
   )

When the package is installed, the hook point is then **active**
meaning that it will be called when the target is called. It may be
disabled via the configuration if necessary.


Hook Point Implementation
-------------------------

A hook point, as it will be applied as a decorator, is an object that
implements a `__call__` method that accepts a single function as the
argument and returns an appropriate function as the result. It is up
to the implementor to ensure the hook point correctly implements the
API of the hook target.

For convenience, there is `BaseHook` that can be used to reuse common
patterns.

.. code-block:: python

   class BaseHook(object):

       OPTS = [
           cfg.BoolOpt('disabled', default=False)
       ]

       def __init__(self, group):
           self.group = group

       @property
       def disabled(self):
           return cfg.CONF[self.group].get('disabled', False)

       def wrapper(self, *args, **kw):
           return self.hook_target(*args, **kw)

       def __call__(self, f):
           # Save our hook target as an attribute for our wrapper method
           self.hook_target = f

           @functools.wraps(self.hook_target)
           def wrapper(*args, **kw):
               if self.disabled:
                   return self.hook_target(*args, **kw)
               return self.hook(*args, **kw)
           return wrapper

The `BaseHook` takes care of:

 1. using `functools.wrap` correctly
 2. disabling the hook when configured to do so
 3. setting the config group for later use of the config
 4. simplifying the decorator implementation

This base class is meant to make development of a hook simpler. A hook
author is free to implement the hook as a normal decorator as
well.

Configuration
~~~~~~~~~~~~~

It is important to note that any configuration must be accessed via
`oslo.config`. The reason being is that the hooks are applied at
import time, where the confguration is typically loaded at run
time. Therefore, the hook may not have access to config data until the
hook target is actually called.


Hook Example
~~~~~~~~~~~~

Here is an example of a hook point that wraps the `create_domain`
method in the Pool Manager service. It validates the domain doesn't
exist in another application that can also manage domains via the same
backends.

.. code-block:: python

    import requests
    from oslo_log import log as logging
    from oslo_config import cfg

    from designate.pool_manager.service import ERROR_STATUS
    from designate.hookpoints import BaseHook


    LOG = logging.getLogger(__name__)


    class CheckDCXDomainHook(BaseHook):
        OPTS = BaseHook.OPTS + [
            cfg.Opt('legacy_dns', required=True),
        ]

        @property
        def sess(self):
            if not hasattr(self, '_sess'):
                sess = requests.Session()
                self._sess = sess
            return self._sess

        @property
        def legacy_dns(self):
            return cfg.CONF[self.group].legacy_dns

        def hook(self, obj, context, domain):
            resp = self.sess.get(self.legacy_dns + '/find_domains?name=%s' % domain.name)

            # The domain is not found in the legacy system. Let Designate create it
            if not resp.ok:
                # We got a 404, so let Designate make the call
                return self.hook_target(obj, context, domain)

            # The legacy system owns the domain. Notify central it was
            # an error.
            #
            # The `obj` is the Service object.
            obj.central_api.update_status(
                context, domain.id, ERROR_STATUS, domain.serial
            )


It is the responsibility of the hook point author to be a good citizen
and properly handle any errors / return values in the original code, as
well as support any internal APIs.

Again, the intent of the Hook Point API is to allow an organization a
means of injecting code, which implies a reasonably intimiate
knowledge with the code.


Hook Point Management and Configuration
---------------------------------------

Hook points are installed by installing a package that includes
`designate.hook_point` entry points. By default, these will be
**enabled** and will be called when the specific hook point target is
called. These hooks **MAY** be **disabled** in the config at the hook
point level.

.. code-block:: cfg

   [hook_point:pool_manager_create_domain]
   disabled = True

If necessary, hook points can also receive configuration details.

.. code-block:: cfg

   [hook_point:pool_manager_create_domain]
   legacy_dns_api = https://my.dns.legacy.org.net:8975

The configuration will be available via the global
`oslo.config.cfg.CONF` object.


Central Changes
---------------

None


Storage Changes
---------------

None

Other Changes
-------------

Hook points can be added liberally or in an extremely limited, known
uses cases. Similarly, no hook points can be formally added and an
organization may apply them as necessary in an org specific patch.


Alternatives
------------

Other than the originally mentioned tactics for hooking into the
Designate code, more specific hook points could be created on a
per-use basis. For example, there could be very a specific API for
hooks that get called when a message is read from the queue. While
providing more specific hooks may allow for a use case specific API,
it would also require each API be designed, documented and tested. The
hook point API provides a single tested means of injecting code that
has limited effect on the API over time and allows a reasonable level
of support.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  eric-larson


Milestones
----------

Target Milestone for completion:
  Liberty-1


Work Items
----------

 - Add `designate.hookpoints`, implementing the `@hook_point`
   decorator
 - Add documentation for writing hook points and enumrate what hook
   points exist.

See `this review <https://review.openstack.org/164748>`_ for the
current implementation.

dependencies
============

- stevedore
