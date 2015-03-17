.. _user-documentation:


Installation
=============

Install Switchboard and its dependencies using ``pip``::

    pip install switchboard

Next, bootstrap Switchboard within the application. The best approach
depends on which application framework is being used.

Pyramid
-------

Switchboard has a pyramid add-on available to make pyramid setup easier::

    pip install pyramid_switchboard

Once the dependency is in place, there are several ways to make sure that
``pyramid_switchboard`` is active and Switchboard is up and running. They are
all equivalent.

1. Add ``pyramid_switchboard`` to the ``pyramid.includes`` section of the
   application's main configuration section::

    [app:main]
    ...
    pyramid.includes = pyramid_switchboard

2. Use the ``includeme`` function via `config.include`::

    config.include('pyramid_switchboard')

3. Optionally setup Switchboad for easy use `in templates`_.

Once activated, Switchboard's admin UI is accessible at ``/_switchboard/`` and
switches can now be used in the code.

Other Frameworks
----------------

Switchboard is compatible with any application framework that uses WebOb_ as the
underlying request/response library. Even if a plugin/add-on doesn't exist,
Switchboard can still be setup manually.

Configuration
^^^^^^^^^^^^^

The first step is to configure switchboard in the application's config file.
Switchboard has only a handful of settings, none of which are required:

+------------------------------+-------------+--------------------------------+
| Key                          | Default     | Description                    |
+==============================+=============+================================+
| switchboard.mongo_host       | localhost   | The host for MongoDB.          |
+------------------------------+-------------+--------------------------------+
| switchboard.mongo_port       | 27017       | The port for MongoDB.          |
+------------------------------+-------------+--------------------------------+
| switchboard.mongo_db         | switchboard | The database name.             |
+------------------------------+-------------+--------------------------------+
| switchboard.mongo_collection | switches    | The collection name.           |
+------------------------------+-------------+--------------------------------+
| switchboard.internal_ips     |             | Comma-delimited list of IPs.   |
+------------------------------+-------------+--------------------------------+

Note that the "switchboard" prefix for the setting keys is also optional; more
on that in `Initializing`_.

Initializing
^^^^^^^^^^^^

In the application's bootstrap or initialization code, pass the settings into
Switchboard's ``configure`` method::

    from switchboard import configure
    ...
    configure(settings, nested=True)

If the setting keys are *not* prefixed with "switchboard" the ``nested=True``
argument can be omitted.

An example configuration that needs ``nested=True``::

    switchboard.mongo_host=mongodb.example.org
    switchboard.mong_port=27018

And one that does not need ``nested=True``::

    mongo_host=mongodb.example.org
    mong_port=27018

The Admin UI
^^^^^^^^^^^^

Once Switchboard is configured, setup a view that exposes Switchboard's admin
UI.

**Really Important Security Note**: Please configure this view so that only
admins can access it. Switchboard is a powerful tool and should be adequately
secured.

Switchboard uses Mako to render its templates, so the framework may need to be
configured to load the Mako_ engine.

Routing
^^^^^^^

Choose a URL within the application to use as Switchboard's root route; this
will be referred to as ``SWITCHBOARD_ROOT``. Additonal routes underneath
``SWITCHBOARD_ROOT`` will also need to be setup:

* ``SWITCHBOARD_ROOT/``
* ``SWITCHBOARD_ROOT/add``
* ``SWITCHBOARD_ROOT/update``
* ``SWITCHBOARD_ROOT/status``
* ``SWITCHBOARD_ROOT/delete``
* ``SWITCHBOARD_ROOT/add_condition``
* ``SWITCHBOARD_ROOT/remove_condition``
* ``SWITCHBOARD_ROOT/history``

Views
^^^^^

Depending on the framework, a view or controller will need to be created to
handle the routes above. Switchboard includes an example_ of integrating with
`Bobo <http://bobo.digicool.com/en/latest/>`_, a lightweight framework that
uses WebOb_. This class will need to do the following:

* Provide handlers for all of the `Routing`_.
* Define the output (HTML or JSON) for each handler.
* Wrap Switchboard's ``switchboard.admin.controllers.CoreAdminController``.

Implement methods within the view class to handle each of the routes below.
They should delegate to the corresponding function in ``CoreAdminController``
and render the specified output.

+---------------------------------------+--------+-------------------------------------------+
| Route                                 | Output | Template                                  |
+=======================================+========+===========================================+
| ``SWITCHBOARD_ROOT/``                 | HTML   | ``switchboard.admin.templates.index.mak`` |
+---------------------------------------+--------+-------------------------------------------+
| ``SWITCHBOARD_ROOT/add``              | JSON   | NA                                        |
+---------------------------------------+--------+-------------------------------------------+
| ``SWITCHBOARD_ROOT/update``           | JSON   | NA                                        |
+---------------------------------------+--------+-------------------------------------------+
| ``SWITCHBOARD_ROOT/status``           | JSON   | NA                                        |
+---------------------------------------+--------+-------------------------------------------+
| ``SWITCHBOARD_ROOT/delete``           | JSON   | NA                                        |
+---------------------------------------+--------+-------------------------------------------+
| ``SWITCHBOARD_ROOT/add_condition``    | JSON   | NA                                        |
+---------------------------------------+--------+-------------------------------------------+
| ``SWITCHBOARD_ROOT/remove_condition`` | JSON   | NA                                        |
+---------------------------------------+--------+-------------------------------------------+
| ``SWITCHBOARD_ROOT/history``          | JSON   | NA                                        |
+---------------------------------------+--------+-------------------------------------------+

For more details, please look through the example_ code. Once the views are
defined switches may be used in the code.

Post-Request Cleanup
^^^^^^^^^^^^^^^^^^^^

The last thing to setup is to trigger an event when the request is finished.
Switchboard needs to cleanup some caching data. If this event is not triggered
changes to the switches will not propogate out without server restarts.
Depending on the framework's architecture invoking something at the end of a
request may mean creating some sort of WSGI middleware or implementing an
event handler. For example, as WSGI middleware::

    from webob import Request
    from switchboard.signals import request_finished

    class SwitchboardMiddleware(object):

        def __init__(self, app):
            self.app = app

        def __call__(self, environ, start_response):
            req = resp = None
            try:
                req = Request(environ)
                resp = req.get_response(self.app)
                return resp(environ, start_response)
            finally:
                self._end_request(req)

        def _end_request(self, req):
            if req:
                # Notify Switchboard that the request is finished
                request_finished.send(req)


Using Switches
==============

By default, Switchboard is set to autocreate switches, which means that a
switch just needs to be checked in code and if it doesn't exist it will be
created and disabled by default. A switch is always referred to by its key, a
string identifier that should be unique.

A Word on Workflow
------------------

The developer can choose whether to take advantage of autocreate or not. There
are two basic workflows. The first, which uses autocreate, is this:

1. Write the code first. Reference the switch in the code.
#. Test the application in such a way that the code containing the switch is
   exercised.
#. Refresh the Switchboard admin UI to see the new switch. Modify it as needed.
#. If necessary, re-test the application with the proper switch status and/or
   condition sets.

The primary advantage of this approach is that there is no chance that the
switch key used in the code will differ from the one in Switchboard, e.g.,
due to a typo. It can also be advantageous, from the perspective of flow_, to
delay having to exit the code editor until a later time. The disadvantage is
having to exercise code twice: once to create the switch and then again to test
switch behavior.

Eschewing autocreate:

1. Create the switch in the admin UI. Modify it as needed.
#. Write the code, making sure to use the key of the newly-created switch.
#. Test the application.

This approach minimizes time spent putting the application through its paces,
but at the expense of switching between the web browser and the code editor.

Use whatever works.

In Python
---------

To use in Python (views, models, etc.), import the operator singleton
and use the ``is_active`` method to see if the switch is on or not::

    from switchboard import operator
    ...
    if operator.is_active('foo'):
        ... do something ...
    else:
        ... do something else ...

If autocreate is on (and it is by default), the ``foo`` switch will be
automatically created and set to disabled the first time it is referenced.
Activating the switch and controlling exactly when the switch is active,
are covered in `Managing switches`_.

In Templates
------------

Every templating engine has its own take on how (or even if) logic may be used.
That said, Switchboard provides a helper to make things easier:
``switchboard.template_helpers.is_active``. This function is just a wrapper
around ``operator.is_active`` to make it easier to check a switch. Here are
examples in some of the common Python templating engines.

In Jinja_, the helper can be setup as a test_ and used like so::

    {% if 'foo' is active %}
    ... do something ...
    {% else %}
    ... do something else ...
    {% endif %}

Check the application framework's documentation for information on how to
setup custom Jinja tests.

In Mako_, the helper can be imported directly::

    <%!
        from switchboard.template_helpers import is_active
    %>
    ...
    % if is_active('foo'):
    ... do something ...
    % else:
    ... do something else ...
    % endif

In Javascript
-------------

The easiest way to use Switchboard in conjunction with Javascript is to set a
flag within the template code. Using Mako's syntax in the template::

    <%!
        from switchboard import operator
    %>
    <script>
        window.switches = window.switches || {};
        % if operator.is_active('foo'):
        switches.foo = true;
        % else:
        switches.foo = false;
        % endif
    </script>

In the Javascript::

    if (switches.foo) {
        ... do something ...
    } else {
        ... do something else ...
    }

Again, this time using Jinja syntax and the Switchboard-provided "active"
test_::

    <script>
        window.switches = {};
        switches.foo = {{ 'true' if 'foo' is active else 'false' }};
    </script>

Custom Conditions
-----------------

Switchboard supports custom conditions, allowing application developers to
adapt switches to their particular needs. Creating a condition typically
consists of extending ``switchboard.conditions.ConditionSet``.

An example: if the application needs to activate switches for visitors from a
particular country, a custom condition can do the geo lookup on the IP from
the request and return the country value::

    from switchboard.conditions import ConditionSet, Regex
    from my_app.geo import country_code_by_addr, client_ip

    class GeoConditionSet(ConditionSet):
        countries = Regex()

        def get_namespace(self):
            ''' Namespaces are unique identifiers for each condition set. '''
            return 'geo'

        def get_field_value(self, instance, field_name):
            ''' Should return the expected value for any given field. '''
            if field_name == 'countries':
                return country_code_by_addr(client_ip())

        def get_group_label(self):
            ''' A human-friendly label used in the UI. '''
            return 'Geo'

The first thing in the custom condition is to define the fields that makeup the
condition. In this case, there is one "countries" field, which is a regex,
allowing admins to specify criteria like ``(US|CA)`` (US or Canada). Here are the
fields supported by Switchboard:

* ``switchboard.conditions.Boolean`` - used for binary, on/off fields
* ``switchboard.conditions.Choice`` - used for multiple choice dropdowns
* ``switchboard.conditions.Range`` - used for numeric ranges
* ``switchboard.conditions.Percent`` - a special type of range specific to
  percentages
* ``switchboard.conditions.String`` - string matching
* ``switchboard.conditions.Regex`` - regex expression matching
* ``switchboard.conditions.BeforeDate`` - before a date
* ``switchboard.conditions.OnOrAfterDate`` - on or after a date

Once the fields are defined, there are some methods that need to be implemented.
``get_namespace`` and ``get_group_label`` are simple functions that return a key and
a UI string respectively. Most of the work happens in the ``get_field_value``
function, which is responsbile for returning the value that is compared against
the user-provided input. Each field type may do the comparison (between the
user-provided input and what's returned by ``get_field_value``) in a different
way; in this case, it's a regex search.

When an admin sets up a Geo condition set and sets the countries field to
"US|CA", that input is compared against the country code returned by
``get_field_value``. If they match, then the switch passes that particular
condition.

Context Objects
---------------

Every switch is evaluated (to see if it is active or not) within a particular
context. By default, that context includes the request object, which allows
Switchboard to specify conditions such as: "make this switch active only for
requests with ``foo`` in the query string." That said, there may be other
objects that would be handy to have available in the context. For example, in
an e-commerce setting, the Product model may have a ``new`` flag. By passing
the model into the ``is_active`` method, Switchboard can now activate
switches based on that flag::

    if operator.is_active('foo', my_product):

Any objects passed into the ``is_active`` method after the switch's key will be
added to the context. Normally when dealing with context objects, a custom
condition will be required to actually evaluate the switch against that object.

Managing switches
=================

Switches are managed in the admin UI, which is located at the
``SWITCHBOARD_ROOT`` within the application. The admin UI allows:

* Viewing and searching all switches.
* Reviewing or auditing a switch's history.
* Adding, editing, and removing switches.
* Controlling a switch's status.
* Setting up condition sets for a switch.

Of all these capabilities, the last two are of the most interest, as the status
and condition sets determine whether a switch is active.

Statuses
--------

There are four statuses:

* Inactive - disabled for everyone
* Selective - active only for matched conditions
* Inherit - inherit from the parent switch
* Global - active for everyone

Inactive and global are opposite extremes: the switch is turned on or
off for everyone. The inherit status is used for `Parent-child switches`_. The
selective status means that the switch is only active if it passes the
condition sets.

By default, a switch will be created and set to the inactive status. Typical
workflow would be to put code using a switch into production. The corresponding
switch will be autocreated the first time the code containing it is executed,
thus visible in the admin UI. Once visible, the admin can set any desired
conditions before finally activating the switch by setting it to the proper
status.

Condition Sets
--------------

When a switch is in selective status, Switchboard checks the
conditions within the condition set to see if the switch should
be active. Conditions are criteria such as "10% of all visitors" or
"only logged in users" that can be applied to the request to see if the
switch should be active. When a switch is in selective status, it will
only be active if it meets the conditions in place.

Parent-child switches
---------------------

Switchboard allows a switch to inherit conditions from a parent, which can be
useful when multiple switches need to share a common condition set. To setup
parent-child relationship, simply prefix the switch with the parent's key,
using a colon ':' as the separator. The parent-child relationships can be as
deep as needed, e.g., ``grandparent:parent:child``.

A real world example: using Switchboard to conduct an AB test. AB tests
have two gates: the first are the visitors who are part of the test, and the
second is to determine who sees which variant. In this example, 10% of site
traffic should be in the test, with half (i.e., 5% of traffic) seeing the normal
(control) A variant and the other half seeing the B variant. The test is setup
with two switches:

* abtest
* abtest:B

The ``abtest`` switch has a "0-10% of traffic" condition set. The ``abtest:B``
switch will inherit from ``abtest`` and can add its own "0-5% of traffic"
condition. Half of those in the test will see the B variant, the rest will see
the control A variant. The ``abtest:B`` switch's status should be set
to selective, for reasons noted below.

Note that an additional tool, like `Google Analytics Content Experiments`_, is
still needed to measure conversion within each variant, but Switchboard can
handle traffic segmentation.

Two potential spots of confusion:

1. Child switches *always* inherit from their parents, even when the child
   switch's status is set to something other than inherit. An inherit status
   just means the child switch isn't adding to the parent switch's status.

2. It is also important to note that when a parent switch is disabled, it takes
   precedence over the statuses of any child switches. On the other hand, if the
   parent switch is enabled, it can be overriden by the child switch, e.g., if
   the parent has a global status but the child has an inactive status, the
   child's inactive wins out.


.. _test: http://jinja.pocoo.org/docs/dev/templates/#tests
.. _example: https://github.com/switchboardpy/switchboard/blob/master/example/server.py
.. _flow: https://en.wikipedia.org/wiki/Flow_(psychology)
.. _WebOb: http://www.webob.org/
.. _Mako: http://makotemplates.org/
.. _Jinja: http://jinja.pocoo.org
.. _`Google Analytics Content Experiments`: https://support.google.com/analytics/answer/1745147?hl=en