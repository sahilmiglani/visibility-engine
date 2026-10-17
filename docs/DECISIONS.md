# Decisions

Dated one-liners in the format `- YYYY-MM-DD — <decision> — <why>`. Append only. If a decision
reverses an earlier one, add a new line saying what it supersedes.

## From the kickoff brief

- 2026-10-17 — Build "Visibility Engine": one core with an outreach mode (Phases 1–2) and a client mode (Phase 3) — the pitch is a demonstration with real data, and the same engine becomes the paid audit and retainer tracking.
- 2026-10-17 — First campaigns target US B2B SaaS companies from seed to Series B, in 2–3 categories — a narrow ICP keeps prompts and competitor sets coherent.
- 2026-10-17 — Sending happens in Instantly; this app never sends email — deliverability and compliance tooling already exist there.
- 2026-10-17 — Report pages live at `visibility.studio5.in` by default, with a configurable base URL (`REPORT_BASE_URL`) — Studio5 brand trust without coupling to the main site.
- 2026-10-17 — Cold email goes out from separate sending domains, never studio5.in, and is configured in Instantly — protects the main domain's reputation; out of this codebase.
- 2026-10-17 — Import from Apollo or Clay CSV exports only, deduped on domain plus email; no scraping — keeps sourcing compliant and simple.
- 2026-10-17 — Generate category prompts per category (15–20, LLM-generated, editable, versioned) and brand prompts per prospect (3–5) — category answers can be reused across prospects, which cuts cost roughly N-fold.
- 2026-10-17 — Probe ChatGPT (OpenAI API with web search), Perplexity (Sonar) and Gemini (Google Search grounding), each prompt twice per cycle — captures variance; these are the three engines buyers use most.
- 2026-10-17 — All engines sit behind one adapter interface — so Claude, Google AI Overviews (through a SERP provider) or a consumer-app capture vendor can be added later.
- 2026-10-17 — Store the raw response, citations, model ID, parameters, timestamp, latency and cost for every engine call — every claim must be provable and every dollar attributable.
- 2026-10-17 — Hook priority: absence > competitor capture > source gap (confirmed by fetching the cited page) > inaccuracy (always human-verified) — strongest true finding first; inaccuracy claims are the riskiest.
- 2026-10-17 — Report pages use unguessable URLs, are noindex, and track views only as a count plus first and last viewed — shareable proof without exposure or personal data.
- 2026-10-17 — Phase 1 exports an Instantly-ready CSV; Phase 2 pushes through the Instantly API — ship the MVP without integration risk.
- 2026-10-17 — Guardrails: evidence IDs on every claim; "In our tests on {date}" framing; a suppression check before any export or push; US-only in Phase 1; no outbound email or social automation; secrets only from env vars; no live APIs in tests; cost logged per call, category run and prospect — non-negotiable per the brief.
- 2026-10-17 — Stack: Next.js App Router plus TypeScript on Vercel; Supabase (Postgres, auth, storage); Trigger.dev for jobs; Claude via the Anthropic API with zod-validated structured output — chosen in the brief.
- 2026-10-17 — Homepage fetching uses plain fetch plus readability first, behind a `PageFetcher` interface with Jina Reader or Firecrawl as a later fallback — cheap default with an escape hatch for JS-heavy sites.

## From spec writing (2026-10-17)

- 2026-10-17 — Keep Trigger.dev rather than Inngest or Supabase queues — its queues with `concurrencyLimit` give us per-engine throttling, waits are checkpointed (no compute billed while a parent waits on children), and idempotency keys are built in. No strong reason to switch.
- 2026-10-17 — A "prospect" is a company (keyed by domain); people are `contacts` under it. Hooks, scores and reports are per prospect; export rows and report links are per contact — analysis is company-level, while outreach and view attribution are person-level.
- 2026-10-17 — Each contact gets its own report link token pointing at the prospect's report — per-lead view attribution for Phase 2 webhooks without storing any viewer data.
- 2026-10-17 — Reports are rendered from a frozen JSON snapshot taken at approval time — the page can't drift from the evidence the hook was approved on.
- 2026-10-17 — Category answers are reused through `probe_cycles`: a prospect joins the category's latest cycle if it's younger than `CYCLE_REUSE_DAYS` (default 14); otherwise a new cycle starts — the reuse rule from the brief, made explicit.
- 2026-10-17 — Failed or empty engine calls count as "no data", never as "not mentioned"; a cycle needs at least 90% successful calls per engine before scoring — absence claims must not come from errors.
- 2026-10-17 — Prospect position and mention detection are computed by deterministic alias matching on the answer text; the LLM extractor supplies brand names, sentiment, recommendations and claims, and code checks that each snippet is a verbatim substring — numbers in hooks must be reproducible, not LLM judgement.
- 2026-10-17 — Prices live in code (`src/core/pricing.ts`) with a `PRICING_VERSION`; each call stores `cost_usd`, a cost breakdown and the pricing version (Perplexity's own returned cost is preferred when present) — testable and auditable; prices change rarely.
- 2026-10-17 — Category cycle cost is allocated to prospects equally across the prospects attached to that cycle — simple and explainable; per-prospect cost = own brand probes + own LLM calls + share of the cycle.
- 2026-10-17 — New category prompt sets start as drafts and need one admin click to activate before probing — prompt quality drives every finding sent to prospects (see open question Q2).
- 2026-10-17 — Anthropic models, configurable per step via env: `claude-haiku-5-5` for extraction, `claude-sonnet-5-5` for categorisation and claim checks, `claude-opus-5-5` for understanding, prompt generation and hooks — cost-scaled to each step's volume and quality needs. Use `output_config.format` with JSON schema (GA) through `zodOutputFormat` plus `messages.parse`.
- 2026-10-17 — Public report pages read data only through server-side code with the service role, looked up by token; the anon role has no table access at all — RLS stays simple (admin-only) and tokens are the only public capability.
- 2026-10-17 — View tracking uses a client-side beacon after the page is visible for 3 seconds, with no IP, user agent or cookies stored — email security scanners prefetch links, so counting server renders would inflate views.
- 2026-10-17 — Perplexity adapter targets the Agent API (`POST /v1/agent`, `model: "perplexity/sonar"` + `web_search` tool), not Sonar Chat Completions — Perplexity ended Sonar Chat Completions support on 2026-09-27 and recommends the Agent API for new projects; presets run on third-party models, so we pin Perplexity's own model (pending Q3).
- 2026-10-17 — Gemini adapter is built and fixture-tested but disabled by default (`ENGINES_ENABLED=openai,perplexity`) — Gemini API grounding terms restrict analysing grounded results and showing them to anyone but the requester, which conflicts with scoring + report pages (pending Q1).
- 2026-10-17 — Gemini grounding redirect URLs are never resolved server-side; Gemini citations are excluded from source-gap page checks — terms prohibit automated collection of Links.
- 2026-10-17 — OpenAI prices/limits recorded as unverified (official docs unreachable from the build environment); live acceptance is blocked until confirmed — no guessing on cost data (Q5).
- 2026-10-17 — Engine adapters use plain `fetch` against REST APIs rather than provider SDKs — exact request/response capture for evidence, fewer dependencies; Anthropic is the exception (`@anthropic-ai/sdk` for `messages.parse` + `zodOutputFormat`).
- 2026-10-17 — Probes send the prompt verbatim with no system prompt/instructions, web search on, US location — measure each engine's default behaviour, not ours.
- 2026-10-17 — Trigger.dev v4.7 `concurrency` option + named `concurrencyLimit`s per engine (not deprecated `queue({concurrencyLimit})`); idempotency keys created with explicit `scope: "global"` — SDK ≥ 4.3.1 defaults raw keys to run scope.
- 2026-10-17 — Phase 2 Instantly webhooks will be authenticated by a shared-secret custom header — Instantly's webhook API offers custom headers but no HMAC signing.
- 2026-10-17 — Summary score v1 = round(100 × (0.5·mention rate + 0.3·share of voice + 0.2·position score)) — simple, explainable, versioned (`score-v1`) so it can change without rewriting history.
