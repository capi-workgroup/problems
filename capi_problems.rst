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
constraints, and the inherent complexity of the C API, which is both technical
and social. Different types of users bring different, sometimes conflicting,
requirements. The tradeoff between stability and progress is an ongoing, highly
contentious, topic of discussion. Several proposals have been put forwards for
improvements, redesign or replacement of the C API, representing deep
analyses of the problems. At the 2023 Language Summit, three back-to-back
sessions were devoted to various aspects of the C API. The conclusion of the
discussion there was that we don't have a common understanding of the problems
we are trying to solve. It was decided that we need to define and agree on the
problems with the C API, before we are able to evaluate any of the proposed
solutions. This document aims to do just that, by summarizing the contributions
that various people submitted to the capi-workgroup repository on GitHub.

At least 60 different issues were created on that repo, each describing a
problem with the C API. They are categorized and a number of recurring themes
were identified. The sections below correspond to these themes, and each
contains a combined description of the issues raised in that category.


API Evolution and Maintenance
=============================

The difficulty of making changes in the C API is central to this report. It is
implicit in many of the issues we discuss here, particularly when we need to
decide whether an incremental bugfix can resolve the issue, or whether it can
only be resolved as part of an API redesign. This problem was raised in
`Issue 44 <https://github.com/capi-workgroup/problems/issues/44>`__. The
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
of backwards compatibility requirements. Between these two extremes are
solutions that allow incremental resolution of minor and localized problems,
but reserves large changes to larger API redesigns.

The problem we have in CPython is that we don't have an agreed, official
approach of the project. Different members of the core team are pulling in
different direction and this is an ongoing source of disagreements and tension
in the team. A new C API needs to come with a clear decision about the model
that its maintenance will follow, as well as the technical and organizational
processes by which this will work.

