# Tooling Implications

Once the compiler understands components and ports, many tools become possible **without AI or heuristics**. They are direct consequences of having an architectural model in the language.

## Principle

Tooling value comes from the compiler's model, not from guessing at programmer intent.

Today, tools that answer architectural questions must approximate: parse names, infer patterns, or rely on conventions. With first-class components and ports, answers are **structural**.

## Architecture graph

The compiler can model the system as a graph:

- **Nodes:** components
- **Edges:** ports (connections between components and to the outside world)
- **Invisible:** libraries (implementation detail inside nodes)

This graph is not inferred from naming conventions. It is derived from declared `component` and `port` constructs.

## Enabled analyses (draft list)

| Capability | Description |
|------------|-------------|
| **Component inventory** | List every component in a system |
| **Architecture graph** | Draw the component/port graph (nodes and edges) |
| **Port inventory** | List every port on every component |
| **Boundary graph** | Show how ports connect components to the environment |
| **Effect reachability** | Show every place that reaches the network, filesystem, clock, etc. |
| **Port categorization** | Count and group ports by category (I/O, time, randomness, …) |
| **Architecture linting** | Warn if a component introduces too many ports |
| **Dead component detection** | Find components never wired into the graph |
| **Cycle checking** | Detect dependency cycles across component wiring |
| **Port wiring validation** | Verify all required ports are connected |
| **Test coverage (ports)** | Warn if a port is never connected to a test adapter |
| **Bypass detection** | Warn if code reaches the environment without going through a declared port |
| **Cross-component import detection** | Warn if a component imports another component's implementation (should use ports or a shared library) |
| **Interpretation visualization** | Show flow of Raw representations into semantic types |
| **Test harness generation** | Generate wiring stubs from the component graph |
| **Deployment metadata** | Export architecture graph for deployment tooling |

None of these require machine learning. They require the compiler to know what components and ports are.

## Plug replacement tooling

Because ports define substitution sites explicitly:

- Test runners can require every port has a test wiring
- CI can verify no production adapter leaks into unit tests
- IDEs can offer "swap adapter" refactoring scoped to a port

## IDE integration (anticipated)

Similar to how Rust IDEs expose ownership and lifetimes once the compiler models them:

- Inline boundary indicators on port declarations
- Navigate from component code to the port that supplies a capability
- Highlight Raw values that have not crossed an interpretation boundary
- Show adapter wiring for current build target (test vs. prod)

## Documentation generation

Port declarations are self-documenting architecture:

- Auto-generated component graphs (components as nodes, ports as edges)
- Auto-generated "system context" diagrams (root component ports on the boundary)
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

Tooling should be designed alongside the component and port semantic model, not bolted on later. The AST / IR representation of components and ports should carry enough structure to support these analyses from day one — even if initial tooling is minimal.

Suggested early tooling targets (when implementation begins):

1. `zemi components list` — component inventory
2. `zemi components graph` — architecture diagram (text or DOT)
3. `zemi ports list` — port inventory per component
4. Compiler warning: Raw value escaping port boundary
5. Compiler warning: external access not through a port
6. Compiler warning: component importing another component's implementation

## Relationship to other concepts

| Concept | Tooling angle |
|---------|---------------|
| Components | Nodes in architecture graph; inventory and cycle detection |
| Libraries | Invisible to graph; used inside component nodes |
| Raw vs. interpreted types | Highlight uninterpreted values in component interior |
| Capabilities | Trace which code holds access to which port |
| Root component | Show external adapter wiring for deployment |
| Effects | Aggregate effects by port, not just by function name |
