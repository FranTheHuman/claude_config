# /tb:model — Design the TigerBeetle Data Model for a Domain

Design the ledger, account code, and transfer code schema for a specific business
domain: a chart of accounts mapped to TigerBeetle's integer-based model, plus
recommendations for `user_data` field usage and asset scale.
Apply knowledge from `.claude/skills/tigerbeetle-java/SKILL.md`.

## Instructions

1. Read `.claude/skills/tigerbeetle-java/SKILL.md` sections 1 and 5 (Architecture
   Context, Data Modeling) before designing anything.
2. If the domain description is vague, ask what's being tracked: currencies/assets
   involved, who the accounts represent (users, operator, third parties), and
   whether multi-currency or currency exchange is needed.
3. For each currency/asset type identified, assign a `ledger` constant and
   determine the asset scale — flag if a higher scale should be used to allow for
   future precision needs (asset scale cannot change later).
4. List all account types needed (operator accounts, user accounts, control
   accounts, liquidity accounts) with their `code` values and required flags
   (`debits_must_not_exceed_credits`, `credits_must_not_exceed_debits`, `history`).
5. List transfer types (`code` values) for the domain's business events.
6. Recommend `user_data_128/64/32` usage for linking to PostgreSQL records.
7. Generate the three Java constants classes (`LedgerCodes`, `AccountCodes`,
   `TransferCodes`) as the final output.

## Output Structure

### Domain Summary
One paragraph restating the domain and the currencies/assets/account types
identified.

### Ledgers

| Ledger constant | Value | Represents | Asset scale | Notes |
|-----------------|-------|-----------|-------------|-------|

### Account Types (Chart of Accounts)

| Account code constant | Value | Account type (asset/liability/etc.) | Flags | Notes |
|-----------------------|-------|--------------------------------------|-------|-------|

### Transfer Types

| Transfer code constant | Value | Represents |
|------------------------|-------|-----------|

### user_data Recommendations

- `user_data_128`: ...
- `user_data_64`: ...
- `user_data_32`: ...

### Generated Java

```java
// LedgerCodes.java, AccountCodes.java, TransferCodes.java
// Complete, ready to paste
```

### Open Questions / Things to Confirm

Anything ambiguous that affects irreversible decisions (especially asset scale).

## Arguments

Usage: `/tb:model <domain description>`

Examples:
- `/tb:model multi-currency wallet app, users hold USD and EUR balances`
- `/tb:model marketplace with buyers, sellers, and platform fees in USD`
- `/tb:model API usage billing — per-request and per-MB rate limiting`
- `/tb:model ride-sharing payments with driver payouts and platform commission`
