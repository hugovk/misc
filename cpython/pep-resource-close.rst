PEP: 799
Title: Add PyResource callback C API to release resources
Author: Victor Stinner <vstinner@python.org>
Status: Draft
Type: Standards Track
Content-Type: text/x-rst
Created: 24-Jul-2023
Python-Version: 3.13

Abstract
========

Add the ``PyResource`` structure to the C API: callback to close a
resource.

Add new variants using ``PyResource`` of functions returning pointer:

* ``PyResource_Close()``
* ``PyByteArray_AsStringRes()``
* ``PyBytes_AsStringRes()``
* ``PyCapsule_GetNameRes()``
* ``PyEval_GetFuncNameRes()``
* ``PyUnicode_AsUTF8AndSizeRes()``
* ``PyUnicode_AsUTF8Res()``

These functions keep the resource valid until ``PyResource_Close()`` is
being called.


Motivation
==========

Problem of dangling pointers
----------------------------

The Python C API has multiple functions returning pointers which give a
direct access to object contents. Python is not notified when the caller
is done with this pointer. Usually, the pointer becomes a **dangling
pointer** once the object is destroyed, and so the caller should ensure
that the object is not destroyed while it uses the pointer.

Using a dangling pointer may crash or not depending if the memory was
reused or not, and if the memory block was freed or not. The behavior is
not deterministic which makes such bug harder to detect and to fix.

The Python C API does not document well which functions can execute
arbitrary Python code and so can indirectly destroyed the used object.
For example, a simple comparison like ``PyObject_RichCompare`` can
execute arbitrary code if a method like ``__eq__()`` is overriden in
Python.

Cannot change Python internals
------------------------------

Functions returning pointers with no associated "close" function prevent
changing Python internals. For example, the ``PyUnicode_AsUTF8()``
function does cache the UTF-8 encoded string in the Python str object
and this cache cannot be removed.


Rationale
=========

Avoid exposing specific cleanup function
----------------------------------------

Having an unique ``PyResource_Close()`` API avoids having to expose many
specific cleanup functions, one per API returning a resource. For
example, Python 3.12 ``_PySequence_BytesToCharpArray()`` allocates
memory and requires calling a specific ``_Py_FreeCharPArray()`` function
to release the memory.

Using ``PyResource_Close()``, the ``_Py_FreeCharPArray()`` function does
not have to be exposed in the public C API.

Moving garbage collector
------------------------

The ``PyResource`` allows Python implementation with a moving garbage
collector, like PyPy, to pin an object in memory while a resource is
being used, and unpin it in ``PyResource_Close()``. Or a function can
create a copy and release the memory in ``PyResource_Close()``.
Currently, objects should always be pinned in memory.

Similar existing API
--------------------

The ``PyResource`` API is similar to the concept of opaque handles used
with Unix file descriptor (``close(fd)``), Windows ``HANDLE``
(``CloseHandle(handle)``), and HPy handle (``HPy_Close(handle)``).

It is also similar to Python ``PyBuffer_Release()`` API.


Specification
=============

PyResource API
--------------

::

    typedef struct {
        void (*close_func) (void *data);
        void *data;
    } PyResource;

    void PyResource_Close(PyResource *res);

PyResource_Close()
------------------

Implementation of the ``PyResource_Close()`` function::

    void PyResource_Close(PyResource *res)
    {
        if (res->close_func == NULL) {
            return;
        }
        res->close_func(res->data);
    }

Example
-------

Example holding a strong reference to an object::

    static void
    close_resource(void *data)
    {
        PyObject *obj = (PyObject*)data;
        Py_DECREF(obj);
    }

    static char*
    get_resource(PyResource *res)
    {
        PyObject *obj = PyUnicode_FromString("resource");
        if (obj == NULL) {
            // res is left uninitialized on purpose.
            // It must not be used in the error path.
            return NULL;
        }
        char *utf8 = PyUnicode_AsUTF8(obj);
        if (utf8 == NULL) {
            Py_DECREF(obj);
            return NULL;
        }
        res->close_func = close_resource;
        res->data = obj;  // strong reference
        return utf8;
    }

Example of ``get_resource()`` usage::

    void use_resource(void)
    {
        PyResource res;
        char *str = get_resource(&res);
        if (str == NULL) {
            // ... report the error ...
            return;
        }
        // ... use str ...
        PyResource_Close(&res);
    }

Function variants using PyResource
----------------------------------

Add the following functions:

* ``const char* PyBytes_AsStringRes(PyObject *op, PyResource *res)``:
  safe variant of ``PyBytes_AsString()``.
* ``char* PyByteArray_AsStringRes(PyObject *self, PyResource *res)``:
  safe variant of ``PyByteArray_AsString()``.
* ``const char* PyCapsule_GetNameRes(PyObject *capsule, PyResource *res)``:
  safe variant of ``PyCapsule_GetName()``.
* ``const char* PyEval_GetFuncNameRes(PyObject *func, PyResource *res)``:
  safe variant of ``PyEval_GetFuncName()``.
* ``const char* PyUnicode_AsUTF8Res(PyObject *unicode, PyResource *res)``:
  safe variant of ``PyUnicode_AsUTF8()``.
* ``const char* PyUnicode_AsUTF8AndSizeRes(PyObject *unicode, Py_ssize_t *psize, PyResource *res)``:
  safe variant of ``PyUnicode_AsUTF8AndSize()``.

These variants hold a strong reference to the object and so the returned
pointer is guaranteed to remain valid until the resource is closed by
``PyResource_Close()`` (delete the strong reference).

Functions left unchanged
------------------------

The following functions which return pointers are left unchanged: no
variant is planned to be added. Most of these functions are safe. For
the unsafe functions, variants using ``PyResource`` can be added later.

* The caller function must release the returned newly allocated memory
  block:

  * ``PyOS_double_to_string()``
  * ``PyUnicode_AsUTF8String()``
  * ``PyUnicode_AsWideCharString()``
  * ``Py_DecodeLocale()``, ``Py_EncodeLocale()``
  * Allocator functions like ``PyMem_Malloc()``

* Get static data (safe in CPython):

  * ``PyUnicode_GetDefaultEncoding()``
  * ``PyImport_GetMagicTag()``
  * ``Py_GetVersion()``
  * ``Py_GetPlatform()``
  * ``Py_GetCopyright()``
  * ``Py_GetCompiler()``
  * ``Py_GetBuildInfo()``
  * ``PyHash_GetFuncDef()``

* Thread local storage:

  * ``PyThread_tss_get()``
  * ``PyThread_get_key_value()``

* Misc functions:

  * ``PyBuffer_GetPointer()``: the caller must call
    ``PyBuffer_Release()``.
  * ``PyCapsule_Import()``:
    the caller must hold a reference to the capsule object.
  * ``Py_GETENV()`` and ``Py_GETENV()`` (``char*``):
    the pointer can become invalid if environment variables are changed.
  * ``PyType_GetSlot()``:
    the caller must hold a reference to the type object.
  * ``PyModule_GetState()``:
    the caller must hold a reference to the module object.
  * ``PyType_GetModuleState()``:
    the caller must hold a reference to the module object of the type
    object.

* Deprecated functions, planned for removal:

  * ``Py_GetExecPrefix()``
  * ``Py_GetPath()``
  * ``Py_GetPrefix()``
  * ``Py_GetProgramFullPath()``
  * ``Py_GetProgramName()``
  * ``Py_GetPythonHome()``

Backwards Compatibility
=======================

Only new functions added: existing API are not affected, and no function
is planned for deprecation.


Security Implications
=====================

Added APIs are safer since they make sure that resource remains valid
until ``PyResource_Close()`` is being called, and prevent a risk of race
conditions.


How to Teach This
=================

When a function returns a resource, like a pointer, the
``PyResource_Close()`` must be called to close the resource.


Reference Implementation
========================

* `Issue #106592 <https://github.com/python/cpython/issues/106592>`_:
  C API: Add PyResource API: generic API to "close a resource"
* `PR #107202 <https://github.com/python/cpython/pull/107202>`_


Rejected Ideas
==============

Use the existing PyBuffer_Release() API
---------------------------------------

The ``Py_buffer`` API already exists and can be modified to fit into
``PyResource_Close()`` use case. Pseudo-code::

    Py_buffer buffer;
    char *utf8 = PyUnicode_AsUTF8Res(str, &buffer);
    // ... use utf8 ...
    PyBuffer_Release(&buffer);

The ``Py_buffer`` structure has 11 members. They would be initialized
to:

* ``buf = NULL``
* ``obj = NULL``
* ``len = 0``
* ``itemsize = 1``
* ``readonly = 1``
* ``ndim = 1``
* ``format = NULL``
* ``shape = NULL``
* ``strides = NULL``
* ``suboffsets = NULL``
* ``internal = NULL``

The problem is that most of these members are irrelevant just to "close
a resource".  Moreover, the structure has no callback to close the
resource: it relies on the ``bf_releasebuffer`` protocol of the object
type (``PyTypeObject.tp_as_buffer.bf_releasebuffer``).

Another problem is that the ``Py_buffer`` structure is now part of the
stable ABI and so it became complicated to add new members.

Copy the resource at each call
------------------------------

Instead of caching the UTF-8 encoding string with
``PyUnicode_AsUTF8()``, the ``PyUnicode_AsUTF8String()`` function can be
used: it allocates a new string at each call. A similar idea can be
applied to other functions returning pointers.

The problem is that a memory copy is inefficient and can be slow,
especially if the resource is large.


References
==========

* `C API Working group: Issue #57
  <https://github.com/capi-workgroup/problems/issues/57>`_:
  Function must not return a pointer to content without an explicit
  resource management


Copyright
=========

This document is placed in the public domain or under the
CC0-1.0-Universal license, whichever is more permissive.
