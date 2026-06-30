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

Must integrate seamlessly with port bodies per [Transformation Pipelines](./06-transformation-pipelines.md).

## Components and libraries

**Priority: high**

See [Components and Libraries](./04-components-and-libraries.md) for the settled direction. Open details:

- Exact syntax: `component` / `library` keywords (or alternatives)?
- Mapping to crates: can one crate contain multiple components? Multiple libraries?
- Subcomponent declaration: nested `component` blocks inside a parent component?
- Visibility rules: what can be private inside a component vs. what must be in a library to share?
- Can a component depend on another component without a port (never, by design)?
- Internal utilities inside a component: always private, or exportable within the same crate?

**Settled direction:** two top-level module kinds; components declare ports; libraries export reusable code; applications are root components.

## Wiring model

**Priority: high**

- How does wiring connect adapters to ports on the root component?
- How do inner components wire to each other through ports?
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

- How do `component` and `library` declarations map to crates and files?
- Can ports span modules within a component?
- Visibility rules for component interiors vs. library exports
- Relationship between root component and binary entry point

**Settled direction:** library/executable/component crate trichotomy generalizes into component + library module kinds, with the application as root component. See [Components and Libraries](./04-components-and-libraries.md).

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

1. **Formalize components and libraries** — declaration model, visibility rules, crate mapping
2. **Formalize the port** — declaration model on components, IR shape, what gets generated
3. **Define Raw vs. interpreted** — type system rules and leakage checks
4. **Specify pipeline syntax** — one transformation model for entire language
5. **Sketch wiring** — root component and inter-component port connection with minimal example
6. **Walk through examples** — hex editor, HTTP API, compiler stage pipeline
7. **Prototype tooling queries** — component graph and port inventory from IR

### POC track (completed for v0)

Steps 1–5 and 7 are addressed for the POC in:

- [POC Specification](./09-poc-spec.md)
- [POC Design Decisions](./10-poc-design-decisions.md)
- [Implementation Plan](./11-implementation-plan.md)

The HTTP API reference program in [`example/`](../../example/) implements step 6 for the POC scenario. Additional scenarios remain for post-POC design exercises.

## Example-driven design exercises

Draft implementations of these scenarios (on paper) to stress-test the model:

| Scenario | Stresses |
|----------|----------|
| HTTP API server | Root component, inner components, ingress pipeline, auth, egress |
| Hex editor | Low-level bytes with semantic wrappers, single root component |
| CLI file processor | Filesystem port, batch vs. stream |
| Compiler frontend | Multi-stage interpretation chain, nested components |
| Test suite | Plug replacement for all ports |
| Shared geometry | Library used by multiple components; no cross-component imports |

---

## Decision log

Record settled decisions here as design progresses.

| Date | Decision | Rationale |
|------|----------|-----------|
| 2025-06-30 | Thesis: application boundaries first-class | Captured from team conversation |
| 2025-06-30 | Guiding principle: representation is not meaning | Philosophical foundation |
| 2025-06-30 | Ports extend pipelines, not replace them | Avoid parallel computation models |
| 2025-06-30 | Ports ≠ effects | One connector, many operations |
| 2025-06-30 | Two module kinds: component and library | Architectural units vs. reusable implementation; few powerful distinctions |
| 2025-06-30 | Applications are root components | Generalize ports from application-only to any component |
| 2025-06-30 | Components are closed; libraries are open | Components export ports; libraries export reusable code |
| 2025-06-30 | Ports exist at component depth only | Inside a component, ordinary code; compiler stops thinking architecturally |
| 2025-06-30 | POC: port = compiler-recognized declaration | Provisional — see [10-poc-design-decisions.md](./10-poc-design-decisions.md) |
| 2025-06-30 | POC: Raw = builtins; Interpreted = user structs | Provisional — sufficient for leak lint in v0 |
| 2025-06-30 | POC: `\|>` pipe operator only | One transformation model for ports and interior code |
| 2025-06-30 | POC: compile-time wiring via external files | Plug replacement without changing port definitions |
| 2025-06-30 | POC scenario: HTTP API + inner component | Reference program in `example/` |
