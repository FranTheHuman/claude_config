# /quine:standing-query — Generate a Standing Query (Pattern + Output)

Generate a complete standing query — pattern, mode, and output(s) — for a
specific detection goal, including chained or recursive stages if needed.
Apply knowledge from `.claude/skills/quine-architecture/SKILL.md`.

## Instructions

1. Read `.claude/skills/quine-architecture/SKILL.md` section 6 (Standing Query
   Design) before generating.
2. Identify which pattern from the catalog below fits, or combine them.
3. Default to Distinct ID mode. Switch to Multiple Values ONLY if the query needs
   per-match property values or a `WHERE` on a computed/aggregated property —
   state which reason applies.
4. For counting/aggregation goals, check the idempotency trap (skill section 7):
   does this rely on `int.add` per ingested record? If so, either propose the
   "relationship as a node" alternative from the worked example, or explicitly
   accept approximate counts and say so.
5. For recursive patterns, the termination condition is MANDATORY — generate it
   and point it out explicitly.
6. State the output decision (emit / update / both) with destination type
   (StandardOut, File, HttpEndpoint, Kafka, Kinesis, SNS, Slack, CypherQuery, Drop).
7. If the output feeds something that can't tolerate missed/duplicate events,
   note the best-effort delivery guarantee (skill section 10) and suggest a
   queue-backed destination.

## Pattern Catalog

| Pattern | Trigger phrases | Skill reference |
|---------|-----------------|------------------|
| Direct pattern detection | "detect when", "find", "alert on" | §6 Pattern 1 |
| Aggregation via graph update | "count", "how many", "track totals" | §6 Pattern 2 |
| Chained/staged queries | "threshold", "after N occurrences", "two-step" | §6 Pattern 3 |
| Recursive propagation | "propagate", "taint", "spread", "transitively" | §6 Pattern 4 |

## Output Structure

### Goal
One sentence restating what's being detected.

### Mode
Distinct ID or Multiple Values — with justification.

### Pattern
```cypher
// MATCH ... RETURN
```

### Idempotency / Aggregation Note
If counting or aggregating: how this behaves on ingest replay, and the
alternative if it's not idempotent.

### Output
```cypher
// Output/enrichment query, if any
```
Destination: [type] — and best-effort delivery note if relevant.

### Additional Stages (if chained/recursive)
Repeat Pattern/Output structure for each stage. For recursive patterns, state
the termination condition explicitly.

## Arguments

Usage: `/quine:standing-query <detection goal>`

Examples:
- `/quine:standing-query alert when an order ships to a different address than billing`
- `/quine:standing-query count distinct destination accounts per device`
- `/quine:standing-query flag devices after they exceed 5 distinct destination accounts`
- `/quine:standing-query propagate a "tainted" flag along transaction paths`
- `/quine:standing-query find users connected to a known-bad IP`
