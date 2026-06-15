# /tb:recipe — Generate Java Code for a TigerBeetle Recipe

Generate complete, adapted Java code for a specific TigerBeetle accounting recipe:
two-phase transfers, currency exchange, multi-debit/multi-credit, close account,
balance-conditional transfers, balance bounds, correcting transfers, rate
limiting, or dual-write account creation with PostgreSQL.
Apply knowledge from `.claude/skills/tigerbeetle-java/SKILL.md`.

## Instructions

1. Read `.claude/skills/tigerbeetle-java/SKILL.md` sections 8–10 (Two-Phase,
   Linked Events, Recipes Library) before generating — or section 17 (Dual-Write
   Consistency) if the recipe is "dual-write-account-creation".
2. Identify the requested recipe from the table below. If ambiguous, ask which
   recipe matches the use case.
3. Use the corresponding pattern from the skill as the base structure — do not
   invent a different transfer chain shape (or a different write order, for
   dual-write account creation).
4. Adapt account names, ledger constants, and transfer codes to the user's domain
   (ask for these if not provided, or use sensible placeholders from `AccountCodes`/
   `TransferCodes`/`LedgerCodes`).
5. Always include the `flags.linked` chain correctly — verify the last event in
   any chain has no `linked` flag.
6. Always include idempotency/result handling for the generated transfers.
7. After the code, give a 2-3 sentence explanation of *why* the chain is
   structured this way (e.g. why `AMOUNT_MAX` avoids a balance query, why the
   control account never holds a non-zero balance, why PostgreSQL is written
   before TigerBeetle).

## Recipe Catalog

| Recipe | Trigger phrases | Skill section |
|--------|-----------------|---------------|
| Two-phase transfer | "pending", "hold", "authorize then capture", "reserve funds" | §8 |
| Currency exchange | "exchange", "convert currency", "multi-currency transfer" | §10 |
| Multi-debit/multi-credit | "split payment", "multiple payers", "multiple payees", "compound entry" | §10 |
| Close account | "close account", "zero out balance", "freeze account" | §10 |
| Balance-conditional transfer | "transfer if balance", "check balance before transfer" | §10 |
| Balance bounds | "keep balance between", "upper and lower bound" | §10 |
| Correcting transfer | "reverse a transfer", "correction", "fix a mistake" | §10 |
| Rate limiting | "rate limit", "leaky bucket", "quota", "requests per minute" | §10 |
| Dual-write account creation | "sign up", "onboarding", "create account in both", "Postgres and TigerBeetle", "system of record" | §17 |

## Output Structure

### Recipe: [name]

**Accounts involved**: list with roles (source, destination, control, liquidity, etc.)

```java
// Complete Java method — package, imports, full TransferBatch construction
```

### Why this works

2-3 sentences on the mechanism (linking, AMOUNT_MAX, pending+void, etc.)

### Preconditions

Any account flags that must be set at creation time for this recipe to work
(e.g. `debits_must_not_exceed_credits`).

---

### Output Structure — Dual-Write Account Creation (special case)

This recipe spans two systems, so the output differs from the others above:

**Systems involved**: PostgreSQL (System of Reference) + TigerBeetle (System of
Record).

```java
// Complete AccountOnboardingService — package, imports, the full
// createAccount/pgCreateAccount/tbCreateAccount methods from skill §17.6,
// adapted: table/column names, AccountData fields, ledger/code values
// to the user's domain.
```

### Why this works

2-3 sentences: PostgreSQL first (no commitment) so a crash before the
TigerBeetle write is harmless; TigerBeetle last makes the account real;
reading `tb_account_id` back from PostgreSQL on retry keeps both writes
using the same id.

### Decision table

Reproduce the table from skill §17.5, adapted only if the user's domain
changes which result is "Created" vs "Exists" — the structure itself never
changes.

## Arguments

Usage: `/tb:recipe <recipe-name> [domain details]`

Examples:
- `/tb:recipe two-phase escrow hold for a marketplace order`
- `/tb:recipe currency-exchange USD to ARS with a 0.5% spread`
- `/tb:recipe split-payment ride fare split between driver and platform fee`
- `/tb:recipe close-account user account closure at end of contract`
- `/tb:recipe balance-conditional only transfer if wallet has at least $50`
- `/tb:recipe rate-limit 100 requests per minute per API key`
- `/tb:recipe correcting-transfer reverse an incorrect refund`
- `/tb:recipe dual-write-account-creation user signup creates a wallet account`