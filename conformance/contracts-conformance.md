# Conformance Declaration — AI Development Contracts (consumer side)

> This framework's declaration against the [AI Development Contracts](https://github.com/Hafeok/ai-development-contracts) tier — the shared wire schemas that cross the specification ↔ execution seam. It composes with this framework's foundation conformance ([`execution-contract-checklist.md`](execution-contract-checklist.md)): foundation conformance says the framework honors the pillar *definitions*; this says it speaks the concrete *wire schemas*.

| | |
|---|---|
| **Side** | Consumer (execution framework — accepts WorkUnits, emits VerdictEvents) |
| **Contracts version checked against** | `0.1.0` |
| **foundation_ref** | `main` |
| **Autonomy level** | **Per-unit-kind, declared in `acceptance-class`** (see below) |
| **Source of truth** | [`contracts/work-unit.md`](https://github.com/Hafeok/ai-development-contracts/blob/main/contracts/work-unit.md), [`contracts/verdict-event.md`](https://github.com/Hafeok/ai-development-contracts/blob/main/contracts/verdict-event.md), [`schemas/`](https://github.com/Hafeok/ai-development-contracts/tree/main/schemas). Where this framework and a contract differ, the contract governs. |

The checkable obligations below track [`conformance/consumer-conformance.md`](https://github.com/Hafeok/ai-development-contracts/blob/main/conformance/consumer-conformance.md) item for item.

---

## Accepting WorkUnits

- [x] **Accepts any well-formed WorkUnit, regardless of producer.** The Work-Unit Interface is producer-owned, specification-stewarded; the executor conforms to the published shape and does not negotiate it. — [`interface/work-unit-interface.md` § Source of truth](../interface/work-unit-interface.md#source-of-truth-the-shared-contracts-layer)
- [x] **Validates against the normative schema + the cross-unit-edge structural check, and rejects non-conforming units.** Every incoming unit is validated against `work-unit.schema.json` / the SHACL shapes for the shared encoding, plus the two harness-enforced invariants — no cross-unit `requires` edge, and every `context-refs` id resolving in the unit's own `context-pool` — before admission. — [`interface/work-unit-interface.md` § Validation before admission](../interface/work-unit-interface.md#schema-1--workunit-specification--execution)
- [x] **Resolves nothing by calling back into the producer.** Given a WorkUnit, the executor runs to a verdict with no further input (Input Contract): the `spmc-bundle` is frozen and mounted read-only; no knowledge tool fires mid-run. — [`docs/01-execution-framework.md` § Input Contract](../docs/01-execution-framework.md)
- [x] **Treats `unit-ref` and `parent-deliverable` as opaque** — labels echoed on the verdict, never dereferenced into the producer's graph.
- [x] **Matches `tier` to its resident model / mode; reads `ladder-position` to pick the tier but does not own the ladder.** The producer advances the rung on re-dispatch; the executor never advances it. — [`docs/02-substrate.md`](../docs/02-substrate.md)
- [x] **Walks `cell-graph` as a sealed interior.** The cell-DAG lives inside one isolation domain; the executor expects no cross-unit edge and requires none.
- [x] **Does not require an executor-specific WorkUnit format.** The Spark's Execution-Contract needs (`environment`, `credential-grant`, `tool-grants`) are carried as **optional additions the contract's interiors permit** — read when present, defaulted to a conformant floor (deny-all network, ephemeral workspace, no credential, knowledge-only tools) when absent. A bare contract WorkUnit runs unchanged. — [`interface/work-unit-interface.md` § Schema 1](../interface/work-unit-interface.md#schema-1--workunit-specification--execution)

## Emitting VerdictEvents

- [x] **Every reached verdict is emitted as a VerdictEvent carrying all schema fields.** — [`interface/work-unit-interface.md` § Schema 2](../interface/work-unit-interface.md#schema-2--verdictevent-execution--stream)
- [x] **Artifacts returned per the unit's declared `artifact-delivery` mode**, with a `content-hash` in either mode: `inline` puts the body in `cell-results[].artifact.delivery.inline`; `workspace` returns `path` + `ref`. — [`interface/work-unit-interface.md` § Artifact return](../interface/work-unit-interface.md#schema-2--verdictevent-execution--stream)
- [x] **Writes artifacts only to the workspace the WorkUnit declared**, never an undeclared location — enforced by the per-unit sandbox, whose only writable mount is the declared workspace. — [`docs/01-execution-framework.md` § Output Contract](../docs/01-execution-framework.md)
- [x] **The event is self-describing** — `unit-ref`, `parent-deliverable`, `bundle-hash` all echoed, reconcilable by a consumer that never saw the dispatch.
- [x] **`bundle-hash` echoes the exact frozen bundle the verdict ran against** — provenance is reproducible.
- [x] **`verdict` maps onto `accepted | rejected | escalate`** — a declared artifact, not a boolean exit code. Richer internal verdicts, if any, map onto this enum before crossing.
- [x] **`next-consequence` maps onto `advance | halt | retry | escalate`.**
- [x] **Emit is fire-and-forget** — the executor does not wait, retry, or hold knowledge of any consumer.
- [x] **The executor never re-dispatches itself** — it emits the verdict that prompts a consumer to.

## Independence

- [x] **Replacing hardware or scheduler requires no change to the producer.** The seam versions on its own axis; the Spark substrate (developer switch, QUEUE/EXPLORER modes, GB10 residency) is entirely below the contract. — [`docs/00-overview.md`](../docs/00-overview.md)

---

## Autonomy level

Autonomy follows from **how `next-consequence` values may fire without human involvement**, and on this framework that is **not global — it is declared per unit kind in `acceptance-class`:**

- `auto-commit-if-green` — an `accepted` verdict binds to `advance` **without a human**. For that unit kind the loop closes autonomously (Level 5).
- `needs-verdict` — an `accepted` verdict is surfaced for human acceptance before it advances (Level 4).

The executor emits the identical VerdictEvent either way; the acceptance-class binding is honored by the **consumer**, not the executor. The autonomy claim is therefore a property of each unit kind's declared binding, not a single setting of the box. — [`docs/01-execution-framework.md` § Transition Contract](../docs/01-execution-framework.md).

---

## The pairing guarantee, as it applies here

Because this framework passes the consumer checklist and any producer passing the producer checklist emits WorkUnits it accepts, the two interoperate **without either having been built against the other** — the swappability the contracts tier exists to provide. Swap product-cli for a different conforming specification framework and this executor is unchanged; swap the Spark for a different conforming executor and the producer is unchanged.
