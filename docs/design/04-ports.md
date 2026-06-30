# Ports

Ports are the central abstraction of Zemi. This document captures what we believe a port *is*, what it *is not*, and how it relates to other concepts.

## Definition (working)

> A **port** is a compiler-recognized transformation boundary between the application and its environment.

A port is **not** a new computational model. It is an ordinary transformation pipeline (or equivalent composition of functions/stages) with additional **architectural semantics** that the compiler understands.

## What the compiler knows about a port

When code is declared as a port, the compiler gains semantic information it cannot derive from an ordinary pipeline:

| Property | Meaning |
|----------|---------|
| External origin | This pipeline begins outside the application |
| Boundary role | This is an ingress or egress connector |
| Interpretation site | Raw representations may enter; application concepts must exit (ingress) or the reverse (egress) |
| Wiring target | Executables connect adapters to this port |
| Substitution site | Test doubles replace the adapter, not application logic |
| Tooling anchor | Diagrams, audits, and analysis start here |

An ordinary iterator pipeline:

```
bytes |> parse_http |> parse_json |> parse_user
```

The compiler sees: `Iterator<Bytes>` becoming `Iterator<User>`. It has no architectural model.

The same pipeline inside a port declaration tells the compiler everything in the table above.

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

Effects exist because ports communicate with the outside world. Effects are not the same thing as ports.

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

- **Ingress:** outside → application (interpret Raw into domain concepts)
- **Egress:** application → outside (encode domain concepts into Raw for emission)

Symmetry and naming are open questions. Some ports may be bidirectional.

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
2. Must not leak Raw into the application interior (ingress)
3. Must not accept uninterpreted application concepts at egress without encoding
4. Are visible to tooling and dependency wiring
5. Participate in plug replacement for testing
6. May carry metadata (category, audit tags, etc.)

## Plug replacement

Production and testing differ only in what is connected behind the port:

```
Production          Testing
──────────          ───────
App                 App
 │                   │
Port                Port
 │                   │
Real adapter        Test adapter
```

The application and port *definition* stay the same. Executables (or runtime wiring) choose the adapter.

This is intentionally stronger than scattered mock frameworks: the compiler knows exactly where substitution is valid.

## Libraries vs. executables (draft)

| Artifact | Responsibility |
|----------|----------------|
| **Library crate** | Defines ports the application needs; exposes the application's boundary *interface* |
| **Executable crate** | Wires concrete adapters to those ports for a specific deployment (prod, test, dev) |

Exact module/crate model is an open question.

## Formal status: unresolved

The teammate conversation ends with the right question:

> What is a port, formally?
>
> - Is it a type?
> - Is it a module?
> - Is it an interface?
> - Is it an object?
> - Is it a capability?
> - Is it a compiler-recognized declaration?

**Current leaning:** a port is primarily a **compiler-recognized declaration** — a named transformation boundary with attached metadata and constraints — that may *manifest as* types, capabilities, or wiring targets in the implementation.

See [Open Questions](./07-open-questions.md) for the formalization track.

## Sketch: port declaration anatomy (illustrative, not spec)

```
port <Name> {
    <source or sink declaration>   // external endpoint
    <transformation pipeline>    // ordinary syntax, architectural semantics
    <metadata?>                  // category, effects exposed, etc.
}
```

Syntax is deliberately illustrative. Final syntax must pass the "minimal unless it buys something" test.
