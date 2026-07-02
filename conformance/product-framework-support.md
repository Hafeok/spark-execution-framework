# Conformance Checklist — Product Framework Support

> The Product Framework's conformance ladder (Described → Realised → Verified → Delivered) restated as the execution-side obligations Kiln supplies the machinery for. Kiln does not *grant* conformance — an instance's own artifacts do — but rungs 2–4 cannot be operationally satisfied without an execution substrate. Each item names what Kiln provides toward the rung. See [`docs/01-execution-framework.md`](../docs/01-execution-framework.md) for the full argument.

The ladder is cumulative; an instance claims the highest rung it satisfies.

---

## 1 — Described

*A machine-readable What exists; interesting behaviour has a Decider, simulated sound and complete before any code.*

- [ ] **No execution-side obligation.** Described is pure specification; the box is not involved.
- [ ] The framework does not run anything before a What is described — EXPLORER output is a *discovery record* feeding the Described level, never code that skips it.

## 2 — Realised

*A conformant How exists, including a repository layout model; work units reference the What and How **by pointer.***

- [ ] The pointer/value tension is resolved at exactly one place: **`product dispatch` resolves What/How pointers into a frozen, by-value bundle.**
- [ ] Pointer in the spec graph (Realised preserved); value on the wire (Input Contract satisfied).
- [ ] The reference survives the resolution as `bundle-hash`, tying the executed value back to the pointed-to spec for provenance.

## 3 — Verified

*Verifications of all required kinds exist, meet the coherence bar, gate acceptance, and back the rationale trace.*

- [ ] The executor is where verifications gate acceptance: each cell-gate **is** a verification; the unit-verdict is their reduction.
- [ ] An `accepted` verdict is what acceptance is gated on — nothing flows on an unverified result.
- [ ] The VerdictEvent's `cell-results` array is the execution-side contribution to the **rationale trace** — every cell-gate outcome, attributable to the frozen `bundle-hash`.
- [ ] The protected, worker-non-writable oracle is what makes these verifications trustworthy rather than self-reported (the coherence bar's intent).

## 4 — Delivered

*Features and releases are graph partitions; "done" is a **computed predicate.***

- [ ] "Done is computed, not claimed" is discharged by two reductions: cell-verdicts → unit-done (in the executor), unit-done → deliverable-done (at the spec graph, by a consumer reconciling VerdictEvents).
- [ ] No human asserts done; the verdict events compute it.
- [ ] `acceptance-class` decides only whether a computed-green unit auto-commits or waits for human acceptance — it never lets a human *declare* green.

---

## The claim, assembled

An instance using the Product Framework for specification and Kiln for execution can claim:

- [ ] **Realised** — dispatch resolves What/How pointers into frozen bundles without losing the reference.
- [ ] **Verified** — cell-gates are the acceptance-gating verifications, computed against a protected oracle, traced in `cell-results`.
- [ ] **Delivered** — "done" is the computed reduction of verdict events, never a human claim.

Described is the instance's own to satisfy before dispatch; Kiln's contribution begins at Realised and runs through Delivered.

---

*Caveat: the §-level references into `docs/product-framework.md` (Decider, data conformance) are named from the Product Framework README's structure and should be verified against the full spec text. The three foundation documents underlying these claims have been read in full.*
