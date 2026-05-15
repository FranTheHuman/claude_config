# /new-entity — Generate a Panache Entity

Generate a complete Hibernate Reactive + Panache entity with its repository, common queries,
and migration SQL. Apply `.claude/skills/quarkus-reactive/SKILL.md` throughout.

## Instructions

1. Read `.claude/skills/quarkus-reactive/SKILL.md` before generating any code.
2. Parse the arguments to extract: entity name, fields, relationships, and pattern preference.
3. If the pattern is not specified, **default to Repository** (better separation of concerns for new code).
4. Scan `src/main/java/` to match the existing package structure and naming conventions.
5. Scan `src/main/resources/application.properties` to confirm the datasource kind (reactive or JDBC).
6. Generate all files listed in the Output section.

## Output Files

For an entity named `Book` in package `org.acme.catalog`:

### 1. `Book.java` — the entity
- `@Entity`, fields as public (Active Record) or private with getters (Repository)
- `@Column` constraints where specified
- Relationships with correct fetch strategy for reactive (lazy by default)
- If Active Record: include the 5 most useful static query methods for the domain

### 2. `BookRepository.java` — repository (if Repository pattern)
- `implements PanacheRepository<Book>`
- `@ApplicationScoped`
- Include: findBy[mainField], findAll with Sort, count, at least one paginated query

### 3. `V{n}__create_book.sql` — Flyway migration (if Flyway is in pom.xml)
- CREATE TABLE with correct column types
- Constraints matching the entity annotations
- Sequence if using `@GeneratedValue(SEQUENCE)`

### 4. Brief usage example
Show how to inject and use the repository in a service or endpoint.

## Arguments

Usage: `/new-entity <EntityName> [field:type ...] [--pattern active|repository] [--package org.acme.x]`

Examples:
- `/new-entity Book title:String author:String published:LocalDate`
- `/new-entity Order --pattern active --package org.acme.sales`
- `/new-entity Product name:String price:BigDecimal category:String stock:int --pattern repository`

## Constraints from Skill

- Import `io.quarkus.hibernate.reactive.panache.*` — never the ORM variant
- Every generated write method must be wrapped in `@WithTransaction` or `Panache.withTransaction()`
- Every generated read method must be annotated `@WithSession`
- Do NOT generate `listAll()` without a page limit warning comment
- Generated IDs use `PanacheEntity` (auto Long ID) unless the user specifies otherwise
