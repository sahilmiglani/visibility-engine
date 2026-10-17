# CLAUDE.md — working in the Visibility Engine repo

Visibility Engine is Studio5's AI-search visibility (AEO/GEO) engine. It asks ChatGPT, Perplexity and
Gemini buyer-style questions, records what they answer and cite, scores each prospect against
their competitors, and turns the strongest true finding into a cold-email hook and a shareable
report page. Read `docs/BRIEF.md` for the why and `docs/STATUS.md` for where things stand.

## Stack

- **App:** Next.js (App Router, TypeScript, strict mode) on Vercel. Two surfaces: the
  auth-protected admin (`/admin/**`) and the public report pages (`/r/[token]`).
- **Database, auth and storage:** Supabase (Postgres plus RLS, a single admin user, private storage buckets).
- **Background jobs:** Trigger.dev (`trigger/**`) for retries, queues and per-engine concurrency.
- **LLM steps:** Claude through the Anthropic API. Every output is validated with a zod schema.
- **Probe engines:** OpenAI Responses API with web search, Perplexity Sonar and Gemini with
  Google Search grounding. All three sit behind the `EngineAdapter` interface (`src/engines/types.ts`).
- **Tests:** Vitest for unit and integration tests, Playwright for report and admin smoke tests. Tests never call live APIs.

## Commands

Commands arrive with the foundation scaffold. Keep this list accurate as they land.

| Command | What it does |
|---|---|
| `pnpm install` | Install dependencies |
| `pnpm dev` | Next.js dev server |
| `pnpm trigger:dev` | Trigger.dev dev worker (`npx trigger.dev@latest dev`) |
| `pnpm db:start` / `pnpm db:reset` | Local Supabase (`supabase start`, `supabase db reset`) |
| `pnpm db:types` | Regenerate `src/core/db.types.ts` from the local DB |
| `pnpm typecheck` | `tsc --noEmit` |
| `pnpm lint` | ESLint plus Prettier check |
| `pnpm test` | Vitest (unit and integration, mock engines, no network) |
| `pnpm test:e2e` | Playwright smoke tests against a local build |
| `pnpm smoke:live` | **Manual only.** Runs the Phase 1 acceptance with real API keys. Never run in CI. |

## Conventions

- **Specs before code.** Every feature has a spec in `docs/specs/`. If the code needs to differ
  from the spec, update the spec in the same PR.
- **Ownership.** `docs/specs/build-plan.md` assigns each directory to one workstream. Don't edit
  files that another workstream owns. If you need a change there, note it in your PR and in `docs/STATUS.md`.
- **Business logic is pure and lives in `src/`.** Trigger tasks in `trigger/` are thin wrappers
  that load data, call `src/` functions and persist results. This keeps the logic testable without Trigger.dev.
- **Every external call is logged.** Engine calls go to `engine_calls` and Anthropic calls go to
  `llm_calls`, each with the raw payload, model ID, parameters, timestamps, latency and `cost_usd`. Nothing calls a provider SDK directly
  except through `src/engines/*` or `src/llm/client.ts`.
- **Structured LLM output.** Each LLM step has a zod schema in `src/llm/schemas/`. Validate the
  output, retry once on a validation failure, then fail loudly. Never accept unvalidated JSON.
- **Evidence or nothing.** A hook or report claim must reference stored `engine_calls.id` rows. No evidence means no hook.
- **Framing.** All prospect-facing copy says "In our tests on {date}". API answers can differ from the consumer apps.
- **Errors are not absences.** A failed or empty engine call is "no data", never "not mentioned".
- **Secrets come only from env vars.** `.env.example` lists every variable. Never commit real values.
- **No live APIs in tests.** Use `fixtures/` and the mock adapter. MSW runs with `onUnhandledRequest: "error"`.
- **No outbound email and no social automation** anywhere in this codebase.
- **SQL:** snake_case, plural table names, `id uuid primary key default gen_random_uuid()`,
  `created_at timestamptz not null default now()`. RLS is on for every table. Migrations live in
  `supabase/migrations/` and are never edited after they merge.
- **TypeScript:** named exports, no `any`, zod at every boundary (CSV, HTTP, LLM, DB JSON columns).
- **Commits:** small and descriptive. Push work in progress often to your own branch.

## Before finishing any task

1. Update `docs/STATUS.md`: what's built, what's in progress, what's next and any blockers.
2. Append dated one-liners to `docs/DECISIONS.md` for anything you decided and why.
3. Run `pnpm typecheck && pnpm lint && pnpm test` once the scaffold exists.

`/wrap` (`.claude/commands/wrap.md`) runs steps 1 and 2 from the session's work.

## If something is missing

If you're missing an API key, a decision or network access, say exactly what's missing and stop.
Don't mock around it silently and don't guess.
