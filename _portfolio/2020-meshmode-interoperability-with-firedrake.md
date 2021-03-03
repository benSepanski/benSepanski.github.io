---
title: "Meshmode interoperability with firedrake"
excerpt: "Mesh and function conversion between the meshmode and firedrake libraries <br/><img src='../files/portfolio/2020-meshmode-interoperability-with-firedrake/mesh_near_gamma.png'>"
collection: portfolio
---

I wrote the routines to convert meshes and functions between the [Firedrake](https://www.firedrakeproject.org/) and [meshmode](https://documen.tician.de/meshmode/) representations. The code is currently maintained [on github](https://github.com/inducer/meshmode/tree/master/meshmode/interop/firedrake). The original pull request can be found [on gitlab](https://gitlab.tiker.net/inducer/meshmode/-/merge_requests/91). Browse the [documentation](https://documen.tician.de/meshmode/interop.html#firedrake) of these features.

This project allows [boundary element method](https://en.wikipedia.org/wiki/Boundary_element_method)s to be easily expressed and solved using [Firedrake](https://www.firedrakeproject.org/). For examples, see my upcoming publication: [Finite elements for Helmholtz equations with a nonlocal boundary condition](../publication/IN-SUBMISSION-Finite-elements-for-Helmholtz-equations-with-a-nonlocal-boundary-condition).

These conversion routines are the result of a year-long undergraduate project. All code is written in [Python](https://en.wikipedia.org/wiki/Python_(programming_language)). 

# Features:
* Conversion of arbitrary-order simplex meshes from firedrake to meshmode and vice versa.
* Preservation of boundary tags across conversion.
* Exclusive conversion of cells near specified source/target boundaries to enable quick conversion for layer potential evaluation.
* Conversion of arbitrary-order functions in a discontinuous Lagrange space from firedrake to meshmode and vice versa.
* Support for arbitrary DOF-sets used to represent discontinuous Lagrange elements on a simplex (see [meshmode documentation](https://documen.tician.de/meshmode/discretization.html) for more details).
