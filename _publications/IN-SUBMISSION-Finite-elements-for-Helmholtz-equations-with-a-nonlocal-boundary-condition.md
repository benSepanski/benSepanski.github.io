---
title: "Finite Elements for Helmholtz equations with a nonlocal boundary condition"
collection: publications
permalink: /publication/IN-SUBMISSION-Finite-Elements-for-Helmholtz-equations-with-a-nonlocal-boundary-condition
excerpt: 'A new nonlocal boundary condition for exterior Helmholtz problems along with the software infrastructure to express these boundary conditions in Unified Form Language'
date: 2021-01-01
venue: 'IN SUBMISSION'
citation: 'Kirby, Robert C. and Klo\"eckner, Andreas and Sepanski, Benjamin. &quot;Finite elements for Helmholtz equations with a nonlocal boundary condition.&quot; <i>In Submission</i>.'
---

In submission. Paper is currently hosted on arxiv [here](https://arxiv.org/abs/2009.08493). A more recent edition is available [here](../files/IN-SUBMISSION-Finite-Elements-for-Helmholtz-equations-with-a-nonlocal-boundary-condition).

# Abstract

Numerical resolution of exterior Helmholtz problems requires some approach to do- main truncation. As an alternative to approximate nonreflecting boundary conditions and invocation of the Dirichlet-to-Neumann map, we introduce a new, nonlocal boundary condition. This condition is exact and requires the evaluation of layer potentials involving the free space Green’s function. How- ever, it seems to work in general unstructured geometry, and Galerkin finite element discretization leads to convergence under the usual mesh constraints imposed by G˚arding-type inequalities. The nonlocal boundary conditions are readily approximated by fast multipole methods, and the resulting linear system can be preconditioned by the purely local operator involving transmission boundary conditions

# Presentations

I will present ideas from this paper in an [upcoming talk](../talks/2021-03-25-Nonlocal-UFL-Finite-elements-for-Helmholtz-equations-with-a-nonlocal-boundary-condition) at [FEniCS 2021](https://fenics2021.com/).

# Bibtex

```
@misc{kirby2020finite,
      title={Finite elements for Helmholtz equations with a nonlocal boundary condition}, 
      author={Robert C. Kirby and Andreas Klöckner and Ben Sepanski},
      year={2020},
      eprint={2009.08493},
      archivePrefix={arXiv},
      primaryClass={math.NA}
}
```
