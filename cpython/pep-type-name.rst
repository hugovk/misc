PEP: Unify type name formatting

::

    PEP: xxx
    Title: Unify type name formatting
    Author: Victor Stinner <vstinner@python.org>
    Status: Draft
    Type: Standards Track
    Content-Type: text/x-rst
    Created: 29-Nov-2023
    Python-Version: 3.13


Abstract
========

Add new convenient APIs to format type names the same way in Python and
in C. No longer format type names differently depending on how types are
implemented. Also, put an end to truncating type names in C. The new C
API is compatible with the limited C API.

Rationale
=========

Standard library
----------------

In the Python standard library, formatting a type name or the type name
of an object is a common operation to format an error message and to
implement a ``__repr__()`` method. There are different ways to format a
type name which give different outputs.

Example with the ``datetime.timedelta`` type:

* The type short name (``type.__name__``) and the type qualified name
  (``type.__qualname__``) are ``'timedelta'``.
* The type module (``type.__module__``) is ``'datetime'``.
* The type fully qualified name is ``'datetime.timedelta'``.


Python code
^^^^^^^^^^^

In Python, ``type.__name__`` gets the type "short name", whereas
``f"{type.__module__}.{type.__qualname__}"`` formats the type "fully
qualified name. Usually, ``type(obj)`` or ``obj.__class__`` are used to
get the type of the object *obj*. Sometimes, the type name is put
between quotes.

Examples:

* ``raise TypeError("str expected, not %s" % type(value).__name__)``
* ``raise TypeError("can't serialize %s" % self.__class__.__name__)``
* ``name = "%s.%s" % (obj.__module__, obj.__qualname__)``

Qualified names were added to types (``type.__qualname__``) in Python
3.3 by `PEP 3155 – Qualified name for classes and functions
<https://peps.python.org/pep-3155/>`_.

C code
^^^^^^

In C, the most common way to format a type name is to get the
``PyTypeObject.tp_name`` member of the type. Example::

    PyErr_Format(PyExc_TypeError, "globals must be a dict, not %.100s",
                 Py_TYPE(globals)->tp_name);

The type qualified name (``type.__qualname__``) is only used at a single
place, by the ``type.__repr__()`` implementation. Using
``Py_TYPE(obj)->tp_name`` is more convenient than calling
``PyType_GetQualName()`` which requires ``Py_DECREF()``. Moreover,
``PyType_GetQualName()`` was only added recently, in Python 3.11.

Some functions use ``%R`` to format a type name. Example::

    PyErr_Format(PyExc_TypeError,
                 "calling %R should have returned an instance "
                 "of BaseException, not %R",
                 type, Py_TYPE(value));


Using PyTypeObject.tp_name is inconsistent
------------------------------------------

The ``PyTypeObject.tp_name`` member is different depending on the type
implementation:

* Static types and heap types in C: type fully qualified name.
* Python class: type short name (``type.__name__``).

So using ``Py_TYPE(obj)->tp_name`` to format an object type name gives
a different output depending if a type is implemented in C or in Python.

It goes against `PEP 399 – Pure Python/C Accelerator Module
Compatibility Requirements <https://peps.python.org/pep-0399/>`__
principles which require code to behave the same way if written in
Python or in C.

Example::

    $ python3.12
    >>> import _datetime; c_obj = _datetime.date(1970, 1, 1)
    >>> import _pydatetime; py_obj = _pydatetime.date(1970, 1, 1)
    >>> my_list = list(range(3))

    >>> my_list[c_obj]  # C type
    TypeError: list indices must be integers or slices, not datetime.date

    >>> my_list[py_obj]  # Python type
    TypeError: list indices must be integers or slices, not date

The error message contains the type fully qualified name
(``datetime.date``) if the type is implemented in C, or the type short
name (``date``) if the type is implemented in Python.

The ``PyTypeObject.tp_name`` member should not be used to format a type
name in an error message.


Limited C API
-------------

The ``Py_TYPE(obj)->tp_name`` code cannot be used with the limited C
API, since the ``PyTypeObject`` members are excluded from the limited C
API.

The type name should be read using ``PyType_GetName()``,
``PyType_GetQualName()`` and ``PyType_GetModule()`` functions which are
less convenient to use.


Truncating type names in C
--------------------------

In 1998, when the ``PyErr_Format()`` function was added, the
implementation used a fixed buffer of 500 bytes. The function came
with the comment::

    /* Caller is responsible for limiting the format */

In 2001, the function was modified to allocate a dynamic buffer on the
heap. Too late, the practice of truncating type names, like using the
``%.100s`` format, already became a habit, and developers forgot why
type names are truncated. In Python, type names are not truncated.

Truncating type names in C but not in Python goes against `PEP 399 –
Pure Python/C Accelerator Module Compatibility Requirements
<https://peps.python.org/pep-0399/>`__ principles which require code to
behave the same way if written in Python or in C.

See the issue: `Replace %.100s by %s in PyErr_Format(): the arbitrary
limit of 500 bytes is outdated
<https://github.com/python/cpython/issues/55042>`__ (2011).


Specification
=============

* Add ``type.__fully_qualified_name__`` attribute.
* Add ``%T`` and ``%#T`` formats to ``PyUnicode_FromFormat()``.
* Add ``PyType_GetFullyQualifiedName()`` function.
* Recommend not truncating type names.

Python API
----------

Add ``type.__fully_qualified_name__`` read-only attribute, the fully
qualified name of a type: similar to
``f"{type.__module__}.{type.__qualname__}"`` or ``type.__qualname__`` if
``type.__module__`` is not a string or is equal to ``"builtins"``.

Add PyUnicode_FromFormat() formats
----------------------------------

Add ``%T`` and ``%#T`` formats to ``PyUnicode_FromFormat()`` to format
a type name:

* ``%T`` formats ``type.__name__``: the type "short name"
* ``%#T`` formats ``type.__fully_qualified_name__``: the type "fully
  qualified name".

Both formats expect a type object.

The hash character (``#``) in the format string stands for
`alternative format
<https://docs.python.org/3/library/string.html#format-specification-mini-language>`_.
For example, ``f"{123:x}"`` returns ``'7b'`` and ``f"{123:#x}"`` returns
``'0x7b'`` (``#`` adds ``0x`` prefix).

The ``%T`` format is used by ``time.strftime()``, but it's not used by
``printf()``.

For example, the code::

    PyErr_Format(PyExc_TypeError,
                 "__format__ must return a str, not %.200s",
                 Py_TYPE(result)->tp_name);

can be replaced with::

    PyErr_Format(PyExc_TypeError,
                 "__format__ must return a str, not %T",
                 Py_TYPE(result));

Advantages of the updated code:

* The ``PyTypeObject.tp_name`` member is no longer read explicitly: the
  code becomes compatible with the limited C API.
* The ``PyTypeObject.tp_name`` bytes string no longer has to be decoded
  from UTF-8 at each ``PyErr_Format()`` call, since
  ``type.__fully_qualified_name__`` is already a Unicode string.
* The type name is no longer truncated.

Add PyType_GetFullyQualifiedName() function
-------------------------------------------

Add the ``PyType_GetFullyQualifiedName()`` function to get the fully
qualified name of a type (``type.__fully_qualified_name__``).

API::

    PyObject* PyType_GetFullyQualifiedName(PyTypeObject *type)

On success, return a new reference to the string. On error, raise an
exception and return ``NULL``.


Recommend not truncating type names
-----------------------------------

Type names must not be truncated. For example, the ``%.100s`` format
should be avoided: use the ``%s`` format instead (or ``%T`` and ``%#T``
formats in C).


Implementation
==============

* Pull request: `Add type.__fully_qualified_name__ attribute <https://github.com/python/cpython/pull/112133>`_.
* Pull requets: `Add %T format to PyUnicode_FromFormat() <https://github.com/python/cpython/pull/111703>`_.


Backwards Compatibility
=======================

Only new APIs are added. No existing API is modified. Changes are fully
backward compatible.


Rejected Ideas
==============

Change str(type)
----------------

The ``type.__str__()`` method can be modified to format a type name
differently. For example, to format the fully qualified name.

The problem is that it's a backward incompatible change. For example,
``enum``, ``functools``, ``optparse``, ``pdb`` and ``xmlrpc.server``
modules of the standard library have to be updated. And
``test_dataclasses``, ``test_descrtut`` and ``test_cmd_line_script``
have to be updated as well.

See the `pull request: type(str) returns the fully qualified name
<https://github.com/python/cpython/pull/112129>`_.


Add formats to type.__format__()
--------------------------------

Examples of proposed formats for ``type.__format__()``:

* ``f"{type(obj):z}"`` formats ``type(obj).__name__``.
* ``f"{type(obj):M.T}"`` formats ``type(obj).__fully_qualified_name__``.
* ``f"{type(obj):M:T}"`` formats ``type(obj).__fully_qualified_name__``
  using colon (``:``) separator.
* ``f"{type(obj):T}"`` formats ``type(obj).__name__``.
* ``f"{type(obj):#T}"`` formats ``type(obj).__fully_qualified_name__``.

Using short format (such as ``z``, a single letter) requires to refer to
format documentation to understand how a type name is formatted, whereas
``type(obj).__name__`` is explicit.

The dot character (``.``) is already used for the "precision" in format
strings. The colon character (``:``) is already used to separated the
expression from the format specification. For example, ``f"{3.14:g}"``
uses ``g`` format which comes after the colon (``:``). Usually, a format
type is a single letter, such as ``g`` in ``f"{3.14:g}"``, not ``M.T``
or ``M:T``. Reusing dot and colon characters for a different purpose can
be misleading and make the format parser more complicated.

Add !t formatter to get an object type
--------------------------------------

Use ``f"{obj!t:T}"`` to format ``type(obj).__name__``, similar to
``f"{type(obj).__name__}"``.

When the ``!t`` formatter was proposed in 2018, `Eric Smith was opposed
to this
<https://mail.python.org/archives/list/python-dev@python.org/message/BMIW3FEB77OS7OB3YYUUDUBITPWLRG3U/>`_.
Eric is the author of `PEP 498 f-string
<https://peps.python.org/pep-0498/>`_.


Add formats to str % args
-------------------------

It was proposed to add formats to format a type name in ``str % arg``.
For example, ``%T`` and ``%#T`` formats.

Nowadays, f-string is preferred for new code.


Use colon separator in fully qualified name
-------------------------------------------

The colon (``:``) separator eliminates guesswork when you want to import
the name, see ``pkgutil.resolve_name()``. A type fully qualified name
can be formatted as ``f"{type.__module__}:{type.__qualname__}"``, or
``type.__qualname__`` if the module is ``"builtins"``.

In the standard library, no code formats a type fully qualified name
this way.

It is already tricky to get a type from its qualified name. The type
qualified name already uses the dot (``.``) separator between different
parts: class name, ``<locals>``, nested class name, etc.

The colon separator is not consistent with dot separator used in a
module fully qualified name (``module.__name__``).


Other ways to format type names in C
------------------------------------

The ``printf()`` function supports multiple size modifiers: ``hh``
(``char``), ``h`` (``short``), ``l`` (``long``), ``ll`` (``long long``),
``z`` (``size_t``), ``t`` (``ptrdiff_t``) and ``j`` (``intmax_t``).
The ``PyUnicode_FromFormat()`` function supports most of them.

Proposed formats using ``h`` and ``hh`` length modifiers:

* ``%hhT`` formats ``type.__name__``.
* ``%hT`` formats ``type.__qualname__``.
* ``%T`` formats ``type.__fully_qualified_name__``.

Other proposed formats:

* ``%Q``
* ``%t``. printf() now uses ``t`` as a length modifier for a
  ``ptrdiff_t`` argument.
* ``%lT`` formats ``type.__fully_qualified_name__``.
* ``%Tn`` formats ``type.__name__``.
* ``%Tq`` formats ``type.__qualname__``.
* ``%Tf`` formats ``type.__fully_qualified_name__``.

Having more options to format type names can lead to inconsistencies
between different modules and make the API more error prone.

Length modifiers are used to specify the C type of the argument, not to
change how an argument is formatted. The alternate form (``#``) changes
how an argument is formatted. Here the argument C type is always
``PyObject*``.

``type.__qualname__`` can be used in Python and ``PyType_GetQualName()``
can be used in C to format a type qualified name.


Omit Py_TYPE() with %T format: pass an instance
-----------------------------------------------

It was proposed to format a type name from a instance, like::

    PyErr_Format(..., "type name: %T", obj);

The intent is to avoid ``Py_TYPE()`` which returns a borrowed reference
to the type. Using a borrowed referencen can cause bug or crash if the
type is finalized or deallocated while being used.

In practice, it's unlikely that a type is finalized while the error
message is formatted. Instances of static types cannot have their type
being deallocated: static types are never deallocated. Since Python 3.8,
instances of heap types hold a strong reference to their type (in
``PyObject.ob_type``) and it's safe to make the assumption that the code
holds a strong reference to the formatted object, so the object type
cannot be deallocated.

In short, it is safe to use ``Py_TYPE(obj)`` borrowed reference while
formatting an error message.

If the ``%T`` format expects an instance, formatting a type cannot use
the ``%T`` format, whereas it's a common operation in stdlib C
extensions. The ``%T`` format would only cover half of cases (only
instances). If the ``%T`` format takes a type, all cases are covered
(types and instances using ``Py_TYPE()``).


Other proposed APIs to get a type fully qualified name
------------------------------------------------------

* ``type.__fullyqualname__`` attribute name: attribute without underscore
  between words. Several dunders, including some of the most recently
  added ones, include an underscore in the word: ``__class_getitem__``,
  ``__release_buffer__``, ``__type_params__``, ``__init_subclass__`` and
  ``__text_signature__``.
* ``type.__fqn__`` attribute name (Fully Qualified Name: FDN).
* Add a function to the ``inspect`` module. Need to import the
  ``inspect`` module to use it.


Omit __main__ module in the type fully qualified name
-----------------------------------------------------

The ``pdb`` module formats a type fully qualified names in a similar way
than proposed ``type.__fully_qualified_name__``, but it omits the module
if the module is equal to ``"__main__"``.

The ``unittest`` module and a lot of existing stdlib code format a type
fully qualified names the same way than proposed
``type.__fully_qualified_name__``: only omits the module if the module
is equal to ``"builtins"``.

It's possible to omit the ``"__main__."`` prefix for the ``__main__``
module with::

    def format_type(cls):
        if cls.__module__ != "__main"__:
            return cls.__fully_qualified_name__
        else:
            return cls.__qualname__


Discussions
===========

* Discourse: `Enhance type name formatting when raising an exception:
  add %T format in C, and add type.__fullyqualname__
  <https://discuss.python.org/t/enhance-type-name-formatting-when-raising-an-exception-add-t-format-in-c-and-add-type-fullyqualname/38129>`_
  (2023).
* Issue: `PyUnicode_FromFormat(): Add %T format to format the type name
  of an object <https://github.com/python/cpython/issues/111696>`_
  (2023).
* Issue: `C API: Investigate how the PyTypeObject members can be removed
  from the public C API
  <https://github.com/python/cpython/issues/105970>`_ (2023).
* python-dev thread: `bpo-34595: How to format a type name?
  <https://mail.python.org/archives/list/python-dev@python.org/thread/HKYUMTVHNBVB5LJNRMZ7TPUQKGKAERCJ/>`_
  (2018).
* Issue: `PyUnicode_FromFormat(): add %T format for an object type name
  <https://github.com/python/cpython/issues/78776>`_ (2018).
* Issue: `Replace %.100s by %s in PyErr_Format(): the arbitrary limit of
  500 bytes is outdated
  <https://github.com/python/cpython/issues/55042>`__ (2011).


Copyright
=========

This document is placed in the public domain or under the
CC0-1.0-Universal license, whichever is more permissive.
