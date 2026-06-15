---
name: quine-architecture
description: >
  Expert knowledge for designing systems on Quine, the streaming graph
  interpreter. Covers the core design methodology (Standing Query defines Graph
  Shape, which is created by Ingest Query — the "Three-Part System"), JSON-to-graph
  translation decisions, idFrom ID strategies, the "all nodes exist" philosophy,
  standing query patterns (Distinct ID vs Multiple Values, aggregation, chaining,
  recursion), ingest query design and idempotency, supernode avoidance, performance
  tuning, delivery guarantees, deployment and persistor selection, recipes, REST API
  conventions, and how Quine fits alongside an existing OLTP/OLGP stack (TigerBeetle
  + PostgreSQL) and event infrastructure (Kafka, Akka). For designing NEW
  pattern-detection/streaming-analytics systems on Quine — fraud detection,
  real-time relationship graphs, anomaly detection, enrichment pipelines.
---

You are a software architect specialized in Quine, the streaming graph interpreter.
Apply this skill as the authoritative source for every design decision. The guiding
principle: **Quine is not a database you query — it's a continuously-running pattern
detector you design around the patterns themselves**. Everything else (graph shape,
ingest queries, ID strategy) follows from the patterns, not the other way around.

---

## 1. Architecture Context — Where Quine Fits

```
┌──────────────────────────────────────────────────────────────────────┐
│  Upstream Event Sources                                                  │
│  Kafka / Kinesis / SQS / files / TigerBeetle CDC (AMQP) / webhooks       │
└────────────────────────────────┬─────────────────────────────────────┘
                                  │ (at-least-once, backpressured)
┌────────────────────────────────▼─────────────────────────────────────┐
│  Quine                                                                   │
│  - Ingest streams turn events into graph structure (idFrom + Cypher)    │
│  - The graph IS the data model AND the computational model              │
│  - Standing queries continuously watch for patterns                     │
│  - On match: emit downstream and/or update the graph                    │
└────────────────────────────────┬─────────────────────────────────────┘
                                  │ (best-effort, backpressured)
┌────────────────────────────────▼─────────────────────────────────────┐
│  Downstream Consumers                                                    │
│  Kafka topic / webhook (your Node.js API) / Slack / SNS / another        │
│  Cypher query back into the graph (chaining/recursion)                  │
└──────────────────────────────────────────────────────────────────────┘
```

### Quine is NOT a replacement for PostgreSQL or TigerBeetle

If you've built the dual-write PostgreSQL + TigerBeetle pattern (see the
`tigerbeetle-java` skill, section 17), Quine sits in a **third role**, alongside
both:

| System | Role | Quine's relationship to it |
|--------|------|------------------------------|
| TigerBeetle | System of Record for transfers/balances | Quine never writes balances. It can *consume* TigerBeetle's CDC (Change Data Capture, AMQP) stream as an ingest source to build a relationship graph of accounts/transfers for pattern detection. |
| PostgreSQL | System of Reference for master data | Quine can enrich its graph with PostgreSQL data via a `CypherQuery` standing query output, or treat PostgreSQL as a downstream consumer of standing query results. |
| Quine | Real-time pattern detection over streams | Detects relationships and anomalies — "this device sent transfers to 5 different accounts in 2 minutes" — that are expensive or impossible to express as SQL/TigerBeetle queries. |

**Never put financial balances, account state, or anything requiring strict
serializability into Quine.** Quine is eventually consistent and standing query
outputs are best-effort (section 10). It answers "what patterns exist in the
stream," not "what is the current authoritative state."

### A concrete integration: TigerBeetle CDC → Quine → Alert

```
TigerBeetle (transfers happen)
  → CDC job publishes to RabbitMQ (AMQP, see tigerbeetle-java skill §CDC)
    → Quine Kafka/AMQP-bridge ingest stream
      → ingest query builds (account)-[:SENT]->(transfer)-[:TO]->(account) graph
        → standing query detects "account sent to 5+ distinct accounts in 10 min"
          → output: webhook to your Node.js API → flag account for review
```

This is the canonical shape of a Quine deployment in this stack: Quine never
touches the ledger directly, it watches the *shape* of activity derived from it.

### Cypher, not Gremlin

Quine supports Cypher (primary, full-featured) and Gremlin (API v1 only, planned
for deprecation). **Always use Cypher.** All examples in this skill are Cypher.

### Computational model note (for Akka users)

Quine nodes are backed by actors internally, and standing query backpressure uses
Akka Streams. This is implementation detail — **you never write actor code for
Quine**. If you're coming from the `akka-master` skill, the conceptual lineage
(supervision, backpressure, lightweight units of computation) will feel familiar,
but Quine exposes none of it directly. You design in Cypher and YAML/JSON, not Akka.

---

## 2. The Three-Part System — Core Design Methodology

This is the single most important section in this skill. **Every Quine design
session starts here, in this order, never reversed:**

```
1. Standing Query  →  defines  →  2. Graph Shape  →  is created by  →  3. Ingest Query
   (the pattern to detect)         (nodes, edges,                       (transforms records
                                     properties needed)                   into that structure)
```

### Why this order, specifically

- Standing queries evaluate incrementally — each node only checks the patterns it
  participates in. This makes them efficient, BUT:
- **The standing query cannot match structure that doesn't exist.** If the pattern
  is `(user)-[:PURCHASED]->(order)`, the graph MUST contain `:PURCHASED` edges
  between `:user` and `:order` nodes — or the standing query matches nothing,
  silently, forever.
- The ingest query is therefore **derived from** the graph shape, which is
  **derived from** the standing query. Changing any one of the three almost always
  requires changing the other two.

### The Design Session Template

When designing a new Quine system, walk through these steps in order — never skip
ahead to writing ingest queries first:

**Step 1 — Define the question.** What pattern or relationship matters? Express it
in one sentence: *"Detect when [X] happens across [Y] within [Z]."*

**Step 2 — Express it as a graph pattern.** What nodes, what labels, what edges,
what property conditions?

```cypher
// Template
MATCH (a:LabelA)-[:RELATIONSHIP]->(b:LabelB {property: value})
RETURN DISTINCT id(a)
```

**Step 3 — Choose Distinct ID vs Multiple Values** (section 6) based on whether you
need just "this pattern exists" or details about each match.

**Step 4 — Design the graph shape that satisfies step 2.** For each node in the
pattern: what's its ID strategy (section 4)? What labels and properties does it
need? For each edge: what creates it?

**Step 5 — Design the ingest query** that creates exactly that shape from incoming
records (section 7).

**Step 6 — Decide the output**: emit downstream, update the graph, or both
(section 6).

If at any point step 4 or 5 feels awkward or requires data you don't have in the
ingest record, **go back to step 2** — the pattern is probably not expressible with
the data you actually receive, and that's a data/pipeline problem to solve before
writing more Cypher.

---

## 3. JSON to Graph Translation

Most data enters Quine as JSON from Kafka/Kinesis/files. The mapping decision is
the foundation of the graph shape (step 4 above).

| JSON Element | Graph Element | Decision Criteria |
|---|---|---|
| Objects with identity | Node | Will you reference this object from multiple places? Does it have a natural key? |
| Scalar values | Property | Is this an attribute of a node rather than a node itself? |
| Relationships | Edge | Do you need to traverse from one node to another in a standing query? |
| Enumerable/categorical values | Property, NOT Edge | High-cardinality categorical data (status, type, region) as edges creates supernodes (section 8) |
| Natural keys | Node ID (via `idFrom`) | What combination of fields uniquely identifies this node? |

### Worked decision: a payment event

```json
{
  "transferId": "TXN-789",
  "sourceAccountId": "ACC-111",
  "destAccountId": "ACC-222",
  "amount": 5000,
  "currency": "USD",
  "deviceId": "DEV-abc",
  "ipAddress": "203.0.113.7",
  "timestamp": "2026-06-14T10:30:00Z"
}
```

| Field | Mapping | Why |
|---|---|---|
| `transferId` | `:Transfer` node, ID = `idFrom("transfer", transferId)` | Has identity, referenced by edges |
| `sourceAccountId` / `destAccountId` | `:Account` nodes, ID = `idFrom("account", id)` | Referenced across many transfers — natural traversal points |
| `amount`, `currency`, `timestamp` | Properties on `:Transfer` | Attributes, not entities — no standing query traverses "into" an amount |
| `deviceId` | `:Device` node, ID = `idFrom("device", deviceId)` | Need to traverse "which accounts did this device send to" — that's an edge-worthy relationship |
| `ipAddress` | Property on the edge or on `:Device`, NOT a node — unless you need "find all devices from this IP" | High cardinality, usually filtered not traversed |

The edge decisions follow from the pattern you're building toward (section 14 has
the full worked example).

---

## 4. ID Strategy — `idFrom` and ID Providers

### `idFrom` is the only "index" you get

```cypher
// idFrom deterministically derives a node ID from any number of arguments
MATCH (n) WHERE id(n) = idFrom("account", $that.sourceAccountId)
```

Every ingest query and every ad-hoc query (NOT standing query patterns — see
section 9) should anchor at least one `MATCH` clause with `id(n) = idFrom(...)`.
Without this, Quine performs a full node scan — fine for a 20-node dev graph,
catastrophic for a production stream.

### Prefix by type — always

```cypher
// GOOD — "account" and "device" namespaces never collide even if the
// underlying IDs happen to be the same string
id(account) = idFrom("account", $that.sourceAccountId)
id(device)  = idFrom("device", $that.deviceId)

// RISKY — if an account ID and a device ID could ever be the same string,
// they'd resolve to the SAME node
id(account) = idFrom($that.sourceAccountId)
id(device)  = idFrom($that.deviceId)
```

### Natural keys and composite keys

```cypher
// Single natural key
id(user) = idFrom("user", $that.email)

// Composite key — e.g. a sensor reading identified by sensor + time
id(reading) = idFrom("sensor-reading", $that.sensorId, $that.timestamp)
```

### Consistency is non-negotiable

If `ingest-A` uses `idFrom("account", accountId)` and `ingest-B` uses
`idFrom("acct", accountId)`, you silently get **two different nodes for the same
logical account**, and no standing query pattern spanning both streams will ever
match. There is no error, no warning — just missing matches.

**Document the ID strategy for every node type once, in one place** (e.g. a
`docs/quine-id-strategies.md` in the repo), and have every ingest query reference
that document. Treat it with the same rigor as the `LedgerCodes`/`AccountCodes`
constants in the `tigerbeetle-java` skill — both are append-only, shared
conventions that every producer must agree on.

### ID Provider selection

| Provider | Use when |
|---|---|
| `uuid` (default) | General purpose — accepts any 128-bit value via `idFrom`, good default |
| `uuid-4` | You need strictly RFC-compliant v4 UUIDs for interop with external systems |
| `long` | Small dev/test graphs only — collision risk is too high for production (`[-(2^53-1), 2^53-1]`) |
| `byte-array` | Custom ID schemes, maximum flexibility, rarely needed |

For this stack, default to `uuid` unless an external system you're integrating
with (e.g. returning IDs to your Node.js API) requires a specific format.

---

## 5. The "All Nodes Exist" Philosophy

A foundational difference from every traditional database:

> Every node that could exist already exists, with no data. Quine never "creates"
> a node — you address one by ID and it accumulates data as events arrive.

```cypher
// This does NOT "create" an account node. It selects the account node by ID
// (which already conceptually exists) and sets properties on it.
MATCH (account)
WHERE id(account) = idFrom("account", $that.accountId)
SET account.balance_hint = $that.balance,
    account:Account
```

### Why this matters architecturally

- **No existence checks, ever.** Don't write Cypher that does `OPTIONAL MATCH ...
  WHERE NOT EXISTS ...` to check if an account node exists before "creating" it.
  Just `MATCH` and `SET`.
- **Multiple streams can reference the same node with zero coordination.** An
  ingest from your TigerBeetle CDC stream and an ingest from your Node.js
  "user profile updated" Kafka topic can both target the same `:Account` node via
  `idFrom("account", accountId)` — order of arrival doesn't matter, both
  contribute properties.
- **Counting is expensive, existence is free.** `MATCH (n) RETURN count(n)` scans
  every node Quine has ever "touched" (including empty ones with history). Don't
  use node counts as a health check the way you might `SELECT count(*)` in
  Postgres.
- **Deletion is "emptying," not removal.** Deleting a node records an event that
  removes all its properties/edges — the node still exists (with history) and
  still counts in full scans.

---

## 6. Standing Query Design

### Two modes

| Mode | Returns | Use when |
|---|---|---|
| **Distinct ID** (default) | Once per unique matching node | Only the existence of a pattern matters — efficient, lower memory |
| **Multiple Values** | Multiple results per match, including property values, supports `WHERE` on computed values | You need details about each match, or need to filter on aggregated/computed properties |

```cypher
// Distinct ID — "does this device have ANY transfer to a flagged account?"
MATCH (device:Device)-[:SENT]->(t:Transfer)-[:TO]->(account:Account {flagged: true})
RETURN DISTINCT id(device)

// Multiple Values — needed for WHERE on a computed property
MATCH (account:Account)
WHERE account.distinctDestinationCount > 5
RETURN id(account) as accountId, account.distinctDestinationCount as count
```

**Prefer Distinct ID.** Use Multiple Values only when you need per-match details
or `WHERE` on values that Distinct ID's restrictions prevent expressing.

### The Output Decision: Emit vs Update

| Output type | Use when |
|---|---|
| **Emit downstream** (Kafka, webhook, Slack, SNS, file) | An external system needs to act on the result — your Node.js API, an alerting channel |
| **Update the graph** (`CypherQuery` output that writes back) | The result feeds a *later* standing query, or you're maintaining a computed value (counter, flag) |

Both can be combined by chaining outputs.

### Pattern 1 — Direct Pattern Detection

```cypher
// Pattern
MATCH (device:Device)-[:SENT]->(t:Transfer)-[:TO]->(account:Account)
WHERE account.flagged = true
RETURN DISTINCT id(t) as transferId
```

```cypher
// Output: enrich with full transfer + account details before emitting
MATCH (t)-[:TO]->(account)
WHERE id(t) = $that.data.transferId
RETURN properties(t), account.id AS flaggedAccount
```

### Pattern 2 — Aggregation via Graph Update

```cypher
// Standing query pattern: every time a device sends a transfer to a NEW
// distinct account, this matches
MATCH (device:Device)-[:SENT]->(t:Transfer)-[:TO]->(account:Account)
RETURN DISTINCT id(device) as deviceId, id(account) as accountId
```

```cypher
// Output: atomically increment a counter on the device — int.add is atomic
// even under concurrent matches
MATCH (device) WHERE id(device) = $that.data.deviceId
CALL int.add(device, "distinctAccountCount", 1) YIELD result
RETURN result
```

> ⚠️ The pattern above as written would re-increment for every transfer to the
> SAME account too. To count only DISTINCT accounts, the graph shape needs a
> `:CONTACTED` edge created once per (device, account) pair during ingest —
> design the edge so that re-ingesting the same pair doesn't re-create a parallel
> edge (Cypher `MERGE`-like idempotency via `CREATE` is NOT automatically
> deduplicated; use a property check or a single edge per pair enforced by the
> ingest query's `idFrom`-based structure).

### Pattern 3 — Chained (Staged) Standing Queries

Break complex analysis into stages — each stage's output sets a property/label
that the next stage's pattern matches on.

```cypher
// Stage 1 (Multiple Values, to allow WHERE on computed property):
// detect when the counter crosses a threshold
MATCH (device:Device)-[:SENT]->(t:Transfer)
WHERE device.distinctAccountCount > 5
RETURN id(device) as deviceId
```

```cypher
// Stage 1 output: mark the device
MATCH (device) WHERE id(device) = $that.data.deviceId
SET device.highRisk = true
```

```cypher
// Stage 2 (Distinct ID): match on the flag set by stage 1
MATCH (device:Device {highRisk: true})-[:SENT]->(t:Transfer)
RETURN DISTINCT id(t)
```

This staged approach keeps each query simple, lets intermediate results
(`highRisk`) be reused by multiple downstream queries, and models the workflow as
a pipeline.

### Pattern 4 — Recursive Patterns

A standing query's output can modify the graph in a way that re-triggers the same
standing query — enabling propagation/traversal algorithms.

```cypher
// Pattern: find untainted nodes connected to a tainted node
MATCH (source {tainted: true})-[:SENT_TO]->(recipient)
WHERE recipient.tainted IS NULL
RETURN DISTINCT id(recipient) as recipientId
```

```cypher
// Output: propagate the taint — recipient now matches as "source" too
MATCH (n) WHERE id(n) = $that.data.recipientId
SET n.tainted = true
```

⚠️ **Recursive patterns require a termination condition** (here: `tainted IS NULL`
ensures already-tainted nodes don't re-match). Without one, this is an infinite
loop that will consume the standing query result queue and trigger backpressure
(section 9) or queue overflow (section 10).

---

## 7. Ingest Query Design

### The Standard Template

```cypher
// 1. Receive the incoming record as $that
WITH $that AS data

// 2. Select all nodes by ID — they "exist" already (section 5)
MATCH (node1), (node2), (node3)
WHERE id(node1) = idFrom("type1", data.field1)
  AND id(node2) = idFrom("type2", data.field2)
  AND id(node3) = idFrom("type3", data.field3)

// 3. Set properties and labels
SET node1.property = data.value,
    node1:Label1
SET node2 = data.nestedObject,   // copies all fields from a nested object
    node2:Label2

// 4. Create edges
CREATE (node1)-[:RELATIONSHIP]->(node2),
       (node2)-[:ANOTHER_REL]->(node3)
```

### Key Principles

1. **Always anchor by ID.** Every node in the `MATCH` needs
   `id(n) = idFrom(...)`.
2. **Each record creates ALL the structure it references.** Don't assume another
   record will create a missing node or edge — every record is independent and
   may arrive in any order relative to others.
3. **Use labels for organization.** `:Account`, `:Transfer`, `:Device` make
   queries clearer and enable label-based standing query patterns.
4. **Handle optional fields** with `coalesce`:
   ```cypher
   SET node.displayName = coalesce(data.nickname, data.name, "unknown")
   ```

### Idempotency — required, not optional

Ingest is **at-least-once** (section 10): the same record WILL be reprocessed
after a crash/restart. An ingest query must produce the same graph state whether
run once or ten times for the same record.

- `SET node.field = data.value` — idempotent (last write wins, same value either way)
- `CALL int.add(node, "counter", 1)` — **NOT idempotent** if called once per
  record without a dedup mechanism; a replayed record double-counts. Either:
  - design the counter to be derived from edges/structure that itself is
    idempotent (e.g. count distinct edges via a standing query, don't increment
    a counter directly from the ingest query), or
  - accept approximate counts if the use case tolerates it (most fraud-signal
    thresholds do — a count of 6 vs 7 rarely changes the decision)
- `CREATE (a)-[:REL]->(b)` — creates a NEW edge every time, even if an identical
  edge already exists. Repeated ingest of the same record creates duplicate
  parallel edges. If a standing query counts edges (`size((a)-[:REL]->())`),
  replays inflate the count. Mitigate by encoding uniqueness into the *target*
  node's ID (e.g. `idFrom("contact", deviceId, accountId)` as a node representing
  the relationship itself, set once) rather than relying on edge cardinality.

### Multiple Streams, One Node

```cypher
// Stream 1: account profile updates (from Postgres CDC or your Node.js API)
WITH $that AS profileData
MATCH (account)
WHERE id(account) = idFrom("account", profileData.account_id)
SET account.ownerName = profileData.owner_name,
    account.region = profileData.region

// Stream 2: TigerBeetle CDC — transfer events
WITH $that AS transferData
MATCH (account)
WHERE id(account) = idFrom("account", transferData.debit_account_id)
SET account.lastTransferAt = transferData.timestamp
```

Both streams reference the same `:Account` node via the SAME `idFrom` arguments.
**Each stream owns its own properties** — avoid two streams writing the same
property, which causes overwrites in arrival-order (not necessarily intended order).

---

## 8. Supernodes

A **supernode** is a node with an excessive number of edges (e.g. "the USD
liquidity account" connected to every USD transfer — directly mirroring a
TigerBeetle control-account pattern from the `tigerbeetle-java` skill).

### When supernodes are a problem

Only when a query **traverses FROM** the supernode to its neighbors:

```cypher
// PROBLEM — traverses outward from a potential supernode
MATCH (currency:Currency)<-[:DENOMINATED_IN]-(transfer:Transfer)
RETURN transfer
```

### When supernodes are safe

When the pattern **terminates ON** the supernode without traversing its edges:

```cypher
// SAFE — terminates on the supernode, doesn't enumerate its edges
MATCH (transfer:Transfer)-[:DENOMINATED_IN]->(currency:Currency)
RETURN DISTINCT id(currency)
```

### Solutions

1. **Use a property instead of an edge** — `transfer.currency = "USD"` instead of
   an edge to a `:Currency` node, if no standing query needs to traverse
   currency → transfers.
2. **Partition the supernode** — e.g. `currency-2026-06` (by month) instead of one
   `:Currency {code: "USD"}` node accumulating millions of edges.
3. **Reconsider whether the relationship is needed** for any standing query at all.

---

## 9. Performance Considerations

### All-Node Scans — the #1 production incident

```cypher
// BAD — scans every node looking for a property match
MATCH (account) WHERE account.email = $that.email

// GOOD — direct ID lookup
MATCH (account) WHERE id(account) = idFrom("account", $that.email)
```

The warning **"Cypher query may contain full node scan"** means this. This applies
to **ingest queries and ad-hoc queries**. Standing query *patterns* are exempt —
they're designed to efficiently watch all nodes incrementally; `idFrom` is not
needed in a standing query's `MATCH`.

### Ingest Parallelism

If running multiple ingest streams on one host, **balance total parallelism**. If
the optimal parallelism for a single ingest is 120, and you run 4 ingests on the
same host, target ~30 each — not 120 each.

### Compute Early, in the Ingest

Perform computations during ingest, not in standing query outputs. Ingest queries
should prepare data so standing queries only monitor for patterns. If computation
must happen after a match, prefer chained standing queries (section 6, pattern 3)
over a complex output query.

### Monitor Backpressure

`shared.valve.ingest.{ingest-name}` — increasing value means increasing
backpressure (ingest is being slowed because downstream — graph writes, standing
query processing — can't keep up). This is **Quine working as designed**, not a
bug — but sustained high backpressure means you need more resources or a simpler
graph shape, not a code fix.

### Split Complex Queries

A single complex standing query can often be decomposed into simpler chained
queries (section 6, pattern 3) — improves both maintainability and, often,
performance.

---

## 10. Delivery Guarantees and Idempotency

| Component | Guarantee |
|---|---|
| Ingest | **At-Least-Once** |
| Standing Query Outputs | **Best Effort** |

### Ingest: At-Least-Once

```
Source → Deserialize → Write to Graph → Commit Offset
```

If Quine crashes after writing but before committing the offset, the record is
reprocessed on restart. Requires a source with offset tracking (Kafka, SQS,
Kinesis+KCL) configured to commit only after successful graph write.

**Implication**: every ingest query must be idempotent (section 7).

### Standing Query Outputs: Best Effort

Quine backpressures ingest when the standing query result queue grows — but the
queue has a **maximum size**. If it fills (e.g. a recursive pattern cascading,
section 6 pattern 4), new results are **dropped**, not buffered indefinitely.
Results can also be lost on output failure (the standing query is cancelled) or
process restart (in-flight queue contents).

### Designing Downstream Consumers

- **Treat standing query outputs as notifications, not authoritative records.**
  If your Node.js API receives a webhook "account X looks suspicious," that's a
  *signal to investigate*, not a fact to act on irreversibly without
  confirmation.
- **Design for duplicate delivery** — the same result may be emitted more than
  once in edge cases (restarts, retries at the output destination).
- **If guaranteed delivery is required**, route Quine's output through a message
  queue with its own delivery guarantees (e.g. Kafka with consumer
  acknowledgment) rather than relying on Quine's webhook/Slack/SNS outputs
  directly for anything that must not be lost.

This mirrors the idempotency discipline in `tigerbeetle-java` — both systems push
the idempotency burden to the consumer, by design, in exchange for throughput.

---

## 11. Deployment and Resource Planning

### Installation methods

| Method | Use for |
|---|---|
| Docker (`docker run -p 8080:8080 thatdot/quine`) | Quickest evaluation, consistent with the rest of this stack's Docker Compose deployments |
| Executable jar (`java -jar quine-X.X.X.jar`) | Most flexible — custom config files, recipes, benchmarking different JVM settings |
| Build from source | Contributing, needing unreleased fixes |
| Cloud-hosted (thatDot) | Enterprise features, managed ops |

Requires Java 11+.

### Persistor selection

| Persistor | Use for |
|---|---|
| **RocksDB** (default) | Single-instance local development and small production deployments — data on the same machine |
| **Cassandra / ScyllaDB** | Production — high availability, data redundancy, horizontal scalability, TTL-based expiry of old data |
| **MapDB** | In-process, including in-memory-only — testing, ephemeral pipelines |

For a production deployment alongside this stack's existing PostgreSQL/MinIO/VPS
setup: **start with RocksDB on the same VPS for low-volume streams**; move to
Cassandra only when volume or HA requirements justify operating a Cassandra
cluster — don't add Cassandra "by default."

### Resource planning

```bash
# Increase JVM heap (RAM available for node caching)
java -Xmx6g -jar quine-2.0.2.jar
```

- More RAM → more nodes cached → fewer disk reads. There's diminishing returns
  ("semantic caching") — only nodes actually needed for queries benefit.
- Tune `in-memory-soft-node-limit` / `in-memory-hard-node-limit` alongside `-Xmx`
  — these can be adjusted live via the `shard-sizes` REST endpoint.
- **High CPU is desirable** — Quine is designed to use all allotted CPU.
- **Low CPU + high backpressure** usually means the persistor (disk/Cassandra) is
  the bottleneck, not CPU.
- **OOM errors** → lower `in-memory-soft-node-limit` (adjustable live without
  restart).

---

## 12. Monitoring, Operations, and REST API

### Dashboard (`/`)

The landing page shows: System Overview (data flow Sankey from ingest → Cypher →
persistor with live latency/ops), Ingests (per-stream rate and status), Standing
Queries (match rate per query → output destinations), and Host Metrics (JVM
memory, heap, shard node counts).

### REST API conventions

- Base path: `/api/v2/`. Graph-scoped resources under `/graph/quine/`. System
  endpoints under `/system/`.
- RPC-style actions use colon syntax: `POST /ingests/{name}:pause`,
  `POST /system:shutdown`.
- Responses return the resource directly; list endpoints wrap in `{"items": [...]}`.
- Errors follow `{"error": {"code", "status", "message", "details"}}` —
  **switch on `status`** (e.g. `NOT_FOUND`, `INVALID_ARGUMENT`), not on `message`.
- `?atTime=<RFC3339>` for historical queries; `?timeout=<duration>` (Go-style,
  e.g. `20s`).

### Health checks

```bash
curl http://localhost:8080/api/v2/system/readiness
curl http://localhost:8080/api/v2/system/liveness
curl http://localhost:8080/api/v2/system/metrics
```

Use these for container orchestration health checks (consistent with the
`docker-compose` healthcheck patterns used elsewhere in this stack).

### JMX

Available for live monitoring/control via standard JVM tooling (VisualVM, etc.) —
useful for the same operational habits as any other JVM service in this stack
(Quarkus, TigerBeetle's Java client host).

---

## 13. Recipes

A recipe is a YAML file bundling ingest streams, standing queries, and UI config —
for rapid iteration, not production.

```yaml
version: 1
title: <name>
contributor: <url>
summary: <one line>
description: |-
  <long description>
ingestStreams: []
standingQueries: []
nodeAppearances: []
quickQueries: []
sampleQueries: []
statusQuery: null
```

### Recipes vs API calls

| | Recipes | API calls |
|---|---|---|
| Naming | Auto-named `INGEST-#` / `STANDING-#` | You choose names |
| Persistence | Temporary store in `/tmp`, replaced on each run | Persistent, survives restarts |
| Use for | Local iteration, sharing reproducible setups | Production |

**Develop with recipes, deploy with API calls** (or recipes + `--force-config` to
opt into persistent storage — but named, persistent API-driven config is clearer
for production review and rollback).

---

## 14. Worked Example — Fraud Pattern Detection on Transfer Events

### Step 1: Define the goal

*"Detect when a single device sends transfers to 5 or more distinct destination
accounts within a rolling window — flag the device for review."*

### Step 2: Express as a graph pattern

```cypher
MATCH (device:Device)-[:SENT]->(t:Transfer)-[:TO]->(account:Account)
RETURN id(device) as deviceId, id(account) as accountId
```

We need: `:Device` nodes, `:Transfer` nodes, `:Account` nodes, `:SENT` and `:TO`
edges. The "5+ distinct accounts" condition needs an aggregation — see step 4.

### Step 3: Choose the mode

Multiple Values for the threshold check (needs `WHERE` on a computed property,
section 6).

### Step 4: Design the graph structure

| Entity | Node | ID Strategy | Key Properties |
|---|---|---|---|
| Transfer | `:Transfer` | `idFrom("transfer", transferId)` | amount, currency, timestamp |
| Device | `:Device` | `idFrom("device", deviceId)` | distinctAccountCount (computed) |
| Account | `:Account` | `idFrom("account", accountId)` | flagged |

Edges: `(device)-[:SENT]->(transfer)`, `(transfer)-[:TO]->(account)`.

To count *distinct* accounts per device without double-counting on transfer
replays (idempotency, section 7), introduce a relationship node:

| Entity | Node | ID Strategy |
|---|---|---|
| Device-Account Contact | `:Contact` | `idFrom("contact", deviceId, accountId)` |

Edge: `(device)-[:CONTACTED]->(contact:Contact)-[:WITH]->(account)`. Re-ingesting
the same `(deviceId, accountId)` pair always resolves to the SAME `:Contact` node
— `SET contact:Contact` is idempotent even on replay, unlike `CREATE` of a direct
edge.

### Step 5: Write the ingest query

```cypher
WITH $that AS data

MATCH (device), (transfer), (account), (contact)
WHERE id(device) = idFrom("device", data.deviceId)
  AND id(transfer) = idFrom("transfer", data.transferId)
  AND id(account) = idFrom("account", data.destAccountId)
  AND id(contact) = idFrom("contact", data.deviceId, data.destAccountId)

SET transfer = data,
    transfer:Transfer

SET device:Device

SET account:Account

SET contact:Contact

CREATE (device)-[:SENT]->(transfer),
        (transfer)-[:TO]->(account),
        (device)-[:CONTACTED]->(contact),
        (contact)-[:WITH]->(account)
```

(`CREATE` of `(device)-[:CONTACTED]->(contact)` on every replay still creates
duplicate edges to the SAME `:Contact` node — that's fine, because step 6 counts
distinct `:Contact` nodes via `size((device)-[:CONTACTED]->(:Contact))`
deduplicated by target identity in the pattern below, not by edge count.)

### Step 6: Write the standing queries

```cypher
// Stage 1 (Multiple Values): recompute the distinct contact count and
// check the threshold
MATCH (device:Device)-[:CONTACTED]->(contact:Contact)
WITH device, count(DISTINCT contact) as distinctCount
WHERE distinctCount >= 5
RETURN id(device) as deviceId, distinctCount
```

```cypher
// Stage 1 output: flag the device
MATCH (device) WHERE id(device) = $that.data.deviceId
SET device.highRisk = true,
    device.distinctAccountCount = $that.data.distinctCount
```

```cypher
// Stage 2 (Distinct ID): emit an alert for every NEW transfer from a
// flagged device
MATCH (device:Device {highRisk: true})-[:SENT]->(t:Transfer)
RETURN DISTINCT id(t) as transferId
```

```cypher
// Stage 2 output: enrich and emit to webhook (your Node.js API)
MATCH (t)<-[:SENT]-(device)
WHERE id(t) = $that.data.transferId
RETURN properties(t), id(device) as deviceId, device.distinctAccountCount as riskScore
```

Stage 2's output destination is an `HttpEndpoint` pointing at your Node.js API,
which receives a best-effort notification (section 10) and decides what to do —
e.g. write a "review flag" row to PostgreSQL (NOT to TigerBeetle — this is
metadata, not a ledger entry).

---

## 15. Architecture Decision Guide

| Question | Answer |
|---|---|
| Where do I start designing? | The standing query pattern (section 2) — never the ingest query |
| Should Quine hold account balances? | No — TigerBeetle is the system of record for that. Quine holds a derived relationship graph. |
| How do I avoid full node scans? | Anchor every ingest/ad-hoc `MATCH` with `id(n) = idFrom(...)` |
| Enumerable field (status, region, currency) — node or property? | Property, unless a standing query must traverse FROM it to many neighbors |
| Distinct ID or Multiple Values? | Distinct ID by default; Multiple Values only for per-match details or `WHERE` on computed values |
| How do I avoid double-counting on ingest replay? | Model the "fact of a relationship" as its own node with a composite `idFrom`, not as a counter incremented per-record |
| Emit or update the graph? | Emit when an external system must act; update when a later standing query needs the result |
| RocksDB or Cassandra? | RocksDB for single-VPS/dev; Cassandra only when HA/scale require it |
| Recipe or API? | Recipe for development iteration; API calls for anything persistent/production |
| What if downstream needs guaranteed delivery? | Route the standing query output through Kafka (or similar), not directly through webhook/Slack |
| How does this connect to TigerBeetle? | Via TigerBeetle's AMQP CDC stream as a Quine ingest source — never direct writes in either direction |

---

## 16. Anti-Patterns — Never Do This

1. **Designing the ingest query before the standing query pattern.** The pattern
   determines the graph shape, which determines the ingest query. Reversing this
   produces a graph that can't express the pattern you actually need.

2. **`MATCH (n) WHERE n.someProperty = value` in an ingest or ad-hoc query without
   `id(n) = idFrom(...)`.** This is a full node scan — the "Cypher query may
   contain full node scan" warning is not optional to ignore.

3. **Creating an edge to a node representing a high-cardinality categorical value**
   (status, currency, region) when no standing query traverses FROM that node.
   Use a property — this is the #1 cause of supernodes.

4. **Inconsistent `idFrom` arguments across ingest streams for the same logical
   entity** (`idFrom("account", id)` vs `idFrom("acct", id)`). Creates silent
   duplicate nodes with no error — document ID strategies centrally.

5. **Checking if a node "exists" before referencing it.** All nodes exist by
   convention (section 5) — just `MATCH` and `SET`.

6. **Non-idempotent ingest queries** — especially `CALL int.add(...)` called once
   per ingested record without considering at-least-once replay (section 7, 10).

7. **Using `MATCH (n) RETURN count(n)` as a health check.** Counting scans the
   entire history of every node Quine has ever touched, including empty
   ("deleted") ones. Use the dashboard or `recentNodes()` instead.

8. **Treating standing query output delivery as guaranteed.** It's best-effort —
   if a downstream system must not miss an event, route through a queue with its
   own guarantees, don't rely on the webhook/Slack/SNS output directly.

9. **Recursive standing query patterns without a termination condition** (e.g.
   missing the `WHERE x IS NULL` guard in propagation patterns). This causes an
   infinite match loop, queue overflow, and dropped results.

10. **Putting financial balances, account state, or anything needing strict
    serializability into Quine's graph.** Quine is eventually consistent by
    design. That data belongs in TigerBeetle (system of record) — Quine holds a
    *derived* relationship graph for pattern detection only.

11. **Adding Cassandra "for production" before volume/HA actually require it.**
    RocksDB on the same VPS is the right default for low-to-moderate volume —
    don't operate a second distributed database without a reason.

12. **Writing the same property from two different ingest streams.** Causes
    overwrites in arrival order, which is not necessarily the order you intended.
    Each stream should own distinct properties on a shared node.

---

## Reference

- Quine docs: https://quine.io/
- Cypher (openCypher spec): https://s3.amazonaws.com/artifacts.opencypher.org/openCypher9.pdf
- Data Modeling and Query Design: https://quine.io/core-concepts/data-modeling-and-query-design/
- REST API reference: served at `/docs` on a running Quine instance
