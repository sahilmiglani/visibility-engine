# Spec: Pipeline (jobs, triggers, retries, idempotency, concurrency, cost)

Trigger.dev **v4** (SDK `@trigger.dev/sdk`; the `/v3` import path is deprecated). Tasks live in
`trigger/`, and business logic lives in `src/` (see CLAUDE.md). Verified against trigger.dev/docs
on 2026-10-17. Sources are at the end.

## 1. Stage overview

```
CSV upload (server action)
  └─ import.commit ─────────────► for each new prospect: prospect.understand
prospect.understand   fetch site pages → LLM understanding → brands upsert → prospect.categorize
prospect.categorize   (global concurrency 1) LLM categorize → assign / create / merge-suggest
                      ├─ new category → category.generatePrompts → draft prompt set (admin activates)
                      └─ category has an active prompt set → cycle.attachProspect
cycle.attachProspect  join the reusable cycle or create a new one → prospect.generateBrandPrompts → cycle.run (if new)
cycle.run             (per cycle) fan out probe.call for category prompts plus attached prospects' brand prompts
probe.call            one engine call, persisted → extract.response
extract.response      LLM extraction + deterministic matching → mentions, citations
cycle.finalize        completeness check → for each prospect: prospect.score
prospect.score        metrics → sources.check (top sources) → hook.generate → report.build
report.build          snapshot + report_links per contact → prospect.stage = hook_ready (approval queue)
```

Every arrow is a `trigger()` or `batchTriggerAndWait()` call with an idempotency key. Admin actions
("re-run understanding", "retry failed calls", "regenerate hook") trigger the same tasks.

## 2. Tasks

| Task id | Payload (zod) | Does | Queue / concurrency | Retry | Idempotency key |
|---|---|---|---|---|---|
| `import.commit` | `{importId}` | Insert prospects and contacts from a parsed import, then trigger understanding for new prospects | per-task, total 1 | 3 | `import:{importId}` |
| `prospect.understand` | `{prospectId, force?}` | Fetch home, pricing and about pages (pages.md rules below), call the `understand` LLM, upsert the prospect's own brand and competitor brands, set `stage=understood` | `concurrencyLimit("llm-reason", total: 4)` | 3, exponential | `understand:{prospectId}:{siteHash or "v1"}` |
| `prospect.categorize` | `{prospectId}` | LLM categorize against the list of existing categories → assign, create or merge-suggest | **task total 1** (serialised so near-duplicate categories aren't created in parallel) | 3 | `categorize:{prospectId}:{understandingVersion}` |
| `category.generatePrompts` | `{categoryId, parentVersionId?}` | LLM generates 15–20 prompts → new `prompt_set_versions` row with `status=draft` | `llm-reason` | 3 | `genprompts:{categoryId}:{parentVersionId or "root"}` |
| `prospect.generateBrandPrompts` | `{prospectId}` | Fill templates (understanding-and-prompts.md), with the LLM picking the competitor and use case | `llm-reason` | 3 | `brandprompts:{prospectId}:{understandingVersion}` |
| `cycle.attachProspect` | `{prospectId}` | Find or create a cycle (§4), insert `cycle_prospects`, run brand prompts if the cycle is already running or complete | per-task, total 1 per category (`concurrencyKey=categoryId`, `perKey: 1`) | 3 | `attach:{prospectId}:{categoryId}` |
| `cycle.run` | `{cycleId}` | Budget check, `batchTriggerAndWait` of `probe.call` for all pending cells, then `cycle.finalize` | per-task, `perKey: 1` keyed by `cycleId` | 2 | `cyclerun:{cycleId}:{runNumber}` |
| `probe.call` | `{cycleId, promptId, engine, sampleIndex}` | Run the adapter, persist `engine_calls`, trigger `extract.response` | **named limit per engine:** `engine-openai` (`OPENAI_CONCURRENCY`=4), `engine-perplexity` (=1), `engine-gemini` (=2) | 5 (see §5) | `probe:{cycleId}:{promptId}:{engine}:{sampleIndex}` |
| `extract.response` | `{engineCallId, extractorVersion}` | LLM extraction plus matching → `extractions`, `mentions`, `citations` | `concurrencyLimit("llm-extract", total: 8)` | 3 | `extract:{engineCallId}:{extractorVersion}` |
| `cycle.finalize` | `{cycleId}` | Wait until every succeeded call is extracted, apply the completeness rule (§6), set status, then `batchTrigger` `prospect.score` | per-task `perKey: 1` | 3 | `finalize:{cycleId}:{runNumber}` |
| `prospect.score` | `{prospectId, cycleId}` | Compute metrics → `prospect_scores`, then `sources.check`, then `hook.generate` | total 4 | 3 | `score:{prospectId}:{cycleId}:{scoringVersion}` |
| `sources.check` | `{prospectId, cycleId}` | Fetch the top N cited non-brand pages (OpenAI and Perplexity citations only), check them for prospect and competitor mentions → `source_checks` | `concurrencyLimit("page-fetch", total: 4)` | 3 | `sources:{prospectId}:{cycleId}` |
| `hook.generate` | `{prospectId, cycleId}` | Build candidates → LLM copy → validate → `hooks`, `hook_evidence` | `llm-reason` | 3 | `hook:{prospectId}:{cycleId}:{hookVersion}` |
| `report.build` | `{prospectId, cycleId}` | Build the draft snapshot plus `report_links` per non-suppressed US contact → `stage=hook_ready` | total 4 | 3 | `report:{prospectId}:{cycleId}` |

Notes:
- Payloads are defined with `schemaTask({ schema })`. **Trigger every task through the task object**
  (`probeCall.trigger(...)`), not by string ID, so the payload is validated before a run exists.
- **Concurrency.** Use the v4.7 `concurrency` option: `{ total }`, `{ perKey, total }` with
  `concurrencyKey` at trigger time, and shared named limits via `concurrencyLimit({ name, total })`
  (at most one inline plus two named limits per task). The older `queue({ concurrencyLimit })` is
  deprecated. **Queues must be defined ahead of time in v4.**
- **Rate limits.** Trigger.dev has no built-in time-based rate limiter. Per-engine throttling is
  concurrency plus 429 handling (§5), which is enough at our volume (Perplexity T0 is 1 QPS).
- Use the `batchTriggerAndWait` limit of **1,000 items per call**. A cycle has at most
  `(20 + 5×N) × 3 × 2` cells, so we chunk at 500.
- Never wrap `triggerAndWait` in `Promise.all`; use the batch variants.
- Waiting parents are checkpointed and release their concurrency slots.
- `trigger.config.ts` sets `maxDuration` (**required**, CPU seconds only; waits don't count) to 300 by default.
  `cycle.run` and `cycle.finalize` get 900. The default machine is `small-1x`.

## 3. Prospect stage machine

`imported → fetching → understood → categorized → probing → scored → hook_ready | no_finding → approved | rejected`
Side states are `error` (with `stage_error`) and `excluded` (non-US, suppressed or manual).
- Each task sets the stage at its start and end inside the same transaction as its writes.
- Admin actions can move a prospect back: "re-understand" → `imported`; "re-categorize" →
  `understood`; "re-probe" → attach to a new cycle.
- **Non-US prospects** (country flag `non_us`) are marked `excluded` at import and don't enter the pipeline
  unless the admin overrides. Contacts flagged `non_us` under a US prospect stay in the DB but are
  skipped at export.

## 4. Probe cycles and category reuse

- `cycle.attachProspect(prospect)` looks for the newest cycle for the prospect's category where:
  - `status in (running, extracting, scoring, complete, partial)`,
  - `prompt_set_version_id = category.active_prompt_set_id`,
  - `created_at > now() - CYCLE_REUSE_DAYS` (default 14), and
  - `engines` equals `ENGINES_ENABLED`.
  - **If one is found:** insert into `cycle_prospects`, generate brand prompts, and trigger `probe.call` for the prospect's brand cells only
    (category cells are reused). Then trigger `prospect.score` for the prospect once its cells are extracted. A per-prospect
    mini-finalize uses the same completeness rule, scoped to brand cells plus the cycle's category cells.
  - **If none is found:** create a new `probe_cycles` row (snapshotting `engine_config` and setting `test_date = current_date`, UTC),
    attach the prospect, and trigger `cycle.run`.
- **Batching new imports.** `import.commit` delays `cycle.run` by `CYCLE_START_DELAY` (default 10 min,
  using the trigger `delay` option with idempotency key `cyclerun:{cycleId}:1`). Prospects from the same import that finish
  categorisation inside that window join the same cycle before category prompts fan out.
- **A category without an active prompt set:** prospects wait in `categorized`. Activating a prompt
  set (admin) triggers `cycle.attachProspect` for every waiting prospect in that category.

## 5. Retries and error handling

`probe.call` retry config: `{ maxAttempts: 5, factor: 2, minTimeoutInMs: 2_000, maxTimeoutInMs: 120_000, randomize: true }`.

- Adapters throw `EngineError` (engines.md §0.1). The task's `catchError` handler maps it like this:
  - Retryable, with `retryAfterMs` set → return `{ retryAt: new Date(now + retryAfterMs) }`. Trigger.dev's
    `retry.fetch` doesn't parse the standard `Retry-After` header, so we do it ourselves.
  - Retryable, without it → default backoff.
  - `bad_request` or `auth` → `{ skipRetrying: true }`. Write `engine_calls.status = failed` with the error.
  - `quota_exhausted` → `{ skipRetrying: true }`. Mark failed, set the `settings` key `engine_paused:{engine} = true`,
    and log loudly. While paused, `cycle.run` stops scheduling cells for that engine and shows a banner in the admin.
- **Final failure** (attempts exhausted) is caught in the task's `onFailure` hook, which writes the
  `engine_calls` row with `status=failed`, `attempts`, `error_code` and `error_message`. **Every attempted cell
  ends with exactly one row.**
- **LLM steps:**
  - SDK `maxRetries: 2` handles 429, 529 and 5xx inside the attempt.
  - A schema validation failure retries once with a "your previous output was invalid: {zod issues}" follow-up. After that the status is `invalid_output` and the task fails, so a human looks at it.
  - A 429 spend-limit error (`enforced_spend_limit_reached`) → `skipRetrying` and pause LLM tasks.
- **Non-retryable task errors** (bad data, missing prospect) throw `AbortTaskRunError`.

## 6. Idempotency

- Every trigger passes `idempotencyKey: await idempotencyKeys.create(key, { scope: "global" })`
  using the keys from §2. **Scope `global` is explicit**, because in SDK ≥ 4.3.1 raw strings default to `run`
  scope. The TTL is the default 30 days. Re-triggering a key that's already active or succeeded returns the existing
  run. A failed run's key is cleared automatically, so "retry failed" just re-triggers.
- Idempotency keys stop duplicate *triggers*. They don't stop duplicate *side effects* on retry. So:
  - `engine_calls.idempotency_key` is **unique in the DB**. `probe.call` first checks for a `succeeded` row
    with that key and returns early if one exists. The write is an upsert on the key.
  - `extractions` are unique on `(engine_call_id, extractor_version)`. `hook.generate` supersedes earlier
    candidates for the same `(prospect, cycle)` in one transaction.
  - Provider calls aren't idempotent. A retry after a timeout may mean a provider call we paid for and
    never stored. We accept that small cost leak and log `attempts`.
- A **re-run** (admin "retry failed cells") sets `runNumber = previous + 1` in `cycle.run`'s key and
  re-triggers only cells with no `succeeded` row.

## 7. Completeness rule (errors are not absences)

- For each engine in a cycle: `success_rate = succeeded / (succeeded + failed + empty)` over category cells.
- `cycle.finalize` sets the status:
  - `complete` if every engine has `success_rate ≥ 0.9`.
  - `partial` if at least one engine reaches 0.9. Engines below 0.9 are **excluded from scoring and
    hooks for that cycle**, and the report shows them as "not tested".
  - `failed` if no engine reaches 0.9.
- In metrics, a missing cell is "no data". A prompt and engine pair needs at least one succeeded sample to be scored.

## 8. Budget guard

- Before fan-out, `cycle.run` computes `estimated = Σ cells × adapter.estimateCost()` plus the
  extraction estimate. If `estimated > budget_usd` (`MAX_CYCLE_COST_USD`, default $30, editable in
  settings), it doesn't start. The cycle is set to `pending` with a `budget_exceeded` note, and the admin must confirm.
- During the run, `probe.call` checks the running `actual_cost_usd`. If it passes `1.5 × budget`,
  remaining cells skip with `status=failed`, `error_code=budget_stop`.

## 9. Cost logging

- Each engine call row stores `cost_usd`, `cost_breakdown`, `pricing_version` and token counts. Each Anthropic call
  row (`llm_calls`) does the same.
- `src/core/cost.ts` provides `recordEngineCost(...)` and `recordLlmCost(...)`. These are the only writers.
- **Per category run:** the `v_cycle_costs` view, plus `probe_cycles.actual_cost_usd` updated by `cycle.finalize`.
- **Per prospect:** the `v_prospect_costs` view (data-model.md): own costs plus `category cost / attached prospects`.
  The allocation is recalculated when a prospect joins an existing cycle, so older prospects' shares go down. The admin
  shows the current allocation.
- **Per engine:** a group-by on `engine_calls.engine`.
- Logs: each task logs `{task, entityId, engine?, costUsd, latencyMs}` through `logger.info` for the Trigger.dev
  dashboard. Progress uses `metadata.set("progress", {done, total})` on `cycle.run`, so the admin
  Runs screen can show it through `useRealtimeRun`.

## 10. Site and source page fetching (`src/fetch/`)

- `PageFetcher` interface: `fetch(url) → {finalUrl, status, html, text, fetcher}`.
  - `native` uses `fetch` plus `@mozilla/readability` on `linkedom`.
  - `jina` and `firecrawl` are stubs behind env vars (`PAGE_FETCHER_FALLBACK=none|jina|firecrawl`, with `JINA_API_KEY` and `FIRECRAWL_API_KEY`).
- `native` rules:
  - User-Agent `Studio5VisibilityBot/1.0 (+{REPORT_BASE_URL}/bot)`.
  - 10s timeout, 5 redirects at most, 3 MB maximum body, `text/html` only.
  - Respect `robots.txt` for our UA and `*`, cached per host for 24h. A disallowed page is recorded as `blocked_by_robots`.
- **Prospect pages:**
  - Fetch the homepage. Find pricing and about links by anchor href or text, matching `/pricing|plans/i` and `/about|company|team/i` on the same registrable domain.
  - Fetch at most 2 more pages.
  - If the extracted text is under 500 characters (a JS-heavy site), try the fallback fetcher if one is configured. Otherwise mark `understanding.low_content = true`.
- **Source pages:** `normalized_url` is unique in `source_pages`. A page fetched within the last 7 days is reused.
  Gemini redirect URLs are never fetched (engines.md §3.2).

## 11. Environment variables (pipeline-relevant)

`TRIGGER_SECRET_KEY`, `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `ANTHROPIC_API_KEY`,
`OPENAI_API_KEY`, `PERPLEXITY_API_KEY`, `GEMINI_API_KEY`, `ENGINES_ENABLED`, `ENGINE_MODE`
(`live|mock`), `LLM_MODE` (`live|mock`), `OPENAI_MODEL`, `PERPLEXITY_MODEL`, `GEMINI_MODEL`,
`GEMINI_GROUNDING`, `*_CONCURRENCY`, `LLM_MODEL_EXTRACT|CLASSIFY|REASON`, `SAMPLES_PER_PROMPT`
(2), `CYCLE_REUSE_DAYS` (14), `CYCLE_START_DELAY` (`10m`), `MAX_CYCLE_COST_USD` (30),
`PAGE_FETCHER_FALLBACK`. The full list is in `.env.example`.

## Sources (checked 2026-10-17)
https://trigger.dev/docs/triggering · https://trigger.dev/docs/queue-concurrency ·
https://trigger.dev/docs/queues · https://trigger.dev/docs/errors-retrying ·
https://trigger.dev/docs/idempotency · https://trigger.dev/docs/tasks/schemaTask ·
https://trigger.dev/docs/config/config-file · https://trigger.dev/docs/runs/max-duration ·
https://trigger.dev/docs/limits · https://trigger.dev/pricing ·
https://trigger.dev/docs/guides/frameworks/nextjs · https://trigger.dev/docs/runs/metadata ·
https://trigger.dev/docs/migrating-from-v3
