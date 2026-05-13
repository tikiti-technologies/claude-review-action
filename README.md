# claude-review-action

Tikiti's reusable GitHub Actions workflows for AI code review on every pull request. Hosts two parallel reviewers:

- **Claude** — via [`anthropics/claude-code-action@v1`](https://github.com/anthropics/claude-code-action). OAuth-authed, posts inline review comments and a top-level summary.
- **Gemini** — direct call to the Gemini 2.5 API. Posts a single comment with severity-tagged findings rendered from a JSON response.

Both reviewers load `prompts/base.md` (universal rules), `prompts/<stack>.md` (stack-specific conventions for the calling repo's stack), the consumer's `CLAUDE.md`, and an optional `extra_prompt` / `extra_prompt_path` for repo-specific layering.

The `prompts/nestjs-fastify-drizzle.md` stack prompt encodes Tikiti's conventions directly (guard names, `@app/common` exception subclasses, `DatabaseService` wrappers, integration orchestrator/provider paths, the `no-explicit-any: off` override, "no AI attribution on commits/PRs/reviews" rule, etc.) so consumer repos only need to ship truly repo-specific extras.

## Usage in a consumer repo

Two thin wrappers under `.github/workflows/`:

### `claude.yml`

```yaml
name: Claude Code

on:
  pull_request:
    types: [opened, synchronize, ready_for_review, reopened]
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]

# Required — reusable workflows can only restrict, not elevate, permissions.
permissions:
  actions: read
  contents: read
  pull-requests: write
  issues: write
  id-token: write

jobs:
  review:
    uses: tikiti-technologies/claude-review-action/.github/workflows/claude-review.yml@v1
    with:
      stack: nestjs-fastify-drizzle
      runner: '["self-hosted","aws","spot"]'
      slack_notify: true
      # Optional — repo-specific notes (this repo's tables, integrations, threshold tunings, etc.)
      extra_prompt_path: .github/review-extras.md
    secrets:
      claude_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
      slack_webhook_url:  ${{ secrets.SLACK_WEBHOOK_URL }}
```

### `gemini.yml`

```yaml
name: Gemini Code Review

on:
  pull_request:
    types: [opened, synchronize, ready_for_review, reopened]
  issue_comment:
    types: [created]

permissions:
  contents: read
  pull-requests: write

jobs:
  review:
    uses: tikiti-technologies/claude-review-action/.github/workflows/gemini-review.yml@v1
    with:
      stack: nestjs-fastify-drizzle
      runner: '["self-hosted","aws","spot"]'
      slack_notify: true
      extra_prompt_path: .github/review-extras.md
    secrets:
      gemini_api_key:    ${{ secrets.GEMINI_API_KEY }}
      slack_webhook_url: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### Mentioning the bots

- `@claude` — re-runs Claude. Also triggers on `pull_request_review_comment` events so you can prompt it inline from a review thread.
- `@gemini` — re-runs Gemini.

The `if:` guard inside each reusable workflow handles routing.

## Layering repo-specific conventions

The stack prompt covers Tikiti's org-shared conventions (applied to every consumer). To layer in **repo-specific** conventions (e.g. forma's multi-tenant invariants, a service's specific table list), use:

- **`extra_prompt: <inline string>`** — for short notes. Multi-line YAML strings get awkward beyond a paragraph.
- **`extra_prompt_path: <path>`** — points at a markdown file inside the consumer repo (e.g. `.github/review-extras.md`). Read at workflow-run time and appended under a "Repo-specific notes" heading.

If both are set, file contents render first, then the inline string — both under one heading.

## Inputs

### Claude (`claude-review.yml`)

| Input | Type | Default | Description |
|---|---|---|---|
| `stack` | string | (required) | Prompt template name in `prompts/<stack>.md` |
| `runner` | string (JSON) | `"ubuntu-latest"` | `runs-on` value, JSON-encoded |
| `ref` | string | `v1` | Pinned ref of this central repo for prompts |
| `slack_notify` | boolean | `false` | Post a Slack message on completion |
| `extra_prompt` | string | `''` | Inline repo-specific addendum |
| `extra_prompt_path` | string | `''` | Path (in consumer repo) to a markdown file appended after the stack prompt |

### Gemini (`gemini-review.yml`)

All Claude inputs above, plus:

| Input | Type | Default | Description |
|---|---|---|---|
| `model` | string | `gemini-2.5-flash` | Gemini model ID |
| `max_total_lines` | number | `5000` | Cap on total source lines sent to Gemini |
| `max_per_file` | number | `1000` | Skip files larger than this |
| `max_files` | number | `20` | Cap on file count |

## Secrets required

| Workflow | Secret | Source |
|---|---|---|
| Claude | `claude_oauth_token` | Anthropic OAuth (Claude.ai → Account → Code → OAuth Token) |
| Gemini | `gemini_api_key` | Google AI Studio |
| Either | `slack_webhook_url` (optional) | Tikiti Deploy Slack app — Incoming Webhooks |

Set on the consumer repo via:

```bash
gh secret set CLAUDE_CODE_OAUTH_TOKEN --repo tikiti-technologies/<repo>
gh secret set GEMINI_API_KEY          --repo tikiti-technologies/<repo>
gh secret set SLACK_WEBHOOK_URL       --repo tikiti-technologies/<repo>
```

## Versioning

Consumers pin to a tag. Three conventions:

- **`@v1` — floating major (recommended for most repos).** The `v1` tag is moved forward to the latest commit in the 1.x line. Consumers automatically pick up prompt updates, bug fixes, and non-breaking input additions on their next PR. Breaking changes never land here — they go on `v2`.
- **`@v1.x.y` — strict pin.** Locks the consumer to that exact commit. Use when a repo needs guaranteed determinism (compliance audits, frozen review criteria).
- **`@main` — bleeding edge.** Not recommended outside of dogfooding within this central repo.

When cutting a release: tag the strict version AND fast-forward `v1` to the same commit. Breaking prompt changes (different output format, removed inputs) require a new major — tag `v2.0.0` + create a new `v2` floating tag; leave `v1` untouched.

## Caller permissions

Reusable workflows are capped by the caller's `permissions:` block — the called workflow can only equal or restrict the caller's grants, never broaden them. **Each consuming workflow must declare its own `permissions:` block** at the workflow or job level, even though this reusable workflow declares them internally.

For Claude:

```yaml
permissions:
  actions: read
  contents: read
  pull-requests: write
  issues: write
  id-token: write
```

For Gemini:

```yaml
permissions:
  contents: read
  pull-requests: write
```

Without these, the effective `GITHUB_TOKEN` falls back to the repo's default (typically read-only on private repos), and the actual review-posting / Slack-notify / OIDC steps silently fail.

## Adding a new stack

1. Create `prompts/<stack-name>.md` with stack conventions Tikiti follows for that stack.
2. Reference it as `stack: <stack-name>` in consuming repo wrappers.
3. Optionally cut a new tag (`v1.x.0`) so consumers can pin to it.

`base.md` covers universal rules and is always loaded first.

## Layout

```
.
├── .github/workflows/
│   ├── claude-review.yml    # reusable workflow (workflow_call)
│   └── gemini-review.yml    # reusable workflow (workflow_call)
├── actions/
│   └── slack-notify/
│       └── action.yml       # composite action used by both reviewers
├── prompts/
│   ├── base.md              # universal review rules (always loaded)
│   └── nestjs-fastify-drizzle.md   # Tikiti's NestJS+Drizzle conventions
└── README.md
```
