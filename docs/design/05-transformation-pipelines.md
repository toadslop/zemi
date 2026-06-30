# Transformation Pipelines

Ports reuse the language's ordinary transformation model. This document explains how pipelines relate to ports and why they must feel seamless — not like a parallel feature.

## The insight

Entry points into an application are transformation pipelines that progressively convert external representations into meaningful domain concepts before data reaches the interior.

This is computationally familiar:

```
A → B → C → D → E
```

Function composition, `map` chains, and pipe operators all express this. **The operations are not new.**

What is new is the compiler knowing that a *specific* pipeline is an application boundary.

## Iterator analogy (with limits)

A port pipeline resembles an iterator pipeline:

```
TCP Bytes
    |> decode UTF-8
    |> parse HTTP
    |> parse JSON
    |> deserialize User
    → Application
```

Conceptually similar to:

```
bytes.map(parse_http).map(parse_json).map(parse_user)
```

But ports are **not** iterators. See [Ports — Ports vs. iterators](./04-ports.md#ports-vs-iterators).

Better analogy: **Unix pipes**

```
cat | grep | sort | uniq
```

Each stage transforms a stream. A port is the compiler's name for "this pipe chain is an architectural connector."

## Async analogy

Rust's `async` keyword does not invent new computation. It marks that a function participates in asynchronous execution. The compiler gains semantic information.

Similarly, a port declaration does not invent new transforms. It marks that a pipeline crosses an architectural boundary.

| Construct | Underlying mechanism | Added semantics |
|-----------|---------------------|---------------|
| `async { }` | Ordinary control flow | Participates in async execution |
| `unsafe { }` | Ordinary scope | Relaxed safety rules |
| `port { }` | Ordinary pipeline | Crosses application boundary |

## One transformation model

**Avoid inventing a separate port mini-language.**

If the language has:

```
|>          // pipe operator
map()       // collection transform
filter()    // collection transform
```

Then port bodies should use those same tools:

```
port http {

    tcp
        |> http
        |> json
        |> user

}
```

Not a parallel vocabulary:

```
// Avoid unless ordinary syntax truly cannot express this
port http {
    receive(...)
    decode(...)
    transform(...)
    adapt(...)
}
```

## What ordinary pipelines cannot express

If the *only* things ports add beyond ordinary pipelines are:

- Visibility and module boundaries
- Wiring (which adapter connects here)
- Metadata for tooling
- Compiler analysis (boundary graph, effect reachability, etc.)
- Constraints (no Raw leakage, external source required)

…then a port **is** an ordinary pipeline plus architectural semantics. No second computation model needed.

## Integration requirements

The port system and the standard transformation/iterator framework must integrate elegantly:

1. **Same syntax** inside and outside ports for transform steps
2. **Same standard library** functions usable in port pipelines
3. **Additional constraints** applied only at port boundaries (compiler-enforced)
4. **No duplication** of map/filter/fold concepts under different names

Programmers should not feel they are learning two languages — one for "normal code" and one for "boundary code."

## Port-specific rules on ordinary transforms

When a pipeline is declared as a port, the compiler attaches rules ordinary pipelines do not have:

| Rule | Rationale |
|------|-----------|
| External source/sink required | Defines where the boundary attaches |
| Raw cannot leak inward | Representation is not meaning |
| Terminates in application types (ingress) | Interpretation completes at boundary |
| Named and discoverable | Tooling and wiring |
| Mockable/substitutable adapter slot | Plug replacement |

These are **architectural** properties. They do not require new transform *functions*.

## Staged interpretation

Port pipelines encode progressive interpretation — the journey from Raw to meaning:

```
Stage          Type level (illustrative)
─────          ─────────────────────────
tcp            Raw bytes from network
utf8           Validated text (still may be uninterpreted)
json           Structured data (still generic)
user           Application concept
authenticated  Enriched application concept
```

Each stage is an ordinary transform. The port wrapper tells the compiler the entire chain is boundary-critical.

## Design heuristic

> New language constructs should extend existing semantics rather than introduce parallel mechanisms.

When pipeline design conflicts arise, prefer:

1; Reusing existing pipe/map/composition syntax
- Adding compiler checks at port boundaries
- Keeping transform functions in shared standard library

Over:

- Port-specific transform keywords
- Separate port-only combinators
- A distinct "adapter DSL"

## Open questions

- Exact pipeline syntax (`|>` vs. method chains vs. both)
- Streaming vs. batch semantics at boundaries
- Error handling in port pipelines (where failures live architecturally)
- Whether egress pipelines are symmetric or use different combinators

See [Open Questions](./07-open-questions.md).
