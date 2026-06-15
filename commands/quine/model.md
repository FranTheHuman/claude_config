# /quine:model — JSON-to-Graph Data Modeling and ID Strategy

Map an incoming JSON event schema to a Quine graph shape: which fields become
nodes, properties, or edges, and the `idFrom` strategy for each node type.
Apply knowledge from `.claude/skills/quine-architecture/SKILL.md`.

## Instructions

1. Read `.claude/skills/quine-architecture/SKILL.md` sections 3 and 4 (JSON to
   Graph Translation, ID Strategy) before mapping anything.
2. If a standing query pattern has already been designed (via `/quine:design` or
   stated by the user), the graph shape must satisfy THAT pattern — map fields
   accordingly, not in the abstract. If no pattern exists yet, ask what pattern
   this data will be used to detect before mapping — mapping without a target
   pattern risks designing a graph that can't express it (skill section 2).
3. For each JSON field, classify using the decision table: node (has identity,
   referenced from multiple places), property (scalar attribute), edge
   (traversal needed), or property-not-edge (high-cardinality categorical).
4. For every node type identified, specify the exact `idFrom` arguments,
   including the type prefix, and check for collisions with other node types in
   this model.
5. If this data will be ingested alongside OTHER streams referencing the same
   entities (e.g. TigerBeetle CDC + a Node.js profile-update stream both
   referencing `:Account`), explicitly state the shared `idFrom` convention both
   streams must use.
6. Flag any field that looks like it should be a node but would create a
   supernode (skill section 8) — categorical fields with few distinct values and
   potentially many connections.

## Output Structure

### Source Data
The JSON shape being mapped (restate or accept as given).

### Target Pattern
The standing query pattern this graph shape must support — restate if provided,
or note "not yet defined — see warnings."

### Field Mapping

| JSON Field | Graph Element | Details |
|------------|---------------|---------|
| `field.path` | Node / Property / Edge | `idFrom` args, or property name, or edge label+direction |

### Node ID Strategies

For each node type:
```cypher
id(<alias>) = idFrom("<type-prefix>", <args>)
```

### Supernode Warnings
Any field mapped to a node that could become a supernode, with the
property-instead-of-edge alternative.

### Cross-Stream Consistency Notes
If applicable — which other ingest streams must use the same `idFrom` arguments
for shared entities.

## Arguments

Usage: `/quine:model <paste JSON event schema> [for pattern: <description>]`

Examples:
- `/quine:model {"orderId": "...", "customerId": "...", "items": [...]} for pattern: customers who order from multiple categories`
- `/quine:model TigerBeetle CDC transfer event for pattern: device sends to 5+ distinct accounts`
- `/quine:model {"userId": "...", "loginIp": "...", "country": "..."} for pattern: users logging in from multiple countries`
