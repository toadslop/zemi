# Tooling Implications

Once the compiler understands ports, many tools become possible **without AI or heuristics**. They are direct consequences of having an architectural model in the language.

## Principle

Tooling value comes from the compiler's model, not from guessing at programmer intent.

Today, tools that answer architectural questions must approximate: parse names, infer patterns, or rely on conventions. With first-class ports, answers are **structural**.

## Enabled analyses (draft list)

| Capability | Description |
|------------|-------------|
| **Port inventory** | List every port in an application |
| **Boundary graph** | Draw how ports connect application to environment |
| **Effect reachability** | Show every place that reaches the network, filesystem, clock, etc. |
| **Port categorization** | Count and group ports by category (I/O, time, randomness, …) |
| **Architecture linting** | Warn if a module introduces too many ports |
| **Test coverage (ports)** | Warn if a port is never connected to a test adapter |
| **Bypass detection** | Warn if code reaches the environment without going through a declared port |
| **Dependency diagrams** | Generate port-level dependency graphs |
| **Interpretation visualization** | Show flow of Raw representations into semantic types |

None of these require machine learning. They require the compiler to know what a port is.

## Plug replacement tooling

Because ports define substitution sites explicitly:

- Test runners can require every port has a test wiring
- CI can verify no production adapter leaks into unit tests
- IDEs can offer "swap adapter" refactoring scoped to a port

## IDE integration (anticipated)

Similar to how Rust IDEs expose ownership and lifetimes once the compiler models them:

- Inline boundary indicators on port declarations
- Navigate from application code to the port that supplies a capability
- Highlight Raw values that have not crossed an interpretation boundary
- Show adapter wiring for current build target (test vs. prod)

## Documentation generation

Port declarations are self-documenting architecture:

- Auto-generated "system context" diagrams (ports as boxes on the boundary)
- Per-port pipeline stage listing (interpretation steps)
- Effect summary per port

## Audit and compliance

For systems with regulatory or security requirements:

- Prove all network access goes through declared HTTP ports
- Prove filesystem access is confined to named ports
- Track which ports handle sensitive data types

Exact metadata attachment on ports (tags, security levels) is an open design area.

## What tooling should not try to do

Per governing principles, tooling should **not**:

- Judge whether domain models are "good enough"
- Enforce a specific architectural pattern beyond boundary explicitness
- Replace code review with heuristics about business logic correctness

The compiler makes boundaries visible. Humans (and tests) judge domain correctness.

## Implementation note

Tooling should be designed alongside the port semantic model, not bolted on later. The AST / IR representation of ports should carry enough structure to support these analyses from day one — even if initial tooling is minimal.

Suggested early tooling targets (when implementation begins):

1. `zemi ports list` — inventory
2. `zemi ports graph` — boundary diagram (text or DOT)
3. Compiler warning: Raw value escaping port boundary
4. Compiler warning: external access not through a port

## Relationship to other concepts

| Concept | Tooling angle |
|---------|---------------|
| Raw vs. interpreted types | Highlight uninterpreted values in application interior |
| Capabilities | Trace which code holds access to which port |
| Library vs. executable | Show wiring differences between build targets |
| Effects | Aggregate effects by port, not just by function name |
