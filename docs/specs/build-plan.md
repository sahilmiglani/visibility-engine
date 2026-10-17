# Build plan (Phase 1)

## Open questions (please answer before or while the foundation is built)

Each has a recommended default. The foundation and W1–W4 can go ahead on the defaults. Items marked
**blocking** must be answered before W5, the live acceptance run.

1. **Gemini grounding terms (blocking for Gemini).** The Gemini API terms for Grounding with Google Search say
   grounded results may be shown only "to the end user who submitted the prompt" and must not be
   "analyze[d]" or cached, beyond narrow storage exceptions (engines.md §3.5). Our product scores those
   answers and shows them on a prospect's report, which looks like a violation. Options:
   - (a) Legal review or written permission from Google.
   - (b) Gemini **without** grounding, labelled "Gemini (model knowledge)".
   - (c) Google AI Overviews or AI Mode through a SERP-data vendor, pulled forward from the roadmap.
   - (d) Drop Gemini for Phase 1.
   - **Recommended:** (d) now, with the adapter built and fixture-tested behind the flag, then evaluate (b) or (c) in Phase 2.
     `ENGINES_ENABLED` defaults to `openai,perplexity`.
2. **Prompt set activation.** Should a newly generated category prompt set need one admin click before
   probing? **Recommended: yes.** Prompts decide every finding we send.
3. **Perplexity engine choice.** Sonar Chat Completions support ended on 2026-09-27, and Perplexity points everyone to the Agent API.
   **Recommended:** Agent API with `model: "perplexity/sonar"` plus `web_search`. That's Perplexity's own grounded model,
   while presets run on OpenAI models. Alternative: the `fast` preset, which is closer to what the API team tunes but isn't Perplexity's model.
4. **Absence threshold.** Should hook (a) require **zero** mentions, or allow at most N mentions (e.g. ≤ 1 of 40)?
   **Recommended: zero for Phase 1.** It's the most defensible claim.
5. **OpenAI model and pricing (blocking).** The build environment couldn't load developers.openai.com or openai.com,
   so prices and rate limits are unverified. Please either allowlist those hosts in this environment's network policy or confirm the
   figures from your OpenAI console.
   **Recommended model:** `gpt-6.1-sol` with reasoning effort `low`, as the closest mainstream ChatGPT proxy. Alternative: `gpt-6-astra` (5× the price).
6. **One report per company, a link per contact.** Several contacts at one company share one report, each with
   its own tracked link. Is that right? **Recommended: yes.**
7. **Cycle reuse window.** Should a new prospect reuse category answers up to 14 days old? **Recommended: 14 days**
   (`CYCLE_REUSE_DAYS`). The report states the test date.
8. **Report branding and copy.** Studio5 logo, legal entity name, a contact email for the footer and a
   "remove me" address, plus the Cal.com link. **Needed before W5.**
9. **"Position gap" hook.** For prospects who *are* visible but rank below competitors, should we add a fifth hook type,
   e.g. "ChatGPT named you 4th, after X, Y and Z"? Without it they end up `no_finding`. **Recommended:** add it in Phase 1.5,
   after we see the first run's distribution.
10. **Non-US handling.** Should non-US contacts and prospects be imported but excluded from export, with a per-row admin override?
    **Recommended: yes.**
11. **Accounts and keys needed for W5 (blocking):**
    - OpenAI, Perplexity (Tier 0 is fine at 1 QPS), Anthropic, and Google AI Studio with billing if Gemini stays.
    - A Supabase project, a Trigger.dev project (Free plan is fine for Phase 1: 20 concurrent runs, $5 credit), and a Vercel project.
    - DNS for `visibility.studio5.in`.
    - A Cal.com event link.
12. **Data retention.** How long do we keep raw answers and site snapshots for prospects who never reply?
    **Recommended:** keep them for 12 months, then purge raw payloads and keep aggregates.

## Delivery rules
- One foundation worker first. Then four parallel workers (W1–W4), each on its own branch
  (`feat/w1-engines`, …) opening its own PR into `main`. Then W5 integration.
- **Directory ownership is exclusive.** A worker edits only the paths it owns, plus new files it
  creates inside them. If a worker needs a change elsewhere, it writes the request in its PR description and in
  `docs/STATUS.md`. The owning worker or the integrator makes the change.
- Shared contracts are owned by the foundation and frozen after the foundation PR merges:
  - `src/core/**` (types, db types, pricing interface, cost recorder).
  - `src/engines/types.ts`.
  - `src/llm/client.ts` and its mock interface.
  - `supabase/migrations/0001_*`.
  - Changes to these need a small contract PR reviewed before dependent work continues.
- **Migrations:** after the foundation, a worker that needs schema adds a **new** file named
  `supabase/migrations/<UTC timestamp>_w<N>_<desc>.sql`, never editing an existing one. The integrator
  regenerates `src/core/db.types.ts` after each merge. Workers regenerate locally but don't commit
  that file, to avoid conflicts.
- Every worker:
  - writes fixtures-based tests (testing.md §4 checklist),
  - keeps `pnpm typecheck && pnpm lint && pnpm test` green,
  - pushes work in progress often,
  - updates `docs/STATUS.md` and `docs/DECISIONS.md` before opening its PR.
  - Specs for its area are its to update.

## Step 0: Foundation (one worker, branch `feat/foundation`)

| Owns | Delivers |
|---|---|
| `package.json`, `pnpm-lock.yaml`, `tsconfig.json`, `next.config.ts`, `eslint.config.mjs`, `.prettierrc`, `vitest.config.ts`, `vitest.setup.ts`, `playwright.config.ts`, `.github/workflows/**`, `.env.example` | Next.js App Router plus TS strict scaffold, pnpm, ESLint and Prettier, Vitest with MSW and the network guard, Playwright config, CI (typecheck, lint, test, build) |
| `supabase/config.toml`, `supabase/migrations/0001_init.sql`, `supabase/seed.sql`, `scripts/create-admin.ts` | All Phase 1 tables, indexes, views, RLS, the `is_admin()` and `is_suppressed()` functions, the `get_report_by_token` and `record_report_view` RPCs, storage buckets, and a seed that loads the fixture scenario |
| `trigger.config.ts`, `trigger/_shared/**`, `trigger/index.ts` | Trigger.dev v4 init, named concurrency limits (`engine-openai`, `engine-perplexity`, `engine-gemini`, `llm-reason`, `llm-extract`, `page-fetch`), the shared `catchError` mapping for `EngineError`, the idempotency helper, `task_runs` mirroring, and stub task files for every task in pipeline.md (each throws "not implemented") so workers fill bodies in their own files |
| `src/core/**` | `db.types.ts`, `json-schemas.ts`, `pricing.ts` (interface plus price table with values from engines.md, flagged unverified for OpenAI), `cost.ts`, `ids.ts` (token generation), `normalize.ts` (domain, email and URL normalization), `env.ts` (zod-validated env), `supabase.ts` (server, service and browser clients) |
| `src/engines/types.ts`, `src/engines/registry.ts`, `src/engines/http.ts`, `src/engines/mock.ts` | The adapter interface, `EngineError`, the fetch wrapper, and `MockAdapter` with manifest plus synthetic fallback |
| `src/llm/client.ts`, `src/llm/mock.ts`, `src/llm/log.ts` | Anthropic client wrapper (`messages.parse` plus `zodOutputFormat`, one validation retry, `llm_calls` logging) and `MockLlm` |
| `src/pipeline/runner.ts` | The in-process fake trigger runner for pipeline tests |
| `app/layout.tsx`, `app/(admin)/admin/layout.tsx`, `app/(admin)/admin/login/**`, `middleware.ts` | Root layout, admin shell with nav placeholders, magic-link login, middleware for admin auth and `/r/*` headers |
| `fixtures/brands.json`, `fixtures/scenario/**`, `fixtures/csv/**` | The shared scenario, which every workstream's tests use |

Exit criteria:
- CI is green.
- `supabase db reset` works, and the RLS tests pass.
- A trivial `probe.call` stub runs end to end with `MockAdapter` in the `runner`.
- Supabase advisors are clean.

## Parallel workstreams

### W1: Engine adapters, extraction and scoring (branch `feat/w1-engines-scoring`)
| Owns |
|---|
| `src/engines/openai.ts`, `src/engines/perplexity.ts`, `src/engines/gemini.ts`, `src/engines/mappers/**` |
| `src/extraction/**`, `src/matching/**`, `src/scoring/**` |
| `src/llm/prompts/extract.*`, `src/llm/schemas/extraction.ts` |
| `trigger/probe/**` (`cycle.attachProspect`, `cycle.run`, `probe.call`, `cycle.finalize`), `trigger/extract/**`, `trigger/score/**` |
| `fixtures/engines/**`, `fixtures/llm/extract/**` |
| `scripts/record-fixture.ts` |
| `tests/engines/**`, `tests/extraction/**`, `tests/matching/**`, `tests/scoring/**` |

Delivers:
- Three adapters, with Gemini behind the flag.
- Cost calculation per engine.
- The cycle orchestration with reuse, budget guard, completeness rule and retries.
- Extraction, reconciliation, the brand dictionary writes, and `ProspectMetrics`.

### W2: Company understanding, categories and prompt generation (branch `feat/w2-understanding`)
| Owns |
|---|
| `src/fetch/**` (PageFetcher, robots, readability) |
| `src/understanding/**`, `src/categorize/**`, `src/prompts/**` |
| `src/llm/prompts/{understand,categorize,category-prompts,brand-prompts}.*`, `src/llm/schemas/{understanding,categorize,prompts}.ts` |
| `trigger/understand/**` (`prospect.understand`, `prospect.categorize`, `category.generatePrompts`, `prospect.generateBrandPrompts`) |
| `fixtures/sites/**`, `fixtures/llm/{understand,categorize,category-prompts,brand-prompts}/**` |
| `tests/fetch/**`, `tests/understanding/**`, `tests/prompts/**` |

Delivers:
- Site fetching with the fallback hook.
- Understanding, with the prospect and competitor brand rows.
- Categorisation, including create, assign and merge-suggest.
- Prompt set generation and versioning (service functions: `createDraftVersion`, `editVersion`, `activateVersion` and `mergeCategories`, which W4's admin UI calls).
- Brand prompts.

### W3: Hooks and report pages (branch `feat/w3-hooks-report`)
| Owns |
|---|
| `src/hooks/**`, `src/report/**` (snapshot builder, fixes rules, shared report components in `src/report/components/**`) |
| `src/sources/**` (source page checks) |
| `src/llm/prompts/{hook,claim-check}.*`, `src/llm/schemas/{hook,claim-check}.ts` |
| `trigger/hooks/**` (`sources.check`, `hook.generate`, `report.build`) |
| `app/r/**`, `app/api/r/**` |
| `fixtures/sources/**`, `fixtures/llm/{hook,claim-check}/**` |
| `tests/hooks/**`, `tests/report/**`, `tests/e2e/report.spec.ts` |

Delivers:
- Source checks.
- Hook candidates, copy, validation and evidence linking.
- The snapshot, the public report page, the view beacon and the noindex headers.
- Report components exported for reuse by the admin (the grid, leaderboard and raw-answer viewer).

### W4: Import, admin and Instantly CSV export (branch `feat/w4-admin-export`)
| Owns |
|---|
| `src/import/**`, `src/export/**`, `src/suppression/**` |
| `trigger/import/**` (`import.commit`) |
| `app/(admin)/admin/**`, except `layout.tsx` and `login/**` (foundation) |
| `app/api/admin/**` |
| `src/admin/**` (server actions, query helpers, admin-only components) |
| `tests/import/**`, `tests/export/**`, `tests/suppression/**`, `tests/e2e/admin.spec.ts` |

Delivers:
- Every admin screen in admin.md.
- CSV import with mapping and flags.
- The suppression list.
- The approval queue, which calls W3's `approveHook`, `rejectHook` and `validateHook`.
- The Instantly CSV export with pre-flight.
- The costs screen.
- Until W1–W3 merge, it uses the seeded scenario data and the services' **interfaces** from the foundation stubs.

**Cross-workstream interfaces**, which the foundation defines as typed stubs in `src/core/services.ts`:
- `startCycleForCategory`, `retryFailedCells` (W1).
- `activatePromptSet`, `editPromptSet`, `reassignCategory`, `mergeCategories`, `editBrand`, `mergeBrands` (W2; brand edits trigger W1 re-scoring through the `rescoreProspects` stub).
- `approveHook`, `rejectHook`, `validateHook`, `buildReportSnapshot` (W3).
- `isSuppressed`, `buildInstantlyCsv` (W4).

## Step 5: Integration and live acceptance (one worker, branch `feat/w5-integration`)
- Merge W1–W4, regenerate the DB types and fix seams. No new features.
- Record real fixtures with `scripts/record-fixture.ts`, which needs keys, and replace the synthetic ones.
- Deploy: Vercel (the app at `visibility.studio5.in` plus the admin), Trigger.dev deploy, Supabase production migrations, and env vars.
- **Phase 1 acceptance:** run a real 10-prospect CSV in one category. It must produce:
  - report pages,
  - an Instantly CSV, and
  - an admin Costs → per-prospect table.

  Record the actual cost against the engines.md §6 estimate in STATUS.md.
- Hand-review every hook against its evidence before any export goes to Instantly.

## Rough sequencing
- Foundation: about 2–3 days.
- W1–W4 in parallel: about 4–6 days each.
- W5: about 2 days, plus the time waiting on keys and DNS.
