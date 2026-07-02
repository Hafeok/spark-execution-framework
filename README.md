# Kiln

**An execution framework for tiered, verified, autonomous work on pinned open models.**

Kiln is a **framework implementation** of the execution pillar — one opinionated, conformant way to satisfy the [AI Development Foundations](https://github.com/Hafeok/ai-development-foundations) **Execution Contract**, exactly as the Assembly Line Protocol does, and exactly as Decision-Driven Design sits on the Specification Framework. It executes the work units a specification pillar produces, routes each to the cheapest model binding that can do it, batches what can be batched, verifies everything, and never trusts a worker's claim of done.

The name is the design. A kiln is batch-fired: many pieces loaded and fired together. A kiln load must be temperature-homogeneous — every piece in a firing tolerates the same fire, or the load was packed wrong. A firing is atomic: pieces needing a hotter fire go into a *different firing*, not a hotter corner. Firing schedules are set by the potter, never by the kiln. And nothing is trusted until it comes out and survives inspection.

```
ai-development-foundations         (the standard — peers, neither contains the other)
   Specification Framework  ║  Execution Contract
            │               ║          │
   Decision-Driven Design   ║  KILN                        ← this repo (an ALP-class impl)
   product-framework        ║          │
            │               ║          │
   project specs/bundles    ║  a running Kiln pipeline     ← implementation repo (separate)
            └──────── Work-Unit Interface ────────┘
                 (producer-owned, specification-stewarded)
```

## What Kiln is — and is not

Kiln is **not a batch framework**, though QUEUE mode batches aggressively. Batching is a mechanism; the policy is the **funnel**: every work-unit carries one pinned model binding, units are routed to the cheapest binding that suffices, a unit whose cells would need mixed bindings is rejected as a decomposition defect, and a unit that fails escalates *whole* to the next binding. The system gets cheaper over time as work types itself downward.

Kiln is **not hardware-specific**. The framework's obligations — the eight Execution Contract building blocks, the conformance ladder support, the Work-Unit Interface — are substrate-neutral. Any executor honouring the same contracts satisfies the same obligations. The **NVIDIA DGX Spark is the reference substrate** (the Spark lights the Kiln), and its constraints shaped the reference design: a single bandwidth-bound box forces the two firing configurations, QUEUE (batched, many small bindings) and EXPLORER (one large binding, serial, discovery only), mutually exclusive and switched by a developer.

Kiln **requires open models**. This is a hard dependency, not a preference: the core invariant — binding-homogeneity over a *pinned* Model binding (identity + served quantization + invocation params) — is only enforceable when the weights are open and the serving stack is yours. A closed API can swap precision under an unchanged name, which dissolves the binding and with it the invariant. Kiln's trust model starts at the pin.

## Read in this order

1. **[docs/00-overview.md](docs/00-overview.md)** — the bridge. What Kiln is, the two-level control model, and the three open questions it does not yet close. Start here.
2. **[docs/01-execution-framework.md](docs/01-execution-framework.md)** — the conformance core. How every one of the Execution Contract's eight building blocks is discharged, how the Product Framework's conformance ladder (Described → Realised → Verified → Delivered) is supported, and the closure trace. This is the normative claim.
3. **[docs/02-substrate.md](docs/02-substrate.md)** — the reference substrate. The developer switch over QUEUE / EXPLORER / OFF-BOX on the DGX Spark, the queue's owned data model (WorkUnit as the schedulable atom, the sealed binding-homogeneous cell interior), and the binding-homogeneity invariant.

The seam to the specification pillar is specified separately, because it versions on its own axis:

- **[interface/work-unit-interface.md](interface/work-unit-interface.md)** — the Work-Unit Interface. Two per-run schemas (WorkUnit out, VerdictEvent in) and two emit-points, plus the out-of-band **CapabilityManifest** Kiln publishes so producers can match a unit's requirements before dispatch; no shared runtime surface. Produced work lands either inline or in a declared git repository (`file://` local, remote for production) — never via an undeclared store, and never with a credential in the unit. Producer-owned, specification-stewarded, per the Two Pillars seam rule.

## Conformance

- **[conformance/execution-contract-checklist.md](conformance/execution-contract-checklist.md)** — the eight building blocks restated as a checkable list, each with the mechanism that discharges it. A Kiln instance (or a CI gate) can check itself against this.
- **[conformance/product-framework-support.md](conformance/product-framework-support.md)** — the four ladder rungs restated as the execution-side obligations Kiln supplies the machinery for.

## Bindings

- **[bindings/](bindings/)** — validated SPMC Model bindings: fully-pinned, hardware-tested firing schedules that fill a WorkUnit's `model-binding` field. Each names its tier. These resident bindings are also exactly what a Kiln instance advertises as `capabilities.bindings` in its published CapabilityManifest — a unit is servable iff its binding matches one. See [qwen3.6-35b-gb10.yaml](bindings/qwen3.6-35b-gb10.yaml) for the first validated binding (day tier, measured ~3× batching), and [qwen3.6-27b-gb10.yaml](bindings/qwen3.6-27b-gb10.yaml) for the quality-tier draft.

## Reference

- **[GLOSSARY.md](GLOSSARY.md)** — every load-bearing term pinned once: those inherited from the foundations (located, not redefined) and those Kiln introduces (normative).
- **[CHANGELOG.md](CHANGELOG.md)** — version history. Kiln versions on its own axis, independently of any implementation.
- **[CONTRIBUTING.md](CONTRIBUTING.md)** and **[rfcs/](rfcs/)** — how normative changes are proposed and gated.

## Status

These documents are **descriptive of a framework, not yet ratified by a running implementation.** Three questions remain open and are called out in the overview; each closes only by carrying a concrete specification through the chain, not by further design:

1. Whether any legitimate work-unit pattern is intrinsically binding-mixed (the one caveat on the binding-homogeneity invariant going normative).
2. Whether a fully-resolved SPMC bundle ever leaves a dangling reference the executor would have to chase (the self-contained-at-dispatch claim).
3. Whether the slice→unit→cell fan-out produces sensible cell-per-unit grain (the utilisation argument's empirical premise).

A degraded sovereign mode (local large-model substitution under an offline constraint) is acknowledged but not yet specified.

## Dependencies

Kiln depends on:

- **[AI Development Foundations](https://github.com/Hafeok/ai-development-foundations)** — the Execution Contract (which it conforms to), the Specification Framework and Two Pillars (which it references).
- **[product-framework](https://github.com/Hafeok/product-framework)** — the specification-pillar instantiation whose work units it executes and whose conformance ladder it supports.

It is depended on by:

- the separate **implementation repo** — an actual running Kiln pipeline (reference substrate: DGX Spark).

## License

Dual-licensed, matching the foundations and product-framework convention:

- **Documentation** (everything in `docs/`, `interface/`, `conformance/`, the glossary): [CC BY 4.0](LICENSE-DOCS). Build on, adapt, and redistribute with attribution.
- **Repository code and configuration** (if any is added): [Apache-2.0](LICENSE).
