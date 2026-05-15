# /nodejs:scaffold — Scaffold a New Node.js Project

Generate the complete base structure for a new production-ready Node.js project.
Includes all shared infrastructure, configuration, and one example feature.
Apply all rules from `.claude/skills/nodejs-best-practices/SKILL.md`.

## Instructions

1. Read `.claude/skills/nodejs-best-practices/SKILL.md` before generating.
2. Parse the arguments: project name, first feature name, database choice.
3. Generate every file listed below — complete, not placeholder stubs.
4. All files are TypeScript unless `--js` flag is explicitly passed.

## Output Files

### Configuration
- `tsconfig.json` — strict mode, ES2022, outDir dist
- `package.json` — scripts: build, start, dev, lint, test, typecheck
- `.eslintrc.json` — TypeScript + security plugins, `no-console: error`
- `.env.example` — all required env vars with placeholder values
- `.gitignore` — node_modules, dist, .env, coverage, logs
- `.dockerignore` — node_modules, dist, .env, test files

### Application core
- `src/app.ts` — Express setup, middleware registration, router mounting
- `src/server.ts` — HTTP server bootstrap with clustering
- `src/shared/config.ts` — Zod env validation, `process.exit(1)` on failure
- `src/shared/errors.ts` — `AppError`, `NotFoundError`, `ConflictError`,
  `ValidationError`, `UnauthorizedError`
- `src/shared/logger.ts` — Pino structured logger with request logging middleware
- `src/shared/database.ts` — DB connection setup (Knex or Prisma based on flag)
- `src/shared/middleware/error.middleware.ts` — centralized error handler
- `src/shared/middleware/validate.middleware.ts` — Zod body/params/query middleware
- `src/shared/middleware/auth.middleware.ts` — JWT `requireAuth` + `requireRole`

### Docker
- `Dockerfile` — multi-stage build, non-root user, healthcheck
- `docker-compose.yml` — app + database + redis services

### First feature (example)
- Full 4-layer scaffold for the specified first feature name
- With Jest test skeleton

### Health check
- `GET /health` endpoint returning `{ status: "ok", uptime, timestamp }`

## Arguments

Usage: `/nodejs:scaffold <project-name> [--feature <name>] [--db knex|prisma]
                          [--auth jwt|session] [--js]`

Examples:
- `/nodejs:scaffold ecommerce-api --feature products --db prisma`
- `/nodejs:scaffold task-manager --feature tasks --db knex --auth jwt`
- `/nodejs:scaffold reporting-service --feature reports --db prisma --js`
