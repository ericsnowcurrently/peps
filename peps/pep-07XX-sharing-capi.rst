

New C-API
=========

At the C layer, the fundamental problem to solve is sharing data
safely between two interpreters.  This refers to both thread-safety
(mutation) and managing lifecycles (deallocation).

The proposed C-API falls into three catagories:

1. users - passing objects/data between interpreters safely
2. types - supporting cross-interpreter instances
3. implementation - enables implementing the "_interpreters" module

The first two categories effectively operate on the same data type.
However, they use a different API prefix to reflect the distinction.

Using Cross-Interpreter Data
----------------------------

The following API facilitates passing object-derived data between
interpreters safely.  It is inspired by the :pep:`3118` buffer API.

Usage will typically follow this pattern::

   /* interpreter A */

   PyCrossInterpreterData *xid = PyObject_GetCrossInterpreterData(obj);
   if (xid == NULL) {
       return -1;
   }

   /* interpreter B */
   PyObject *obj = PyCrossInterpreterData_NewObject(xid);
   PyCrossInterpreterData_Release(xid);
   if (obj == NULL) {
       return -1;
   }

The new API:

::

   typedef struct xid PyCrossInterpreterData

The opaque type that encapsulates the information needed to safely
pass/share data between interpreters.

::

   int PyObject_CheckCrossInterpreterData(PyObject *obj)

Return true if the object's type supports "sharing".

::

   PyCrossInterpreterData * PyObject_GetCrossInterpreterData(PyObject *obj);

Return the cross-interpreter data corresponding to the object.

::

   PyObject * PyCrossInterpreterData_NewObject(PyCrossInterpreterData *xid)

Return a new object corresponding to the given data.

::

   int PyCrossInterpreterData_Release(PyCrossInterpreterData *xid)

Release the associated object, if any, and free the raw data,
if necessary.  Both tasks are performed in the original interpreter.
If that is the current interpreter then it happens immediately.
Otherwise it happens "soon" in any one of that interpreter's threads
where the eval loop is running.

Regardiess, the given "xid" is always deallocated after the object
and raw data have been released.

Defining a Cross-Interpreter Type
---------------------------------

For cross-interpreter data to be useful, there must be a way for types
to define how their instances are converted to cross-interpreter data
and back.

That involves:

1. a function to convert an instance of the type to raw data
2. a function to convert raw data back into an object

The new API:

::

   typedef struct xid Py_xid_t

The opaque type that encapsulates the information needed to safely
pass/share data between interpreters.

::

   typedef int (*xidfunc)(PyThreadState *, PyObject *, Py_xid_t *)

   /* for builtin static types and internal data */
   typedef struct {
       ...
       xidfunc tp_xid;
       ...
   } PyTypeObject

   /* for heap types */
   #define Py_tp_xid 82

For a type to define that its instances are shareable, it must have its
``tp_xid`` slot set.  This function takes the current thread state, an
instance of the type, and a cross-interpreter data object.  The function
must initialize the cross-interpreter data using the instance.  This
includes the raw data and the function to convert it back to an object.

::

   typedef PyObject *(*xid_newobjectfunc)(Py_xid_t *xid, void *data)

   void PyXID_InitWithData(Py_xid_t *xid, void *data,
                           xid_newobjectfunc new_object)

   void PyXID_InitWithObject(Py_xid_t *xid, void *data, PyObject *obj,
                             xid_newobjectfunc new_object)

   void * PyXID_InitWithSize(Py_xid_t *xid, const size_t size,
                             xid_newobjectfunc new_object);

In the ``tp_xid`` function, the type uses one of these functions
to initialize the cross-interpreter data.

``PyXID_InitWithData()`` is used when the raw data already exists,
but isn't associated with the object (or with the object's lifetime).
This can be useful in a number of situations, such as when the object
maps to an entry in a static lookup table or is a singleton.
No memory is allocated and the caller is responsible for cleaning up
(if needed).

``PyXID_InitWithObject()`` is used when the raw data comes from
(or is tied to the lifetime of) an object, like the underlying data
of a ``bytes`` object.  No memory is allocated, but it is expected
that the raw data will be deallocated when the object is deallocated.

``PyXID_InitWithSize()`` is used when the raw data will be filled in
dynamically afterward.  This implies it is completely independent
of the object's lifetime.

In each case, the "data" arg may be ``NULL``, but the "newobject"
function is required.  This function is called by
``PyCrossInterpreterData_NewObject()``.

::

   int64_t PyXID_GetInterpID(PyCrossInterpreterData *xid)

The "newobject" function set by the type may need additional data
stored in the cross-interpreter data.  These helper functions provide
that information.

Implementing the _interpreters Module
-------------------------------------

The implementation of the low-level ``_interpreters`` module has
demonstrated a need for various additional public API.  These would
be useful to anyone building their own library for managing
interpreters.

Existing "private"/internal API:

* ``PyInterpreterConfig_INIT`` (AKA ``_PyInterpreterConfig_INIT``)
* ``PyInterpreterConfig_LEGACY_INIT`` (AKA ``_PyInterpreterConfig_LEGACY_INIT``)
* ``PyInterpreterState_RequireIDRef()`` (AKA ``_PyInterpreterState_RequireIDRef()``)
* ``PyErr_SetFromPyStatus()`` (AKA ``_PyErr_SetFromPyStatus()``)
* ``PyErr_ChainExceptions()`` (AKA ``_PyErr_ChainExceptions1()``, see `gh-89101 <https://github.com/python/cpython/issues/89101>`_)
* ``PyArg_BadArgument()`` (AKA ``_PyArg_BadArgument()``)

New public API:

* ``PyInterpreterState_IsRunningMain()``
* ``PyInterpreterState_FailIfRunningMain()``
* ``PyInterpreterState_SetRunningMain()``
* ``PyInterpreterState_SetNotRunningMain()``

The above all relate to tracking if an interpreter is currently running
a thread against its ``__main__`` module.

New public API:

* ``PyThreadState_WHENCE_EXEC``
* ``PyThreadState_WHENCE_INTERP``
* ``PyThreadState_GetWhence()``
* ``PyThreadState_SetWhence()``


Open Questions
==============

* Is ``tp_xid`` a good name?
