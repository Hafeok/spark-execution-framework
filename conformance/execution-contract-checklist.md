# Conformance Checklist — Execution Contract

> The Execution Contract's eight building blocks and its closure rule, restated as a checkable list. An implementation of this framework — or a CI gate at dispatch — checks itself against these. A box that fails any item is, in the contract's own words, "running unsupervised." Each item names the discharging mechanism specified in [`docs/01-execution-framework.md`](../docs/01-execution-framework.md).

The contract's test is **closure**: a unit of work traced end to end, nothing dangling. The items below are that trace, decomposed.

---

## Group 1 — the actors

### Workers
- [ ] Every executing role (cell-worker, the executor-as-dispatcher, the verifier) is declared, with an unambiguous permitted/forbidden surface.
- [ ] A role's surface is its declared capability set, never inherited from "whatever the model can do."
- [ ] The role binds to a resident model **binding** (identity + served quantization + invocation params), not a loose tier label.

### Verification
- [ ] Every cell output is judged by a declared verifier before acceptance — no output flows on an unverified result.
- [ ] The verdict uses the declared vocabulary: **accepted / rejected / escalate**. Not a boolean in an exit code.
- [ ] The oracle the verifier checks against is **not writable by the worker** that produced the output (protected oracle).
- [ ] Unit "done" is the computed reduction of cell verdicts — never a worker's self-claim.

## Group 2 — the grants

### Capabilities
- [ ] Each worker holds an explicit set over the **read / write / call / spawn** floor; nothing implicit.
- [ ] A cell-worker holds read (its bundle), write (its declared workspace), call (its declared effect tools); it does **not** hold spawn.
- [ ] Capability rises no higher than the task needs. A worker needing broad capability is flagged as an upstream decomposition defect, not granted.

### Environment
- [ ] Each work-unit runs in a **per-unit ephemeral sandbox**, declared before execution.
- [ ] The frozen bundle is mounted read-only; a private workspace is mounted writable.
- [ ] No network except declared destinations; no reach to the host or to other in-flight units.
- [ ] The sandbox is **destroyed when the unit reaches a verdict.** The boundary is enforced by construction, not by instructing the worker.

### Tools
- [ ] Every cell-invokable tool is declared and classified **effect** or **knowledge**.
- [ ] Effect tools (file write, service call, git mutate) fire only during execution, at the cell where the unit takes effect.
- [ ] Knowledge tools do **not** fire mid-execution — all knowledge entered frozen, with provenance, at dispatch.

### Credentials
- [ ] The work-unit carries a **credential-grant reference**, never a standing secret.
- [ ] The boundary exchanges the reference for short-lived, least-privilege credentials scoped to the unit's declared effect destinations only.
- [ ] Credentials are **revoked when the sandbox is destroyed** at verdict.
- [ ] A unit with no effect destination receives no credential.

## Group 3 — the flows

### Input Contract
- [ ] Each unit's input is the SPMC bundle, **frozen and bounded** at execution start, identified by `bundle-hash`.
- [ ] The bundle is fully resolved — no reference the executor must chase by calling back.
- [ ] The worker's behaviour is a function of this input alone; nothing is acquired through an undeclared channel after start.
- [ ] Each incoming WorkUnit is **validated against the [contracts-layer](https://github.com/Hafeok/ai-development-contracts) normative schema** for the shared encoding, plus the two structural invariants (no cross-unit edge; every `context-refs` id resolves in-unit), and a non-conforming unit is **rejected** before admission. (Consumer-conformance; see [`contracts-conformance.md`](contracts-conformance.md).)

### Output Contract
- [ ] Each unit's output has a declared shape (the VerdictEvent schema; effect-tool writes to the declared workspace) and a declared destination (the event stream; the workspace).
- [ ] State crosses between stages only through declared channels — no shared mutable store two units could exchange state through off-contract.

### Transition Contract
- [ ] Every verdict binds to a declared consequence: **advance / halt / retry / escalate**.
- [ ] The human-free bindings are declared explicitly (in `acceptance-class`): which verdict→consequence pairs may proceed without a human.
- [ ] Escalation re-enqueues the **whole unit** at the next binding (unit-atomic); a new binding is a new attribution point.

## The funnel (Rule 1)

- [ ] No defect is worked around by granting more capability or a bigger model. A worker needing broad capability, or a unit spanning bindings, is surfaced as an upstream under-decomposition defect and fixed there.
- [ ] **Binding-homogeneity** is checked before dispatch: every cell in a unit requires the same Model binding, or the unit fails the check.

## Closure

- [ ] A unit can be traced end to end: worker → environment → capabilities → tools → credentials → frozen input → output → verdict → consequence.
- [ ] Every grant traces to a task need. Every output is verified. Every verdict has a consequence. **Nothing dangles.**

---

*All eight blocks must pass. The contract does not contain a lower tier of rigour — a framework missing any block is not running at a lower level, it is running unsupervised.*
