# Representation and Meaning

## The core distinction

**Raw types** are representation types. They describe *how* data is stored, not *what* data is.

**Application types** carry semantic meaning. They describe *what* a value represents in the problem domain.

| Category | Answers | Examples |
|----------|---------|----------|
| Representation | How many bits? What encoding? What layout? | `u32`, `usize`, `String`, `Vec<u8>`, `f64` |
| Meaning | What does this value represent? | `UserId`, `OrderTotal`, `FileBuffer`, `SourceFile` |

## Raw is uninterpreted

Avoid describing Raw as "unvalidated." A value is Raw because it has not been **interpreted**, not because it is invalid.

```
"42"     → valid UTF-8, still Raw (age? port? user ID? ZIP?)
u64      → valid integer, still Raw until named (OrderId? Timestamp?)
```

The compiler does not know what these represent until explicit interpretation occurs.

## Where interpretation happens

Interpretation is the transition from representation to meaning. In Zemi, this transition is expected to happen **at ports** — the architectural boundary — before values enter application interior.

```
Outside World
    ↓  (Raw representations)
Port pipeline (interpretation stages)
    ↓  (Application concepts)
Application interior
```

The application interior manipulates meaningful concepts. Raw representations should not leak inward without the compiler flagging it.

## Examples across domains

### Hex editor

A hex editor's job involves bytes, but the application does not reason about arbitrary `Vec<u8>`. It reasons about:

- An opened file
- Current selection
- Clipboard state
- Undo history

The central buffer might be:

```
struct FileBuffer {
    bytes: Vec<u8>   // representation, encapsulated
}
```

`FileBuffer` is the concept. `Vec<u8>` is how it is stored.

### Compiler

A compiler does not manipulate "strings." It manipulates a progression of concepts:

```
SourceFile → TokenStream → SyntaxTree → TypedProgram
```

At every stage, representation is wrapped in a type that carries semantic meaning for that stage.

## Design rule (draft)

> Raw types may appear at port boundaries and within representation-holding fields. Application logic operates on interpreted types.

Exact enforcement rules (type system details, escape hatches, etc.) are **open questions** — see [Open Questions](./07-open-questions.md).

## Relationship to ports

Ports are where the outside world's representations enter the application. A port pipeline progressively interprets Raw into application concepts:

```
TCP bytes
    |> decode UTF-8
    |> parse HTTP
    |> parse JSON
    |> deserialize User
    |> authenticate
    |> authorize
    → Application
```

Each stage may narrow or enrich meaning. The compiler's interest is not the specific transforms — it is that **this pipeline is the boundary** and that Raw does not escape it uninterpreted.

## What the compiler guarantees (intent)

The compiler's role is **not** to judge whether domain concepts are correct. It is to ensure:

1. The transition from representation to meaning is **explicit**
2. That transition is **visible** to tooling
3. Boundaries cannot be crossed **accidentally** or **invisibly**

This mirrors Rust's guarantee around ownership: explicit, visible, not accidentally bypassed — not "always correct."

## Non-goals

- Mandating rich domain-driven design everywhere
- Rejecting programs whose domain is inherently low-level (hex editors, codecs, etc.)
- Requiring validation at every boundary (interpretation ≠ validation)
