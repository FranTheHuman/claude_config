# /nodejs:explain — Explain a Node.js Concept or Code

Explain a Node.js concept, pattern, API, or specific file from this project.
Always include concrete TypeScript examples and a "when to use / when NOT to use" section.

## Instructions

1. Read `.claude/skills/nodejs-best-practices/SKILL.md` before answering.
2. Parse the argument: concept, pattern, API method, or file path.
3. If a file path is given, read the file and explain THAT specific code in context.
4. Structure the answer as below.
5. Never skip the "When NOT to use" — it is often the most valuable guidance.

## Output Structure

### What it is
One clear paragraph. Define any jargon used.

### How it works
3–5 bullet points on the runtime behavior or architectural role.

### Code example — minimal
Simplest possible working TypeScript example.

### Code example — realistic
Closer to what this project would actually need.

### Common mistakes
The 2–3 most frequent errors. Format: ❌ Wrong → ✅ Correct

### When to use / When NOT to use
Concise decision table.

## Arguments

Usage: `/nodejs:explain <concept|path>`

Examples:
- `/nodejs:explain async/await`
- `/nodejs:explain centralized error handling`
- `/nodejs:explain repository pattern`
- `/nodejs:explain event loop`
- `/nodejs:explain clustering vs worker threads`
- `/nodejs:explain Zod vs Joi`
- `/nodejs:explain dependency injection`
- `/nodejs:explain src/users/users.service.ts`
- `/nodejs:explain why console.log is bad in production`
- `/nodejs:explain Redis caching strategy`

## Concepts Fully Covered by the Skill

`async/await`, `Promise.all`, `Promise.allSettled`, `try/catch`, `next(error)`,
`AppError`, `centralized error handler`, `Zod schema`, `validateBody middleware`,
`Helmet`, `rate limiting`, `JWT auth`, `requireAuth`, `requireRole`, `bcrypt`,
`event loop`, `Worker Threads`, `BullMQ`, `Redis cache`, `clustering`, `PM2`,
`Pino`, `structured logging`, `repository pattern`, `service layer`, `controller`,
`dependency injection`, `constructor injection`, `Jest`, `supertest`, `TypeScript strict`,
`tsconfig.json`, `Docker multi-stage`, `OpenAPI`, `Swagger`, `ESLint security plugin`
