---
title: "Synthesizing fine-grained synchronization protocols for implicit monitors"
collection: publications
permalink: /publication/2022-12-01-Synthesizing-fine-grained-synchronization-protocols-for-implicit-monitors
excerpt: 'Automatatically implementing fine-grained explicit synchronization protocols from an implicit monitor specification using formal methods'
date: 2022-12-01
venue: 'Proc. ACM Program. Lang. 6, OOPSLA1'
citation: 'Kostas Ferles, Benjamin Sepanski, Rahul Krishnan, James Bornholt, and Işil Dillig. 2022. &quot;Synthesizing fine-grained synchronization protocols for implicit monitors.&quot; <i>Proc. ACM Program. Lang. 6, OOPSLA1, Article 67 (December 2022), 26 pages</i>. https://doi.org/10.1145/3527311'
---


View the paper on the ACM Digital Library: [Synthesizing fine-grained synchronization protocols for implicit monitors](https://dl.acm.org/doi/10.1145/3527311).

# Abstract

A monitor is a widely-used concurrent programming abstraction that encapsulates all shared state between threads. Monitors can be classified as being either implicit or explicit depending on the primitives they provide. Implicit monitors are much easier to program but typically not as efficient. To address this gap, there has been recent research on automatically synthesizing explicit-signal monitors from an implicit specification, but prior work does not exploit all paralellization opportunities due to the use of a single lock for the entire monitor. This paper presents a new technique for synthesizing fine-grained explicit-synchronization protocols from implicit monitors. Our method is based on two key innovations: First, we present a new static analysis for inferring safe interleavings that allow violating mutual exclusion of monitor operations without changing its semantics. Second, we use the results of this static analysis to generate a MaxSAT instance whose models correspond to correct-by-construction synchronization protocols. We have implemented our approach in a tool called Cortado and evaluate it on monitors that contain parallelization opportunities. Our evaluation shows that Cortado can synthesize synchronization policies that are competitive with, or even better than, expert-written ones on these benchmarks.

# Presentations

*Upcoming:* Presentation at OOPSLA 22, visit the conference website for more info: https://2022.splashcon.org/track/splash-2022-oopsla

# Bibtex

```
@article{10.1145/3527311,
author = {Ferles, Kostas and Sepanski, Benjamin and Krishnan, Rahul and Bornholt, James and Dillig, I\c{s}il},
title = {Synthesizing Fine-Grained Synchronization Protocols for Implicit Monitors},
year = {2022},
issue_date = {December 2022},
publisher = {Association for Computing Machinery},
address = {New York, NY, USA},
volume = {6},
number = {OOPSLA1},
url = {https://doi.org/10.1145/3527311},
doi = {10.1145/3527311},
abstract = {A monitor is a widely-used concurrent programming abstraction that encapsulates all shared state between threads. Monitors can be classified as being either implicit or explicit depending on the primitives they provide. Implicit monitors are much easier to program but typically not as efficient. To address this gap, there has been recent research on automatically synthesizing explicit-signal monitors from an implicit specification, but prior work does not exploit all paralellization opportunities due to the use of a single lock for the entire monitor. This paper presents a new technique for synthesizing fine-grained explicit-synchronization protocols from implicit monitors. Our method is based on two key innovations: First, we present a new static analysis for inferring safe interleavings that allow violating mutual exclusion of monitor operations without changing its semantics. Second, we use the results of this static analysis to generate a MaxSAT instance whose models correspond to correct-by-construction synchronization protocols. We have implemented our approach in a tool called Cortado and evaluate it on monitors that contain parallelization opportunities. Our evaluation shows that Cortado can synthesize synchronization policies that are competitive with, or even better than, expert-written ones on these benchmarks.},
journal = {Proc. ACM Program. Lang.},
month = {apr},
articleno = {67},
numpages = {26},
keywords = {implicit signal monitors, symbolic reasoning, verification conditions, fine-grained locking, monitor invariant, concurrent programming}
}
```

