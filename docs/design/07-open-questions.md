# Open Questions

This document tracks unresolved design decisions. Items here are intentionally **not** settled — they are the focus of continued design work before implementation.

## Formal definition of a port

**Priority: high**

What is a port, formally?

| Candidate | Pros | Cons |
|-----------|------|------|
| **Compiler-recognized declaration** | Matches block-decorator model; minimal syntax | Need runtime representation for wiring |
| **Type** | Strong static reasoning | May not fit request/response, callbacks |
| **Module** | Natural boundary in code organization | Conflates file structure with architecture |
| **Interface / trait** | Familiar OOP/idiom | May duplicate pipeline body elsewhere |
| **Capability** | Good for access control | Capabilities may be finer-grained than ports |
| **Object** | Runtime polymorphism for adapters | Unclear compile-time analysis story |

**Current leaning:** declaration-first, generating types/capabilities/wiring hooks as needed — but this needs explicit design.

## Type system: Raw vs. interpreted

**Priority: high**

- Is Raw a type kind, marker trait, or namespace?
- Can Raw appear anywhere, or only at port boundaries?
- How do newtypes (`OrderId(u64)`) relate to interpretation?
- Are there degrees of interpretation, or a binary Raw/Domain split?
- What are the escape hatches, if any?

## Port syntax

**Priority: medium** (depends on pipeline syntax)

- Keyword: `port` block vs. attribute vs. declaration statement?
- Ingress/egress: separate keywords, direction annotation, or inferred?
- Source/sink syntax: `source tcp`, adapter references, capability grants?

Illustrative only — not spec:

```
port HttpIngress {
    source tcp
    tcp |> http |> json |> user
}
```

## Pipeline syntax

**Priority: medium**

- Pipe operator `|>` vs. method chains vs. both?
- Relationship to async streams and generators?
- Standard library naming and placement of transform functions

Must integrate seamlessly with port bodies per [Transformation Pipelines](./05-transformation-pipelines.md).

## Wiring model

**Priority: high** (for executables vs. libraries)

- How does an executable connect an adapter to a port?
- Compile-time DI? Link-time selection? Runtime configuration?
- How are test adapters declared and selected?
- Can one port have multiple adapters in one build (e.g. logging + primary)?

## Effects system

**Priority: medium** (consequence of ports, not driver)

- How are effects declared on a port?
- Effect polymorphism / effect rows?
- Relationship between effect types and port effect lists
- Do effects propagate inward, or stay contained at boundary?

**Explicitly deferred:** detailed effect system design until port formalization is clearer.

## Egress ports

**Priority: medium**

- Symmetric to ingress, or different rules?
- Must application concepts be encoded before boundary exit?
- Streaming egress semantics

## Error handling at boundaries

**Priority: medium**

- Where do parse failures, network errors, etc. live?
- Are failed interpretations still Raw?
- Effect on port pipeline typing

## Module and crate model

**Priority: medium**

- Library exposes ports; executable wires — exact mechanism?
- Can ports span modules? Crates?
- Visibility rules for port interiors vs. application interiors

## Capabilities

**Priority: low** (mentioned in conversation, depends on port model)

- Capability = access token for a port?
- Least-privilege: grant subset of port effects?
- Relationship to adapter implementation

## Metadata and annotations

**Priority: low**

- Port categories for tooling (I/O, time, randomness)
- Security / audit tags
- Custom lint thresholds (e.g. max ports per module)

## Name and branding

**Priority: low**

- Language name: **Zemi** (working title in repo)
- Final keyword choices (`port`, `raw`, `interpret`, etc.)

---

## Suggested design sequence

Before writing compiler code:

1. **Formalize the port** — declaration model, IR shape, what gets generated
2. **Define Raw vs. interpreted** — type system rules and leakage checks
3. **Specify pipeline syntax** — one transformation model for entire language
4. **Sketch wiring** — library/executable split with minimal example
5. **Walk through examples** — hex editor, HTTP API, compiler stage pipeline
6. **Prototype tooling queries** — what the compiler must record in IR

## Example-driven design exercises

Draft implementations of these scenarios (on paper) to stress-test the model:

| Scenario | Stresses |
|----------|----------|
| HTTP API server | Ingress pipeline, auth, egress responses |
| Hex editor | Low-level bytes with semantic wrappers |
| CLI file processor | Filesystem port, batch vs. stream |
| Compiler frontend | Multi-stage interpretation chain |
| Test suite | Plug replacement for all ports |

---

## Decision log

Record settled decisions here as design progresses.

| Date | Decision | Rationale |
|------|----------|-----------|
| 2025-06-30 | Thesis: application boundaries first-class | Captured from team conversation |
| 2025-06-30 | Guiding principle: representation is not meaning | Philosophical foundation |
| 2025-06-30 | Ports extend pipelines, not replace them | Avoid parallel computation models |
| 2025-06-30 | Ports ≠ effects | One connector, many operations |
