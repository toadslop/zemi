# Vision

## Elevator pitch

**Zemi is a language that makes application boundaries first-class.**

It is not primarily "Rust with an effect system." It is a language that gives the compiler a vocabulary to recognize, analyze, and assist with the boundary between a program and its environment.

## The thesis

Most languages treat application boundaries as an **architectural convention**. Developers write books about Hexagonal Architecture, Clean Architecture, Ports and Adapters, and Onion Architecture — but the compiler has no model of what a port is. It cannot help enforce, visualize, lint, or optimize around boundaries because boundaries are invisible to the language.

Zemi proposes an analogy:

| Language | First-class concept | What the compiler gains |
|----------|--------------------|-------------------------|
| **C** | Memory is just bytes | Nothing about ownership; discipline is entirely conventional |
| **Rust** | Ownership is a language concept | Enforcement, visualization, linting, optimization, IDE support |
| **Zemi** | Ports are a language concept | Boundary analysis, wiring, plug replacement, architectural tooling |

The compiler is **not** trying to enforce "good architecture" in general. It is trying to make **one** architectural concept first-class:

> The boundary between a program and its environment.

That is a focused objective. Everything else — effects, raw representations, capabilities, library vs. executable wiring — should fall out as consequences rather than drive the design independently.

## What we are not claiming

We are **not** saying:

- Every application must have rich domain models
- One specific architectural style (hexagonal, onion, etc.) is mandatory
- The compiler judges whether domain concepts are *correct*

We **are** saying:

- Machine representations are never, by themselves, application concepts
- The transition from representation to meaning must be explicit and visible
- Application boundaries are important enough to deserve language-level recognition

## Consequences of the thesis

Once ports are first-class, several ideas become natural follow-ons:

| Concept | Role |
|---------|------|
| **Effects** | Ports communicate with the outside world; effects are what ports expose |
| **Raw values** | Ports carry uninterpreted representations across the boundary |
| **Capabilities** | Model access to ports |
| **Libraries** | Expose port *definitions* (what the application needs) |
| **Executables** | Decide how ports are *connected* (production vs. test adapters) |
| **Tooling** | Reason about ports because the compiler understands them |

## Historical analogy: Rust and ownership

Rust's borrow checker is famous, but the important innovation is not the checker itself — it is that **ownership became part of the language's model of the program**. Once the compiler knows ownership exists, enforcement and tooling follow.

Zemi aims for the same relationship with application boundaries. The innovation is semantic: teaching the compiler what a boundary *is*, not inventing new ways to compute.

## Design north star

> Don't invent new computation — invent new semantics.

Mapping, filtering, parsing, decoding, and validation already exist. Zemi's job is to teach the compiler: *this particular pipeline crosses the application's architectural boundary with the outside world.*

Successful language features often extend existing constructs with stronger meaning:

- A lambda is a function with lexical capture
- `async fn` is a function returning a future
- An `unsafe` block is a scope with relaxed safety rules
- A **port** is a transformation pipeline with architectural semantics

The syntax need not be revolutionary. The compiler's understanding of what that syntax *means* is the revolution.
