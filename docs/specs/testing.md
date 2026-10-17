# Spec: Testing

**Rule: tests never call live APIs.** CI has no provider keys. MSW runs with
`onUnhandledRequest: "error"` in every Vitest setup, and a global `fetch` guard rejects any host other than
`localhost`, `127.0.0.1` and the local Supabase. `ENGINE_MODE=mock` and `LLM_MODE=mock` are forced by
`vitest.setup.ts`.

## 1. Layers

| Layer | Tool | Scope | Runs in CI |
|---|---|---|---|
| Unit | Vitest | Pure functions: mappers, matcher, metrics, hook rules, validators, CSV parse and export, pricing | ✅ |
| Integration | Vitest plus local Supabase (`supabase start`) | Service functions against a real Postgres with migrations, RLS policies and RPCs | ✅ (Supabase CLI in CI via `supabase/setup-cli`) |
| Pipeline | Vitest plus local Supabase | Task bodies called in-process through `src/pipeline/runner.ts` (a synchronous fake of `trigger`/`batchTriggerAndWait`), with mock engines and LLM | ✅ |
| E2E smoke | Playwright | `next build && next start` against seeded local Supabase: admin login, queue approve, export download, report page | ✅ |
| Live smoke | `scripts/live-smoke.ts` | The Phase 1 acceptance run with real keys | ❌ manual only, guarded by `LIVE_SMOKE=1` |

## 2. Fixtures (`fixtures/`)
```
fixtures/
  engines/
    openai/       *.json   raw /v1/responses bodies (recorded) + *.expected.json (normalized ProbeResult)
    perplexity/   *.json   raw /v1/agent bodies (+ one legacy /v1/sonar body)
    gemini/       *.json   raw generateContent bodies (grounded + ungrounded)
    errors/       429-openai.json, 529-anthropic.json, 400-gemini.json, quota-openai.json …
    manifest.json          (engine, promptHash, sample) → fixture file
  llm/
    understand/   <prospect-slug>.json      validated UnderstandingSchema outputs
    categorize/   …    category-prompts/ …    brand-prompts/ …
    extract/      <engine-call-fixture>.json
    claim-check/  …    hook/ …
    manifest.json          (purpose, inputHash) → fixture file
  sites/          <domain>/{home,pricing,about}.html   saved prospect pages
  sources/        <slug>.html                         saved cited pages (mention / no-mention cases)
  csv/            apollo-10.csv, clay-10.csv, messy.csv (dupes, non-US, bad emails, BOM, CRLF)
  brands.json     seed brand dictionary for the test category
  scenario/       acme-category.json  — the end-to-end scenario definition (below)
```
- **Recording** (`scripts/record-fixture.ts`): needs live keys. Run it manually, never in CI. It calls one engine with one
  prompt, strips headers, redacts IDs that could identify our account, and writes the raw body plus the expected
  normalized output, which a human reviews before committing.
- Phase 1 needs at least 3 recorded fixtures per engine (a list answer, a comparison answer and a refusal or empty
  answer) before W5. Until keys exist, W1 hand-writes fixtures from the documented response
  shapes in engines.md and marks them `"synthetic": true` in the manifest. They're replaced with recordings in W5.
- The test scenario uses fictional companies (e.g. "Acme Outbound", `acme-outbound.test`) and a fictional category.
  We never commit real prospects' data.

## 3. Mock adapter and mock LLM
- `MockAdapter` (engines.md §5): manifest lookup, then the real mapper, then the synthetic fallback. Failure injection uses
  `MOCK_ENGINE_FAILURES`.
- `MockLlm` (`src/llm/mock.ts`): the same interface as `src/llm/client.ts`.
  - Looks up `fixtures/llm/manifest.json` by `(purpose, inputHash)`.
  - On a miss, runs a purpose-specific deterministic generator. For example, `extract` uses the deterministic matcher output to build a schema-valid `ExtractionOutput`.
  - Validates every output with the real zod schema.
  - `MOCK_LLM_INVALID=extract:1` returns an invalid output once, to test the retry path.
- Both write `engine_calls` and `llm_calls` rows with the mock pricing tables, so cost views are tested.

## 4. Unit test checklist (must exist before each workstream's PR merges)
- **Engines (W1):**
  - Each mapper turns its fixtures into the expected `ProbeResult`.
  - Error mapping for 429 with `Retry-After`, 429 quota, 400, 401, 5xx and timeout.
  - Gemini byte→UTF-16 offset conversion, with emoji and accented-character fixtures.
  - The Perplexity `[n]` → result-id mapping.
  - Cost calculation for each engine, including Gemini's price change on 2027-01-01.
- **Matching (W1):** normalization table tests, ambiguous names ("Close", "Front"), possessives and plurals,
  overlapping aliases (longest wins), and domain matching with subdomains.
- **Extraction (W1):** snippet verification and fuzzy repair, hallucinated-brand drop, deterministic-only
  prospect mention flagged, refusal handling.
- **Scoring (W1):**
  - Golden-file test: scenario responses produce an exact `ProspectMetrics` JSON.
  - Formulas checked against hand-computed small cases.
  - No-data cells are excluded from denominators.
- **Understanding and prompts (W2):** schema validation and the retry path, the evidence-quote substring check, the category
  name collision rule, prompt de-duplication, the intent mix check, and brand template filling, including the ambiguous-name form.
- **Hooks (W3):** each rule's qualify and not-qualify boundary cases, priority selection, the validator (numbers, brands,
  framing phrase, banned phrases), and the template fallback. **Property test:** a hook is never `pending_review` without evidence.
- **Report (W3):** snapshot zod round-trip; the noindex meta tag and header; the beacon route returns 204 for unknown tokens; bot UAs aren't counted.
- **Import, export and suppression (W4):**
  - Apollo and Clay header auto-mapping, plus the messy CSV flags.
  - Dedupe on domain plus email.
  - Non-US detection.
  - CSV injection escaping.
  - The Instantly column set, with header length ≤ 20.
  - Suppressed contacts never appear in an export. This is a property test over random suppression sets.
- **RLS (foundation):**
  - The anon key can't select from any table (a loop over all tables).
  - An authenticated non-admin can't select.
  - An admin can.
  - The RPCs aren't executable by anon.

## 5. End-to-end smoke test (`tests/pipeline/e2e.test.ts`)
The scenario in `fixtures/scenario/acme-category.json`:
- 10 prospects in one category (fixture CSV), 18 category prompts and 4 brand prompts each.
- Engines: openai, perplexity and gemini (all mock).
- Prospects are designed so that each hook type and `no_finding` appear at least once:
  - Prospect A: absent everywhere.
  - Prospect B: captured on "versus".
  - Prospect C: a source-gap page fixture.
  - Prospect D: a pricing contradiction.
  - Prospect E: strongly visible, so `no_finding`.
  - One contact is suppressed and one is non-US.

Steps, with assertions:
1. `importCsv(apollo-10.csv)`: 10 prospects, 11 contacts, 1 non-US flagged, 1 suppressed flagged.
2. Run the pipeline through `runner` until idle:
   - Every prospect is `understood` and categorised into **one** category.
   - The prompt set is a draft, so the pipeline waits. The test then calls `activatePromptSet`.
3. The cycle completes:
   - `engine_calls` = 18×3×2 category calls plus 10×4×3×2 brand calls.
   - Each call has `cost_usd > 0`, `latency_ms`, `model_returned` and `raw_response`.
4. Injected 429s on OpenAI recover through retry. One injected permanent 400 leaves a `failed` row that
   counts as no data. The cycle is `complete` because the success rate is still ≥ 90%.
5. Hooks:
   - A, B, C and D get the expected hook types, D with `needs_verification`. E is `no_finding`.
   - Every hook text contains "In our tests on".
   - Every hook has primary evidence that resolves to existing `engine_calls`.
6. Reports:
   - Each `hook_ready` prospect has a draft report and one link per eligible contact.
   - The snapshot parses.
7. Approve A, B and C. D can't be approved until it's verified; verify it, then approve.
8. Export:
   - The CSV equals the golden file `fixtures/scenario/expected-instantly.csv` (row order by company).
   - The suppressed and non-US contacts are absent.
9. Costs: `v_prospect_costs` has a row for every prospect, and the sum of allocations equals the cycle's category cost to within $0.000001.
10. Idempotency: running the whole pipeline again creates no new `engine_calls`.

Playwright (`tests/e2e/`):
- Log in with a seeded admin and magic-link bypass (test-only Supabase admin session generation).
- Open the queue, approve one prospect, export, download the CSV.
- Open `/r/{token}`. Check `X-Robots-Tag` and the meta robots tag. Check that `view_count` becomes 1 after the beacon fires (using a fake timer of 3s),
  and stays 1 after a reload in the same session.

## 6. CI (`.github/workflows/ci.yml`)
- Run on `push` and `pull_request`: `pnpm install --frozen-lockfile`, `pnpm typecheck`, `pnpm lint`, `supabase start`,
  `pnpm db:reset`, `pnpm test`, `pnpm build`, `pnpm test:e2e`.
- Required checks before merge: typecheck, lint, unit and integration tests. Playwright joins once W3 and W4 land.
- CI has no provider secrets at all. A test that needs a key is a bug.
