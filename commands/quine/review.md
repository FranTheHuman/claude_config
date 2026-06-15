# /quine:review — Review Cypher Queries or Recipes for Anti-Patterns

Review existing ingest queries, standing queries, or recipe YAML for correctness,
performance (full node scans, supernodes), idempotency, and architectural fit
(is this the right thing to put in Quine at all).
Apply all rules from `.claude/skills/quine-architecture/SKILL.md`.

## Instructions

1. Read `.claude/skills/quine-architecture/SKILL.md` sections 7, 8, 9, 10, and 16
   (Anti-Patterns) before reviewing — plus section 6 if a standing query is
   present.
2. Read the target file(s)/query. If no path or query given, ask for it.
3. Check immediately for hard rule violations — instant CRITICAL:
   - `MATCH (n) WHERE n.<property> = ...` (ingest or ad-hoc, not a standing query
     pattern) without `id(n) = idFrom(...)` anywhere in the query — full node scan
   - `idFrom` arguments for a node type that are inconsistent with how that same
     type is addressed elsewhere in the codebase (search for other `idFrom(...)`
     calls with the same type prefix and compare argument lists)
   - `CALL int.add(...)` or similar counters incremented once per ingested record
     with no replay/idempotency handling
   - Existence-check patterns (`OPTIONAL MATCH` + `IS NULL` used to decide
     whether to "create" something) — violates "all nodes exist"
   - A node type with edges from many other node types AND no corresponding
     "terminates ON this node" standing query — likely supernode with no
     justification
   - Recursive standing query patterns without a visible termination condition
     (e.g. propagation patterns missing an `IS NULL` / flag guard)
   - Any property or node clearly representing financial balance, account state,
     or anything that should be in TigerBeetle instead
4. Check standing query mode choice: Multiple Values used where Distinct ID would
   suffice (unnecessary memory overhead).
5. Check recipe YAML structure against the standard attributes (skill section 13)
   if reviewing a recipe.
6. Output the review in the structure below.

## Output Structure

### Summary
One sentence on overall design quality and the most critical issue, if any.

### Issues

Group by severity:

```
[SEVERITY] Short title
Location: file/query section
Problem:  What is wrong and why it matters (performance, correctness,
          idempotency, architectural fit).
Fix:      Corrected Cypher or YAML.
```

Severity levels:
- `[CRITICAL]` — full node scan in production path, silent duplicate nodes from ID
  inconsistency, financial state in Quine, or unterminated recursion
- `[WARNING]` — non-idempotent aggregation, unjustified supernode risk, wrong SQ mode
- `[SUGGESTION]` — style, chaining opportunities, monitoring gaps

### Verdict
- ✅ Approved — correct and production-ready
- ⚠️ Approved with warnings — functional but needs improvement before scale
- ❌ Blocked — critical issues (especially full scans or ID inconsistency) will
  cause silent incorrect behavior in production

## Arguments

Usage: `/quine:review [path or pasted Cypher/YAML]`

Examples:
- `/quine:review recipes/fraud-detection.yaml`
- `/quine:review` then paste an ingest query
- `/quine:review` then paste a standing query pattern + output
