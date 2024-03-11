Headers::

    PEP: xxx
    Title: Add Py_COMPAT_API_VERSION to Python C API
    Author: Victor Stinner <vstinner@python.org>
    Discussions-To: https://github.com/capi-workgroup/api-evolution/issues/24
    Status: Draft
    Type: Standards Track
    Created: 11-Mar-2024
    Python-Version: 3.13
    Post-History: `11-Oct-2023 <https://github.com/capi-workgroup/api-evolution/issues/24>`__,
                  `23-Jun-2023  <https://github.com/capi-workgroup/problems/issues/54>`__,


Abstract
========

Add ``Py_COMPAT_API_VERSION`` and ``Py_COMPAT_API_VERSION_MAX`` macros
to opt-in for planned incompatible C API changes in a C extension.
Maintainers can decide when they make their C extension compatible
and also decide which future Python version they want to be compatible
with.


Rationale
=========

Python enforces C API changes
-----------------------------

Every Python 3.x release has a long list of C API changes, including
incompatible changes. C extensions have to be updated to work on the
newly released Python.

Some incompatible changes are driven by new features: they cannot be
avoided, unless we decide to not add these features. Some incompatible
changes are driven by other reasons:

* Remove deprecated API (see :pep:`387`).
* Ease the implementation of another change.
* Change or remove error-prone API.
* Other reasons.

Currently, there is no middle ground between "not change the C API" and
"incompatible C API changes impact everybody". Either a C extension is
updated or Python cannot be updated. Such all-or-nothing deal does not
satisfy C extension maintainers nor C extensions users.


Limited C API
-------------

The limited C API is versioned: the ``Py_LIMITED_API`` macro can be set
to a Python version to select which API is available. On the Python
side, it allows introducing incompatible changes at a specific
``Py_LIMITED_API`` version. For example, if ``Py_LIMITED_API`` is set to
Python 3.11 or newer, the ``<stdio.h>`` is no longer included by
``Python.h``.

The difference here is that upgrading Python does not change if
``<stdio.h>`` is included or not, but updating ``Py_LIMITED_API`` does,
and updating ``Py_LIMITED_API`` is an action made by C extension
maintainer. It gives more freedom to decide **when** the maintainer is
ready to deal with the latest batch of incompatible changes.

A similiar version can be used with the regular (non-limited) C API.


Deprecation and compiler warnings
---------------------------------

Some C API changes are scheduled over 3 years: deprecations. For
example, a function is deprecated in Python 3.11 and only removed in
Python 3.13. Deprecated functions are marked with ``Py_DEPRECATED()``
which emits a compiler warning. The problem is that ``pip`` and
``build`` tools hide compiler logs by default, unless a build fails.
Moreover, it's easy to miss a single warning in the middle of hundred
logs lines.

Schedule changes
----------------

Currently, there is no way to schedule a C API change: announce it but
also provide a test to C extension maintainers to test their C
extensions with the change. Either a change is not made, or everybody
must update their code if they want to update Python.

Again, making incompatible changes "immediately" during at single Python
release does not satisfy C API consumers and C extensions users.


Specification
=============

New macros
----------

Add new ``Py_COMPAT_API_VERSION`` and ``Py_COMPAT_API_VERSION_MAX``
macros. They can be set to test if a C extension is prepared for future
C API changes: compatible with future Python versions.

The ``Py_COMPAT_API_VERSION`` macro can be set to a specific Python
version. For example, ``Py_COMPAT_API_VERSION=0x030e0000`` tests C API
changes scheduled in Python 3.14.

If the ``Py_COMPAT_API_VERSION`` macro is set to
``Py_COMPAT_API_VERSION_MAX``, all scheduled C API changes are tested at
once.

If the ``Py_COMPAT_API_VERSION`` macro is not set, it is to
``PY_VERSION_HEX`` by default.

The ``Py_COMPAT_API_VERSION`` macro can be set in a single C file or for
a whole project in compiler flags. The macro does not affected other
projects or Python itself, its scope is "local".


Example in Python
-----------------

For example, the ``PyImport_ImportModuleNoBlock()`` function is
deprecated in Python 3.13 and scheduled for removal in Python 3.15. The
function can be declared in the Python C API with the following
declaration::

    #if Py_COMPAT_API_VERSION < 0x030f0000
    Py_DEPRECATED(3.13) PyAPI_FUNC(PyObject *) PyImport_ImportModuleNoBlock(
        const char *name            /* UTF-8 encoded string */
        );
    #endif

Goals
-----

* Reduce the number of C API changes affecting C extensions when
  updating Python.
* When testing C extensions (ex: optional CI test),
  ``Py_COMPAT_API_VERSION`` can be set to ``Py_COMPAT_API_VERSION_MAX``
  to detect future incompatibilities. For mandatory tests, it is
  recommended to set ``Py_COMPAT_API_VERSION`` to a specific Python
  version.
* For core developers, make sure that the C API can still evolve
  without being afraid of breaking an unknown number of C extensions.

Non-goals
---------

* Freeze the API forever: this is not the stable ABI. For example,
  deprecated functions will continue to be removed on a regular basis.
* C extensions maintainers not using ``Py_COMPAT_API_VERSION`` will
  still be affected by C API changes when updating Python.
* Provide a stable ABI: the macro only impacts the API.
* Silver bullet solving all C API issues.


Examples of ``Py_COMPAT_API_VERSION`` usages
============================================

* Remove deprecated functions.
* Remove deprecated members of a structure, such as
  ``PyBytesObject.ob_shash``.
* Remove a standard ``#include``, such as ``#include <string.h>``,
  from ``<Python.h>``.
* Change the behavior of a function or a macro. For example, calling
  ``PyObject_SetAttr(obj, name, NULL)`` can fail, to enforce the usage
  of the ``PyObject_DelAttr()`` function instead to delete an attribute.


Implementation
==============

* `Issue gh-116587 <https://github.com/python/cpython/issues/116587>`_
* PR: `Add Py_COMPAT_API_VERSION and Py_COMPAT_API_VERSION_MAX macros
  <https://github.com/python/cpython/pull/116588>`_


Backwards Compatibility
=======================

There is no impact on backward compatibility.

Adding ``Py_COMPAT_API_VERSION`` and ``Py_COMPAT_API_VERSION_MAX``
macros has no effect on backward compatibility. Only developers setting
the ``Py_COMPAT_API_VERSION`` macro in their project will be impacted by
effects of this macro which is the expected behavior.


Discussions
===========

* C API Evolutions: `Macro to hide deprecated functions
  <https://github.com/capi-workgroup/api-evolution/issues/24>`_
  (October 2023)
* C API Problems: `Opt-in macro for a new clean API? Subset of functions
  with no known issues
  <https://github.com/capi-workgroup/problems/issues/54>`_
  (June 2023)


Prior Art
=========

* ``Py_LIMITED_API`` macro of `PEP 384 – Defining a Stable ABI
  <https://peps.python.org/pep-0384/>`_.
* Rejected `PEP 606 – Python Compatibility Version
  <https://peps.python.org/pep-0606/>`_ which has a global scope.


Copyright
=========

This document is placed in the public domain or under the
CC0-1.0-Universal license, whichever is more permissive.
