# Overview

> What the Spark Execution Framework is, where it sits, how its two-level control model works, and what it does not yet close. The bridge document — read it to understand how the pieces relate; read each numbered document to understand what it requires.

---

## What this framework is

The Specification Framework and the Execution Contract are peers — two foundations, neither containing the other. The specification pillar says *what correct is*; the execution pillar guarantees *work stays in bounds and correctness is verified before anything is trusted*. This framework is one concrete implementation of the **execution** pillar: an Assembly-Line-Protocol-class framework that discharges all eight building blocks of the Execution Contract on a specific, bounded substrate — a single NVIDIA DGX Spark.

It is named and opinionated on purpose. The Execution Contract is technology-neutral; this framework is not. It commits to one machine, one control model, and one data model, and it argues conformance against the contract. A different executor honouring the same contract and the same Work-Unit Interface would satisfy the same obligations differently — the conformance is portable even though the substrate is fixed.

## Where it sits

```
ai-development-foundations  (Specification Framework ║ Execution Contract)
        │  defines the standard
        ▼
Spark Execution Framework   (this repo — discharges the Execution Contract)
        │  executes work units produced by
        ▲
product-framework  ──── Work-Unit Interface ────►  (the seam: producer-owned, spec-stewarded)
        │
        ▼
the running Spark pipeline  (separate implementation repo — depends on this one)
```

The framework consumes work units a specification pillar (e.g. product-framework via product-cli) produces, runs them, and emits verdict events. It owns nothing on the specification side and negotiates nothing about the interface — it conforms to a contract the specification pillar stewards.

## The premise: a bandwidth-bound box forces the architecture

The Spark is strong at exactly one thing — **batched inference of small/medium models**, where many concurrent executions share a resident model — and weak at its opposite: a single large model decoding serially is bandwidth-bound to roughly 33 tok/s, its worst regime. Two workloads want the box in mutually exclusive configurations: batched execution wants many small models resident; local exploration wants one large model resident, deep and serial. They cannot co-reside in VRAM.

Therefore the box has a **mode**, set externally by a developer, never inferred by the box. This is the spine of the whole framework: a human-thrown switch over what the single box is for, with all machine-rate dynamism contained below it.

## The two-level control model

```
┌─ DEVELOPER SWITCH  (human-rate · rare · declarative) ──────────────┐
│   box configuration:  QUEUE  |  EXPLORER     (exclusive in VRAM)   │
│                                                                    │
│   EXPLORER → a local frontier reasoner: largest resident model,    │
│     OpenCode loop, serial. Produces a DISCOVERY RECORD, not code.  │
│                                                                    │
│   QUEUE → a batched execution engine. Below the switch, the        │
│     autonomous machinery runs at machine-rate:                     │
│   ┌──────────────────────────────────────────────────────────┐    │
│   │  work-unit priority set (flat · reorderable · independent)│    │
│   │  executor admits a unit → walks its sealed cell-DAG →     │    │
│   │     batches the unblocked cell frontier on the resident   │    │
│   │     binding · gates · rolls verdicts up · escalates       │    │
│   └──────────────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────────────────┘
  OFF-BOX (Opus-class · always available · not a box mode):
     define-step authoring · non-local exploration
```

Two loops, two change-rates. The top is a deliberate human act, rare because residency flips are expensive. The bottom is continuous and autonomous. The structure exists to keep these from mixing: queue dynamism never escapes upward to flip the box; the box configuration never reaches down to micromanage the queue.

`docs/02-substrate.md` specifies this in full.

## How conformance works

The framework's normative weight is in `docs/01-execution-framework.md`: a discharge of each of the Execution Contract's eight building blocks (Workers, Verification, Capabilities, Environment, Tools, Credentials, Input/Output/Transition contracts) by a named mechanism, plus the funnel (Rule 1), plus an end-to-end closure trace. The contract's own test is closure — a unit of work traced end to end with nothing dangling — and the framework is built to pass it.

On the specification side it *supports* the Product Framework's conformance ladder: it supplies the execution-side machinery that lets an instance reach Realised (pointer-to-frozen-value resolution at dispatch), Verified (cell-gates against a protected oracle), and Delivered (done as a computed reduction of verdict events). It does not grant conformance — the instance's artifacts do — but rungs 2–4 cannot be operationally satisfied without it.

## The one invariant worth knowing up front

A work-unit is well-formed iff all of its cells require the same **SPMC Model binding** — same model identity, same served quantization, same invocation parameters. Stated over the *binding*, not a loose capability tier, because on this hardware quantization is what makes a model fit and two cells at the same "tier" but different quantization are different bindings, hence a confound. This is the funnel at the work-unit scale, asserted by both foundations (the Execution Contract's Rule 1 and the Specification Framework's Model axis). It does triple duty: a pre-dispatch conformance check, an escalation simplifier (unit-atomic escalation becomes lossless), and a maturation metric (the heterogeneity rate falls as the corpus types itself).

## What this framework does NOT yet close

Three questions are open, and each is resolvable only by carrying a concrete specification through the chain — not by more design:

1. **Is any legitimate work-unit pattern intrinsically binding-mixed?** The invariant goes normative only if the answer is no. Test-first is homogeneous; that is one pattern, not a proof.
2. **Does a fully-resolved SPMC bundle ever leave a dangling reference?** The loose-coupling claim depends on dispatch freezing a callback-free bundle. Only freezing real bundles shows whether one dangles.
3. **Does the slice→unit→cell fan-out produce sensible grain?** The utilisation argument assumes units decompose into a dense-enough cell frontier. That is an empirical property of real specs.

A **degraded sovereign mode** — a local large model substituting for off-box authoring/exploration under a hard offline constraint — is acknowledged but unspecified.

These are not gaps in the design; they are the design correctly declaring what it cannot know without a concrete instance. The product-framework's worked example is the intended fixture for closing all three.

---

*This document is the bridge across the framework's three numbered documents and its interface spec. Read it for orientation; read each document for what it requires.*
