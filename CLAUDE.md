## Project Overview

**Remote Agentic Coding Platform**: drive Claude Code SDK / Codex SDK from Slack, Telegram,
GitHub, Discord, CLI and web. Bun + TypeScript + SQLite/PostgreSQL, single-developer tool — no
multi-tenancy. Strict TS; `any` needs a written justification.

Reference material that lives outside this file, because it is generated and would rot here:
package layout (`ls packages/*/src`), CLI surface (`bun run cli --help`), HTTP surface
(`GET /api/openapi.json`), user docs (`packages/docs-web/`).

## Verifying work

```bash
bun run validate     # type-check + lint + format:check + test — must be green before any PR
bun run test         # per-package, isolated processes
```

- **Never `bun test` from the repo root.** It discovers every package in one process and
  `mock.module()` pollution produces ~135 failures. `bun run test` shells out per package.
- `mock.module()` is process-global and **irreversible** — `mock.restore()` does not undo it
  ([oven-sh/bun#7823](https://github.com/oven-sh/bun/issues/7823)). Packages with conflicting
  mocks split into separate `bun test` invocations (`@archon/core` 7 batches, `@archon/workflows` 5,
  `@archon/adapters` 4, `@archon/isolation` 3 — exact splits in each `package.json`). A new test
  file using `mock.module()` must land in a batch with no conflicting path.
- Use `spyOn()` for internal modules other test files import directly; `spy.mockRestore()` *does*
  work.
- Lint is `--max-warnings 0` in CI. Inline `eslint-disable` is acceptable only for a wrong external
  SDK type or a validated assertion, and must name the reason. Never file-level, never to pass CI.
- End-to-end checks go through the web API or the CLI, not a second platform adapter.

## Git workflow

- `main` is the release branch — **never commit to it directly**. `dev` is the working branch;
  feature branches fork from `dev` and merge back into `dev`.
- Release via the `/release` skill (`/release`, `/release minor`, `/release major`): it diffs `dev`
  against `main`, writes `CHANGELOG.md` (Keep a Changelog), bumps the single `version` in the root
  `package.json`, opens the PR.
- Commits: present tense, first line under 72 chars.
- **Never run `git clean -fd`** — it deletes untracked files permanently. `git checkout .` instead.
- Call git through `@archon/git`; when calling directly use `execFileAsync`, never `exec`.

## Package layering

`paths` ← `git` ← `isolation` ← `workflows` ← `core` ← `adapters` ← `server`; `cli` and `web` sit on
top. The arrows are enforced constraints, not description:

- `@archon/paths` has zero `@archon/*` deps (path utils + Pino logger factory).
- `@archon/workflows` depends only on `git` + `paths` + zod. DB, AI and config arrive injected via
  `WorkflowDeps`; `core` supplies the bridge with `createWorkflowStore()`.
- **`@archon/web` must never import `@archon/workflows`** — it is a server package. Web takes
  `DagNode`, `WorkflowDefinition`, `WorkflowRunStatus` from `@/lib/api`, re-exported from
  `api.generated.d.ts`. Regenerate with the server already up on 3090:
  `bun run dev:server` then `bun --filter @archon/web generate:types`.

Imports: `import type` for types, named imports for values, `import * as` only for submodules with
many exports (`import * as git from '@archon/git'`). Never `import * as core from '@archon/core'`.
Workflow engine internals import from direct subpaths (`@archon/workflows/executor`, `/store`,
`/schemas/workflow`). Prefer SDK types verbatim over re-declared local copies.

## Zod + OpenAPI

- Import `z` from `@hono/zod-openapi`, not from `zod`.
- Every new or modified route goes through the local `registerOpenApiRoute(createRoute({...}),
  handler)` wrapper — it carries the TypedResponse bypass.
- Types always derive: `z.infer<typeof schema>`. Never a hand-written parallel interface.
- Route schemas: `packages/server/src/routes/schemas/`, one file per domain. Engine schemas:
  `packages/workflows/src/schemas/`, one file per concern, all re-exported from `index.ts`.
  camelCase with a descriptive suffix (`workflowRunSchema`, `dagNodeSchema`).
- `TRIGGER_RULES` and `WORKFLOW_HOOK_EVENTS` derive from schema `.options` — never duplicated as a
  plain array. Sole exception: `@archon/web`, where `api.generated.d.ts` is type-only and cannot
  export runtime values.
- `loader.ts` validates nodes with `dagNodeSchema.safeParse()`; graph-level checks (cycles, deps,
  `$nodeId.output` refs) stay imperative in `validateDagStructure()`.

## Storage and sessions

SQLite at `~/.archon/archon.db` by default, zero setup; setting `DATABASE_URL` switches to
PostgreSQL, whose migrations are manual (`psql $DATABASE_URL < migrations/000_combined.sql`).
Tables are prefixed `remote_agent_`.

- **Conversation id is platform-specific** and is the join key everywhere: Slack `thread_ts`,
  Telegram `chat_id`, GitHub `owner/repo#number`, Discord channel id, web a user-supplied string.
- One active session per conversation. **Sessions are immutable** — a transition writes a new row
  linked by `parent_session_id` with an explicit `transition_reason`. Only plan→execute creates the
  successor immediately; every other trigger just deactivates the current one.
- Codebase command *bodies* live on the filesystem; only their paths go in `codebases.commands`.

## Platform adapters

Each implements `IPlatformAdapter` and exposes `onMessage(handler)`; the caller owns errors.
Slack and Telegram poll (no webhooks); GitHub is webhooks + `gh`.

**Authorization happens inside the adapter**, never in the caller — whitelist parsed from its env
var in the constructor (e.g. `TELEGRAM_ALLOWED_USER_IDS`), checked before `onMessage` fires.
Unauthorized input is rejected **silently**, logged with the user id masked.

GitHub `@archon` mentions are parsed in **comments only** (`issue_comment` events), never in issue
or PR descriptions — descriptions routinely contain example commands, and treating them as
invocations was bug #96. Verify webhook signatures (`X-Hub-Signature-256`, HMAC SHA-256) reading the
raw body with `c.req.text()`; return 200 immediately and process async.

## Workflows and commands

- Definitions in `.archon/workflows/` (searched recursively), commands in `.archon/commands/`.
  Load priority: bundled defaults < global (`$ARCHON_HOME/.archon/workflows/`) < repo, overriding by
  filename. Opt out with `defaults.loadDefaultCommands` / `loadDefaultWorkflows: false`.
- Discovery is resilient: one broken YAML does not abort the scan, its error shows in
  `/workflow list`.
- `nodes:` is the DAG format — `depends_on` edges, nodes in a topological layer run concurrently.
  Types: `command:`, `prompt:`, `bash:`, `script:` (bun/uv, needs explicit `runtime:`),
  `loop:`, `approval:`. `bash:` and `script:` capture stdout as `$nodeId.output` and invoke no AI.
  `approval:` needs `capture_response: true` for its comment to reach downstream nodes.
- Claude-only node options: `allowed_tools`/`denied_tools`, `hooks`, `mcp`, `skills`, and
  `effort`/`thinking`/`maxBudgetUsd`/`systemPrompt`/`fallbackModel`/`betas`/`sandbox`.
- **`interactive: true` forces foreground execution on web** — required for any workflow with an
  approval gate, or the UI has nothing to approve against.
- Option precedence: workflow YAML > `.archon/config.yaml` `assistants.*` > SDK defaults. Model and
  provider are validated **at load time**, so an incompatible pair fails discovery, not execution.
- `resolveWorkflowName()` (`router.ts`) resolves names by exact → case-insensitive → `-name` suffix
  → substring, with ambiguity detection; CLI and every chat platform share it.
- Routing falls back to `archon-assist` when no `/invoke-workflow` is produced. Claude routing calls
  pass `tools: []` to block tool use at the API level; a detected Codex tool bypass takes the same
  fallback.

**Substitution variables:** `$1`…`$3`, `$ARGUMENTS`, `$ARTIFACTS_DIR`, `$WORKFLOW_ID`,
`$BASE_BRANCH` (auto-detected from git unless `worktree.baseBranch` is set; fails only if a prompt
references it *and* detection failed), `$DOCS_DIR` (`docs.path`, default `docs/`, never throws),
`$LOOP_USER_INPUT` (only the first iteration of a resumed interactive loop, empty otherwise),
`$REJECTION_REASON` (only inside `on_reject` prompts, empty otherwise).

## Isolation and worktrees

Worktrees are the **default** for `cli workflow run`; `--no-worktree` opts back into the live
checkout. Workflow and isolation commands must run inside a git repo (subdirectories resolve to the
root). Workspaces sync with origin before creating a worktree.

Ports auto-allocate by hashing the worktree path into 3190–4089, so the same worktree always gets
the same port; the main repo stays on 3090 and `PORT=` overrides both. Let agents self-test through
the web API on that port rather than starting a second set of platform adapters — the database is
shared.

Let git enforce what git already enforces (it refuses to drop a worktree with uncommitted changes).
Surface conflicts and uncommitted-change errors to the user; swallow only the expected ones, like a
missing directory during cleanup. Map raw git failures through `classifyIsolationError()` from
`@archon/isolation` and log the original error alongside the friendly message.

`--allow-env-keys` (CLI) and `PATCH /api/codebases/:id` grant the env-leak gate for repos whose
`.env` holds real secrets. Every grant and revoke is audit-logged at `warn`
(`env_leak_consent_granted` / `..._revoked`) with the actor — do not downgrade that level.

Layout: `~/.archon/workspaces/owner/repo/` holds `source/` (clone or symlink), `worktrees/`,
`logs/`, and `artifacts/` — **artifacts never go into git**. `ARCHON_HOME` moves the base; Docker
sets it to `/.archon/`.

## Logging

`createLogger('<domain>')` from `@archon/paths` (Pino). Event names are
`{domain}.{action}_{state}` — `workflow.step_started`, `isolation.create_failed`. Every `_started`
is paired with a `_completed` or a `_failed`; no generic `processing`/`handling`. Carry ids,
durations and `errorType` in the object argument.

**Never log** API keys or tokens (mask as `token.slice(0, 8) + '...'`), user message content, or PII.

Verbosity: `archon --quiet` / `--verbose` on the CLI, `LOG_LEVEL=debug` on the server.

## Error handling

Throw early and explicitly; a silent fallback in an agent runtime is how permissions and cost
quietly broaden. If a fallback is deliberate and safe, say so in a comment at the fallback.
DB writes log the failing params and rethrow a caller-meaningful error; updates rely on the store
throwing when no row matched, so never treat a zero-row update as success.
