# Ports

Ports are the connectors through which components communicate — with each other and with the outside world. This document captures what we believe a port *is*, what it *is not*, and how it relates to other concepts.

See [Components and Libraries](./04-components-and-libraries.md) for the architectural model ports participate in.

## Definition (working)

> A **port** is a compiler-recognized transformation boundary on a **component**.

A port is **not** a new computational model. It is an ordinary transformation pipeline (or equivalent composition of functions/stages) with additional **architectural semantics** that the compiler understands.

Ports belong to components, not to libraries. An application is a component (the root component); its ports connect to the outside world. Inner components have ports that connect to sibling components or to the application's exterior.

## What the compiler knows about a port

When code is declared as a port, the compiler gains semantic information it cannot derive from an ordinary pipeline:

| Property | Meaning |
|----------|---------|
| Component boundary | This pipeline crosses a component's architectural edge |
| External or inter-component origin | This pipeline begins outside the component (ingress) or terminates outside it (egress) |
| Interpretation site | Raw representations may enter; meaningful concepts must exit (ingress) or the reverse (egress) |
| Wiring target | Adapters connect to this port at build or deploy time |
| Substitution site | Test doubles replace the adapter, not component logic |
| Tooling anchor | Architecture diagrams, audits, and analysis use ports as edges |

An ordinary iterator pipeline:

```
bytes |> parse_http |> parse_json |> parse_user
```

The compiler sees: `Iterator<Bytes>` becoming `Iterator<User>`. It has no architectural model.

The same pipeline inside a port declaration on a component tells the compiler everything in the table above.

## Ports and component depth

Ports exist at the **component** level — no deeper.

Inside a component's implementation, developers write ordinary code: libraries, algorithms, helper functions, private utilities. The compiler does not think architecturally below the component boundary.

This answers a recurring design question: "How far down should ports go?" Exactly as far as components go.

## Ports vs. effects

**Do not conflate ports with effects.**

- A **port** is an architectural connector (like a USB port)
- An **effect** is an operation performed *through* that connector (like a protocol on USB)

One port, many effects:

```
HTTP Port
  ├── GET
  ├── POST
  ├── PUT
  ├── DELETE
  └── streaming
```

Effects exist because ports communicate across component boundaries. Effects are not the same thing as ports.

## Ports vs. iterators

**Do not make ports literally iterators.**

Iterators produce zero or more values lazily. Not every port fits that model:

| Port kind | Why it may not be an iterator |
|-----------|------------------------------|
| HTTP request/response | Single round-trip, not a lazy sequence |
| Filesystem write | Mutation, not production |
| Database transaction | Scoped operation with commit/rollback |
| Timer callback | Event-driven, not pull-based |
| GUI event | Push-based input |

Instead:

> A port is a compiler-recognized transformation boundary. It may *use* iterators, streams, generators, or function composition internally — but it is not defined as any one of those.

## Ingress and egress

Ports likely have direction:

- **Ingress:** outside → component (interpret Raw into meaningful concepts)
- **Egress:** component → outside (encode meaningful concepts into Raw for emission)

Ports between components follow the same model: one component's egress may connect to another's ingress. Symmetry and naming are open questions. Some ports may be bidirectional.

## The block-decorator model

Following the Rust `async` / `unsafe` block pattern, a port may be expressed as a decorated pipeline:

```
port HttpIngress {

    source tcp

    tcp
        |> http
        |> json
        |> user

}
```

Nothing inside the block uses special syntax. The `port` keyword (or equivalent declaration) gives the compiler architectural semantics.

### Constraints ports add (beyond ordinary pipelines)

Ordinary pipelines can do anything. Port pipelines have rules — architectural properties, not computational ones:

1. Must begin at an external source (ingress) or terminate at an external sink (egress)
2. Must not leak Raw into the component interior (ingress)
3. Must not accept uninterpreted concepts at egress without encoding
4. Are visible to tooling and component wiring
5. Participate in plug replacement for testing
6. May carry metadata (category, audit tags, etc.)
7. May only be declared on **components**, never on libraries

## Plug replacement

Production and testing differ only in what is connected behind the port:

```
Production          Testing
──────────          ───────
Component           Component
 │                   │
Port                Port
 │                   │
Real adapter        Test adapter
```

The component and port *definition* stay the same. Wiring (at the root component or deployment level) chooses the adapter.

This is intentionally stronger than scattered mock frameworks: the compiler knows exactly where substitution is valid.

## Root component and wiring

The **root component** (typically the application binary) is where external adapters connect. Inner components wire to each other through their ports; the root component's ports connect to the outside world.

| Role | Responsibility |
|------|----------------|
| **Component** | Declares ports; contains implementation (using libraries internally) |
| **Root component** | Outermost component; connects to external adapters for deployment |
| **Library** | Provides reusable code; no ports; invisible to architecture |

Exact wiring mechanics (compile-time, link-time, runtime) are open questions — see [Open Questions](./08-open-questions.md).

## Formal status: unresolved

The right formal question:

> What is a port, formally?
>
> - Is it a type?
> - Is it a module?
> - Is it an interface?
> - Is it an object?
> - Is it a capability?
> - Is it a compiler-recognized declaration?

**Current leaning:** a port is primarily a **compiler-recognized declaration** on a component — a named transformation boundary with attached metadata and constraints — that may *manifest as* types, capabilities, or wiring targets in the implementation.

See [Open Questions](./08-open-questions.md) for the formalization track.

## Sketch: port declaration anatomy (illustrative, not spec)

```
component <Name> {

    port <PortName> {
        <source or sink declaration>   // external or inter-component endpoint
        <transformation pipeline>      // ordinary syntax, architectural semantics
        <metadata?>                    // category, effects exposed, etc.
    }

    // ordinary implementation using libraries
}
```

Syntax is deliberately illustrative. Final syntax must pass the "minimal unless it buys something" test.
