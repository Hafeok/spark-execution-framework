# Spark Execution Framework

A **framework implementation** of the execution pillar — one opinionated, conformant way to run autonomous AI development work on bounded local hardware. It sits on the [AI Development Foundations](https://github.com/Hafeok/ai-development-foundations) **Execution Contract** exactly as the Assembly Line Protocol does, and exactly as Decision-Driven Design sits on the Specification Framework. It is named and concrete: its substrate is a single NVIDIA DGX Spark, and its job is to execute the work units a specification pillar produces, at the highest sustained utilisation the hardware allows, without weakening any trust property the Execution Contract requires.

```
ai-development-foundations         (the standard — peers, neither contains the other)
   Specification Framework  ║  Execution Contract
            │               ║          │
   Decision-Driven Design   ║  Spark Execution Framework   ← this repo (an ALP-class impl)
   product-framework        ║          │
            │               ║          │
   project specs/bundles    ║  the running Spark pipeline  ← the implementation repo (separate)
            └──────── Work-Unit Interface ────────┘
                 (producer-owned, specification-stewarded)
```

This repo is the **framework**: normative, stable, technology-described-but-implementation-free. The running pipeline — the executor, the queue daemon, the sandbox, the dispatch fan-out — lives in a separate implementation repo and depends on this one. Stability flows downward; volatility is contained upward.

## Read in this order

1. **[docs/00-overview.md](docs/00-overview.md)** — the bridge. What this framework is, where it sits, the two-level control model, and the three open questions it does not yet close. Start here.
2. **[docs/01-execution-framework.md](docs/01-execution-framework.md)** — the conformance core. How every one of the Execution Contract's eight building blocks is discharged, how the Product Framework's conformance ladder (Described → Realised → Verified → Delivered) is supported, and the closure trace. This is the normative claim.
3. **[docs/02-substrate.md](docs/02-substrate.md)** — the substrate. The developer switch over QUEUE / EXPLORER / OFF-BOX, the queue's owned data model (WorkUnit as the schedulable atom, the sealed binding-homogeneous cell interior), and the binding-homogeneity invariant.

The seam to the specification pillar is specified separately, because it versions on its own axis:

- **[interface/work-unit-interface.md](interface/work-unit-interface.md)** — the Work-Unit Interface. Two schemas (WorkUnit out, VerdictEvent in), two emit-points, no shared runtime surface. Producer-owned, specification-stewarded, per the Two Pillars seam rule.

## Conformance

- **[conformance/execution-contract-checklist.md](conformance/execution-contract-checklist.md)** — the eight building blocks restated as a checkable list, each with the mechanism that discharges it. A framework instance (or a CI gate) can check itself against this.
- **[conformance/product-framework-support.md](conformance/product-framework-support.md)** — the four ladder rungs restated as the execution-side obligations this framework supplies the machinery for.

## Reference

- **[GLOSSARY.md](GLOSSARY.md)** — every load-bearing term pinned once: those inherited from the foundations (located, not redefined) and those this framework introduces (normative).
- **[CHANGELOG.md](CHANGELOG.md)** — version history. The framework versions on its own axis, independently of any implementation.
- **[CONTRIBUTING.md](CONTRIBUTING.md)** and **[rfcs/](rfcs/)** — how normative changes are proposed and gated.

## Bindings

- **[bindings/](bindings/)** — validated SPMC Model bindings: fully-pinned, hardware-tested serving configurations that fill a WorkUnit's `model-binding` field. See [qwen3.6-35b-gb10.yaml](bindings/qwen3.6-35b-gb10.yaml) for the first validated binding (Qwen3.6-35B on GB10).

## Status

These documents are **descriptive of a framework, not yet ratified by a running implementation.** Three questions remain open and are called out in the overview; each can only be closed by carrying a concrete specification through the chain, not by further design:

1. Whether any legitimate work-unit pattern is intrinsically binding-mixed (the one caveat on the binding-homogeneity invariant going normative).
2. Whether a fully-resolved SPMC bundle ever leaves a dangling reference the executor would have to chase (the self-contained-at-dispatch claim).
3. Whether the slice→unit→cell fan-out produces sensible cell-per-unit grain (the utilisation argument's empirical premise).

A degraded sovereign mode (local large-model substitution under an offline constraint) is acknowledged but not yet specified.

## Dependencies

This framework depends on:

- **[AI Development Foundations](https://github.com/Hafeok/ai-development-foundations)** — the Execution Contract (which it conforms to), the Specification Framework and Two Pillars (which it references).
- **[product-framework](https://github.com/Hafeok/product-framework)** — the specification-pillar instantiation whose work units it executes and whose conformance ladder it supports.

It is depended on by:

- the separate **implementation repo** — the actual running Spark pipeline.

## License

Dual-licensed, matching the foundations and product-framework convention:

- **Documentation** (everything in `docs/`, `interface/`, `conformance/`, the glossary): [CC BY 4.0](LICENSE-DOCS). Build on, adapt, and redistribute with attribution.
- **Repository code and configuration** (if any is added): [Apache-2.0](LICENSE).
