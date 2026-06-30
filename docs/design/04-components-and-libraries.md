# Components and Libraries

This document captures a major refinement of Zemi's architectural model: the distinction between **components** (architectural units) and **libraries** (reusable implementation). It generalizes ports from an application-only feature into a language-wide model of architectural composition.

## The problem with flat module systems

Current module systems flatten everything into one kind of thing: `module`.

Programmers do not think that way. When opening a project, experienced engineers already mentally classify things as:

- "This is a reusable utility."
- "This is an actual subsystem."

The compiler has no idea. A folder called `utils/` and a folder called `services/` are both just modules — organizational conventions with no semantic weight.

Zemi proposes two top-level kinds of module, each with different semantics:

| Kind | Role |
|------|------|
| **Component** | Participates in the architecture |
| **Library** | Reusable implementation; invisible to the architecture |

This is not richer in features. It is richer in **intent**.

## Physical analogy

Imagine opening a laptop. You do not see:

```
folder/
folder/
folder/
```

You see **components**: Motherboard, SSD, Battery, Cooling, Keyboard, Screen. These are machines — each has a purpose, dependencies, and an interface.

Zoom into the motherboard. Inside are capacitors, traces, resistors, ICs. Those are not components in the architectural sense. They are implementation details — gears, resistors, wires, bearings. Nobody asks "what does the resistor depend on?" It is just a resistor.

Software is the same:

```
Application (root component)
    │
    ├── Component (HTTP server)
    │       └── libraries, helpers, algorithms (internal)
    │
    ├── Component (Authentication)
    │       └── libraries, helpers, algorithms (internal)
    │
    └── Component (Repository)
            └── libraries, helpers, algorithms (internal)

Library (Geometry)  ──used by──▶  multiple components
Library (UTF-8)
Library (SHA256)
```

## Components

A **component** is an architectural unit — a deployable machine that participates in system composition. It is not merely a namespace.

A component has:

- a **purpose** (what job it does in the system),
- **dependencies** on other components (through ports, not through implementation imports),
- an **interface** (its ports),
- potentially **subcomponents** (nested architectural structure).

Examples: HTTP server, authentication service, order processor, cache, database adapter, CLI.

### Components are closed

A component exists to do a job. It exposes **ports**, not implementation.

Inside a component, developers are free to write ordinary code:

- helper functions,
- internal data structures,
- parsers and algorithms,
- private submodules,
- component-local utilities that are not exported.

All of that belongs to the component. It is not intended to be imported piecemeal elsewhere — analogous to opening a power supply: inside are custom brackets, wires, and PCBs, but none of those are part of the power supply's public interface.

**Rule:** Components do not export things to be reused outside themselves. If code is genuinely shared between components, it belongs in a library.

### What only components can do

If components are first-class, then only components may:

- declare ports,
- require ports (dependencies on other components),
- participate in wiring,
- appear in architecture diagrams.

Everything else is invisible to the architecture. Libraries disappear into the implementation details of component nodes.

### How far do ports go?

A question we have wrestled with repeatedly: "How far down should ports go?"

**Answer:** Exactly as far as components go.

Once inside a component's implementation, you are just writing code — using ordinary libraries, composing values, running algorithms. The compiler stops thinking architecturally. Ports are boundaries between components (and between a component and the outside world), not between every internal function.

## Libraries

A **library** is reusable implementation. It is not architectural.

Libraries provide building blocks used to construct components. They do not own architecture. They do not communicate with the outside world. They simply export things to be reused.

Examples: Math, Geometry, UTF-8, SHA256, EmailAddress, UUID, Color, Matrix, Collections, Crypto.

### Libraries are open

A library exists to provide reusable building blocks:

```
library Geometry {
    export Vector
    export Matrix
    export Quaternion
    export Transform
}
```

Any number of components may use a library. Libraries have no ports and no dependencies on other components.

### The semantic distinction

This is not merely syntactic:

| | Libraries | Components |
|---|-----------|------------|
| **Export** | Functions, types, traits, algorithms | Ports |
| **Purpose** | Reusable implementation | Architectural unit |
| **Wiring** | Cannot participate | Connected through ports |
| **Architecture graph** | Invisible (implementation detail) | Visible (node) |

Libraries export things to be **called**. Components export **ports** to be **wired**.

## Applications as root components

This model generalizes the earlier "application boundary" framing.

**Before:** Applications have ports.

**After:** Components have ports. Applications are components.

An **application** is not a fundamentally different kind of thing. It is simply the **root component** — the outermost architectural unit that connects to the outside world.

```
            Outside
               │
        ┌──────────────┐
        │ Application  │  ← root component
        └──────┬───────┘
               │
       ┌───────┴────────┐
       │                │
┌─────────────┐  ┌─────────────┐
│  Component  │  │  Component  │
└──────┬──────┘  └──────┬──────┘
       │                │
   Libraries         Libraries
```

And components may nest recursively:

```
Component
    │
 ┌──┴────────┐
 │           │
Component  Component
```

There is no conceptual difference between an application and an inner component. The application is the outermost one.

### Eliminating special cases

This generalization removes several awkward special cases:

| Old question | New answer |
|--------------|------------|
| Why are ports only on binaries? | Ports are on components; binaries are root components |
| Why can't libraries have ports? | Libraries are not architectural; only components have ports |
| Why are effects different in binaries? | Effects flow through ports; all components use the same model |
| Library vs. executable vs. component crate? | Three crate kinds generalize into two module kinds + root |

A binary is not special because of ports. It is special because it is the **root component** — the deployment entry point.

## Declaration model

Developers must declare whether a unit is a `component` or a `library`. This is a semantic choice, not an organizational one.

```
library Geometry { ... }

component HttpServer { ... }

component Authentication { ... }
```

### Internal structure

A component may contain arbitrary implementation code internally — including utility-like constructs that are private to that component and not exported. The declaration boundary is at the **unit** level (component or library), not at every file or function.

If reusable code should be shared across components, extract it to a library. The language gently nudges toward this refactoring:

```
Before (architectural erosion):

Component A ──imports internal helper from──▶ Component B

After (clean architecture):

        Library
        /     \
       /       \
Component A   Component B
```

Component A cannot import Component B's implementation. If A needs B's behavior, it talks through B's ports. If they genuinely share implementation, that implementation belongs in a library.

## Architecture as a graph

If components are first-class, the compiler can understand the application's architecture as a **graph**:

- **Nodes:** components
- **Edges:** ports (connections between components and to the outside world)
- **Invisible:** libraries (implementation inside nodes)

That graph could power tooling without inventing new concepts:

- automatic architecture diagrams,
- dependency analysis between components,
- dead component detection,
- cycle checking across component wiring,
- port wiring validation,
- test harness generation,
- deployment metadata.

None of this requires heuristics or naming conventions. It falls out of the language knowing what the architecture actually is.

## Design philosophy

### Few powerful distinctions

Resist classifying everything. Rust succeeded in part by introducing a few powerful distinctions rather than a complete ontology of software.

Zemi aims for exactly **two** top-level kinds of module:

1. **Component** — architectural unit that connects through ports
2. **Library** — implementation building block that does not

That is an extremely understandable mental model. It is remarkably close to how experienced engineers already think about systems. The compiler makes that distinction explicit and gives it semantic weight, rather than inventing a new way to organize code.

### Generalization over addition

The goal is not to add arbitrary new concepts. It is to **generalize** what already exists:

- Ports were at the application boundary → ports are on every component
- Library/executable/component crates → component and library module kinds
- Application as special case → application as root component

> Eliminate special cases by finding the more general abstraction.

### Intent over convention

Today, hexagonal architecture, clean architecture, and ports-and-adapters are conventions enforced by code review and team discipline. Zemi asks:

> Can the compiler understand architecture as well as it understands ownership?

The language does not try to make arbitrary things impossible. It tries to make **architectural intent explicit** — turning conventions into something statically checked.

## Relationship to other concepts

| Concept | Relationship |
|---------|--------------|
| [Ports](./05-ports.md) | Declared on components; connect components to each other and to the outside world |
| [Representation and Meaning](./03-representation-and-meaning.md) | Interpretation happens at port boundaries between components |
| [Transformation Pipelines](./06-transformation-pipelines.md) | Port bodies reuse ordinary pipeline syntax |
| [Tooling](./07-tooling-implications.md) | Architecture graph, wiring validation, diagrams |
| Effects | Operations performed through ports |
| Capabilities | Access to ports within a component |

## Open questions

See [Open Questions](./08-open-questions.md) for unresolved details: exact syntax (`component` / `library` keywords), crate-to-module mapping, subcomponent declaration, visibility rules for component interiors, and wiring mechanics.
