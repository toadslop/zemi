# Governing Principles

These principles guide design decisions. When two options conflict, the higher-priority principle wins unless there is a documented reason to override it.

## 1. Representation is not meaning

> Primitive types describe representation. Ports convert representations into meaningful concepts. The application manipulates meaningful concepts.

This is the language's equivalent of Rust's "ownership is explicit." It is not a guarantee of correctness — it is a guarantee that a crucial design decision cannot happen accidentally or invisibly.

**Implication:** `String`, `u64`, and `Vec<u8>` answer questions about bits, encoding, and layout. They never answer what a value *represents*. Application types answer the latter.

## 2. Don't invent new computation — invent new semantics

Reuse ordinary transformation mechanisms (functions, pipelines, composition). Add compiler-recognized meaning where architecture requires it.

**Implication:** Avoid parallel mini-languages. If `map`, `|>`, or pipeline syntax exists for ordinary code, ports should reuse it — not introduce a separate transformation vocabulary.

## 3. Extend existing semantics rather than introduce parallel mechanisms

New constructs should decorate or constrain familiar ones:

```
{ ... }           → ordinary scope
unsafe { ... }    → scope with relaxed safety rules
async { ... }     → scope participating in async execution
port { ... }      → pipeline crossing an architectural boundary
```

**Implication:** A programmer who understands blocks, functions, and pipelines should not need to learn an entirely separate "port language."

## 4. Ports are not effects

A port is an architectural connector. A single port may expose many effects.

| Port | Example effects through that port |
|------|-----------------------------------|
| HTTP Port | GET, POST, PUT, DELETE, streaming |
| Database Port | query, transaction, migrate |
| Clock | now, sleep |
| Randomness | next_u64, shuffle |

Think of a USB connector: one physical port, many protocols.

**Implication:** Do not collapse "port" and "effect" into one concept. Effects are what you *do through* a port; the port is the *boundary*.

## 5. Plug replacement, not mocking

Testing replaces the adapter behind a port — not the application logic around it.

```
Production                    Testing
──────────                    ───────
Application                   Application
     │                             │
 HTTP Port                     HTTP Port
     │                             │
 Internet                      Fake Server
```

Nothing else changes. This is stronger than conventional "mocking" because the compiler knows where substitution is valid.

## 6. Raw is uninterpreted, not unvalidated

A value is Raw not because it failed validation, but because it has not been *interpreted*.

- `"42"` is valid UTF-8 and still Raw — it could be an age, port number, user ID, or ZIP code
- `u64` is a valid integer and still Raw until wrapped as `OrderId`, `Timestamp`, or `MemoryAddress`

**Implication:** Prefer "interpretation" over "validation" as the conceptual foundation for boundary crossing.

## 7. Strong opinions with clear reasons

When someone asks "Why can't I just use `String` in my domain logic?", the answer is not "strings are bad." It is:

> Because `String` tells the compiler nothing about what your program *means*.

Same style as Rust's answer to shared mutable state: not arbitrary restriction, but a belief that semantic interpretation is fundamental.

## 8. Minimal syntax unless it buys something

Invert the design process:

- **Don't ask first:** "What syntax should ports have?"
- **Ask first:** "What can't an ordinary function pipeline express?"

If the answer is only visibility, wiring, metadata, and compiler analysis — then a port is a named, compiler-recognized pipeline with architectural semantics attached. No second transformation model required.

## 9. Opinionated about boundaries, neutral about domain style

The language enforces that interpretation happens explicitly at boundaries. It does not mandate how rich or simple domain models must be.

Even a hex editor manipulates `FileBuffer { bytes: Vec<u8> }`, not bare `Vec<u8>`. Even a compiler progresses through `SourceFile` → `TokenStream` → `SyntaxTree` → `TypedProgram`. Representation is always wrapped in a concept that carries meaning.

## Heuristic checklist

When evaluating a proposed feature, ask:

1. Does it serve the boundary-between-program-and-environment thesis?
2. Can it reuse existing transformation/composition syntax?
3. Does it add semantics without duplicating computation models?
4. Is the distinction between port, effect, and representation clear?
5. Does it make implicit architectural decisions visible to the compiler?

If a proposal fails most of these checks, it likely belongs in a library rather than the language core.
