# Zemi Design Documents

This directory captures the evolving design of **Zemi**, a programming language whose central thesis is making **application boundaries first-class**.

These documents are intentionally design-first. Implementation should follow once the core concepts are stable enough to guide compiler and tooling work.

## Reading order

| Document | Summary |
|----------|---------|
| [Vision](./01-vision.md) | Why the language exists; the thesis moment |
| [Governing Principles](./02-governing-principles.md) | Non-negotiable design rules and heuristics |
| [Representation and Meaning](./03-representation-and-meaning.md) | Raw types, interpretation, and application concepts |
| [Ports](./04-ports.md) | The central abstraction: boundaries with the outside world |
| [Transformation Pipelines](./05-transformation-pipelines.md) | How ports reuse ordinary computation with extra semantics |
| [Tooling Implications](./06-tooling-implications.md) | What becomes possible once the compiler knows about ports |
| [Open Questions](./07-open-questions.md) | Unresolved design decisions and next steps |

## Guiding sentence

> **Representation is not meaning.**

Everything else in these documents flows from that principle.

## Status

**Phase:** Early design exploration  
**Last updated:** 2025-06-30

Documents marked with open questions should be treated as proposals, not settled specification.
