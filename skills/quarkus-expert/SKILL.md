---
name: quarkus-expert
description: >
  Expert Quarkus development skill covering reactive programming with Mutiny, Hibernate Reactive with Panache,
  non-blocking I/O, event bus, datasource configuration, and the reactive execution model.
  Use this skill whenever the user asks to build, debug, refactor, or explain Quarkus applications
  — especially when involving Uni/Multi, reactive REST endpoints, reactive ORM, Kafka/Pulsar messaging,
  gRPC, Infinispan, Stork, or any component that interacts with the Quarkus reactive engine.
  Also use when comparing reactive vs. imperative approaches, choosing between @Blocking/@NonBlocking,
  designing persistence layers with Panache, or architecting reactive microservices on Quarkus.
license: MIT
---

You are an expert Quarkus architect and developer. The user is asking for help building, reviewing, debugging,
or explaining a Quarkus application. Apply deep knowledge of the Quarkus reactive stack: Mutiny, Hibernate
Reactive with Panache, Vert.x event bus, reactive SQL clients, reactive messaging, and the Quarkus execution model.

The user may provide: a feature description, existing code to refactor, an error message, an architecture question,
or a comparison between approaches. Always reason about correctness, performance, and thread-safety before generating code.

---

## 1. Understand the Context First

Before writing any code, determine:

- **Execution model**: Is this a purely reactive pipeline (I/O thread only), a blocking operation needing `@Blocking`,
  or a mixed workload? Never assume — look at method signatures and what they call.
- **Persistence strategy**: Is the user using Hibernate ORM (JDBC, blocking) or Hibernate Reactive (non-blocking)?
  These are mutually exclusive in their session/transaction handling. Never mix their APIs.
- **Composition scope**: Is the user chaining multiple async operations? Identify whether they need sequential
  composition (`transformToUni`), parallel execution (`Uni.combine().all()`), or streaming (`Multi`).
- **Transaction boundaries**: Every write operation in Hibernate Reactive requires a session and a transaction.
  Identify the correct boundary annotation or programmatic wrapper.

---

## 2. The Quarkus Reactive Execution Model

### Golden Rule: Never block an I/O thread.

Quarkus uses a small pool of Vert.x I/O threads to handle all non-blocking work. Blocking one of these threads
(e.g., with `Thread.sleep`, JDBC calls, or `.await().indefinitely()` inside a reactive pipeline) causes severe
throughput degradation. Always verify where the code runs.

### Thread Dispatch Decision Matrix

| Situation | Correct approach |
|-----------|-----------------|
| Method returns `Uni<T>` or `Multi<T>` | Runs on I/O thread by default — **never block** |
| Method is annotated `@Blocking` | Quarkus dispatches it to worker thread pool — blocking is allowed |
| CDI bean with `@Transactional` on a reactive Uni method | Use `@WithTransaction` instead; mixing causes `UnsupportedOperationException` |
| Quarkus REST endpoint returning `Uni<T>` | Non-blocking path; Quarkus REST subscribes for you |
| Quarkus REST endpoint returning plain `T` | Blocking worker thread; safe to use JDBC/ORM |

### The Proactor Pattern

Quarkus automatically routes requests to the correct thread based on method signature and annotations.
Explicit hints (`@Blocking`, `@NonBlocking`, `@RunOnVirtualThread`) override the defaults. Prefer
explicit annotations when the intent is non-obvious.

---

## 3. Mutiny — Core Reactive API

### Uni vs Multi

| Type | Emits | Use case |
|------|-------|----------|
| `Uni<T>` | 0 or 1 item, then optionally a failure | DB lookup, HTTP call, single insert |
| `Multi<T>` | 0..n items, then completion or failure | Streaming results, Kafka consumer, event streams |

### Essential Operators

Always prefer the shorthand aliases — they are easier to read and maintain:

```java
// Transform item synchronously
uni.map(item -> transform(item))                   // alias: onItem().transform()

// Chain to another async operation (sequential composition)
uni.chain(item -> anotherUni(item))                // alias: onItem().transformToUni()

// Side effects without changing the item
uni.invoke(item -> log(item))                      // sync side-effect
uni.call(item -> asyncSideEffect(item))            // async side-effect, original item passes through

// Replace the item
uni.replaceWith(newValue)
uni.replaceWith(otherUni)

// Run something whether item or failure (like finally)
uni.eventually(() -> cleanup())
uni.eventually(() -> asyncCleanup())

// Failure handling
uni.onFailure().recoverWithItem(fallback)
uni.onFailure().recoverWithUni(failure -> fallbackUni(failure))
uni.onFailure().retry().atMost(3)
uni.onFailure().retry().withBackOff(Duration.ofMillis(100)).atMost(5)

// Parallel execution — combine when ALL complete
Uni.combine().all().unis(uniA, uniB).asTuple()
Uni.combine().all().unis(uniA, uniB).with((a, b) -> merge(a, b))

// Null handling
uni.onItem().ifNull().continueWith(defaultValue)
uni.onItem().ifNull().failWith(new IllegalStateException("Expected a value"))

// Timeout
uni.ifNoItem().after(Duration.ofSeconds(5)).fail()
```

### Multi — Streaming Operations

```java
// Filter items
multi.select().where(item -> item.isActive())

// Transform each item
multi.map(item -> transform(item))

// Flatten into a new stream per item
multi.onItem().transformToUniAndMerge(item -> fetchDetails(item))   // concurrent
multi.onItem().transformToUniAndConcatenate(item -> fetchDetails(item))  // ordered

// Collect into a list (converts Multi back to Uni)
multi.collect().asList()

// Limit and skip
multi.select().first(10)
multi.skip().first(5)

// Overflow strategy (when producer is faster than consumer)
multi.onOverflow().drop()
multi.onOverflow().buffer(100)
```

### Subscription — Mandatory to Trigger Execution

`Uni` and `Multi` are **lazy**. Nothing runs until subscribed. In Quarkus, extensions (Quarkus REST,
Reactive Messaging, Panache) subscribe on your behalf when you return a `Uni`/`Multi`. When subscribing manually:

```java
uni.subscribe().with(
    item    -> handle(item),
    failure -> handleError(failure)
);

multi.subscribe().with(
    item       -> handle(item),
    failure    -> handleError(failure),
    ()         -> onCompletion()
);
```

**Warning**: Never call `.await().indefinitely()` or `.await().atMost(duration)` on an I/O thread.
Use them only in `@Test` methods or inside methods annotated with `@Blocking`.

---

## 4. Hibernate Reactive with Panache

### Imports — Always Use the Reactive Variants

```java
import io.quarkus.hibernate.reactive.panache.PanacheEntity;        // NOT hibernate-orm
import io.quarkus.hibernate.reactive.panache.PanacheRepository;    // NOT hibernate-orm
import io.quarkus.hibernate.reactive.panache.Panache;
```

### Active Record Pattern (preferred for simple entities)

```java
@Entity
public class Book extends PanacheEntity {
    public String title;
    public String author;
    public LocalDate published;

    // Custom queries as static methods — co-located with the entity
    public static Uni<Book> findByTitle(String title) {
        return find("title", title).firstResult();
    }

    public static Uni<List<Book>> findByAuthor(String author) {
        return list("author", author);
    }

    public static Uni<Long> countByAuthor(String author) {
        return count("author", author);
    }

    public static Uni<Long> deleteByAuthor(String author) {
        return delete("author", author);
    }
}
```

### Repository Pattern (preferred when business logic is complex)

```java
@ApplicationScoped
public class BookRepository implements PanacheRepository<Book> {

    public Uni<List<Book>> findRecentByAuthor(String author, int year) {
        return list("author = ?1 and year(published) >= ?2", author, year);
    }

    public Uni<Optional<Book>> findLatestByAuthor(String author) {
        return find("author = ?1 order by published desc", author)
            .firstResultOptional();
    }
}

// Inject and use:
@Inject BookRepository bookRepository;

Uni<List<Book>> books = bookRepository.findRecentByAuthor("Borges", 1940);
```

### Query Shorthand Forms

Panache expands partial HQL automatically:

| Written as | Expanded to |
|-----------|-------------|
| `find("author", "Cortázar")` | `from Book where author = 'Cortázar'` |
| `find("author = ?1 and title like ?2", a, t)` | Full HQL with positional params |
| `find("name = :name", Map.of("name", v))` | Named parameters |
| `list("order by title")` | `from Book order by title` |
| `update("title = 'New' where id = ?1", id)` | `update Book set title = 'New' where id = ?` |
| `delete("author", "Anonymous")` | `delete from Book where author = 'Anonymous'` |

### Transactions and Sessions — Critical Rules

1. **Every write** (persist, delete, update) **requires a transaction**.
2. **Every Panache call requires an active Mutiny session**.
3. Do **not** mix `@Transactional` with `@WithTransaction`/`@WithSession` in the same application.
   Pick one model and apply it consistently throughout.

```java
// Declarative transaction — wraps the whole Uni pipeline
@WithTransaction
public Uni<Void> saveBook(Book book) {
    return book.persist();
}

// Programmatic transaction — inline
public Uni<Void> saveBook(Book book) {
    return Panache.withTransaction(() -> book.persist());
}

// Declarative session — for read-only operations
@WithSession
public Uni<List<Book>> getAllBooks() {
    return Book.listAll();
}

// Multiple persistence units
@WithTransaction("inventory-pu")
public Uni<Void> updateInventory(Item item) {
    return item.persist();
}
```

### Paging Large Result Sets

Never use `listAll()` for unbounded tables. Use `PanacheQuery` with pages:

```java
PanacheQuery<Book> query = Book.find("author", author);
query.page(Page.ofSize(25));

Uni<List<Book>> firstPage  = query.list();
Uni<List<Book>> secondPage = query.nextPage().list();
Uni<List<Book>> page7      = query.page(Page.of(7, 25)).list();
Uni<Integer>    totalPages = query.pageCount();
Uni<Long>       totalCount = query.count();
```

### Query Projection (DTO Mapping)

```java
@RegisterForReflection
public class BookSummary {
    public final String title;
    public final String author;

    public BookSummary(String title, String author) {
        this.title  = title;
        this.author = author;
    }
}

// Only title and author columns are loaded from the DB
PanacheQuery<BookSummary> query = Book.find("status", Status.PUBLISHED)
    .project(BookSummary.class);
```

### persistAndFlush for Immediate Constraint Feedback

```java
@WithTransaction
public Uni<Book> createWithValidation(Book book) {
    return book.persistAndFlush()
        .onFailure(PersistenceException.class)
        .recoverWithItem(e -> {
            log.errorf("Constraint violation saving book '%s': %s", book.title, e.getMessage());
            return null;
        });
}
```

### Lock Management

```java
// Pessimistic write lock via findById
Panache.withTransaction(() ->
    Book.<Book>findById(id, LockModeType.PESSIMISTIC_WRITE)
        .invoke(book -> book.status = Status.RESERVED)
);

// Pessimistic write lock via find()
Panache.withTransaction(() ->
    Book.<Book>find("title", title)
        .withLock(LockModeType.PESSIMISTIC_WRITE)
        .firstResult()
        .invoke(book -> book.status = Status.RESERVED)
);
```

---

## 5. Datasource Configuration

### JDBC (blocking — use with Hibernate ORM, not Hibernate Reactive)

```properties
quarkus.datasource.db-kind=postgresql
quarkus.datasource.username=user
quarkus.datasource.password=secret
quarkus.datasource.jdbc.url=jdbc:postgresql://localhost:5432/mydb
quarkus.datasource.jdbc.max-size=16
quarkus.datasource.jdbc.min-size=4
```

### Reactive (non-blocking — use with Hibernate Reactive and reactive SQL clients)

```properties
quarkus.datasource.db-kind=postgresql
quarkus.datasource.username=user
quarkus.datasource.password=secret
quarkus.datasource.reactive.url=postgresql://localhost:5432/mydb
quarkus.datasource.reactive.max-size=20
```

### Using Both JDBC and Reactive Simultaneously

```properties
%prod.quarkus.datasource.jdbc.url=jdbc:postgresql://localhost:5432/mydb
%prod.quarkus.datasource.reactive.url=postgresql://localhost:5432/mydb

# Explicitly disable one if not needed
quarkus.datasource.jdbc=false      # disable JDBC
quarkus.datasource.reactive=false  # disable reactive
```

### Multiple Named Datasources

```properties
# Default datasource
quarkus.datasource.db-kind=postgresql
quarkus.datasource.jdbc.url=jdbc:postgresql://localhost:5432/main

# Named datasource
quarkus.datasource.reporting.db-kind=postgresql
quarkus.datasource.reporting.jdbc.url=jdbc:postgresql://localhost:5432/reporting
quarkus.datasource.reporting.username=reporter
quarkus.datasource.reporting.password=secret
```

```java
@Inject
AgroalDataSource defaultDs;

@Inject
@DataSource("reporting")
AgroalDataSource reportingDs;
```

### Dev Services (zero-config for dev and test)

If no connection URL is configured in dev mode, Quarkus automatically starts a containerized database.
Isolate production credentials with `%prod.` prefix:

```properties
%prod.quarkus.datasource.reactive.url=postgresql://prod-server:5432/mydb
```

### Activating / Deactivating Datasources at Runtime

```properties
quarkus.datasource."pg".active=false      # disabled at build time, enabled via runtime config
quarkus.datasource."oracle".active=false
```

```java
// Inject safely without triggering activation failure
@Inject @Any
InjectableInstance<DataSource> dataSource;

DataSource active = dataSource.getActive();
```

---

## 6. Vert.x Event Bus

Use for decoupled, non-blocking intra-application messaging. Supports point-to-point, pub/sub, and request/reply.

### Consumer

```java
@ApplicationScoped
public class NotificationService {

    // Non-blocking consumer — runs on I/O thread
    @ConsumeEvent("user.created")
    public Uni<String> onUserCreated(String userId) {
        return sendWelcomeEmail(userId);
    }

    // Blocking consumer — runs on worker thread
    @ConsumeEvent(value = "report.generate", blocking = true)
    public void generateReport(String params) {
        // safe to call blocking code here
    }

    // Fire and forget — no reply expected
    @ConsumeEvent("audit.log")
    public void logAuditEvent(String event) {
        auditLogger.write(event);
    }
}
```

### Producer

```java
@Inject EventBus bus;

// Request-reply (async)
Uni<String> response = bus.<String>request("user.created", userId)
    .onItem().transform(Message::body);

// Fire and forget
bus.requestAndForget("audit.log", event);

// Publish to all consumers at the address
bus.publish("broadcast.announcement", message);
```

### Custom Object Codec

```java
// Sender
bus.<MyResult>request("process", new MyInput(data),
    new DeliveryOptions().setCodecName(MyInputCodec.class.getName()))
    .onItem().transform(Message::body);

// Consumer
@ConsumeEvent(value = "process", codec = MyInputCodec.class)
public Uni<MyResult> process(MyInput input) { ... }
```

---

## 7. Reactive Messaging (Kafka / AMQP / Pulsar)

### Consuming Messages

```java
@ApplicationScoped
public class OrderProcessor {

    // Simple payload consumer
    @Incoming("orders")
    public Uni<Void> processOrder(Order order) {
        return inventoryService.reserve(order)
            .chain(reserved -> notificationService.notify(order));
    }

    // Message wrapper for manual ack/nack control
    @Incoming("orders")
    public Uni<Void> processWithAck(Message<Order> message) {
        return processOrder(message.getPayload())
            .chain(() -> message.ack())
            .onFailure().recoverWithUni(e -> message.nack(e));
    }
}
```

### Producing Messages

```java
// Emitter — inject and send from anywhere
@Inject
@Channel("order-results")
MutinyEmitter<OrderResult> emitter;

public Uni<Void> emitResult(OrderResult result) {
    return emitter.send(result);
}

// Processor — consume and re-emit transformed messages
@Incoming("raw-events")
@Outgoing("enriched-events")
public Uni<EnrichedEvent> enrich(RawEvent raw) {
    return metadataService.enrich(raw);
}
```

### Kafka Configuration

```properties
mp.messaging.incoming.orders.connector=smallrye-kafka
mp.messaging.incoming.orders.topic=orders
mp.messaging.incoming.orders.group.id=order-processor
mp.messaging.incoming.orders.value.deserializer=io.quarkus.kafka.client.serialization.JsonbDeserializer

mp.messaging.outgoing.order-results.connector=smallrye-kafka
mp.messaging.outgoing.order-results.topic=order-results
mp.messaging.outgoing.order-results.value.serializer=io.quarkus.kafka.client.serialization.JsonbSerializer
```

---

## 8. SmallRye Stork — Service Discovery & Load Balancing

Use the `stork://` URI scheme in REST client definitions to delegate instance selection to Stork.
Stork handles both discovery (find available instances) and selection (pick the best one).

```java
@RegisterRestClient(baseUri = "stork://inventory-service")
public interface InventoryClient {
    @GET @Path("/stock/{id}")
    Uni<StockLevel> getStock(@PathParam("id") String id);
}
```

```properties
# Consul-based discovery + round-robin load balancing
quarkus.stork.inventory-service.service-discovery.type=consul
quarkus.stork.inventory-service.service-discovery.consul-host=localhost
quarkus.stork.inventory-service.service-discovery.consul-port=8500
quarkus.stork.inventory-service.load-balancer.type=round-robin
```

Stork integrates with Consul, Kubernetes, Eureka, and DNS out of the box. The load balancer supports
`round-robin`, `random`, `least-requests`, `power-of-two-choices`, and custom strategies.

---

## 9. Infinispan — Distributed Caching

```java
@Inject
@Remote("books-cache")
RemoteCache<String, Book> booksCache;

// Synchronous get (fine inside @Blocking or worker thread)
Book book = booksCache.get(isbn);

// Async put
CompletableFuture<Book> future = booksCache.putAsync(isbn, book);

// Named client injection
@Inject
@InfinispanClientName("site-lon")
RemoteCacheManager lonClient;
```

Near-cache for read-heavy workloads (reduces round-trips to server):

```properties
quarkus.infinispan-client.cache.books-cache.near-cache-mode=INVALIDATED
quarkus.infinispan-client.cache.books-cache.near-cache-max-entries=500
quarkus.infinispan-client.cache.books-cache.near-cache-use-bloom-filter=true
```

Protobuf serialization is required for user types. Use `@Proto` on records/classes and define a `@ProtoSchema`:

```java
@Proto
public record Book(String title, String author, int year) {}

@ProtoSchema(includeClasses = { Book.class }, schemaPackageName = "library")
interface LibrarySchema extends GeneratedSchema {}
```

---

## 10. Code Quality Rules

Apply these rules to every piece of code you generate or review:

### Reactive pipeline correctness
- **Always return** the `Uni`/`Multi` — forgetting to return breaks the chain silently with no error.
- **Never subscribe inside a Quarkus-managed method** that already returns `Uni`/`Multi`. The framework subscribes.
- **Use `.chain()` for sequential async operations**; never nest `.subscribe()` calls inside pipelines.
- **Do not call blocking code on I/O threads**. If unsure, annotate the method with `@Blocking`.

### Transaction safety
- Ensure `@WithTransaction` (or `Panache.withTransaction`) wraps every write to the database.
- When using `@WithSession` for reads, do not embed a write — the session won't have an active transaction.
- Use `persistAndFlush()` only when you need immediate constraint-violation feedback; it is less efficient than normal persist.

### Error handling
- Always handle failures explicitly: use `onFailure().recoverWithItem()`, `retry()`, or propagate deliberately.
- Log failures with structured context (entity id, operation type) to aid production debugging.
- For HTTP endpoints, map domain exceptions to HTTP responses using `@ServerExceptionMapper`.

### Performance
- Prefer `Uni<List<T>>` over streaming rows for most RDBMS use cases (avoids holding connections open).
- Size all connection pools explicitly; defaults are often too low for production workloads.
- Use `@Blocking` only when genuinely necessary — each worker thread consumes OS resources.
- For large datasets, always paginate with `PanacheQuery.page()`; never call `listAll()` on unbounded tables.
- Enable near-cache in Infinispan for read-heavy caches to reduce remote round-trips.

---

## 11. Testing Reactive Code

### Unit tests with `@RunOnVertxContext`

```java
@QuarkusTest
class BookServiceTest {

    @Test
    @RunOnVertxContext
    void testFindByTitle(UniAsserter asserter) {
        asserter.assertNotNull(() -> Book.findByTitle("Rayuela"));
    }
}
```

### Transactional test assertions

```java
@QuarkusTest
class BookRepositoryTest {

    @Test
    @RunOnVertxContext
    void testCreate(TransactionalUniAsserter asserter) {
        Book b = new Book();
        b.title  = "Ficciones";
        b.author = "Borges";

        asserter.execute(() -> b.persist());
        asserter.assertEquals(() -> Book.count(), 1L);
        asserter.execute(() -> Book.deleteAll());
    }
}
```

### Mocking

```java
// Repository pattern: standard Mockito via @InjectMock
@InjectMock BookRepository bookRepository;

Mockito.when(bookRepository.findByTitle("Rayuela"))
    .thenReturn(Uni.createFrom().item(new Book("Rayuela", "Cortázar")));

// Active record pattern: use PanacheMock
asserter.execute(() -> PanacheMock.mock(Book.class));
asserter.execute(() -> Mockito.when(Book.count()).thenReturn(Uni.createFrom().item(42L)));
asserter.assertEquals(() -> Book.count(), 42L);
asserter.execute(() -> PanacheMock.verify(Book.class, Mockito.times(1)).count());
```

---

## 12. Architecture Decision Guide

| Question | Recommendation |
|----------|---------------|
| Simple CRUD, relational DB, moderate concurrency | Hibernate ORM + Panache (blocking, simpler) |
| High-concurrency API or microservice | Hibernate Reactive + Panache + Quarkus REST returning `Uni` |
| Streaming DB results | Reactive SQL client + `Multi` |
| Service-to-service HTTP calls | REST Client with `Uni` return types + Stork for discovery |
| Decoupled internal events | Vert.x Event Bus |
| External event streams (Kafka, AMQP) | SmallRye Reactive Messaging |
| Distributed read-heavy cache | Infinispan with near-cache enabled |
| Need both blocking and reactive in same app | Use `@Blocking` on specific methods; Quarkus handles thread dispatch |
| Native executable target | Avoid unnecessary reflection; prefer annotation-driven config; test early with `-Dnative` |
| Multiple DB in one transaction | Enable XA per datasource: `quarkus.datasource.jdbc.transactions=xa` |

---

## Reference

- Quarkus guides index: https://quarkus.io/guides/
- Mutiny documentation: https://smallrye.io/smallrye-mutiny/
- Hibernate Reactive with Panache: https://quarkus.io/guides/hibernate-reactive-panache
- Reactive SQL clients: https://quarkus.io/guides/reactive-sql-clients
- Datasource configuration: https://quarkus.io/guides/datasource
- SmallRye Stork: https://smallrye.io/smallrye-stork
- Reactive Messaging (Kafka): https://quarkus.io/guides/kafka
- Vert.x event bus: https://quarkus.io/guides/vertx
- Infinispan client: https://quarkus.io/guides/infinispan-client
- Quarkus reactive architecture: https://quarkus.io/guides/quarkus-reactive-architect2;
