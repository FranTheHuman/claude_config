---
name: nodejs-best-practices
description: >
  Expert knowledge of Node.js application architecture, best practices, and production
  patterns. Covers project structure, layered architecture, async/await, error handling,
  input validation, security, event loop management, clustering, logging, testing,
  containerization, caching, TypeScript, OpenAPI, dependency injection, and deployment.
  Use this skill whenever the task involves writing, reviewing, debugging, or explaining
  Node.js server-side code or architecture decisions.
---

You are an expert Node.js architect and developer. Apply this skill as the authoritative
source of truth for every Node.js decision. Prioritize scalability, security,
maintainability, and production readiness from the very first line of code.

---

## 1. Understand the Context First

Before writing any code, determine:

- **Application size**: Small API, monolith, or microservice? This dictates structure depth.
- **Async model**: Is existing code using callbacks, promises, or mixed patterns? Normalize first.
- **Layer violations**: Is business logic in route handlers? Is database logic in controllers?
- **Security posture**: Are secrets hardcoded? Is validation missing? Is auth enforced?
- **Production readiness**: Is there logging, error handling, clustering, and health checks?
- **TypeScript**: New projects should default to TypeScript unless explicitly told otherwise.

---

## 2. Project Structure — Feature-Based

Always organize by business domain (feature), not by technical layer.
Never create flat `/controllers`, `/models`, `/routes` folders at the root level.

### Correct structure

```
src/
├── users/
│   ├── users.routes.ts
│   ├── users.controller.ts
│   ├── users.service.ts
│   ├── users.validation.ts
│   └── users.repository.ts
├── orders/
│   ├── orders.routes.ts
│   ├── orders.controller.ts
│   ├── orders.service.ts
│   └── orders.repository.ts
├── payments/
│   ├── payments.routes.ts
│   ├── payments.service.ts
│   └── payments.gateway.ts
├── shared/
│   ├── database.ts
│   ├── logger.ts
│   ├── config.ts
│   ├── errors.ts
│   └── middleware/
│       ├── auth.middleware.ts
│       ├── error.middleware.ts
│       └── rate-limit.middleware.ts
├── scripts/              ← dev scripts, migrations, seeds (never imported by app)
├── app.ts                ← Express app setup, middleware registration
└── server.ts             ← HTTP server bootstrap, clustering
```

### Rules

- Each feature folder owns its routes, controller, service, validation, and repository.
- `shared/` contains only utilities used by two or more features.
- `scripts/` is isolated from application code — never imported by `app.ts`.
- Root level: only `app.ts`, `server.ts`, `package.json`, config files.
- When a feature grows large, split into sub-features within its folder.

---

## 3. Layered Architecture

Every feature follows the same layer contract. Never skip or collapse layers.

### Layer responsibilities

| Layer | File suffix | Responsibilities | Never does |
|-------|-------------|-----------------|-----------|
| Routes | `.routes.ts` | Register endpoints, apply middleware | Business logic, DB queries |
| Controller | `.controller.ts` | Parse request, call service, return response | Business logic, DB queries |
| Service | `.service.ts` | Business rules, workflows, orchestration | HTTP response handling |
| Repository | `.repository.ts` | Database queries, ORM interactions | Business logic, validation |
| Validation | `.validation.ts` | Schema definitions (Joi/Zod) | Request handling, DB access |

### Example — correct layering

```typescript
// users.routes.ts — thin, registers path + middleware only
import { Router } from 'express';
import { UserController } from './users.controller';
import { validateBody } from '../shared/middleware/validate.middleware';
import { createUserSchema } from './users.validation';

const router = Router();
router.post('/', validateBody(createUserSchema), UserController.create);
router.get('/:id', UserController.getById);
export default router;

// users.controller.ts — parses request, delegates to service
export class UserController {
    static async create(req: Request, res: Response, next: NextFunction) {
        try {
            const user = await UserService.create(req.body);
            res.status(201).json(user);
        } catch (error) {
            next(error); // always forward to centralized error handler
        }
    }

    static async getById(req: Request, res: Response, next: NextFunction) {
        try {
            const user = await UserService.findById(req.params.id);
            if (!user) return res.status(404).json({ message: 'User not found' });
            res.json(user);
        } catch (error) {
            next(error);
        }
    }
}

// users.service.ts — business logic only
export class UserService {
    static async create(data: CreateUserDto): Promise<User> {
        const existing = await UserRepository.findByEmail(data.email);
        if (existing) throw new ConflictError('Email already registered');
        const hashed = await bcrypt.hash(data.password, 12);
        return UserRepository.create({ ...data, password: hashed });
    }

    static async findById(id: string): Promise<User | null> {
        return UserRepository.findById(id);
    }
}

// users.repository.ts — database queries only
export class UserRepository {
    static async findByEmail(email: string): Promise<User | null> {
        return db('users').where({ email }).first();
    }

    static async create(data: CreateUserDto): Promise<User> {
        const [user] = await db('users').insert(data).returning('*');
        return user;
    }
}
```

---

## 4. Async/Await — Rules and Patterns

Use `async/await` exclusively. Never mix callbacks, raw promises, and async/await in the
same flow.

### Core rules

```typescript
// CORRECT — consistent async/await with try/catch
async function getUserProfile(req: Request, res: Response): Promise<void> {
    try {
        const user = await userService.findById(req.params.id);
        if (!user) {
            res.status(404).json({ message: 'User not found' });
            return;
        }
        res.json(user);
    } catch (error) {
        res.status(500).json({ message: 'Could not load user profile' });
    }
}

// WRONG — mixing patterns
function getUserProfile(req, res) {
    userService.findById(req.params.id)
        .then(user => {
            res.json(user);
        })
        .catch(err => res.status(500).json({ message: err.message }));
}
```

### Parallel async operations

```typescript
// Run independent async operations in parallel
const [user, orders, preferences] = await Promise.all([
    UserService.findById(userId),
    OrderService.findByUser(userId),
    PreferenceService.findByUser(userId),
]);

// Use Promise.allSettled when partial failure is acceptable
const results = await Promise.allSettled([
    notificationService.sendEmail(user),
    analyticsService.track(event),
]);
```

### Converting callbacks to promises

```typescript
// Promisify old callback-based APIs before using with await
import { promisify } from 'util';
const readFile = promisify(fs.readFile);
const content = await readFile('./config.json', 'utf-8');
```

---

## 5. Centralized Error Handling

Never return error responses from controllers or services directly.
Always forward errors to centralized Express error middleware via `next(error)`.

### Custom error classes

```typescript
// shared/errors.ts
export class AppError extends Error {
    constructor(
        public message: string,
        public statusCode: number,
        public isOperational = true
    ) {
        super(message);
        this.name = this.constructor.name;
        Error.captureStackTrace(this, this.constructor);
    }
}

export class NotFoundError extends AppError {
    constructor(resource = 'Resource') {
        super(`${resource} not found`, 404);
    }
}

export class ConflictError extends AppError {
    constructor(message: string) {
        super(message, 409);
    }
}

export class ValidationError extends AppError {
    constructor(message: string) {
        super(message, 400);
    }
}

export class UnauthorizedError extends AppError {
    constructor(message = 'Unauthorized') {
        super(message, 401);
    }
}
```

### Centralized error middleware

```typescript
// shared/middleware/error.middleware.ts
import { Request, Response, NextFunction } from 'express';
import { AppError } from '../errors';
import { logger } from '../logger';

export function errorHandler(
    error: Error,
    req: Request,
    res: Response,
    next: NextFunction
): void {
    // Operational errors — expected, return clean response
    if (error instanceof AppError && error.isOperational) {
        logger.warn({
            message: error.message,
            statusCode: error.statusCode,
            path: req.path,
            method: req.method,
        });
        res.status(error.statusCode).json({
            status: 'error',
            message: error.message,
        });
        return;
    }

    // Programmer errors — unexpected, log full details
    logger.error({
        message: error.message,
        stack: error.stack,
        path: req.path,
        method: req.method,
    });

    res.status(500).json({
        status: 'error',
        message: 'Internal server error',
    });
}

// app.ts — register LAST, after all routes
app.use(errorHandler);
```

---

## 6. Input Validation and Sanitization

Validate ALL incoming data before it reaches business logic — body, params, query, headers.

### Schema validation with Zod (preferred for TypeScript)

```typescript
// users.validation.ts
import { z } from 'zod';

export const createUserSchema = z.object({
    email: z.string().email('Invalid email format'),
    password: z.string()
        .min(8, 'Password must be at least 8 characters')
        .regex(/[A-Z]/, 'Must contain uppercase')
        .regex(/[0-9]/, 'Must contain a number'),
    name: z.string().min(2).max(100).trim(),
    age: z.number().int().min(18).max(120).optional(),
});

export const updateUserSchema = createUserSchema.partial().omit({ email: true });
export type CreateUserDto = z.infer<typeof createUserSchema>;
export type UpdateUserDto = z.infer<typeof updateUserSchema>;

// Validation middleware
export function validateBody<T>(schema: z.ZodSchema<T>) {
    return (req: Request, res: Response, next: NextFunction) => {
        const result = schema.safeParse(req.body);
        if (!result.success) {
            return res.status(400).json({
                status: 'error',
                errors: result.error.flatten().fieldErrors,
            });
        }
        req.body = result.data; // sanitized and typed
        next();
    };
}
```

### Validation checklist

- Validate `req.body`, `req.params`, `req.query` separately.
- Trim string fields automatically in schema definitions.
- Use `z.string().email()`, `.url()`, `.uuid()` for format validation.
- Block unexpected fields with `.strict()` when the schema should be exact.
- Validate numeric IDs: `z.string().uuid()` or `z.coerce.number().int().positive()`.

---

## 7. Security

Apply security in layers — middleware, auth, secrets, and process isolation.

### Helmet and rate limiting

```typescript
import helmet from 'helmet';
import rateLimit from 'express-rate-limit';

// app.ts
app.use(helmet()); // sets secure HTTP headers automatically

// Rate limit for auth routes
const authLimiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 10,
    message: { status: 'error', message: 'Too many attempts, try again later' },
    standardHeaders: true,
    legacyHeaders: false,
});
app.use('/api/auth', authLimiter);

// Global rate limit for all routes
const globalLimiter = rateLimit({
    windowMs: 60 * 1000,
    max: 100,
});
app.use(globalLimiter);
```

### JWT authentication

```typescript
// shared/middleware/auth.middleware.ts
import jwt from 'jsonwebtoken';
import { UnauthorizedError } from '../errors';

export function requireAuth(req: Request, res: Response, next: NextFunction) {
    const token = req.headers.authorization?.split(' ')[1];
    if (!token) throw new UnauthorizedError('No token provided');

    try {
        const payload = jwt.verify(token, process.env.JWT_SECRET!);
        req.user = payload as JwtPayload;
        next();
    } catch {
        throw new UnauthorizedError('Invalid or expired token');
    }
}

export function requireRole(...roles: string[]) {
    return (req: Request, res: Response, next: NextFunction) => {
        if (!roles.includes(req.user?.role)) {
            throw new UnauthorizedError('Insufficient permissions');
        }
        next();
    };
}

// Usage in routes
router.delete('/:id', requireAuth, requireRole('admin'), UserController.delete);
```

### Environment variables

```typescript
// shared/config.ts — validate on startup, fail early
import { z } from 'zod';

const envSchema = z.object({
    NODE_ENV: z.enum(['development', 'staging', 'production']),
    PORT: z.coerce.number().default(3000),
    DATABASE_URL: z.string().url(),
    JWT_SECRET: z.string().min(32),
    REDIS_URL: z.string().url().optional(),
});

const parsed = envSchema.safeParse(process.env);
if (!parsed.success) {
    console.error('Invalid environment variables:', parsed.error.flatten());
    process.exit(1);
}

export const config = parsed.data;
```

### Security checklist

- Never hardcode secrets — always use `process.env`.
- Commit `.env.example`, never `.env`.
- Add `.env` to `.gitignore`.
- Run Node.js processes as non-root in Docker.
- Use `bcrypt` with cost factor ≥ 12 for password hashing.
- Set `httpOnly`, `secure`, `sameSite` on cookies.
- Sanitize input against SQL injection (use parameterized queries) and XSS.

---

## 8. Event Loop — Never Block It

The event loop processes one task at a time. A blocked event loop stalls ALL requests.

### What blocks the event loop

```typescript
// BLOCKS — never use in request handlers
fs.readFileSync('./large-file.json');       // synchronous file I/O
JSON.parse(veryLargeJsonString);            // large CPU task on main thread
crypto.pbkdf2Sync(password, salt, ...);    // synchronous crypto
for (let i = 0; i < 1_000_000_000; i++) {} // heavy loop
```

### Correct alternatives

```typescript
// Non-blocking file I/O
const content = await fs.promises.readFile('./config.json', 'utf-8');

// CPU-intensive work → Worker Threads
import { Worker } from 'worker_threads';
function runImageProcessing(data: Buffer): Promise<Buffer> {
    return new Promise((resolve, reject) => {
        const worker = new Worker('./workers/image-processor.js', { workerData: data });
        worker.on('message', resolve);
        worker.on('error', reject);
    });
}

// Heavy background jobs → BullMQ queue
import { Queue } from 'bullmq';
const reportQueue = new Queue('reports', { connection: redisConnection });

// Add job to queue — returns immediately, processed in background
await reportQueue.add('generate-report', { userId, filters });
```

### Caching to reduce repeated work

```typescript
// Redis cache — shared across all Node.js instances
import { createClient } from 'redis';
const redis = createClient({ url: config.REDIS_URL });

async function getCachedOrFetch<T>(
    key: string,
    ttlSeconds: number,
    fetcher: () => Promise<T>
): Promise<T> {
    const cached = await redis.get(key);
    if (cached) return JSON.parse(cached);

    const data = await fetcher();
    await redis.setEx(key, ttlSeconds, JSON.stringify(data));
    return data;
}

// Usage
const products = await getCachedOrFetch(
    'products:featured',
    300, // 5 minutes
    () => ProductRepository.findFeatured()
);
```

---

## 9. Clustering and Process Management

Node.js is single-threaded — one process uses one CPU core. Use clustering to utilize all cores.

```typescript
// server.ts — clustering setup
import cluster from 'cluster';
import os from 'os';
import { createServer } from './app';

if (cluster.isPrimary) {
    const numCPUs = os.cpus().length;
    console.log(`Master ${process.pid} running — forking ${numCPUs} workers`);

    for (let i = 0; i < numCPUs; i++) {
        cluster.fork();
    }

    cluster.on('exit', (worker, code, signal) => {
        console.log(`Worker ${worker.process.pid} died (${signal || code}). Restarting...`);
        cluster.fork(); // auto-restart crashed workers
    });
} else {
    const app = createServer();
    app.listen(config.PORT, () => {
        console.log(`Worker ${process.pid} listening on port ${config.PORT}`);
    });
}
```

### PM2 ecosystem config

```javascript
// ecosystem.config.js
module.exports = {
    apps: [{
        name: 'api',
        script: 'dist/server.js',
        instances: 'max',        // one per CPU core
        exec_mode: 'cluster',
        watch: false,
        max_memory_restart: '500M',
        env_production: {
            NODE_ENV: 'production',
        },
        error_file: 'logs/err.log',
        out_file: 'logs/out.log',
    }],
};
```

---

## 10. Structured Logging

Replace ALL `console.log()` with structured logging. Never use raw console in production code.

```typescript
// shared/logger.ts
import pino from 'pino';
import { config } from './config';

export const logger = pino({
    level: config.NODE_ENV === 'production' ? 'info' : 'debug',
    transport: config.NODE_ENV !== 'production'
        ? { target: 'pino-pretty', options: { colorize: true } }
        : undefined,
    base: { service: 'api', version: process.env.npm_package_version },
    redact: ['req.headers.authorization', 'body.password'], // never log secrets
});

// Request logging middleware
export function requestLogger(req: Request, res: Response, next: NextFunction) {
    const start = Date.now();
    res.on('finish', () => {
        logger.info({
            method: req.method,
            path: req.path,
            statusCode: res.statusCode,
            durationMs: Date.now() - start,
            requestId: req.headers['x-request-id'],
        });
    });
    next();
}
```

### Log levels guide

| Level | When to use |
|-------|------------|
| `trace` | Fine-grained debug (disabled in prod) |
| `debug` | Development diagnostics |
| `info` | Successful operations, startup, business events |
| `warn` | Slow queries, degraded behavior, deprecated usage |
| `error` | Failed operations, unhandled exceptions |
| `fatal` | Application crash, unrecoverable state |

---

## 11. Testing

Test pyramid: many unit tests, some integration tests, few end-to-end tests.

### Unit test — service with mocked repository

```typescript
// users.service.test.ts
import { UserService } from './users.service';
import { UserRepository } from './users.repository';

jest.mock('./users.repository');
const MockedUserRepository = jest.mocked(UserRepository);

describe('UserService.create', () => {
    beforeEach(() => jest.clearAllMocks());

    it('should throw ConflictError when email already exists', async () => {
        MockedUserRepository.findByEmail.mockResolvedValue({ id: '1', email: 'a@b.com' });

        await expect(
            UserService.create({ email: 'a@b.com', password: 'Password1!', name: 'Alice' })
        ).rejects.toThrow('Email already registered');
    });

    it('should create user with hashed password', async () => {
        MockedUserRepository.findByEmail.mockResolvedValue(null);
        MockedUserRepository.create.mockResolvedValue({ id: '2', email: 'b@c.com', name: 'Bob' });

        const user = await UserService.create({
            email: 'b@c.com', password: 'Password1!', name: 'Bob'
        });

        expect(user.email).toBe('b@c.com');
        const [createArgs] = MockedUserRepository.create.mock.calls[0];
        expect(createArgs.password).not.toBe('Password1!'); // must be hashed
    });
});
```

### Integration test — Express endpoint

```typescript
// users.routes.test.ts
import request from 'supertest';
import { createApp } from '../app';
import { db } from '../shared/database';

const app = createApp();

describe('POST /api/users', () => {
    afterEach(async () => db('users').truncate());

    it('should return 201 with created user', async () => {
        const response = await request(app)
            .post('/api/users')
            .send({ email: 'test@example.com', password: 'Password1!', name: 'Test' })
            .expect(201);

        expect(response.body).toMatchObject({ email: 'test@example.com', name: 'Test' });
        expect(response.body.password).toBeUndefined(); // never return password
    });

    it('should return 409 when email already exists', async () => {
        await request(app).post('/api/users')
            .send({ email: 'test@example.com', password: 'Password1!', name: 'Test' });

        await request(app).post('/api/users')
            .send({ email: 'test@example.com', password: 'Password1!', name: 'Test2' })
            .expect(409);
    });

    it('should return 400 with validation errors for invalid input', async () => {
        const response = await request(app)
            .post('/api/users')
            .send({ email: 'not-an-email', password: '123' })
            .expect(400);

        expect(response.body.errors).toBeDefined();
    });
});
```

### Testing checklist

- Mock external dependencies (DB, Redis, external APIs) in unit tests.
- Use a real test database (separate from dev/prod) in integration tests.
- Never use `process.env` directly — inject via `config` module (easier to mock).
- Run tests in CI on every PR before merge.
- Target 80%+ coverage on services (business logic layer).

---

## 12. TypeScript Configuration

All new Node.js projects default to TypeScript unless explicitly stated otherwise.

```json
// tsconfig.json
{
    "compilerOptions": {
        "target": "ES2022",
        "module": "commonjs",
        "lib": ["ES2022"],
        "outDir": "dist",
        "rootDir": "src",
        "strict": true,
        "noUnusedLocals": true,
        "noUnusedParameters": true,
        "noImplicitReturns": true,
        "esModuleInterop": true,
        "skipLibCheck": true,
        "forceConsistentCasingInFileNames": true,
        "resolveJsonModule": true,
        "declaration": true,
        "sourceMap": true
    },
    "include": ["src"],
    "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

### package.json scripts

```json
{
    "scripts": {
        "build": "tsc",
        "start": "node dist/server.js",
        "dev": "ts-node-dev --respawn --transpile-only src/server.ts",
        "lint": "eslint src --ext .ts",
        "test": "jest --coverage",
        "test:watch": "jest --watch",
        "typecheck": "tsc --noEmit"
    }
}
```

---

## 13. Containerization

```dockerfile
# Dockerfile — multi-stage build
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22-alpine AS production
WORKDIR /app

# Non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force
COPY --from=builder /app/dist ./dist

USER appuser
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD node -e "require('http').get('http://localhost:3000/health', r => r.statusCode === 200 ? process.exit(0) : process.exit(1))"

CMD ["node", "dist/server.js"]
```

```
# .dockerignore
node_modules
dist
.env
*.test.ts
coverage
.git
```

---

## 14. API Documentation — OpenAPI / Swagger

Define the API contract explicitly. Self-documenting APIs reduce integration friction.

```typescript
// Inline JSDoc for swagger-jsdoc
/**
 * @openapi
 * /api/users:
 *   post:
 *     summary: Create a new user
 *     tags: [Users]
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             $ref: '#/components/schemas/CreateUserRequest'
 *     responses:
 *       201:
 *         description: User created successfully
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/UserResponse'
 *       400:
 *         $ref: '#/components/responses/ValidationError'
 *       409:
 *         $ref: '#/components/responses/ConflictError'
 */
```

---

## 15. Dependency Injection

Use constructor injection for testability. Avoid module-level singletons in service classes.

```typescript
// Without DI — hard to test (tightly coupled to real DB)
export class UserService {
    async findById(id: string) {
        return db('users').where({ id }).first(); // direct DB access, untestable
    }
}

// With DI — injectable, testable
export class UserService {
    constructor(private readonly userRepository: UserRepository) {}

    async findById(id: string) {
        return this.userRepository.findById(id);
    }
}

// Wiring in app.ts (manual DI — sufficient for most projects)
const userRepository = new UserRepository(db);
const userService = new UserService(userRepository);
const userController = new UserController(userService);

// For larger projects, use tsyringe or InversifyJS
```

---

## 16. Code Quality — ESLint

```json
// .eslintrc.json
{
    "extends": [
        "eslint:recommended",
        "@typescript-eslint/recommended",
        "@typescript-eslint/recommended-requiring-type-checking"
    ],
    "plugins": ["@typescript-eslint", "security"],
    "rules": {
        "no-console": "error",
        "@typescript-eslint/no-explicit-any": "error",
        "@typescript-eslint/no-floating-promises": "error",
        "@typescript-eslint/await-thenable": "error",
        "security/detect-sql-injection": "warn",
        "security/detect-object-injection": "warn",
        "@typescript-eslint/explicit-function-return-type": "warn"
    }
}
```

Run `no-console: error` so raw `console.log` never makes it to production.

---

## 17. Architecture Decision Guide

| Question | Answer |
|----------|--------|
| Where does business logic go? | `service` layer only — never in controllers or routes |
| Where do DB queries go? | `repository` layer only — never in services directly |
| How to handle errors? | Throw `AppError` subclasses, forward with `next(error)` |
| How to validate input? | Zod schemas in `.validation.ts`, applied via middleware |
| How to manage secrets? | `process.env` only, validated at startup with Zod |
| New project: JS or TS? | TypeScript by default |
| How to handle CPU work? | Worker threads or BullMQ background queue |
| How to cache? | Redis with TTL via `getCachedOrFetch` helper |
| How to scale horizontally? | Clustering + PM2 + Nginx reverse proxy |
| How to log? | Pino structured JSON — never `console.log` |
| How to test business logic? | Jest unit tests with mocked repositories |
| How to test endpoints? | Supertest integration tests against real Express app |
| How to document APIs? | OpenAPI/Swagger via swagger-jsdoc |
| How to inject dependencies? | Constructor injection, manual wiring in `app.ts` |
| How to deploy containers? | Docker multi-stage builds, non-root user, health checks |
| Missing env var on startup? | Fail fast — `process.exit(1)` immediately |

---

## 18. Common Anti-Patterns — Never Generate These

| Anti-pattern | Correct approach |
|-------------|-----------------|
| `fs.readFileSync()` in a route handler | `await fs.promises.readFile()` |
| Business logic inside `router.get(...)` | Move to service layer |
| `console.log()` in production code | `logger.info()` with Pino |
| Hardcoded secrets in source code | `process.env` with Zod validation |
| Missing `await` on async calls | Enable `@typescript-eslint/no-floating-promises` |
| Generic `catch (e) { res.status(500) }` everywhere | Centralized `errorHandler` middleware |
| Fat controllers with DB queries | Repository layer for all DB access |
| No input validation on route params | Validate `req.params`, `req.query`, `req.body` |
| Synchronous crypto in request handler | `await bcrypt.hash()` async variant |
| Single process serving all traffic | Clustering or PM2 cluster mode |
| Root user in Docker container | `adduser` + `USER appuser` in Dockerfile |
| Untyped `any` in TypeScript | Explicit types or `unknown` with guards |

---

## Reference

- Node.js best practices: https://github.com/goldbergyoni/nodebestpractices
- Express security: https://expressjs.com/en/advanced/best-practice-security.html
- Zod documentation: https://zod.dev
- Pino logging: https://getpino.io
- BullMQ queues: https://docs.bullmq.io
- PM2 process manager: https://pm2.keymetrics.io
- Jest testing: https://jestjs.io
