# zemi

A programming language that makes **software architecture first-class**.

> Representation is not meaning.

## Status

**POC design complete** — ready to begin compiler implementation. Core vision and principles are documented; the POC track narrows scope to a provable first milestone.

| Track | Location | Status |
|-------|----------|--------|
| Core design | [docs/design/](./docs/design/) | Exploration (docs 01–08) |
| POC plan | [docs/poc/](./docs/poc/) | Ready for Phase 0 |
| Reference program | [example/](./example/) | Syntax target for parser |

## Design documents

### Core

| Document | Topic |
|----------|-------|
| [Vision](./docs/design/01-vision.md) | Thesis and elevator pitch |
| [Governing Principles](./docs/design/02-governing-principles.md) | Core design rules |
| [Representation and Meaning](./docs/design/03-representation-and-meaning.md) | Raw vs. application types |
| [Components and Libraries](./docs/design/04-components-and-libraries.md) | Architectural units vs. reusable implementation |
| [Ports](./docs/design/05-ports.md) | Connectors between components |
| [Transformation Pipelines](./docs/design/06-transformation-pipelines.md) | Ports + ordinary pipelines |
| [Tooling Implications](./docs/design/07-tooling-implications.md) | What the compiler model enables |
| [Open Questions](./docs/design/08-open-questions.md) | Unresolved decisions |

### POC (implementation-ready)

| Document | Topic |
|----------|-------|
| [POC Specification](./docs/design/09-poc-spec.md) | Scope, success criteria, reference program |
| [POC Design Decisions](./docs/design/10-poc-design-decisions.md) | Provisional answers for the POC |
| [Implementation Plan](./docs/design/11-implementation-plan.md) | Phased build plan |
