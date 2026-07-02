# Glossary

> Every load-bearing term in Kiln, pinned once. Where a term is inherited from a foundation it is marked and its meaning is not re-defined here, only located. Where this framework introduces or specialises a term, the definition here is normative.

---

## Inherited from the foundations

**Worker** *(Execution Contract)* — a role, not a model. A declared unit of authority with a bounded permitted/forbidden surface. The same model fills different worker roles at different bindings.

**Capability** *(Execution Contract)* — an explicit grant over the read / write / call / spawn floor. Held to exactly what the task needs.

**Input Contract** *(Execution Contract)* — the frozen, bounded input a worker is a function of. Here, the SPMC bundle, mounted read-only, identified by `bundle-hash`.

**Output Contract** *(Execution Contract)* — the declared shape and destination of what a worker produces. Here, the VerdictEvent (to the stream) and produced work landing per `artifact-delivery` (inline by value, or into the declared git repository).

**Verification** *(Execution Contract)* — a declared verifier producing a verdict in the vocabulary *accepted / rejected / escalate*, against an oracle the worker cannot write.

**Transition Contract** *(Execution Contract)* — the binding of every verdict to a consequence (advance / halt / retry / escalate). Its shape *is* the autonomy level.

**Rule 1 / the funnel** *(Execution Contract & Specification Framework)* — constraint density rises toward the action; required capability falls correspondingly. A worker needing broad capability signals upstream under-decomposition. Asserted by both pillars.

**SPMC** *(Specification Framework)* — Schema, Prompt, Model, Context: the four-axis execution unit a model consumes. One bundle per discrete work-unit.

**Model binding** *(Specification Framework, Model axis)* — the pinned Model axis: identity (provider/endpoint, version), **served precision/quantization**, and invocation parameters (temperature, top-p, seed where honored). A model *name* is not a binding. A change of binding under an unchanged name is still a Model-axis change and resets the quality baseline.

**Derivation contract** *(Specification Framework)* — every How element traces to a What element; every SPMC bundle to a How element. Nothing dangles.

**Closure contract** *(Execution Contract)* — a unit of work traced end to end; every grant traces to a need, every output is verified, every verdict has a consequence. The execution-side mirror of the derivation contract.

**Producer-owns / specification-stewarded** *(Two Pillars)* — the side that freezes and hands over the work-unit defines its shape; execution conforms. The interface versions on its own axis, owned by no single framework or executor.

---

## Introduced or specialised by Kiln

**Cell** — the execution primitive. The smallest thing the executor runs. Cells live *inside* a work-unit, ordered by an intra-unit dependency DAG (e.g. test-before-implement). Cells are never reordered or scheduled by the queue; they are the sealed interior of a work-unit.

**Work-unit** — the **schedulable atom**. The queue holds, reorders, and escalates work-units, never cells. A work-unit carries one SPMC bundle, one Model binding, an acceptance-class, and a sealed cell-DAG interior. Independent of all other work-units (no cross-unit dependencies). It is the object both pillars name as their edge, and therefore the interface type across the seam.

**Cell-DAG** — the dependency graph of cells within one work-unit. Sealed: opaque to the queue, walked only by the executor once the unit is admitted. The single isolation domain of a work-unit.

**Binding-homogeneity invariant** — a work-unit is well-formed iff all of its cells require the same Model binding. The funnel at the work-unit scale. Serves as a pre-dispatch conformance check, an escalation simplifier (unit-atomic escalation is lossless), and a maturation metric (heterogeneity rate falls as the corpus types itself).

**Unit-atomic escalation** — on a rejecting/escalating verdict, the *whole* work-unit re-enqueues at the next Model binding; the entire cell-DAG re-runs. Cells are never escalated individually across a residency boundary. Made lossless by binding-homogeneity.

**Tier** — a routing handle *over* a Model binding (which rung, day vs night residency). Not a substitute for the binding: the binding determines output and is what is recorded for attribution; the tier is only its routing label.

**Box configuration / mode** — the whole-box state the developer switch selects: QUEUE or EXPLORER. Mutually exclusive in VRAM. Set externally by a human, never inferred by the box.

**QUEUE** — the box configuration for batched execution: small/medium model bindings resident, the executor draining the work-unit priority set. The box's best regime. Trust properties (frozen input, computed verification, declared transition) hold in full.

**EXPLORER** — the box configuration for local frontier reasoning: the largest resident model binding, an OpenCode agentic loop, serial and slow. Produces a **discovery record**, not accepted code. Used only for novel work bound to the box by locality, cost-at-volume, or offline need.

**OFF-BOX** — not a box configuration. Opus-class inference off the Spark entirely, for define-step authoring and any non-local exploration. Always available.

**Discovery record** — the output of EXPLORER (or off-box exploration): a candidate decomposition (proposed slice, work-unit boundaries, cells). Not shippable code. The only sanctioned channel by which exploratory work re-enters engineered mode — it must pass back through dispatch as conformant work-units.

**Developer switch** — the single human-thrown, human-rate control selecting the box configuration. The mode-controller is its actuator, not an inference engine; it enforces the VRAM mutual-exclusion and loads the chosen configuration.

**Dispatch (`product dispatch`)** — the specification side's last act: resolve What/How pointers into a frozen by-value SPMC bundle, run the binding-homogeneity check, and emit the WorkUnit. The boundary between the pointer-world of specification and the frozen-world of execution. Mechanical; no model required.

**WorkUnit (schema)** — the outbound wire contract across the seam. Carries: unit-ref, parent-deliverable, bundle-hash, the by-value SPMC bundle, the Model binding, tier, acceptance-class, ladder-position, `artifact-delivery` (the declared destination — inline or repository), environment declaration, credential-grant reference, tool-grants, and the sealed cell-graph. It carries **no credential material**: it is frozen, hashed, stored, and re-emitted, so repository/forge access is exchanged executor-side at the boundary, never inlined.

**VerdictEvent (schema)** — the inbound wire contract across the seam. Self-describing: carries event-id, emitted-at, unit-ref, parent-deliverable, bundle-hash, verdict (accepted | rejected | escalate | **not-admitted**), tier-ran, cell-results, next-consequence, plus `missing-capabilities` (required with not-admitted; the distance) and `delivery-result` (repository mode; branch, commit, pr-url). On not-admitted the unit never ran, so tier-ran and cell-results are absent. Emitted fire-and-forget to a stream; the producer knows no consumer.

**CapabilityManifest (schema)** — the executor's published self-description of what can run here: servable `bindings`, `delivery` modes/url-schemes/integration-methods/forges, `shape-languages`, `gate-kinds` (normative for matching), plus advisory `operational` hints (capacity, queue-depth, mode). The one seam artifact Kiln *authors* rather than consumes. Crosses **out of band** (published, queryable), not per run. Its `capabilities` are the queryable, matchable form of the Execution Contract's Capabilities block.

**Artifact-delivery / repository delivery** — the WorkUnit's declared destination for produced work. `inline`: artifact bodies return by value in the VerdictEvent. `repository`: the executor works against the declared git repository (`{ url, base-ref, integration }`; `file:///` local, remote for production) and lands the run per the integration method; the VerdictEvent then carries only status + refs. Replaces the earlier `workspace` mode.

**Integration method** — how a completed repository-delivery run lands: `push-branch` (the executor pushes a branch) or `pull-request` (the executor opens a PR against a target ref via a forge API call — itself a declared capability). Contract material, because the producer must know what ref shape to expect back to reconcile deliverable-done; how the executor isolates concurrent runs against the repository (worktrees, clones) is not.

**Requirements / distance / reach** — a unit's *requirements* are derived from the unit itself (its binding, delivery mode + url-scheme + integration method, cell shape-languages, cell gate-kinds), no extra declaration. *Distance* is `requirements(unit) − capabilities(executor)`; empty distance = executable. *Reach* is the set of executors whose manifests cover a unit — a property of how the unit was specified, widened by demanding less.

**Admission / not-admitted** — the boundary gate that precedes execution: a unit whose derived requirements the executor cannot cover is answered `not-admitted` with the concrete `missing-capabilities`, having *never run*. Admission failure is distinct from execution failure — a higher tier fixes the latter, never the former — so not-admitted binds to halt/escalate (route elsewhere or surface to a human), never to advancing the ladder. A CapabilityManifest match is pre-flight; admission stays authoritative, so not-admitted after a match is expected (a signal the manifest was stale), not a contradiction.

**Acceptance-class** — a per-work-unit field declaring the human-free transition bindings: `auto-commit-if-green` (a verdict→advance binding that proceeds without a human — Level 5 for that unit kind) or `needs-verdict` (accepted still escalates to a human — Level 4). The autonomy level is thus per-unit-kind, declared, not a global setting.

**Per-unit ephemeral sandbox** — the Environment mechanism: each work-unit runs in its own isolation domain (frozen bundle read-only; the writable working tree is a per-run worktree/clone of the declared repository, or a private scratch workspace under inline delivery; no network except declared destinations — the repo host and, for pull-request delivery, the forge; no host/peer reach), destroyed at verdict. Why the cell-DAG is sealed: it is one sandbox. Isolating concurrent runs against one repository is the sandbox's concern, invisible to the seam.

**Kiln** — this framework: an execution-pillar implementation for tiered, verified, autonomous work on pinned open models. The metaphor is normative shorthand: a *firing* is a batched execution against one binding; a load must be *temperature-homogeneous* (binding-homogeneity); firing schedules are set by the potter (the developer switch), never the kiln.

**Reference substrate** — the concrete hardware the reference implementation targets (NVIDIA DGX Spark). Kiln's obligations are substrate-neutral; the substrate motivates, but does not own, the two-configuration design.

**Maturation** — the compounding loop: exploration mints binding-homogeneous work-units; executing them cheaply by day feeds accepted work back; the heterogeneity rate falls and more work descends to cheap bindings permanently. The funnel made temporal.
