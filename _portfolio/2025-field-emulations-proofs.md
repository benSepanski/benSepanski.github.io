---
title: "Field Emulation Proofs"
excerpt: "Z3 proofs of soundness for field emulation in ZK-circuits"
collection: portfolio
---

Check out the below repository! It contains [Z3](http://github.com/z3prover) proofs (i.e. formal, machine-checkable mathematical proofs) that ZK arithmetizations emulating cryptographically-sized finite field operations are sound. This means that, any solution to the constraints, corresponds to a valid "bigfield" operation.

https://github.com/VeridiseAuditing/field-emulation-proofs

I worked on this during an audit of [Aztec bigfield circuits](https://github.com/AztecProtocol/aztec-packages/tree/65107c5d576474150d6926ffe0b10f554d7fe046/barretenberg/cpp/src/barretenberg/stdlib/primitives/bigfield). These files have since been back-contributed to Aztec (see https://github.com/AztecProtocol/aztec-packages/pull/17152), although the Veridise repository is maintained separately in case there are any future additions.
