---
title: "Nonlocal UFL: Finite elements for Helmholtz equations with a nonlocal boundary condition"
collection: talks
type: "Talk"
permalink: /talks/2021-03-25-Nonlocal-UFL-Finite-elements-for-Helmholtz-equations-with-a-nonlocal-boundary-condition
venue: "FEniCS 2021"
date: 2021-03-25
location: "Virtual Presentation through the University of Cambridge in Cambridge, England"
---

Presented at the [FEniCS 2021 Conference](https://mscroggs.github.io/fenics2021) ([slides, abstract, and recording](https://mscroggs.github.io/fenics2021/talks/sepanski.html)), joint work with [Dr. Robert Kirby](https://sites.baylor.edu/robert_kirby/) and [Dr. Andreas Klöckner](https://mathema.tician.de/aboutme/). This work introduces nonlocal boundary conditions for exterior Helmholtz problems that are exact (rather than approximate), relying on layer potentials evaluated via fast multipole methods. Integration of the layer potential library [pytential](https://documen.tician.de/pytential) with [Firedrake](https://www.firedrakeproject.org) allows these boundary conditions to be expressed naturally in UFL.

# Abstract

Numerical resolution of exterior Helmholtz problems require some approach to domain truncation. As an alternative to approximate nonreflecting boundary conditions and invocation of the Dirichlet-to-Neumann map, we introduce new, nonlocal boundary conditions. These conditions are exact and require the evaluation of layer potentials involving Green's functions. The nonlocal boundary conditions are readily approximated by fast multipole methods, and the resulting linear system can be preconditioned by the purely local operator. Integration of the layer potential evaluation library pytential with the new external operator feature of Firedrake allows us to express these boundary conditions in UFL.

# Papers

Many of the ideas in this talk come from a paper in submission: [Kirby, Robert C. and Kl&ouml;ckner, Andreas and Sepanski, Benjamin.(2021). &quot;Finite Elements for Helmholtz Equations with a Nonlocal Boundary Condition.&quot; <i>SIAM Journal on Scientific Computing, 43(3), A1671-A1691</i>](../publication/2021-05-10-Finite-elements-for-Helmholtz-equations-with-a-nonlocal-boundary-condition).
