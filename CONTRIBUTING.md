# Contributing

This is **Kiln**, a framework repository: normative, stable, implementation-free. The
running pipeline lives in a separate implementation repo that depends on this one.

## What changes go where

- **Conceptual / normative changes** (a building-block discharge, the interface
  schema, the binding-homogeneity invariant, a ladder-support claim) → here, via
  an RFC (see `rfcs/`).
- **Implementation changes** (executor, queue daemon, sandbox, dispatch code,
  model-binding configs) → the implementation repo, never here.

## The stability discipline

Dependencies point downward, toward stability. This framework depends on the
[AI Development Foundations](https://github.com/Hafeok/ai-development-foundations);
implementations depend on Kiln. Volatility is contained upward,
never pushed down. A change here that would force every implementation to change
is a signal to reconsider the change, not the implementations.

## Changing a normative document

1. Open an RFC in `rfcs/` (copy `0000-template.md`).
2. State the problem, the proposed change, and the conformance impact —
   specifically, whether any Execution Contract block's discharge changes, and
   whether the Work-Unit Interface schema changes (a schema change is breaking
   for every executor and must be justified against the producer-owns rule).
3. The three open validation questions (see `docs/00-overview.md`) are resolved
   by evidence from carrying a concrete specification through the chain — an RFC
   that claims to close one must cite that evidence, not argue it from design.

## License

Documents are CC BY 4.0 (`LICENSE-DOCS`); repository code/config is Apache-2.0
(`LICENSE`). By contributing you agree your contributions are licensed the same.
