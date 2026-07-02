# The Reference Substrate: DGX Spark

> Kiln's reference substrate: a single bandwidth-bound box, owned by a developer-thrown switch, running either a batched execution queue or a local explorer — never both. The queue's owned data model and the governing invariant specified here are Kiln-normative and substrate-neutral; the residency constraints that motivate the two firings are the Spark's.

---

## Where this sits

This is a **framework implementation** in the sense of the Two Pillars: it depends on the **Execution Contract** (pillar two foundation) and consumes specifications produced under the **Specification Framework** (pillar one). It does not redefine either. Every term used here that is already normative — Worker, Capability, Input Contract, Verification, Transition Contract, SPMC bundle, the funnel — is used in its existing sense and cited, not re-coined.

The substrate is a property of one machine (an NVIDIA DGX Spark: 128 GB unified memory, ~273 GB/s bandwidth) and the control structure that decides what that machine is doing right now.

---

## Premise

The box is a single resource with a hardware shape that forces the architecture. It is strong at exactly one thing — **batched inference of small/medium models**, where many concurrent executions share a resident model — and weak at its opposite: a single large model decoding serially is bandwidth-bound to roughly 33 tok/s, the box's worst regime. Two workloads want the box in mutually exclusive configurations:

- **Batched execution** wants many small models resident, dense frontier, high concurrency.
- **Local exploration** wants one large model resident, deep and serial, latency-tolerant.

They cannot co-reside in VRAM. Therefore the box has a **mode**, and the mode is set externally — by a developer, not inferred by the box. This is the standing principle that something outside the box sets what it is for; the most reliable external something is a deliberate human switch.

A third path is always available and is **not** a box mode: **off-box** frontier inference (Opus-class) for define-step authoring and for any exploration not bound to the box by locality. The box competes with off-box only where locality, cost-at-volume, or offline operation make local execution a requirement rather than a preference.

---

## Goal

Maximize useful utilization of the box without sacrificing the trust properties the Execution Contract requires (frozen input, computed verification, declared transition). Utilization is a property of *work waiting and being drained densely*, not of keeping any one model warm. The slice→work-unit fan-out is the gear ratio that lets a low-rate authoring process keep a high-volume execution queue saturated.

---

## The two-level control model

The system is two nested control loops with different change rates. Mixing them is the failure mode this structure exists to prevent.

**Top level — the developer switch (human-rate, rare, declarative).**
A developer throws a switch selecting the box's whole configuration: `QUEUE` or `EXPLORER`. Mutually exclusive in VRAM. The mode-controller is a switch *actuator* — it enforces the exclusion and loads the chosen configuration. It does not read state and decide; it executes a declared intent. Flips are expensive (unload/load residency) and therefore rare and deliberate.

**Bottom level — the queue's autonomous machinery (machine-rate, continuous).**
Once the box is in `QUEUE`, everything below runs without a human in the loop: work-unit admission, cell-DAG execution, batching, verdict rollup, escalation. None of this dynamism escapes upward to touch the box configuration; the box configuration never reaches down to micromanage the queue.

```
┌─ DEVELOPER SWITCH  (human-rate · rare · declarative) ───────────────┐
│                                                                     │
│   box configuration:  QUEUE   |   EXPLORER     (exclusive in VRAM)  │
│                                                                     │
│   EXPLORER:                                                         │
│     large coder resident · OpenCode agentic loop · serial · slow-ok │
│     produces a DISCOVERY RECORD (candidate decomposition)           │
│     output is NOT accepted code; re-enters only via a flip to QUEUE │
│                                                                     │
│   QUEUE:                                                            │
│   ┌─ autonomous machinery  (machine-rate · continuous) ──────────┐ │
│   │  work-unit priority set   (flat · reorderable · independent)  │ │
│   │  executor admits a unit → walks its sealed cell-DAG →         │ │
│   │     batches the unblocked cell frontier on the resident model │ │
│   │  cell verdicts → unit-done → deliverable-done   (rollup)      │ │
│   │  gate-fail → re-enqueue the WHOLE unit one tier up            │ │
│   │  tier-homogeneity check gates units before dispatch           │ │
│   └───────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘

  OFF-BOX  (Opus-class · always available · not a box mode):
     define-step authoring  ·  non-local exploration
```

---

## The three configurations

### QUEUE — the box as a batched execution engine

- **What it makes the box:** small/medium coder models resident, configured for batch throughput. The executor drains the work-unit priority set.
- **What it gives:** the box in its best regime. Batch density is the one thing the bandwidth-bound hardware genuinely excels at. The Execution Contract holds in full — frozen Input Contract per work-unit, computed Verification, declared Transition — so output flows into the graph as *accepted* work with provenance. Utilization is free here: slice→unit fan-out keeps the set non-empty; each unit's unblocked cell frontier keeps the batch dense.
- **What it costs:** it can only run work that already exists as well-formed, tier-homogeneous work-units. It is powerless on novel work — nothing to dispatch until something has done the decomposition.
- **Switch in when:** there is typed structure waiting to be executed cheaply and provably. This is the default working state.

### EXPLORER — the box as a local frontier reasoner

- **What it makes the box:** the largest coder the box can hold resident, driven by an OpenCode agentic loop. Single deep serial session. Slow decode accepted because nothing waits on latency.
- **What it gives:** a *local* explorer for the funnel's wide end — novel work, open problem space, no typed path yet. Its product is **not shippable code**; it is a **discovery record**: a candidate slice with proposed work-unit boundaries and cells. This is the only sanctioned channel by which exploratory work enters engineered mode.
- **What it costs:** the box's worst regime — serial, unbatchable, slow. Output is *unaccepted by construction*: an agent claiming done inside its own writable session, with no computed verdict and no protected oracle. Must be treated as candidate structure, not artifact. Monopolizes VRAM — QUEUE is not running while EXPLORER is.
- **Switch in when:** there is no typed path, the work is genuinely novel, **and** it is bound to the box by locality, cost-at-volume, or offline operation. Novel-but-not-local exploration belongs off-box.

### OFF-BOX — define and non-local explore (Opus-class)

- **Not a box mode.** Named because "switch between configurations" includes switching off the box entirely.
- **What it gives:** the right hardware for reasoning-heavy, latency-visible, big-context work — define-step authoring (What→How) and any exploration not pinned local. Off-box is strictly better at this profile than a Spark-resident large model; the box is worst at it.
- **Switch here when:** the work is reasoning-heavy and not bound to the box by locality, cost, or offline need.

---

## The switch decision tree

```
1.  Is there typed structure to execute?
        yes → QUEUE   (default; the box earns its keep)
        no  → 2

2.  Is the work novel (no typed path)?
        yes → exploration needed → 3

3.  Must exploration be local? (sensitive data · cost-prohibitive at volume · offline)
        yes → EXPLORER  (on the box)
        no  → OFF-BOX explore  (Opus-class)

4.  Exploration produced a discovery record
        → decompose into tier-homogeneous work-units
        → flip to QUEUE → dispatch → batched execution → computed-done → graph
```

The maturation loop crosses the top-level switch exactly once per cycle. The flip *is* the boundary between exploratory and engineered mode — which is correct, because the discovery record is the only channel between them.

---

## The queue's owned data model

The queue owns **work-units**. It does not own, see, reorder, or reason about cells. A work-unit carries an SPMC bundle (one bundle per discrete work-unit, per the Specification Framework); its cells are the execution primitive beneath that bundle and live entirely inside the unit.

```
WorkUnit                       — the schedulable, reorderable, escalatable atom
  ├─ identity / lineage        unit-ref, parent-deliverable
  ├─ frozen input              SPMC bundle (Input Contract: frozen, bounded)
  ├─ tier                      single model-tier for the whole unit  (see invariant)
  ├─ mode-tag                  PROJECTION of tier through the residency schedule
  ├─ acceptance-class          auto-commit-if-green | needs-verdict
  ├─ ladder-position           current rung on the worker-role escalation ladder
  ├─ state                     queued | running | done | failed | escalated
  └─ cell-graph                SEALED interior — the executor's concern, not the queue's
```

Three structural facts follow from work-unit independence and seal the design:

1. **Dependencies live only inside a unit.** Units are mutually independent, so the queue is a **flat priority set with no edges** — reordering is unconstrained, any prioritization policy is legal. No partial-order machinery at the queue level.

2. **The cell-DAG is local and bounded.** Inside one admitted unit, the executor runs the unblocked cell frontier, gates, advances. The test-before-implement pattern is a *within-unit* edge — the canonical intra-unit dependency, and (per the invariant) tier-homogeneous, because writing a test from a frozen spec and implementing against it are the same constraint-density class.

3. **Done aggregates through two reductions in two places.** Cell verdicts reduce to unit-done **inside the executor**. Unit-done reduces to deliverable-done **at the graph level**. The queue observes only unit-level completion. "Done is computed, not claimed" holds at the unit boundary.

---

## The binding-homogeneity invariant

> A work-unit is well-formed iff all of its cells require the same SPMC **Model binding**.
> `well-formed(unit) ⟺ ∀ cell c ∈ unit.cell-graph : binding(c) = binding(unit)`

The unit is stated over the **Model binding**, not a capability tier — and the distinction is load-bearing on this hardware. The Specification Framework's Model axis is explicit that a model *name* is not a specification: what is pinned is the binding — identity, **served precision/quantization**, and invocation parameters — because "if two things can carry this same label and produce different output, the label is not yet a binding, and the axis is underspecified." On the Spark, quantization is precisely what makes a model fit 128GB, so two cells nominally at the "same tier" but at different quantizations are *different bindings* and a confound. Homogeneity over a loose tier label would let exactly that confound into a unit. So the invariant is over the binding; `tier` survives only as a routing handle on top of it.

This is the funnel principle stated at the work-unit scale, and it is stated by *both* foundations: the Execution Contract's **Rule 1** (constraint density rises toward the action, required capability falls correspondingly) and the Specification Framework's **Model axis** ("when a worker requires a large, expensive model, the signal is that the How Spec has not decomposed far enough; capability requirement is a consequence of spec completeness, not a fixed property of the task"). Binding-heterogeneity across a unit's cells is not a runtime cost to manage — it is a **decomposition defect to detect**, fixed upstream by re-cutting the unit boundary, never by letting the executor straddle bindings.

The invariant does triple duty:

- **Conformance check (pre-dispatch, static).** Each cell's required binding is derivable from its frozen bundle, so a unit can be checked for homogeneity before anything runs — the same family as the existing `graph check` / `gap check` / `drift check`. A heterogeneous unit fails the check the way an untested feature does.
- **Escalation simplifier.** Because a well-formed unit has no cheap cells mixed with hard ones, escalating the *whole unit* to the next binding is lossless — every cell genuinely needed that binding, or the unit was malformed and should have been caught. This is what earns **unit-atomic escalation**: the entire cell-DAG re-runs under the heavier resident binding, so the unit never splits across the day/night residency boundary (which would break both provenance and the cell-DAG's internal dependencies). Escalation is a Transition Contract consequence (`gate-fail → re-enqueue one binding up`); when the next binding is night-only, escalation *is* deferral across the switch. Because the Model axis resets the quality baseline on any binding change, a re-dispatch at a new binding is correctly a new attribution point, not a continuation.
- **Maturation metric.** A heterogeneous unit is the system reporting that the decomposition has not matured. Resolving it mints structure — split along the binding boundary, and one side may become a newly-typed task that descends to a cheaper binding permanently. The heterogeneity rate falling over time *is* architectural maturation measured at the unit grain.

**Consequence for the model:** since all cells share it, `model-binding` is a property of the work-unit, singular and well-defined — not a per-cell tag. `mode-tag` is then not independent data; it is a projection of the binding through the residency schedule (heavy binding → EXPLORER-adjacent night residency; light binding → QUEUE day residency). One less thing to author, one less thing to drift.

**Open verification before freezing as normative:** confirm there is no legitimate work-unit pattern that is intrinsically binding-mixed. The test-first pattern is homogeneous (writing a test from a frozen spec and implementing against it are the same constraint-density class, hence the same binding), which is good evidence the invariant fits the grain you have. If a real mixed pattern exists, the invariant must account for it rather than forbid it.

---

## Authority split (who owns each decision)

| Stage | Owner | Location | Nature |
|---|---|---|---|
| **Define** (What→How) | Opus-class | off-box | reasoning-heavy, latency-visible; the funnel's apex |
| **Explore** (novel work) | OpenCode loop / Opus | box *iff* local-bound, else off-box | produces a discovery record, not accepted code |
| **Dispatch** | mechanical projection | wherever (no model) | freezes SPMC bundle, fans slice→units, sets tier→mode-tag, enqueues; a sensing-to-queue act |
| **Execute** | resident Worker (role) | box, mode-gated | walks the cell-DAG, batches the frontier, gates |
| **Switch** | developer | top level | declares QUEUE or EXPLORER |

Workers are roles, not models (Execution Contract §Workers); a tier is the model-capability binding for a role. The same role fills at different tiers as it escalates.

---

## Success criteria

1. The box is always either draining a non-empty work-unit set (QUEUE) or producing a discovery record (EXPLORER) — idle is a scheduling defect, not a steady state.
2. No work-unit is dispatched that fails the tier-homogeneity check.
3. Every accepted artifact in the graph traces to a frozen SPMC bundle, a computed unit-verdict, and a declared transition (Execution Contract closure holds at the unit boundary).
4. Explorer output never enters the graph as accepted work except by passing through dispatch as a re-decomposed, conformant work-unit.
5. Top-level flips are developer-initiated and rare; no machine-rate process triggers a residency flip.
6. The unit-tier-heterogeneity rate is tracked over time and trends down as the corpus types itself.

---

## Excludes

- **Per-cell escalation across the mode boundary.** Forbidden: it would split a unit's cell-DAG across two residency windows. The unit is the escalation atom.
- **Autonomous mode flipping by the controller.** The controller actuates a developer switch; it does not infer the mode from queue composition.
- **Accepting Explorer output as shippable code.** Explorer produces candidate structure only.
- **Cross-unit cell dependencies.** Dependencies are within-unit only; this is what makes the queue a flat set.
- **Off-box as a fallback substitute under sovereignty/offline requirements** is acknowledged below, not specified here.

---

## Acknowledges

- **Residency flips are expensive on this hardware.** The two-level structure contains that cost by making flips human-rate and rare; all machine-rate dynamism lives below the switch, inside QUEUE.
- **The local Explorer runs the box in its worst regime** (serial, slow, unbatched). This is accepted *only* when locality, cost-at-volume, or offline operation make local exploration a requirement; otherwise exploration is off-box.
- **A degraded sovereign mode** — where the box's large coder substitutes for off-box Opus on define and explore when no external dependency is permitted — is a real requirement when offline/sovereignty is the binding constraint, and needs its own specification (preferred-when-available off-box; local fallback always present).
- **Work-unit boundary-drawing is now a funnel-efficiency decision**, not only a logical grouping: the authoring/define step must cut units so their cells share a tier. Heterogeneous units are a signal to split.

---

## References

- **Specification Framework** — What / How / SPMC; one SPMC bundle per discrete work-unit; derivation contract.
- **Execution Contract** — Workers (role, not model); Capabilities; Input Contract (frozen, bounded); Verification (computed verdict, protected oracle); Transition Contract (verdict→consequence, the autonomy level); Rule 1 (the funnel — capability falls as constraint density rises); Tools (knowledge enters at the frozen boundary).
- **Two Pillars** — framework-implementation layer depends on foundation; stability flows downward.
- **Decision-Driven Design** — the funnel and its temporal companion, maturation; the discovery record as the sole sanctioned channel from exploratory to engineered mode; decisions first-class, execution last-class.
- **product-cli** — slice → work-unit → cell decomposition; `build` (to be split into `dispatch` + executor); computed-done (ADR-071); protected oracle (ADR-076); test-first work units (ADR-075); worker capability catalog and role-binding escalation ladder.
