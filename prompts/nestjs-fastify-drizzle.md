# Stack — NestJS + Fastify + Drizzle (PostgreSQL) + Redis (BullMQ) + pino + JWT (Tikiti)

Review prompt for Tikiti's NestJS-on-Fastify backend repositories. Apply these conventions in addition to the base review areas. Repo-specific deviations (e.g. a service's specific table list, environment-tuned thresholds) come from the consumer's `extra_prompt` / `extra_prompt_path` input and render under the "Repo-specific notes" section.

## Architecture & Patterns

- **Controllers are thin.** They map HTTP shape only and delegate to services. No business logic in the controller layer.
- **Services are constructor-injectable and unit-testable.** Avoid `new` inside services; let the DI container compose dependencies.
- **DTOs use `class-validator` + `class-transformer`** decorators for every request body, query string, and route param shape. Never read `req.body` directly past the controller signature.
- **Guards for auth, Interceptors for transforms, Pipes for validation** — keep the responsibilities separated; don't smuggle business logic into a Pipe.
- **Don't re-import `@Global()` modules.** Globals (`DatabaseModule`, `AppLoggerModule`, `AppCacheModule`, `MonitoringModule`, `EventModule`, `JwtModule`) are already available everywhere via the root module.
- **Feature modules** follow a consistent layout: `<feature>.module.ts`, `<feature>.controller.ts`, `<feature>.service.ts`, optional `<feature>.repository.ts`, a `dto/` subdirectory, and a `<feature>.constants.ts` for magic values.
- **Throw typed `AppException` subclasses from `@app/common`** (`NotFoundException`, `ConflictException`, `BadRequestException`, `UnauthorizedException`, `ForbiddenException`, …). **Never raw `HttpException`, never raw `Error`.** The global filter relies on the `AppException` shape for error-code mapping in the response envelope.
- **Don't manually construct the response envelope.** The global `ResponseInterceptor` wraps every response in `{ status, data, meta: { requestId, responseTimeMs }, error }`. Use `@SkipEnvelope()` to opt out (webhooks, raw downloads, external-callback shapes).
- **Read the authenticated user via `@CurrentUser()` / `@OptionalCurrentUser()`** — never `request.user` directly. The decorators centralise null-handling and type narrowing.
- **Pagination returns `{ items, _meta: { total, page, limit } }`** from services so the response interceptor merges `_meta` consistently across endpoints.
- **External integrations follow the orchestrator/provider/strategy pattern** under `libs/common/src/integrations/<domain>/` with three subdirectories — `interfaces/` (the shared contract), `orchestrators/` (the only thing app code injects), `providers/` (interchangeable adapters). App code **always** injects the orchestrator, never a provider adapter directly. Flag any controller / service that imports from `integrations/<domain>/providers/` — that's a layer-break.
- **`@AuditLog()` decorator + `AuditInterceptor`** for sensitive-mutation audit emission. Flag any write to financial or user-PII tables that lacks `@AuditLog()`.

## Guards

The full Tikiti guard list — flag any non-public endpoint missing at least one of these:

- `JwtAuthGuard` — required auth, rejects unauthenticated
- `OptionalJwtAuthGuard` — auth optional, sets user if present
- `ProducerVerifiedGuard` — gates producer-only endpoints; requires verified producer profile
- `AdminRoleGuard` — gates platform-admin endpoints; requires admin role claim on the JWT

## Security

- **Auth guards on every non-public endpoint.** Any `@Public()`-style escape hatch must be justified inline.
- **Input validation via DTOs + `class-validator` only** — never `req.body` directly.
- **Drizzle ORM for all queries** — no raw SQL string concatenation. If raw SQL is unavoidable, parameterise via the driver's `sql` template tag.
- **Global `ThrottlerGuard` at 100 req / 60s.** Flag endpoints that need a custom (usually tighter) limit: login / OTP send / OTP verify / password reset / payment / webhook / producer signup / any financial mutation.
- **Log redaction config in `AppLoggerModule`** must cover `req.headers.authorization`, `req.headers.cookie`, `req.body.password`, `req.body.passwordConfirmation`, `req.body.token`, `req.body.refreshToken`, `req.body.secret`, `req.body.apiKey`. Flag any sensitive field added to a payload that isn't on the redaction list.
- **CORS configured per environment.** Never `origin: '*'` in production. Production allow-list must match the deployed frontend origins.
- **Helmet CSP must be configured and not regress.** Any `unsafe-inline` / `unsafe-eval` addition needs an inline justification.
- **Webhook controllers** use `@SkipEnvelope()` and verify HMAC against the raw request body (`req.rawBody`), not the parsed JSON. The Fastify body parser must preserve `rawBody` on webhook routes.

## Database

- **All Drizzle calls go through `DatabaseService` wrappers**: `db.query(promise, 'context-string')` and `db.tx(fn, 'context-string')`. Never the raw drizzle instance. The context string surfaces in the structured-log query line; missing it produces unsearchable logs.
- **Tables use `snake_case` columns; TypeScript uses `camelCase`.** Drizzle's column-name mapping handles the translation. Schema helpers `primaryId`, `timestamps`, `softDelete` from `common.schema.ts`.
- **Every read query touching a domain table** includes `isNull(schema.<table>.deletedAt)` in its `where` — soft-deleted rows must not leak.
- **Single-entity reads** use `.limit(1)` followed by destructuring: `const [entity] = await this.db.query(...)`. Don't `.then(r => r[0])` style.
- **Foreign-key conventions:**
  - `restrict` on parent FKs of financial / audit tables (orders, payments, refunds, payouts, ledger entries, discount_usage — anything that must be preserved for accounting)
  - `cascade` on dependent child rows (e.g. order_items → orders)
  - `set null` on circular FKs (use `AnyPgColumn` to break the circular import)
- **Index coverage for query patterns** — flag missing indexes on columns used in `WHERE` / `JOIN` / `ORDER BY` of new queries. Composite-index column order: most selective filter first.
- **Transaction boundaries** (`db.tx`) cover multi-step writes that must commit atomically.
- **N+1 query prevention** — eager-load related rows or batch-load via `inArray()`. Flag any `for ... of` loop containing an awaited DB call.
- **Migrations**: `pnpm db:generate --name=<descriptive>` — ALWAYS with `--name`. **Never `pnpm db:push` against a migrated database** — migrations only. Drizzle Studio holds `.git/index.lock` — flag any PR-prep guidance recommending studio without killing it first.

## Code Quality

- **`no-explicit-any` is OFF in Tikiti's ESLint config.** This is deliberate — the codebase has wide interop surface with third-party SDKs whose types are unstable. Don't flag `any` usages as findings UNLESS:
  - On a public service-layer return type (callers can't narrow)
  - On a Drizzle query result type (Drizzle infers; you shouldn't need `any`)
  - On a DTO field (class-validator can't validate `any`)
- **No `console.log`** — always use the pino-backed logger. `this.logger.setContext(ClassName.name)` in every service/repository constructor (flag a class wiring `LoggerService` without `setContext`).
- **Unused variables prefixed with `_`** to satisfy ESLint.
- **`async`/`await` throughout** — no raw `.then()` chains, no unhandled rejections, no `void`-promise floats.
- **Proper HTTP status codes:** 201 for create, 204 for delete (no body), 404 for missing resources (never `200 OK` with `null` payload), 409 for conflicts, 422 for semantic validation failures.
- **No circular dependencies between modules.** If module A imports module B which imports module A, refactor — usually by extracting the shared piece into a third module.
- **Config accessed via a service extending `BaseConfigService`** — never `process.env.*` outside the config layer.
- **BullMQ processors extend `WorkerHost`** and implement `process(job)`. Concurrency, retries, backoff via `@Processor()` decorator options — not hand-coded inside `process`.
- **Body limit is 1 MB at the Fastify adapter.** Flag any endpoint that needs more (probably should use a multipart upload path, not raise the limit).
- **URI versioning** default `v1`; health endpoint is version-neutral.
- **Request ID cascades** `x-request-id` → `x-correlation-id` → random UUID (Fastify `genReqId`).

## Domain Events

`EventEmitterModule` is wired globally (`libs/common/src/infrastructure/events/event.module.ts`) with `wildcard: true` and `.` delimiter. Listeners use `@OnEvent('domain.action')`. Emit domain events instead of direct cross-module calls when fan-out is needed — the audit pipeline (`shared/audit/audit.listener.ts`) consumes events emitted by services without those services knowing about the listener.

## Infrastructure (for `infra/` changes)

- **IAM policies scoped to the minimum required permissions.** Flag any role binding that grants broader access than the resource needs.
- **Prefer OIDC / Workload Identity Federation for cloud auth from CI** — flag any long-lived static credential (`AWS_ACCESS_KEY_ID` env var, JSON service-account key in a secret) being introduced.
- **Environment-specific configuration properly separated** (dev / staging / prod). No prod defaults bleeding into dev configs.
- **Terraform lock files** (`.terraform.lock.hcl`) consistent across environments and **committed** (not gitignored).
- **Destructive changes** (column drops, table renames, type changes, removed env vars consumed by live services) require a documented backfill / migration plan.
- **Secrets via Infisical** (`infisical run --env=<env>`) — never inline in `.tfvars` / shell scripts / `.env` files. The husky `pre-commit` hook blocks `.env` files (except `.env.*.example`).
- **Self-hosted CI runners** are `[self-hosted, aws, spot]` — flag any workflow that needs a different runner label.

## Workflow / process

- **Branch naming**: `<prefix>/<issue-number>-<description>` where prefix ∈ `{feature, bugfix, hotfix, chore, release, update, task, fix, ci, infra, refactor, docs}`. Husky `pre-push` enforces.
- **Commit messages**: conventional commits (`feat:`, `fix:`, `chore:`, etc., with optional scope). Husky `commit-msg` enforces.
- **PR title**: `<type>: <description> (#<issue>)`. PR body must include `Closes #<issue>`.
- **One ticket = one PR, squash merge.** Flag PRs that bundle multiple unrelated tickets.
- **Never add AI / Claude attribution to commits, PRs, or reviews** — internal Tikiti policy.

## Cross-referenced runbooks

For PRs touching deep operational topics, the reviewer references the relevant runbook in the consumer repo's `.claude/context/`:

| Topic | Runbook |
|-------|---------|
| OTP / auth changes | `.claude/context/auth.md` |
| Schema / migration changes | `.claude/context/drizzle.md` |
| RDS user / IAM auth changes | `.claude/context/rds-runbook.md` |
| Secrets handling | `.claude/context/infisical.md` |
| Logging changes | `.claude/context/grafana-logs.md` |
| Email / OTP behaviour | `.claude/context/ses-email.md` |
| Gitflow / branching | `.claude/context/gitflow.md`, `.claude/context/team-workflow.md` |
