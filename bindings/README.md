# Bindings

Validated SPMC **Model bindings** — Kiln's firing schedules — for the reference substrate. Each file is a fully-pinned,
hardware-validated instance of the Specification Framework's Model axis: the exact
tuple (image, driver, model, quantization, kernel-routing env, serving flags,
sampling defaults, thinking behaviour) that makes a model *name* into one
determinate behaviour on this box.

A binding here is what fills the `model-binding` field of a WorkUnit. The
binding-homogeneity invariant requires every cell in a work-unit to share one of
these — same model, same quantization, same params.

A file earns a place here only when it has a **validation record**: it served,
answered correctly, and its batching behaviour was measured on real hardware.
An unvalidated binding is a draft, not a binding.

## Current bindings

- **[qwen3.6-35b-gb10.yaml](qwen3.6-35b-gb10.yaml)** — Qwen3.6-35B-A3B-FP8 on
  DGX Spark GB10. Day tier, QUEUE mode. Validated 2026-07-02; ~3× batching at
  `max_num_seqs=4`.
- **[qwen3.6-27b-gb10.yaml](qwen3.6-27b-gb10.yaml)** — Qwen3.6-27B dense FP8 on
  DGX Spark GB10. **Quality tier (draft)** — higher coding quality, lower decode
  speed and batch density. Awaiting hardware validation.
