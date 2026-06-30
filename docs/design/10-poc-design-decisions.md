# POC Design Decisions

This document resolves the **high-priority open questions** from [Open Questions](./08-open-questions.md) with answers scoped to the POC. These are **provisional** — sufficient to implement and validate the thesis, not necessarily final language choices.

**Status:** POC-scoped (provisional)  
**Companion:** [POC Specification](./09-poc-spec.md)

When a POC decision conflicts with exploratory text in documents 01–08, **this document wins for POC work**. Update the decision log in [08-open-questions.md](./08-open-questions.md) as items graduate from provisional to settled.

---

## How to read this document

Each section follows:

1. **Question** — what was unresolved
2. **POC decision** — what we will implement
3. **Rationale** — why, tied to governing principles
4. **Deferred** — what we explicitly postpone
5. **Validate** — how the reference program tests this decision

---

## 1. Formal definition of a port

**Question:** Is a port a type, module, interface, object, capability, or declaration?

**POC decision:** A port is a **compiler-recognized declaration** on a component — a named block containing a transformation pipeline plus boundary metadata.

```
port <Name> {
    <direction>
    <source declaration>
    <pipeline>
}
```

The compiler generates internal IR nodes (`PortNode`); it does **not** require ports to be traits, interfaces, or runtime objects in the POC.

**Rationale:** Matches the block-decorator model from [Ports](./05-ports.md) and [Governing Principles](./02-governing-principles.md) §3. Minimizes new semantics: ordinary pipeline + architectural wrapper.

**Deferred:** Runtime adapter vtables, capability tokens, generated port types for effect polymorphism.

**Validate:** `UserService.GetUser` and `App.HttpIngress` parse as `PortNode` with pipeline stages preserved in IR.

---

## 2. Raw vs. interpreted types

**Question:** Is Raw a type kind, marker trait, or namespace? Where can Raw appear?

**POC decision:**

| Classification | Types (POC) |
|----------------|-------------|
| **Raw** | `Bytes`, `String`, `u8`–`u64`, `i8`–`i64`, `f32`, `f64`, `bool`, `Vec<T>` where `T` is Raw |
| **Interpreted** | Any user-defined `struct` or `type` alias wrapping meaning (e.g. `UserId`, `User`, `HttpRequest`, `HttpResponse`) |

**Rules (POC):**

1. Ingress port pipeline **input** at the external boundary must be Raw (or a designated boundary type alias to Raw).
2. Ingress port pipeline **output** must be Interpreted.
3. Component interior functions may use Interpreted types freely.
4. Raw may appear in component interior **only** inside port pipeline stages and as fields of Interpreted structs (representation-holding fields).
5. Diagnostic `Z004` (**error**) fires on other Raw uses in interior code.

**Rationale:** Implements "representation is not meaning" without a full kind system. Raw leakage is a **hard error**, not a lint — the same stance as [Governing Principles](./02-governing-principles.md) §1 and §7: the transition from representation to meaning must be explicit and cannot happen accidentally. If the compiler only warns, programmers can suppress or ignore the boundary; an error makes the architectural contract non-optional.

**Why not a lint?** Lints imply optional hygiene. Raw in component interior violates a core language invariant (boundaries are explicit), so it belongs alongside `Z001` (cross-component import) and `Z003` (uninterpreted ingress output) as a compile error. Representation-holding fields inside interpreted structs (e.g. `FileBuffer { bytes: Vec<u8> }`) remain allowed — the error targets Raw used as the *semantic type* of interior logic, not Raw encapsulated within an interpreted wrapper.

**Deferred:** Degrees of interpretation, explicit `raw` keyword, escape hatches, newtype auto-classification.

**Validate:** `decode_utf8(bytes: Bytes) -> String` is allowed in a pipeline stage; `fn handle(bytes: Bytes)` in component interior fails with error `Z004`; `struct FileBuffer { bytes: Vec<u8> }` remains valid.

---

## 3. Component and library syntax

**Question:** Exact keywords, nesting, crate mapping, visibility.

**POC decision:**

| Construct | Syntax | Rules |
|-----------|--------|-------|
| Library | `library Name { ... }` | Exports via `export fn` / `export type`; no ports; no subcomponents |
| Component | `component Name { ... }` | May contain `port`, `component` (subcomponent), `fn`, `type`, internal `wiring` |
| Application | Root `component` in main file | Same as component; designated entry in wiring file |
| Nesting | `component Child` inside parent | Child appears as node in parent's subgraph |
| Visibility | `export` keyword on library items only | Components export ports implicitly; interior is private |
| Cross-import | **Forbidden** | Component A cannot `use` component B's interior; shared code goes in a library |

**File layout (POC):**

```
example/
  app.zemi           # root component + subcomponents
  lib/json.zemi      # libraries
  lib/http.zemi
  wiring/prod.zemi
  wiring/test.zemi
```

One POC crate = one directory; multi-crate packaging deferred.

**Rationale:** Two module kinds from [Components and Libraries](./04-components-and-libraries.md). Minimal file model for parser work.

**Deferred:** Multiple components per crate with separate files, `pub` visibility lattice, module path resolution across crates.

**Validate:** `library Json` parses; `port` inside `library` → `Z002` error.

---

## 4. Pipeline syntax

**Question:** Pipe operator vs. method chains vs. both?

**POC decision:** **Pipe operator only** — `|>` — for both port pipelines and interior code.

```
input
    |> stage_one
    |> stage_two
    |> stage_three
```

Each stage is a function call or callable reference. Stages are sequential (no parallelism in POC).

**Rationale:** One transformation model per [Transformation Pipelines](./06-transformation-pipelines.md). Simple parser.

**Deferred:** Method chains, async pipeline stages, stream combinators.

**Validate:** Port bodies and ordinary functions use identical `|>` syntax; same parse node type in AST.

---

## 5. Port direction and source

**Question:** How are ingress/egress and source/sink expressed?

**POC decision:**

```
port Name {
    ingress | egress          // direction keyword (egress parsed, not enforced)
    source <kind>             // ingress: where data enters
    sink <kind>               // egress: where data exits (stub)

    <pipeline>
}
```

| Source kind (POC) | Meaning |
|-------------------|---------|
| `tcp`, `stdio`, `file` | External adapter slot on root component |
| `sibling` | Connected from another port in same parent via internal wiring |
| `<Component>.<Port>` | Explicit inter-component reference |

**Rationale:** Explicit direction for tooling inventory. External vs. inter-component origin is structurally visible.

**Deferred:** Bidirectional ports, inferred direction, streaming sources.

**Validate:** `HttpIngress` has `source tcp`; `GetUser` has `source sibling`; graph shows external edge only on `HttpIngress`.

---

## 6. Wiring model

**Question:** Compile-time DI? Link-time? Runtime? Test adapter selection?

**POC decision:** **Compile-time wiring** via separate wiring declarations.

**Internal wiring** (inside a component body):

```
wiring {
    HttpIngress.route_to_user -> UserService.GetUser
}
```

Connects one port's pipeline terminus to another port's entry (inter-component edge).

**External wiring** (root component, separate file):

```
wiring App {
    App.HttpIngress -> TcpListener { ... }
    App.UserService.GetUser -> UserServiceImpl
}
```

**Profile selection:** `zemi check --wiring wiring/test.zemi` loads the test profile. Default profile uses `wiring/prod.zemi` if present.

Adapters are **named references** in the POC — no adapter implementation language yet. The compiler validates that every external `source` port has a wiring edge.

**Rationale:** Plug replacement from [Ports](./05-ports.md): same port definition, different adapter in wiring file. Structural, not heuristic.

**Deferred:** Runtime configuration, multiple adapters per port, adapter implementation syntax, link-time adapter selection.

**Validate:** `--wiring wiring/test.zemi` resolves `FakeTcp` instead of `TcpListener` for `HttpIngress`; port definition unchanged.

---

## 7. Effects system

**Question:** How are effects declared on ports?

**POC decision:** **Not modeled.** Ports may carry an optional metadata comment or stub attribute for future use:

```
port HttpIngress {
    ingress
    source tcp
    // @effects network  (ignored by POC compiler)
    ...
}
```

**Rationale:** [Open Questions](./08-open-questions.md) explicitly defer effects until port formalization is validated.

**Deferred:** Entire effect system.

**Validate:** POC compiler ignores effect annotations; port IR includes optional `metadata: []` field.

---

## 8. Egress ports

**POC decision:** Parse `egress` keyword and `sink` declaration. **No semantic enforcement** in POC — ingress rules are the priority.

**Deferred:** Egress pipeline typing, encode-to-Raw requirements, symmetry with ingress.

---

## 9. Error handling at boundaries

**POC decision:** Ordinary return types. Pipeline stages may return `Result<T, E>` like any function. No special port error channel.

**Deferred:** Architectural classification of parse failures, effect on port typing.

---

## 10. Module and crate model

**POC decision:**

- Single directory = one compilable unit
- Root file declares root component
- Libraries in `lib/*.zemi`
- Wiring in `wiring/*.zemi`
- `zemi check example/` compiles all `.zemi` files in tree

**Deferred:** Package manager, semver, multi-crate dependencies, `zemi.toml`.

---

## 11. Toolchain choices

**Question:** What do we build the POC with?

**POC decision:**

| Choice | Selection |
|--------|-----------|
| Implementation language | **Rust** |
| Parser | **logos** (lexer) + **chumsky** or hand-written recursive descent |
| CLI | **clap** |
| First output | **Diagnostic JSON + IR JSON** (no codegen) |
| Graph output | **DOT** (Graphviz) + indented text fallback |

**Rationale:** Rust ecosystem fits compiler work; JSON IR enables rapid tooling iteration without committing to a codegen target.

**Deferred:** Self-hosting, WASM target, LSP.

---

## Decision summary table

| Topic | POC answer | Final language TBD |
|-------|------------|-------------------|
| Port formal model | Compiler-recognized declaration | Maybe |
| Raw vs. interpreted | Builtin = Raw; user struct = Interpreted | Maybe |
| Module kinds | `component`, `library` keywords | Likely |
| Pipeline | `\|>` only | Likely |
| Wiring | External files + compile-time selection | Maybe |
| Effects | Ignored | No |
| Egress | Parsed only | No |
| Codegen | None (IR JSON) | No |

---

## Graduation criteria

A POC decision **graduates** to the main decision log when:

1. The reference program exercises it
2. At least one alternative was considered and rejected (documented)
3. Implementation matches this spec without workarounds
4. Team review promotes it from "POC-scoped" to "settled"

Record graduations in the decision log at the bottom of [08-open-questions.md](./08-open-questions.md).
