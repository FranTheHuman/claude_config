# /tb:explain — Explain a TigerBeetle / Double-Entry Accounting Concept

Explain a TigerBeetle concept (debits/credits, linked events, two-phase transfers,
strict serializability, asset scale, etc.) in plain terms, grounded in the user's
own domain when possible. Apply knowledge from `.claude/skills/tigerbeetle-java/SKILL.md`.

## Instructions

1. Read `.claude/skills/tigerbeetle-java/SKILL.md` for the relevant section(s)
   before explaining — don't rely on general knowledge alone, the skill has the
   project-specific framing (e.g. how this maps to the app's PostgreSQL + Java setup).
2. If the user has an existing data model in this project (ledgers/account codes
   already defined), frame the explanation using THEIR account types and ledgers
   rather than generic examples.
3. Prefer a short conceptual explanation followed by a minimal, concrete code
   example over a long prose explanation.
4. For debit/credit sign conventions, always clarify which account TYPE (asset,
   liability, equity, income, expense) is being discussed, since the same balance
   change is a debit for one type and a credit for another.
5. If the concept has a common pitfall (e.g. "people expect Exists to be an
   error"), mention it briefly.

## Topics This Command Covers

- Debits vs credits, and why an account's "balance" depends on its type
- Strict serializability and what it guarantees vs traditional isolation levels
- Why TigerBeetle assigns timestamps (and why clients send 0)
- Linked events / chains and atomicity
- Two-phase transfers (pending/post/void) and timeouts
- Idempotency: `id` as idempotency key, `Created` vs `Exists`
- Asset scale and why it can't change later
- `user_data` fields and how they relate to the PostgreSQL control plane
- Batching and why it matters for throughput
- The OLTP vs OLGP split and why TigerBeetle doesn't replace PostgreSQL
- Account flags (`debits_must_not_exceed_credits`, etc.) and balance invariants
- System of Record vs System of Reference, and why TigerBeetle is the former for
  account existence
- Write Last, Read First — why PostgreSQL is written first but TigerBeetle is
  read first to check existence
- Consistent vs Traceable as safety properties, and why temporary inconsistency
  is acceptable but losing traceability is not
- `EXISTS_SAME` vs `EXISTS_DIFFERENT` and why the latter must never be silently
  accepted

## Output Structure

### Concept
1-2 sentence definition.

### In your project's terms
If the user has defined ledgers/accounts/codes in this project, show how the
concept applies to those specifically. Otherwise use a representative example.

### Minimal example
```java
// Small, focused code snippet — not a full recipe unless asked
```

### Common pitfall
One sentence, if applicable.

## Arguments

Usage: `/tb:explain <concept>`

Examples:
- `/tb:explain why is my wallet account a liability and not an asset`
- `/tb:explain what does Exists mean and why isn't it an error`
- `/tb:explain how do linked transfers guarantee atomicity`
- `/tb:explain why can't I change the asset scale later`
- `/tb:explain strict serializability vs what Postgres gives me`
- `/tb:explain when should a pending transfer time out`
- `/tb:explain why does Postgres get written first if TigerBeetle is the source of truth`
- `/tb:explain what happens if my process crashes between the Postgres and TigerBeetle writes`
- `/tb:explain why is an orphaned TigerBeetle account worse than an orphaned Postgres row`