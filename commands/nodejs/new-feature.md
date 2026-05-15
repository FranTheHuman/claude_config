# /nodejs:new-feature — Generate a Complete Node.js Feature

Generate all four layers of a new feature: routes, controller, service, and repository.
Always generates them together — never a single layer in isolation.
Apply all rules from `.claude/skills/nodejs-best-practices/SKILL.md`.

## Instructions

1. Read `.claude/skills/nodejs-best-practices/SKILL.md` before generating any code.
2. Parse the arguments: feature name, fields/domain, HTTP operations needed.
3. Scan `src/` for existing package structure and naming conventions.
4. Generate all files listed below — always the full set, never partial.
5. Register the new router in `app.ts` with a comment showing where to add it.

## Output Files

For a feature named `products` with fields `name`, `price`, `category`:

### 1. `src/products/products.validation.ts`
- Zod schema for create, update (partial), and ID param
- Exported TypeScript types inferred from schemas (`CreateProductDto`, etc.)

### 2. `src/products/products.repository.ts`
- DB query methods only: `findById`, `findAll`, `create`, `update`, `delete`
- Uses parameterized queries or ORM — never raw string interpolation
- Returns typed domain objects

### 3. `src/products/products.service.ts`
- Business logic: existence checks, validation rules, orchestration
- Throws typed `AppError` subclasses (`NotFoundError`, `ConflictError`, etc.)
- Injects repository via constructor

### 4. `src/products/products.controller.ts`
- Parses request, calls service, returns response
- Always uses `next(error)` — never inline catch responses
- Returns correct HTTP status codes: 201 for create, 200 for update, 204 for delete

### 5. `src/products/products.routes.ts`
- Thin router: registers paths + middleware only
- Applies `validateBody()` middleware for POST and PUT
- Applies `requireAuth` middleware where specified

### 6. `src/products/products.service.test.ts`
- Jest unit tests for service layer
- Mocks repository with `jest.mock()`
- Covers: happy path, not found, conflict, validation failure

## Arguments

Usage: `/nodejs:new-feature <name> [field:type ...] [--ops get,list,create,update,delete]
                              [--auth] [--no-test]`

Examples:
- `/nodejs:new-feature products name:string price:number category:string`
- `/nodejs:new-feature orders --ops create,list,get --auth`
- `/nodejs:new-feature categories name:string --ops get,list,create --no-test`

## Rules from Skill

- All methods in service and repository are `async` — no sync alternatives
- Repository never imports from service — dependency flows one way only
- Controller never accesses `req.params.id` as a number — validate with Zod first
- Passwords and sensitive fields are never returned in responses
- `findAll` always supports pagination — never returns unbounded results
