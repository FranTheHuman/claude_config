# /quine:design — Three-Part System Design Session

Walk through the complete Quine design methodology for a new pattern-detection
use case: Standing Query defines Graph Shape, which is created by Ingest Query.
Apply knowledge from `.claude/skills/quine-architecture/SKILL.md`.

## Instructions

1. Read `.claude/skills/quine-architecture/SKILL.md` section 2 (The Three-Part
   System) and section 14 (Worked Example) before responding.
2. Follow the Design Session Template from section 2 **in order, as labeled
   steps** — never produce ingest query code before the pattern and graph shape
   are defined, even if the user explicitly asked for "the ingest query."
3. If the user's description doesn't contain enough information for a step,
   ask — but prefer making a reasonable, explicitly-stated assumption and
   continuing, flagging it as an assumption to confirm.
4. If at step 4 or 5 the design feels awkward (the data doesn't support the
   pattern), stop, say so, and go back to step 2 with the gap identified.
5. Address idempotency for the ingest query as part of step 5, not as an
   afterthought (skill sections 7 and 10).
6. End with the output decision (emit / update / both) and the destination type.

## Output Structure

### Step 1 — The Question
Restate the pattern to detect as one sentence: "Detect when [X] happens across
[Y] within [Z]."

### Step 2 — The Graph Pattern
```cypher
// The MATCH ... RETURN shape that expresses the question
```
Identify: which node labels, which edges, which property conditions.

### Step 3 — Mode Choice
Distinct ID or Multiple Values, with a one-sentence justification.

### Step 4 — Graph Structure

| Entity | Node Label | ID Strategy (`idFrom`) | Key Properties |
|--------|-----------|------------------------|-----------------|

Edges needed, with direction and label.

### Step 5 — Ingest Query
```cypher
// Complete ingest query following the standard template from skill §7
```
Idempotency note: how this query behaves on at-least-once replay.

### Step 6 — Standing Query and Output
```cypher
// Pattern query
```
```cypher
// Output query (if enrichment/update needed)
```
Output destination and emit-vs-update rationale.

### Assumptions Made
Bullet list of anything assumed rather than confirmed — only if applicable.

## Arguments

Usage: `/quine:design <pattern to detect, in plain language>`

Examples:
- `/quine:design detect when a device sends transfers to 5+ distinct accounts within 10 minutes`
- `/quine:design find users who log in from 3+ different countries in a day`
- `/quine:design alert when an order ships to a different address than billing`
- `/quine:design track which support tickets are linked to the same underlying issue across channels`
- `/quine:design detect circular payment chains (A pays B pays C pays A)`
