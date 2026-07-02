# Changelog

All notable changes to the Spark Execution Framework are recorded here.
Format follows [Keep a Changelog](https://keepachangelog.com/); the framework
versions on its own axis, independently of any implementation.

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

### Added — 2026-07-02 (contracts conformance)
- `conformance/contracts-conformance.md` — consumer-side declaration against the
  [AI Development Contracts](https://github.com/Hafeok/ai-development-contracts) tier
  (contracts `0.1.0`, foundation_ref `main`). Walks the consumer checklist item for
  item; autonomy declared per-unit-kind via `acceptance-class`.

### Changed — 2026-07-02 (contracts conformance)
- `interface/work-unit-interface.md` — now the **Spark-specific projection** of the
  shared contracts, not a competing schema. Names ai-development-contracts as source
  of truth; aligns the WorkUnit to the published shape (`spmc-bundle.model` /
  `context-pool`, cell-graph interior); **adds `artifact-delivery`** and the
  per-cell **artifact `content-hash` + delivery** in the VerdictEvent; reframes
  `environment` / `credential-grant` / `tool-grants` as optional executor-side
  additions with conformant defaults; states schema-validation-before-admission.
- `conformance/execution-contract-checklist.md` — Input Contract gains a
  validate-against-normative-schema-and-reject item.
- `docs/01-execution-framework.md` — Output Contract row now covers artifact-delivery
  modes and content-hash; contracts layer added to references.

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
