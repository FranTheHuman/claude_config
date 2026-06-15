# /tb:review — Review TigerBeetle Java Code for Anti-Patterns

Review existing Java code that uses the TigerBeetle client for correctness,
performance (batching), idempotency handling, and adherence to double-entry
invariants. Apply all rules from `.claude/skills/tigerbeetle-java/SKILL.md`.

## Instructions

1. Read `.claude/skills/tigerbeetle-java/SKILL.md` sections 4, 6, 7, 9, 11, 16
   (Anti-Patterns), and 17 (Dual-Write Consistency) before reviewing.
2. Read the target file(s). If no path given, ask which file(s) to review.
3. Check immediately for hard rule violations — instant CRITICAL:
   - Loop calling `createAccounts`/`createTransfers` once per item
   - `Account.id`/`Transfer.id` from auto-increment, `UUID.randomUUID()`, or
     anything other than `UInt128.id()`
   - `setTimestamp()` called with a non-zero value without `flags.imported`
   - `Exists` treated as an error/exception
   - Same `id` reused for a pending transfer and its post/void
   - A batch ending on an event with `flags.linked` set
   - Metadata (account names, descriptions) fetched from a database inside a
     transfer-creation method
   - Code that calls `update`/`delete` semantics on a transfer (there is no such
     API, but check for workarounds like "soft delete" patterns applied to
     TigerBeetle-mirrored data)
   - **If the code creates an account in both PostgreSQL and TigerBeetle**: the
     TigerBeetle write happening before the PostgreSQL write/commit (violates
     Write Last, Read First — see skill §17.3)
   - **In dual-write account creation**: `ExistsWithDifferent*` (TigerBeetle) or
     a "row exists but data differs" case (PostgreSQL) handled identically to
     `Exists`/`EXISTS_SAME` instead of throwing/alerting (skill §17.4–§17.5)
   - **In dual-write account creation retries**: a fresh `UInt128.id()` generated
     on every attempt instead of reading `tb_account_id` back from the existing
     PostgreSQL row (skill §17.6–§17.7) — this can orphan the PostgreSQL row on retry
4. Check for missing idempotency handling (no switch/check on `CreateTransferStatus`
   / `CreateAccountStatus`).
5. Check client lifecycle: is `Client` created once and reused, or instantiated
   per-request?
6. For any account-existence check ("does this user have an account", "is this
   account active"): confirm it queries TigerBeetle, not just PostgreSQL — a
   PostgreSQL-only check can return true for an account that was never actually
   created in TigerBeetle (skill §17.9).
7. Output the review in the structure below.

## Output Structure

### Summary
One sentence on overall code quality and the most critical issue, if any.

### Issues

Group by severity:

```
[SEVERITY] Short title
Location: file:line (or method name)
Problem:  What is wrong and why it matters (performance, correctness, idempotency,
          financial integrity).
Fix:      Corrected Java code.
```

Severity levels:
- `[CRITICAL]` — financial integrity risk, double-charge potential, or will break
  under retries/network failures
- `[WARNING]` — performance issue (e.g. unbatched calls), missing flags, or
  fragile error handling
- `[SUGGESTION]` — style, readability, or future-proofing (e.g. asset scale choice)

### Verdict
- ✅ Approved — correct and production-ready
- ⚠️ Approved with warnings — functional but needs improvement before scale
- ❌ Blocked — critical issues must be fixed before this touches real money

## Arguments

Usage: `/tb:review [path/to/file.java]`

Examples:
- `/tb:review src/main/java/com/yourapp/ledger/TransferService.java`
- `/tb:review src/main/java/com/yourapp/ledger/` → review all files in directory
- `/tb:review` → review whatever code is in context