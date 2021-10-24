---
title: 'Denali: A Goal-directed Superoptimizer'
date: 2021-03-10
permalink: /posts/2021/03/10/Denali-a-goal-directed-superoptimizer/
tags:
  - superoptimization
  - E-graphs
  - Denali
---
This blog post was written for [Dr. James Bornholt](https://www.cs.utexas.edu/~bornholt/)'s [CS 395T: Systems Verification and Synthesis, Spring 2021](https://www.cs.utexas.edu/~bornholt/courses/cs395t-21sp/). It summarizes the context and contributions of the paper [Denali: A Goal-directed Superoptimizer](https://dl.acm.org/doi/10.1145/543552.512566), written by [Dr. Rajeev Joshi](https://rjoshi.org/bio/index.html), [Dr. Greg Nelson](https://en.wikipedia.org/wiki/Greg_Nelson_(computer_scientist)), and [Dr. Keith Randall](http://people.csail.mit.edu/randall/).

None of the technical ideas discussed in this blog are my own, they are summaries/explanations based on the referenced works.


# Denali: A Goal-directed Superoptimizer

Tdoday's paper is [Denali: A Goal-directed Superoptimizer](https://dl.acm.org/doi/10.1145/543552.512566). At the time of its publication (2002), it was one of the first **superoptimizer**s: a code generator which seeks to find truly optimal code. This is a dramatically different approach from traditional compiler optimizations, and is usually specific to efficiency-critical straight-line kernels written at the assembly level.

## Background

### What *is* Super&#129464;optimization?

Plenty of compilers are *optimizing compilers*. However, in the strictest sense of the word, they don't really find an *optimal* translation. They just find one that, according to some heuristics, ought to improve upon a naive translation. Why? Finding optimal translations is, in general, undecidable. Even for simplified, decidable versions of the problem, it is prohibitively time consuming to insert into any mortal programmer's build-run-debug development cycle.

However, sometimes it is worth the effort to find a *truly optimal* solution. To disambiguate between these two "optimization" procedures, we use the term *superoptimization* when we are seeking a "truly optimal" solution. Superoptimization is an offline procedure and typically targets straight-line sequences of machine code inside critical loops.

With a few simplifying assumptions, the shortest straight-line code is the fastest. Consequently, we seek the shortest program.

### Super&#129464;optimization: The Pre-Denali Era (Beginning of Time -- 2002)

[Alexia Massalin](https://en.wikipedia.org/wiki/Alexia_Massalin) coined the term "[superoptimization](https://en.wikipedia.org/wiki/Superoptimization)" in her 1987 paper [Superoptimizer -- A look at the Smallest Program](https://dl.acm.org/doi/10.1145/36177.36194). Massalin used a (pruned) exhaustive search to find the shortest implementation of various straight line computations in the [68000 instruction set](https://en.wikipedia.org/wiki/Motorola_68000#Instruction_set_details). For instance, she found the shortest programs to compute the signum function, absolute value, max/min, and others. Her goal was to identify unintuitive idioms in these shortest programs so that performance engineers could use them in practice.

While Massalin's technique was powerful, it did not scale well (the shortest programs were at most 13 instructions long in Massalin's paper). Moreover, the output programs were not automatically verified to be equivalent to the input programs. They are instead highly likely to be equivalent, and must be verified by hand.

Granlund \& Kenner [followed up on Massalin's work](https://dl.acm.org/doi/10.1145/143103.143146) in 1992 with the [GNU Superoptimizer](https://www.gnu.org/software/superopt/). They integrated a variation of Massalin's superoptimizer into GCC to eliminate branching. 

Until 2002, research in superoptimizers seemed to stall. Judging by citations during that period, most researchers considered Massalin's work to fit inside the field of optimizing compilers. These researchers viewed superoptimization as a useful engineering tool, but of little theoretical interest or scalability. Rather, superoptimization was seen as an interesting application of brute-force search. Massalin and the GNU Superoptimizer seemed to become a token citation in the optimizing compiler literature.

## The Denali Approach

### Goal-Directed vs. Brute Force

Massalin's superoptimizer relies on brute-force search: enumerate candidate programs until you find the desired program. Given the massive size of any modern instruction set, this does not scale well. However, since we want the shortest program, we have to rely on some kind of brute-force search. Denali's insight is that Massalin's search algorithm was enumerating *all* candidate programs, instead of only enumerating *relevant* candidate programs.

Denali users specify their desired program as a set of (memory location, expression to evaluate) pairs.  For instance, (*%rdi*, *2 * %rdi*) is the program which doubles the value of *%rdi*.

Denali's algorithm only "enumerates" candidate programs which it can prove are equivalent to the desired program. For efficiency, it stores this enumeration in a compact graph structure called an [E-Graph](#e-graphs), then searches the E-Graph using a SAT solver.


### E-Graphs

#### What is an E-Graph?

An E-Graph is used to represent expressions. For instance, a literal 4 or a register value *%rdi* is represented as a node with no children.

![](/files/posts/2021-03-10-Denali-a-goal-directed-superoptimizer/egraph-rdi-and-4.svg?token=AIEHSZA5TESJ3BC6LBD6FELAGFZ3O)

The expression *%rdi \* 4* is represented as a node '*\**' whose first child represents *%rdi* and whose second child represents 4.

![](/files/posts/2021-03-10-Denali-a-goal-directed-superoptimizer/egraph-rdi-times-4.svg?token=AIEHSZACZTJLTY7GVZSQQUDAGF2EM)

Bigger expressions are represented just like you would think. For instance, the expression *%rdi * 4 + 1* would be represented as

![](/files/posts/2021-03-10-Denali-a-goal-directed-superoptimizer/egraph-rdi-times-4-plus-1.svg?token=AIEHSZH4LPUKAMS5CIODF5TAGF3AO)

So far, this just looks like an [Abstract Syntax Tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree). E-Graphs are distinguished from ASTs by the ability to represent **multiple equivalent expressions**. Suppose we wish to add the equivalence 4<span style="color:blue">=</span>2\*\*2 to our E-graph. We do this by adding a special *<span style="color:blue">equivalence edge</span>*

![](/files/posts/2021-03-10-Denali-a-goal-directed-superoptimizer/egraph-rdi-times-4-plus-1-with-exp.svg?token=AIEHSZB7XJBWPFALUQVHFJDAGF4SC)

Since there is no machine exponentiation instruction, this does not look useful at first. However, now we can add a further <span style="color:blue">equivalence edge</span> based on the fact that *%rdi << 2 <span style="color:blue">=</span> %rdi \* 2**2 <span style="color:blue">=</span> %rdi \* * 4*.

![](/files/posts/2021-03-10-Denali-a-goal-directed-superoptimizer/egraph-rdi-times-4-plus1-with-shift.svg?token=AIEHSZBXN3ND7R2LHKI3VYTAGF6EU)

Since E-Graphs represent **A=B** by keeping both **A** and **B** around, they can become quite massive.

#### How do we build E-Graphs?

We can use proof rules to repeatedly grow the E-Graph and/or add <span style="color:blue">equivalence edge</span>s. If we keep applying our proof rules until our graph stops changing, then we've deduced all the ways we can provably compute our expression (relative to our proof rules). For instance, in the previous example we had only three proof rules:

1. 4 = 2**2
2. x * 2**n = x << n
3. If a = b, and b = c, then a = c

If we add more proof rules, we may be able to deduce faster ways to compute our expression.


#### Other uses of E-Graphs

An early variant of E-Graphs is described in Greg Nelson's (one of the Denali authors) [Ph.D. Thesis](http://scottmcpeak.com/nelson-verification.pdf). These were used by Nelson in the automated theorem prover [Simplify](https://www.hpl.hp.com/techreports/2003/HPL-2003-148.pdf) for equational reasoning. Since then, search over E-graphs via the [congruence closure](https://people.eecs.berkeley.edu/~necula/Papers/NelsonOppenCong.pdf) algorithm is used by many modern SMT solveres for reasoning about equality of uninterpreted functions (it is even taught in [Dr. Isil Dillig](https://www.cs.utexas.edu/~isil/)'s [CS 389L course](https://www.cs.utexas.edu/~isil/cs389L/lecture11-6up.pdf) here at UT!). For example, the [Z3 SMT solver](https://github.com/Z3Prover/z3) [implements an E-graph](https://github.com/Z3Prover/z3/blob/830f314a3f32ce896fcf93fd40666d4a390fc330/src/ast/euf/euf_enode.h#L30), and the [CVC4 Solver](https://github.com/CVC4/CVC4) [implements an incremental congruence closure](https://github.com/CVC4/CVC4/blob/e4fd524b02054a3ac9724f184e55a983cb6cb6b9/src/theory/uf/equality_engine.h#L50).

### Search over E-Graphs

Nodes in an E-Graph that are connected by an <span style="color:blue">equivalence edge</span> represent expressions that are equivalent according to the proof rules. Therefore, we only need to evaluate one of the nodes. Denali can use a SAT solver to figure out the optimal choice of nodes. Their encoding is not too complicated. 

The basic idea of the encoding is as follows:
```
For each machine instruction node T,
    L(i, T) = { 1    T starts executing on cycle i 
              { 0    otherwise
```
Then, all we have to do is add constraints so that
* Exactly one instruction starts executing per cycle.
* Each instruction's arguments are available when the instruction gets executed.
* Some node equivalent to the root node gets computed.

Now we can find the shortest program encoded in our E-Graph by constraining the SAT solver to look for a program of length 1, then length 2, then length 3, .... until we find a solution.

## Impact of the Denali Superoptimizer

### Preliminary Results

The Denali paper presents several preliminary results. For the [Alpha instruction set architecture](https://en.wikipedia.org/wiki/DEC_Alpha#:~:text=Alpha%2C%20originally%20known%20as%20Alpha,set%20computer%20(CISC)%20ISA.), they are able to generate some programs of length up to 31 instructions. For comparison, the GNU superoptimizer is unable to generate (near) optimal instructions sequences of length greater than 5.

However, in addition to Denali's built-in architectural axioms, the programmers specify program-specific axioms in their examples. This trades off automation for the ability to generate longer (near) optimal instruction sequences.

### Super&#129464;optimization: The Post-Denali Era (2002 -- Present Day)

Denali demonstrated that, for small programs, it is possible to generate provably equivalent, (nearly) optimal code. Since then, there has been a lot of interest in superoptimization. Here are some projects/papers that have popped up since Denali.

* [Souper](https://arxiv.org/pdf/1711.04422.pdf) is an [open-source](https://github.com/google/souper) project that extracts straight-line code from the LLVM IR and applies superoptimization. It uses caching so that it can be run online (2017).
    - SMT-based goal-directed search
    - Maintained by [Google](https://www.google.com), jointly developed by researchers at Google, [NVIDIA](https://www.nvidia.com/en-us/), [TNO](https://www.tno.nl/en/), [Microsoft](https://www.microsoft.com/en-us/), [SES](https://www.ses.com/), and the University of Utah.
* [slumps](https://github.com/KTH/slumps) is based on souper and targets web assembly (2020).
* [STOKE](http://stoke.stanford.edu/) is a superoptimizer for x86 (2013) (which we will be [reading on March 22](https://www.cs.utexas.edu/~bornholt/courses/cs395t-21sp/schedule/) after spring break)
    - Uses stochastic enumerative search.
    - Is still maintained and open source at [StanfordPL/stoke](https://github.com/StanfordPL/stoke).
* [embecosm](https://www.embecosm.com/about/), a compiler research group, is developing [GSO 2.0](https://www.embecosm.com/services/superoptimization/#Related-Services) (2015)
* There has been research into [automatic peephole optimizations](https://theory.stanford.edu/~aiken/publications/papers/asplos06.pdf) (2006).

However, while there is active industry and research interest in the *problem* that Denali presented (finding a provably equivalent, near-optimal translation), most modern approaches (e.g. souper) rely on SMT-based synthesis techniques. Denali's methods of superoptimization seem to have largely fallen by the wayside. Part of this is because Denali's provably (almost) optimal program relies on a set of user-specified axioms, and is only optimal with respect to those axioms. Part of the appeal of an SMT solver is standardized theories for certain objects and operations.

Both enumerative search (e.g. STOKE) and goal-directed search (e.g. souper) are used today. In addition, Denali's general notion of specification (a set of symbolic input-output pairs) is still used, with various project-specific modifications. Projects still rely on (often hand-written) heuristics to measure the cost/cycle-count of candidate programs.

# Discussion Questions

* As computation times are increasingly memory-bound, does superoptimization run into concerns with Amdahl's law?
* SMT solvers are powerful tools, but incredibly general-purpose. What types of computations are likely to be compute-bound, and can we use that domain-specific knowledge to make superoptimization faster?
* Superoptimization seems naturally related to [automated vectorization](https://en.wikipedia.org/wiki/Automatic_vectorization). However, people seem to treat the two problems as separate. Is there any reason automated vectorization might make superoptimization much more difficult?
* [BLIS](https://github.com/flame/blis) is a framework for creating [BLAS](http://www.netlib.org/blas/) implementations for new architectures by only implementing small, finite computations kernels. Can BLIS be combined with superoptimization to automatically generate BLAS libraries for specific architectures?

# References

All graphs were built using [graphviz](https://graphviz.org/). The example E-Graph in section [What is an E-Graph?](#What-is-an-E-Graph) is based on the example in today's paper.
