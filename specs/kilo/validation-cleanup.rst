..

This work is licensed under a Creative Commons Attribution 3.0 Unported License.
http://creativecommons.org/licenses/by/3.0/legalcode


=============================
 Centralize Validation Logic
=============================

https://blueprints.launchpad.net/designate/+spec/validation-cleanup

Problem description
===================

Today, validations are duplicated between the V1 and V2 APIs, and validations
will be required in additional places going forward (Inbound AXFR, DynamicDNS
etc). Centralizing these validations into the Designate Objects provides a
single re-usable home for all entry points to use.

Proposed change
===============

Centralizing this logic will require a number of phases to complete:

1.  Implement an Object Registry
2.  Implement Object Validation
3.  Implement an "Adaptor" layer, replacing the V2 APIs Views
4.  Migrate schemas from `designate/resources/schemas` into the Objects
5.  Update the API layer (both V1 and V2) to use the new Validations and
    Adaptors

Object Registry
---------------

The object registry will allow for looking up a reference to any
DesignateObject's class via the class name. This will allow an object's schema
to reference other object easily.

.. note:: The Object Registry will **not** replace the standard and existing
          method of retrieving an Object class by importing it. The Registry
          provides an **alternative** method of obtaining a class reference
          using only the class name. This is useful for scenarios where we need
          to reference an Object outside of python code. For example, in a
          JSON-Schema, or in JSON messages passed from service to service with
          oslo.messaging.

To implement the registry, the `DesignateObjectMetaclass` class will be updated
to track a reference to each of the Object classes as they are constructed.
These references will be stored in a dictionary attached to the
`DesignateObject` base class.

.. note:: The `DesignateObjectMetaclass` code is executed while the object
          classes are constructed, rather than when the object instances are
          created. This ensures the code is only executed once upon startup of
          the Designate services.

Registry lookups will be performed via a new
`DesignateObject.obj_cls_from_name()` method, which will accept a single
string argument for the Object Name.

.. code-block:: python

    class DesignateObject(object):
        @classmethod
        def obj_cls_from_name(cls, name):
            pass

An example usage of the registry:

.. code-block:: python

    class RecordSet(DesignateObject):
        FIELDS = {
            'id': {},
            'name': {},
        }

    RecordSet = DesignateObject.obj_cls_from_name('RecordSet')

    my_recordset = RecordSet(id='12345', name='example.org.')


Object Validation
-----------------

Object Validation rules will continue to use JSON-Schema, implemented on a
per-field level:

.. code-block:: python

    class ValidatableObject(DesignateObject):
        FIELDS = {
            'id': {
                'required': True,
                'schema': {
                    'type': 'string',
                    'format': 'uuid'
                }
            },
            'ttl': {
                'schema': {
                    'type': 'integer',
                    'minimum': 0,
                    'maximum': 100
                }
            },
            'recursive': {
                'schema': {
                    '$ref': 'obj://ValidatableObject/#',
                }
            },
            'nested': {
                'schema': {
                    '$ref': 'obj://AnotherObject/#',
                }
            }
        }

To construct the final and complete scheme, and instanciate the schema
validator, the `DesignateObjectMetaclass` class will be updated
to call a `make_class_validator(cls)` method, implemented similarily to
the `make_class_properties(cls)` method.

This `make_class_validator` method will assemble the per-field schema fragments
into a full JSON Schema, with the necessary boilerplate being generated.
Additionally, this method will construct the python-jsonschema Validator
instance and attach it to the objects class as cls._obj_validator.

Finally, three new methods will be added to the base `DesignateObject` class:

1.  A `obj_get_schema(cls)` method:

    .. code-block:: python

        class DesignateObject(object):
            @classmethod
            def obj_get_schema(cls):
                """Returns the JSON Schema for this Object."""

2.  A `is_valid(self)` method:

    .. code-block:: python

        class DesignateObject(object):
            def is_valid(self):
                """Returns True if the Object is valid."""

3.  A `validate(self)` method:

    .. code-block:: python

        class DesignateObject(object):
            def validate(self):
                """
                Raises an InvalidObject exception if the Object is invalid

                Attached to the `errors` attribute of exception will be a
                `ValidationErrorList` object containing the details of the
                failures.
                """

An example usage of the validation:

.. code-block:: python

    class RecordSet(DesignateObject):
        FIELDS = {
            'id': {
                'required': True,
                'schema': {
                    'type': 'string',
                    'format': 'uuid'
                }
            },
            'ttl': {
                'schema': {
                    'type': 'integer',
                    'minimum': 0,
                    'maximum': 100
                }
            }
        }

    my_recordset = RecordSet(id='12345', ttl=50)

    # Returns False, as the 12345 is NOT a UUID.
    my_recordset.is_valid()

    try:
        # Raises an InvalidObject exception, as the 12345 is NOT a UUID.
        my_recordset.validate()
    except InvalidObject as e:
        LOG.warning('Invalid Object, Errors below:')

        for error in e.errors:
            LOG.warning('Error at path %s, Message: %s', e.absolute_path,
                        e.message)


Object Adaptors
---------------

Object Adaptors will replace the current V2 API Views, allowing for a
structured way to convert from Object to V1 or V2 API formats. This will
include renaming of fields in standard output, rendered JSON Schemas,
as well as in ValidationError messages, and will support hiding fields which
should not be visible in the matching API version.

.. note:: Below is a WIP mockup - Expect changes!

Example usage of the Object Adaptors:

.. code-block:: python

    # Standard Object Definition
    class Domain(DesignateObject):
        FIELDS = {
            'id': {
                'required': True,
                'schema': {
                    'type': 'string',
                    'format': 'uuid'
                }
            },
            'name': {
                'schema': {
                    'type': 'string',
                    'pattern': 'domainname'
                }
            },
            'ttl': {
                'schema': {
                    'type': 'integer',
                    'minimum': 0,
                    'maximum': 100
                }
            },
            'version': {
                'schema': {
                    'type': 'integer',
                    'minimum': 0,
                    'maximum': 100
                }
            }
        }


    # Define the V2 API Adaptor for the Domain Object above
    class DomainAdaptorV2(DesignateObjectAdaptorV2):
        obj_cls = Domain
        obj_list_cls = DomainList

        # Any fields NOT specificed will not be returned by the API.
        FIELDS = {
            'id': {
                # No V2 Specific Customization Needed
            }
            'ttl': {
                # Let's rename "ttl" to "default_ttl" in V2
                'name': 'default_ttl'
            }
        }


    # Use the Adaptor in the API
    class ZonesController(rest.RestController):
        _adaptor = DomainAdaptorV2()

        @pecan.expose(template='json:', content_type='application/json')
        @utils.validate_uuid('zone_id')
        def get_one(self, zone_id):
            """Get Zone"""

            # Real life would Fetch a zone from designate-central
            domain = Domain(id='2b9e1b86-d4f1-42d2-88ff-b888f2dd068a'
                            name='example.com.',
                            ttl=50)

            return self._adaptor.render(domain, single=True)

        @pecan.expose(template='json:', content_type='application/json')
        def post_all(self):
            """Create Zone"""
            request = pecan.request
            response = pecan.response
            context = request.environ['context']

            # The Adaptor class will parse the incoming JSON into an
            # approperiate Object instance, trigger validation, and raise
            # an exception if there are any failures. The
            # `FaultWrapperMiddleware` will catch and render this exception.
            domain = self._adaptor.parse(request.body_dict, single=True)

            # Create the Domain
            domain = self.central_api.create_domain(context, domain)

            # Prepare the response headers and status
            response.status_int = 201
            response.headers['Location'] = '<url for new zone>'

            # Send the response
            return self._adaptor.render(domain)


Other Changes
-------------

Any other changes to Designate, broken down by which sub system is being
changed


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  kiall

Milestones
----------

Target Milestone for completion:
  Kilo-1

Work Items
----------

Work items are as per the Proposed change section.


Dependencies
============

- No known dependencies
