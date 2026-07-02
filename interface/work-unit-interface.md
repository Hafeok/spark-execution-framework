# The Work-Unit Interface

> The contract between a specification-pillar implementation (product-cli) and an execution-pillar implementation (Kiln; reference substrate DGX Spark). Two per-run data schemas and two emit-points, plus one out-of-band self-description the executor publishes. No shared runtime surface. Either side may be rebuilt from scratch against this contract alone.

---

## Where this sits

This is the seam between the two pillars, and it versions on its own axis — independently of either side it connects. The Two Pillars foundation requires that framework implementations be swappable without disturbing the other pillar; this document is what makes that swap mechanical. Replace the Spark substrate with a different executor, or product-cli with a different spec tool, and this contract survives unchanged as long as the replacement honors the schemas below (WorkUnit and VerdictEvent per run, CapabilityManifest published).

It depends on both foundations and redefines neither:

- From the **Specification Framework**: the work-unit is the floor of specification — *one SPMC bundle per discrete work-unit*.
- From the **Execution Contract**: the Input Contract is a *frozen, bounded bundle* handed to a Worker that is a function of that input alone; the Output Contract has *a declared shape and a declared destination*; Verification produces a *declared verdict*; the Transition Contract binds verdict to consequence.

The work-unit is named by *both* pillars as their edge — the smallest thing specification bottoms out into, and the unit execution consumes. That coincidence-of-boundary is why it is the interface, and not a choice made here for convenience.

**Ownership is producer-owns, specification-stewarded.** The Two Pillars foundation is explicit: the side that freezes and hands over the artifact defines its shape. The **specification pillar owns this interface** — it builds the unit, so it declares the unit's form; execution conforms to it and does not negotiate it. This follows from frozen-input discipline (a unit is a function of its declared input alone, so the producer of that input declares its form). But ownership is *directional, not containing*: the interface is **not an artifact of any one Specification Framework** — if it were, swapping the framework would break every executor. So it versions on its own axis, stewarded by the specification pillar, beholden to neither a single framework nor a single executor. The practical test, met here: swap the executor (different hardware, different scheduler) and the specification side does not change; swap the spec tool and the executor does not change.

---

## Premise: the interface is a data contract, not a call

Loose coupling means the two sides share no live surface — not a store, not an endpoint, not an address. They share three data shapes, two of which cross **per run** and one **out of band**:

- **WorkUnit** — what the specification side emits, per run.
- **VerdictEvent** — what the execution side emits, per run.
- **CapabilityManifest** — what the execution side *publishes* (not per run): its self-description of what can run here, which the producer matches a unit's requirements against *before* dispatch. It is the one seam artifact that is not producer-authored, because it is the executor's declaration about itself — the complement of producer-owns, not a violation of it.

product-cli never invokes the executor. The executor never reaches back into the graph. A work-unit is frozen and emitted; from that moment it is self-contained, and the executor resolves nothing by calling back. This is the frozen-input principle of the Execution Contract applied at the framework boundary: the freeze *is* the decoupling.

The return path is an **event**, which is looser than push or pull. Push would name a consumer (executor knows product-cli's address). Pull would name a poller (product-cli knows the executor's location). An event names **neither**: the executor emits a verdict to a stream and is done, with no knowledge of who consumes it or whether anyone does. product-cli is one subscriber among any number. The producer knowing nothing about consumers is the operational definition of decoupled.

```
executor     ──publishes──▶  CapabilityManifest   (out of band · what can run here · queryable)
                                   │
product-cli  ──matches unit requirements against manifest BEFORE dispatch──┘

product-cli  ──emits──▶  WorkUnit            (by value · bundle inlined · hash = identity)
                              │
                              ▼
                  [ Kiln executor: admit (authoritative) → walk sealed cell-DAG → unit-verdict ]
                              │           └─ requirements uncoverable → verdict: not-admitted
                              ▼              (never ran; missing-capabilities is the distance)
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

```
WorkUnit
  unit-ref           : string        # opaque id. The executor treats this as a label,
                                      # NOT a pointer into product-cli's graph.
  parent-deliverable : string        # opaque lineage tag. Carried solely so a verdict
                                      # can later be rolled up. Opaque to the executor.
  bundle-hash        : string        # content hash of the frozen SPMC bundle below.
                                      # This IS the work-unit's identity. Present even
                                      # though the bundle travels by value (see note).
  spmc-bundle        : object        # the frozen Input Contract — FULLY self-contained.
                                      # No reference the executor must resolve by calling
                                      # back. Knowledge entered at this boundary, with
                                      # provenance; nothing is acquired live mid-execution.
  model-binding      : object        # the pinned SPMC Model axis for the whole unit
                                      # (Specification Framework: Model is a BINDING, not
                                      # a name). Fixes model identity (provider/endpoint,
                                      # version) AND served precision/quantization AND
                                      # invocation params (temperature, top-p, seed where
                                      # honored). On the Spark, quantization is what makes a
                                      # model fit 128GB, so it is load-bearing, not an
                                      # implementation detail: two units at the same model
                                      # name but different quantization are DIFFERENT
                                      # bindings and not interchangeable. Homogeneous across
                                      # the unit's cells (the homogeneity invariant, stated
                                      # over binding — see note). The executor matches this
                                      # binding to a resident model / box configuration.
  tier               : string        # a label OVER the binding, for routing/escalation
                                      # convenience only (which rung, day vs night residency).
                                      # NOT a substitute for model-binding: the binding is
                                      # what determines output and what is recorded for
                                      # attribution; the tier is just its routing handle.
  acceptance-class   : enum          # auto-commit-if-green | needs-verdict
                                      # Tells a CONSUMER whether a green verdict may
                                      # auto-commit. The executor passes it through; it
                                      # does not interpret deliverable-level meaning.
  ladder-position    : int           # current rung on the Worker-role escalation ladder.
                                      # Set by dispatch; advanced on re-dispatch after a
                                      # rejecting verdict. The executor reads it to pick
                                      # the tier; it does not own the ladder.
  artifact-delivery  : object        # DECLARED destination for produced work:
                                      #   mode: inline     → artifacts return by value in the
                                      #                      VerdictEvent cell-results
                                      #   mode: repository → the executor works against the
                                      #                      declared git repository
                                      #                      { url, base-ref, integration{
                                      #                        method: push-branch | pull-request,
                                      #                        target-ref, branch-name } }.
                                      #                      file:/// URLs cover local development;
                                      #                      remote URLs cover production. The run
                                      #                      lands per the integration method and
                                      #                      the VerdictEvent carries status + refs
                                      #                      only (delivery-result). NO credential
                                      #                      material, ever — the unit is frozen,
                                      #                      hashed, stored, and re-emitted; repo
                                      #                      access is executor-side configuration.
  cell-graph         : object        # the SEALED interior. Opaque to the queue; walked
                                      # only by the executor. Carries intra-unit cell
                                      # dependencies (e.g. test-before-implement). No
                                      # cross-unit edges exist, ever.
  environment        : object        # the declared execution boundary for this unit
                                      # (Execution Contract: Environment). Permitted
                                      # network destinations, isolation guarantees. In
                                      # repository delivery the executor's working tree
                                      # (a per-run worktree or clone of the declared repo)
                                      # is the writable mount; how it is isolated from
                                      # concurrent runs is executor-internal. Enforced by
                                      # construction:
                                      # the executor runs the unit in a per-unit sandbox
                                      # destroyed at verdict. Cells take effects (file
                                      # writes, service calls) INSIDE this boundary only.
  credential-grant   : string        # a REFERENCE, never a secret (Execution Contract:
                                      # Credentials). The isolation boundary exchanges it
                                      # for short-lived, least-privilege credentials scoped
                                      # to this unit's declared effect destinations, and
                                      # revokes them when the sandbox is destroyed. Absent
                                      # when the unit needs no effect destination. Repository
                                      # push access and forge (PR) API access are exchanged
                                      # HERE, at the boundary — never read from the WorkUnit,
                                      # whose repository block deliberately carries no token.
  tool-grants        : array         # each cell-invokable tool, classified effect|knowledge
                                      # (Execution Contract: Tools). Effect tools may fire
                                      # during execution; knowledge tools may NOT — all
                                      # knowledge was frozen into spmc-bundle at dispatch.
```

**Constraints the emitter must honor (checked before emit):**

1. **The SPMC bundle is fully resolved.** No element requires a callback into product-cli to read. If the executor would ever need `product context …` mid-run, the unit is not frozen and must not be emitted.
2. **Tier-homogeneity holds.** Every cell in `cell-graph` requires `tier`. A heterogeneous unit is a decomposition defect and fails dispatch — this check lives on the **specification side**, alongside `graph check` / `gap check` / `drift check`, because a malformed decomposition is a spec defect, not a runtime condition.
3. **`bundle-hash` is the hash of `spmc-bundle` as emitted.** Identity and body agree at emit time.
4. **The destination is declared.** `artifact-delivery` states where produced work lands: `inline` (by value in the VerdictEvent — the seam stays store-free) or `repository` (a declared git repository; `file:///` for local development, remote for production, with the integration method the producer must know to reconcile). An undeclared destination is forbidden; a declared repository is a transport choice, not a hidden store. The integration method (`push-branch` vs `pull-request`, and the target ref) is contract material because the producer must know what ref shape to expect back; how the executor isolates concurrent runs against one repository (worktrees, clones) is deliberately absent.
5. **No credential material, by design.** The WorkUnit is frozen, hashed, stored, and re-emitted on re-dispatch — a token inline would bake a secret into an immutable, logged artifact. Repository access (and the forge API call that opens a pull request) is an executor-side, work-unit-scoped grant exchanged at the boundary, never a field of the unit.

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
  verdict            : enum          # accepted | rejected | escalate | not-admitted
                                      # The Execution Contract verdict vocabulary (its
                                      # declared minimum) plus the contract-tier extension
                                      # not-admitted: the unit NEVER EXECUTED because the
                                      # executor lacked a required capability. Admission
                                      # failure is distinct from execution failure — a
                                      # boolean in an exit code cannot express it, and a
                                      # higher tier does not fix a missing capability.
  missing-capabilities : string[]?   # REQUIRED with not-admitted, absent otherwise: the
                                      # DISTANCE — prefixed capability identifiers the
                                      # executor lacks for this unit (see CapabilityManifest).
  tier-ran           : string        # the tier the unit actually executed at. Equals the
                                      # WorkUnit tier on a first run; on a re-dispatch it
                                      # records the escalated tier that produced this verdict.
                                      # Absent on not-admitted (nothing ran).
  cell-results       : array         # per-cell verdicts from walking the sealed cell-DAG.
                                      # The executor's evidence for the unit verdict. A
                                      # consumer may ignore this and act on `verdict` alone;
                                      # it is present for audit and for heterogeneity metrics.
                                      # Absent on not-admitted (nothing ran).
  delivery-result    : object?       # REPOSITORY mode only — where the run landed:
                                      #   { branch, commit, pr-url? }. The commit SHA IS the
                                      #   content hash over the produced tree (git provides
                                      #   the canonicalization), so provenance is preserved
                                      #   with no artifact payload; pr-url is present when the
                                      #   integration method was pull-request.
  next-consequence   : enum          # advance | halt | retry | escalate
                                      # The Transition Contract binding the executor applied
                                      # or recommends. On `escalate`, signals that a
                                      # re-dispatch one tier up is the declared next step —
                                      # which, when the next rung is night-only, is deferral
                                      # across the developer switch. On not-admitted the
                                      # expected consequence is halt or escalate (route
                                      # elsewhere / surface to a human), never advance — a
                                      # retry against the same executor is meaningful only
                                      # after its capabilities change.
```

**Consumer obligations (product-cli as the canonical subscriber):**

- **Reconcile, do not poll-and-block.** product-cli observes verdict events and updates its graph: unit-verdict → work-unit state → roll up to deliverable-done. The rollup is the specification side's business; the executor neither performs nor understands it.
- **`escalate` / re-dispatch is a fresh emit.** On a rejecting or escalating verdict whose ladder is not exhausted, product-cli re-runs `dispatch` for the unit with `ladder-position` advanced — producing a new WorkUnit (new run, same `unit-ref`, same `bundle-hash` if the bundle is unchanged). Escalation is unit-atomic: the whole cell-DAG re-runs at the higher tier. The executor never re-dispatches itself; it only emits the verdict that prompts a consumer to.
- **`not-admitted` does not advance the ladder.** A missing capability is not a difficulty a higher tier overcomes. On `not-admitted` product-cli re-routes to an executor whose manifest covers the `missing-capabilities` distance, escalates to a human, or (if the missing capability appears in the target's manifest) treats the manifest as stale — it never bumps `ladder-position` and re-dispatches at the same executor.
- **Repository delivery reconciles on refs, not payloads.** When the unit declared `artifact-delivery: repository`, the verdict carries `delivery-result` (branch, commit, and `pr-url` when the method was pull-request) and no per-cell artifact bodies. product-cli reconciles deliverable-done against those refs; the commit SHA is the provenance anchor (it is the content hash over the produced tree). `inline` delivery still returns artifact bodies by value in `cell-results`.
- **Acceptance-class is honored by the consumer, not the executor.** `auto-commit-if-green` means a consumer may commit on an `accepted` verdict without human review; `needs-verdict` means it surfaces for a human. The executor emitted the same event either way.

---

## Schema 3 — CapabilityManifest (execution → published)

Published by the executor, **not per run**. It is the executor's declared answer to "what can run here," so a producer can compute a unit's executability at *planning time* rather than discover it as a runtime surprise, and so a production deploy target is selectable as "an executor whose manifest covers this unit's requirements." Kiln is the **authoring** side of this contract — the one seam artifact it writes rather than consumes.

```
CapabilityManifest
  executor-id        : string        # stable identity of this Kiln instance (e.g. a box id).
  emitted-at         : timestamp     # when this snapshot was published. Manifests go stale;
                                      # a match is pre-flight, not a guarantee (see below).
  capabilities       : object        # NORMATIVE for admission matching — stable facts:
    bindings[]       :   array       #   Model bindings this box can serve, each pinned with
                                      #   the same identity fields a WorkUnit binding carries
                                      #   (provider, model-id, revision?, quantization). A unit
                                      #   is servable iff its binding matches one. On the Spark
                                      #   these are exactly the resident/validated bindings.
    delivery         :   object      #   modes (inline | repository) and, for repository:
                                      #   url-schemes (file, https, ssh), integration-methods
                                      #   (push-branch, pull-request), and forges reachable
                                      #   for PR creation.
    shape-languages[]:   array       #   shape languages the executor can validate output
                                      #   against (e.g. "JSON Schema", "SHACL").
    gate-kinds[]     :   array       #   gate kinds the executor can run (open vocabulary;
                                      #   matched by string equality to each cell gate.kind).
  operational        : object?       # ADVISORY ONLY — routing hints, never admission facts:
                                      #   capacity, queue-depth, current mode (QUEUE|EXPLORER).
```

**Requirements, distance, reach.** A unit's **requirements** are derived from the unit *itself* — no extra declaration: the binding it carries, its delivery mode + URL scheme + integration method, the shape languages its cells use, the gate kinds its cells declare. The **distance** is `requirements(unit) − capabilities(executor)`; **empty distance = executable**. A unit's **reach** is the set of executors whose manifests cover it — a minimal inline-delivery unit reaches everywhere; a unit demanding an exotic quantization plus GitHub-PR delivery reaches few boxes. Reach is a property of how the unit was specified, so a producer widens it by demanding less.

**The normative / advisory split is load-bearing.** Matching MUST use only `capabilities`. `operational` (queue depth, mode) is volatile by nature and MAY order routing *among boxes that all match* — it MUST NOT make a unit "inexecutable." Capacity is a queueing concern, never a capability gap; without this split a stale queue reading would make executability flicker.

**The manifest is pre-flight; admission is authoritative.** A published manifest is a snapshot, and the box's actual state at dispatch may differ — so a successful match **does not guarantee** admission. The executor remains the authority at its own boundary: a `not-admitted` verdict is still possible (and correct) after a match. The manifest exists to make inexecutability *predictable*, not impossible; a `not-admitted` whose missing capability appears in the manifest is a signal the manifest is stale.

**Capability identifiers.** `missing-capabilities` entries (and manifest extension entries) use prefixed strings by convention: `binding:<provider>/<model-id>@<quantization>` · `delivery:<mode>` · `url-scheme:<scheme>` · `integration:<method>` · `forge:<host>` · `shape-language:<name>` · `gate:<kind>`. The prefixes are contract-defined; frameworks may add their own prefixed identifiers, matched by string equality.

---

## What crosses the seam, exhaustively

Per run — outbound: **WorkUnit (by value)**; inbound: **VerdictEvent (by event)**. Out of band — the executor **publishes a CapabilityManifest** the producer matches against before dispatch (and the routing surface for production deploys). Nothing else. No live call, no shared store, no shared endpoint. The one permitted destination for produced work is the git repository the WorkUnit itself declares (`artifact-delivery: repository`; `file:///` for local development, remote for production) — a declared transport, not a hidden channel. The two sides share three schemas and two per-run emit-points. That is the entire surface area of the coupling.

---

## Success criteria

1. The executor resolves nothing by calling product-cli. Given a WorkUnit, it executes to a verdict with no further input.
2. A VerdictEvent is reconcilable by a consumer that never saw the dispatch — its lineage and `bundle-hash` are sufficient.
3. Replacing the executor (different hardware, different scheduler) requires no change to product-cli, and vice versa, as long as both schemas are honored.
4. Every accepted verdict is attributable to an immutable frozen bundle by `bundle-hash` — provenance is reproducible.
5. The producer (executor) holds no knowledge of any consumer.
6. Temporal decoupling holds: a verdict emitted at night is correctly reconciled whenever a consumer next reads the stream, with no executor-side wait.
7. A unit's executability is computable before dispatch by matching its derived requirements against a published CapabilityManifest; a `not-admitted` verdict names the concrete distance (`missing-capabilities`) rather than failing silently or partially.
8. No credential ever crosses in a WorkUnit: repository push and forge-API (PR) access are executor-side, work-unit-scoped grants exchanged at the boundary; the frozen, hashed unit carries none.

---

## Excludes

- **Either side's internals.** How product-cli decomposes a slice, and how the executor schedules or batches, are out of scope — owned by their respective framework-implementation specs.
- **The transport of the event stream.** Whether the stream is a log, a bus, or a file is an implementation choice below this contract; the contract requires only that emit is fire-and-forget and that events are self-describing.
- **By-reference bundle transport.** Acknowledged as a future additive migration (the `bundle-hash` reserves it), not specified here.
- **The developer switch and box modes.** Those are the substrate's concern; this contract is mode-agnostic — a WorkUnit's `tier` is all the executor needs to match residency.
- **How the executor isolates concurrent runs against one repository.** Worktrees, separate clones, or serialization are executor-internal; the contract fixes only the declared repository, the integration method, and the refs that come back.
- **The transport by which the CapabilityManifest is published.** Whether it is served over HTTP, committed to a registry, or read from a file is below this contract; the contract fixes only its shape and that it is queryable before dispatch.

---

## Acknowledges

- **By-value re-sends the bundle on every (re-)dispatch.** Negligible on a single box; the `bundle-hash` keeps the by-reference migration open for federation without a schema break.
- **The tier-homogeneity check sits on the specification side.** It is a spec-side conformance gate run before emit, even though the property it guarantees is what makes execution-side escalation lossless. The check is pillar-one's; the benefit is pillar-two's. That split is intended.
- **`escalate` couples the two pillars only through the verdict vocabulary.** The executor recommends escalation by emitting a verdict; the consumer decides to re-dispatch. Neither owns the whole escalation loop — it closes across the seam, by data, exactly as intended.
- **The CapabilityManifest is the only seam artifact the execution side authors.** Producer-owns governs what the *producer* freezes and hands over; a consumer publishing facts about itself is the complement, not a breach. A match is pre-flight — admission at the boundary stays authoritative, so `not-admitted` after a match is expected, not a contradiction.

---

## References

- **Specification Framework** — one SPMC bundle per discrete work-unit; the derivation contract; the work-unit as specification's floor.
- **Execution Contract** — Input Contract (frozen, bounded); Output Contract (declared shape and destination); Verification (declared verdict: accepted / rejected / escalate); Transition Contract (verdict → consequence); Capabilities (every action category explicit, granted or withheld — the block the CapabilityManifest publishes and makes matchable); Tools (knowledge enters at the frozen boundary, never live).
- **Two Pillars** — peers, swappable independently; stability flows downward; the interface between framework implementations must not bind their lifecycles.
- **The Spark Execution Substrate** — the execution-pillar implementation on the consuming end of this contract; the developer switch, QUEUE/EXPLORER modes, and the tier-homogeneity invariant in full.
- **product-cli** — the specification-pillar implementation on the emitting end; slice → work-unit → cell; `build` to be split into `dispatch` + executor; computed-done (ADR-071); protected oracle (ADR-076); test-first work units (ADR-075); worker capability catalog and role-binding escalation ladder.
