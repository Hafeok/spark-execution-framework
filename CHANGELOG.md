# Changelog

All notable changes to **Kiln** are recorded here.
Format follows [Keep a Changelog](https://keepachangelog.com/); the framework
versions on its own axis, independently of any implementation.

## [0.3.0] — 2026-07-02

Conforms to the **repository-delivery + CapabilityManifest** revision of
[ai-development-contracts](https://github.com/Hafeok/ai-development-contracts)
(checked against contracts `0.2.0`).

### Added
- **CapabilityManifest — Kiln now authors a seam artifact.** The executor
  publishes a self-description of what can run here (servable `bindings`,
  `delivery` modes/schemes/integration-methods/forges, `shape-languages`,
  `gate-kinds`; advisory `operational` hints) so a producer matches a unit's
  requirements *before* dispatch. New "Schema 3" in the Work-Unit Interface, a
  "Publishing capabilities" section in `docs/01-execution-framework.md` and both
  conformance checklists, and a substrate note tying `capabilities.bindings` to
  the box's resident bindings and the developer switch to an advisory hint.
- **`not-admitted` verdict + `missing-capabilities`.** Admission is a distinct
  gate before verification: a unit whose requirements the box cannot cover never
  runs and is answered with the concrete distance. It binds to halt/escalate,
  never to advancing the ladder (a higher tier does not add a capability).
- **`delivery-result` on the VerdictEvent** (repository mode): branch, commit,
  `pr-url` — refs, not payloads; the commit SHA is the content hash over the
  produced tree.

### Changed
- **Artifact delivery: `workspace` → `repository`.** Produced work lands inline
  (by value) or in a **declared git repository** (`file:///` local, remote for
  production) with an integration method (`push-branch` | `pull-request`). The
  producer knows the ref shape to expect back; concurrency isolation (worktrees,
  clones) is executor-internal. Updated across the interface, both docs, the
  glossary, and both conformance checklists.
- **No credential material in the WorkUnit, by design.** The unit is frozen,
  hashed, stored, and re-emitted; repository push and forge-API (PR) access are
  exchanged executor-side at the boundary from the `credential-grant` reference.
  Opening a pull request is a declared **capability**. Environment/Credentials
  discharge updated accordingly (writable mount is the declared repository
  working tree; network floor includes the repo host and, for PR delivery, the
  forge).

## [0.2.0] — 2026-07-02

### Changed
- **Renamed and repositioned: the Spark Execution Framework is now Kiln.** The
  framework was never hardware-specific; its obligations (eight-block discharge,
  ladder support, the Work-Unit Interface) are substrate-neutral. The DGX Spark
  is demoted to *reference substrate*. Two identity claims are promoted to the
  front: the **funnel** is the policy (batching is only QUEUE mode's mechanism),
  and **pinned open models are a hard dependency** (binding-homogeneity is
  unenforceable against a closed API that can swap precision under a name).
- `docs/02-substrate.md` retitled "The Reference Substrate: DGX Spark"; its data
  model and invariant remain Kiln-normative.

### Added
- `bindings/qwen3.6-27b-gb10.yaml` — quality-tier binding (dense 27B), DRAFT
  pending hardware validation; placeholders for the comparative batching test.

## [0.1.0] — 2026-06-26

First public draft. Descriptive of a framework, not yet ratified by a running
implementation. Three validation questions remain open (see the overview).

### Added
- `docs/00-overview.md` — bridge document; the two-level control model and the
  three open questions.
- `docs/01-execution-framework.md` — normative conformance core: all eight
  Execution Contract building blocks discharged, Product Framework ladder
  support (Described → Realised → Verified → Delivered), end-to-end closure trace.
- `docs/02-substrate.md` — the developer switch (QUEUE / EXPLORER / OFF-BOX),
  the queue's owned data model (WorkUnit atom, sealed binding-homogeneous cell
  interior), and the binding-homogeneity invariant.
- `interface/work-unit-interface.md` — the Work-Unit Interface: WorkUnit out (by
  value, hash-identified), VerdictEvent in (self-describing, fire-and-forget);
  producer-owned, specification-stewarded.
- `conformance/execution-contract-checklist.md` — the eight blocks as a checkable list.
- `conformance/product-framework-support.md` — the four ladder rungs as a checkable list.

### Added — 2026-07-02
- `bindings/qwen3.6-35b-gb10.yaml` — first hardware-validated Model binding
  (Qwen3.6-35B-A3B-FP8 on DGX Spark GB10). Serves via the pinned native
  `timothystewart6/vllm-gb10:v0.20.1-gb10.0` image with `VLLM_USE_DEEP_GEMM=0`.
  Batching measured at ~3x (12 concurrent / 18.9 s) at `max_num_seqs=4`.
- `bindings/README.md` — what a validated binding is and requires.

### Known open
- Whether any legitimate work-unit pattern is intrinsically binding-mixed.
- Whether a fully-resolved SPMC bundle can leave a dangling reference at dispatch.
- Whether the slice→unit→cell fan-out yields sensible cell-per-unit grain.
- Degraded sovereign mode (offline large-model substitution) — acknowledged, unspecified.
