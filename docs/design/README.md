# Zemi Design Documents

This directory captures the evolving design of **Zemi**, a programming language whose central thesis is making **software architecture first-class**.

These documents are intentionally design-first. The **POC track** (documents 09–11) narrows scope enough to begin implementation while full-language questions remain open in document 08.

## Reading order

### Core design

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

### POC track (implementation-ready)

| Document | Summary |
|----------|---------|
| [POC Specification](./09-poc-spec.md) | Scope, success criteria, reference program, IR requirements |
| [POC Design Decisions](./10-poc-design-decisions.md) | Provisional answers to high-priority open questions |
| [Implementation Plan](./11-implementation-plan.md) | Phased build plan, repo layout, testing strategy |

See also: [POC track index](../poc/README.md) and the reference program in [`example/`](../../example/).

## Guiding sentence

> **Representation is not meaning.**

Everything else in these documents flows from that principle.

## Status

**Phase:** POC design complete — ready to begin implementation (Phase 0)  
**Last updated:** 2025-06-30

Documents 01–08 are exploratory; documents 09–11 are POC-scoped and govern implementation work until graduated into the main decision log.
