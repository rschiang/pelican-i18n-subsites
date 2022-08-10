=======================
 I18n Subsites
=======================

.. image:: https://img.shields.io/github/workflow/status/pelican-plugins/i18n-subsites/build
  :target: https://github.com/pelican-plugins/i18n-subsites/actions
  :alt: Build Status
.. image:: https://img.shields.io/pypi/v/pelican-i18n-subsites
  :target: https://pypi.org/project/pelican-i18n-subsites/
  :alt: PyPI Version
.. image:: https://img.shields.io/pypi/l/pelican-i18n-subsites?color=blue
  :alt: License

This plugin extends `Pelican`_’s translations functionality by creating
internationalized sub-sites for the default site. A demo site can be seen live
`here <http://smartass101.github.io/pelican-plugins/>`_.

This plugin is designed for Pelican 4.5 and later.

.. _Pelican: https://getpelican.com/

What it does
============

1. When the content of the main site is being generated, the settings
   are saved and the generation stops when content is ready to be
   written. While reading source files and generating content objects,
   the output queue is modified in certain ways:

  - translations that will appear as native in a different (sub-)site
    will be removed
  - untranslated articles will be transformed to drafts if
    ``I18N_UNTRANSLATED_ARTICLES`` is ``'hide'`` (default), removed if
    ``'remove'`` or kept as they are if ``'keep'``.
  - untranslated pages will be transformed into hidden pages if
    ``I18N_UNTRANSLATED_PAGES`` is ``'hide'`` (default), removed if
    ``'remove'`` or kept as they are if ``'keep'``.''
  - additional content manipulation similar to articles and pages can
    be specified for custom generators in the ``I18N_GENERATOR_INFO``
    setting.

2. For each language specified in the ``I18N_SUBSITES`` dictionary the
   settings overrides are applied to the settings from the main site
   and a new sub-site is generated in the same way as with the main
   site until content is ready to be written.
3. When all (sub-)sites are waiting for content writing, all removed
   contents, translations and static files are interlinked across the
   (sub-)sites.
4. Finally, all the output is written.

Installation
============

This plugin can be installed via:

.. code-block:: shell

    python -m pip install pelican-i18n-subsites

Setting it up
=============

Either let Pelican `automatically discover`_ the plugin, or explicitly specify
plugins via ``PLUGINS``:

.. code-block:: python

    PLUGINS = ['pelican.plugins.i18n_subsites', ...]

.. code-block:: python

    from pelican.plugins import i18n_subsites
    PLUGINS = [i18n_subsites, ...]

For each extra used language code, a language-specific settings overrides
dictionary must be given (but can be empty) in the ``I18N_SUBSITES`` dictionary

.. code-block:: python

    # mapping: language_code -> settings_overrides_dict
    I18N_SUBSITES = {
        'cz': {
	        'SITENAME': 'Hezkej blog',
	    }
	}

You must also have the following in your pelican configuration

.. code-block:: python

    JINJA_ENVIRONMENT = {
        'extensions': ['jinja2.ext.i18n'],
    }

.. _automatically discover: https://docs.getpelican.com/en/latest/plugins.html#how-to-use-plugins

Default and special overrides
-----------------------------
The settings overrides may contain arbitrary settings, however, there
are some that are handled in a special way:

``SITEURL``
  Any overrides to this setting should ensure that there is some level
  of hierarchy between all (sub-)sites, because Pelican makes all URLs
  relative to ``SITEURL`` and the plugin can only cross-link between
  the sites using this hierarchy. For instance, with the main site
  ``http://example.com`` a sub-site ``http://example.com/de`` will
  work, but ``http://de.example.com`` will not. If not overridden, the
  language code (the language identifier used in the ``lang``
  metadata) is appended to the main ``SITEURL`` for each sub-site.
``OUTPUT_PATH``, ``CACHE_PATH``
  If not overridden, the language code is appended as with ``SITEURL``.
  Separate cache paths are required as parser results depend on the locale.
``STATIC_PATHS``, ``THEME_STATIC_PATHS``
  If not overridden, they are set to ``[]`` and all links to static
  files are cross-linked to the main site.
``THEME``, ``THEME_STATIC_DIR``
  If overridden, the logic with ``THEME_STATIC_PATHS`` does not apply.
``DEFAULT_LANG``
  This should not be overridden as the plugin changes it to the
  language code of each sub-site to change what is perceived as translations.
``L10N``
  Partially translated ``dict``s under ``L10N`` will be merged recursively with
  the default locale instead of replaced altogether.

Localizing templates
--------------------

Most importantly, this plugin can use localized templates for each
sub-site. There are two approaches to having the templates localized:

- You can set a different ``THEME`` override for each language in
  ``I18N_SUBSITES``, e.g. by making a copy of a theme ``my_theme`` to
  ``my_theme_lang`` and then editing the templates in the new
  localized theme. This approach means you don't have to deal with
  gettext ``*.po`` files, but it is harder to maintain over time.
- You use only one theme and localize the templates using the
  ``jinja2.ext.i18n`` `Jinja2 extension`_. For a kickstart
  read this `guide <docs/localizing_using_jinja2.rst>`_.

.. _Jinja2 extension: https://jinja.palletsprojects.com/en/3.1.x/templates/#i18n

Additional context variables
............................

It may be convenient to add language buttons to your theme in addition
to the translation links of articles and pages. These buttons could,
for example, point to the ``SITEURL`` of each (sub-)site. For this
reason the plugin adds these variables to the template context:

``main_lang``
  The language of the main site — the original ``DEFAULT_LANG``
``main_siteurl``
  The ``SITEURL`` of the main site — the original ``SITEURL``
``lang_siteurls``
  An ordered dictionary, mapping all used languages to their
  ``SITEURL``. The ``main_lang`` is the first key with ``main_siteurl``
  as the value. This dictionary is useful for implementing global
  language buttons that show the language of the currently viewed
  (sub-)site too.
``extra_siteurls``
  An ordered dictionary, subset of ``lang_siteurls``, the current
  ``DEFAULT_LANG`` of the rendered (sub-)site is not included, so for
  each (sub-)site ``set(extra_siteurls) == set(lang_siteurls) -
  set([DEFAULT_LANG])``. This dictionary is useful for implementing
  global language buttons that do not show the current language.
``relpath_to_site``
  A function that returns a relative path from the first (sub-)site to
  the second (sub-)site where the (sub-)sites are identified by the
  language codes given as two arguments.

If you don't like the default ordering of the ordered dictionaries,
use a Jinja2 filter to alter the ordering.

All the siteurls above are always absolute even in the case of
``RELATIVE_URLS == True`` (it would be to complicated to replicate the
Pelican internals for local siteurls), so you may rather use something
like ``{{ SITEURL }}/{{ relpath_to_site(DEFAULT_LANG, main_lang }}``
to link to the main site.

This short `howto <docs/implementing_language_buttons.rst>`_ shows two
example implementations of language buttons.

Additional config option
........................

If you use plugins like  ``photos``, ``thumbnailer`` and want to prevent
the system from copying the files into each language directory, it is possible
to set a list of directories in the variable ``I18N_LINK_DIRS``.
For each path a symbolic link is created which links to the original directory.

.. code-block:: python

    I18N_LINK_DIRS = ['images/thumbnails', 'photos']

.. code-block::

   └── output/                                              # base output directory
       ├── images/
       │   └── thumbnails/                                  # original directory
       ├── photos/                                          # original directory
       └─── de/                                             # language subfolder
            ├── photos -> /output/photos                    # symbolic link to original directory
            └── images/
                └── thumbnails -> /output/images/thumbnails # symbolic link to original directory

Usage notes
===========

- It is **mandatory** to specify ``lang`` metadata for each article
  and page as ``DEFAULT_LANG`` is later changed for each sub-site, so
  content without ``lang`` metadata would be rendered in every
  (sub-)site.
- As with the original translations functionality, ``slug`` metadata
  is used to group translations. It is therefore often convenient to
  compensate for this by overriding the content URL (which defaults to
  slug) using the ``url`` and ``save_as`` metadata. You could also
  give articles e.g. ``name`` metadata and use it in ``ARTICLE_URL =
  '{name}.html'``.

Development
===========

- A demo site used for automated end to end testing is defined in
  ``pelican/plugins/i18n_subsites/test_data``.
- Run the tests using ``poetry run invoke tests``.

Contributing
============

Contributions are welcome and much appreciated. Every little bit helps. You can
contribute by improving the documentation, adding missing features, and fixing
bugs. You can also help out by reviewing and commenting on `existing issues`_.

To start contributing to this plugin, review the `Contributing to Pelican`_
documentation, beginning with the **Contributing Code** section.

.. _existing issues: https://github.com/pelican-plugins/i18n-subsites/issues
.. _Contributing to Pelican: https://docs.getpelican.com/en/latest/contribute.html

Credits
=======

Originally authored by `Ondrej Grover <https://github.com/smartass101>`_,
February 2014, and subsequently enhanced by members of the Pelican community,
including `Poren Chiang <https://poren.tw>`_, who re-packaged it for publication
to PyPI.

License
=======

This project is licensed under the AGPL-3.0 license.
