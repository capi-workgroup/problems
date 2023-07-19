Title: An Evaluation of Python's C API


Abstract
========

This document describes our shared view of the C API. We aim to cover its
purposes, the different stakeholders and their particular use cases and
requirements, and to identify the strengths and weaknesses of the C API.
This document does not propose solutions to any of the problems it identifies.
Instead, it is intended to be used to guide the design of such solutions,
and to provide criteria by which to evaluate solutions that are proposed.

Introduction
============

The original purpose of Python's C API was to embed Python into C/C++
applications and to make it possible to write extension modules in C/C++. These
capabilities were instrumental in the growth of Python's ecosystem.  Over the
decades, the C API evolved to provide different tiers of stability, conventions
changed, and new usage patterns have emerged. In addition, lessons were learned
and mistakes in both the design and the implementation of the C API were
identified.

Evolving the C API is hard due to a combination of backwards compatibility
constraints and the inherent complexity of the C API, which is both technical
and social. Different types of users bring different, sometimes conflicting,
requirements. The tradeoff between stability and progress is an ongoing, highly
contentious topic of discussion. Several proposals have been put forwards for
improvements, redesign or replacement of the C API, representing deep
analyses of the problems. At the 2023 Language Summit, three back-to-back
sessions were devoted to various aspects of the C API. The conclusion of the
discussions there was that we don't have a common understanding of the problems
we are trying to solve. It was decided that we need to agree on the current
problems with the C API, before we are able to evaluate any of the proposed
solutions. This document aims to do just that, by summarizing the contributions
that various people submitted to the
[`capi-workgroup <https://github.com/capi-workgroup/problems/issues/>`__]
repository on GitHub in the aftermath of the language summit.

Over 60 different issues were created on that repo, each describing a
problem with the C API. They were categorized and a number of recurring
themes were identified. The sections below correspond to these themes,
and each contains a combined description of the issues raised in that
category, along with links to the individual issues.


API Specification and Abstraction
=================================

The C API does not have a formal specification, it is described
semi-formally in the documentation and exposed through C header
files. This creates a number of problems.

Bindings for languages other than C/C++ are required to parse C code
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

Another set of problems arises from the fact that a PyObject* is
exposed in the C API as an actual pointer rather than a handle. The
address of an object serves as its ID and is used for comparison,
and this complicates matters for alternative Python implementations
that move objects during GC 
[`Issue 37 <https://github.com/capi-workgroup/problems/issues/37>`__].


Object References
=================

The API's 


Error Handling
==============

Error handling in the C API is based on the error indicator which is stored
on the thread state (in global scope). The design intention was that each
API function returns a value indicating whether an error has occurred (by
convention, ``-1`` or ``NULL``). When the program know that an error occurred,
it can fetch the exception object which is stored in the error indicator.
A number of problems were identified which are related to error handling,
pointing at APIs which are too easy to use incorrectly.

Functions that Suppress Errors
------------------------------
There are functions that do not report all errors that occur while they
execute. For example, ``PyDict_GetItem`` clears any errors that occur
when it calls the key's hash function, or while performing a lookup
in the dictionary.
[`Issue 51 <https://github.com/capi-workgroup/problems/issues/51>`__].

Functions Called with Error Indicator Set
-----------------------------------------
Python code never executes with an in-flight exception (by definition),
and by the same token C API functions should never be called with the error
indicator set. This is currently not checked in most C API functions, and
there are places in the interpreter where error handling code calls a C API
function while an exception is set. For example, see the call to
``PyUnicode_FromString`` in the error handler of ``_PyErr_WriteUnraisableMsg``
[`Issue 2 <https://github.com/capi-workgroup/problems/issues/2>`__].

Missing or Ambiguous Return Values
----------------------------------
There are functions that do not return a value, so a caller is forced to
query the error indicator in order to identify whether an error has occurred.
An example is ``PyBuffer_Release``
[`Issue 20 <https://github.com/capi-workgroup/problems/issues/20>`__].

There are other functions which do have a return value, but this return value
does not unambiguously indicate whether an error has occurred. For example,
``PyLong_AsLong`` returns ``-1`` in case of error, or when the value of the
argument is indeed ``-1``
[`Issue 1 <https://github.com/capi-workgroup/problems/issues/1>`__].

This is error prone because it is possible that the error indicator was already
set before the function was called, and the error is incorrectly attributed.
The fact that the error was not detected before the call is a bug in the
calling code, but the behaviour of the program in this case doesn't make it
easy to identify and debug the problem.

``NULL`` as a Valid ``PyObject*`` Argument Value
------------------------------------------------
There are functions that take a ``PyObject*`` argument, with special meaning
when it is ``NULL``. For example, if ``PyObject_SetAttr`` receives ``NULL`` as
the value to set, this mean that the attribute should be cleared. This is error
prone because it could be that ``NULL`` indicates an error in the construction
of the value, and the program failed to check for this error. The program will
misinterpret the ``NULL`` to mean something different than error
[`Issue 47 <https://github.com/capi-workgroup/problems/issues/47>`__].


API Evolution and Maintenance
=============================

The difficulty of making changes in the C API is central to this report. It is
implicit in many of the issues we discuss here, particularly when we need to
decide whether an incremental bugfix can resolve the issue, or whether it can
only be resolved as part of an API redesign
[`Issue 44 <https://github.com/capi-workgroup/problems/issues/44>`__]. The
benefit of each incremental change is often viewed as too small to justify the
disruption. Over time, this implies that every mistake we make in an API's
design or implementation remains with us indefinitely.

We can take two views on this issue. One is that this is a problem and the
solution needs to be baked into any new C API we design, in the form of a
process for incremental API evolution. The other possible approach is that
this is not a problem to be solved, but rather a feature of any API. In this
view, API evolution should not be incremental, but rather through large
redesigns, each of which learns from the mistakes of the past. The new API can
be designed to the best of our understanding at the time, without the shackles
of backwards compatibility requirements. A realistic approach will be somewhere
between these two extremes, fixing issues which are easy or important enough
to tackle incrementally, and leaving others alone.

The problem we have in CPython is that we don't have an agreed, official
approach to API evolution. Different members of the core team are pulling in
different directions and this is an ongoing source of disagreements and
tension. A new C API needs to come with a clear decision about the model
that its maintenance will follow, as well as the technical and organizational
processes by which this will work.
