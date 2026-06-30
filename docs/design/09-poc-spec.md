# POC Specification

This document defines the **Proof of Concept** for Zemi: what we will build first, what success looks like, and the reference program that stress-tests the design.

**Status:** POC-scoped proposal (not final language specification)  
**Depends on:** [Open Questions](./08-open-questions.md), [POC Design Decisions](./10-poc-design-decisions.md)  
**Leads to:** [Implementation Plan](./11-implementation-plan.md)

---

## Purpose

The POC exists to answer one question:

> Can the compiler recognize components and ports as first-class architectural constructs and produce **structural** architecture information — without heuristics?

We are **not** trying to ship a usable language. We are trying to prove the thesis from [Vision](./01-vision.md): once the compiler knows what components and ports *are*, enforcement and tooling follow.

## POC thesis (narrowed)

| In scope | Out of scope |
|----------|--------------|
| Parse `component`, `library`, `port`, `wiring` | Full type inference |
| Build IR preserving component graph | Code generation to a production target |
| Enforce Raw/interpreted boundary (ingress) | Effect system |
| Wire one ingress port to prod/test adapters | Egress ports |
| Emit architecture diagnostics and CLI queries | IDE integration |
| Reject cross-component implementation imports | Capabilities / least-privilege |
| One pipeline operator (`\|>`) shared by ports and interior code | Async streams, generators |

## Success criteria

The POC is complete when all of the following hold:

1. **Parse** — The reference program in [Reference program](#reference-program) parses without ambiguity.
2. **IR** — The compiler produces an IR that preserves components, ports, wiring, and pipeline stages (see [IR requirements](#ir-requirements)).
3. **Graph** — `zemi components graph` renders the component/port graph for the reference program.
4. **Inventory** — `zemi ports list` lists every port with direction, component, and pipeline stages.
5. **Leak lint** — The compiler warns when a Raw type appears in component interior logic outside allowed sites.
6. **Import lint** — The compiler errors when one component imports another component's implementation.
7. **Wiring** — Selecting `--test` vs. default wiring chooses a different adapter for the same port.
8. **Documentation** — A reader can follow [10-poc-design-decisions.md](./10-poc-design-decisions.md) and this document to understand every construct in the reference program.

## Reference scenario

**HTTP API with one inner component and plug replacement.**

This scenario stresses:

- Root component with external ingress
- Nested subcomponent
- Shared library (invisible to architecture graph)
- Port pipeline with staged interpretation
- Compile-time wiring (production vs. test adapter)
- Cross-component communication through ports only

Alternatives considered for POC (deferred):

| Scenario | Why deferred |
|----------|--------------|
| Hex editor | Less wiring; weaker plug-replacement story |
| Compiler frontend | Heavy type progression; distracts from architecture |
| CLI file processor | Good second example; not needed for v0 |

## Reference program

Illustrative syntax — POC-scoped, not final spec. See [POC Design Decisions](./10-poc-design-decisions.md) for rationale.

### Shared library

```
library Json {
    export fn parse(text: String) -> JsonValue
}

library Http {
    export fn parse_request(raw: String) -> HttpRequest
}
```

Libraries export reusable code. They have no ports and do not appear in the architecture graph.

### Inner component

```
component UserService {

    port GetUser {
        ingress
        source sibling

        HttpRequest
            |> extract_user_id
            |> load_user
            |> to_response
    }

    fn extract_user_id(req: HttpRequest) -> UserId { ... }
    fn load_user(id: UserId) -> User { ... }
    fn to_response(user: User) -> HttpResponse { ... }
}
```

`UserService` declares one **ingress** port. The pipeline progressively interprets `HttpRequest` (boundary type) into `HttpResponse` (application output). Internal functions operate on interpreted types (`UserId`, `User`).

### Root component

```
component App {

    port HttpIngress {
        ingress
        source tcp

        Bytes
            |> decode_utf8
            |> Http.parse_request
            |> Json.parse
            |> route_request
    }

    component UserService

    wiring {
        HttpIngress.route_to_user -> UserService.GetUser
    }

    fn decode_utf8(bytes: Bytes) -> String { ... }
    fn route_request(req: HttpRequest) -> HttpResponse { ... }
}
```

`App` is the **root component**. Its `HttpIngress` port connects to the outside world via `tcp`. An inner `UserService` subcomponent connects through an inter-component wiring edge.

### External wiring (deployment)

Wiring lives outside component bodies for the root component's external adapters:

```
// wiring/prod.zemi
wiring App {
    App.HttpIngress -> TcpListener { host = "0.0.0.0", port = 8080 }
    App.UserService.GetUser -> UserServiceImpl
}

// wiring/test.zemi
wiring App {
    App.HttpIngress -> FakeTcp
    App.UserService.GetUser -> InMemoryUserService
}
```

The POC compiler selects wiring via a flag: `zemi build --wiring wiring/test.zemi`.

### What the architecture graph shows

```
         [Outside: tcp]
                │
                ▼
         ┌─────────────┐
         │     App     │
         │ HttpIngress │
         └──────┬──────┘
                │ (inter-component)
                ▼
         ┌─────────────┐
         │ UserService │
         │   GetUser   │
         └─────────────┘

Libraries (Json, Http): not shown — implementation detail
```

## IR requirements

The POC IR must carry enough structure for tooling from day one ([Tooling Implications](./07-tooling-implications.md)). Minimal node shapes:

### Module kinds

```
ModuleKind ::= Component | Library

ComponentNode {
    name: String
    ports: Vec<PortNode>
    subcomponents: Vec<ComponentRef>
    interior: ModuleTree        // ordinary functions, types, private items
    wiring: Vec<InternalWiring> // edges between ports within this component
}

LibraryNode {
    name: String
    exports: Vec<Export>
}
```

### Port

```
PortNode {
    name: String
    owner: ComponentRef
    direction: Ingress | Egress    // POC: Ingress only required; Egress stubbed
    source: ExternalSource | SiblingPort | ComponentPort
    pipeline: PipelineNode
    span: SourceSpan               // for diagnostics
}

PipelineNode {
    stages: Vec<Stage>
    input_type: TypeRef            // must be Raw or boundary type at ingress start
    output_type: TypeRef           // must be interpreted at ingress end
}

Stage {
    name: String                   // function or transform reference
    input_type: TypeRef
    output_type: TypeRef
    span: SourceSpan
}
```

### Wiring

```
WiringNode {
    root: ComponentRef
    edges: Vec<WiringEdge>
}

WiringEdge {
    from: PortRef | ExternalAdapter
    to: PortRef | ExternalAdapter
    build_profile: Default | Test | Custom(String)
}
```

### Type classification (for leak lint)

```
TypeKind ::= Raw | Interpreted

// POC rule: primitives + String + Bytes + Vec<u8> = Raw
//           user-defined structs/newtypes = Interpreted
//           fields may hold Raw inside Interpreted wrappers (e.g. FileBuffer.bytes)
```

## Compiler diagnostics (POC set)

| Code | Severity | Trigger |
|------|----------|---------|
| `Z001` | error | Component A imports component B's interior module |
| `Z002` | error | Port declared on a library |
| `Z003` | error | Ingress port pipeline output is Raw |
| `Z004` | warn | Raw type used in component interior outside port pipeline |
| `Z005` | error | Required port has no wiring edge |
| `Z006` | warn | Component declared but never connected in wiring graph |

## CLI commands (POC set)

| Command | Output |
|---------|--------|
| `zemi check <file>` | Parse + semantic checks + diagnostics |
| `zemi components list` | All components with nesting |
| `zemi components graph` | DOT or indented text graph |
| `zemi ports list` | Port inventory with direction and stages |
| `zemi wiring show` | Resolved wiring for current profile |

## Explicit deferrals

These remain open for the full language. The POC must **not** block on them:

| Topic | POC stance |
|-------|------------|
| Effect system | Ignore; mention in port metadata stub only |
| Egress ports | Parse keyword; no enforcement |
| Error handling at boundaries | Ordinary function return types; no special port error model |
| Capabilities | Not modeled |
| Async / streaming | Synchronous pipeline stages only |
| Full type inference | Explicit type annotations on port pipeline stage boundaries |
| Crate / package manager | Single-file or single-crate POC layout |
| Code generation | Optional stub that prints IR JSON |

## Design exercises (post-POC)

After the POC succeeds, stress-test the model on paper before expanding scope:

| Scenario | New pressure |
|----------|--------------|
| Hex editor | Low-level domain; representation-holding fields |
| CLI file processor | Filesystem port; batch vs. stream |
| Compiler frontend | Multi-stage interpretation chain |
| Shared geometry library | Two components, one library, zero cross-imports |

## Relationship to other documents

| Document | Role |
|----------|------|
| [08-open-questions.md](./08-open-questions.md) | Full-language unresolved items |
| [10-poc-design-decisions.md](./10-poc-design-decisions.md) | POC-scoped answers to high-priority questions |
| [11-implementation-plan.md](./11-implementation-plan.md) | Phased build plan and repo layout |

## Decision log (POC)

| Date | Decision | Rationale |
|------|----------|-----------|
| 2025-06-30 | POC scenario: HTTP API + inner component | Richest proof of graph, wiring, plug replacement |
| 2025-06-30 | Ingress-only enforcement | Egress deferred; reduces POC surface |
| 2025-06-30 | External wiring files for root adapters | Separates deployment from component logic |
| 2025-06-30 | IR JSON as optional output | Enables tooling without codegen |
