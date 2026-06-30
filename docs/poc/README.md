# POC Track

Documents and artifacts for the Zemi proof of concept — proving that **software architecture can be first-class in the compiler**.

## Start here

| Step | Document | Question it answers |
|------|----------|---------------------|
| 1 | [POC Specification](../design/09-poc-spec.md) | What are we building? What does success look like? |
| 2 | [POC Design Decisions](../design/10-poc-design-decisions.md) | How do we resolve open questions for the POC? |
| 3 | [Implementation Plan](../design/11-implementation-plan.md) | In what order do we build it? |

## Prerequisites

Read the core design sequence first if you are new to Zemi:

1. [Vision](../design/01-vision.md)
2. [Governing Principles](../design/02-governing-principles.md)
3. [Components and Libraries](../design/04-components-and-libraries.md)
4. [Ports](../design/05-ports.md)

## Reference program

Concrete syntax lives in [`../../example/`](../../example/). These files are **design artifacts** until the compiler exists; they define the target input for Phase 1 parsing.

| File | Role |
|------|------|
| `example/app.zemi` | Root component + subcomponent |
| `example/lib/json.zemi` | Shared library |
| `example/lib/http.zemi` | Shared library |
| `example/wiring/prod.zemi` | Production adapter wiring |
| `example/wiring/test.zemi` | Test adapter wiring |

## Status

| Phase | Status |
|-------|--------|
| Design (docs 09–11) | **Complete** |
| Phase 0: Bootstrap | Not started |
| Phase 1: Parse | Not started |
| Phase 2: IR | Not started |
| Phase 3: Semantics | Not started |
| Phase 4: Tooling CLI | Not started |
| Phase 5: Reference example | Not started |

Update this table as implementation progresses.

## Walkthrough

Created during Phase 5: [walkthrough.md](./walkthrough.md) (placeholder — document how to read `zemi components graph` output).
