# The Spark Execution Framework

> An execution-pillar framework implementation. Conformant to the Two Pillars **Execution Contract**; built to satisfy the **Product Framework**'s conformance ladder (Described → Realised → Verified → Delivered) at the execution end. It is to execution what the Product Framework is to specification: one opinionated, conformant way to satisfy the foundation.

---

## Where this sits in the chain

```
ai-development-foundations  (the Two Pillars: Specification Framework · Execution Contract)
        │  defines the standard — shapes and rules, not any product
        ├────────────────────────────────────────────────┐
        ▼                                                 ▼
Product Framework                                The Spark Execution Framework
(conformant SPEC instantiation:                 (conformant EXECUTION instantiation:
 What / How / Delivery)                           how work units actually run)
        │                                                 │
        │  emits work units (What/How by pointer)         │  consumes work units, runs them,
        └──────────────►  The Work-Unit Interface  ◄──────┘  emits verdict events
                         (the seam: WorkUnit out, VerdictEvent in)
```

The foundation defines the standard. The Product Framework is one conformant way to *specify* software for it; this is one conformant way to *execute* what that specification produces. The two meet only at the Work-Unit Interface — neither contains the other.

> **Note on the foundation source.** All three foundation documents — the Execution Contract, the Specification Framework, and the Two Pillars — have now been read in full from the published `ai-development-foundations` text and are the authority for every claim below. All eight Execution-Contract building blocks are discharged against the read contract. The binding-homogeneity invariant is grounded in *both* the Execution Contract's Rule 1 and the Specification Framework's Model axis. The Two Pillars names the **Assembly Line Protocol** as the second-pillar framework-implementation exemplar and mandates **producer-owns, specification-stewarded** ownership of the work-unit interface; this framework is an ALP-class implementation and conforms to that ownership rule.

---

## Premise

The Product Framework specifies *what is true and what must hold*; it deliberately says nothing about the machine that makes it true. Its conformance ladder names execution obligations — work units that reference What and How, verifications that gate acceptance, a computed "done" — without prescribing how any of them run. This framework is the missing half: the execution substrate, control model, and scheduler that **discharge those obligations** on real, bounded hardware.

It is bound to one hardware reality (a single bandwidth-bound box, the NVIDIA DGX Spark) but is written so the *conformance* it provides is hardware-independent: a different executor honouring the same Execution Contract and the same Work-Unit Interface would satisfy the same ladder rungs. The Spark specifics are an implementation, not the standard.

---

## Goal

Discharge every execution-side obligation of the Product Framework's conformance ladder, with the trust properties the Execution Contract requires (frozen input, computed verification, declared transition), at the highest sustained utilisation the hardware allows — without ever trading a trust property for throughput.

---

## The two components this framework is built from

This framework is the composition of two already-specified pieces, plus the conformance argument that binds them to the foundation and the Product Framework.

1. **The Spark Execution Substrate** — the control model and scheduler. A developer-thrown switch over two mutually-exclusive box configurations (QUEUE, EXPLORER), with OFF-BOX as the always-available third path; inside QUEUE, the autonomous machinery: work-unit priority set, sealed binding-homogeneous cell interiors, batched cell-frontier execution, verdict rollup, unit-atomic escalation, and the binding-homogeneity invariant as a pre-dispatch conformance check.

2. **The Work-Unit Interface** — the seam. Two schemas (WorkUnit out, VerdictEvent in), two emit-points, no shared runtime surface. Verdict-as-event: the executor emits and is done, knowing no consumer.

This document does not restate their internals; it cites them and argues their conformance.

---

## Conformance to the Execution Contract (the foundation)

The Execution Contract requires **eight building blocks** in three groups, and its closure test is explicit: a framework that omits any of the eight "is running unsupervised." Conformance therefore means discharging **all eight** — not most — with a named mechanism, plus the funnel (Rule 1). This framework is an **Assembly-Line-Protocol-class framework implementation**: it sits on the Execution Contract exactly as the ALP does, and as Decision-Driven Design sits on the Specification Framework.

### Group 1 — the actors (who acts, who judges)

| Block | Completeness requirement | Discharged by |
| --- | --- | --- |
| **Workers** (a role, not a model) | Every executing role declared; its responsibilities and its permitted/forbidden surface unambiguous. | The executor binds a Worker-role to a resident model **binding** (SPMC Model axis: identity, served quantization, invocation params — not a name). The same role fills at different bindings as a unit escalates. The binding is a property of the work-unit (the binding-homogeneity invariant), not a loose capability label. A role's permitted/forbidden surface is its capability set (below), declared per role, never inherited from "whatever the model can do." |
| **Verification** (declared verifier, declared verdict) | Who validates each output, with what authority, producing a verdict in a declared vocabulary; no output accepted without one. | The executor walks the sealed cell-DAG; each cell-gate is a verification against a **protected oracle the worker cannot write** (ADR-076). Cell-verdicts reduce to a unit-verdict in the declared vocabulary — *accepted / rejected / escalate* — never a boolean in an exit code. Done is computed, not claimed (ADR-071). |

### Group 2 — the grants (what a worker is permitted, and where)

| Block | Completeness requirement | Discharged by |
| --- | --- | --- |
| **Capabilities** (read / write / call / spawn floor) | Every action category a worker may take declared explicitly; held exactly to task need; nothing implicit. | Each Worker-role declares its capability set over the floor. A coder cell-worker holds **read** (its frozen bundle), **write** (its declared output workspace), and **call** (its declared effect tools); it does **not** hold **spawn**. The executor itself holds **spawn** (it dispatches cell-workers) but not **write** to source. Excess capability is a decomposition signal (Rule 1). |
| **Environment** (boundary by construction, not instruction) | The execution boundary declared per worker: reachable filesystem, permitted network, host/peer reachability, what is destroyed at unit end — enforced structurally. | **Per-work-unit ephemeral isolation.** Each admitted work-unit runs in its own sandbox: a private workspace mounted writable, the frozen bundle mounted read-only, **no network except declared destinations**, no reach to the host or to other in-flight units, and the whole sandbox **destroyed when the unit reaches a verdict**. The sealed cell-DAG lives entirely inside one such boundary — this is *why* the cell interior is sealed: it is one isolation domain. On a single shared box running many units batched against one resident model, this per-unit sandbox is the structural separation the contract demands; the model is shared, but each unit's effects, filesystem, and credentials are not. |
| **Tools** (effect vs. knowledge, classified) | Every callable tool declared, classified, scoped to workers and conditions. | The executor declares each cell-invokable tool as **effect** or **knowledge**. **Effect tools** (file write, service call, git mutate) are **permitted during execution**, at the cell where the unit takes effect — these are the cell-DAG's leaf actions. **Knowledge tools** (lookup, retrieval) **may not fire mid-execution**: all knowledge entered at the frozen input boundary, with provenance, when `product dispatch` resolved the bundle. This preserves the worker-as-function-of-input property: effects change the world and are audited; knowledge is frozen so behaviour is reproducible. |
| **Credentials** (work-unit-scoped, short-lived, never standing) | Access to external systems scoped to the work-unit, least-privilege, time-bounded, revoked at unit end; the worker holds a reference, not a standing secret. | The WorkUnit carries a **credential-grant reference**, not a secret. At admission the isolation boundary exchanges that reference for short-lived, least-privilege credentials scoped to the unit's declared effect destinations only, and **revokes them when the sandbox is destroyed at verdict**. The executor never injects a standing credential; authority derives from the unit being executed, not from the executor's identity. A unit that needs no effect destination gets no credential. |

### Group 3 — the flows (in, out, and how work moves)

| Block | Completeness requirement | Discharged by |
| --- | --- | --- |
| **Input Contract** (frozen, bounded) | Each worker's input declared, frozen and bounded at execution start; behaviour a function of input alone; nothing acquired through an undeclared channel after start. | The WorkUnit's `spmc-bundle`, resolved from What/How pointers and frozen by `product dispatch`, carried **by value**, identified by `bundle-hash`, mounted read-only in the sandbox. The executor resolves nothing live; no knowledge tool fires mid-run. |
| **Output Contract** (declared shape + destination) | Each worker's output declared, with where it lands and how it reaches the next stage; state crosses explicitly, no hidden channel. | The VerdictEvent schema (shape) emitted to the event stream (destination). Produced artifacts return **per the unit's declared `artifact-delivery` mode** — `inline` by value in `cell-results`, or `workspace` written to the workspace the WorkUnit declared (and *only* there) with path+ref in the event — each carrying a **`content-hash`** in either mode, so an accepted artifact is attributable regardless of transport. State crosses only via these declared channels; the per-unit sandbox guarantees there is no shared mutable store through which two units could exchange state off-contract. |
| **Transition Contract** (verdict → consequence; the autonomy level) | Every verdict bound to a consequence — advance / halt / retry / escalate — with the human-free bindings declared. The autonomy level *is* this contract's shape. | `next-consequence` on the VerdictEvent: advance / halt / retry / escalate. Escalation re-enqueues the whole unit one tier up (deferral across the developer switch when the next tier is night-only). The human-free set lives in `acceptance-class`: `auto-commit-if-green` is a verdict→advance binding that proceeds without a human (Level 5 for that unit kind); `needs-verdict` binds *accepted* to escalate-to-human (Level 4). The autonomy level is thus per-unit-kind, declared, not a global setting. |

### The funnel (Rule 1)

| Principle | Discharged by |
| --- | --- |
| **Rule 1 — capability falls as constraint density rises** | The binding-homogeneity invariant *is* the funnel at the work-unit scale — and it is stated by **both** foundations: the Execution Contract's Rule 1 (capability falls as constraint density rises) and the Specification Framework's Model axis ("capability requirement is a consequence of spec completeness, not a fixed property of the task"). A unit whose cells span bindings (or a worker needing broad capability) is an under-decomposition defect, surfaced before dispatch, fixed upstream — never worked around by granting more capability or a bigger model. |

### Closure trace

Running the contract's own end-to-end test on this framework: a **cell-worker** (Worker), holding declared **read/write/call capabilities** (Capabilities), is dispatched into a **per-unit ephemeral sandbox** (Environment), able to invoke declared **effect tools** (Tools) using **unit-scoped credentials exchanged from a grant-reference** (Credentials). It receives the **frozen by-value bundle** (Input Contract), does its work, and writes to its **declared output workspace** while the executor emits a **VerdictEvent** (Output Contract). Each cell-gate is a **verification against a protected oracle** (Verification), reduced to a unit-verdict. **`next-consequence`** binds that verdict to advance / halt / retry / escalate (Transition Contract). Every reference resolves; every grant traces to a task need; every output is verified; every verdict has a consequence. **The system is closed — all eight blocks discharged.**

---

## Supporting the Product Framework's conformance ladder

The Product Framework's ladder is cumulative; an instance claims the highest rung it satisfies. Each rung names an obligation; this framework supplies the execution-side machinery that lets an instance *reach* it. The framework does not grant conformance — the instance's own artifacts do — but without an execution substrate, rungs 2–4 cannot be operationally satisfied at all.

**1. Described** — a machine-readable What exists; interesting behaviour has a Decider, simulated sound and complete before any code.
*Execution-side role: none.* Described is pure specification; the box is not involved. This framework respects that boundary — nothing here runs before a What is described. (This is also why the EXPLORER mode's output is a *discovery record*, not code: exploration feeds the Described level, it does not skip it.)

**2. Realised** — a conformant How exists, including a repository layout model; **work units reference the What and How by pointer.**
*Execution-side role: the pointer-resolution boundary.* The Product Framework requires the work-unit to reference What/How *by pointer* in the spec graph. The Work-Unit Interface requires the bundle to cross *by value*, fully frozen. These are reconciled at exactly one place: **`product dispatch` resolves the pointers into a frozen by-value bundle.** Pointer in the graph (Realised conformance preserved); value on the wire (Input Contract satisfied). The dispatch step *is* the boundary between the pointer-world of specification and the frozen-world of execution. The `bundle-hash` then ties the executed value back to the pointed-to spec, so the reference survives as provenance.

**3. Verified** — verifications of all required kinds exist, meet the coherence bar, gate acceptance, and back the rationale trace.
*Execution-side role: the verification engine.* The executor *is* where verifications gate acceptance: each cell's gate is a verification, the unit-verdict is their reduction, and an `accepted` verdict is what acceptance is gated on. The VerdictEvent's `cell-results` array is the execution-side contribution to the rationale trace — every cell-gate outcome, attributable to the frozen `bundle-hash`. The protected, worker-non-writable oracle is what makes these verifications *trustworthy* rather than self-reported, satisfying the coherence bar's intent.

**4. Delivered** — features and releases are graph partitions; **"done" is a computed predicate.**
*Execution-side role: the predicate's computation.* "Done is computed, not claimed" is discharged by the two reductions: cell-verdicts → unit-done (inside the executor), unit-done → deliverable-done (at the spec graph, by a consumer reconciling VerdictEvents). The executor computes the leaf and the unit predicate; the spec side computes the partition rollup. No human asserts done; the verdict events compute it. The `acceptance-class` decides only whether a *computed-green* unit auto-commits or waits for human acceptance — it never lets a human *declare* green.

---

## The conformance argument, stated plainly

An instance using the Product Framework for specification and this framework for execution can claim:

- **Realised**, because dispatch resolves What/How pointers into frozen bundles without losing the reference (it becomes `bundle-hash`).
- **Verified**, because the executor's cell-gates are the verifications that gate acceptance, computed against a protected oracle, traced in `cell-results`.
- **Delivered**, because "done" is the computed reduction of verdict events, never a human claim.

Described is the instance's own to satisfy before anything is dispatched; this framework's contribution begins at Realised and runs through Delivered.

---

## Success criteria

1. Every Execution Contract construct maps to a live mechanism (the conformance table holds, not just on paper but in the running executor).
2. An instance specified under the Product Framework can reach Delivered using only this framework's execution machinery — no obligation in rungs 2–4 is left without a discharging mechanism.
3. The pointer-to-value resolution happens exactly once, at dispatch, and the reference survives as `bundle-hash` for provenance.
4. No trust property (frozen input, protected oracle, computed done) is ever weakened to raise utilisation.
5. The framework's conformance claim is hardware-independent: swapping the Spark for another Execution-Contract-honouring executor changes performance, not which ladder rungs are reachable.
6. Utilisation is sustained (the work-unit set stays non-empty via slice→unit fan-out; the executor always has an unblocked cell frontier or a discovery record in flight).

---

## Excludes

- **The Described level's machinery.** Deciders, simulation, sound-and-complete checks are specification-side; this framework begins at dispatch.
- **What/How authoring.** Owned by the Product Framework and product-cli; this framework consumes their output, never produces it.
- **The substrate's and interface's internals.** Cited, not restated. They version independently.
- **Any claim to grant conformance.** Conformance is the instance's, evidenced by its artifacts; this framework supplies the machinery that makes the higher rungs operationally reachable.

---

## Acknowledges

- **The §-level cross-references into `docs/product-framework.md`** (e.g. the Decider at §3.3, data conformance at §13) are named from the Product Framework README's structure and should be verified against the full spec text on reconciliation; the three foundation documents themselves have been read in full and need no such caveat.
- **"Tier" was corrected to "Model binding."** An earlier draft stated the homogeneity invariant over a capability *tier* label. The Specification Framework's Model axis requires pinning the binding (identity, served quantization, invocation params), and on the Spark quantization is load-bearing for fit — so a tier label is a confound. The invariant is now stated over the binding; `tier` survives only as a routing handle.
- **The pointer/value tension is real and resolved by location, not compromise.** Realised wants pointers; the Input Contract wants frozen values. Putting the resolution at dispatch satisfies both; neither framework bends.
- **EXPLORER output is pre-Described.** It produces discovery records that feed the Described level; it never lets exploratory work enter at Realised or above without passing through specification. This keeps the ladder's first rung load-bearing.
- **A degraded sovereign mode** (local large-coder substituting for off-box authoring/exploration under an offline constraint) remains a separate spec; it affects which models are resident, not which ladder rungs are reachable.

---

## References

- **[ai-development-contracts](https://github.com/Hafeok/ai-development-contracts)** — the shared wire schemas (WorkUnit, VerdictEvent) this framework consumes, with normative machine-checkable `schemas/`. Consumer-side conformance is declared in [`conformance/contracts-conformance.md`](../conformance/contracts-conformance.md).
- **ai-development-foundations / Two Pillars** — Specification Framework; Execution Contract (Worker, Capability, Input/Output/Transition Contracts, Verification, Tools, Rule 1 — the funnel); peers, swappable independently, stability flows downward.
- **Product Framework** — What / How / Delivery; the symmetry of Decider and UI step; the conformance ladder (Described → Realised → Verified → Delivered); "done" as a computed predicate; work units reference What/How by pointer; a conformant instantiation of the Two Pillars.
- **[The Substrate](02-substrate.md)** — the control model (developer switch over QUEUE / EXPLORER / OFF-BOX), the queue's owned data model (WorkUnit atom, sealed binding-homogeneous cell interior), and the binding-homogeneity invariant.
- **[The Work-Unit Interface](../interface/work-unit-interface.md)** — the seam: WorkUnit (by value, hash-identified) out, VerdictEvent (self-describing, fire-and-forget) in; the `dispatch`/executor cut.
- **product-cli** — the specification-pillar implementation that emits work units; `build` split into `dispatch` + executor; computed-done (ADR-071); protected oracle (ADR-076); test-first work units (ADR-075).
