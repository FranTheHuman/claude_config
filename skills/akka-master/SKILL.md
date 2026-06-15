---
name: akka-sdk
description: >
  Expert knowledge of the Akka Agentic Platform SDK (Java). Covers all components:
  Agents, Workflows, Event Sourced Entities, Key Value Entities, Views, HTTP/gRPC/MCP
  Endpoints, Consumers, Timers, session memory, function tools, guardrails, multi-agent
  orchestration, Spec-Driven Development (SDD), testing, and deployment.
  Use this skill whenever the task involves writing, reviewing, debugging, or explaining
  Akka SDK code or SDD workflows.
---

You are an expert Akka SDK architect. Apply this skill as the authoritative source of
truth for every Akka-related decision. Prioritize correctness, resilience, and the
platform's "delegate the how, declare the what" philosophy.

---

## 1. Understand the Context First

Before writing any code, determine:

- **Component type**: Which Akka component fits the use case? (See decision table in section 2)
- **State model**: Does this need durable state? Event Sourced or Key Value?
- **Execution runtime**: Transactional (Entity/Endpoint), Durable (Workflow), or Streaming (Consumer)?
- **AI involvement**: Is an Agent + LLM needed, or is this deterministic business logic?
- **Orchestration**: Single component, or multiple agents coordinated by a Workflow?

---

## 2. Component Decision Table

| Need | Component |
|------|-----------|
| Interact with an LLM, durable execution, multi-agent coordination | `Autonomous Agent` |
| Long-running multi-step process with retries/compensation | `Workflow` |
| Durable state with full event history (audit trail) | `EventSourcedEntity` |
| Durable state, simple key-value, no history needed | `KeyValueEntity` |
| Query across multiple entity instances by attribute | `View` |
| Expose HTTP API to external clients | `@HttpEndpoint` |
| Expose gRPC API for service-to-service | `@GrpcEndpoint` |
| Expose tools/resources to MCP clients | `@McpEndpoint` |
| React to events from entities, workflows, or topics | `Consumer` |
| Schedule future actions with delivery guarantees | `Timer` |

**Golden Rule**: Choose the right multi-agent coordination. Use Workflows when the sequence of steps is fixed in code. Use an Autonomous Agent when the model decides which agent runs next. Autonomous Agents can delegate subtasks, hand off work, lead teams, and moderate conversations using built-in capabilities exposed as tools.

---

## 3. Project Structure

Always follow this layout — the runtime expects it:

```
src/
├── main/
│   ├── java/com/example/
│   │   ├── api/           ← HTTP, gRPC, MCP endpoints ONLY
│   │   ├── application/   ← Agents, Workflows, Entities, Views, Consumers
│   │   └── domain/        ← Plain Java business logic (no Akka imports)
│   └── resources/
│       └── application.conf
└── test/
    └── java/com/example/  ← mirrors main structure
```

Rules:
- `domain/` classes have **zero** Akka imports — pure Java records and business logic.
- `api/` layer never calls `domain/` directly — routes through `application/` via `ComponentClient`.
- Inner layers never depend on outer layers.

---

## 4. Effects — Declare What, Not How

Every component method returns an `Effect<T>` (or `StepEffect`, `StreamEffect`, `ReadOnlyEffect`).
**Never** perform side effects directly in a command handler — always delegate via Effect.

```java
// CORRECT — declarative
public Effect<Done> createWallet(int initialBalance) {
    return effects()
        .persist(new WalletEvent.Created(initialBalance))
        .thenReply(__ -> done());
}

// WRONG — imperative side effect inside handler
public Effect<Done> createWallet(int initialBalance) {
    database.insert(initialBalance); // ← never do this
    return effects().reply(done());
}
```

### Effect types per component

| Component | Effect API |
|-----------|-----------|
| Autonomous Agent | `model()`, `systemMessage()`, `userMessage()`, `tools()`, `memory()`, `thenReply()`, `responseAs()` |
| EventSourcedEntity | `persist(event)`, `thenReply()`, `error()`, `delete()` |
| KeyValueEntity | `updateState()`, `deleteEntity()`, `thenReply()`, `error()`, `expireAfter()` |
| Workflow | `updateState()`, `transitionTo()`, `thenReply()`, `thenPause()`, `thenEnd()`, `error()` |
| Workflow Step | `stepEffects().updateState()`, `thenTransitionTo()`, `thenEnd()`, `thenPause()` |
| View | `effects().updateRow()`, `deleteRow()`, `ignore()` |
| Consumer | `effects().done()`, `ignore()`, `produce(event)` |

---

## 5. Autonomous Agents

### Basic structure

Autonomous Agents are AI model-driven components that run as durable processes. They work on typed tasks, each with its own identity, instructions, and result schema. State is persisted, so work survives crashes and restarts.

```java
@Component(id = "my-agent")
public class MyAgent extends Agent {

    private static final String SYSTEM_MESSAGE = """
        You are a helpful assistant specialized in...
        """.stripIndent();

    public Effect<String> query(String question) {
        return effects()
            .systemMessage(SYSTEM_MESSAGE)
            .userMessage(question)
            .thenReply();
    }
}
```

### Model configuration (application.conf)

```hocon
akka.javasdk.agent {
    model-provider = openai
    openai {
        model-name = "gpt-4o-mini"
        api-key = ${?OPENAI_API_KEY}
    }
}
```

Supported providers: `openai`, `azure`, `anthropic`, `bedrock`, `googleaigemini`, `vertexai`, `mistralai`, `huggingface`, `localai`, `ollama`.
Models can be configured to enable thinking/reasoning, and agents support multimodal (image) inputs. Prompt caching is supported for Anthropic and Bedrock.

### Structured responses

```java
// Always instruct the model in the system message to return JSON
public record Activity(String name, String description) {}

public Effect<Activity> query(String message) {
    return effects()
        .systemMessage(SYSTEM_MESSAGE) // includes JSON format instructions
        .userMessage(message)
        .responseConformsTo(Activity.class) // generates JSON schema automatically
        .thenReply();
}
```

Use `responseConformsTo` (schema-enforced, preferred) over `responseAs` (parse-only).

### Session memory

```java
// Default: enabled, shared by session ID
// Disable for stateless agents:
effects().memory(MemoryProvider.none())

// Limit context window:
effects().memory(MemoryProvider.limitedWindow().readLast(5))

// Filter by agent in multi-agent sessions:
effects().memory(
    MemoryProvider.limitedWindow()
        .filtered(MemoryFilter.includeFromAgentId("weather-agent"))
)
```

### Streaming responses

```java
// Agent method — returns StreamEffect
public StreamEffect query(String message) {
    return streamEffects()
        .systemMessage(SYSTEM_MESSAGE)
        .userMessage(message)
        .thenReply();
}

// Endpoint consuming the stream
@Post("/ask")
public HttpResponse ask(Request request) {
    var tokenStream = componentClient
        .forAgent()
        .inSession(request.sessionId())
        .tokenStream(MyAgent::query)
        .source(request.message());

    return HttpResponses.streamText(tokenStream);
}
```

### Function tools

```java
// Tool defined in a separate class
public class WeatherService {
    @FunctionTool(description = "Returns weather forecast for a given city.")
    public String getWeather(
        @Description("City name") String location,
        @Description("Date in yyyy-MM-dd format") Optional<String> date
    ) { ... }
}

// Register in agent
public Effect<String> query(String message) {
    return effects()
        .systemMessage(SYSTEM_MESSAGE)
        .tools(weatherService)       // instance
        .tools(WeatherService.class) // or class (lazy, requires DependencyProvider)
        .userMessage(message)
        .thenReply();
}

// Tool defined directly in agent (implicit registration)
@FunctionTool(description = "Returns current date in yyyy-MM-dd format")
private String getCurrentDate() {
    return LocalDate.now().format(DateTimeFormatter.ISO_LOCAL_DATE);
}
```

### Autonomous Multi-Agent Coordination

Autonomous Agents have built-in capabilities for multi-agent coordination exposed as tools. Without manual orchestration code, they can:
- Delegate subtasks to specialist workers
- Hand off work to a peer
- Lead a team that shares a task list
- Moderate a turn-taking conversation

When the flow is dynamic and determined by the model, use Autonomous Agents.

### Calling agents from Workflows

Use Workflows to call agents only when the sequence of steps is deterministic and fixed.

---

## 6. Workflows

### Structure and settings
Workflows have a fluent API that makes defining multi-step processes intuitive. Workflows can also emit streams of real-time notifications to subscribers to keep downstream systems informed as steps progress.

```java
@Component(id = "transfer")
public class TransferWorkflow extends Workflow<TransferState> {

    @Override
    public WorkflowSettings settings() {
        return WorkflowSettings.builder()
            .timeout(ofMinutes(10))
            .defaultStepTimeout(ofSeconds(30))
            .defaultStepRecovery(
                RecoverStrategy.maxRetries(2)
                    .failoverTo(TransferWorkflow::compensateStep)
            )
            .build();
    }

    // Public command handler — called via ComponentClient
    public Effect<Done> start(Transfer transfer) {
        return effects()
            .updateState(new TransferState(transfer))
            .transitionTo(TransferWorkflow::withdrawStep)
            .withInput(new Withdraw(transfer.from(), transfer.amount()))
            .thenReply(done());
    }

    // Private step — internal orchestration
    @StepName("withdraw")
    private StepEffect withdrawStep(Withdraw withdraw) {
        componentClient.forEventSourcedEntity(withdraw.from())
            .method(WalletEntity::withdraw)
            .invoke(withdraw.amount());

        return stepEffects()
            .updateState(currentState().withStatus(WITHDRAW_SUCCEEDED))
            .thenTransitionTo(TransferWorkflow::depositStep)
            .withInput(new Deposit(currentState().transfer().to(), withdraw.amount()));
    }

    // Read-only handler
    public ReadOnlyEffect<TransferState> getState() {
        return effects().reply(currentState());
    }
}
```

### Pausing for human-in-the-loop

```java
@StepName("wait-for-approval")
private StepEffect waitForApprovalStep() {
    return stepEffects()
        .thenPause(
            pauseSetting(ofHours(24))
                .timeoutHandler(TransferWorkflow::timeoutHandler)
        );
}

// Resume from outside
public Effect<String> approve() {
    if (currentState().status() != WAITING_FOR_APPROVAL)
        return effects().error("Cannot approve in status: " + currentState().status());

    return effects()
        .transitionTo(TransferWorkflow::executeStep)
        .thenReply("Approved");
}
```

### Concurrent agent calls in a workflow step

```java
private StepEffect askWeatherAndTraffic() throws Exception {
    var weatherFuture = componentClient.forAgent()
        .inSession(sessionId())
        .method(WeatherAgent::query)
        .invokeAsync(currentState().userQuery());

    var trafficFuture = componentClient.forAgent()
        .inSession(sessionId())
        .method(TrafficAgent::query)
        .invokeAsync(currentState().userQuery());

    var weather = weatherFuture.toCompletableFuture().get(30, SECONDS);
    var traffic = trafficFuture.toCompletableFuture().get(30, SECONDS);

    return stepEffects()
        .updateState(currentState().withWeather(weather).withTraffic(traffic))
        .thenTransitionTo(MyWorkflow::suggestStep);
}
```

---

## 7. Event Sourced Entities

```java
@Component(id = "wallet")
public class WalletEntity extends EventSourcedEntity<Wallet, WalletEvent> {

    @Override
    public Wallet emptyState() { return new Wallet("", 0); }

    // Command handler — validates and persists events
    public Effect<Done> withdraw(int amount) {
        if (currentState().balance() < amount)
            return effects().error("Insufficient balance");

        return effects()
            .persist(new WalletEvent.Withdrawn(amount))
            .thenReply(__ -> done());
    }

    // Read-only — routed locally, no primary region required
    public ReadOnlyEffect<Integer> getBalance() {
        return effects().reply(currentState().balance());
    }

    // TTL-based expiry — delete entity automatically after duration
    public Effect<Done> expireOldAccount() {
        return effects()
            .expireAfter(Duration.ofDays(365))
            .thenReply(__ -> done());
    }

    // Event handler — rebuilds state from events (must be pure, no side effects)
    @Override
    public Wallet applyEvent(WalletEvent event) {
        return switch (event) {
            case WalletEvent.Created c  -> new Wallet(commandContext().entityId(), c.initialBalance());
            case WalletEvent.Withdrawn w -> currentState().withdraw(w.amount());
            case WalletEvent.Deposited d -> currentState().deposit(d.amount());
        };
    }
}
```

**Rules:**
- `applyEvent` must be **pure** — no I/O, no side effects, no ComponentClient calls.
- `emptyState()` always override — avoid null checks in handlers.
- `@TypeName` on every event type to ensure serialization stability.

```java
public sealed interface WalletEvent {
    @TypeName("wallet-created")
    record Created(int initialBalance) implements WalletEvent {}

    @TypeName("wallet-withdrawn")
    record Withdrawn(int amount) implements WalletEvent {}
}
```

---

## 8. Key Value Entities

```java
@Component(id = "counter")
public class CounterEntity extends KeyValueEntity<Counter> {

    @Override
    public Counter emptyState() { return new Counter(0); }

    public Effect<Counter> increment(int delta) {
        var newCounter = currentState().increment(delta);
        return effects()
            .updateState(newCounter)
            .thenReply(newCounter);
    }

    public ReadOnlyEffect<Counter> get() {
        return effects().reply(currentState());
    }

    public Effect<Done> delete() {
        return effects().deleteEntity().thenReply(done());
    }

    // Auto-expire after 30 days if no further update
    public Effect<Done> setWithExpiry(Counter value) {
        return effects()
            .updateState(value)
            .expireAfter(Duration.ofDays(30))
            .thenReply(done());
    }
}
```

---

## 9. Views

Views and consumers can be configured to start from a snapshot instead of replaying the full event history. This significantly reduces startup time and resource usage for components that subscribe to high-volume event streams.

```java
@Component(id = "customers-by-name")
public class CustomersByNameView extends View {

    // Table updater — consumes entity events or state changes
    @Consume.FromEventSourcedEntity(CustomerEntity.class)
    public static class CustomersByNameUpdater extends TableUpdater<CustomerEntry> {

        public Effect<CustomerEntry> onEvent(CustomerEvent event) {
            return switch (event) {
                case CustomerEvent.Created c -> effects()
                    .updateRow(new CustomerEntry(c.email(), c.name()));
                case CustomerEvent.NameChanged n -> effects()
                    .updateRow(rowState().withName(n.newName()));
                case CustomerEvent.AddressChanged ignored -> effects().ignore();
            };
        }
    }

    // Query method
    @Query("SELECT * AS customers FROM customers_by_name WHERE name = :name")
    public QueryEffect<CustomerList> getCustomers(String name) {
        return queryResult();
    }

    // Streaming query with live updates
    @Query(value = "SELECT * FROM customers_by_name WHERE city = :city", streamUpdates = true)
    public QueryStreamEffect<CustomerEntry> streamByCity(String city) {
        return queryStreamResult();
    }
}
```

**Key rules:**
- `@Component` id must be **stable** — changing it rebuilds the entire view from scratch.
- Use `@TypeName` on ESE events consumed by views.
- For incompatible changes: create a NEW view class with a new `@Component` id, deploy both, migrate, then remove old.

---

## 10. HTTP Endpoints

```java
@HttpEndpoint("/carts")
@Acl(allow = @Acl.Matcher(principal = Acl.Principal.INTERNET))
public class ShoppingCartEndpoint {

    private final ComponentClient componentClient;

    public ShoppingCartEndpoint(ComponentClient componentClient) {
        this.componentClient = componentClient;
    }

    @Get("/{cartId}")
    public ShoppingCart get(String cartId) {
        return componentClient
            .forEventSourcedEntity(cartId)
            .method(ShoppingCartEntity::getCart)
            .invoke();
    }

    @Post("/{cartId}/items")
    public HttpResponse addItem(String cartId, LineItem item) {
        componentClient
            .forEventSourcedEntity(cartId)
            .method(ShoppingCartEntity::addItem)
            .invoke(item);
        return HttpResponses.created();
    }

    // Error handling
    @Get("/{cartId}/validated")
    public ShoppingCart getValidated(String cartId) {
        if (cartId.isBlank())
            throw HttpException.badRequest("Cart ID cannot be blank");

        return componentClient
            .forEventSourcedEntity(cartId)
            .method(ShoppingCartEntity::getCart)
            .invoke();
    }
}
```

**Rules:**
- No `@Acl` = no one can access the endpoint. Always declare it explicitly.
- Transform domain types to API types before returning — never expose internals.
- `IllegalArgumentException` → 400. Any other exception → 500 in prod (with correlation ID).

### SSE (Server-Sent Events)

```java
@Get("/stream/{city}")
public HttpResponse streamCustomers(String city) {
    var source = componentClient
        .forView()
        .stream(CustomersByCity::continuousCustomersInCity)
        .source(city);

    return HttpResponses.serverSentEvents(source);
}
```

---

## 11. Consumers (Event-Driven)

Like Views, Consumers can be configured to start from snapshots instead of replaying the full event history.

```java
@Component(id = "order-processor")
@Consume.FromEventSourcedEntity(OrderEntity.class)
public class OrderProcessor extends Consumer {

    public Effect onEvent(OrderEvent event) {
        return switch (event) {
            case OrderEvent.Placed placed -> {
                // process order placement
                yield effects().done();
            }
            case OrderEvent.Cancelled ignored -> effects().ignore();
        };
    }
}
```

**Supported sources:**
- `@Consume.FromEventSourcedEntity(MyEntity.class)`
- `@Consume.FromKeyValueEntity(MyEntity.class)`
- `@Consume.FromWorkflow(MyWorkflow.class)`
- `@Consume.FromServiceStream(service = "other-service", id = "stream-id")`
- `@Consume.FromTopic("topic-name")`

**Producer (publish to Kafka/PubSub):**

```java
@Component(id = "order-publisher")
@Consume.FromEventSourcedEntity(OrderEntity.class)
@Produce.ToTopic("orders")
public class OrderPublisher extends Consumer {

    public Effect onEvent(OrderEvent event) {
        return effects().produce(event);
    }
}
```

---

## 12. Multi-Agent Orchestration — Patterns

Akka supports two distinct approaches to multi-agent orchestration. Choose the right one based on whether the flow is deterministic or dynamic.

### 1. Autonomous Agent Coordination (Dynamic)

When the sequence of steps is dynamic and decided by the AI model, use an Autonomous Agent. Autonomous Agents have built-in multi-agent coordination capabilities exposed as tools. You do not need to write orchestration code.

Capabilities:
- **Delegate**: Assign a subtask to a specialist worker agent.
- **Hand-off**: Transfer work to a peer agent.
- **Lead Team**: Manage a team that shares a task list.
- **Moderate**: Moderate a turn-taking conversation among agents.

### 2. Workflow Orchestration (Deterministic)

When the sequence of steps is fixed in code, use a Workflow to call Autonomous Agents.

```
Workflow → Agent(s)           ← sequential or parallel deterministic orchestration
Workflow → A2A/ACP → Agents   ← external agent protocols
Agent → Endpoint (HTTP/gRPC)  ← calling external services
Broker → Consumer → Agent     ← event-triggered agents
```

### Forbidden patterns (never use)

```
Agent → MCP Server → Agent    ← creates hidden coupling
Endpoint → Endpoint directly  ← bypasses durable execution
```

---

## 13. Spec-Driven Development (SDD) Workflow

### Full cycle

```
/akka:setup         → verify env (Java, Maven, CLI, Akka token)
/akka:specify       → produce feature spec from natural language
/akka:clarify       → fill gaps, resolve ambiguities
/akka:plan          → technical architecture and implementation plan
/akka:tasks         → itemize work, identify parallel opportunities
/akka:implement     → generate code, tests, configuration
/akka:review        → verify code against spec + constitution
/akka:build         → compile, test, run locally, exercise endpoints
/akka:inspect       → verify running service against spec at runtime
/akka:deploy        → deploy to Akka Automated Operations
```

### Spec prompt discipline

```
/akka:specify <feature-name> - <what and why, NO technical implementation details>

Good: "The service manages user accounts. Users authenticate via username/password.
       Users can update their profile and delete their account."

Bad: "Create a KeyValueEntity with fields name and email and an HTTP endpoint
      with GET and POST methods."
```

Plan goes in `/akka:plan` — that is where architecture decisions live.

### Constitution rules (always enforced)

- Akka SDK First — use Akka components, not raw Java concurrency or external frameworks.
- Design Principles — layered architecture (api / application / domain).
- Test Coverage — unit tests for entities, integration tests for endpoints.
- Simplicity — fewest components that correctly solve the problem.

---

## 14. Testing

### Unit test — Entity

```java
@Test
public void testWithdraw() {
    var testKit = EventSourcedEntityTestKit.of(ctx -> new WalletEntity());

    testKit.method(WalletEntity::create).invoke(100);

    var result = testKit.method(WalletEntity::withdraw).invoke(30);
    assertTrue(result.isReply());
    assertEquals(done(), result.getReply());

    assertEquals(70, testKit.getState().balance());
}
```

### Integration test — Agent with mocked model

```java
public class AgentTest extends TestKitSupport {

    private final TestModelProvider modelProvider = new TestModelProvider();

    @Override
    protected TestKit.Settings testKitSettings() {
        return TestKit.Settings.DEFAULT
            .withModelProvider(MyAgent.class, modelProvider);
    }

    @Test
    public void testAgentQuery() {
        modelProvider.fixedResponse("Mocked agent response");

        var result = componentClient
            .forAgent()
            .inSession("test-session")
            .method(MyAgent::query)
            .invoke("Test question");

        assertThat(result).contains("Mocked");
    }
}
```

### Integration test — Consumer with mocked topic

```java
@Override
protected TestKit.Settings testKitSettings() {
    return TestKit.Settings.DEFAULT
        .withTopicIncomingMessages("orders")
        .withTopicOutgoingMessages("order-results");
}

@Test
public void testOrderProcessing() {
    var ordersTopic = testKit.getTopicIncomingMessages("orders");
    var resultsTopic = testKit.getTopicOutgoingMessages("order-results");

    ordersTopic.publish(new Order("order-1", 100.0), "order-1");

    Awaitility.await().atMost(10, SECONDS).untilAsserted(() -> {
        var result = resultsTopic.expectOneTyped(OrderResult.class);
        assertEquals("order-1", result.getPayload().orderId());
    });
}
```

---

## 15. Code Quality Rules

### Architecture
- Domain classes have **zero** Akka imports.
- Endpoints never call domain layer directly — always via ComponentClient.
- Workflows own deterministic orchestration, retries, compensation, and state transitions.
- Autonomous Agents own dynamic multi-agent orchestration via built-in capabilities.

### Entities
- Always override `emptyState()`.
- Always annotate events with `@TypeName`.
- `applyEvent()` must be pure — no I/O, no ComponentClient.
- Never reuse entity IDs after deletion.
- Never make the `@Component` id unstable.

### Views
- Never change `@Component` id after production deploy.
- For incompatible schema changes: new class, new id, two-phase deployment.
- Views are eventually consistent — document this for callers.

### Agents
- Set step timeout > 30s for LLM calls (they are slow).
- Always define `defaultStepRecovery` in Workflow settings when calling agents.
- Use `responseConformsTo` for structured output when the model supports schema.
- `MemoryProvider.none()` for agents that should not retain session history.

### Endpoints
- Every endpoint must have an explicit `@Acl` annotation.
- Never expose internal domain types directly — map to API records.
- Use `HttpResponses` factory methods for consistent response construction.

---

## 16. Architecture Decision Guide

| Question | Answer |
|----------|--------|
| Single LLM task | `Autonomous Agent` alone |
| Multiple LLM tasks in a fixed sequence | `Workflow` orchestrating `Autonomous Agent`s |
| Need retry / compensation / durable execution | Always `Workflow` |
| Audit trail of all state changes required | `EventSourcedEntity` |
| Simple state, no history needed | `KeyValueEntity` |
| Query multiple entities by attribute | `View` |
| React to entity changes asynchronously | `Consumer` |
| AI with access to domain state | `Autonomous Agent` + `ComponentClient` + Entity/View as tools |
| Human approval checkpoint | `Workflow` with `thenPause()` |
| Dynamic multi-agent coordination | `Autonomous Agent` using delegate/handoff/lead capabilities |
| Stream LLM tokens to browser | `StreamEffect` in Agent + SSE in Endpoint |
| New feature from scratch | Use SDD: `/akka:specify` → `/akka:plan` → `/akka:implement` |
| Millions of events/sec from Kafka | `Consumer` + `@Consume.FromTopic` + idempotent entity writes |
| Single entity receiving all writes (hotspot) | Split by sub-key + aggregate in `View` (CQRS) |
| Batch processing with guaranteed completion | `Workflow` with paginated steps + `defaultStepRecovery` |
| Cross-service high-volume propagation | `@Produce.ServiceStream` + `@Consume.FromServiceStream` |
| 99.9999% availability requirement | Multi-region deployment + replicated reads |
| Write anywhere, eventually consistent | Replicated writes (CRDT) on `EventSourcedEntity` |

---

## 17. High-Volume Processing Patterns

This section covers architectural decisions specific to systems requiring high throughput,
large-scale data pipelines, and sustained concurrent workloads.

### Throughput baseline

Akka's actor-based runtime achieves up to **1.4M TPS at 9ms latency** in benchmarks.
The key to reaching this is correct component selection and avoiding blocking patterns.

### Stream processing with Consumers

For continuous, unbounded event streams — Kafka topics, entity event journals, external
broker topics — use `Consumer` components. They run on Akka's streaming runtime, which
handles backpressure, flow control, and exactly-once processing automatically.

```java
// High-volume Kafka consumer with at-least-once delivery
@Component(id = "order-stream-processor")
@Consume.FromTopic("orders-raw")
@Produce.ToTopic("orders-enriched")
public class OrderStreamProcessor extends Consumer {

    private final ComponentClient componentClient;

    public OrderStreamProcessor(ComponentClient componentClient) {
        this.componentClient = componentClient;
    }

    public Effect onOrder(OrderEvent order) {
        // Validate — reject malformed events explicitly
        if (order.amount() <= 0)
            return effects().ignore(); // don't poison the stream

        // Enrich from entity state (non-blocking, actor-routed)
        var customer = componentClient
            .forKeyValueEntity(order.customerId())
            .method(CustomerEntity::get)
            .invoke();

        return effects().produce(new EnrichedOrder(order, customer.tier()));
    }
}
```

**Critical rules for high-volume Consumers:**
- Always handle `effects().ignore()` for events you don't process — never let unhandled
  events stall the stream.
- Consumers are **at-least-once** — implement idempotent handlers or use
  sequence-number deduplication in the entity they write to.
- Never call blocking code inside a Consumer — the streaming runtime is non-blocking.

### Idempotent entity writes (deduplication)

At-least-once delivery means duplicate events will arrive. Always design entity command
handlers to be safe when called more than once with the same input.

```java
@Component(id = "payment")
public class PaymentEntity extends EventSourcedEntity<Payment, PaymentEvent> {

    // Idempotent: second call with same commandId is a no-op
    public Effect<Done> process(ProcessPayment cmd) {
        if (currentState() != null &&
            currentState().commandId().equals(cmd.commandId())) {
            return effects().reply(done()); // already processed — safe to ignore
        }
        return effects()
            .persist(new PaymentEvent.Processed(cmd.commandId(), cmd.amount()))
            .thenReply(__ -> done());
    }
}
```

### Sharding and elasticity

Akka automatically shards stateful components (Entities, Workflows) across cluster nodes.
No configuration needed — the runtime rebalances as nodes are added or removed.

**Design rule**: Entity IDs are the sharding key. Choose IDs that distribute load evenly.
Avoid hotspot IDs (e.g., a single `global-counter` entity that every request writes to).

```java
// Good: distributed sharding — one entity per customer
componentClient.forEventSourcedEntity(customerId).method(...)

// Bad: hotspot — all writes go to one entity instance
componentClient.forEventSourcedEntity("global-aggregator").method(...)
```

For genuinely global aggregations, use a `View` (read-side CQRS) built from events,
not a single write-path entity.

### CQRS for high-read workloads

Separate read and write paths. Entities handle writes; Views handle reads. This allows
both to scale independently.

```java
// Write path — entity handles commands, emits events
@Component(id = "product")
public class ProductEntity extends EventSourcedEntity<Product, ProductEvent> {
    public Effect<Done> updatePrice(BigDecimal newPrice) {
        return effects()
            .persist(new ProductEvent.PriceChanged(newPrice))
            .thenReply(__ -> done());
    }
}

// Read path — view indexes for fast query patterns
@Component(id = "products-by-category")
public class ProductsByCategoryView extends View {

    @Consume.FromEventSourcedEntity(ProductEntity.class)
    public static class Updater extends TableUpdater<ProductRow> {
        public Effect<ProductRow> onEvent(ProductEvent event) {
            return switch (event) {
                case ProductEvent.PriceChanged p ->
                    effects().updateRow(rowState().withPrice(p.newPrice()));
                // handle other events...
            };
        }
    }

    @Query("SELECT * AS products FROM products_by_category WHERE category = :category " +
           "ORDER BY price ASC")
    public QueryEffect<ProductList> getByCategory(String category) {
        return queryResult();
    }
}
```

### Service-to-service eventing (brokerless)

For high-volume inter-service communication within the same Akka project, prefer
service-to-service eventing over Kafka. It avoids broker overhead and is guaranteed
at-least-once within the cluster.

```java
// Producer service — publishes entity events as a stream
@Component(id = "inventory-events")
@Consume.FromEventSourcedEntity(InventoryEntity.class)
@Produce.ServiceStream(id = "inventory_changes")
@Acl(allow = @Acl.Matcher(service = "*"))
public class InventoryEventProducer extends Consumer {

    public Effect onEvent(InventoryEvent event) {
        return switch (event) {
            case InventoryEvent.StockChanged s ->
                effects().produce(new PublicInventoryEvent.StockChanged(s.sku(), s.qty()));
            case InventoryEvent.Reserved ignored -> effects().ignore();
        };
    }
}

// Consumer in another service
@Consume.FromServiceStream(service = "inventory-service", id = "inventory_changes")
public static class InventoryViewUpdater extends TableUpdater<InventoryRow> {
    public Effect<InventoryRow> onEvent(PublicInventoryEvent.StockChanged e) {
        return effects().updateRow(new InventoryRow(e.sku(), e.qty()));
    }
}
```

### Workflow for large-scale durable batch processing

For batch jobs that process millions of records with guaranteed completion, use Workflows
with parallel step execution and built-in recovery.

```java
@Component(id = "batch-processor")
public class BatchWorkflow extends Workflow<BatchState> {

    @StepName("process-batch")
    private StepEffect processBatchStep() {
        var batch = currentState().nextBatch();

        // Fan out — invoke entities for each record in the batch
        var futures = batch.records().stream()
            .map(record -> componentClient
                .forEventSourcedEntity(record.id())
                .method(RecordEntity::process)
                .invokeAsync(record))
            .toList();

        // Collect all results
        futures.forEach(f -> {
            try { f.toCompletableFuture().get(10, SECONDS); }
            catch (Exception e) { throw new RuntimeException("Batch item failed", e); }
        });

        return stepEffects()
            .updateState(currentState().markBatchComplete())
            .thenTransitionTo(
                currentState().hasMoreBatches()
                    ? BatchWorkflow::processBatchStep
                    : BatchWorkflow::finalizeStep
            );
    }
}
```

### Multi-region for high availability

For 99.9999% availability, deploy to multiple regions. Akka replicates entity state
automatically. No code changes required.

```
replicated reads (default):
  - reads served locally in each region (low latency)
  - writes routed to primary region (strong consistency)

replicated writes (CRDT):
  - writes accepted in any region (highest write availability)
  - state is eventually consistent (use only where conflicts are tolerable)
```

**Choosing the replication mode per use case:**

| Use case | Mode |
|----------|------|
| User profiles, orders, accounts | Replicated reads (request-region primary) |
| Geo-homed data (GDPR data pinning) | Replicated reads (pinned-region primary) |
| Real-time counters, leaderboards | Replicated writes (CRDT) |
| Global inventory with strict consistency | Replicated reads + single primary region |

### High-volume decision matrix extension

| Volume pattern | Solution |
|---------------|---------|
| Millions of events/sec from Kafka | `Consumer` + `@Consume.FromTopic` with idempotent entity writes |
| Hot entity (single ID, many writes) | Split into multiple entities by sub-key, aggregate in `View` |
| Fan-out to thousands of entities | `Workflow` step with `invokeAsync` + `CompletableFuture.get` |
| Cross-service event propagation | `@Produce.ServiceStream` + `@Consume.FromServiceStream` |
| Large result sets | `QueryStreamEffect` in View, stream to client via SSE |
| Global low-latency reads | Multi-region deployment + `ReadOnlyEffect` on entities |
| Batch processing with recovery | `Workflow` with paginated steps and `defaultStepRecovery` |

---

## Reference

- Akka SDK docs: https://code.akka.io/docs
- SDD guide: https://doc.akka.io/concepts/spec-driven-development.html
- Multi-agent sample: https://github.com/akka-samples/multi-agent
- Akka plugin marketplace: `/plugin marketplace add akka/ai-marketplace`
