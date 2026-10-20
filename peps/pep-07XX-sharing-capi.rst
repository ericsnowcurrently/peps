

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









PyAPI_FUNC(_PyXIData_t *) _PyXIData_New(void);
PyAPI_FUNC(void) _PyXIData_Free(_PyXIData_t *data);
PyAPI_FUNC(void) _PyXIData_Init(
PyAPI_FUNC(int) _PyXIData_InitWithSize(
PyAPI_FUNC(void) _PyXIData_Clear(PyInterpreterState *, _PyXIData_t *);
PyAPI_FUNC(PyObject *) _PyXIData_GetNotShareableErrorType(PyThreadState *);
PyAPI_FUNC(void) _PyXIData_SetNotShareableError(PyThreadState *, const char *);
PyAPI_FUNC(void) _PyXIData_FormatNotShareableError(
PyAPI_FUNC(_PyXIData_getdata_t) _PyXIData_Lookup(
PyAPI_FUNC(int) _PyObject_CheckXIData(
PyAPI_FUNC(int) _PyObject_GetXIDataNoFallback(
PyAPI_FUNC(int) _PyObject_GetXIData(
PyAPI_FUNC(PyObject *) _PyXIData_NewObject(_PyXIData_t *);
PyAPI_FUNC(int) _PyXIData_Release(_PyXIData_t *);
PyAPI_FUNC(int) _PyXIData_ReleaseAndRawFree(_PyXIData_t *);

PyAPI_FUNC(int) _PyObject_GetXIDataWrapped(
PyAPI_FUNC(int) _PyObject_GetXIDataWrappedWithObj(
PyAPI_FUNC(int) _PyXIData_SetUnwrapper(
PyAPI_FUNC(_PyXIData_unwrapper_t *) _PyXIData_GetUnwrapper(
//PyAPI_FUNC(int) _PyXIData_UnwrapperFromObj(
PyAPI_DATA(PyTypeObject) _PyXIDataWrapper_Type;
PyAPI_FUNC(int) _PyXIDataWrapper_Check(PyObject *);
PyAPI_FUNC(int) _PyXIDataWrapper_CheckExact(PyObject *);
PyAPI_FUNC(PyObject *) _PyXIDataWrapper_New(
PyAPI_FUNC(PyObject *) _PyXIDataWrapper_NewWithoutFallback(
PyAPI_FUNC(PyObject *) _PyXIDataWrapper_FromWrapFunc(
PyAPI_FUNC(PyObject *) _PyXIDataWrapper_FromWrapObj(
PyAPI_FUNC(PyObject *) _PyXIDataWrapper_FromData(

PyAPI_FUNC(int) _PyXI_UnwrapNotShareableError(


PyAPI_FUNC(int) _PyBytes_GetData(PyObject *, _PyBytes_data_t *);
PyAPI_FUNC(PyObject *) _PyBytes_FromData(_PyBytes_data_t *);
PyAPI_FUNC(PyObject *) _PyBytes_FromXIData(_PyXIData_t *);
PyAPI_FUNC(int) _PyBytes_GetXIData(
PyAPI_FUNC(_PyBytes_data_t *) _PyBytes_GetXIDataWrapped(
PyAPI_DATA(PyObject *) _PyPickle_LoadFromXIData(_PyXIData_t *);
PyAPI_FUNC(int) _PyPickle_GetXIData(
PyAPI_FUNC(PyObject *) _PyMarshal_ReadObjectFromXIData(_PyXIData_t *);
PyAPI_FUNC(int) _PyMarshal_GetXIData(
PyAPI_FUNC(PyObject *) _PyCode_FromXIData(_PyXIData_t *);
PyAPI_FUNC(int) _PyCode_GetXIData(
PyAPI_FUNC(int) _PyCode_GetScriptXIData(
PyAPI_FUNC(int) _PyCode_GetPureScriptXIData(
PyAPI_FUNC(PyObject *) _PyFunction_FromXIData(_PyXIData_t *);
PyAPI_FUNC(int) _PyFunction_GetXIData(




















int PyTypeObject.tp_share(obj, _PyXIData_t \*);

data types::

  typedef struct _xidata _PyXIData_t;
  typedef PyObject *(*xid_newobjfunc)(_PyXIData_t *);
  typedef void (*xid_freefunc)(void *);

  #define _PyXIData_DATA(DATA) ((DATA)->data)
  #define _PyXIData_OBJ(DATA) ((DATA)->obj)
  #define _PyXIData_INTERPID(DATA) ((DATA)->interpid)

  #define _PyXIData_SET_FREE(DATA, FUNC) \
      do { \
          (DATA)->free = (FUNC); \
      } while (0)
  #define _PyXIData_CHECK_FREE(DATA, FUNC) \
      ((DATA)->free == (FUNC))

  #define _PyXIData_SET_NEW_OBJECT(DATA, FUNC) \
      do { \
          (DATA)->new_object = (FUNC); \
      } while (0)
  #define _PyXIData_CHECK_NEW_OBJECT(DATA, FUNC) \
      ((DATA)->new_object == (FUNC))

  typedef int xidata_fallback_t;
  #define _PyXIDATA_XIDATA_ONLY (0)
  #define _PyXIDATA_FULL_FALLBACK (1)

  typedef int (*xidatafunc)(PyThreadState *, PyObject *, _PyXIData_t *);
  typedef int (*xidatafbfunc)(
          PyThreadState *, PyObject *, xidata_fallback_t, _PyXIData_t *);
  typedef struct {
      xidatafunc basic;
      xidatafbfunc fallback;
  } _PyXIData_getdata_t;

  typedef int (*xid_wrapfunc)(PyThreadState *, PyObject *,
                              PyObject **, _PyXIData_unwrapper_t*);
  typedef PyObject *(*xid_unwrapfunc)(PyThreadState *, void *, PyObject *);
  typedef struct _xidata_unwrapper {
      xid_unwrapfunc func;
      void *data;
      xid_freefunc free;
  } _PyXIData_unwrapper_t;
  typedef struct {
      // The _PyXIData_t fields that are swapped out for an unwrapper:
      void *data;
      xid_newobjfunc new_object;
      xid_freefunc free;
  } _PyXIData_wrapped_orig_t;

defining cross-interpreter data::

  PyAPI_FUNC(_PyXIData_t *) _PyXIData_New(void);
  PyAPI_FUNC(void) _PyXIData_Free(_PyXIData_t *data);
  PyAPI_FUNC(void) _PyXIData_Init(
          _PyXIData_t \*data,
          PyInterpreterState \*interp, void \*shared, PyObject \*obj,
          xid_newobjfunc new_object);
  PyAPI_FUNC(int) _PyXIData_InitWithSize(
          _PyXIData_t \*,
          PyInterpreterState \*interp, const size_t, PyObject \*,
          xid_newobjfunc);
  PyAPI_FUNC(void) _PyXIData_Clear(PyInterpreterState \*, _PyXIData_t \*);

getting cross-interpreter data::

  PyAPI_FUNC(PyObject *) _PyXIData_GetNotShareableErrorType(PyThreadState *);
  PyAPI_FUNC(void) _PyXIData_SetNotShareableError(PyThreadState *, const char *);
  PyAPI_FUNC(void) _PyXIData_FormatNotShareableError(
          PyThreadState *,
          const char *,
          ...);

  PyAPI_FUNC(_PyXIData_getdata_t) _PyXIData_Lookup(
          PyThreadState *,
          PyObject *);
  PyAPI_FUNC(int) _PyObject_CheckXIData(
          PyThreadState *,
          PyObject *);

  PyAPI_FUNC(int) _PyObject_GetXIDataNoFallback(
          PyThreadState *,
          PyObject *,
          _PyXIData_t *);
  PyAPI_FUNC(int) _PyObject_GetXIData(
          PyThreadState *,
          PyObject *,
          xidata_fallback_t,
          _PyXIData_t *);

.. XXX bytes, code, function, marshal, pickle?

using cross-interpreter data::

  PyAPI_FUNC(PyObject *) _PyXIData_NewObject(_PyXIData_t *);
  PyAPI_FUNC(int) _PyXIData_Release(_PyXIData_t *);
  PyAPI_FUNC(int) _PyXIData_ReleaseAndRawFree(_PyXIData_t *);

wrapping cross-interpreter data::

  PyAPI_FUNC(int) _PyObject_GetXIDataWrapped(
          PyThreadState *tstate,
          PyObject *,
          xid_wrapfunc,
          _PyXIData_t *);
  PyAPI_FUNC(int) _PyObject_GetXIDataWrappedWithObj(
          PyThreadState *tstate,
          PyObject *,
          PyObject *,  // wrap(obj) -> (C-shareable obj, unwrap)
          _PyXIData_t *);

un-wrapping cross-interpreter data::

  PyAPI_FUNC(int) _PyXIData_SetUnwrapper(
          PyThreadState *,
          _PyXIData_t *,
          _PyXIData_unwrapper_t *);
  PyAPI_FUNC(_PyXIData_unwrapper_t *) _PyXIData_GetUnwrapper(
          _PyXIData_t *,
          _PyXIData_wrapped_orig_t *);

XIData wrapper objects::

  PyAPI_DATA(PyTypeObject) _PyXIDataWrapper_Type;

  PyAPI_FUNC(int) _PyXIDataWrapper_Check(PyObject *);
  PyAPI_FUNC(int) _PyXIDataWrapper_CheckExact(PyObject *);

  PyAPI_FUNC(PyObject *) _PyXIDataWrapper_New(
          PyThreadState *,
          PyObject *);
  PyAPI_FUNC(PyObject *) _PyXIDataWrapper_NewWithoutFallback(
          PyThreadState *,
          PyObject *);
  PyAPI_FUNC(PyObject *) _PyXIDataWrapper_FromWrapFunc(
          PyThreadState *,
          PyObject *,
          xid_wrapfunc);
  PyAPI_FUNC(PyObject *) _PyXIDataWrapper_FromWrapObj(
          PyThreadState *,
          PyObject *,
          PyObject *);  // same as _PyObject_GetXIDataWrappedWithObj()
  PyAPI_FUNC(PyObject *) _PyXIDataWrapper_FromData(
          PyThreadState *,
          _PyXIData_t *);
