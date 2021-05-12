---
title: "Finite Elements for Helmholtz equations with a nonlocal boundary condition"
collection: publications
permalink: /publication/IN-SUBMISSION-Finite-elements-for-Helmholtz-equations-with-a-nonlocal-boundary-condition
excerpt: 'A new nonlocal boundary condition for exterior Helmholtz problems along with the software infrastructure to express these boundary conditions in Unified Form Language'
date: 2021-05-10
venue: 'SIAM Journal on Scientific Computing'
citation: 'Kirby, Robert C. and Kl&ouml;ckner, Andreas and Sepanski, Benjamin. &quot;Finite elements for Helmholtz equations with a nonlocal boundary condition.&quot; <i>In Submission</i>.'
---

Download paper from the SISC website [here](https://epubs.siam.org/doi/abs/10.1137/20M1368100).

# Abstract

Numerical resolution of exterior Helmholtz problems requires some approach to do- main truncation. As an alternative to approximate nonreflecting boundary conditions and invocation of the Dirichlet-to-Neumann map, we introduce a new, nonlocal boundary condition. This condition is exact and requires the evaluation of layer potentials involving the free space Green’s function. How- ever, it seems to work in general unstructured geometry, and Galerkin finite element discretization leads to convergence under the usual mesh constraints imposed by G˚arding-type inequalities. The nonlocal boundary conditions are readily approximated by fast multipole methods, and the resulting linear system can be preconditioned by the purely local operator involving transmission boundary conditions

# Presentations

I presented ideas from this paper in a [talk](../talks/2021-03-25-Nonlocal-UFL-Finite-elements-for-Helmholtz-equations-with-a-nonlocal-boundary-condition) at [FEniCS 2021](https://fenics2021.com/).

# Bibtex

```
@article{kirbyKlocknerSepanski2021,
author = {Kirby, Robert C. and Klöckner, Andreas and Sepanski, Ben},
title = {Finite Elements for Helmholtz Equations with a Nonlocal Boundary Condition},
journal = {SIAM Journal on Scientific Computing},
volume = {43},
number = {3},
pages = {A1671-A1691},
year = {2021},
doi = {10.1137/20M1368100},

URL = { 
        https://doi.org/10.1137/20M1368100
    
},
eprint = { 
        https://doi.org/10.1137/20M1368100
    
}

}
```
