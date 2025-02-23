---
title: 'Mastering o1js on Mina: Four key strategies for secure development'
date: 2025-02-19
permalink: /posts/2025/02/19/Mastering-o1js-on-Mina-Four-key-strategies-for-secure-development/
tags:
  - Mina
  - ZK
  - o1js
  - Secure development
---

For the original (and properly formatted) publication, see [Veridise's post on medium](https://medium.com/veridise/mastering-o1js-on-mina-four-key-strategies-for-secure-development-fff3a3f4f6d1).

---

If you're building on the Mina blockchain, this blog post is a must-read.

After completing an extensive [security audit of the o1js v1 library](https://www.o1labs.org/blog/the-o1js-audit) --- spanning 39 person-weeks with 4 security analysts --- we've gained a comprehensive understanding of the framework. We're eager to share our insights with the community.

To help you develop securely with the o1js TypeScript library, we've distilled our findings into four topics. In this post, we highlight common pitfalls, share real-world examples of vulnerabilities, and provide actionable guidance to ensure your projects harness o1js effectively and securely.

Writing ZK Circuits with o1js and TypeScript
============================================

o1js brings cutting-edge technology to developers, enabling them to write ZK circuits in TypeScript and seamlessly deploy them to the Mina blockchain. While o1js abstracts much of the complexity of zero-knowledge proofs, it also introduces unique challenges and anti-patterns that developers must carefully navigate.

In this post, we dive into four illustrative examples:

-   Example #1: Trust the Types
-   Example #2: It's still ZK under the hood: No conditional data flow!
-   Example #3. Account Updates: Mastering on-chain state management
-   Example #4: Don't get DoS'ed: Actions & Reducers

Let's dive in!

Example #1: Trust the Types
===========================

A `UInt64` type is a supposed to represent a value in the range [0, 2⁶⁴), i.e. 0, 1, 2, 3, ..., up to 18,446,744,073,709,551,615

`UInt64.assertLessThan()` allows you to assert that a certain value `x` is less than another value

```
Provable.runAndCheck(() => {\
  // Introduce a UInt64 variable in the program\
  let x = Provable.witness(UInt64, () => {return new UInt64(100n);});\
  // Prove the variable is at most 2**48\
  x.assertLessThan(new UInt64(2n**48n));\
})
```

In the above program, all we prove is that `x` is some number less than 2⁴⁸. We set `x` to `100` in our generator, but anyone can change the witness generator. More specifically, *anything inside the *`*Provable.witness*`* function is not proven! *`*x*`* can have any value, so long as it satisfies the constraints!*

But where are constraints defined?

`Provable.witness(UInt64, ...)` adds constraints defined in `UInt64.check()` to ensure that verification won't succeed unless `x` is in the range [0, 2⁶⁴)

`x.assertLessThan(new UInt64(2n**48n))` then asserts that `x` is in the range [0, 2⁴⁸)

So far so good....

Now a clever user looks at this and might think: "Why check `x` is in [0, 2⁶⁴) if we are going to check `x` is in [0, 2⁴⁸) anyway? We can just do the second check!"

```
// BUG IN CODE: DO NOT USE\
Provable.runAndCheck(() => {\
  // Introduce an unconstrained variable in the program\
  let xAsField = Provable.exists(1, () => {return new UInt64(100n);});\
  // Unsafe cast it to a UInt64. This adds no constraints\
  let x = UInt64.Unsafe.from(xAsField);\
  // Prove the variable is at most 2**48\
  x.assertLessThan(new UInt64(2n**48n));\
})
```

While this looks innocuous, it is actually under-constrained! We can use values for `x` which are much larger than the input field.

Why?

-   `UInt64.assertLessThan()` assumes that *we already know *`*x < 2**64*`*.* Under the hood, it then asserts that 2⁴⁸ - x is in the range [0, 2⁶⁴). Remember that x is *really* a field element, so all arithmetic occurs modulo `p` for [some large ](https://electriccoin.co/blog/the-pasta-curves-for-halo-2-and-beyond/)`[p](https://electriccoin.co/blog/the-pasta-curves-for-halo-2-and-beyond/)`, and we can think of the range [0, p) as all possible values of `x`!

The implementation has the following cases:

1.  If x ∈ [0, 2⁴⁸), then 0 ≤ 2⁴⁸ --- x < 2⁴⁸ < 2⁶⁴, so the check passes ✅
2.  If x ∈ [2⁴⁸, p + 2⁴⁸ --- 2⁶⁴), then 2⁶⁴ ≤ 2⁴⁸ --- x, so the check fails p + 2⁴⁸ --- 2⁶⁴ ≈ p ≈ 2²⁵⁴ (note p + 2⁴⁸ --- 2⁶⁴ ≈ p ≈ 2²⁵⁴ ) ✅ (just like we want)
3.  If x ∈ [p + 2⁴⁸ --- 2⁶⁴, p) (i.e. the 2⁶⁴ --- 2⁴⁸ --- 1 *largest elements in the field, much larger than 2⁴⁸), *then the computation overflows all the way back into the range 0 ≤ 2⁴⁸ --- x < 2⁴⁸! ❌ This is bad: the check will pass but x is much larger than 2⁴⁸

This is why we can't cheat: we *need* the constraint x ∈ 2⁶⁴ from `Provable.witness(UInt64, ...)` to eliminate this 3rd "bad case".

Example #2: It's still ZK under the hood: No conditional data flow!
===================================================================

One common mistake we've seen across ecosystems comes from a limitation of ZK itself: the circuit's control flow is entirely static.

No if/else

How can we do computations then? Conditional data flow.

Instead of

if case1 {\
  x = f(y)\
}\
else {\
  x = g(y)\
}

We have to compute `f(y)`, compute `g(y)`, and then set

x = case1 ? f(y) : g(y)

Of course, in `o1js` this would look like

x = Provable.if(\
  case1,\
  MyProvableType, // tell o1js the type to use\
  x,\
  y\
)

How can this cause issues? We often want to do computations which *might*succeed. A classical example is division: division-by-zero is undefined.

Suppose we are implementing a simple vault. The vault holds funds. You can deposit funds equaling 1% of the currently managed funds, and the vault will mint you a number of tokens equal to 1% of the currently existing tokens.

For example, imagine the supply of vault `Token`s is `1000 Token`, and the vault is holding `100 USDC`. If I deposit `1 USDC`, the vault will mint me `1 USDC * 1000 Token / 100 USDC = 10 Token`. If I deposit `10 USDC`, the vault will mint `10 USDC * 1000 Token / 100 USDC = 100 Token`.

Ignoring decimals, this might be written simply as

amountToMint = totalSupply.mul(depositedAmount).div(managedFunds)

But what about the starting case? Suppose there are *no* funds and we are the first depositor. Commonly, we will set some fixed ratio: e.g. mint 1 token per initial USDC deposited. A first attempt at implementing this might be the below:

```
// BUG IN CODE: DO NOT USE\
amountToMint = Provable.if(\
  managedFunds.equals(new UInt64(0n)),\
  UInt64,                               // Output type\
  depositedAmount,\
  totalSupply.mul(depositedAmount).div(managedFunds)\
)
```

This looks like it will work: if no funds are currently managed, `amountToMint`is `depositedAmount`. Otherwise, we compute the ratio of tokens to managed funds.

The problem is simple: we *always compute*`totalSupply.mul(depositedAmount).div(managedFunds)`, even when `managedFunds` is equal to zero. To guarantee its correctness, `UInt64.div()` will cause an assertion failure when the denominator is zero.

This may seem not so bad: we'll catch it during testing. The problem is, it isn't always so obvious. For example, what if the above contract starts with a non-zero amount of funds/total supply set up before the vault was opened to the public? Then this issue will only manifest if the `managedFunds` reaches 0, at which point it can never be deposited into again.

A more serious (but analogous) example could prevent the last user from withdrawing from the vault.

Example #3: Account Updates: Mastering on-chain state management
================================================================

While o1js looks a lot like other popular smart contract languages, there are some important differences.

Each `@method` creates an `AccountUpdate`: A single object representing a change of on-chain state. These `AccountUpdate`s have preconditions on the state to ensure valid access. For example, if I am decreasing a smart contract Mina balance by `10`, the node must validate the account has at least 10 Mina---even if I have a proof that I executed the smart contract correctly.

Why? Remember that ZK is "state-less:" it has no hard drive, disk, or memory. All you prove is that, for some inputs which only you know, you did the correct computation.

When dealing with on-chain values, we need to prove that we used the actual on-chain values, and not just some random numbers! How? We output the "private-input" as "public preconditions" of the `AccountUpdate`. The node can then check the public preconditions

With state, we do this via the `getAndRequireEquals()` function. o1js will cause an error if you call `get()` without `getAndRequireEquals()`. It will now also call an error if you `getAndRequireEquals()` with two different values, due to an issue reported by Veridise (see [#1712 Assert Preconditions are not already set when trying to set their values](https://github.com/o1-labs/o1js/pull/1712))

Let's take a simple example.

// BUG IN CODE: DO NOT USE\
export class Pool extends SmartContract {\
  @state(Boolean) paused = State<Boolean>();

  @method function mint(amount: UInt64): UInt64 {\
    this.paused.get().assertFalse("Pool paused!");\
    // Minting logic\
  }\
}

The above snippet is intended to prevent minting when the protocol is paused. In actuality, it just proves that the prover set `paused` to `False` in their local environment when generating the proof. To ensure that the network validates this assumption, the code should instead use `getAndRequireEquals()`. This way, the assumption `paused = False` is included in the `AccountUpdate` as part of the proof, forcing the network node to validate that the on-chain protocol is not paused.

export class Pool extends SmartContract {\
  @state(Boolean) paused = State<Boolean>();

  @method function mint(amount: UInt64): UInt64 {\
    this.paused.getAndRequireEquals().assertFalse("Pool paused!");\
    // Minting logic\
  }\
}

Example #4: Don't get DoS'ed: Actions & Reducers
================================================

Preconditions can cause problems when concurrent accesses are occurring. Say you and I are both incrementing a counter. Our `AccountUpdate` will have two important parts:

-   A precondition that the current counter value is the old value `x`
-   A new value for the counter: `x+1`

If you and I both call this function when `x = 3`, we *both have a precondition *`*x=3*`*!* That means whichever one of us is executed second will have our `AccountUpdate` fail, since after the first person goes, `x = 4 != 3`

How can we fix this? We queue up actions.

Mina has a feature called [actions & reducers](https://www.google.com/search?client=safari&rls=en&q=actions+and+reducers+mina&ie=UTF-8&oe=UTF-8). You can submit an "Action", which gets put in a queue. Later, users can call a "reducer" which calls a function on those actions

Let's look at an example taken from [reducer-composite.ts](https://github.com/o1-labs/o1js/blob/main/src/examples/zkapps/reducer/reducer-composite.ts):

class MaybeIncrement extends Struct({\
  isIncrement: Bool,\
  otherData: Field,\
}) {}\
const INCREMENT = { isIncrement: Bool(true), otherData: Field(0) };

class Counter extends SmartContract {\
  // the "reducer" field describes a type of action that we can dispatch, and reduce later\
  reducer = Reducer({ actionType: MaybeIncrement });

  // on-chain version of our state. it will typically lag behind the\
  // version that's implicitly represented by the list of actions\
  @state(Field) counter = State<Field>();\
  // helper field to store the point in the action history that our on-chain state is at\
  @state(Field) actionState = State<Field>();

  @method async incrementCounter() {\
    this.reducer.dispatch(INCREMENT);\
  }\
  @method async dispatchData(data: Field) {\
    this.reducer.dispatch({ isIncrement: Bool(false), otherData: data });\
  }

  @method async rollupIncrements() {\
    // get previous counter & actions hash, assert that they're the same as on-chain values\
    let counter = this.counter.getAndRequireEquals();\
    let actionState = this.actionState.getAndRequireEquals();

    // compute the new counter and hash from pending actions\
    let pendingActions = this.reducer.getActions({\
      fromActionState: actionState,\
    });

    let newCounter = this.reducer.reduce(\
      pendingActions,\
      // state type\
      Field,\
      // function that says how to apply an action\
      (state: Field, action: MaybeIncrement) => {\
        return Provable.if(action.isIncrement, state.add(1), state);\
      },\
      counter,\
      { maxUpdatesWithActions: 10 }\
    );

    // update on-chain state\
    this.counter.set(newCounter);\
    this.actionState.set(pendingActions.hash);\
  }\
}

In this code, `incrementCounter()` dispatches an action to the queue requesting an increment. `dispatchData()` adds a queue with some other unrelated data.

Anyone can process the entire queue by calling `rollupIncrements()`. This will go through the whole queue, incrementing once for each submitted (but unprocessed) request to increment.

Note:

-   The contract is responsible for managing the `actionState` field, which tracks "where we are in the queue." In particular, `this.actionState` tracks what parts of the queue have been *processed*, while the Mina node automatically tracks what actions have been *submitted* via `dispatch`

Suppose that, instead of just incrementing by one, the user provided a number to add (e.g. an account balance change).

class MaybeIncrement extends Struct({\
  isIncrement: Bool,\
  otherData: Field,\
  amount: UInt64\
}) {}

// function that says how to apply an action\
(state: Field, action: MaybeIncrement) => {\
  return Provable.if(action.isIncrement, state.add(action.amount), state);\
},

A malicious user could submit several large `amount`s, e.g.

{\
  isIncrement: new Bool(0),\
  otherData: new Field(0),\
  amount: new UInt64(UInt64.MAX),\
}

Action submission will work smoothly, but this single action can permanently prevent all other actions from being processed!

Using this simplified example, the only way to process actions is in a single, large batch. Using more complex constructions like the batch reducer, or a custom rollup proof can get around this issue, but at the time of audit, only simple examples were made available for us to review.

If you decide to use the actions and reducer pattern, the reducer must be *guaranteed to succeed* once an action is submitted. This means that any arithmetic must be inspected carefully, actions must be strictly validated at submission, and must be canonicalized (see this [PR #1759 Canonical representation of provable types](https://github.com/o1-labs/o1js/pull/1759), which was developed as a solution to an issue raised during the Veridise audit).

To see what other solutions the O(1) community is working on, [check out this article from zkNoid.](https://medium.com/zknoid/mina-action-reducers-guide-writing-our-own-reducers-81802287776f)

Closing thoughts
================

o1js empowers developers to build cutting-edge zero-knowledge applications on the Mina blockchain, leveraging the simplicity of TypeScript to create and deploy ZK circuits.

However, this power comes with responsibilities. Developers must be vigilant about potential vulnerabilities, from respecting type constraints and avoiding under-constrained circuits to managing state updates effectively and mitigating concurrency risks.

We hope the examples and strategies shared in this blog provide a solid foundation for developing securely on Mina. The challenges and pitfalls of working with zero-knowledge circuits can be intricate, but with careful attention to detail and adherence to best practices, they can be navigated successfully.

Author
======

Ben Sepanski, Chief Security Officer at Veridise

Want to learn more about Veridise?
==================================

[Twitter](https://twitter.com/VeridiseInc) | [LinkedIn](https://www.linkedin.com/company/veridise/) | [Github](https://github.com/Veridise) | [Request Audit](https://veridise.com/request-audit/)

[](https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fbookmark%2Fp%2Ffff3a3f4f6d1&operation=register&redirect=https%3A%2F%2Fmedium.com%2Fveridise%2Fmastering-o1js-on-mina-four-key-strategies-for-secure-development-fff3a3f4f6d1&source=---footer_actions--fff3a3f4f6d1---------------------bookmark_footer------------------)

[

![Veridise](https://miro.medium.com/v2/resize:fill:96:96/1*LwZDepFjUwzEVNUrgIFXeA.png)

](https://medium.com/veridise?source=post_page---post_publication_info--fff3a3f4f6d1---------------------------------------)

[

Published in Veridise
---------------------

](https://medium.com/veridise?source=post_page---post_publication_info--fff3a3f4f6d1---------------------------------------)

[62 Followers](https://medium.com/veridise/followers?source=post_page---post_publication_info--fff3a3f4f6d1---------------------------------------)

-[Last published 4 days ago](https://medium.com/veridise/mastering-o1js-on-mina-four-key-strategies-for-secure-development-fff3a3f4f6d1?source=post_page---post_publication_info--fff3a3f4f6d1---------------------------------------)

Our mission in to harden blockchain security with formal methods. We write about blockchain security, zero-knowledge proofs, and our bug discoveries.

Follow

[

![Veridise](https://miro.medium.com/v2/resize:fill:96:96/1*LwZDepFjUwzEVNUrgIFXeA.png)

](https://medium.com/@veridise?source=post_page---post_author_info--fff3a3f4f6d1---------------------------------------)

[

Written by Veridise
-------------------

](https://medium.com/@veridise?source=post_page---post_author_info--fff3a3f4f6d1---------------------------------------)

[277 Followers](https://medium.com/@veridise/followers?source=post_page---post_author_info--fff3a3f4f6d1---------------------------------------)

-[3 Following](https://medium.com/@veridise/following?source=post_page---post_author_info--fff3a3f4f6d1---------------------------------------)

Hardening blockchain security with formal methods. We write about blockchain & zero-knowledge proof security. Contact us for industry-leading security audits.

Follow

No responses yet
----------------

[](https://policy.medium.com/medium-rules-30e5502c4eb4?source=post_page---post_responses--fff3a3f4f6d1---------------------------------------)

[

What are your thoughts?

](https://medium.com/m/signin?operation=register&redirect=https%3A%2F%2Fmedium.com%2Fveridise%2Fmastering-o1js-on-mina-four-key-strategies-for-secure-development-fff3a3f4f6d1&source=---post_responses--fff3a3f4f6d1---------------------respond_sidebar------------------)

Cancel

Respond

Respond

Also publish to my profile

More from Veridise and Veridise
-------------------------------

![Zero Knowledge for Dummies: Introduction to ZK Proofs](https://miro.medium.com/v2/resize:fit:1358/1*uwizzJfUnbne-5Y-pXBo2A.png)

[

![Veridise](https://miro.medium.com/v2/resize:fill:40:40/1*LwZDepFjUwzEVNUrgIFXeA.png)

](https://medium.com/veridise?source=post_page---author_recirc--fff3a3f4f6d1----0---------------------d84e01f0_b292_493f_bbea_aac71074a8f8--------------)

In

[

Veridise

](https://medium.com/veridise?source=post_page---author_recirc--fff3a3f4f6d1----0---------------------d84e01f0_b292_493f_bbea_aac71074a8f8--------------)

by

[

Veridise

](https://medium.com/@veridise?source=post_page---author_recirc--fff3a3f4f6d1----0---------------------d84e01f0_b292_493f_bbea_aac71074a8f8--------------)

[

Zero Knowledge for Dummies: Introduction to ZK Proofs
-----------------------------------------------------

### Do you have zero knowledge about zero knowledge? Do you want to learn more about it? You're in the right place and we have cookies.

](https://medium.com/veridise/zero-knowledge-for-dummies-introduction-to-zk-proofs-29e3fe9604f1?source=post_page---author_recirc--fff3a3f4f6d1----0---------------------d84e01f0_b292_493f_bbea_aac71074a8f8--------------)

Aug 24, 2023

[

150

1

](https://medium.com/veridise/zero-knowledge-for-dummies-introduction-to-zk-proofs-29e3fe9604f1?source=post_page---author_recirc--fff3a3f4f6d1----0---------------------d84e01f0_b292_493f_bbea_aac71074a8f8--------------)

[](https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fbookmark%2Fp%2F29e3fe9604f1&operation=register&redirect=https%3A%2F%2Fmedium.com%2Fveridise%2Fzero-knowledge-for-dummies-introduction-to-zk-proofs-29e3fe9604f1&source=---author_recirc--fff3a3f4f6d1----0-----------------bookmark_preview----d84e01f0_b292_493f_bbea_aac71074a8f8--------------)

![Zero Knowledge for Dummies: Demystifying ZK Circuits](https://miro.medium.com/v2/resize:fit:1358/1*7n6YZB61hMFV4kZrj8Azjw.png)

[

![Veridise](https://miro.medium.com/v2/resize:fill:40:40/1*LwZDepFjUwzEVNUrgIFXeA.png)

](https://medium.com/veridise?source=post_page---author_recirc--fff3a3f4f6d1----1---------------------d84e01f0_b292_493f_bbea_aac71074a8f8--------------)

In

[

Veridise

](https://medium.com/veridise?source=post_page---author_recirc--fff3a3f4f6d1----1---------------------d84e01f0_b292_493f_bbea_aac71074a8f8--------------)

by

[

Veridise

](https://medium.com/@veridise?source=post_page---author_recirc--fff3a3f4f6d1----1---------------------d84e01f0_b292_493f_bbea_aac71074a8f8--------------)

[

Zero Knowledge for Dummies: Demystifying ZK Circuits
----------------------------------------------------

### ZK circuits are the "magic tools" that enable ZK proofs

](https://medium.com/veridise/zero-knowledge-for-dummies-demystifying-zk-circuits-c140a64c6ed3?source=post_page---author_recirc--fff3a3f4f6d1----1---------------------d84e01f0_b292_493f_bbea_aac71074a8f8--------------)

Jan 19, 2024

[

27

](https://medium.com/veridise/zero-knowledge-for-dummies-demystifying-zk-circuits-c140a64c6ed3?source=post_page---author_recirc--fff3a3f4f6d1----1---------------------d84e01f0_b292_493f_bbea_aac71074a8f8--------------)

[](https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fbookmark%2Fp%2Fc140a64c6ed3&operation=register&redirect=https%3A%2F%2Fmedium.com%2Fveridise%2Fzero-knowledge-for-dummies-demystifying-zk-circuits-c140a64c6ed3&source=---author_recirc--fff3a3f4f6d1----1-----------------bookmark_preview----d84e01f0_b292_493f_bbea_aac71074a8f8--------------)

![Highlights from the Veridise o1js v1 audit: Three zero-knowledge security bugs explained](https://miro.medium.com/v2/resize:fit:1358/1*7kkTlG2Ah7C3RzsphuyMjQ.jpeg)

[

![Veridise](https://miro.medium.com/v2/resize:fill:40:40/1*LwZDepFjUwzEVNUrgIFXeA.png)

](https://medium.com/veridise?source=post_page---author_recirc--fff3a3f4f6d1----2---------------------d84e01f0_b292_493f_bbea_aac71074a8f8--------------)

In

[

Veridise

](https://medium.com/veridise?source=post_page---author_recirc--fff3a3f4f6d1----2---------------------d84e01f0_b292_493f_bbea_aac71074a8f8--------------)

by

[

Veridise

](https://medium.com/@veridise?source=post_page---author_recirc--fff3a3f4f6d1----2---------------------d84e01f0_b292_493f_bbea_aac71074a8f8--------------)

[

Highlights from the Veridise o1js v1 audit: Three zero-knowledge security bugs explained
----------------------------------------------------------------------------------------

### In 2024, the Veridise team conducted a comprehensive security audit of o1js, a crucial TypeScript library that powers zero-knowledge...

](https://medium.com/veridise/highlights-from-the-veridise-o1js-v1-audit-three-zero-knowledge-security-bugs-explained-2f5708f13681?source=post_page---author_recirc--fff3a3f4f6d1----2---------------------d84e01f0_b292_493f_bbea_aac71074a8f8--------------)

Feb 3

[](https://medium.com/veridise/highlights-from-the-veridise-o1js-v1-audit-three-zero-knowledge-security-bugs-explained-2f5708f13681?source=post_page---author_recirc--fff3a3f4f6d1----2---------------------d84e01f0_b292_493f_bbea_aac71074a8f8--------------)

[](https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fbookmark%2Fp%2F2f5708f13681&operation=register&redirect=https%3A%2F%2Fmedium.com%2Fveridise%2Fhighlights-from-the-veridise-o1js-v1-audit-three-zero-knowledge-security-bugs-explained-2f5708f13681&source=---author_recirc--fff3a3f4f6d1----2-----------------bookmark_preview----d84e01f0_b292_493f_bbea_aac71074a8f8--------------)

![Recursive SNARKs and Incrementally Verifiable Computation (IVC)](https://miro.medium.com/v2/resize:fit:1358/1*fcK7LTcLreBHGYykLcggcw.jpeg)

[

![Veridise](https://miro.medium.com/v2/resize:fill:40:40/1*LwZDepFjUwzEVNUrgIFXeA.png)

](https://medium.com/veridise?source=post_page---author_recirc--fff3a3f4f6d1----3---------------------d84e01f0_b292_493f_bbea_aac71074a8f8--------------)

In

[

Veridise

](https://medium.com/veridise?source=post_page---author_recirc--fff3a3f4f6d1----3---------------------d84e01f0_b292_493f_bbea_aac71074a8f8--------------)

by

[

Veridise

](https://medium.com/@veridise?source=post_page---author_recirc--fff3a3f4f6d1----3---------------------d84e01f0_b292_493f_bbea_aac71074a8f8--------------)

[

Recursive SNARKs and Incrementally Verifiable Computation (IVC)
---------------------------------------------------------------

### PART I: Recursive SNARKs and Incrementally Verifiable Computation

](https://medium.com/veridise/introduction-to-nova-and-zk-folding-schemes-4ef717574484?source=post_page---author_recirc--fff3a3f4f6d1----3---------------------d84e01f0_b292_493f_bbea_aac71074a8f8--------------)

Jul 27, 2023

[

70

](https://medium.com/veridise/introduction-to-nova-and-zk-folding-schemes-4ef717574484?source=post_page---author_recirc--fff3a3f4f6d1----3---------------------d84e01f0_b292_493f_bbea_aac71074a8f8--------------)

[](https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fbookmark%2Fp%2F4ef717574484&operation=register&redirect=https%3A%2F%2Fmedium.com%2Fveridise%2Fintroduction-to-nova-and-zk-folding-schemes-4ef717574484&source=---author_recirc--fff3a3f4f6d1----3-----------------bookmark_preview----d84e01f0_b292_493f_bbea_aac71074a8f8--------------)

[

See all from Veridise

](https://medium.com/@veridise?source=post_page---author_recirc--fff3a3f4f6d1---------------------------------------)

[

See all from Veridise

](https://medium.com/veridise?source=post_page---author_recirc--fff3a3f4f6d1---------------------------------------)

Recommended from Medium
-----------------------

![Animation in react native](https://miro.medium.com/v2/resize:fit:1358/0*dY_bMLLwDsM7qxqQ)

[

![Ankit](https://miro.medium.com/v2/resize:fill:40:40/1*FZQmGhGfj4f4gHYh3e3Olg@2x.jpeg)

](https://medium.com/@mohantaankit2002?source=post_page---read_next_recirc--fff3a3f4f6d1----0---------------------ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

[

Ankit

](https://medium.com/@mohantaankit2002?source=post_page---read_next_recirc--fff3a3f4f6d1----0---------------------ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

[

Making React Native Animations Buttery Smooth on Budget Phones
--------------------------------------------------------------

### Got a beautiful animation that turns into a slideshow on low-end devices? Let's fix that without compromising your app's wow factor.

](https://medium.com/@mohantaankit2002/making-react-native-animations-buttery-smooth-on-budget-phones-f6ff3d4215bd?source=post_page---read_next_recirc--fff3a3f4f6d1----0---------------------ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

Feb 15

[](https://medium.com/@mohantaankit2002/making-react-native-animations-buttery-smooth-on-budget-phones-f6ff3d4215bd?source=post_page---read_next_recirc--fff3a3f4f6d1----0---------------------ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

[](https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fbookmark%2Fp%2Ff6ff3d4215bd&operation=register&redirect=https%3A%2F%2Fmedium.com%2F%40mohantaankit2002%2Fmaking-react-native-animations-buttery-smooth-on-budget-phones-f6ff3d4215bd&source=---read_next_recirc--fff3a3f4f6d1----0-----------------bookmark_preview----ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

![Mastering React Native in 2025: Real Talk from a Developer's Perspective](https://miro.medium.com/v2/resize:fit:1358/1*3xKF0UIKrDYxA7qVq4rsew.png)

[

![Stackademic](https://miro.medium.com/v2/resize:fill:40:40/1*U-kjsW7IZUobnoy1gAp1UQ.png)

](https://medium.com/stackademic?source=post_page---read_next_recirc--fff3a3f4f6d1----1---------------------ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

In

[

Stackademic

](https://medium.com/stackademic?source=post_page---read_next_recirc--fff3a3f4f6d1----1---------------------ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

by

[

Abvhishek kumar

](https://medium.com/@abvhishekkumaar?source=post_page---read_next_recirc--fff3a3f4f6d1----1---------------------ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

[

Mastering React Native in 2025: Real Talk from a Developer's Perspective
------------------------------------------------------------------------

### React Native has been a go-to choice for cross-platform mobile development for a while now, and in 2025, it's more powerful and flexible...

](https://medium.com/stackademic/mastering-react-native-in-2025-real-talk-from-a-developers-perspective-96aa64910a20?source=post_page---read_next_recirc--fff3a3f4f6d1----1---------------------ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

6d ago

[

2

1

](https://medium.com/stackademic/mastering-react-native-in-2025-real-talk-from-a-developers-perspective-96aa64910a20?source=post_page---read_next_recirc--fff3a3f4f6d1----1---------------------ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

[](https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fbookmark%2Fp%2F96aa64910a20&operation=register&redirect=https%3A%2F%2Fblog.stackademic.com%2Fmastering-react-native-in-2025-real-talk-from-a-developers-perspective-96aa64910a20&source=---read_next_recirc--fff3a3f4f6d1----1-----------------bookmark_preview----ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

Lists
-----

[

![An illustration of a Featured story push notification for a publication follower.](https://miro.medium.com/v2/resize:fill:96:96/1*I1q8VWkZEXxUyv2B4umMzQ.png)

![](https://miro.medium.com/v2/resize:fill:96:96/0*oC1WfZBerS8zD1aW.jpeg)

![](https://miro.medium.com/v2/da:true/resize:fill:96:96/0*ayla5Doy42EwGrWu)

Staff picks
-----------

817 stories-1632 saves

](https://medium.com/@MediumStaff/list/staff-picks-c7bc6e1ee00f?source=post_page---read_next_recirc--fff3a3f4f6d1---------------------------------------)

[

![](https://miro.medium.com/v2/resize:fill:96:96/1*4zC5ohNcmVDb1NXmzCvmNA.jpeg)

![](https://miro.medium.com/v2/resize:fill:96:96/1*0dul7hn9LeV7U2XLVPvYYw.jpeg)

![](https://miro.medium.com/v2/resize:fill:96:96/1*oO7uwYs0NMWV7B4mUCuoIw.png)

Stories to Help You Level-Up at Work
------------------------------------

19 stories-942 saves

](https://medium.com/@MediumStaff/list/stories-to-help-you-levelup-at-work-faca18b0622f?source=post_page---read_next_recirc--fff3a3f4f6d1---------------------------------------)

[

![](https://miro.medium.com/v2/resize:fill:96:96/1*VQhBEVqZRXlxFvFVqyTYVA.jpeg)

![](https://miro.medium.com/v2/resize:fill:96:96/1*XRekBY0_j_KEo2_lK_SrkQ.jpeg)

![](https://miro.medium.com/v2/resize:fill:96:96/1*c2stHqktjG8_XTqkC-bkfw.jpeg)

Self-Improvement 101
--------------------

20 stories-3313 saves

](https://medium.com/@MediumForTeams/list/selfimprovement-101-3c62b6cb0526?source=post_page---read_next_recirc--fff3a3f4f6d1---------------------------------------)

[

![](https://miro.medium.com/v2/resize:fill:96:96/1*HWxGot_WiEOZ0fkB_ee2gg.jpeg)

![](https://miro.medium.com/v2/resize:fill:96:96/1*kAzwx9sMsEYm0bMYDTa-lQ.jpeg)

![](https://miro.medium.com/v2/resize:fill:96:96/1*GGLm6Juo_CxQAKEM-mAWfw.jpeg)

Productivity 101
----------------

20 stories-2787 saves

](https://medium.com/@MediumForTeams/list/productivity-101-f09f1aaf38cd?source=post_page---read_next_recirc--fff3a3f4f6d1---------------------------------------)

![A laptop, 2 phones, notebook, and other things](https://miro.medium.com/v2/resize:fit:1358/0*zQIIg7HiXOW75M4S)

[

![Andrew Zuo](https://miro.medium.com/v2/resize:fill:40:40/1*FZEG_DxaZ4g-w10VST7WGg.jpeg)

](https://medium.com/@impure?source=post_page---read_next_recirc--fff3a3f4f6d1----0---------------------ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

[

Andrew Zuo

](https://medium.com/@impure?source=post_page---read_next_recirc--fff3a3f4f6d1----0---------------------ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

[

Go Is A Poorly Designed Language, Actually
------------------------------------------

### I read an article titled Go is a Well-Designed Language, Actually. Hard disagree. I originally liked Go because it did simplify things. I...

](https://medium.com/@impure/go-is-a-poorly-designed-language-actually-a8ec508fc2ed?source=post_page---read_next_recirc--fff3a3f4f6d1----0---------------------ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

4d ago

[

263

24

](https://medium.com/@impure/go-is-a-poorly-designed-language-actually-a8ec508fc2ed?source=post_page---read_next_recirc--fff3a3f4f6d1----0---------------------ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

[](https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fbookmark%2Fp%2Fa8ec508fc2ed&operation=register&redirect=https%3A%2F%2Fandrewzuo.com%2Fgo-is-a-poorly-designed-language-actually-a8ec508fc2ed&source=---read_next_recirc--fff3a3f4f6d1----0-----------------bookmark_preview----ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

![MacOS: Building the Perfect Development Machine in 2025](https://miro.medium.com/v2/resize:fit:1358/1*GXomu5teh5riMgkvNvTLiw.jpeg)

[

![Level Up Coding](https://miro.medium.com/v2/resize:fill:40:40/1*5D9oYBd58pyjMkV_5-zXXQ.jpeg)

](https://medium.com/gitconnected?source=post_page---read_next_recirc--fff3a3f4f6d1----1---------------------ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

In

[

Level Up Coding

](https://medium.com/gitconnected?source=post_page---read_next_recirc--fff3a3f4f6d1----1---------------------ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

by

[

Promise Chukwuenyem

](https://medium.com/@promisepreston?source=post_page---read_next_recirc--fff3a3f4f6d1----1---------------------ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

[

MacOS: Building the Perfect Development Machine in 2025
-------------------------------------------------------

### How I Transformed My MacBook Into a Development Powerhouse

](https://medium.com/gitconnected/from-linux-to-mac-building-the-perfect-development-machine-in-2025-14db582f239f?source=post_page---read_next_recirc--fff3a3f4f6d1----1---------------------ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

Jan 13

[

351

3

](https://medium.com/gitconnected/from-linux-to-mac-building-the-perfect-development-machine-in-2025-14db582f239f?source=post_page---read_next_recirc--fff3a3f4f6d1----1---------------------ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

[](https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fbookmark%2Fp%2F14db582f239f&operation=register&redirect=https%3A%2F%2Flevelup.gitconnected.com%2Ffrom-linux-to-mac-building-the-perfect-development-machine-in-2025-14db582f239f&source=---read_next_recirc--fff3a3f4f6d1----1-----------------bookmark_preview----ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

![Highlights from the Veridise o1js v1 audit: Three zero-knowledge security bugs explained](https://miro.medium.com/v2/resize:fit:1358/1*7kkTlG2Ah7C3RzsphuyMjQ.jpeg)

[

![Veridise](https://miro.medium.com/v2/resize:fill:40:40/1*LwZDepFjUwzEVNUrgIFXeA.png)

](https://medium.com/veridise?source=post_page---read_next_recirc--fff3a3f4f6d1----2---------------------ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

In

[

Veridise

](https://medium.com/veridise?source=post_page---read_next_recirc--fff3a3f4f6d1----2---------------------ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

by

[

Veridise

](https://medium.com/@veridise?source=post_page---read_next_recirc--fff3a3f4f6d1----2---------------------ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

[

Highlights from the Veridise o1js v1 audit: Three zero-knowledge security bugs explained
----------------------------------------------------------------------------------------

### In 2024, the Veridise team conducted a comprehensive security audit of o1js, a crucial TypeScript library that powers zero-knowledge...

](https://medium.com/veridise/highlights-from-the-veridise-o1js-v1-audit-three-zero-knowledge-security-bugs-explained-2f5708f13681?source=post_page---read_next_recirc--fff3a3f4f6d1----2---------------------ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

Feb 3

[](https://medium.com/veridise/highlights-from-the-veridise-o1js-v1-audit-three-zero-knowledge-security-bugs-explained-2f5708f13681?source=post_page---read_next_recirc--fff3a3f4f6d1----2---------------------ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

[](https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fbookmark%2Fp%2F2f5708f13681&operation=register&redirect=https%3A%2F%2Fmedium.com%2Fveridise%2Fhighlights-from-the-veridise-o1js-v1-audit-three-zero-knowledge-security-bugs-explained-2f5708f13681&source=---read_next_recirc--fff3a3f4f6d1----2-----------------bookmark_preview----ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

!["Bitcoin is Going to Zero Within a Decade."](https://miro.medium.com/v2/resize:fit:1358/1*CblrYTeERV5wSbWrKeUwdw.jpeg)

[

![Crypto with Lorenzo](https://miro.medium.com/v2/resize:fill:40:40/1*6sOhW3xrimX5kCDlGd27ZA.png)

](https://medium.com/@cryptowithlorenzo?source=post_page---read_next_recirc--fff3a3f4f6d1----3---------------------ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

[

Crypto with Lorenzo

](https://medium.com/@cryptowithlorenzo?source=post_page---read_next_recirc--fff3a3f4f6d1----3---------------------ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

[

"Bitcoin is Going to Zero Within a Decade."
-------------------------------------------

### Breaking down the arguments made by an economics expert.

](https://medium.com/@cryptowithlorenzo/bitcoin-is-going-to-zero-5562122f5481?source=post_page---read_next_recirc--fff3a3f4f6d1----3---------------------ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

Feb 5

[

390

36

](https://medium.com/@cryptowithlorenzo/bitcoin-is-going-to-zero-5562122f5481?source=post_page---read_next_recirc--fff3a3f4f6d1----3---------------------ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

[](https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fbookmark%2Fp%2F5562122f5481&operation=register&redirect=https%3A%2F%2Fcryptowithlorenzo.medium.com%2Fbitcoin-is-going-to-zero-5562122f5481&source=---read_next_recirc--fff3a3f4f6d1----3-----------------bookmark_preview----ebb31a75_5f5f_46b7_8d3d_a06f39dfabc3--------------)

[

See more recommendations

](https://medium.com/?source=post_page---read_next_recirc--fff3a3f4f6d1---------------------------------------)

[

Help

](https://help.medium.com/hc/en-us?source=post_page-----fff3a3f4f6d1---------------------------------------)

[

Status

](https://medium.statuspage.io/?source=post_page-----fff3a3f4f6d1---------------------------------------)

[

About

](https://medium.com/about?autoplay=1&source=post_page-----fff3a3f4f6d1---------------------------------------)

[

Careers

](https://medium.com/jobs-at-medium/work-at-medium-959d1a85284e?source=post_page-----fff3a3f4f6d1---------------------------------------)

[

Press

](mailto:pressinquiries@medium.com)

[

Blog

](https://blog.medium.com/?source=post_page-----fff3a3f4f6d1---------------------------------------)

[

Privacy

](https://policy.medium.com/medium-privacy-policy-f03bf92035c9?source=post_page-----fff3a3f4f6d1---------------------------------------)

[

Terms

](https://policy.medium.com/medium-terms-of-service-9db0094a1e0f?source=post_page-----fff3a3f4f6d1---------------------------------------)

[

Text to speech

](https://speechify.com/medium?source=post_page-----fff3a3f4f6d1---------------------------------------)

[

Teams

](https://medium.com/business?source=post_page-----fff3a3f4f6d1---------------------------------------)ridise%2Fhighlights-from-the-veridise-o1js-v1-audit-three-zero-knowledge-security-bugs-explained-2f5708f13681&source=post_page---top_nav_layout_nav-----------------------global_nav------------------)

![](https://miro.medium.com/v2/resize:fill:64:64/1*dmbNkD5D-u45r44go_cf0g.png)

Highlights from the Veridise o1js v1 audit: Three zero-knowledge security bugs explained
========================================================================================

[

![Veridise](https://miro.medium.com/v2/resize:fill:88:88/1*LwZDepFjUwzEVNUrgIFXeA.png)

](https://medium.com/@veridise?source=post_page---byline--2f5708f13681---------------------------------------)

[

![Veridise](https://miro.medium.com/v2/resize:fill:48:48/1*LwZDepFjUwzEVNUrgIFXeA.png)

](https://medium.com/veridise?source=post_page---byline--2f5708f13681---------------------------------------)

[Veridise](https://medium.com/@veridise?source=post_page---byline--2f5708f13681---------------------------------------)

[Follow](https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fsubscribe%2Fuser%2Fd54583b14487&operation=register&redirect=https%3A%2F%2Fmedium.com%2Fveridise%2Fhighlights-from-the-veridise-o1js-v1-audit-three-zero-knowledge-security-bugs-explained-2f5708f13681&user=Veridise&userId=d54583b14487&source=post_page-d54583b14487--byline--2f5708f13681---------------------post_header------------------)

Published in

[

Veridise

](https://medium.com/veridise?source=post_page---byline--2f5708f13681---------------------------------------)

10 min read

Feb 3, 2025

[](https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fbookmark%2Fp%2F2f5708f13681&operation=register&redirect=https%3A%2F%2Fmedium.com%2Fveridise%2Fhighlights-from-the-veridise-o1js-v1-audit-three-zero-knowledge-security-bugs-explained-2f5708f13681&source=---header_actions--2f5708f13681---------------------bookmark_footer------------------)

[](https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2Fplans%3Fdimension%3Dpost_audio_button%26postId%3D2f5708f13681&operation=register&redirect=https%3A%2F%2Fmedium.com%2Fveridise%2Fhighlights-from-the-veridise-o1js-v1-audit-three-zero-knowledge-security-bugs-explained-2f5708f13681&source=---header_actions--2f5708f13681---------------------post_audio_button------------------)

![](https://miro.medium.com/v2/resize:fit:1400/1*7kkTlG2Ah7C3RzsphuyMjQ.jpeg)

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
