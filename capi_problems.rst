
===============================
An Evaluation of Python's C API
===============================


Abstract
========

This document describes our shared view of the public C API, with an emphasis
on identifying problems. We aim to define its purposes, the different
stakeholders and their particular use cases and requirements, and to
outline the strengths and weaknesses of the C API. We do not propose
solutions to any of the problems. The intention is that this list of
issues will be used to guide the discussions about such proposals,
and to provide criteria by which to evaluate them.

Introduction
============

Python's C API was not designed for the purposes it currently fulfils.
It evolved from what was initially the internal API between the C code
of the interpreter and the Python language and libraries. In its first
incarnation, it was exposed to make it possible to embed Python into C/C++
applications and to write extension modules in C/C++.
These capabilities were instrumental to the growth of Python's ecosystem.
Over the decades, the C API grew to provide different tiers of stability,
conventions changed, and new usage patterns have emerged, such as bindings
to languages other than C/C++. In addition, CPython is no longer the only
implementation of the C API, and some of the design decisions made when
it was, are difficult for alternative implementations to work with
[`Issue 64 <https://github.com/capi-workgroup/problems/issues/64>`__].
Finally, lessons were learned and mistakes in both the design and the
implementation of the C API were identified.

Evolving the C API is hard due to the combination of backwards
compatibility constraints and its inherent complexity, both
technical and social. Different types of users bring different,
sometimes conflicting, requirements. The tradeoff between stability
and progress is an ongoing, highly contentious topic of discussion
when suggestions are made for incremental improvements.
Several proposals have been put forward for improvement, redesign
or replacement of the C API, each representing a deep analysis of
the problems.  At the 2023 Language Summit, three back-to-back
sessions were devoted to different aspects of the C API. There is
general agreement that a new design can remedy the problems that
the C API has accumulated over the last 30 years, while at the same
time updating it for use cases that it was not originally designed for.

However, there was also a sense at the Language Summit that we are
trying to discuss solutions without a clear common understanding
of the problems that we are trying to solve. It was decided that
we need to agree on the current problems with the C API, before
we are able to evaluate any of the proposed solutions. We
therefore created the
[`capi-workgroup <https://github.com/capi-workgroup/problems/issues/>`__]
repository on GitHub in order to collect everyone's ideas on that
question.

Over 60 different issues were created on that repository, each
describing a problem with the C API. They were categorized and
a number of recurring themes were identified. The sections below
mostly correspond to these themes, and each contains a combined
description of the issues raised in that category, along with
links to the individual issues. In addition, we included a section
that aims to identify the different stakeholders of the C API,
and the particular requirements that each of them has.


C API Stakeholders
==================

As mentioned in the introduction, the C API was originally
created as the internal interface between CPython's
interpreter and the Python layer. It was later exposed as
a way for third party developers to extend and embed Python
programs. Over the years, new types of stakeholders emerged,
with different requirements and areas of focus. This section
describes this complex state of affairs in terms of the
actions that different stakeholders need to perform through
the C API.

There are actions which are generic, and required by
all types of API users:

* Define functions and call them
* Create classes and instantiate them
* Create instances of builtin and user defined types
  and perform operations on them
* Introspect objects, including types, instances, and functions.
* Raise and handle exceptions
* Import modules
* Manage threads
* Manage sub-interpreters
* Handle and send signals

Groups of users are:

**Extension writers**

These are the traditional users of the C API, and their requirements
are listed above.

**Embedders**

Applications with an embedded Python interpreter. Examples are
`Blender <https://docs.blender.org/api/current/info_overview.html>`__ and
`OBS <https://obsproject.com/wiki/Getting-Started-With-OBS-Scripting>`__.

They need to be able to

* Configure the interpreter (import paths, inittab, sys.argv, ...)
* Interact with the execution model and program lifetime, including
  clean interpreter shutdown and restart
* Represent complex data models in a way Python can use without
  having to create deep copies.
* Provide and import frozen modules.
* Run multiple independent interpreters (in particular, when embedded
  in a library that wants to avoid global effects).

**Alternative Python Implementations**

Alternative implementations of Python (such as
`PyPy <https://www.pypy.org>`__,
`GraalPy <https://www.graalvm.org/python/>`__,
`IronPython <https://ironpython.net>`__,
`RustPython <https://github.com/RustPython/RustPython>`__,
`MicroPython <https://micropython.org>`__,
and `Jython <https://www.jython.org>`__), may take
very different approaches for the implementation of
different subsystems. They need:

* The API to be abstract and hide implementation details.
* A specification of the API, ideally with a test suite
  that ensures compatibility.
* It would be nice to have an ABI that can be shared
  across Python implementations.

**Alternative APIs**

There are several projects that implement alternatives to the
C API, which offer extension users advantanges over programming
directly with the C API. These APIs are implemented with the
C API, and in some cases by using cpython internals.
Some examples are
`Cython <https://cython.org>`__,
`HPy <https://hpyproject.org>`__ and
`pythoncapi-compat <https://pythoncapi-compat.readthedocs.io/en/latest/>`__.
CPython's DSL for parsing function arguments, the
`Argument Clinic <https://docs.python.org/3/howto/clinic.html>`__,
can also be seen as belonging to this category of stakeholders.

Such systems need minimal building blocks for accessing CPython
efficiently. They don't necessarily need an ergonomic API, because
they typically generate code that is not intended to be read
by humans. But they do need it to be comprehensive enough so that
they don't need to access internals, while offering them stability,
and without sacrificing performance.

An alternative is to have a fast API tier with less error checking
and lower stability guarantees. Then the developers and users of
these tools can choose whether to generate code that uses the
faster or the safer and more stable version of the API.

**Binding generators**

Libraries that create bindings between Python and other object models,
paradigms or languages, such as
`pybind11 <https://pybind11.readthedocs.io/en/stable/>`__ for C++11,
`PyO3 <https://github.com/PyO3/pyo3>`__ for Rust,
`PySide <https://pypi.org/project/PySide/>`__ for Qt,
`PyGObject <https://pygobject.readthedocs.io/en/latest/>`__ for GTK,
`Pygolo <https://gitlab.com/pygolo/py>`__ for Go,
`PyJNIus <https://github.com/kivy/pyjnius/>`__ for Java, or
`SWIG <https://swig.org/>`__ for C/C++.

They need to:

* Create custom objects (e.g. function/module objects
  and traceback entries) that match the behavior of equivalent
  Python code as closely as possible.
* Dynamically create objects which are static in traditional
  C extensions (e.g. classes/modules), and need CPython to manage
  their state and lifetime.
* Adapt foreign objects (strings, GC'd containers), with low overhead.
* Adapt external mechanisms, execution models and guarantees to the
  Python way (green threads/continuations, one-writer-or-multiple-readers
  semantics, virtual multiple inheritance, 1-based indexing, super-long
  inheritance chains, goroutines, channels, ...)

Strengths of the C API
======================

While the bulk of this document is devoted to problems with the
C API that we would like to see fixed in any new design, it is
also important to point out the strengths of the C API, and to
make sure that they are preserved.

As mentioned in the introduction, the C API enabled the
development and growth of the Python ecosystem over the last
three decades, while evolving to support use cases that it was
not originally designed for. This track record in itself is
indication of how effective and valuable it has been.

A number of specific strengths were mentioned in the
capi-workgroup discussions. Heap types were identified
as much safer and easier to use than static types
[`Issue 4 <https://github.com/capi-workgroup/problems/issues/4#issuecomment-1542324451>`__].

API functions that take a C string literal for lookups based
on a Python string are very convenient
[`Issue 30 <https://github.com/capi-workgroup/problems/issues/30#issuecomment-1550098113>`__].

The Limited API and stable ABI hide implementation details and
make it easier to evolve Python
[`Issue 30 <https://github.com/capi-workgroup/problems/issues/30#issuecomment-1560083258>`__].

API Evolution and Maintenance
=============================

The difficulty of making changes in the C API is central to this report. It is
implicit in many of the issues we discuss here, particularly when we need to
decide whether an incremental bugfix can resolve the issue, or whether it can
only be addressed as part of an API redesign
[`Issue 44 <https://github.com/capi-workgroup/problems/issues/44>`__]. The
benefit of each incremental change is often viewed as too small to justify the
disruption. Over time, this implies that every mistake we make in an API's
design or implementation remains with us indefinitely.

We can take two views on this issue. One is that this is a problem and the
solution needs to be baked into any new C API we design, in the form of a
process for incremental API evolution. The other possible approach is that
this is not a problem to be solved, but rather a feature of any API. In this
view, API evolution should not be incremental, but rather through large
redesigns, each of which learns from the mistakes of the past and is not
shackled by backwards compatibility requirements. A compromise approach
is somewhere between these two extremes, fixing issues which are easy
or important enough to tackle incrementally, and leaving others alone.

The problem we have in CPython is that we don't have an agreed, official
approach to API evolution. Different members of the core team are pulling in
different directions and this is an ongoing source of disagreements.
Any new C API needs to come with a clear decision about the model
that its maintenance will follow, as well as the technical and
organizational processes by which this will work.

If the model does include provisions for incremental evolution of the API,
it will include processes for managing the impact of the change on users
[`Issue 60 <https://github.com/capi-workgroup/problems/issues/60>`__],
perhaps through introducing an external backwards compatibility module
[`Issue 62 <https://github.com/capi-workgroup/problems/issues/62>`__],
or a new API tier of "blessed" functions
[`Issue 55 <https://github.com/capi-workgroup/problems/issues/55>`__].


API Specification and Abstraction
=================================

The C API does not have a formal specification, it is described
semi-formally in the documentation and exposed through C header
files. This creates a number of problems.

Bindings for languages other than C/C++ must parse C code
[`Issue 7 <https://github.com/capi-workgroup/problems/issues/7>`__].
Some C language features are hard to handle in this way, because
they produce compiler-dependent output (such as enums) or require
a C preprocessor/compiler rather than just a parser (such as macros)
[`Issue 35 <https://github.com/capi-workgroup/problems/issues/35>`__].

Furthermore, C header files tend to expose more than what is intended
to be part of the public API
[`Issue 34 <https://github.com/capi-workgroup/problems/issues/34>`__].
In particular, implementation details such as the fields of C structs
can be exposed
[`Issue 22 <https://github.com/capi-workgroup/problems/issues/22>`__
and `PEP 620 <https://peps.python.org/pep-0620/>`__].
This can make API evolution very difficult, in particular when it
occurs in the stable ABI as in the case of ``ob_refcnt`` and ``ob_type``,
which are accessed via the reference counting macros
[`Issue 45 <https://github.com/capi-workgroup/problems/issues/45>`__].

A deeper issue was identified in relation to the way that reference
counting is exposed. The way that C extensions are required to
manage references with calls to ``Py_INCREF`` and ``Py_DECREF`` is
specific to CPython's memory model, and is hard for alternative
Python implementations to emulate.
[`Issue 12 <https://github.com/capi-workgroup/problems/issues/12>`__].

Another set of problems arises from the fact that a ``PyObject*`` is
exposed in the C API as an actual pointer rather than a handle. The
address of an object serves as its ID and is used for comparison,
and this complicates matters for alternative Python implementations
that move objects during GC
[`Issue 37 <https://github.com/capi-workgroup/problems/issues/37>`__].

A separate issue is that object references are opaque to the runtime,
discoverable only through calls to ``tp_traverse``/``tp_clear``,
which have their own purposes. If there was a way for the runtime to
know the structure of the object graph, and keep up with changes in it,
this would make it possible for alternative implementations to implement
different memory management schemes
[`Issue 33 <https://github.com/capi-workgroup/problems/issues/33>`__].

Object Reference Management
===========================

There are C API functions that return borrowed references, and
functions that steal references to arguments, but there isn't a
naming convention that makes this obvious, so this is error prone
[`Issue 8 <https://github.com/capi-workgroup/problems/issues/8>`__
and `Issue 52 <https://github.com/capi-workgroup/problems/issues/52>`__].
The terminology used to describe these situations in the documentation
can also be improved
[`Issue 11 <https://github.com/capi-workgroup/problems/issues/11>`__].

A more radical change is necessary in the case of functions that
return borrowed references (such as ``PyList_GetItem``)
[`Issue 5 <https://github.com/capi-workgroup/problems/issues/5>`__ and
`Issue 21 <https://github.com/capi-workgroup/problems/issues/21>`__]
or pointers to parts of the internal structure of an object
(such as ``PyBytes_AsString``)
[`Issue 57 <https://github.com/capi-workgroup/problems/issues/57>`__].
In both cases, the reference/pointer is valid for as long as the
owning object is alive, but this time is hard to reason about. Such
functions should not exist in the API without a mechanism that can
make them safe.

For containers, the API is currently missing bulk operations on the
references of contained objects. This is particularly important for
a stable ABI where ``INCREF`` and ``DECREF`` cannot be macros, making
bulk operations expensive when implemented as a sequence of function
calls
[`Issue 15 <https://github.com/capi-workgroup/problems/issues/15>`__].

Type Definition and Object Creation
===================================

The C API has functions that make it possible to create incomplete
or inconsistent Python objects, such as ``PyTuple_New`` and
``PyUnicode_New``. This causes problem when the object is tracked
by GC or its ``tp_traverse``/``tp_clear`` functions are called.
Such functions should be removed from the C API. Related functions,
such as ``PyTuple_SetItem`` which is used to modify a partially
initialized tuple, should also be removed (tuples are immutable
once fully initialized)
[`Issue 56 <https://github.com/capi-workgroup/problems/issues/56>`__].

A few issues were identified with type definition APIs. For legacy
reasons, there is often a significant amount of code duplication
between ``tp_new`` and ``tp_vectorcall``
[`Issue 24 <https://github.com/capi-workgroup/problems/issues/24>`__].
The type slot function should be called indirectly, so that their
signatures can change to include context information
[`Issue 13 <https://github.com/capi-workgroup/problems/issues/13>`__].
Several aspects of the type definition and creation process are not
well defined, such as which stage of the process is responsible for
initializing and clearing different fields of the type object
[`Issue 49 <https://github.com/capi-workgroup/problems/issues/49>`__].

Error Handling
==============

Error handling in the C API is based on the error indicator which is stored
on the thread state (in global scope). The design intention was that each
API function returns a value indicating whether an error has occurred (by
convention, ``-1`` or ``NULL``). When the program knows that an error
occurred, it can fetch the exception object which is stored in the
error indicator. A number of problems were identified which are related
to error handling, pointing at APIs which are too easy to use incorrectly.

There are functions that do not report all errors that occur while they
execute. For example, ``PyDict_GetItem`` clears any errors that occur
when it calls the key's hash function, or while performing a lookup
in the dictionary
[`Issue 51 <https://github.com/capi-workgroup/problems/issues/51>`__].

Python code never executes with an in-flight exception (by definition),
and by the same token C API functions should never be called with the error
indicator set. This is currently not checked in most C API functions, and
there are places in the interpreter where error handling code calls a C API
function while an exception is set. For example, see the call to
``PyUnicode_FromString`` in the error handler of ``_PyErr_WriteUnraisableMsg``
[`Issue 2 <https://github.com/capi-workgroup/problems/issues/2>`__].


There are functions that do not return a value, so a caller is forced to
query the error indicator in order to identify whether an error has occurred.
An example is ``PyBuffer_Release``
[`Issue 20 <https://github.com/capi-workgroup/problems/issues/20>`__].
There are other functions which do have a return value, but this return value
does not unambiguously indicate whether an error has occurred. For example,
``PyLong_AsLong`` returns ``-1`` in case of error, or when the value of the
argument is indeed ``-1``
[`Issue 1 <https://github.com/capi-workgroup/problems/issues/1>`__].
In both cases, the API is error prone because it is possible that the
error indicator was already set before the function was called, and the
error is incorrectly attributed. The fact that the error was not detected
before the call is a bug in the calling code, but the behaviour of the
program in this case doesn't make it easy to identify and debug the
problem.

There are functions that take a ``PyObject*`` argument, with special meaning
when it is ``NULL``. For example, if ``PyObject_SetAttr`` receives ``NULL`` as
the value to set, this means that the attribute should be cleared. This is error
prone because it could be that ``NULL`` indicates an error in the construction
of the value, and the program failed to check for this error. The program will
misinterpret the ``NULL`` to mean something different than error
[`Issue 47 <https://github.com/capi-workgroup/problems/issues/47>`__].


API Tiers and Stability Guarantees
==================================

The different API tiers provide different tradeoffs of stability vs
API evolution, and sometimes performance.

The stable ABI was identified as an area that needs to be looked into. At
the moment it is incomplete and not widely adopted. At the same time, its
existence is making it hard to make changes to some implementation
details, because it exposes struct fields such as ``ob_refcnt``,
``ob_type`` and ``ob_size``. There was some discussion about whether
the stable ABI is worth keeping. Arguments on both sides can be
found in [`Issue 4 <https://github.com/capi-workgroup/problems/issues/4>`__]
and [`Issue 9 <https://github.com/capi-workgroup/problems/issues/9>`__].

Alternatively, it was suggested that in order to be able to evolve
the stable ABI, we need a mechanism to support multiple versions of
it in the same Python binary. It was pointed out that versioning
individual functions within a single ABI version is not enough
because it may be necessary to evolve, together, a group of functions
that interoperate with each other
[`Issue 39 <https://github.com/capi-workgroup/problems/issues/39>`__].

The limited API was introduced in 3.2 as a blessed subset of the C API
which is recommended for users who would like to restrict themselves
to high quality APIs which are not likely to change often. The
``Py_LIMITED_API`` flag allows users to restrict their program to older
versions of the limited API, but we now need the opposite option, to
exclude older versions. This would make it possible to evolve the
limited API by replacing flawed elements in it
[`Issue 54 <https://github.com/capi-workgroup/problems/issues/54>`__].
More generally, in a redesign we should revisit the way that API
tiers are specified and consider designing a method that will unify the
way we currently select between the different tiers
[`Issue 59 <https://github.com/capi-workgroup/problems/issues/59>`__].

API elements whose names begin with an underscore are considered
private, essentially an API tier with no stability guarantees.
However, this was only clarified recently, in
`PEP 689 <https://peps.python.org/pep-0689/>`__. It is not clear
what the change policy should be with respect to such API elements
that predate PEP 689
[`Issue 58 <https://github.com/capi-workgroup/problems/issues/58>`__].

There are API functions which have an unsafe (but fast) version as well as
a safe version which performs error checking (for example,
``PyTuple_GET_ITEM`` vs ``PyTuple_GetItem``). It may help to
be able to group them into their own tiers - the "unsafe API" tier and
the "safe API" tier
[`Issue 61 <https://github.com/capi-workgroup/problems/issues/61>`__].

The C Language
==============

A number of issues were raised with respect to the way that CPython
uses the C language. First there is the issue of which C dialect
we use, and how we test our compatibility with it
[`Issue 42 <https://github.com/capi-workgroup/problems/issues/42>`__].

Usage of ``const`` in the API is currently sparse, but it is not
clear whether this is something that we should consider changing
[`Issue 38 <https://github.com/capi-workgroup/problems/issues/38>`__].

We currently use the C types ``long`` and ``int``, where ``stdint``
and ``int32_t`` would have been better choices
[`Issue 27 <https://github.com/capi-workgroup/problems/issues/27>`__].

We are using C language features which are hard for other languages
to interact with
[`Issue 35 <https://github.com/capi-workgroup/problems/issues/35>`__].

There are API functions that take a ``PyObject*`` arg which must be
of a more specific type (such as ``PyTuple_Size``, which fails if
its arg is not a ``PyTupleObject*``). It is an open question whether this
is a good pattern to have, or whether the API should expect the
more specific type
[`Issue 31 <https://github.com/capi-workgroup/problems/issues/31>`__].

There are functions in the API that take concrete types, such as
``PyDict_GetItemString`` which performs a dictionary lookup for a key
specified as a c string rather than ``PyObject*``. At the same time,
for ``PyDict_ContainsString`` it is not considered appropriate to
add a concrete type alternative. The principle around this should
be documented in the guidelines
[`Issue 23 <https://github.com/capi-workgroup/problems/issues/23>`__].

Implementation Flaws
====================

Below is a list of localized implementation flaws. Most of these can
probably be fixed incrementally, if we choose to do so. They should,
in any case, be avoided in any new API design.

There are functions that don't follow the convention of
returning ``0`` for success and ``-1`` for failure. For
example, ``PyArg_ParseTuple`` returns 0 for success and
non-zero for failure
[`Issue 25 <https://github.com/capi-workgroup/problems/issues/25>`__].

The macros ``Py_CLEAR`` and ``Py_SETREF`` access their arg more than
once, so if the arg is an expression with side effects, they are
duplicated
[`Issue 3 <https://github.com/capi-workgroup/problems/issues/3>`__].

The meaning of ``Py_SIZE`` depends on the type and is not always
reliable
[`Issue 10 <https://github.com/capi-workgroup/problems/issues/10>`__].

**Naming**

``PyLong`` and ``PyUnicode`` use names which don't match the python
types they represent (int/str). This can be fixed in a new API
[`Issue 14 <https://github.com/capi-workgroup/problems/issues/14>`__].

There are identifiers in the API which are lacking a ``Py``/``_Py``
prefix
[`Issue 46 <https://github.com/capi-workgroup/problems/issues/46>`__].

Some API function do not have the same behaviour as their Python
equivalents.  The behaviour of ``PyIter_Next`` is different from
``tp_iternext``.
[`Issue 29 <https://github.com/capi-workgroup/problems/issues/29>`__].
The behaviour of ``PySet_Contains`` is different from ``set.__contains__``
[`Issue 6 <https://github.com/capi-workgroup/problems/issues/6>`__].

The fact that ``PyArg_ParseTupleAndKeywords`` takes a non-const
char* array as argument makes it more difficult to use.
[`Issue 28 <https://github.com/capi-workgroup/problems/issues/28>`__].

Python.h does not expose the whole API. Some headers (like marshal.h)
are not included from Python.h.
[`Issue 43 <https://github.com/capi-workgroup/problems/issues/43>`__].


Missing Functionality
=====================

This section consists of a list of feature requests, i.e., functionality
that was identified as missing in the current C API.

**Debug Mode**

A debug mode that can be activated without recompilation and which
activates various checks that can help detect various types of errors.
[`Issue 36 <https://github.com/capi-workgroup/problems/issues/36>`__].

**Introspection**

There aren't currently reliable introspection capabilities for objects
defined in C in the same way as there are for Python objects.
[`Issue 32 <https://github.com/capi-workgroup/problems/issues/32>`__].

Efficient type checking for heap types, similar to what ``Py*_Check``
can do for a static type.
[`Issue 17 <https://github.com/capi-workgroup/problems/issues/17>`__].

**Improved Interaction with Other Languages**

Interfacing with other GC based languages, and integrating their
GC with Python's GC.
[`Issue 19 <https://github.com/capi-workgroup/problems/issues/19>`__].

Inject foreign stack frames to the traceback.
[`Issue 18 <https://github.com/capi-workgroup/problems/issues/18>`__].

Concrete strings that can be used in other languages
[`Issue 16 <https://github.com/capi-workgroup/problems/issues/16>`__].

