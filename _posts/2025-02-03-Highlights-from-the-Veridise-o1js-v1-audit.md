---
title: 'Highlights from the Veridise o1js v1 audit: Three zero-knowledge security bugs explained'
date: 2025-02-03
permalink: /posts/2025/02/03/Highlights-from-the-Veridise-o1js-v1-audit/
tags:
  - Mina
  - ZK
  - o1js
  - audit
---

For the original (and properly formatted) publication, see [Veridise's post on medium](https://medium.com/veridise/highlights-from-the-veridise-o1js-v1-audit-three-zero-knowledge-security-bugs-explained-2f5708f13681).

---

In 2024, the Veridise team conducted a comprehensive security audit of [o1js](https://www.o1labs.org/o1js), a crucial TypeScript library that powers zero-knowledge application development on the [Mina blockchain](https://minaprotocol.com/).

The security assessment spanned 39 person-weeks, with four security analysts working over a period of 13 weeks. The audit strategy combined tool-assisted analysis of the source code by Veridise engineers with extensive manual, line-by-line code reviews.

In this blog post, we'll dive into three of the most intriguing vulnerabilities uncovered during our audit. These issues are particularly noteworthy because they span different layers of the cryptographic stack, ranging from low-level field arithmetic to high-level protocol design. What unites them all is their relation to range checks.

To make the findings easier to follow and understand, we've simplified the bugs into illustrative examples. Full reporting on the actual vulnerabilities can be found in the full audit report.

Working together with o1Labs
============================

Our collaboration with o1Labs was both productive and engaging. We had weekly meetings and maintained constant communication via Slack.

Most of our interactions were with Gregor and Florian, who were highly active and deeply involved. They worked closely with us to enhance our understanding of the system and even identified some of the bugs independently, working in parallel with our team.

They frequently shared detailed insights through long Slack threads, and were responsive to any queries from our auditors. Their deep knowledge of the codebase allowed them to efficiently guide us to the areas our auditors needed to focus on.

A highlight of the collaboration was Gregor's incredibly thorough writeups on optimizations. These were invaluable in helping us navigate and comprehend complex circuits, such as emulated field arithmetic and multi-scalar multiplication. His detailed explanations were helpful in our ability to follow and address these intricate components.

Three vulnerabilities Veridise fixed in o1js
============================================

1) Importance of type checks: range validation in zero-knowledge circuits
=========================================================================

Among the high-severity vulnerabilities we discovered in o1js was a subtle but dangerous flaw in how circuits which validate ECDSA signatures or manipulate foreign curve points are verified. This bug (V-O1J-VUL-006) highlights how missing range checks can undermine the security of cryptographic protocols.

As an overview, in o1js, data is often decomposed into smaller components for processing. A common example is **bit decomposition**, where a number is broken down into its binary representation:

For instance:

-   The number `3` can be written as `1 + 1 * 2`, which is encoded as `[1, 1]`.
-   The number `7` can be written as `1 + 1 * 2 + 1 * 4`, encoded as `[1, 1, 1]`.

This same decomposition concept can be applied to larger bases. For example, in o1js, instead of base `2`, you might use `2^88`:

-   `[1, 1, 1]` in this context represents:
-   `1 + 1 * 2^88 + 1 * (2^88)^2 = 1 + 1 * 2^88 + 1 * 2^176`.

The problem:
============

The decomposition is only well-defined (or unique) if each component in the representation remains within a specified range.

Bit decomposition example
=========================

In bit decomposition, each entry must be either `0` or `1`. If this condition is violated, ambiguities arise.

For instance, `7` could be decomposed in multiple ways:`[1, 3, 0]` or `[1, 1, 1]`.

This happens because you can "borrow" from higher components. Example:

-   `1 + 1 * 2 + 1 * 4` can also be expressed as:
-   `1 + (1 + 2) * 2 + (2 - 2) * 4 = 1 + 3 * 2`.

Larger Bases (e.g., `2^88`)
===========================

For larger bases like `2^88`, each entry must satisfy:`0 ≤ entry < 2^88`.

Without this constraint, similar ambiguity occurs: You can "add" or "subtract" between components to create alternate decompositions.

Specific bug details
====================

In this case, a custom type was represented using **three limbs**, each of size `2^88`.

However, there was **no check** to ensure that the limbs were actually within the range `[0, 2^88 - 1]`.

Impact and the fix:
===================

An attacker can manipulate the values of these limbs and carefully choose values that **overflow** during subsequent computations.

This creates opportunities for cryptographic exploits and undermines the integrity of the protocol.

The root cause of this vulnerability --- and similar ones --- is **missing range checks**. Ensuring proper type and range validation is critical to maintaining the security and correctness of zero-knowledge circuits.

2) Hash collisions in merkle map key-to-index mapping
=====================================================

The basic idea of bug V-O1J-VUL-002 is that a mapping is being stored in a Merkle tree with more leaves than keys in the map.

The problem
===========

`key`s are limited to 254 bits, so they lie within the range `[0, 2**254)`. However, the Merkle tree has more `index` es than there are unique `key` s!

This means some `index` es in the tree must map to the **same **`**key**`.

In fact, it is straightforward to determine which `indexes` share the same `key`---there are trillions of possibilities.

Simplified example & exploit
============================

Suppose a `key` is an address and a `value` indicates whether the address is blacklisted (`true`) or not (`false`).

A single `key` might correspond to **two distinct indexes** in the Merkle tree: At one index, the value stored is `true`. At another index, the value stored is `false` (an edge case overlooked by developers).

The attacker can **choose which index to use**, enabling them to exploit the system. Naturally, the attacker will select the index with the value advantageous to them.

To summarize, the core issue is that instead of a one-to-one relationship between `keys` and `indexes`, some `keys` correspond to multiple `indexes`. This allows attackers to exploit the ambiguity and choose the mapping that benefits them.

Impact and the fix
==================

As shown in the above proof of concept, a user may prove that some entries of a MerkleMap are empty, even after they have been set. This can have critical consequences, such as user could prove their address is not in a blacklist, or that a certain nullifier is not in a Merkle tree.

We recommended a range-check on to prevent this overflow.

While this high-level overview omits some details, it captures the essence of the vulnerability. Full description of the bug can be found in the audit report, PDF page 15.

3) The hidden dangers of hash inversion in ZK circuits
======================================================

The core concept in the first bug (V-O1J-VUL-001, PDF page 15) revolves around *multiscalar multiplication* and a bug in a *compress-select-decompress*process. This bug would have enabled attackers to have control over the output of the multi-scalar computation.

In this blog post, we're giving a simplified example of the bug for easier comprehension and readability. The actual bug specific to o1js can be studied in the audit report.

What are multiscalar multiplications?
=====================================

At a high level, multiscalar multiplication essentially means performing multiple multiplications and adding the results together. However, instead of multiplying numbers, in this context one is usually dealing with cryptographic constructs called elliptic curve points.

Multiscalar multiplication (MSM) is usually implemented as a specialized operation designed to make common calculations faster and more efficient.

Rather than describe the full context, this blog will focus on a particular step in the multi-scalar multiplication algorithm. At this step, o1js needs to choose between two possible values for an array. For details on how this fits into the larger MSM algorithm, see [here](https://crypto.stackexchange.com/questions/99975/strauss-shamir-trick-on-ec-multiplication-by-scalar). For the purposes of this blog, just know that if an attacker can control which of the two possible arrays is chosen, they can potentially alter the output of standard cryptographic algorithms like [ECDSA](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm).

Challenges in ZK Circuits: there are no if-else statements
==========================================================

ZK circuits lack traditional control flow structures like `if-else` statements. Instead, both options (e.g., left and right) must be computed, and the desired option is selected by multiplying it by `1`, while the undesired option is multiplied by `0`.

This approach can be inefficient, especially when the options involve large datasets. For example, if Option 1 sets 10 values (e.g., `1, 2, 3, ... 10`) and Option 2 sets another 10 values (e.g., `11, 12, 13, ... 20`), the circuit essentially computes both options in full and pays the computational cost for each, even though only one is used. This inefficiency is the problem o1js aims to optimize.

Optimization Approach
=====================

1.  Compression: Instead of working with all 10 values directly, the values are first hashed together. This creates a single "commitment" value representing the 10 values.
2.  Decision: Both the left and right options are compressed into single hashes. The algorithm then decides between these compressed options.
3.  Decompression: After the decision is made, the chosen compressed value is decompressed back into the original 10 values.

By compressing the data first, the circuit avoids the overhead of making the decision 10 times. Instead, it only makes the decision once, based on the compressed hashes.

The core issue --- Compress-select-decompress process
===================================================

The key vulnerability lies in the **compress-select-decompress** process. In ZK circuits, decompression must reliably produce the exact original values from the compressed hash. Any mismatch here could lead to attackers gaining control of the execution.

Why this can fail
=================

Compression functions typically handle data as `1s` and `0s`. These binary representations might correspond to scalars, field elements, or booleans, or other types.

If the same binary data (`1s` and `0s`) can be decoded into multiple possible values, the decompression process might yield incorrect or unexpected results. This ambiguity creates a potential exploit.

Compression is not injective
============================

In simple terms, the issue in the o1js code arises from an optimization routine that is not *injective*. Injective means that the decoding process can produce multiple valid solutions, leading to potential vulnerabilities.

Here's how the original optimization process works:

1.  **Compression:**

-   The values `[a0, a1]` are compressed into a single hash: `hash(encode([a0, a1]))`.
-   Similarly, `[b0, b1]` is compressed: `hash(encode([b0, b1]))`.

**2\. "Switch" step:**

-   The algorithm selects one of the two compressed options and assigns it to `choice`.

**3\. Decompression:**

-   The algorithm verifies that the decompressed result matches the chosen hash:`assert(hash(encode([r0, r1])) == choice)`.

**4\. Result Extraction:**

-   After decompression, `[r0, r1]` are obtained as the resulting values.

The encoding process lacks the necessary property to guarantee that decoding and re-encoding will always yield the same result. In other words:

`decode(encode([r0, r1])) == choice`

does not consistently hold true. This means that the same encoded value can be decoded into multiple different outputs, introducing ambiguity.

Such behavior creates a vulnerability, as the system may fail to reliably match the original values after decompression. This non-injective encoding undermines the security and correctness of the algorithm.

Let's understand this with an example encoding function
=======================================================

Consider an array of length 2, where each value is a boolean (0 or 1). Possible values for the array are:`[0, 0]`, `[0, 1]`, `[1, 0]`, or `[1, 1]`

We define the following:

-   **Option 0:** `[a0, a1]`
-   **Option 1:** `[b0, b1]`
-   **Selector variable:** `s`, which can be `0` or `1`
-   **Goal:** Store the correct value into the result `[r0, r1]`

The slow way
============

For each element in the result:

Compute `r0` using:`r0 := (1-s) * a0 + s * b0`

**Why does this work?**

-   When `s = 0`:`r0 := (1 - 0) * a0 + 0 * b0 = a0`
-   When `s = 1`:`r0 := 0 * a0 + 1 * b0 = b0`

Similarly, compute `r1` using:

-   `r1 := (1-s) * a1 + s * b1`

This computation is sometimes called "switching" between a and b.

This approach works but can be **slow**, as it requires repeating the computation for each element. With an array of length two, this is not so bad. But as the arrays get longer and longer, the repeated computation can take a toll.

Bad optimization
================

Instead of directly switching, the optimization uses an **encoding function**:

-   `encode([a0, a1]) = a0 + 2 * a1`
-   `encode([b0, b1]) = b0 + 2 * b1`

This encoding maps the arrays uniquely:

-   `[0, 0] → 0 + 2 * 0 = 0`
-   `[0, 1] → 0 + 2 * 1 = 2`
-   `[1, 0] → 1 + 2 * 0 = 1`
-   `[1, 1] → 1 + 2 * 1 = 3`

This seems fine so far, as the mappings are unique.

Optimization process
====================

1.  **Compression:** Compress both options into single values:

cA := a0 + 2 * a1\
cB := b0 + 2 * b1

1.  **Switch:** Use the selector `s` to choose between the compressed values:

cR := (1-s) * cA + s * cB

1.  **Decompression:** Generate `r0, r1` from the witness generator and add a constraint to ensure the decompressed values match:

r0 + 2 * r1 = cR

Example exploit
===============

The issue lies in the **decompression** step. The witness generator is not constrained to produce valid boolean values (0 or 1) for `r0` and `r1`.

This means that instead of `[r0, r1]` being restricted to `[0, 1]`, they can take arbitrary values.

An attacker could choose `r0 = 1,000,000` and `r1 = -500,000`. Let's compute the compressed value:

cR = r0 + 2 * r1 = 1,000,000 + 2 * (-500,000) = 1,000,000 - 1,000,000 = 0

This satisfies the constraint `r0 + 2 * r1 = cR`, but clearly, the values `[r0, r1]` do not represent valid boolean values.

Root cause
==========

During compression, the original arrays `[a0, a1]` and `[b0, b1]` were constrained to boolean values (0 or 1).

However, during decompression, the original code failed to enforce these constraints, allowing the witness generator to produce invalid values.

This lack of constraints makes the encoding **non-injective**, meaning the decompressed values can correspond to multiple possible outputs, enabling attackers to exploit the system.

Impact and the fix
==================

The vulnerability arises because the decompression step lacks proper constraints, breaking the injective property of the encoding-decoding process. To fix this, the system must enforce that `r0` and `r1` remain within their valid range (0 or 1) during decompression.

Final remarks
=============

o1js has a very active set of contributors and is very interested in security. They have worked hard to bring ZK technology to TypeScript developers, and overcome a set of unique challenges in bringing smart contract logic to the Mina blockchain.

We found out it remarkably straightforward to prototype proof-of-concept applications using the TypeScript library. This ease of use extends to identifying under-constrained bugs and testing circuits in ways that developers might not have anticipated.

If you're planning to develop dapps on the Mina blockchain using TypeScript, stay tuned --- our upcoming blog post will provide insights and best practices for secure development, drawing from our auditing experience.

Full audit report
=================

Download the full o1js [security audit report here.](https://veridise.com/audits-archive/company/o1-labs/o1-labs-o1js-2024-08-27/)

**Author**:\
Ben Sepanski, Chief Security Officer at Veridise

Editor: Mikko Ikola, VP of Marketing
