# Stack — Next.js (App Router) + Turborepo + shadcn/ui + Tailwind + TypeScript

A general review prompt for Next.js 14/15 monorepos using the App Router, pnpm workspaces + Turborepo, shadcn/ui on Tailwind v4, React Query / Zustand / react-hook-form + Zod for state and forms, and a BFF route handler for backend access. Reviewers should apply these conventions in addition to the base review areas. Any project-specific deviations (custom feature paths, store names, auth flow specifics, allowed `any` deviations, etc.) should be supplied via the consumer's `extra_prompt` or `extra_prompt_path` input and will appear under the "Repo-specific notes" section.

## Architecture & Patterns

- **`app/` is routes-only.** Page components stay thin — they compose UI from `features/` (or `shared/`) and never hold business logic, transformers, or data-fetching primitives directly.
- **Feature-based organisation.** Cross-cutting concerns for a feature live together: `features/<feature>/{components,hooks,api,transformers,types,lib}/`. No cross-feature imports — a feature should only import from `@workspace/shared` (the workspace package), the app's `shared/` folder, or its own directory.
- **Data flow is `API → Transformer → Hook → Component`.** Components consume only transformed, UI-shaped data. Never call `fetch` / `axios` directly from a component, and never consume raw API DTO shapes — always go through a transformer (even if today the shapes look identical; the transformer is the seam).
- **React Query is the single source of truth for server state.** Server data is never duplicated into `useState`, Zustand, Context, or any in-memory cache as a fallback. Use query-key factories (object literals) for consistent invalidation.
- **Mutation hooks return `mutateAsync` + `isPending`** (not `mutate`) so callers can await. `onSuccess` invalidates every related query (list, detail, counts).
- **Workspace packages** (`@workspace/shared`, etc.) must be added to `next.config.{mjs,ts}` under `transpilePackages`. Flag any new workspace import that isn't transpiled.

## Security

- **All backend calls go through the BFF route handler** at `app/api/[...path]/route.ts` (or equivalent). Never call the backend origin directly from the browser, and never expose backend URLs in `NEXT_PUBLIC_*` env vars unless they are public.
- **Auth tokens live in `httpOnly` cookies set server-side** by the BFF (and the shared `tokenManager` / equivalent). NEVER store access or refresh tokens in `localStorage`, `sessionStorage`, or any client-readable state. Flag any code that writes a token to `document.cookie` from the client.
- **No server data in `localStorage` / `sessionStorage` / Context / Zustand-persisted state.** Every page must fetch its data via hooks on mount; if a refresh causes data to disappear, the implementation is wrong. The only acceptable client-storage uses are UI-only preferences (theme, sidebar collapsed, last-selected tab) and explicit auth-state flags (`{app}_authenticated`).
- **No hardcoded secrets, API keys, account IDs, vendor secrets, or production URLs** in source. `NEXT_PUBLIC_*` env vars are shipped to the browser — never put a secret behind that prefix.
- **Input validation with Zod at every system boundary** — form submit, query-string parse, BFF route-handler input. `react-hook-form`'s `zodResolver` is the standard form path.
- **`dangerouslySetInnerHTML` requires a sanitiser** (DOMPurify or equivalent) and an inline justification. Same for `eval` / `Function(...)`-style dynamic code.
- **Webhooks proxied through the BFF must verify the upstream signature against the raw body**, not the parsed JSON. The body parser must preserve the raw bytes.

## TypeScript & Code Quality

- **No `any` types.** Use proper types or `unknown` + narrowing. (The consumer repo may opt out via the extras section.)
- **Inline type imports:** `import { type Foo, bar } from "module"` — not `import type { Foo } from "module"`. Enforced by `consistent-type-imports` with `fixStyle: "inline-type-imports"`.
- **`noUncheckedIndexedAccess: true`** is the default; `array[0]` is `T | undefined`. Flag indexed access used without checking.
- **No `console.log`** in committed code. Use `console.warn` / `console.error` for unexpected paths, or a structured logger if the project has one. Client-side log sinks (a `ClientLogger` posting to `/api/logs` or equivalent) are the preferred pattern over ad-hoc `console`.
- **No nested ternaries.** Use early returns, lookup objects, or extracted helpers.
- **Named exports only.** Default exports are reserved for Next.js pages / layouts / route handlers where the framework requires them.
- **Object shorthand, template literals, unused-vars prefixed with `_`** — all enforced by ESLint; reviewers should flag drift.

## React & Component Patterns

- **shadcn/ui is the UI primitive layer.** Always compose from `@workspace/shared/components/*` (or the consumer's shared package equivalent). Never hand-roll a custom button/input/dialog/card/etc. when a shadcn equivalent exists. New shadcn components are added from `packages/shared/` (or equivalent), and any new file's `src/lib/utils` import must be rewritten to `@/lib/utils`.
- **Use `cn()` for `className` composition.** Don't string-concatenate Tailwind classes or conditionally branch with ternaries inline; pass a structured arg to `cn()`.
- **`class-variance-authority` (cva) for component variants.** Don't fork a component to vary a prop — define variants.
- **Forms: `react-hook-form` + Zod schema** in `features/<feature>/lib/`. Inline error handling via `getFieldErrors(error)` (or equivalent) mapping API field errors back to `setError`.
- **Animations: `framer-motion`.** Prefer it over CSS animations/transitions for any stateful or interactive case (mount, exit, layout, gestures). CSS is fine for purely static fades.
- **`'use client'` only where required.** Server components by default. Flag any page-level component that's marked client-only without a justification (a hook, an event handler, a browser API).

## Next.js & Routing

- **App Router only.** No `pages/` patterns, no `getServerSideProps`, no `_app.tsx`. Data loading goes through server components, route handlers, or React Query on the client.
- **`output: "standalone"`** in `next.config.{mjs,ts}` for any app that ships in a container. Flag a missing standalone output on a deployed app.
- **Middleware is the route-protection layer.** Public routes (`/signin`, `/signup`) bypass; protected routes redirect to `/signin?callbackUrl=…`. The check reads the httpOnly access-token cookie — never a `localStorage`-backed flag.
- **Path aliases:** `@/*` → app root, `@workspace/shared/*` → shared package, app's `@/shared/*` → app's own shared folder. Flag relative imports (`../../../...`) that should use an alias.

## Tooling & Workflow

- **pnpm exclusively.** No `npm` or `yarn` commands in scripts, READMEs, or CI. Lockfile is `pnpm-lock.yaml`.
- **Turborepo for task orchestration.** Tasks declared in `turbo.json` must list their inputs/outputs so the cache is correct. Don't add a task that ignores caching without justification.
- **Imports auto-sorted** by `simple-import-sort` (or `eslint-plugin-import` order). Don't hand-order imports.
- **Lint must pass with `--max-warnings 0`** in CI. `onlyWarn` (if used) demotes errors to warnings; `--max-warnings 0` still makes them blocking. Don't disable a rule inline without a `// eslint-disable-next-line <rule> — <why>` comment.
- **No `*.test.ts` if no test runner is wired up.** Reviewers should check the project for Jest/Vitest/Playwright before suggesting test files; if only Playwright e2e is configured, unit-test files won't run and shouldn't be added.

## Styling

- **Tailwind v4 patterns only.** Design tokens via `@theme` directive in `globals.css`; no `tailwind.config.{js,ts}` v3-style config file unless the project explicitly opts back in.
- **Shared global styles** live in the workspace shared package (e.g. `packages/shared/src/styles/globals.css`). App-specific extensions go in the app's own globals.
- **No magic colour / spacing values** in JSX — use the design tokens. Flag literal hex codes / `px` values that should be Tailwind utilities or `@theme` tokens.

## Observability

- **Correlation IDs propagate end-to-end.** Generated per session (or per request) and injected as `x-correlation-id` on every BFF→backend call and into every client log. The BFF preserves the inbound header.
- **Structured logger over `console`.** Server-side: pino (with secret redaction for `authorization`, `cookie`, `*.password`, `*.refreshToken`, `*.accessToken`). Client-side: a batching logger that posts to `/api/logs` (or equivalent) — never raw `fetch('/log', …)` in components.
- **`/api/health` endpoint** returns `{ status, app, version, startedAt, timestamp }` for every deployed app.

## What "Repo-specific notes" Should Cover

Consumer repos extend this prompt via `extra_prompt_path` (a markdown file in their own `.github/`) for things this generic prompt can't know:

- Whether `no-explicit-any` is genuinely enforced or relaxed (some monorepos disable it).
- The exact set of feature directories and which ones are fully built vs scaffolded.
- App-specific auth flows (consumer dialog vs producer email-OTP vs admin email-password).
- Cookie name prefixes, BFF path conventions, custom middleware behaviour.
- Custom guards / decorators / interceptors specific to the project.
- Allowed deviations from the rules above and *why*.
- Files / directories whose conventions diverge from the rest of the app (e.g. consumer-vs-producer architecture differences).
- Pre-commit / pre-push hook behaviour, branch-naming regex, commit-message format.
