# /tb:scaffold — Scaffold a New TigerBeetle Java Project

Generate the complete starting structure for a new Java project that uses
TigerBeetle as its OLTP ledger: Maven dependency, client configuration, package
structure, and a minimal AccountService/TransferService ready to extend.
Apply knowledge from `.claude/skills/tigerbeetle-java/SKILL.md`.

## Instructions

1. Read `.claude/skills/tigerbeetle-java/SKILL.md` sections 1–4, 6–7, and 17.
2. If the framework isn't specified, ask: plain Maven, Quarkus (CDI), or Spring Boot.
3. Ask whether the project needs to create accounts in PostgreSQL alongside
   TigerBeetle (true for almost any user-facing app). If yes, generate
   `AccountOnboardingService` (item below) instead of a bare `AccountService`
   that only touches TigerBeetle.
4. Generate all files listed below for the chosen framework.
5. Always make the `Client` a singleton with `destroyMethod`/`@Observes ShutdownEvent`
   for proper cleanup.
6. Include `application.properties`/`application.yml` with placeholder addresses.
7. Include a brief README snippet explaining how to run a local TigerBeetle
   instance for development (`--development` mode, single replica).

## Generated Files

### pom.xml dependency
The `tigerbeetle-java` Maven coordinate with version comment pointing to
maven-metadata.xml for the latest.

### Configuration class
- Quarkus: `TigerBeetleConfig.java` with `@Produces @Singleton` and
  `@Observes ShutdownEvent` for cleanup
- Spring Boot: `TigerBeetleConfig.java` with `@Bean(destroyMethod = "close")`
- Plain Maven: a simple `TigerBeetleClientFactory` with explicit lifecycle methods

### Code/ID constants skeleton
- `LedgerCodes.java` — empty constants class with comments explaining the
  append-only convention
- `AccountCodes.java` — same
- `TransferCodes.java` — same

### Service skeletons
- `AccountService.java` — `createAccount`, `lookupAccounts` with idempotency
  handling. Generate this ONLY if the project does not need PostgreSQL-backed
  accounts (rare).
- `AccountOnboardingService.java` — the default for user-facing accounts: the
  full dual-write pattern from skill §17.6 (`createAccount` /
  `pgCreateAccount` / `tbCreateAccount`), adapted with placeholder table/column
  names and an `AccountData` record the user can extend. Includes the
  `SyncResult` enum (`CREATED` / `EXISTS_SAME` / `EXISTS_DIFFERENT`).
- `accounts` table migration (SQL) — minimal schema matching the
  `AccountOnboardingService` queries: `uuid`, `tb_account_id`, plus whatever
  columns `AccountData` needs.
- `TransferService.java` — `createTransfers` (batched, chunked at 8189) with
  idempotency handling

### Exception classes
- `LedgerException.java`
- `TransferRejectedException.java`

### Local development setup (README snippet)

```bash
# Download and run a single-replica development cluster
curl -Lo tigerbeetle.zip https://linux.tigerbeetle.com && unzip tigerbeetle.zip
./tigerbeetle format --cluster=0 --replica=0 --replica-count=1 --development ./0_0.tigerbeetle
./tigerbeetle start --addresses=3000 --development ./0_0.tigerbeetle
```

```properties
# application.properties
tigerbeetle.cluster-id=0
tigerbeetle.addresses=127.0.0.1:3000
```

## Arguments

Usage: `/tb:scaffold [maven|quarkus|spring]`

Examples:
- `/tb:scaffold` → asks which framework, then generates everything
- `/tb:scaffold quarkus` → full scaffold with CDI producer
- `/tb:scaffold spring` → full scaffold with Spring `@Configuration`
- `/tb:scaffold maven` → plain Java, explicit client factory