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
