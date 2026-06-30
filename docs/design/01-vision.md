# Vision

## Elevator pitch

**Zemi is a language that makes software architecture first-class.**

It is not primarily "Rust with an effect system." It is a language that gives the compiler a vocabulary to recognize, analyze, and assist with how programs are composed — components, ports, boundaries, and wiring — not just how they compute.

## The thesis

Most languages treat software architecture as **convention**. Developers write books about Hexagonal Architecture, Clean Architecture, Ports and Adapters, and Onion Architecture — but the compiler has no model of components, ports, or boundaries. It cannot help enforce, visualize, lint, or optimize around architecture because architecture is invisible to the language.

Zemi proposes an analogy:

| Language | First-class concept | What the compiler gains |
|----------|--------------------|-------------------------|
| **C** | Memory is just bytes | Nothing about ownership; discipline is entirely conventional |
| **Rust** | Ownership is a language concept | Enforcement, visualization, linting, optimization, IDE support |
| **Zemi** | Architecture is a language concept | Component graphs, port wiring, boundary analysis, architectural tooling |

The compiler is **not** trying to enforce "good architecture" in general. It is trying to make architectural composition first-class:

> Components connect through ports. Applications are root components. Libraries are invisible implementation.

That is a focused objective. Everything else — effects, raw representations, capabilities, wiring — should fall out as consequences rather than drive the design independently.

## What we are not claiming

We are **not** saying:

- Every application must have rich domain models
- One specific architectural style (hexagonal, onion, etc.) is mandatory
- The compiler judges whether domain concepts are *correct*

We **are** saying:

- Machine representations are never, by themselves, application concepts
- The transition from representation to meaning must be explicit and visible
- Architectural boundaries are important enough to deserve language-level recognition

## Consequences of the thesis

Once components and ports are first-class, several ideas become natural follow-ons:

| Concept | Role |
|---------|------|
| **Components** | Architectural units that declare ports and connect to each other |
| **Libraries** | Reusable implementation; invisible to the architecture graph |
| **Applications** | Root components — the outermost architectural unit |
| **Ports** | Connectors between components and to the outside world |
| **Effects** | Operations performed through ports |
| **Raw values** | Uninterpreted representations crossing port boundaries |
| **Capabilities** | Model access to ports within a component |
| **Tooling** | Reason about architecture because the compiler understands the component graph |

See [Components and Libraries](./04-components-and-libraries.md) for the full model.

## Historical analogy: Rust and ownership

Rust's borrow checker is famous, but the important innovation is not the checker itself — it is that **ownership became part of the language's model of the program**. Once the compiler knows ownership exists, enforcement and tooling follow.

Zemi aims for the same relationship with software architecture. The innovation is semantic: teaching the compiler what components and boundaries *are*, not inventing new ways to compute.

## Design north star

> Don't invent new computation — invent new semantics.

Mapping, filtering, parsing, decoding, and validation already exist. Zemi's job is to teach the compiler: *this unit is a component; this pipeline is a port; this code is reusable implementation.*

Successful language features often extend existing constructs with stronger meaning:

- A lambda is a function with lexical capture
- `async fn` is a function returning a future
- An `unsafe` block is a scope with relaxed safety rules
- A **component** is a module that participates in architectural composition
- A **library** is a module that provides reusable implementation
- A **port** is a transformation pipeline with architectural semantics on a component

The syntax need not be revolutionary. The compiler's understanding of what that syntax *means* is the revolution.
