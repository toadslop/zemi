# POC Design Decisions

This document resolves the **high-priority open questions** from [Open Questions](./08-open-questions.md) with answers scoped to the POC. These are **provisional** — sufficient to implement and validate the thesis, not necessarily final language choices.

**Status:** POC-scoped (provisional)  
**Companion:** [POC Specification](./09-poc-spec.md)  
**Last design review:** 2026-06-30 — wiring, subcomponents, and newtype rules captured from team conversation

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
| **Interpreted** | Any user-defined `struct` or newtype wrapping meaning (e.g. `UserId`, `User`, `HttpRequest`, `HttpResponse`) |

**Rules (POC):**

1. Ingress port pipeline **input** at the external boundary must be Raw (or a designated boundary type alias to Raw).
2. Ingress port pipeline **output** must be Interpreted.
3. Component interior functions may use Interpreted types freely.
4. Raw may appear in component interior **only** inside port pipeline stages and as fields of Interpreted structs (representation-holding fields).
5. Diagnostic `Z004` (**error**) fires on other Raw uses in interior code.
6. **Component interior types are not exportable.** Other components cannot import them; communication is through ports only. Cross-component Raw leakage via exported interior types is therefore not a concern — encapsulation handles it.

**Libraries (directional, not locked):**

Libraries are shared implementation visible to many components. The interpretation status of library-exported types is **unspecified** and intended to be validated through experimentation. Current leaning:

- Libraries should eventually be able to **declare a type as Raw**, forcing importers to wrap it in their own newtype before treating it as domain meaning.
- Library types not marked Raw carry the **library author's interpretation** when imported.

POC does **not** enforce library Raw marking or import-time wrapping. Focus remains on ingress boundary and interior leak checking inside components.

**Rationale:** Implements "representation is not meaning" without a full kind system. Raw leakage is a **hard error**, not a lint — the same stance as [Governing Principles](./02-governing-principles.md) §1 and §7: the transition from representation to meaning must be explicit and cannot happen accidentally. If the compiler only warns, programmers can suppress or ignore the boundary; an error makes the architectural contract non-optional.

**Why not a lint?** Lints imply optional hygiene. Raw in component interior violates a core language invariant (boundaries are explicit), so it belongs alongside `Z001` (cross-component import) and `Z003` (uninterpreted ingress output) as a compile error. Representation-holding fields inside interpreted structs (e.g. `FileBuffer { bytes: Vec<u8> }`) remain allowed — the error targets Raw used as the *semantic type* of interior logic, not Raw encapsulated within an interpreted wrapper.

**Deferred:** Degrees of interpretation, explicit `raw` keyword, escape hatches, `type Alias = u64` classification, library Raw declaration syntax, import-time wrapping enforcement.

**Validate:** `decode_utf8(bytes: Bytes) -> String` is allowed in a pipeline stage; `fn handle(bytes: Bytes)` in component interior fails with error `Z004`; `struct FileBuffer { bytes: Vec<u8> }` remains valid; `struct UserId(u64)` is Interpreted.

---

## 3. Component and library syntax

**Question:** Exact keywords, nesting, crate mapping, visibility.

**POC decision:**

| Construct | Syntax | Rules |
|-----------|--------|-------|
| Library | `library Name { ... }` | Exports via `export fn` / `export type`; no ports; no subcomponents |
| Component | `component Name { ... }` | May contain `port`, `component` (subcomponent), `fn`, `type`, internal `wiring` |
| Application | Root `component` in main file | Same as component; designated entry in wiring file |
| Nesting | `component Child` inside parent | See [Subcomponents](#3a-subcomponents) below |
| Visibility | `export` keyword on library items only | Components export ports implicitly; interior is private |
| Cross-import | **Forbidden** | Component A cannot `use` component B's interior; shared code goes in a library |

### 3a. Subcomponents

**Physical analogy:** A component is a machine. Sub-assemblies are installed inside it. The connection model is always the same — subcomponents have ports; the parent wires to them — but **where the subcomponent is defined** determines encapsulation.

| Pattern | Analogy | Definition | Visibility |
|---------|---------|------------|------------|
| **Installed part** | Off-the-shelf motor bolted into a chassis | Defined **externally**, then **referenced** inside the parent (`component UserService`) | Reusable — the same definition can be installed in many parents |
| **Proprietary part** | Custom PCB built only for this product | Defined **inline** inside the parent (`component Cache { port ...; fn ... }`) | Private to the enclosing component; not referenceable from outside |

**Multi-installation:** An externally defined component exists so it can be used in many places — like one engine model in many cars. Each installation is a **separate instance** in the architecture graph:

```
App.UserService.GetUser          // installation in App
AdminApp.UserService.GetUser     // same definition, different installation
```

**Rules (POC):**

1. Inline nested components are **private to the enclosing component** — proprietary parts stay inside the enclosure.
2. Externally defined components may be **installed in multiple parents** — each install is a distinct graph node.
3. **No wiring across encapsulation boundaries** without going through the parent's ports — you cannot reach inside another component's chassis.
4. Subcomponents always appear in the architecture graph **under their parent**; visibility controls referenceability, not existence.
5. Internal `wiring { }` connects parent ports to child ports within the same enclosing component.

**Validate:** `example/app.zemi` demonstrates the installed-part pattern (`component UserService` reference + external definition). Proprietary inline subcomponents are supported in syntax/IR but optional in the reference program.

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

**Question:** Compile-time DI? Link-time? Runtime? Test adapter selection? What granularity do wiring edges attach to?

**POC decision:** **Compile-time wiring** via wiring declarations. Wiring connects **ports to ports** — like plugging a cable into a socket on a device, not into an internal wire inside the device.

**Physical analogy:** A port is a plug socket (conceptually one hole). Wiring is whatever connects to that socket — a cable, splitter, or hub. Topology complexity (1:1, 1:many, many:many) lives in **wiring**, not in the port definition. The full topology model is **not specified yet**; keep it simple for POC.

**Internal wiring** (inside a component body):

```
wiring {
    HttpIngress -> UserService.GetUser
}
```

Port-to-port edges within the enclosing component. No sub-point labels (e.g. no `HttpIngress.route_to_user`).

**External wiring** (root component, separate file):

```
wiring App {
    App.HttpIngress -> TcpListener { ... }
    App.UserService.GetUser -> UserServiceImpl
}
```

**Profile selection:** `zemi check --wiring wiring/test.zemi` loads the test profile. Default profile uses `wiring/prod.zemi` if present.

Adapters are **named references** in the POC — no adapter implementation language yet. The compiler validates that every external `source` port has **at least one** wiring edge (`Z005`).

**POC wiring limits:**

| In scope | Deferred |
|----------|----------|
| Port-to-port edges | Hub/splitter semantics (multiple adapters on one port) |
| Parse multiple edges to the same port | What happens when 2+ adapters attach to one port |
| External wiring files for plug replacement | Port-local or inferred wiring (long-term preference to reduce explicit wiring ceremony) |
| Internal wiring 1:1 in reference example | Runtime configuration, adapter implementation syntax |

**Long-term direction (not POC):** Explicit wiring files prove the plug-replacement model but are not necessarily permanent. Wiring may eventually collapse into port-local connection declarations or inference.

**Rationale:** Plug replacement from [Ports](./05-ports.md): same port definition, different adapter in wiring file. Structural, not heuristic. Port-to-port edges keep the plug analogy clean.

**Deferred:** Runtime configuration, hub/splitter topology rules, adapter implementation syntax, link-time adapter selection, port-local wiring syntax.

**Validate:** `--wiring wiring/test.zemi` resolves `FakeTcp` instead of `TcpListener` for `HttpIngress`; port definition unchanged; internal wiring uses `HttpIngress -> UserService.GetUser`.

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
| Raw vs. interpreted | Builtin = Raw; user struct/newtype = Interpreted | Maybe |
| Component interior types | Not exportable; cross-component type leak N/A | Likely |
| Library type interpretation | Unspecified; libraries may declare Raw types (experiment) | Yes |
| Subcomponents | Installed (external def) + proprietary (inline def); multi-install | Likely |
| Module kinds | `component`, `library` keywords | Likely |
| Pipeline | `\|>` only | Likely |
| Wiring | Port-to-port; external files for POC; topology deferred | Maybe |
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
