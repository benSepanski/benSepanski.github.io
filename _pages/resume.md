---
layout: archive
title: "Resume"
permalink: /resume/
author_profile: true
redirect_from:
  - /cv
  - /vita
---

Click [here](../files/sepanski-benjamin-resume-full.pdf) to download my resume.

Security
========

Before compilers, I spent a few years at [Veridise](https://veridise.com) doing security work on smart contracts and [zero-knowledge](https://en.wikipedia.org/wiki/Zero-knowledge_proof) circuits.

I performed over 35 manual source code security reviews, turning up dozens of high and critical severity bugs across ecosystems including EVM, Mina, Linea, Monero, and NEAR.
The reviews covered a wide range of targets — ERC-4626 vaults, NFT collections, lending protocols, [AMMs](https://chain.link/education-hub/what-is-an-automated-market-maker-amm), orderbooks, zkEVMs, and cryptographic primitives like [FROST](https://eprint.iacr.org/2020/852) distributed signing, ECDSA, Keccak, and recursive ZK-verifiers — across frameworks including [halo2](https://zcash.github.io/halo2/), [gnark](https://github.com/Consensys/gnark), [o1js](https://docs.minaprotocol.com/zkapps/o1js), [circom](https://docs.circom.io), and [arkworks](https://github.com/arkworks-rs).
On the tooling side, I developed detectors for an LLVM-based static analyzer and set up and ran static analyzers and fuzzers for clients.

I also worked on the board/executive team where I worked on company strategy.
During that time I hired and managed the auditing team, growing it from one to seven.

Research
========

Programming Languages
---------------------

I worked to automate the implementation of certain concurrent programs
from a declarative specification
with [Dr. Işil Dillig](https://cs.utexas.edu/~isil), [Dr. James Bornholt](https://www.cs.utexas.edu/~bornholt/),
and [Dr. Kostas Ferles](https://kferles.github.io/).

Check out the paper [here](../publication/2022-12-01-Synthesizing-fine-grained-synchronization-protocols-for-implicit-monitors)!

High Performance Computing
--------------------------

I worked with [Dr. Samuel Williams](https://crd.lbl.gov/divisions/amcr/computer-science-amcr/par/members/staff/samuel-williams/) and [Dr. Hans Johansen](https://crd.lbl.gov/divisions/amcr/computational-science-dept/anag/about/staff-and-postdocs/hans-johansen/) at [Lawrence Berkeley National Labs](https://crd.lbl.gov) on the [Bricks project](https://github.com/CtopCsUtahEdu/bricklib), which optimizes high-performance codes by transforming data layouts rather than code.

Check out the paper [here](../publication/2022-11-13-Maximizing-performance-through-memory-hierarchy-driven-data-layout-transformations).

Scientific Computing
--------------------

At Baylor, I worked with [Dr. Robert Kirby](https://www.baylor.edu/math/index.php?id=90540) and [Dr. Andreas Kloeckner](https://mathema.tician.de/aboutme/) to integrate [pytential](https://documen.tician.de/pytential) into the [Firedrake](https://www.firedrakeproject.org) finite element framework.
This enabled the use of [fast multipole methods](https://www.en.wikipedia.org/wiki/Fast_multipole_method) to approximate far-field boundary conditions — useful for problems like the [Helmholtz equation](https://www.en.wikipedia.org/wiki/Helmholtz_equation) with a [Sommerfeld radiation condition](https://www.en.wikipedia.org/wiki/Sommerfeld_radiation_condition).

Check out the paper [here](../publication/2021-05-10-Finite-elements-for-Helmholtz-equations-with-a-nonlocal-boundary-condition).
