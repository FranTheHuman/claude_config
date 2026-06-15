---
name: tigerbeetle-java
description: >
  Expert knowledge for building new Java applications on TigerBeetle, the OLTP
  database for financial accounting based on double-entry bookkeeping. Covers
  project setup with Maven (plain Java, Quarkus, or Spring Boot), client lifecycle
  and configuration, ID generation and idempotency, data modeling (ledgers, account
  codes, transfer codes, user_data fields), batching accounts and transfers, two-phase
  transfers, linked event chains, the full recipes library (currency exchange,
  multi-debit/multi-credit, close account, balance-conditional transfers, balance
  bounds, correcting transfers, rate limiting), error handling and result codes,
  testing strategy, and deployment. Designed for greenfield Java projects that need
  a ledger alongside an existing OLGP database (PostgreSQL).
---

You are an expert TigerBeetle developer specialized in Java integrations for new
projects. Apply this skill as the authoritative source for every decision. The
guiding principle: **TigerBeetle is the ledger, not the application** — it enforces
debit/credit invariants and immutability; everything else (auth, metadata, business
rules that don't affect balances) lives in the application and the OLGP database.

---

## 1. Architecture Context — Where TigerBeetle Fits

```
┌─────────────────────────────────────────────────────────────────┐
│  App / Website                                                     │
│  - Initiates transactions                                          │
│  - Generates Transfer and Account IDs (time-based, client-side)    │
└───────────────────────────┬───────────────────────────────────────┘
                             │
┌────────────────────────────▼──────────────────────────────────────┐
│  Stateless API Service (Java — Quarkus / Spring Boot)               │
│  - Authentication and authorization                                  │
│  - Creates account records in BOTH the OLGP DB and TigerBeetle       │
│    when users sign up                                                 │
│  - Caches ledger/account/transfer code mappings (never fetched        │
│    on the hot path of a transfer)                                     │
│  - Batches transfers                                                   │
│  - Applies exchange rates for currency exchange                       │
└───────────────┬─────────────────────────────────┬────────────────────┘
                │                                 │
┌────────────────▼─────────────────┐ ┌────────────▼─────────────────────┐
│  PostgreSQL (OLGP — control plane) │ │  TigerBeetle (OLTP — data plane)   │
│  - Ledger/account/code metadata    │ │  - Records transfers between       │
│    (string names, descriptions)    │ │    accounts                        │
│  - Mapping: int code <-> string    │ │  - Tracks balances                 │
│  - Anything updated infrequently   │ │  - Enforces balance limits          │
│                                     │ │  - Enforces double-entry invariants │
│                                     │ │  - Strict serializability           │
│                                     │ │  - user_data fields point back to   │
│                                     │ │    records in this DB               │
└─────────────────────────────────────┘ └─────────────────────────────────────┘
```

### Critical rule: no metadata lookups on the transfer hot path

⚠️ Initiating a transfer must NOT require a query to PostgreSQL for metadata.
Ledger, account code, and transfer code mappings (e.g. `"USD"` → `ledger=700`,
`"refund"` → `code=9000`) must be **hard-coded as enums or cached in memory** —
loaded once at startup, never re-fetched per request. These mappings are immutable
and append-only, so cache invalidation is a non-issue.

### TigerBeetle has no authentication

TigerBeetle never sits on the public internet and never receives requests directly
from untrusted clients. The API service is the only thing that talks to TigerBeetle.
The data file must not be readable by untrusted processes.

### Creating an account touches BOTH systems — order matters

"Creates account records in both the OLGP DB and TigerBeetle" above is not a
detail to improvise. PostgreSQL and TigerBeetle do not share a transaction
boundary, so the order of these two writes — and how you handle retries —
determines whether your system can ever end up with money in an account nobody
can identify. **See section 17 (Dual-Write Consistency) before writing any account
creation code.** This is the single most important architectural rule in this
skill after batching.

---

## 2. Project Setup — Maven

### Dependency (pom.xml)

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.yourapp</groupId>
  <artifactId>ledger-service</artifactId>
  <version>1.0-SNAPSHOT</version>

  <properties>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

  <dependencies>
    <dependency>
      <groupId>com.tigerbeetle</groupId>
      <artifactId>tigerbeetle-java</artifactId>
      <!-- Check latest: https://repo1.maven.org/maven2/com/tigerbeetle/tigerbeetle-java/maven-metadata.xml -->
      <version>0.16.43</version>
    </dependency>
  </dependencies>
</project>
```

### Recommended project structure

```
src/main/java/com/yourapp/ledger/
├── TigerBeetleConfig.java       # client bean / producer
├── LedgerCodes.java              # ledger int constants (currencies, asset types)
├── AccountCodes.java             # account type int constants
├── TransferCodes.java            # transfer reason int constants
├── AccountService.java           # create/lookup accounts
├── TransferService.java          # create/lookup transfers, batching
├── recipes/
│   ├── CurrencyExchangeService.java
│   ├── RateLimitService.java
│   └── AccountClosingService.java
└── exceptions/
    ├── LedgerException.java
    └── TransferRejectedException.java
```

### Required Linux capability for memory locking

TigerBeetle requires `RLIMIT_MEMLOCK` (for `io_uring` and to prevent swap). If
running the API service and TigerBeetle on the same host (development), no action
needed beyond what's in section 13. In Docker, see section 13.

---

## 3. Client Lifecycle and Configuration

The `Client` is **thread-safe and must be a singleton**, shared across the entire
application. Creating multiple clients defeats automatic batching. Close it only on
application shutdown.

### Configuration properties (application.properties / application.yml)

```properties
# Quarkus / Spring Boot — application.properties
tigerbeetle.cluster-id=0
tigerbeetle.addresses=127.0.0.1:3000
```

```yaml
# application.yml
tigerbeetle:
  cluster-id: 0
  addresses: 127.0.0.1:3000
```

In production, `addresses` is a comma-separated list of all 6 replicas:
`10.0.1.10:3000,10.0.1.11:3000,10.0.1.12:3000,10.0.2.10:3000,10.0.2.11:3000,10.0.2.12:3000`

### Quarkus — CDI producer (recommended if pairing with quarkus-master stack)

```java
package com.yourapp.ledger;

import com.tigerbeetle.Client;
import com.tigerbeetle.UInt128;
import io.quarkus.runtime.ShutdownEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.enterprise.inject.Produces;
import jakarta.inject.Singleton;
import org.eclipse.microprofile.config.inject.ConfigProperty;

@ApplicationScoped
public class TigerBeetleConfig {

    private Client client;

    @Produces
    @Singleton
    public Client tigerBeetleClient(
            @ConfigProperty(name = "tigerbeetle.cluster-id", defaultValue = "0") long clusterId,
            @ConfigProperty(name = "tigerbeetle.addresses") String addresses) {

        byte[] clusterIdBytes = UInt128.asBytes(clusterId);
        String[] replicaAddresses = addresses.split(",");
        this.client = new Client(clusterIdBytes, replicaAddresses);
        return this.client;
    }

    void onShutdown(@Observes ShutdownEvent event) {
        if (client != null) {
            client.close();
        }
    }
}
```

### Spring Boot — @Configuration bean

```java
package com.yourapp.ledger;

import com.tigerbeetle.Client;
import com.tigerbeetle.UInt128;
import org.springframework.beans.factory.DisposableBean;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class TigerBeetleConfig {

    @Bean(destroyMethod = "close")
    public Client tigerBeetleClient(
            @Value("${tigerbeetle.cluster-id:0}") long clusterId,
            @Value("${tigerbeetle.addresses}") String addresses) {

        byte[] clusterIdBytes = UInt128.asBytes(clusterId);
        String[] replicaAddresses = addresses.split(",");
        return new Client(clusterIdBytes, replicaAddresses);
    }
}
```

`destroyMethod = "close"` ensures Spring closes the client on context shutdown —
no manual `DisposableBean` needed.

---

## 4. ID Generation and Reliable Transaction Submission

### Always use TigerBeetle time-based IDs

```java
import com.tigerbeetle.UInt128;

// Generates a 128-bit ID: high 48 bits = millisecond timestamp, low 80 bits = random
byte[] transferId = UInt128.id();
```

These IDs are lexicographically sortable, have negligible collision risk, require
no central oracle, and are optimized for TigerBeetle's LSM storage. **Never use
auto-increment IDs from PostgreSQL for Account.id or Transfer.id** — that creates a
central bottleneck and defeats LSM optimizations.

### Who generates the ID?

The **client-facing application** (mobile app, web frontend) generates the ID,
persists it locally (e.g. browser storage or app local DB) *before* submitting,
and reuses the same ID on every retry. This makes the entire chain — app → API →
TigerBeetle — idempotent end to end.

```
1. User initiates a transfer
2. App generates transfer.id = UInt128.id()  (or equivalent in the client's language)
3. App persists the id locally
4. App submits {id, debit_account_id, credit_account_id, amount, ...} to the API
5. API forwards this in a create_transfers request
6. TigerBeetle creates the transfer with this id once and only once
7. TigerBeetle replies created (first time) or exists (retry)
8. API and app treat both created and exists as success
```

If your API itself generates IDs (e.g. server-initiated transfers, batch jobs),
generate with `UInt128.id()` at the point of creation and persist the ID before
calling TigerBeetle, so retries after a crash reuse it.

### Idempotency handling pattern (always use this)

```java
import com.tigerbeetle.CreateTransferResultBatch;
import com.tigerbeetle.CreateTransferStatus;

CreateTransferResultBatch results = client.createTransfers(transfers);
while (results.next()) {
    switch (results.getStatus()) {
        case Created:
            // New transfer recorded — proceed normally
            log.info("Transfer created at timestamp {}", results.getTimestamp());
            break;
        case Exists:
            // Retry of a previously successful transfer — treat as success
            log.info("Transfer already existed (idempotent retry), timestamp {}",
                     results.getTimestamp());
            break;
        case ExceedsCredits:
        case ExceedsDebits:
            // Business-level rejection — not a system error
            throw new TransferRejectedException(results.getPosition(), results.getStatus());
        default:
            // Validation error — likely a bug in request construction
            throw new LedgerException("Transfer failed: " + results.getStatus());
    }
}
```

**Created and Exists are both success.** Only treat other statuses as failures, and
distinguish business rejections (`ExceedsCredits`, `ExceedsDebits`,
`PendingTransferAlreadyPosted`, etc.) from validation errors (`*MustNotBeZero`,
`*MustBeNonNegative`, etc.) — the former are expected outcomes your application
logic should handle gracefully, the latter indicate a bug.

---

## 5. Data Modeling — Ledgers, Codes, and user_data

### Ledgers — one per currency / asset type

```java
package com.yourapp.ledger;

/**
 * Ledger IDs partition accounts by currency or asset type.
 * Transfers can only occur between accounts on the SAME ledger
 * (use linked transfers across ledgers for currency exchange — see recipes).
 *
 * Keep this enum append-only. Never reuse or change a value.
 */
public final class LedgerCodes {
    public static final int USD = 1;
    public static final int EUR = 2;
    public static final int ARS = 3;
    // Asset scale for ARS is 2 (cents) — see "Asset Scale" notes below

    private LedgerCodes() {}
}
```

### Account codes — the "why" of an account

```java
package com.yourapp.ledger;

/**
 * Account.code — represents the account's role in your chart of accounts.
 * Map these to human-readable names in PostgreSQL for reporting,
 * but the integer mapping itself is immutable and append-only.
 */
public final class AccountCodes {
    // Operator's accounts
    public static final int OPERATOR_CASH = 1000;        // Asset
    public static final int OPERATOR_REVENUE = 2000;      // Income
    public static final int OPERATOR_FEES_PAYABLE = 3000; // Liability

    // User accounts
    public static final int USER_WALLET = 1100;           // Asset (from user's perspective)
                                                            // Liability (from operator's perspective)

    // Control accounts (used in recipes — see section 10)
    public static final int CONTROL_ACCOUNT = 9000;
    public static final int LIQUIDITY_PROVIDER = 9100;

    private AccountCodes() {}
}
```

### Transfer codes — the "why" of a transfer

```java
package com.yourapp.ledger;

public final class TransferCodes {
    public static final int DEPOSIT = 100;
    public static final int WITHDRAWAL = 101;
    public static final int PAYMENT = 200;
    public static final int REFUND = 201;
    public static final int FEE = 300;
    public static final int CURRENCY_EXCHANGE = 400;
    public static final int CORRECTION = 900;

    private TransferCodes() {}
}
```

### Asset Scale

The smallest useful unit of a currency maps to integer `1`. For USD with asset
scale 2 (cents): `$123.45` → `12345`. **Asset scale cannot be changed after data
exists** on a ledger — if you might need finer precision later, use a higher scale
from the start (e.g. scale 4 instead of 2 for currencies that might need fractional
cents in the future).

### user_data fields — pointing back to PostgreSQL

```java
// user_data_128: "who"/"what" — e.g. a UUID pointing to a row in PostgreSQL,
//                 encoded as the high/low bits of a UUID
transfers.setUserData128(userUuid.getMostSignificantBits(), userUuid.getLeastSignificantBits());

// user_data_64: secondary "when" — real-world transaction time, if different
//                from TigerBeetle's cluster timestamp
transfers.setUserData64(realWorldTimestampMillis);

// user_data_32: "where" — e.g. a region/jurisdiction code
transfers.setUserData32(jurisdictionCode);
```

All `user_data` fields are indexed for point and range queries via
`query_transfers` / `get_account_transfers`.

---

## 6. Accounts — Create and Lookup

> ⚠️ The `createUserWallet` example below shows TigerBeetle account creation in
> isolation. If the account also needs a corresponding row in PostgreSQL (almost
> always true for user-facing accounts), do **not** call this in isolation — use
> the dual-write orchestration pattern in **section 17** instead. This section
> covers the TigerBeetle-only mechanics that section 17 builds on.

### Create accounts (always batched)

```java
package com.yourapp.ledger;

import com.tigerbeetle.*;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class AccountService {

    @Inject Client client;

    /**
     * Creates a new user wallet account on the given ledger.
     * Returns the generated account ID.
     */
    public byte[] createUserWallet(int ledger) {
        byte[] accountId = UInt128.id();

        AccountBatch accounts = new AccountBatch(1);
        accounts.add();
        accounts.setId(accountId);
        accounts.setLedger(ledger);
        accounts.setCode(AccountCodes.USER_WALLET);
        // Wallets must never go negative from the operator's perspective
        accounts.setFlags(AccountFlags.CREDITS_MUST_NOT_EXCEED_DEBITS);

        CreateAccountResultBatch results = client.createAccounts(accounts);
        while (results.next()) {
            switch (results.getStatus()) {
                case Created:
                case Exists:
                    return accountId; // idempotent
                default:
                    throw new LedgerException("Account creation failed: " + results.getStatus());
            }
        }
        return accountId;
    }
}
```

### Account flags reference

| Flag | Effect |
|------|--------|
| `LINKED` | Chains this account's creation with the next event |
| `DEBITS_MUST_NOT_EXCEED_CREDITS` | Enforces non-negative debit-side balance (asset/expense accounts that shouldn't go negative) |
| `CREDITS_MUST_NOT_EXCEED_DEBITS` | Enforces non-negative credit-side balance (liability accounts, e.g. user wallets, that shouldn't go negative) |
| `HISTORY` | Retains historical point-in-time balances, queryable via `get_account_balances` |
| `CLOSED` | Account is closed — rejects new transfers (set via closing transfer, see recipes) |

### Lookup accounts (batched)

```java
public List<AccountSnapshot> lookupAccounts(byte[]... ids) {
    IdBatch idBatch = new IdBatch(ids.length);
    for (byte[] id : ids) {
        idBatch.add(id);
    }

    AccountBatch accounts = client.lookupAccounts(idBatch);
    List<AccountSnapshot> results = new ArrayList<>();
    while (accounts.next()) {
        results.add(new AccountSnapshot(
            accounts.getId(UInt128.LeastSignificant),
            accounts.getDebitsPosted(),
            accounts.getCreditsPosted(),
            accounts.getDebitsPending(),
            accounts.getCreditsPending()
        ));
    }
    return results; // may be smaller than ids.length — missing accounts are simply absent
}
```

---

## 7. Transfers — Create, Batch, and Lookup

### Create transfers with batching (handles >8189 events)

```java
package com.yourapp.ledger;

import com.tigerbeetle.*;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.math.BigInteger;
import java.util.List;

@ApplicationScoped
public class TransferService {

    private static final int MAX_BATCH_SIZE = 8189;

    @Inject Client client;

    public record TransferRequest(
        byte[] id, byte[] debitAccountId, byte[] creditAccountId,
        BigInteger amount, int ledger, int code) {}

    /**
     * Creates transfers in batches of up to 8189. Returns the number
     * of newly created transfers (Exists results are not counted as new).
     */
    public int createTransfers(List<TransferRequest> requests) {
        int created = 0;

        for (int offset = 0; offset < requests.size(); offset += MAX_BATCH_SIZE) {
            int end = Math.min(offset + MAX_BATCH_SIZE, requests.size());
            List<TransferRequest> chunk = requests.subList(offset, end);

            TransferBatch transfers = new TransferBatch(chunk.size());
            for (TransferRequest req : chunk) {
                transfers.add();
                transfers.setId(req.id());
                transfers.setDebitAccountId(req.debitAccountId());
                transfers.setCreditAccountId(req.creditAccountId());
                transfers.setAmount(req.amount());
                transfers.setLedger(req.ledger());
                transfers.setCode(req.code());
                // timestamp left as 0 — TigerBeetle assigns it
            }

            CreateTransferResultBatch results = client.createTransfers(transfers);
            while (results.next()) {
                switch (results.getStatus()) {
                    case Created -> created++;
                    case Exists -> { /* idempotent retry — not new, not an error */ }
                    case ExceedsCredits, ExceedsDebits ->
                        throw new TransferRejectedException(results.getPosition(), results.getStatus());
                    default ->
                        throw new LedgerException("Transfer failed at position "
                            + results.getPosition() + ": " + results.getStatus());
                }
            }
        }
        return created;
    }
}
```

### Single transfer convenience method

```java
public boolean transfer(byte[] id, byte[] from, byte[] to, BigInteger amount, int ledger, int code) {
    return createTransfers(List.of(new TransferRequest(id, from, to, amount, ledger, code))) >= 0;
}
```

### Querying account history

```java
public List<TransferSnapshot> getAccountTransfers(byte[] accountId, int limit, boolean reversed) {
    AccountFilter filter = new AccountFilter();
    filter.setAccountId(accountId);
    filter.setLimit(limit);
    filter.setDebits(true);
    filter.setCredits(true);
    filter.setReversed(reversed); // true = most recent first

    TransferBatch transfers = client.getAccountTransfers(filter);
    // ... map to snapshots
}
```

⚠️ Only accounts created with `flags.history` retain point-in-time balances for
`get_account_balances`. Set this flag at account creation if you need balance
history — it cannot be added retroactively.

---

## 8. Two-Phase Transfers (Pending → Post / Void)

Use two-phase transfers whenever funds need to be **reserved** before they're
confirmed — holds, escrow, authorization-then-capture flows.

### Reserve funds (pending)

```java
TransferBatch pending = new TransferBatch(1);
pending.add();
byte[] pendingId = UInt128.id();
pending.setId(pendingId);
pending.setDebitAccountId(userAccountId);
pending.setCreditAccountId(merchantAccountId);
pending.setAmount(new BigInteger("5000")); // $50.00 at scale 2
pending.setLedger(LedgerCodes.USD);
pending.setCode(TransferCodes.PAYMENT);
pending.setTimeout(3600); // expires in 1 hour if not resolved
pending.setFlags(TransferFlags.PENDING);

client.createTransfers(pending);
```

### Post the full amount (confirm)

```java
TransferBatch post = new TransferBatch(1);
post.add();
post.setId(UInt128.id());           // NEW id — never reuse the pending transfer's id
post.setPendingId(pendingId);        // references the original
post.setAmount(TransferBatch.AMOUNT_MAX); // posts the full pending amount
post.setFlags(TransferFlags.POST_PENDING_TRANSFER);

client.createTransfers(post);
```

### Post a partial amount

```java
post.setAmount(new BigInteger("3000")); // posts $30.00, restores $20.00 to source
```

### Void (release the hold entirely)

```java
TransferBatch voidTransfer = new TransferBatch(1);
voidTransfer.add();
voidTransfer.setId(UInt128.id());
voidTransfer.setAmount(BigInteger.ZERO);
voidTransfer.setPendingId(pendingId);
voidTransfer.setFlags(TransferFlags.VOID_PENDING_TRANSFER);

client.createTransfers(voidTransfer);
```

### Rules

- A pending transfer can be posted or voided **only once**. Retrying with the same
  id is idempotent (`Exists`), but trying to post an already-voided transfer
  returns `PendingTransferAlreadyVoided`.
- The post/void transfer has its **own new id** — never the pending transfer's id.
- If no post/void happens before `timeout` seconds, the cluster automatically
  voids the transfer and restores the funds.
- `debit_account_id`, `credit_account_id`, `ledger`, `code` on the post/void
  transfer may be left as zero or must match the pending transfer's values.

---

## 9. Linked Event Chains

Use `flags.linked` to make a sequence of transfers (or accounts) succeed or fail
**atomically as a unit**. The chain ends at the first event without `linked` set.

```java
TransferBatch transfers = new TransferBatch(3);

// Transfer 1: linked — part of the chain
transfers.add();
transfers.setId(UInt128.id());
transfers.setDebitAccountId(sourceAccount);
transfers.setCreditAccountId(feeAccount);
transfers.setAmount(feeAmount);
transfers.setLedger(LedgerCodes.USD);
transfers.setCode(TransferCodes.FEE);
transfers.setFlags(TransferFlags.LINKED);

// Transfer 2: linked — part of the chain
transfers.add();
transfers.setId(UInt128.id());
transfers.setDebitAccountId(sourceAccount);
transfers.setCreditAccountId(taxAccount);
transfers.setAmount(taxAmount);
transfers.setLedger(LedgerCodes.USD);
transfers.setCode(TransferCodes.FEE);
transfers.setFlags(TransferFlags.LINKED);

// Transfer 3: NOT linked — closes the chain
transfers.add();
transfers.setId(UInt128.id());
transfers.setDebitAccountId(sourceAccount);
transfers.setCreditAccountId(destinationAccount);
transfers.setAmount(netAmount);
transfers.setLedger(LedgerCodes.USD);
transfers.setCode(TransferCodes.PAYMENT);
// no flags — ends the chain

CreateTransferResultBatch results = client.createTransfers(transfers);
```

If transfer 3 fails (e.g. `ExceedsCredits`), transfers 1 and 2 also fail with
`LinkedEventFailed`, and **none** of the three are committed. The association
between linked transfers is NOT persisted — if you need to query "these 3
transfers were part of the same operation," encode a shared ID in `user_data_128`.

⚠️ **The last event in a batch must never have `flags.linked` set** — this leaves
an open-ended chain and returns `LinkedEventChainOpen`.

---

## 10. Recipes Library

All recipes below use `flags.linked` to guarantee atomicity. Helper:
`id()` means `UInt128.id()`.

### Currency Exchange

Two linked transfers across different ledgers, mediated by liquidity accounts.

```java
public void exchangeCurrency(byte[] sourceAccount, byte[] destAccount,
                              BigInteger sourceAmount, BigInteger destAmount) {
    TransferBatch transfers = new TransferBatch(2);

    // T1: source -> source liquidity (USD)
    transfers.add();
    transfers.setId(UInt128.id());
    transfers.setDebitAccountId(sourceAccount);
    transfers.setCreditAccountId(usdLiquidityAccount);
    transfers.setAmount(sourceAmount);
    transfers.setLedger(LedgerCodes.USD);
    transfers.setCode(TransferCodes.CURRENCY_EXCHANGE);
    transfers.setFlags(TransferFlags.LINKED);

    // T2: dest liquidity -> destination (INR)
    transfers.add();
    transfers.setId(UInt128.id());
    transfers.setDebitAccountId(inrLiquidityAccount);
    transfers.setCreditAccountId(destAccount);
    transfers.setAmount(destAmount);
    transfers.setLedger(LedgerCodes.INR);
    transfers.setCode(TransferCodes.CURRENCY_EXCHANGE);
    // no flag — closes the chain

    client.createTransfers(transfers);
}
```

To add a spread/fee, insert an extra linked USD transfer from the source account
to the liquidity account *before* T2, rather than adjusting the exchange rate —
this preserves an auditable record of the rate and fee separately.

### Multi-Debit, Multi-Credit (via Control Account)

```java
public void splitPayment(byte[] payerA, byte[] payerB,
                          byte[] payeeX, byte[] payeeY, byte[] payeeZ,
                          BigInteger amountA, BigInteger amountB,
                          BigInteger amountX, BigInteger amountY, BigInteger amountZ) {
    TransferBatch transfers = new TransferBatch(5);

    // Debits into the control account
    transfers.add();
    transfers.setId(UInt128.id());
    transfers.setDebitAccountId(payerA);
    transfers.setCreditAccountId(controlAccount);
    transfers.setAmount(amountA);
    transfers.setLedger(LedgerCodes.USD);
    transfers.setCode(TransferCodes.PAYMENT);
    transfers.setFlags(TransferFlags.LINKED);

    transfers.add();
    transfers.setId(UInt128.id());
    transfers.setDebitAccountId(payerB);
    transfers.setCreditAccountId(controlAccount);
    transfers.setAmount(amountB);
    transfers.setLedger(LedgerCodes.USD);
    transfers.setCode(TransferCodes.PAYMENT);
    transfers.setFlags(TransferFlags.LINKED);

    // Credits out of the control account
    transfers.add();
    transfers.setId(UInt128.id());
    transfers.setDebitAccountId(controlAccount);
    transfers.setCreditAccountId(payeeX);
    transfers.setAmount(amountX);
    transfers.setLedger(LedgerCodes.USD);
    transfers.setCode(TransferCodes.PAYMENT);
    transfers.setFlags(TransferFlags.LINKED);

    transfers.add();
    transfers.setId(UInt128.id());
    transfers.setDebitAccountId(controlAccount);
    transfers.setCreditAccountId(payeeY);
    transfers.setAmount(amountY);
    transfers.setLedger(LedgerCodes.USD);
    transfers.setCode(TransferCodes.PAYMENT);
    transfers.setFlags(TransferFlags.LINKED);

    transfers.add();
    transfers.setId(UInt128.id());
    transfers.setDebitAccountId(controlAccount);
    transfers.setCreditAccountId(payeeZ);
    transfers.setAmount(amountZ);
    transfers.setLedger(LedgerCodes.USD);
    transfers.setCode(TransferCodes.PAYMENT);
    // closes the chain

    client.createTransfers(transfers);
}
```

If the sums on each side don't match, the chain fails entirely — the control
account never holds a non-zero balance after a successful chain.

### Close Account (Zero Balance + Reject Future Transfers)

```java
public void closeAccount(byte[] accountId, byte[] controlAccountId, int ledger) {
    TransferBatch transfers = new TransferBatch(2);

    // T1: balancing transfer — moves the account's entire net balance
    // to the control account, using AMOUNT_MAX so you don't need to
    // query the balance first
    transfers.add();
    transfers.setId(UInt128.id());
    transfers.setDebitAccountId(accountId);
    transfers.setCreditAccountId(controlAccountId);
    transfers.setAmount(TransferBatch.AMOUNT_MAX);
    transfers.setLedger(ledger);
    transfers.setCode(TransferCodes.CORRECTION);
    transfers.setFlags(TransferFlags.LINKED
                      | TransferFlags.BALANCING_DEBIT);

    // T2: closing transfer — must also be pending so it's reversible
    transfers.add();
    transfers.setId(UInt128.id());
    transfers.setDebitAccountId(accountId);
    transfers.setCreditAccountId(controlAccountId);
    transfers.setAmount(BigInteger.ZERO);
    transfers.setLedger(ledger);
    transfers.setCode(TransferCodes.CORRECTION);
    transfers.setFlags(TransferFlags.CLOSING_DEBIT | TransferFlags.PENDING);

    client.createTransfers(transfers);
}
```

To re-open: void the pending closing transfer (T2's id) with
`flags.void_pending_transfer` — this reverts the `closed` flag but not the balance.

### Balance-Conditional Transfer

Execute a transfer only if the source account has at least `threshold` balance,
checked atomically:

```java
public boolean transferIfBalanceAtLeast(byte[] source, byte[] destination, byte[] controlAccount,
                                         BigInteger threshold, BigInteger transferAmount, int ledger) {
    byte[] checkId = UInt128.id();

    TransferBatch transfers = new TransferBatch(3);

    // T1: reserve `threshold` from source into control (pending)
    transfers.add();
    transfers.setId(checkId);
    transfers.setDebitAccountId(source);
    transfers.setCreditAccountId(controlAccount);
    transfers.setAmount(threshold);
    transfers.setLedger(ledger);
    transfers.setCode(TransferCodes.CORRECTION);
    transfers.setFlags(TransferFlags.LINKED | TransferFlags.PENDING);

    // T2: immediately void T1 — restores the threshold if T1 succeeded
    transfers.add();
    transfers.setId(UInt128.id());
    transfers.setPendingId(checkId);
    transfers.setAmount(BigInteger.ZERO);
    transfers.setFlags(TransferFlags.LINKED | TransferFlags.VOID_PENDING_TRANSFER);

    // T3: the actual transfer — only committed if T1 succeeded
    transfers.add();
    transfers.setId(UInt128.id());
    transfers.setDebitAccountId(source);
    transfers.setCreditAccountId(destination);
    transfers.setAmount(transferAmount);
    transfers.setLedger(ledger);
    transfers.setCode(TransferCodes.PAYMENT);
    // closes the chain

    CreateTransferResultBatch results = client.createTransfers(transfers);
    // If T1 fails (ExceedsCredits on source, since source has
    // debits_must_not_exceed_credits / credits_must_not_exceed_debits),
    // T2 and T3 fail with LinkedEventFailed — return false in that case.
    while (results.next()) {
        if (results.getStatus() != CreateTransferStatus.Created
            && results.getStatus() != CreateTransferStatus.Exists) {
            return false;
        }
    }
    return true;
}
```

Precondition: `source` must have `debits_must_not_exceed_credits` (credit balance)
or `credits_must_not_exceed_debits` (debit balance) set at creation.

### Correcting Transfers

Never modify or delete a transfer. Reverse it with a new transfer in the opposite
direction, using the same `user_data_128` to link them in queries:

```java
public void correctTransfer(byte[] originalDebit, byte[] originalCredit,
                             BigInteger correctionAmount, long correctionLinkId, int ledger) {
    TransferBatch transfers = new TransferBatch(1);
    transfers.add();
    transfers.setId(UInt128.id());
    // Reversed direction from the original
    transfers.setDebitAccountId(originalCredit);
    transfers.setCreditAccountId(originalDebit);
    transfers.setAmount(correctionAmount);
    transfers.setLedger(ledger);
    transfers.setCode(TransferCodes.CORRECTION);
    transfers.setUserData128(correctionLinkId, correctionLinkId); // same link as original

    client.createTransfers(transfers);
}
```

### Rate Limiting (Leaky Bucket)

Use a dedicated ledger per resource type (requests, bandwidth, spend). Each user
gets a balance-limited account; each request is a pending transfer with a timeout
that auto-expires.

```java
public final class RateLimitLedgers {
    public static final int REQUEST_RATE = 100; // separate ledger, not a currency
    private RateLimitLedgers() {}
}

@ApplicationScoped
public class RateLimitService {

    @Inject Client client;

    /** Call once per user, at account creation. Grants 10 requests/minute. */
    public void grantRequestQuota(byte[] userAccount, BigInteger requestsPerMinute) {
        TransferBatch t = new TransferBatch(1);
        t.add();
        t.setId(UInt128.id());
        t.setDebitAccountId(operatorAccount);
        t.setCreditAccountId(userAccount);
        t.setAmount(requestsPerMinute);
        t.setLedger(RateLimitLedgers.REQUEST_RATE);
        t.setCode(TransferCodes.DEPOSIT);
        client.createTransfers(t);
    }

    /**
     * Call on each incoming request. Returns true if the request is allowed.
     * Pending transfers auto-expire after 60s, replenishing the quota.
     */
    public boolean checkAndConsumeRequest(byte[] userAccount) {
        TransferBatch t = new TransferBatch(1);
        t.add();
        t.setId(UInt128.id());
        t.setDebitAccountId(userAccount);
        t.setCreditAccountId(operatorAccount);
        t.setAmount(BigInteger.ONE);
        t.setLedger(RateLimitLedgers.REQUEST_RATE);
        t.setCode(TransferCodes.FEE);
        t.setTimeout(60); // expires in 60 seconds, restoring the quota
        t.setFlags(TransferFlags.PENDING);

        CreateTransferResultBatch results = client.createTransfers(t);
        while (results.next()) {
            return results.getStatus() == CreateTransferStatus.Created
                || results.getStatus() == CreateTransferStatus.Exists;
            // ExceedsCredits => quota exhausted => return false
        }
        return false;
    }
}
```

The user account must have `flags.debits_must_not_exceed_credits` so that
`checkAndConsumeRequest` fails with `ExceedsCredits` once quota is exhausted.

---

## 11. Error Handling and Result Codes

### Account/Transfer creation status precedence (0.16.4+)

1. **Idempotency checks first**: `Exists` / `ExistsWithDifferent*` are checked
   before semantic validation. Submitting a transfer with an existing `id` but
   different fields returns `ExistsWithDifferentAmount` (etc.), not a field
   validation error.
2. **Validation errors**: `*MustNotBeZero`, `*MustNotBeNegative`,
   `DebitAccountIdMustNotBeCreditAccountId`, etc. — these indicate a bug in
   request construction, not a business outcome.
3. **Semantic / business errors**: `ExceedsCredits`, `ExceedsDebits`,
   `AccountClosed`, `PendingTransferAlreadyPosted`, etc. — these are expected
   outcomes the application must handle.

### Transient failures and `id_already_failed`

If a transfer fails with a **transient** error (e.g. `ExceedsCredits` because the
account didn't have enough balance *at that moment*), TigerBeetle guarantees that
**retrying with the same id will never succeed later** — it returns
`IdAlreadyFailed`. If you need to retry the same logical operation after a
transient failure, generate a **new id**.

### `Exists` vs `ExistsWithDifferent*` — collapse to a 3-way result

For **transfers**, `Exists` (idempotent retry, identical fields) is the only
"already existed" case you'll normally see, because the application controls every
field and a retry resends the exact same request. Treat `Created` and `Exists` the
same, as shown in sections 6–7.

For **accounts created as part of a dual-write with PostgreSQL** (section 17), you
must additionally distinguish `Exists` (same data — a safe retry) from any
`ExistsWithDifferentLedger` / `ExistsWithDifferentCode` / `ExistsWithDifferentFlags`
/ `ExistsWithDifferentUserData*` (different data — a conflict that must **not** be
silently accepted). Collapse the family into a `SyncResult` enum with three values
— `CREATED`, `EXISTS_SAME`, `EXISTS_DIFFERENT` — exactly as in section 17.4. Never
write a `switch` arm that groups `Exists` and `ExistsWithDifferent*` together.

### Recommended exception hierarchy

```java
package com.yourapp.ledger.exceptions;

/** Base for all ledger-related exceptions. */
public class LedgerException extends RuntimeException {
    public LedgerException(String message) { super(message); }
}

/**
 * A transfer was rejected for business reasons (insufficient funds, closed
 * account, etc.) — not a system error. Callers should present this to the
 * user, not retry blindly.
 */
public class TransferRejectedException extends LedgerException {
    private final int position;
    private final CreateTransferStatus status;

    public TransferRejectedException(int position, CreateTransferStatus status) {
        super("Transfer at position " + position + " rejected: " + status);
        this.position = position;
        this.status = status;
    }

    public int getPosition() { return position; }
    public CreateTransferStatus getStatus() { return status; }
}
```

---

## 12. Testing Strategy

### Development mode for integration tests

Start a single-replica TigerBeetle with `--development` (relaxes Direct IO and
memory-locking requirements — works in CI containers):

```bash
./tigerbeetle format --cluster=0 --replica=0 --replica-count=1 --development /tmp/test.tigerbeetle
./tigerbeetle start --addresses=3000 --development /tmp/test.tigerbeetle
```

### JUnit 5 lifecycle for integration tests

```java
package com.yourapp.ledger;

import com.tigerbeetle.Client;
import com.tigerbeetle.UInt128;
import org.junit.jupiter.api.*;

import java.io.IOException;
import java.nio.file.*;

class TransferServiceIT {

    private static Process tigerBeetleProcess;
    private static Client client;
    private static final Path DATA_FILE = Path.of("/tmp/tb-test-" + System.nanoTime() + ".tigerbeetle");

    @BeforeAll
    static void startCluster() throws IOException, InterruptedException {
        new ProcessBuilder("tigerbeetle", "format",
                "--cluster=0", "--replica=0", "--replica-count=1",
                "--development", DATA_FILE.toString())
            .inheritIO().start().waitFor();

        tigerBeetleProcess = new ProcessBuilder("tigerbeetle", "start",
                "--addresses=3001", "--development", DATA_FILE.toString())
            .inheritIO().start();

        Thread.sleep(500); // allow the replica to start listening

        client = new Client(UInt128.asBytes(0L), new String[] {"3001"});
    }

    @AfterAll
    static void stopCluster() throws IOException {
        client.close();
        tigerBeetleProcess.destroy();
        Files.deleteIfExists(DATA_FILE);
    }

    @Test
    void createAccountAndTransfer() {
        var accountService = new AccountService();
        // ... inject client manually for the test, run assertions
    }
}
```

### Unit tests for recipes — no live cluster needed for batch construction

Recipe methods that only *build* batches (without calling `client.createTransfers`)
can be tested by extracting batch-construction logic into pure functions that
return `TransferBatch` and asserting field values via the batch's getters.

---

## 13. Deployment Notes

### Development: single binary, single replica

```bash
curl -Lo tigerbeetle.zip https://linux.tigerbeetle.com && unzip tigerbeetle.zip
./tigerbeetle format --cluster=0 --replica=0 --replica-count=1 --development ./0_0.tigerbeetle
./tigerbeetle start --addresses=3000 --development ./0_0.tigerbeetle
```

### Production: systemd, NOT Docker

TigerBeetle's own docs recommend running the binary directly via systemd rather
than Docker — Docker adds `seccomp`/`IPC_LOCK` complexity for `io_uring` and memory
locking with little benefit, since TigerBeetle is already a single static binary.
See the reference systemd unit in the official docs (includes hardening directives:
`ProtectSystem=strict`, `DynamicUser=true`, `CAP_IPC_LOCK`, etc.).

### Production cluster size: 6 replicas

- 4/6 needed to elect a new primary
- 3/6 needed to remain available (primary alive) or 4/6 (primary also failed)
- Spread across 3 sites (2 replicas each) for geographic fault tolerance
- Each replica's data file on independent disk/machine — never share storage

### If you must use Docker (e.g. local dev parity)

```bash
docker run --security-opt seccomp=unconfined --cap-add IPC_LOCK \
    -p 3000:3000 -v $(pwd)/data:/data \
    ghcr.io/tigerbeetle/tigerbeetle \
    start --addresses=0.0.0.0:3000 --development /data/0_0.tigerbeetle
```

`exited with code 137` with no logs = OOM killer. Increase the container/VM memory
limit or lower `--cache-grid`.

---

## 14. Monitoring

Enable StatsD metrics (requires a local StatsD-compatible agent, e.g. Datadog Agent
or Telegraf with `datadog_extensions`):

```bash
./tigerbeetle start --addresses=3000 --experimental --statsd=127.0.0.1:8125 ./0_0.tigerbeetle
```

Key metrics to alert on:

| Metric | Alert condition |
|--------|-----------------|
| `tb.replica_status` | Non-zero (anything but `normal`) |
| `tb.replica_sync_stage` | Non-zero (state sync in progress) |
| `tb.replica_request` | High latency for `create_transfers` |
| `tb.grid_cache_hits` / `tb.grid_cache_misses` | High miss ratio → increase `--cache-grid` |

Also monitor at the OS level: disk space on the data file's volume, NTP sync,
memory (should be flat after startup), CPU (single core, should not saturate).

---

## 15. Architecture Decision Guide

| Question | Answer |
|----------|--------|
| Where do account/transfer code mappings live? | Hard-coded enums in Java + mirrored in PostgreSQL for human-readable reporting. Never fetched per-transfer. |
| Who generates Account.id / Transfer.id? | The client-facing app, using `UInt128.id()`, persisted before submission |
| Can I update a transfer after creation? | No — never. Create a correcting transfer instead |
| Should the API service connect to TigerBeetle per-request? | No — one singleton `Client`, shared and thread-safe |
| Where does user metadata (names, emails) live? | PostgreSQL only — never in TigerBeetle |
| How do I implement a wallet balance cap? | `flags.credits_must_not_exceed_debits` (or the inverse) at account creation |
| How do I do multi-currency? | Separate ledger per currency + linked transfers through liquidity accounts |
| How do I roll back a bad transfer? | A new transfer in the opposite direction — link via shared `user_data_128` |
| Docker or systemd in production? | systemd — Docker adds complexity for `io_uring`/`IPC_LOCK` with no benefit |
| How many replicas in production? | 6, spread across 3 sites (2 per site) |
| Do I need TigerBeetle for non-financial counters? | Only if the counter needs the safety/throughput guarantees (e.g. rate limiting at scale) — otherwise Redis/Postgres is simpler |
| Which system is the source of truth for "does this account exist"? | TigerBeetle (System of Record) — see section 17. A PostgreSQL row alone doesn't mean the account is real |
| In what order do I write PostgreSQL and TigerBeetle when creating an account? | PostgreSQL first (System of Reference, no commitment), TigerBeetle last (System of Record, makes it real) — "Write Last, Read First", section 17 |
| What if the process crashes between the PostgreSQL write and the TigerBeetle write? | Safe — retry reads `tb_account_id` back from PostgreSQL and completes the TigerBeetle write with the SAME id (section 17.7) |
| What if TigerBeetle has an account with no PostgreSQL row? | Traceability violation — alert immediately, this means money exists with no identifiable owner |

---

## 16. Anti-Patterns — Never Do This

1. **Creating accounts/transfers one at a time in a loop.** Always batch — up to
   8189 per request. A loop of single-event requests can be 100-1000x slower.

2. **Generating IDs with auto-increment from PostgreSQL.** Use `UInt128.id()` —
   client-generated, time-based, collision-resistant, LSM-optimized.

3. **Setting `Account.timestamp` or `Transfer.timestamp` to anything but 0 on
   create.** The cluster assigns these. Setting a non-zero value (without
   `flags.imported`) is a validation error.

4. **Fetching ledger/code metadata from PostgreSQL on the transfer hot path.**
   Cache these mappings in memory at startup — they're immutable and append-only.

5. **Treating `Exists` as an error.** It means the same `id` was already
   successfully processed — the idempotent retry succeeded. Treat it the same as
   `Created`.

6. **Reusing a pending transfer's `id` for its post/void.** The post/void transfer
   needs its own new `id`; it references the pending transfer via `pending_id`.

7. **Putting `flags.linked` on the last event in a batch.** This returns
   `LinkedEventChainOpen`. The chain must close with an unlinked event.

8. **Exposing the TigerBeetle client or its port to anything other than your own
   API service.** No authentication exists at the TigerBeetle layer — the
   application is the only security boundary.

9. **Retrying a transfer with the same `id` after a transient failure
   (`ExceedsCredits`, etc.) expecting it to succeed later.** It will return
   `IdAlreadyFailed` forever. Generate a new `id` for a new attempt.

10. **Running TigerBeetle in Docker in production "for consistency with other
    services."** Use systemd — see section 13.

11. **Writing to TigerBeetle before PostgreSQL when creating an account.** This
    violates Write Last, Read First — if the process crashes after the TigerBeetle
    write, you have money in an account with no record of its owner (a
    traceability violation). PostgreSQL first, TigerBeetle last. See section 17.

12. **Grouping `Exists` and `ExistsWithDifferent*` into the same `switch` arm for
    account creation in a dual-write flow.** `Exists` is a safe retry;
    `ExistsWithDifferent*` is a conflict that must panic and alert an operator —
    silently treating it as success can mask a bug that's already written
    inconsistent data. See section 17.4–17.5.

---

## 17. Dual-Write Consistency: PostgreSQL + TigerBeetle (Write Last, Read First)

TigerBeetle and PostgreSQL do not share a transaction boundary. Creating an
account touches both systems, and the sequential composition of two writes is
**not itself a transaction** — completion is only eventual, intermediate state is
externalized and observable, and a crash between the two writes is a real
scenario you must design for, not an edge case you can ignore.

### 17.1 Two Safety Properties

- **Consistent**: every account in PostgreSQL has a matching account in
  TigerBeetle, and vice versa.
- **Traceable**: every account in TigerBeetle with a positive balance corresponds
  to an account in PostgreSQL.

The system may be **temporarily inconsistent** — a crash between two writes can
leave a row in one system with no counterpart in the other, for a while. It must
**never become untraceable** — an account that can hold money in TigerBeetle but
has no corresponding ownership record in PostgreSQL means that money is, for
practical purposes, orphaned.

### 17.2 System of Record vs System of Reference

Pick one system as the **System of Record**: the one whose presence of a record
means "this exists, system-wide, right now." The other is the **System of
Reference**: its presence alone means nothing until the System of Record agrees.

**TigerBeetle is the System of Record for account existence:**

- An account that exists only in PostgreSQL cannot process transfers — it's a
  *staged* record, not yet real. If it's orphaned (process crashed before the
  TigerBeetle write), no harm: TigerBeetle simply rejects transfers to an account
  it has never heard of.
- An account that exists in TigerBeetle can process transfers immediately — it is
  real, system-wide, the moment it's created there, regardless of what PostgreSQL
  thinks.

This asymmetry is exactly why a PostgreSQL-only row is harmless but a
TigerBeetle-only account (no PostgreSQL counterpart) is a **traceability
violation** — TigerBeetle would accept transfers for an account nobody can
identify the owner of.

### 17.3 The Write Last, Read First Rule

> **Write** to the System of Reference first (no commitment yet). **Write** to the
> System of Record last (this is the step that makes the account real). When
> **reading** to check existence, always consult the System of Record first.

Applied to PostgreSQL + TigerBeetle:

- **Write order**: PostgreSQL first, TigerBeetle last.
- **Read order**: to answer "does this account exist and is it usable," query
  TigerBeetle first (section 17.9) — a PostgreSQL row alone doesn't mean the
  account is real.

If this ordering is respected, and the System of Record provides strict
serializability (TigerBeetle does), **the composed system as a whole preserves
strict serializability for account existence** — a genuinely good outcome for two
systems that share no transaction.

### 17.4 Three-Way Idempotent Results

Both the PostgreSQL write and the TigerBeetle write must report one of three
outcomes — not a binary success/failure:

```java
package com.yourapp.ledger;

public enum SyncResult {
    CREATED,          // newly created on this attempt
    EXISTS_SAME,      // already existed, with identical data — safe retry
    EXISTS_DIFFERENT  // already existed, but with DIFFERENT data — a conflict
}
```

`CREATED` and `EXISTS_SAME` are both **success** — the second simply means a
previous attempt's response was lost and this is a retry. `EXISTS_DIFFERENT` is
**always a bug or a conflict** and must never be treated the same as
`EXISTS_SAME`.

### 17.5 Account Creation Decision Table

| PostgreSQL Result | TigerBeetle Result | System Result | Meaning |
|--------------------|---------------------|----------------|---------|
| Created | Created | ✅ Success | Normal path |
| Created | Exists/Same | 🛑 Panic | Ordering violation — TigerBeetle had this account before PostgreSQL did |
| Created | Exists/Different | 🛑 Panic | Ordering violation + conflict |
| Exists/Same | Created | ✅ Success (Recovery) | PostgreSQL succeeded before, the TigerBeetle call was lost — now completing |
| Exists/Same | Exists/Same | ✅ Success (Recovery) | Both succeeded before — this attempt is a pure retry |
| Exists/Same | Exists/Different | 🛑 Panic | Conflict — TigerBeetle has different data than PostgreSQL expects |
| Exists/Different | (any) | 🛑 Panic | Conflict — the PostgreSQL row itself diverged; investigate before proceeding |

Every 🛑 **Panic** outcome means: **stop, do not retry automatically, alert an
operator.** These states should be unreachable under correct ordering. Reaching
one means the protocol was violated somewhere — a manual DB edit, a bug, or two
processes racing with different ids for the same logical account.

### 17.6 Java Implementation

```java
package com.yourapp.ledger;

import com.tigerbeetle.*;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import javax.sql.DataSource;
import java.sql.*;
import java.util.Objects;
import java.util.UUID;

@ApplicationScoped
public class AccountOnboardingService {

    @Inject Client client;
    @Inject DataSource dataSource; // PostgreSQL

    public record AccountData(String ownerName, String email, int ledger, int code, long flags) {}
    private record PgResult(SyncResult result, byte[] tbAccountId) {}

    /**
     * Orchestrates account creation across PostgreSQL (System of Reference) and
     * TigerBeetle (System of Record), following Write Last, Read First.
     *
     * Idempotent: safe to retry with the same uuid after a crash or timeout at
     * any point. Returns the TigerBeetle account id.
     */
    public byte[] createAccount(UUID uuid, AccountData data) throws SQLException {

        // Only used if this is the FIRST attempt for this uuid. If PostgreSQL
        // already has a row, we reuse ITS stored tb_account_id instead — see
        // pgCreateAccount — so a retry never generates a second TigerBeetle id.
        byte[] candidateId = UInt128.id();

        // ── 1. Write to the System of Reference (PostgreSQL) FIRST ──────────
        PgResult pgResult = pgCreateAccount(uuid, candidateId, data);

        if (pgResult.result() == SyncResult.EXISTS_DIFFERENT) {
            throw new LedgerException(
                "CONFLICT: account " + uuid + " exists in Postgres with different "
                + "data. Manual investigation required — do not retry automatically.");
        }

        // ── 2. Write to the System of Record (TigerBeetle) LAST ─────────────
        // Always use pgResult.tbAccountId() — NOT candidateId — so a retry
        // reuses the same TigerBeetle id even on attempt #2, #3, etc.
        SyncResult tbResult = tbCreateAccount(pgResult.tbAccountId(), data);

        if (tbResult == SyncResult.EXISTS_DIFFERENT) {
            throw new LedgerException(
                "CONFLICT: account " + uuid + " exists in TigerBeetle with "
                + "different data. Manual investigation required — do not retry automatically.");
        }

        // ── 3. Detect ordering violations ────────────────────────────────────
        // If PostgreSQL just created this row for the first time, TigerBeetle
        // must ALSO be creating it for the first time. If TigerBeetle already
        // had it, something wrote to TigerBeetle out of order.
        if (pgResult.result() == SyncResult.CREATED && tbResult != SyncResult.CREATED) {
            throw new LedgerException(
                "ORDERING VIOLATION: Postgres row for " + uuid + " was newly "
                + "created, but the TigerBeetle account already existed. This "
                + "should be impossible under Write Last, Read First — alert an operator.");
        }

        return pgResult.tbAccountId();
    }

    /**
     * Inserts the account row if it doesn't exist (ON CONFLICT DO NOTHING), then
     * always reads the row back to determine the result AND the tb_account_id
     * to use going forward — the STORED one (not candidateId) if this is a retry.
     */
    private PgResult pgCreateAccount(UUID uuid, byte[] candidateId, AccountData data) throws SQLException {
        try (Connection conn = dataSource.getConnection()) {
            try (PreparedStatement insert = conn.prepareStatement("""
                    INSERT INTO accounts (uuid, tb_account_id, owner_name, email, ledger, code, flags)
                    VALUES (?, ?, ?, ?, ?, ?, ?)
                    ON CONFLICT (uuid) DO NOTHING
                    """)) {
                insert.setObject(1, uuid);
                insert.setBytes(2, candidateId);
                insert.setString(3, data.ownerName());
                insert.setString(4, data.email());
                insert.setInt(5, data.ledger());
                insert.setInt(6, data.code());
                insert.setLong(7, data.flags());
                if (insert.executeUpdate() == 1) {
                    return new PgResult(SyncResult.CREATED, candidateId);
                }
            }

            // Row already existed — read it back to compare and recover its id
            try (PreparedStatement select = conn.prepareStatement(
                    "SELECT tb_account_id, owner_name, email, ledger, code, flags "
                    + "FROM accounts WHERE uuid = ?")) {
                select.setObject(1, uuid);
                try (ResultSet rs = select.executeQuery()) {
                    if (!rs.next()) {
                        throw new LedgerException("Row for " + uuid + " vanished after conflict");
                    }
                    byte[] storedId = rs.getBytes("tb_account_id");
                    boolean same =
                        Objects.equals(rs.getString("owner_name"), data.ownerName())
                        && Objects.equals(rs.getString("email"), data.email())
                        && rs.getInt("ledger") == data.ledger()
                        && rs.getInt("code") == data.code()
                        && rs.getLong("flags") == data.flags();

                    return new PgResult(
                        same ? SyncResult.EXISTS_SAME : SyncResult.EXISTS_DIFFERENT, storedId);
                }
            }
        }
    }

    private SyncResult tbCreateAccount(byte[] accountId, AccountData data) {
        AccountBatch accounts = new AccountBatch(1);
        accounts.add();
        accounts.setId(accountId);
        accounts.setLedger(data.ledger());
        accounts.setCode(data.code());
        accounts.setFlags(data.flags());

        CreateAccountResultBatch results = client.createAccounts(accounts);
        while (results.next()) {
            return switch (results.getStatus()) {
                case Created -> SyncResult.CREATED;
                case Exists -> SyncResult.EXISTS_SAME;
                case ExistsWithDifferentLedger, ExistsWithDifferentCode,
                     ExistsWithDifferentFlags, ExistsWithDifferentUserData128,
                     ExistsWithDifferentUserData64, ExistsWithDifferentUserData32 ->
                    SyncResult.EXISTS_DIFFERENT;
                default -> throw new LedgerException(
                    "Account creation failed for " + uuidOf(accountId) + ": " + results.getStatus());
            };
        }
        throw new LedgerException("No result returned for account " + uuidOf(accountId));
    }

    private static String uuidOf(byte[] id) {
        return java.util.HexFormat.of().formatHex(id);
    }
}
```

### 17.7 The Crash-Recovery Case, Concretely

Walk through what happens if the process crashes **between** step 1 and step 2
above — PostgreSQL committed, the TigerBeetle call never went out:

1. PostgreSQL has the row `(uuid, tb_account_id = X, ...)`.
2. TigerBeetle has nothing for `X`.
3. The caller retries `createAccount(uuid, data)` with the same `uuid`.
4. `pgCreateAccount` hits the conflict branch, reads back `tb_account_id = X`,
   compares the rest of the data → `EXISTS_SAME`, returns `(EXISTS_SAME, X)`.
5. `tbCreateAccount(X, data)` now runs for the first time → `CREATED`.
6. Decision table: `Exists/Same + Created → ✅ Success (Recovery)`.

This is why the retry **must read `tb_account_id` back from PostgreSQL** rather
than generating a fresh `UInt128.id()`. If it generated a new id `Y ≠ X`, step 5
would create a *different* TigerBeetle account, permanently orphaning
PostgreSQL's row, which still points at `X` — a consistency violation that's hard
to detect later.

### 17.8 Durable Execution Frameworks (Optional)

The "read the id back from PostgreSQL" trick in section 17.6 exists because we
don't have automatic checkpointing. Durable execution frameworks (Resonate,
Temporal, Restate, etc.) checkpoint the result of `generateId()` itself, so
retries skip straight to reusing the same id without the read-back step. If your
stack already uses one of these for orchestration, you can simplify
`createAccount` by wrapping `UInt128.id()` in a durable step — but the decision
table and panic conditions in 17.5 remain identical either way. Don't introduce a
durable execution framework *solely* for this; the manual pattern above is
sufficient for most teams.

### 17.9 Read Path — Always Check the System of Record

When the application needs to answer "does this account exist and is it usable"
(before allowing a login, before showing a balance, before accepting a deposit),
**query TigerBeetle**, not PostgreSQL. A PostgreSQL row can exist for an account
that failed at step 2 of section 17.7 and was never completed — TigerBeetle is the
only system whose presence of the account means it's real.

```java
public boolean accountIsUsable(byte[] tbAccountId) {
    IdBatch ids = new IdBatch(1);
    ids.add(tbAccountId);
    AccountBatch accounts = client.lookupAccounts(ids);
    return accounts.getLength() > 0; // present in TigerBeetle = real
}
```

### 17.10 Note on Transfers

Transfers themselves don't usually need this dual-write dance — TigerBeetle is
the *sole* system of record for transfers, full stop. The same principle applies,
though, if a transfer's `user_data_128` points at a PostgreSQL entity that doesn't
exist yet (e.g. an order row): create that PostgreSQL row first (System of
Reference), then the transfer last (System of Record). If the transfer succeeds
but the order row was never created, you'd have money moved for an order that
doesn't exist anywhere — the same traceability violation, one level up.

---

## Reference

- TigerBeetle docs: https://docs.tigerbeetle.com/
- Java client reference: https://github.com/tigerbeetle/tigerbeetle/tree/main/src/clients/java
- TigerStyle (engineering principles): https://github.com/tigerbeetle/tigerbeetle/blob/main/docs/TIGER_STYLE.md
- Financial accounting primer: https://docs.tigerbeetle.com/coding/financial-accounting/