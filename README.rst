================
Django URL tools
================

------------------------------------------------------
Django helper tools for dealing with URLs in templates
------------------------------------------------------

.. contents::

Overview
========

Django URL tools are context processors, and template tags that help you deal
with URL manipulations in templates. The heavy lifting is done by the
``url_tools.helper.UrlHelper`` class which wraps around ``urllib``,
``urlparse``, and Django's ``QueryDict`` to provide facilities for parsing and
manipulating URLs.

Installation
============

Simply install the ``django-url-tools`` package using ``easy_install`` or
``pip``::

    pip install django-url-tools

Configuring your Django project
===============================

To use the context processor, add the following to the middlewares stack::

    TEMPLATE_CONTEXT_PROCESSORS = (
        ...
        'url_tools.context_processors.current_url',
    )

If you want to use the template tags, add ``url_tools`` to installed apps::

    INSTALLED_APPS = (
        ...
        'url_tools',
    )

UrlHelper class
===============

``UrlHelper`` class implements all methods for manipulating URLs that are used
in other parts of this app. You can also use this class directly by importing
it from the ``helper`` module::

    from url_tools.helper import UrlHelper

``UrlHelper`` constructor accepts only one argument, which is the full path of
the URL you want to manipulate. Although we can technically make ``UrlHelper``
deal with full absolute URLs, we have opted to implement only methods for
dealing with paths instead. Therefore, if you pass UrlHelper an full URL with
scheme, host, port, and user credentials, it would still only use the path,
query parameters, and the fragment identifiers.

You can also pass an instance of ``UrlHelper`` class to the constructor if you
need to do so.

The class has following properties:

+ ``path``: URL's path without query string and fragment identifier
+ ``fragment``: URL's fragment identifier (without the pound character ``#``)
+ ``query_dict``: ``QueryDict`` instance containing the URL's query parameters
+ ``query``: similar to ``query_dict`` but also does more when assigning
+ ``query_string``: URL's query string
+ ``hash``: MD5 hexdigest of the full path including query parameters

UrlHelper.path
--------------

This is a simple string property containing the URl's path. For example, in an
URL ``'/foo/bar?baz=1#boo'``, the property contains ``'/foo/bar'``.

UrlHelper.fragment
------------------

Contains the fragment identifier. In the URL ``'/foo/bar?baz=1#boo'``, this
property contains ``'foo'``.

UrlHelper.query_dict
--------------------

Contains the query parameters parsed from the URL in form of
``django.http.request.QueryDict`` instance. You can read more about the
``QueryDict`` API in `Django documentation on QueryDict`_.

UrlHelper.query
---------------

This is a property returns the ``UrlHelper.query_dict`` when read, but
overrides it when assigend a normal dictionary or a string. For example::

    u = UrlHelper('/foo/bar')
    u.query = 'foo=1&bar=2'
    # or
    u.query = dict(foo=1, bar=2)

Both above assignment work.

UrlHelper.query_string
----------------------

This property returns a query string when read, and behaves the same way as the
query property when assigning a string. However, you cannot assign dictionaries
to this property. ::

    u = UrlHelper('/foo/bar')
    u.query_string = 'foo=1&bar=2'       # this works
    u.query_string = dict(foo=1, bar=2)  # but this doesn't

UrlHelper.hash
--------------

Returns the MD5 hexdigest of the full path including query parameters. This can
be useful for use with caching and other situations where we need to
differentiate same paths with different query parameters. ::

    u = UrlHelper('/foo/bar')
    u.query = dict(foo=1) # URL is now '/foo/bar?foo=1'
    u.hash  # returns '06f0a42bdd474f053fb1343165a31d42'

UrlHelper.get_query_string(**kwargs)
------------------------------------

This method returns the query string using ``QueryDict``'s ``urlencode()``
method. Any keyword parameters you pass to this method are forwarded to the
``urlencode()`` method. Currently, the only keyword argument is ``safe`` which
instructs the method to not escape specified characters.

UrlHelper.get_query_data()
--------------------------

Returns the ``UrlHelper.query_dict`` property. This methods exist mostly to
help customize the behavior of ``UrlHelper.query`` in subclasses, since the
getter calls this method instead of returning the ``query_dict`` property
directly.

UrlHelper.update_query_data(**kwargs)
-------------------------------------

This method takes any number of keyword arguments and updates the
``UrlHelper.query_dict`` instance. Since, unlike Python dictionary, each
``QueryDict`` key can have multple values, you can pass multiple values as
Python iterables such as lists or tuples. For example::

    u = UrlHelper('/foo')
    u.update_query_data(bar=[1, 2, 3])
    u.query_string  # returns 'bar=1&bar=2&bar=3'

UrlHelper.overload_params(**kwargs)
-----------------------------------

This method adds query parameters. As its name suggests, it will not update
existing keys, but instead add new values for the existing parameters. Here is
a simple example::

    u = UrlHelper('/foo')
    u.overload_params(bar=1)  # /foo?bar=1
    u.overload_params(bar=2)  # /foo?bar=1&bar=2

UrlHelper.toggle_params(**kwargs)
-----------------------------------

This method adds or removes query parameters depending on whether they
already exist. It looks for both a matching parameter and value, and
adds new parameters using ``UrlHelper.overload_params``. Here is
a simple example::

    u = UrlHelper('/foo')
    u.toggle_params(bar=1)  # /foo?bar=1
    u.toggle_params(bar=1, foo=2)  # /foo?foo=2
    u.toggle_params(bar=1, bar=2)  # /foo?bar=1&bar=2&foo=2
    
UrlHelper.get_path()
--------------------

Returns the ``UrlHelper.path`` property. This method exist to help
customization of ``UrlHelper.get_full_path()`` method in subclasses. Other than
that, it's the same as using the ``path`` property.

UrlHelper.get_full_path(**kwargs)
---------------------------------

Returns the full path with query string and fragment identifier (if any). The
keyword arguments passed to this function are passed onto 
``UrlHelper.get_query_string()`` method, and therefore to
``QueryDict.urlencode()`` method. Not that using the ``safe`` parameters with
query parameter values that contain unicode characters _will_ break your app.
See [issue #4](https://bitbucket.org/monwara/django-url-tools/issue/4/) for
more information.

UrlHelper.get_full_quoted_path(**kwargs)
----------------------------------------

Same as ``UrlHelper.get_full_path()`` method, but returns the full path quoted
so that it can be used as an URL parameter value. As with ``get_full_path()``
method, the arguments are passed onto ``QueryDict.urlencode()`` method, and the
same restriction about the unicode values in query parameters apply when using
the ``safe`` argument.

UrlHelper.del_param(param)
--------------------------

Delete a single query parameter. ::

    u = UrlHelper('/foo?bar=1&baz=2')
    u.del_param('baz')
    u.get_full_path() # returns '/foo?bar=1'

UrlHelper.del_params(*params, **kwargs)
---------------------------------------

Delete multiple parameters. If no parameters are specified, _all_ parameters
are removed. You can also specify a set of key-value pairs to remove specific
parameters with specified _values_. Here are a few examples::

    u = UrlHelper('/foo?bar=1&baz=2&foo=3')
    u.del_params('foo', 'bar')
    u.get_full_path() # returns '/foo?baz=2'

    u = UrlHelper('/foo?bar=1&baz=2&foo=3')
    u.del_params()
    u.get_full_path() # returns '/foo'

    u = UrlHelper('/foo?bar=1&bar=2')
    u.del_params(bar=2)
    u.get_full_path() # returns '/foo?bar=1'

ContextProcessors
=================

current_url
-----------

The ``current_url`` context processor will add a new variable to the template's
context.  This variable is called ``current_url``, and it's an ``UrlHelper``
instance.  Therefore, this variable has all the properties and methods of the
``UrlHelper`` class. For instance, if we are currently on ``/foo/bar?baz=1``
path, you can do the following in a template::

    {{ current_url.query_string }} {# renders `baz=1` #}
    {{ current_url.get_path }} {# renders `/foo/bar` #}

and so on. The variable itself renders as full relative path with query string
and fragment identifier (identical to output of ``UrlHelper.get_full_path()``
method).

Template tags
=============

To use the template tags, first load the ``urls`` library::

    {% load urls %}

URL tools currently has only one template tag, which is an assignment tag.

{% add_params %}
----------------

This template tag outputs a path with query string parameters given as keyword
arguments. For instance, if we are on a page at ``/foo``, we can use this tag::

    {% add_params request.get_full_path foo='bar' %}

and the output would be::

    /foo?foo=bar

Existing URL parameters are overridden by the ones specified as keyword
arguments.

{% overload_params %}
---------------------

Similar to ``{% add_params %}`` tag, except that it does not update existing
parameters but overloads them with new values. For example, if we are on a
page at ``/foo?bar=1``, we can use this tag like so::

    {% overload_params request.get_full_path bar=2 %}

and the output would be::

    /foo?bar=1&bar=2

{% del_params %}
----------------

This tag outputs a path stripped of specified parameters, or all query 
parameters if none are specified. If you use keyword arguments, only the
specified name-value pairs will be removed.

For example, if we are on the ``/foo?bar=1&bar=2&baz=2`` URL::

    {% del_param request.get_full_path 'bar' %}

outputs::

    /foo?baz=2

and ::

    {% del_params request.get_full_path %}

outputs::

    /foo

Finally::

    {% del_params request.get_full_path bar=2 %}

outputs::

    /foo?bar=1&baz=2

{% toggle_params %}
---------------------

This tag adds or removes parameters, depending on whether the parameter and
value exists.  For example, if we are on a page at ``/foo?bar=1``, we can 
toggle the state of ``bar=1`` using::

    {% toggle_params request.get_full_path bar=1 %}

and the output would be::

    /foo
    
If we are on a page at ``/foo``, then the output of would be::

    /foo?bar=1
    
Multiple parameters and values can be used.  For example, on a page at
``/foo?bar=1&foo=2``, parameters can be toggled like this::

    {% toggle_params request.get_full_path bar=3 foo=2 foo=3 %}
    
To give::

    /foo?bar=1&bar=3&foo=3

{% url_params %}
----------------

This tag is used as an assignment tag. Its first argument is an URL, followed
by any number of keyword arguments that represent the URL parameters. For
example, if we are requesting a page on ``'/foo'`` path, and do this::

    {% url_params request.get_full_path foo='bar' as new_url %}

We can use the ``new_url`` variable from that point on, that represents the
``/foo?foo=bar`` URL. To use this with your configured URLs, you can use the
built-in ``url`` tag::

    {% url 'foo' as foo_url %}
    {% url_arams foo_url foo='bar' as foo_url %}

If the reverse match for ``'foo'`` is, say, ``'/foo'``, then the ``foo_url``
variable will, predictably, contain ``'/foo?foo=bar'``.

This tag will override existing parameters rather than adding new values for
existing keywords. Therefore, you can safely use it to set URL parameters
whether they exist or not. This is typically useful when you are building URLs
for controls like pagers. Regardless of whether there is a ``page`` parameter
or not, setting it with ``url_params`` tag will correctly set the parameter to
desired value::

    {% url_params current_url page=2 %}
    {# this works for both ``/foo?page=1`` and just ``/foo`` #}

Template filters
================

URL tools also include filters for manipulating data that will be used as part
of URLs. To use them, you need to load the ``urls`` library first::

    {% load urls %}

quote
-----

The ``quote`` filter quotes URL parameters. It accepts optional safe characters
that can be used to prevent quoting of certain characters. This filter uses
`urllib.quote`_ for quoting. Safe characters inlude only the slash ``/`` by
default. ::

    {{ value|quote:"~/" }}

Note that using unicode characters with this filter may not work.

quote_plus
----------

The ``quote_plus`` filter is similart ot the `quote`_ filter, except that it
converts all spaces to ``+``. This filter also takes optional safe 
characters. The filter uses `urllib.quote_plus`_ for quoting. ::

    {{ value|quote_plus }}

Note that using unicode characters with this filter may not work.

Reporting bugs
==============

Please report any bugs to our BitBucket `issue tracker`_.

Contributors
============

We thank the following contributors (in no particular order):

+ nlaurance_ for contributing the ``overload_params`` and improvements to
  ``del_params``, as well as compatibility with Django 1.4.x.
+ `David Townshend`_ for ``toggle_param`` method and tag.
+ `Marc Kirkwood`_ for fix on ``update_query_data``.
+ `iaa`_ for unicode-related unit tests

.. _Django documentation on QueryDict: https://docs.djangoproject.com/en/dev/ref/request-response/?from=olddocs#querydict-objects
.. _issue tracker: https://bitbucket.org/monwara/django-url-tools/issues
.. _urllib.quote: http://docs.python.org/2/library/urllib.html#urllib.quote
.. _urllib.quote_plus: http://docs.python.org/2/library/urllib.html#urllib.quote_plus
.. _nlaurance: https://bitbucket.org/nlaurance
.. _David Townshend: https://bitbucket.org/aquavitae
.. _Marc Kirkwood: https://bitbucket.org/trojjer
.. _iaa: https://github.com/iaa
