# The Work-Unit Interface

> The contract between a specification-pillar implementation (product-cli) and an execution-pillar implementation (the Spark substrate). Two data schemas and two emit-points. No shared runtime surface. Either side may be rebuilt from scratch against this contract alone.

---

## Source of truth: the shared contracts layer

The wire-level shape of what crosses this seam is **no longer defined here.** It is published, versioned, and owned on its own tier by the [**AI Development Contracts**](https://github.com/Hafeok/ai-development-contracts) repository — the [WorkUnit](https://github.com/Hafeok/ai-development-contracts/blob/main/contracts/work-unit.md) and [VerdictEvent](https://github.com/Hafeok/ai-development-contracts/blob/main/contracts/verdict-event.md) contracts, with **normative machine-checkable schemas** under [`schemas/`](https://github.com/Hafeok/ai-development-contracts/tree/main/schemas). This framework is checked against **contracts version `0.1.0`** (foundation_ref `main`).

That tier sits *beneath both* frameworks (Stable Dependency Principle): neither the specification framework nor this execution framework contains it, so either can be swapped without touching the other. This document is therefore **the Spark-specific projection** of those contracts — how this executor consumes the published shape and what execution-side detail it reads *within* it — **not** a second, competing definition. Where this doc and the published contract differ, **the contract governs.**

Two consequences the rest of this document honors:

- The Spark executor **conforms to the published shape**; it does not require any producer to emit a Spark-specific WorkUnit format. The Execution-Contract fields this framework needs (`environment`, `credential-grant`, `tool-grants`) are carried as **optional additions the contract's interiors explicitly permit** — read when present, defaulted to a conformant floor when absent — never as fields a producer must supply.
- The executor **validates every incoming WorkUnit** against the contracts-layer normative schema for the shared encoding ([`work-unit.schema.json`](https://github.com/Hafeok/ai-development-contracts/blob/main/schemas/work-unit.schema.json) for JSON, the SHACL shapes for RDF) **plus the two structural invariants the harness enforces** — no cross-unit `requires` edge, and every cell `context-refs` id resolving in the unit's own context pool — and **rejects anything non-conforming** before admission.

The framework's full consumer-side conformance is stated in [`conformance/contracts-conformance.md`](../conformance/contracts-conformance.md).

---

## Where this sits

This is the seam between the two pillars, and it versions on its own axis — independently of either side it connects. The Two Pillars foundation requires that framework implementations be swappable without disturbing the other pillar; this document is what makes that swap mechanical. Replace the Spark substrate with a different executor, or product-cli with a different spec tool, and this contract survives unchanged as long as the replacement honors the two schemas below.

It depends on both foundations and redefines neither:

- From the **Specification Framework**: the work-unit is the floor of specification — *one SPMC bundle per discrete work-unit*.
- From the **Execution Contract**: the Input Contract is a *frozen, bounded bundle* handed to a Worker that is a function of that input alone; the Output Contract has *a declared shape and a declared destination*; Verification produces a *declared verdict*; the Transition Contract binds verdict to consequence.

The work-unit is named by *both* pillars as their edge — the smallest thing specification bottoms out into, and the unit execution consumes. That coincidence-of-boundary is why it is the interface, and not a choice made here for convenience.

**Ownership is producer-owns, specification-stewarded.** The Two Pillars foundation is explicit: the side that freezes and hands over the artifact defines its shape. The **specification pillar owns this interface** — it builds the unit, so it declares the unit's form; execution conforms to it and does not negotiate it. This follows from frozen-input discipline (a unit is a function of its declared input alone, so the producer of that input declares its form). But ownership is *directional, not containing*: the interface is **not an artifact of any one Specification Framework** — if it were, swapping the framework would break every executor. So it versions on its own axis, stewarded by the specification pillar, beholden to neither a single framework nor a single executor. The practical test, met here: swap the executor (different hardware, different scheduler) and the specification side does not change; swap the spec tool and the executor does not change.

---

## Premise: the interface is a data contract, not a call

Loose coupling means the two sides share no live surface — not a store, not an endpoint, not an address. They share exactly two data shapes:

- **WorkUnit** — what the specification side emits.
- **VerdictEvent** — what the execution side emits.

product-cli never invokes the executor. The executor never reaches back into the graph. A work-unit is frozen and emitted; from that moment it is self-contained, and the executor resolves nothing by calling back. This is the frozen-input principle of the Execution Contract applied at the framework boundary: the freeze *is* the decoupling.

The return path is an **event**, which is looser than push or pull. Push would name a consumer (executor knows product-cli's address). Pull would name a poller (product-cli knows the executor's location). An event names **neither**: the executor emits a verdict to a stream and is done, with no knowledge of who consumes it or whether anyone does. product-cli is one subscriber among any number. The producer knowing nothing about consumers is the operational definition of decoupled.

```
product-cli  ──emits──▶  WorkUnit            (by value · bundle inlined · hash = identity)
                              │
                              ▼
                  [ Spark executor: admit → walk sealed cell-DAG → unit-verdict ]
                              │
executor     ──emits──▶  VerdictEvent         (self-describing · carries its own lineage)
                              │
                              ▼
                      [ event stream ]  ◀── product-cli subscribes → reconciles → deliverable-done
                                        ◀── any other interested consumer (metrics, notify, …)
```

---

## The cut that creates the seam

`product build`, as it stands, fuses the two pillars: it dispatches a worker **and** runs gates **and** fixes in place **and** escalates — synchronously, inside the spec tool, reaching into execution. That is a tight coupling across the pillar boundary.

This contract requires `build` to be split at the work-unit:

- **`product dispatch`** — the specification side's last act. Decompose to work-units, assemble and fully resolve the SPMC bundle, run the tier-homogeneity check, freeze, and **emit the WorkUnit**. Involvement ends here until a verdict event returns.
- **executor** — the execution side's whole job. Receive a frozen WorkUnit, schedule it, admit it, walk the sealed cell-DAG, gate, and **emit a VerdictEvent**.

Dispatch is a mechanical projection from the graph — no model required. It is the freeze-and-emit, not a model invocation. The verb change is the cut; the cut is the loose coupling.

---

## Schema 1 — WorkUnit (specification → execution)

Emitted by `product dispatch`. Travels **by value**: the SPMC bundle is serialized into the unit in full, with every reference (ADRs, test criteria, context fragments) resolved and inlined. The executor needs nothing but the unit in hand.

The **normative field set and meaning are the contract's** ([`contracts/work-unit.md`](https://github.com/Hafeok/ai-development-contracts/blob/main/contracts/work-unit.md)). Restated here as the executor reads it, with the Spark-specific additions marked:

```
WorkUnit                             # ── contract envelope (published; the executor accepts as-is) ──
  unit-ref           : string        # opaque id. The executor treats this as a label,
                                      # NOT a pointer into product-cli's graph.
  parent-deliverable : string        # opaque lineage tag. Carried solely so a verdict
                                      # can later be rolled up. Opaque to the executor.
  bundle-hash        : string        # content hash (algo:digest) of the frozen SPMC bundle.
                                      # This IS the work-unit's identity. Present even
                                      # though the bundle travels by value (see note).
  tier               : string        # the single worker-role tier for the whole unit.
                                      # The executor matches it to its resident model / box
                                      # mode. A routing handle OVER the Model binding — the
                                      # binding (spmc-bundle.model) is what determines output
                                      # and is recorded for attribution; tier picks residency.
  acceptance-class   : enum          # auto-commit-if-green | needs-verdict. Tells a CONSUMER
                                      # whether a green verdict may auto-commit. The executor
                                      # echoes it; it does not interpret deliverable meaning.
  ladder-position    : int           # escalation-ladder rung. The executor READS it to pick
                                      # the tier; it does not own the ladder. Advanced by the
                                      # producer on re-dispatch after a rejecting verdict.
  artifact-delivery  : object        # DECLARED transport for produced artifacts. The executor
                                      #   honors the mode the unit declares:
                                      #     mode: inline    → artifacts return by value in the
                                      #                       VerdictEvent cell-results
                                      #     mode: workspace → the executor writes into the
                                      #                       declared workspace (kind,
                                      #                       location, ref — e.g. a local git
                                      #                       clone) and ONLY there; the
                                      #                       VerdictEvent references path+hash
  spmc-bundle        : object        # the frozen Input Contract — FULLY self-contained. No
                                      #   reference the executor must resolve by calling back.
    model            :   object      #   M — ONE fully-pinned binding for the whole unit
                                      #       (tier homogeneity): identity (provider/endpoint,
                                      #       version), served precision/quantization, and
                                      #       invocation params (temperature, top-p, seed where
                                      #       honored). On the Spark, quantization is what makes
                                      #       a model fit 128GB — load-bearing, not a detail:
                                      #       same model name at a different quantization is a
                                      #       DIFFERENT binding. The executor matches this to a
                                      #       resident model / box configuration.
    context-pool     :   object      #   C — content-addressed fragment pool, every fragment
                                      #       inline with an id and provenance; cells select by
                                      #       id. Knowledge entered here, frozen, at dispatch.
  cell-graph         : object        # the SEALED interior DAG. Walked only by the executor;
    cells[]          :   array       #   each cell is ONE LLM call carrying its own S and P:
      id             :     string
      requires       :     string[]  #   intra-unit cell ids only; NO cross-unit edge, ever
      schema         :     object    #   S — shape-language + shape document inline
      prompt         :     object    #   P — prompt content inline, never a callback ref
      context-refs   :     string[]  #   C — fragment ids; every id resolves in this unit
      output         :     object    #   the artifact this cell produces (artifact-id,
                                      #       media-type, path under workspace mode)
      gate           :     object?   #   optional extra gate beyond schema conformance

# ── Spark execution-side additions (OPTIONAL; the contract's interiors permit extra fields) ──
# Read when the producer supplies them; defaulted to a conformant floor when absent. The
# executor NEVER requires a producer to emit these — a bare contract WorkUnit runs unchanged.
  environment        : object?       # the declared execution boundary (Execution Contract:
                                      # Environment). Permitted network destinations, isolation
                                      # guarantees. Default when absent: deny-all network, a
                                      # private ephemeral workspace, destroyed at verdict.
  credential-grant   : string?       # a REFERENCE, never a secret (Execution Contract:
                                      # Credentials). Exchanged at the boundary for short-lived,
                                      # least-privilege credentials, revoked at verdict. Absent ⇒
                                      # the unit gets no credential (no effect destination).
  tool-grants        : array?        # each cell-invokable tool, classified effect|knowledge
                                      # (Execution Contract: Tools). Absent ⇒ knowledge-only:
                                      # no effect tool may fire. Knowledge tools never fire
                                      # mid-run in any case — all knowledge froze at dispatch.
```

**Validation before admission.** The executor validates every incoming WorkUnit against the contracts-layer **normative schema** for the shared encoding (`work-unit.schema.json` / the SHACL shapes) **plus** the two structural invariants the schemas cannot scope — no cross-unit `requires` edge, and every `context-refs` id resolving in this unit's `context-pool` — and **rejects** any unit that fails. A structurally valid but unexecutable unit (a cell missing its prompt or shape, an unresolvable context ref) is rejected, not admitted.

**Constraints the emitter must honor (checked on the specification side, before emit):**

1. **The SPMC bundle is fully resolved.** No element requires a callback into product-cli to read. If the executor would ever need `product context …` mid-run, the unit is not frozen and must not be emitted.
2. **Tier-homogeneity holds.** Every cell in `cell-graph` requires the single `tier` / binding. A heterogeneous unit is a decomposition defect and fails dispatch — this check lives on the **specification side**, alongside `graph check` / `gap check` / `drift check`, because a malformed decomposition is a spec defect, not a runtime condition.
3. **`bundle-hash` is the hash of `spmc-bundle` as emitted.** Identity and body agree at emit time.
4. **Artifact transport is declared.** `artifact-delivery` states `inline` or `workspace`; the seam forbids **undeclared** side channels.

**Note on by-value + hash.** The bundle travels bodily, so a single-box executor needs no shared store. `bundle-hash` is nonetheless always present, because the verdict event needs it and because it is the unit's identity. This makes a future migration to by-reference **additive, not breaking**: move the bundle body into a content-addressed store, leave the hash in the unit, and federation of many executors pulling one frozen bundle becomes possible without a schema change. By-value is correct for v0 and for a single Spark; the hash keeps the door open.

---

## Schema 2 — VerdictEvent (execution → stream)

Emitted by the executor to an event stream when a unit reaches a verdict. **Self-describing**: a consumer that never observed the dispatch can reconcile it from the event alone. The executor emits and is done — it does not wait, retry, or know who listens.

```
VerdictEvent
  event-id           : string        # unique per emitted event.
  emitted-at         : timestamp     # when the executor emitted. (A night verdict may
                                      # sit in the stream until morning reconciliation;
                                      # this is the temporal decoupling working.)
  unit-ref           : string        # echoes the WorkUnit. Lets a consumer match the
                                      # verdict to the unit without shared state.
  parent-deliverable : string        # echoes the WorkUnit lineage tag. Enables a
                                      # consumer to roll the verdict up to a deliverable.
  bundle-hash        : string        # the exact frozen bundle this verdict ran against.
                                      # Closes the loop: verdict is attributable to an
                                      # immutable input. Provenance is reproducible.
  verdict            : enum          # accepted | rejected | escalate
                                      # The Execution Contract verdict vocabulary. Not a
                                      # boolean in an exit code — a declared artifact.
  tier-ran           : string        # the tier the unit actually executed at. Equals the
                                      # WorkUnit tier on a first run; on a re-dispatch it
                                      # records the escalated tier that produced this verdict.
  cell-results       : array         # per-cell results from walking the sealed cell-DAG.
    cell-id          :   string      #   echoes the WorkUnit cell id
    verdict          :   enum        #   accepted | rejected | skipped
    artifact         :   object?     #   the produced artifact, CONTENT-HASH identified in
                                      #     BOTH delivery modes: artifact-id (echoes the cell's
                                      #     declared output), content-hash, and delivery —
                                      #     inline{content} OR workspace{path, ref} — matching
                                      #     the WorkUnit's declared artifact-delivery mode.
    evidence         :   object?     #   gate output / validation report (interior open).
                                      # The executor's evidence for the unit verdict. A consumer
                                      # may act on `verdict` alone; cell-results is present for
                                      # audit, the rationale trace, and heterogeneity metrics.
  next-consequence   : enum          # advance | halt | retry | escalate
                                      # The Transition Contract binding the executor applied
                                      # or recommends. On `escalate`, signals that a
                                      # re-dispatch one tier up is the declared next step —
                                      # which, when the next rung is night-only, is deferral
                                      # across the developer switch.
```

**Consumer obligations (product-cli as the canonical subscriber):**

- **Reconcile, do not poll-and-block.** product-cli observes verdict events and updates its graph: unit-verdict → work-unit state → roll up to deliverable-done. The rollup is the specification side's business; the executor neither performs nor understands it.
- **`escalate` / re-dispatch is a fresh emit.** On a rejecting or escalating verdict whose ladder is not exhausted, product-cli re-runs `dispatch` for the unit with `ladder-position` advanced — producing a new WorkUnit (new run, same `unit-ref`, same `bundle-hash` if the bundle is unchanged). Escalation is unit-atomic: the whole cell-DAG re-runs at the higher tier. The executor never re-dispatches itself; it only emits the verdict that prompts a consumer to.
- **Acceptance-class is honored by the consumer, not the executor.** `auto-commit-if-green` means a consumer may commit on an `accepted` verdict without human review; `needs-verdict` means it surfaces for a human. The executor emitted the same event either way.

**Artifact return follows the unit's declared `artifact-delivery` mode.** Under `inline`, the executor puts the artifact body in the event by value (the seam stays store-free). Under `workspace`, the executor has written into the workspace the WorkUnit *itself* declared — never an undeclared location — and the event carries the resulting `path` and `ref`. In **both** modes every artifact carries a `content-hash`, so an accepted artifact is attributable and verifiable regardless of transport, and a consumer never fetches from a location the unit did not name.

---

## What crosses the seam, exhaustively

Outbound: **WorkUnit (by value).** Inbound: **VerdictEvent (by event).** Nothing else. No live call, no shared store, no shared endpoint. The two sides share two schemas and two emit-points. That is the entire surface area of the coupling.

---

## Success criteria

1. The executor resolves nothing by calling product-cli. Given a WorkUnit, it executes to a verdict with no further input.
2. A VerdictEvent is reconcilable by a consumer that never saw the dispatch — its lineage and `bundle-hash` are sufficient.
3. Replacing the executor (different hardware, different scheduler) requires no change to product-cli, and vice versa, as long as both schemas are honored.
4. Every accepted verdict is attributable to an immutable frozen bundle by `bundle-hash` — provenance is reproducible.
5. The producer (executor) holds no knowledge of any consumer.
6. Temporal decoupling holds: a verdict emitted at night is correctly reconciled whenever a consumer next reads the stream, with no executor-side wait.

---

## Excludes

- **Either side's internals.** How product-cli decomposes a slice, and how the executor schedules or batches, are out of scope — owned by their respective framework-implementation specs.
- **The transport of the event stream.** Whether the stream is a log, a bus, or a file is an implementation choice below this contract; the contract requires only that emit is fire-and-forget and that events are self-describing.
- **By-reference bundle transport.** Acknowledged as a future additive migration (the `bundle-hash` reserves it), not specified here.
- **The developer switch and box modes.** Those are the substrate's concern; this contract is mode-agnostic — a WorkUnit's `tier` is all the executor needs to match residency.

---

## Acknowledges

- **By-value re-sends the bundle on every (re-)dispatch.** Negligible on a single box; the `bundle-hash` keeps the by-reference migration open for federation without a schema break.
- **The tier-homogeneity check sits on the specification side.** It is a spec-side conformance gate run before emit, even though the property it guarantees is what makes execution-side escalation lossless. The check is pillar-one's; the benefit is pillar-two's. That split is intended.
- **`escalate` couples the two pillars only through the verdict vocabulary.** The executor recommends escalation by emitting a verdict; the consumer decides to re-dispatch. Neither owns the whole escalation loop — it closes across the seam, by data, exactly as intended.

---

## References

- **[AI Development Contracts](https://github.com/Hafeok/ai-development-contracts)** — the normative source of truth for the two wire shapes this seam carries: [WorkUnit](https://github.com/Hafeok/ai-development-contracts/blob/main/contracts/work-unit.md), [VerdictEvent](https://github.com/Hafeok/ai-development-contracts/blob/main/contracts/verdict-event.md), and the machine-checkable [`schemas/`](https://github.com/Hafeok/ai-development-contracts/tree/main/schemas). This framework consumes them per the [consumer checklist](https://github.com/Hafeok/ai-development-contracts/blob/main/conformance/consumer-conformance.md); its declaration is [`conformance/contracts-conformance.md`](../conformance/contracts-conformance.md).
- **Specification Framework** — one SPMC bundle per discrete work-unit; the derivation contract; the work-unit as specification's floor.
- **Execution Contract** — Input Contract (frozen, bounded); Output Contract (declared shape and destination); Verification (declared verdict: accepted / rejected / escalate); Transition Contract (verdict → consequence); Tools (knowledge enters at the frozen boundary, never live).
- **Two Pillars** — peers, swappable independently; stability flows downward; the interface between framework implementations must not bind their lifecycles.
- **The Spark Execution Substrate** — the execution-pillar implementation on the consuming end of this contract; the developer switch, QUEUE/EXPLORER modes, and the tier-homogeneity invariant in full.
- **product-cli** — the specification-pillar implementation on the emitting end; slice → work-unit → cell; `build` to be split into `dispatch` + executor; computed-done (ADR-071); protected oracle (ADR-076); test-first work units (ADR-075); worker capability catalog and role-binding escalation ladder.
