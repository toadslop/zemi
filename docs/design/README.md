# Zemi Design Documents

This directory captures the evolving design of **Zemi**, a programming language whose central thesis is making **software architecture first-class**.

These documents are intentionally design-first. Implementation should follow once the core concepts are stable enough to guide compiler and tooling work.

## Reading order

| Document | Summary |
|----------|---------|
| [Vision](./01-vision.md) | Why the language exists; the thesis moment |
| [Governing Principles](./02-governing-principles.md) | Non-negotiable design rules and heuristics |
| [Representation and Meaning](./03-representation-and-meaning.md) | Raw types, interpretation, and application concepts |
| [Components and Libraries](./04-components-and-libraries.md) | Architectural units vs. reusable implementation |
| [Ports](./05-ports.md) | Connectors between components and the outside world |
| [Transformation Pipelines](./06-transformation-pipelines.md) | How ports reuse ordinary computation with extra semantics |
| [Tooling Implications](./07-tooling-implications.md) | What becomes possible once the compiler knows the architecture |
| [Open Questions](./08-open-questions.md) | Unresolved design decisions and next steps |

## Guiding sentence

> **Representation is not meaning.**

Everything else in these documents flows from that principle.

## Status

**Phase:** Early design exploration  
**Last updated:** 2025-06-30

Documents marked with open questions should be treated as proposals, not settled specification.
