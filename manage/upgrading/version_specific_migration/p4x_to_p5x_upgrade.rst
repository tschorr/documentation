=========================================================
Upgrading Plone 4.x to 5.0
=========================================================


.. admonition:: Description

   Instructions and tips for upgrading to a newer Plone version.

.. contents:: :local:

Changes due to implemented PLIPS
================================

PLIP 13350 "Define extra member properties TTW"
-----------------------------------------------

In Plone 5, the custom fields displayed in the user profile form and the registration form are managed by `plone.schemaeditor`.
They are dynamically editable from the Plone control panel, and can be imported from a Generic Setup profile file named `userschema.xml`.

If you have some custom member properties in your Plone site, be aware that:

- extra attributes defined in `memberdata_properties.xml` will be preserved, but they will not be automatically shown in the user profile form or the registration form,
- if you have implemented some custom forms in order to display your custom member attributes, they will not work anymore as `plone.app.users` is now based on `z3c.form`.
  You can easily replace them by declaring their schema in `userschema.xml`.

.. note::

    When a custom field is defined in `userschema.xml`, its corresponding attribute is automatically created in the `portal_memberdata` tool, so there is no need to declare it in `memberdata_properties.xml`.
    `memberdata_properties.xml` will only handled attributes that are not related to the user profile form or the registration form.


Other PLIP changes
------------------

PLIPs that resulted in changes that will have to be documented in this upgrade-guide. Check back soon for updates!


plone.api
  Todo: Tell people to use it. Explain how to configure plone.recipe.codeanalysis to check for old-style code
  Roel

plone.app.multilingual
  Todo: How to migrate from LP to PAM

Convert control panels to use z3c.form
  Todo: How to migrate your custom control-panels
  Who: Tisto

Main_template rebirth to HTML5  bloodbare
  Todo: What to do when you customized your main_templates.
  Who: ?

Automatic CSRF Protection
  Todo: How to protect your existing forms
  Who: Nathan

Linkintegrity in Plone 5
  Who: pbauer



Changed imports and functions
========================================


Products.CMFPlone.interfaces.IFactoryTool
-----------------------------------------

This is now moved to ATContentTypes.

Example:

.. code-block:: python

    try:
        # plone 4
        from Products.CMFPlone.interfaces import IFactoryTool
    except ImportError:
        # plone 5
        from Products.ATContentTypes.interfaces.factory import IFactoryTool


plone.app.multilingual
======================

..  warning::

    This is still work in progess

There are 3 different parts to the migration from LinguaPlone to plone.app.multilingual:

* From LP to PAM 2.X - on Plone4 and than to Plone5 (PAM 3.X)

  See: https://github.com/plone/plone.app.multilingual/issues/181

* From PAM 1.X to 2.X - on Plone4 and than to Plone5 (PAM 3.X)

  Step 1: plone.multilingual is merged into plone.app.multilingual. Imports in your custom code needs to be changed:
  See:https://github.com/plone/plone.app.multilingual/issues/181#issuecomment-141661848

  Step 2: Removed plone.multilingualbehavior: https://github.com/plone/plone.app.multilingual/issues/183

  Step 3: TODO

* From PAM 2.X on Plone4 to Plone5 (PAM 3.X)

  Step 1: plone.multilingual is merged into plone.app.multilingual. Imports in your custom code needs to be changed: See:https://github.com/plone/plone.app.multilingual/issues/181#issuecomment-141661848
  https://github.com/plone/Products.CMFPlone/issues/1187


Archetypes
============

Plone 5 now uses dexterity as the content type engine instead of Archetypes.

For packages that still use Archetypes, you'll need to install the ATContentTypes base package.

The easiest way to get the dependencies for Archetypes (uuid_catalog, reference_catalog, archetypes_tool) is to add the following profile to your dependencies in ``metadata.xml``:

..  code-block:: xml

    <dependencies>
         ...
        <dependency>Products.ATContentTypes:base</dependency>
    </dependencies>

See https://github.com/smcmahon/Products.PloneFormGen/blob/master/Products/PloneFormGen/profiles/default/metadata.xml for a working example.


Resource Registry
=================

.. seealso::

   http://docs.plone.org/adapt-and-extend/theming/resourceregistry.html

Plone 5 introduces some new concepts, for some, with working with JavaScript in Plone.
Plone 5 utilizes Asynchronous Module Definition (AMD) with `requirejs <http://requirejs.org/>`_.
We chose AMD over other module loading implementations(like commonjs) because AMD can be used in non-compiled form in the browser.
This way, someone can click "development mode" in the resource registry control panel and work with the non-compiled JavaScript files directly.

Getting back on point, much of Plone's JavaScript was or still is using JavaScript in a non-AMD form.
Scripts that expect JavaScript dependency scripts and objects to be globally available and not loaded synchronously will have a difficult time figuring out what is going on when upgrading to Plone 5.

There are two scenarios where this will happen that we'll tackle in this post. 1) You have JavaScript registered in portal_javascripts that are not AMD compatible. 2) You have JavaScript included in the head tag of your theme and/or specific page templates that are not AMD compatible.


1) Working with deprecated portal_javascripts
---------------------------------------------

The deprecated resource registries(and portal_javascripts) has no concept of dependency management.
It simply allowed you to specify an order in which JavaScript files should be included on your site.
It also would combined and minify them for you in deployment mode.

Registration changes
~~~~~~~~~~~~~~~~~~~~

Prior to Plone 5, JavaScript files were added to the registry by using a `Generic Setup Profile <http://docs.plone.org/develop/addons/components/genericsetup.html>`_ and including a jsregistry.xml file to it.
This would add your JavaScript to the registry, with some options and potentially set ordering.

In Plone 5.0, Plone will still recognize these jsregistry.xml files.
Plone tries to provide a shim for those that are stubborn to migrate.
How it does this is by adding all jsregistry.xml JavaScripts into a "plone-legacy" Resource Registry bundle.
This bundle simply includes a global jQuery object and includes the resources in sequential order after it.

Old style jsregistry.xml
~~~~~~~~~~~~~~~~~~~~~~~~

An old style Resource Registry would look like this:

.. code-block:: xml

    <?xml version="1.0"?>
    <object name="portal_javascripts">
      <javascript
        id="++resource++foobar.js"
        inline="False"
      />
    </object>


To migrate this to Plone 5, resource registrations are all done in the `Configuration Registry <https://pypi.python.org/pypi/plone.app.registry>`_.

New style with registry.xml
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The new registration will look something like:

.. code-block:: xml

    <?xml version="1.0"?>
    <registry>
      <records prefix="plone.resources/foobar"
               interface='Products.CMFPlone.interfaces.IResourceRegistry'>
        <value key="js">.++resource++foobar.js</value>
        <value key="deps">jquery</value>
      </records>
    </registry>

Notice how I've now added the deps property of "jquery".
This is not necessary--I'm just giving an example that this script needs a global jquery available.

This alone will not get your JavaScript included however.
In order to modernize our JavaScript stack, Plone needed to make some changes with how it included JavaScript.
All we've done so far is define a resource.
In order for a resource to be included, it needs to be part of a bundle. A bundle defines a set of resources that should be compiled together and distributed to the browser.
So you either need to add your resource to an existing bundle or create your own bundle.

In this post, we'll describe the process of creating your own bundle. Again, we use registry.xml for configuration:

.. code-block:: xml

    <records prefix="plone.bundles/foobar"
             interface='Products.CMFPlone.interfaces.IBundleRegistry'>
      <value key="resources">
        <element>foobar</element>
      </value>
      <value key="enabled">True</value>
      <value key="jscompilation">++resource++foobar-compiled.min.js</value>
      <value key="last_compilation">2015-02-06 00:00:00</value>
    </records>

One important aspect here is the "jscompilation" settings.
This defines the compiled resource used in production mode.


But, it's a bit more work
~~~~~~~~~~~~~~~~~~~~~~~~~~

Yes, we know. We tried very hard to figure out the easiest way to modernize Plone's JavaScript development stack.
The old, sequential inclusion is not useful these days.

That being said, adding resources, bundles and compiling them can all be done Through The Web(TTW) in the new Resource Registries configuration panel.
That way you can turn on development mode, compile your resources and then copy that compiled version into your package for distribution and not need to know any newfangled nodejs technologies like grunt, gulp, bower, npm, etc.


Updating non-AMD scripts
------------------------

If you are not including your JavaScript in the Resource Registries and just need it to work alongside Plone's JavaScript because you're manually including the JavaScript files in one way or another(page templates, themes), there are a number of techniques available to read on the web that describe how to make your scripts conditionally work with AMD.

For the sake of this post, I will describe one technique used in Plone core to fix the JavaScript.
The change we'll be investigating can be seen with `in a commit to plone.app.registry <https://github.com/plone/plone.app.registry/commit/ad904f2d55ea6e45bb983f1fcc12ead7a191f50a>`_. plone.app.registry has a control panel that allows some ajax searching and modals for editing settings.

To utilize the dependency management that AMD provides and have the javascript depend on jQuery, we can wrap the script in an AMD `require` function.
This function allows you to define a set of dependencies and a function that takes as arguments, those dependencies you defined. After the dependencies are loaded, the function you defined is called.

Example:

.. code-block:: javascript

    require([
      'jquery',
      'pat-registry'
    ], function($, Registry) {
      'use strict';
      ...
      // All my previous JavaScript file code here
      ...
    });

Here, the two dependencies we have are jQuery and the pattern registry.
I will not get into the pattern registry as it's off topic for this discussion--it is basically a registry of JavaScript components.
The necessity for using it here is with ajax calls and binding new DOM elements dynamically added to the page.

Additionally, above this `require` call, I provide some backward compatible code that you can inspect.
It's not necessary in this case but I added it to show how someone could make their script work when requirejs was available and when it was not.


Caveats
-------

Compilation
~~~~~~~~~~~

Prior to Plone 5, when a resource was changed or added to the javascript registry, the registry would automatically re-compile all your JavaScript files.

In switching to AMD, the compile step is much more resource intensive.
It takes so long, there is no way we could do this real-time.
Additionally, it can not be done in Python.

When changes are made to existing bundles, re-compilation will need to be done TTW in the Resource Registries control panel.
There is a build button next to each bundle.
For advanced users, compilation can be done using a tool like grunt in your development environment.

Conditional resources
~~~~~~~~~~~~~~~~~~~~~

In Plone 5, individual resources can not be registered conditionally to certain page.
This is due to the way we build JavaScript with AMD.

Instead we have Python helper-methods in the Resource Registry to add custom JS and CSS to your views or forms.

Instead of useing the legacy fill-slot like this (Plone 4):

..  code-block:: xml

    <metal:slot fill-slot="javascript_head_slot">
      ...
    </metal:slot>
    <metal:slot fill-slot="css_slot">
      ...
    </metal:slot>

In Plone 5 it’s recommended to instead use the new Python methods you can find in ``Products.CMFPlone.resources``:

..  code-block:: python

    from Products.CMFPlone.resources import add_bundle_on_request
    from Products.CMFPlone.resources import add_resource_on_request

    add_resource_on_request(self.request, 'jquery.recurrenceinput')
    add_bundle_on_request(self.request, 'thememapper')

This is better than always loading a resource or bundle for your whole site.

Only bundles can be conditionally included. So if you have a resource that needs to be conditionally included, it will likely need its own bundle.


Control Panel
=============

In Plone 4.x, the Plone configuration settings have been stored as portal properties spread across the ZMI. In Plone 5, those settings are all stored as plone.app.registry entries in registry.xml.

There are now sections in the control panel, this can be set from the controlpanel.xml. See the current definitions for more information.

The display of icons for control panels is now controlled by css.  The name of the control panel is normalized into a css class, which is applied to the link in the main layout of all control panels.  For example, if the “appId” of your control panel (as set in controlpanel.xml in your install profile) is “MyPackage” then the css class that will be generated is “.icon-controlpanel-MyPackage”. In order to have an icon for your control panel you must make sure that a css rule exists for that generated css class.  An example might be::

    .icon-controlpanel-MyPackage:before { content: ‘\e844’; }

The value you use for this css rule should identify one of the fontello icons included in Plone, or a font-based icon provided by your package itself.

It is not possible at this time to set an icon for your add-on package control panels without including css in your package.

For documentation on how to use it in your own add-ons see http://training.plone.org/5/registry.html


Generic Setup
-------------

All settings are stored in the registry.xml Generic Setup file. This file can be exported through the ZMI (Zope Management Interface). Go to the Plone Site Setup, choose "Management Interface" from the "Advanced" section. Click on "portal_setup". Go to the "export" tab. Choose the "Export the configuration registry schemata" checkbox and click the "Export selected steps" button. The registry.xml file will contain entries like this::

  <record name="plone.available_editors"
          interface="Products.CMFPlone.interfaces.controlpanel.IEditingSchema" field="available_editors">
    <value>
      <element>TinyMCE</element>
      <element>None</element>
    </value>
  </record>

  <record name="plone.available_languages" interface="Products.CMFPlone.interfaces.controlpanel.ILanguageSchema" field="available_languages">
    <value>
      <element>en-us</element>
    </value>
  </record>

Drop the settings you want to change into registry.xml in you Generic Setup profile folder. Re-install your add-on product and the settings will be available.


Python Code
-----------

All Generic Setup settings can be looked up with Python code.

First we lookup the registry utility::

  >>> from zope.component import getUtility
  >>> from plone.registry.interfaces import IRegistry
  >>> registry = getUtility(IRegistry)

Now we use the schema 'ISearchSchema' to lookup for a RecordProxy object with
all fields::

  >>> from Products.CMFPlone.interfaces import ISearchSchema
  >>> search_settings = registry.forInterface(ISearchSchema, prefix='plone')

Now we an get and set all fields of the schema above like::

  >>> search_settings.enable_livesearch
  True

If you want to change a setting, just change the attribute::

  >>> search_settings.enable_livesearch = False

Now the enable_livesearch should disabled::

  >>> search_settings.enable_livesearch
  False


Editing Control Panel
---------------------

Plone 5.x::

  >>> from Products.CMFPlone.interfaces import IEditingSchema
  >>> editing_settings = registry.forInterface(IEditingSchema, prefix='plone')

  >>> editing_settings.default_editor
  u'TinyMCE'

  >>> editing_settings.ext_editor
  False

  >>> editing_settings.enable_link_integrity_checks
  True

  >>> editing_settings.lock_on_ttw_edit
  True


Language Control Panel
----------------------

Plone 5.x::

  >>> from Products.CMFPlone.interfaces import ILanguageSchema
  >>> language_settings = registry.forInterface(ILanguageSchema, prefix='plone')

  >>> language_settings.default_language
  'en'

  >>> language_settings.available_languages
  ['en']

  >>> language_settings.use_combined_language_codes
  True

  >>> language_settings.display_flags
  False

  >>> language_settings.always_show_selector
  False

  >>> language_settings.use_content_negotiation
  False

  >>> language_settings.use_path_negotiation
  False

  >>> language_settings.use_cookie_negotiation
  False

  >>> language_settings.authenticated_users_only
  False

  >>> language_settings.set_cookie_always
  False

  >>> language_settings.use_subdomain_negotiation
  False

  >>> language_settings.use_cctld_negotiation
  False

  >>> language_settings.use_request_negotiation
  False


Mail Control Panel
------------------

Plone 4.x::

  >>> portal = getSite()
  >>> mailhost = getToolByName(portal, 'MailHost')
  >>> mailhost.smtp_host = 'localhost'
  >>> mailhost.smtp_port = '1234'
  >>> mailhost.smtp_uid = 'admin'
  >>> mailhost.smtp_pwd = 'secret'
  >>> getUtility(ISiteRoot).email_from_name = 'Plone Site'
  >>> getUtility(ISiteRoot).email_from_address = 'plone@example.com'


Maintenance Control Panel
-------------------------

Plone 5.x::

  >>> from Products.CMFPlone.interfaces import IMaintenanceSchema
  >>> maintenance_settings = registry.forInterface(IMaintenanceSchema, prefix='plone')

  >>> maintenance_settings.days
  7


Navigation Control Panel
------------------------

Plone 5.x::

  >>> from Products.CMFPlone.interfaces import INavigationSchema
  >>> navigation_settings = registry.forInterface(INavigationSchema, prefix='plone')

  >>> navigation_settings.generate_tabs
  True

  >>> navigation_settings.nonfolderish_tabs
  True

  >>> navigation_settings.displayed_types
  ('Image', 'File', 'Link', 'News Item', 'Folder', 'Document', 'Event')

  >>> navigation_settings.filter_on_workflow
  False

  >>> navigation_settings.workflow_states_to_show
  ()

  >>> navigation_settings.show_excluded_items
  True


Search Control Panel
--------------------

Plone 5.x::

  >>> from Products.CMFPlone.interfaces import ISearchSchema
  >>> search_settings = registry.forInterface(ISearchSchema, prefix='plone')

  >>> search_settings.enable_livesearch
  False

  >>> search_settings.types_not_searched
  (...)


Site Control Panel
------------------

Plone 4.x::

  >>> portal = getSite()
  >>> portal_properties = getToolByName(portal, "portal_properties")
  >>> site_properties = portal_properties.site_properties

  >>> portal.site_title = settings.site_title
  >>> portal.site_description = settings.site_description
  >>> site_properties.enable_sitemap = settings.enable_sitemap
  >>> site_properties.exposeDCMetaTags = settings.exposeDCMetaTags
  >>> site_properties.webstats_js = settings.webstats_js

  >>> settings.enable_sitemap -> plone.app.layout

Plone 5.x::

  >>> from Products.CMFPlone.interfaces import ISiteSchema
  >>> site_settings = registry.forInterface(ISiteSchema, prefix='plone')

  >>> site_settings.site_title
  u'Plone site'

  >>> site_settings.exposeDCMetaTags
  False

  >>> site_settings.enable_sitemap
  False

  >>> site_settings.webstats_js
  u''


Overview Control Panel
----------------------

Plone 5.x::

  >>> from Products.CMFPlone.interfaces.controlpanel import IDateAndTimeSchema
  >>> tz_settings = registry.forInterface(IDateAndTimeSchema, prefix='plone')

  >>> tz_settings.portal_timezone = 'UTC'


Markup Control Panel
--------------------

Plone 5.x::

  >>> from Products.CMFPlone.interfaces import IMarkupSchema
  >>> markup_settings = registry.forInterface(IMarkupSchema, prefix='plone')

  >>> markup_settings.default_type
  u'text/html'

  >>> markup_settings.allowed_types
  ('text/html', 'text/x-web-textile')


User and Groups Control Panel
-----------------------------

Plone 5.x::

  >>> from Products.CMFPlone.interfaces import IUserGroupsSettingsSchema
  >>> usergroups_settings = registry.forInterface(IUserGroupsSettingsSchema, prefix='plone')

  >>> usergroups_settings.many_groups
  False

  >>> usergroups_settings.many_users
  False
