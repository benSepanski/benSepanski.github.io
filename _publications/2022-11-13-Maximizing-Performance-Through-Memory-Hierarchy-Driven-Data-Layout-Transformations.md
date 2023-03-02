---
title: "Maximizing Performance Through Memory-Hierarchy Driven Data Layout Transformations"
collection: publications
permalink: /publication/2022-11-13-Maximizing-performance-through-memory-hierarchy-driven-data-layout-transformations
excerpt: 'Extend the Bricks framework to optimize high-dimensional code through data layout transformation'
date: 2022-11-13
venue: 'MCHPC 2022: Workshop on Memory Centric High Performance Computing at SC22'
citation: 'B. Sepanski, T. Zhao, H. Johansen and S. Williams, "Maximizing Performance Through Memory Hierarchy-Driven Data Layout Transformations," in <i>2022 IEEE/ACM Workshop on Memory Centric High Performance Computing (MCHPC), Dallas, TX, USA, 2022 pp. 1-10.</i>
doi: 10.1109/MCHPC56545.2022.00006'
---

You can download a copy of the workshop paper [here](https://doi.ieeecomputersociety.org/10.1109/MCHPC56545.2022.00006).

# Abstract

Computations on structured grids using standard multidimensional array layouts can incur substantial data movement costs through the memory hierarchy. This presentation explores the benefits of using a framework (Bricks) to separate the complexity of data layout and optimized communication from the functional representation. To that end, we provide three novel contributions and evaluate them on several kernels taken from GENE, a phase-space fusion tokamak simulation code. We extend Bricks to support 6-dimensional arrays and kernels that operate on complex data types, and integrate Bricks with cuFFT. We demonstrate how to optimize Bricks for data reuse, spatial locality, and GPU hardware utilization achieving up to a 2.67× speedup on a single A100 GPU. We conclude with insights on how to rearchitect memory subsystems.

# Presentations

This work was presented at [MCHPC 2022](https://passlab.github.io/mchpc/mchpc2022/) in Dallas, Texas at the annual Supercomputing conference in November of 2022. See more on the presentation [here](../../talks/2022-11-13-Maximizing-performance-through-memory-hierarchy-driven-data-layout-transformations).


# Bibtex

```
@INPROCEEDINGS {10024060,
author = {B. Sepanski and T. Zhao and H. Johansen and S. Williams},
booktitle = {2022 IEEE/ACM Workshop on Memory Centric High Performance Computing (MCHPC)},
title = {Maximizing Performance Through Memory Hierarchy-Driven Data Layout Transformations},
year = {2022},
volume = {},
issn = {},
pages = {1-10},
abstract = {Computations on structured grids using standard multidimensional array layouts can incur substantial data movement costs through the memory hierarchy. This paper explores the benefits of using a framework (Bricks) to separate the complexity of data layout and optimized communication from the functional representation. To that end, we provide three novel contributions and evaluate them on several kernels taken from GENE, a phase-space fusion tokamak simulation code. We extend Bricks to support 6-dimensional arrays and kernels that operate on complex data types, and integrate Bricks with cuFFT. We demonstrate how to optimize Bricks for data reuse, spatial locality, and GPU hardware utilization achieving up to a 2.67 &amp;#x00D7; speedup on a single A100 GPU. We conclude with insights on how to rearchitect memory subsystems.},
keywords = {costs;high performance computing;layout;graphics processing units;tokamak devices;spatial databases;hardware},
doi = {10.1109/MCHPC56545.2022.00006},
url = {https://doi.ieeecomputersociety.org/10.1109/MCHPC56545.2022.00006},
publisher = {IEEE Computer Society},
address = {Los Alamitos, CA, USA},
month = {nov}
}

```

