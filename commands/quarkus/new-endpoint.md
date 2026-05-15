# /new-endpoint — Generate a Reactive REST Endpoint

Generate a complete Quarkus REST endpoint (formerly RESTEasy Reactive) for an existing entity.
All methods return `Uni<T>` and run on the I/O thread unless explicitly blocked.
Apply `.claude/skills/quarkus-reactive/SKILL.md` throughout.

## Instructions

1. Read `.claude/skills/quarkus-reactive/SKILL.md` before generating any code.
2. Locate the entity and its repository in `src/main/java/`. Read both files.
3. Determine the HTTP operations requested (default: full CRUD if not specified).
4. Generate a resource class following the conventions below.
5. Generate OpenAPI annotations if `quarkus-smallrye-openapi` is in `pom.xml`.
6. Generate a test class using `@QuarkusTest` + `RestAssured`.

## Generated Resource — Conventions

```java
@Path("/api/v1/{resource}")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
@ApplicationScoped
public class {Entity}Resource {

    @Inject {Entity}Repository repository;  // or use entity directly for Active Record

    // GET /api/v1/{resource}?page=0&size=25
    @GET
    public Uni<List<{Entity}>> list(
        @QueryParam("page") @DefaultValue("0") int page,
        @QueryParam("size") @DefaultValue("25") int size) { ... }

    // GET /api/v1/{resource}/{id}
    @GET @Path("/{id}")
    public Uni<Response> getById(@PathParam("id") Long id) { ... }

    // POST /api/v1/{resource}
    @POST
    @WithTransaction
    public Uni<Response> create(@Valid {Entity} entity) { ... }

    // PUT /api/v1/{resource}/{id}
    @PUT @Path("/{id}")
    @WithTransaction
    public Uni<Response> update(@PathParam("id") Long id, @Valid {Entity} updates) { ... }

    // DELETE /api/v1/{resource}/{id}
    @DELETE @Path("/{id}")
    @WithTransaction
    public Uni<Response> delete(@PathParam("id") Long id) { ... }
}
```

## Error Handling

Always generate a `@ServerExceptionMapper` for:
- `NotFoundException` → 404 with a descriptive message
- `ConstraintViolationException` → 400 with field-level validation errors
- `PersistenceException` → 409 for constraint violations

## Arguments

Usage: `/new-endpoint <EntityName> [--ops get,list,create,update,delete] [--path /custom/path] [--no-test]`

Examples:
- `/new-endpoint Book` → full CRUD for Book
- `/new-endpoint Book --ops get,list` → read-only endpoint
- `/new-endpoint Order --path /api/v2/orders --no-test`

## Constraints from Skill

- All methods return `Uni<T>` or `Uni<Response>` — never a raw `T` unless justified with `@Blocking`
- `list()` must always paginate — never return unbounded results
- POST returns `201 Created` with `Location` header pointing to the new resource URI
- PUT returns `200 OK` with the updated entity, or `404` if not found
- DELETE returns `204 No Content` on success, or `404` if not found
- `@WithTransaction` on every write method — never omit it
