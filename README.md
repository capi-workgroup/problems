# C API Workgroup

The workgroup idea was initiated following the [recent Language Summit discussions about the C API](https://pyfound.blogspot.com/2023/05/the-python-language-summit-2023-three.html).

At this point, we don't have a formal setup for the workgroup yet, but in order to start, we are collecting input from the community to hash out where we'd like to go with the Python C API.

The formal setup of the workgroup will then be a second step.

## Goals

The overarching goal of the workgroup is to create and maintain a design for a consistent, complete and useful Python C API.

This goal specification is intentionally vague at this point. Once we have collected enough input from the core developers and the community, we will create more specific goals.

## Members

Since the workgroup has not yet been formally created, there are no formal members.

# Problems with the Python C API

The purpose of this repo is to collect problems with the current C API, and to determine which of them can be fixed incrementally or require a higher level design change or specification to come up with a solution.

## What is considered a problem ?

Since we do not yet have more specific goals we're following, please submit issues for anything you find problematic with the current Python C API.

## Submitting a problem

Please [create an issue](issues) in the repo for each problem you identify, and that particular issue can be discussed there.

We should focus, for now, on enumerating the problems rather than discussing API redesign proposals. We will be able to discuss new API designs only when we have a comprehensive view of the problems we are trying to solve.

However, we do not need to wait for that before we start fixing problems that can be fixed incrementally in the current C API. Issues can be marked with the "fixable" label, and within those issues it is fine to discuss pointed solutions to the specific problem covered by this issue.

# Summaries of the submitted problems

A summary of the current problem landscape is maintained at irregular intervals in the file [capi_problems.rst](capi_problems.rst).

The document tries to provide a holistic overview of the perceived problems with the Python C API, without getting into details on how these could be solved or where the eventual direction of the developments will go.

# Next steps

This is a rough roadmap of how things could be progressing. This is not necessarily complete and at this point just a sketch, so please don't read too much into it.

- [ ] Announce this effort more broadly to get more community input, especially from Python C extension authors.
- [ ] Formally create the workgroup, once the problem space has been narrowed down.
- [ ] Have the workgroup come up with a proposal to define goals and non-goals of the future Python C API.
- [ ] Submit this proposal as a PEP for discussion and later approval by the Steering Council.
- [ ] Have the workgroup create PEPs which specify the different design areas in more detail, so that they can provide guidance on how to improve the Python C API, make changes and which aspects to consider when making additions to the C API.

# FAQ

TBD
