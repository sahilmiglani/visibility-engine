# Spec: Engines (adapter interface and providers)

**Verified on 2026-10-17** against official docs. Each fact is tagged:

- **[docs]** — read on an official docs page, linked in Sources.
- **[sdk]** — read in the provider's official SDK source on GitHub. Its types are generated from the API spec.
- **[unverified]** — seen only in search-result snippets of official pages we could not load. Must be confirmed before the live run (see the build-plan open questions).

> ⚠️ **Three findings change the brief's assumptions.** Each is an open question in `build-plan.md`.
> 1. **Gemini grounding terms.** They prohibit analysing grounded results and allow display only "to the end user who submitted the prompt" (§3.5). That conflicts with scoring answers and showing them on a prospect's report. Until resolved, the Gemini adapter is built but **disabled by default**.
> 2. **Perplexity Sonar is legacy.** "Sonar Chat Completions support ended on September 27, 2026." New projects should use the **Agent API** (`POST /v1/agent`). We spec the Agent API with model `perplexity/sonar` plus the `web_search` tool (§2).
> 3. **OpenAI docs could not be loaded.** `developers.openai.com` and `openai.com` are unreachable from this build environment (DNS failure or proxy 403). The request and response shapes below come from OpenAI's official Python SDK types. **Prices, rate limits and the recommended model are unverified.**

---

## 0. Adapter interface

Location: `src/engines/types.ts`. Every engine implements it, including `mock` (now) and `claude`,
`google_aio` (via a SERP provider) and `consumer_capture` (later).

```ts
export type EngineId = "openai" | "perplexity" | "gemini" | "mock"; // later: "claude" | "google_aio" | "consumer_capture"

export interface EngineConfig {
  engine: EngineId;
  model: string;                 // e.g. "gpt-6.1-sol"
  params: Record<string, unknown>; // engine-specific, normalized (search_context_size, reasoning effort, …)
  userCountry: "US";             // Phase 1: always US
  timeoutMs: number;             // default 90_000
}

export interface ProbeRequest {
  promptText: string;            // exact text, sent as a single user turn; no system prompt (see §0.2)
  config: EngineConfig;
  idempotencyKey: string;        // `${cycleId}:${promptId}:${engine}:${sampleIndex}`
}

export interface NormalizedCitation {
  position: number;              // 1-based order in the provider's list
  url: string;                   // as returned (may be a redirect for Gemini)
  title: string | null;
  inline: boolean;               // referenced inside the answer text
  spans?: { start: number; end: number }[]; // char offsets into answerText (UTF-16), when provided
}

export interface ProbeResult {
  status: "succeeded" | "empty";
  answerText: string;            // plain text; inline markers like [1] kept as-is
  citations: NormalizedCitation[];
  sources: { url: string; title: string | null }[]; // full consulted list when available (superset of citations)
  searchQueries: string[];
  modelReturned: string;
  providerResponseId: string | null;
  usage: { inputTokens: number; outputTokens: number; reasoningTokens: number; cachedInputTokens: number; searchCalls: number };
  cost: CostBreakdown;           // see §0.3
  requestBody: unknown;          // exactly what was sent (no secrets)
  rawResponse: unknown;          // full provider JSON
  httpStatus: number;
  latencyMs: number;
}

export class EngineError extends Error {
  constructor(
    message: string,
    public kind: "rate_limited" | "overloaded" | "timeout" | "bad_request" | "auth" | "quota_exhausted" | "server" | "network",
    public retryable: boolean,
    public retryAfterMs?: number,
    public httpStatus?: number,
    public rawBody?: unknown,
  ) { super(message); }
}

export interface EngineAdapter {
  id: EngineId;
  adapterVersion: string;        // bump when the mapping changes; stored on engine_calls
  defaultConfig(): EngineConfig; // from env (models, params)
  probe(req: ProbeRequest): Promise<ProbeResult>; // throws EngineError
  estimateCost(config: EngineConfig): number;     // USD per call, for budget checks
}
```

Registry: `src/engines/registry.ts` exports `getAdapter(id)`. Which engines run is set by
`ENGINES_ENABLED` (default `openai,perplexity`; see the Gemini warning). `ENGINE_MODE=mock`
replaces every adapter with `MockAdapter`. Tests always run that way.

### 0.1 Shared behaviour (`src/engines/http.ts`)
- Plain `fetch` against the REST APIs (no provider SDKs), so request and response bodies are captured exactly and dependencies stay small.
- `AbortController` timeout (`timeoutMs`). Latency is measured around the fetch only.
- Errors map to `EngineError`:
  - 429 → `rate_limited`, retryable; `retryAfterMs` comes from `Retry-After` or the provider's reset headers.
  - 408 → `timeout`, retryable.
  - 5xx and 529 → `server` or `overloaded`, retryable.
  - 400 and 422 → `bad_request`, not retryable.
  - 401 and 403 → `auth`, not retryable.
  - 402, or a spend-limit 429 → `quota_exhausted`, not retryable, and it also pauses the engine's queue (see pipeline.md).
- `answerText` is empty after trimming → `status: "empty"`. The pipeline stores it, but it counts as **no data**, not as "not mentioned".
- Secrets never appear in `requestBody`. Auth goes in headers only, and headers are not stored.

### 0.2 Prompting rules (all engines)
- Send the prompt text **exactly** as stored. Add no system prompt or instructions that could bias
  brand selection. We want each engine's default behaviour for an ordinary US user.
- Web search on, user location approximate with country US, default search context size.
- No domain filters.
- Each sample is an independent request: no conversation state and no `previous_response_id`.

### 0.3 Cost model (`src/core/pricing.ts`)
```ts
export const PRICING_VERSION = "2026-10-17";
export interface CostBreakdown { input: number; output: number; search: number; requestFee: number; total: number; source: "computed" | "provider"; }
```
- Each engine and model has per-1M-token input, cached-input and output rates, plus a search fee rule.
  Prices with a known future change carry an `effectiveFrom` date. Gemini 3.8 Flash's price rises on 2027-01-01.
- If the provider returns its own cost (Perplexity `usage.cost`), we store it with `source: "provider"`.
  We also compute our own estimate and log any drift above 10%.
- Free allowances (for example Gemini's 5,000 free grounded queries a month) are **ignored**. We
  always log list price, so cost per prospect stays comparable.

---

## 1. OpenAI (ChatGPT proxy): Responses API with `web_search`

| Item | Value | Tag |
|---|---|---|
| Endpoint | `POST https://api.openai.com/v1/responses` | [sdk] |
| Auth | `Authorization: Bearer $OPENAI_API_KEY` | [sdk] |
| Tool | `{"type": "web_search"}` (pinned version `web_search_2025_08_26`). The legacy `web_search_preview` lacks `filters`; don't use it. | [sdk] |
| Default model | `OPENAI_MODEL=gpt-6.1-sol` with `reasoning.effort: "low"` | [sdk] for the ID; [unverified] that it supports web search and is the right ChatGPT proxy |
| Alternatives | `gpt-6-astra` (flagship), `gpt-6-luna` (cheap; web search support unconfirmed). `gpt-5`, `gpt-5-mini` and `o3` shut down on 2026-12-11. | [sdk] |
| Chat Completions | `web_search_options` exists on `/v1/chat/completions` with search models. We don't use it; the Responses API is recommended. | [sdk] / [unverified] |

### 1.1 Request
```json
{
  "model": "gpt-6.1-sol",
  "input": "What are the best sales engagement platforms for seed-stage startups?",
  "tools": [{ "type": "web_search", "search_context_size": "medium",
              "user_location": { "type": "approximate", "country": "US" } }],
  "tool_choice": "auto",
  "include": ["web_search_call.action.sources"],
  "reasoning": { "effort": "low" },
  "store": false
}
```
- `search_context_size` is `low`, `medium` or `high`; the default is `medium`. [sdk]
- If `user_location` is omitted it defaults to the US. We set it explicitly anyway. [sdk]
- `tool_choice: "auto"` matches ChatGPT, which decides itself whether to search. If an answer ran no search, we record `searchCalls = 0`. It still counts as valid data. [sdk]
- `temperature` and `top_p` are reportedly rejected unless reasoning effort is `none`, and `gpt-6.1-sol` reportedly doesn't support `none`. So we send neither and rely on the two samples to capture variance. There is no `seed` parameter. [unverified]
- `store: false`: OpenAI doesn't keep the response, and we store the raw JSON ourselves. [sdk]

### 1.2 Response mapping
```json
{ "id": "resp_…", "model": "gpt-6.1-sol-2026-09-29",
  "output": [
    { "type": "reasoning" },
    { "type": "web_search_call", "id": "ws_…", "status": "completed",
      "action": { "type": "search", "query": "…", "sources": [{ "type": "url", "url": "https://…" }] } },
    { "type": "message", "role": "assistant",
      "content": [{ "type": "output_text", "text": "…",
        "annotations": [{ "type": "url_citation", "url": "https://…", "title": "…", "start_index": 120, "end_index": 180 }] }] } ],
  "usage": { "input_tokens": 0, "input_tokens_details": { "cached_tokens": 0 },
             "output_tokens": 0, "output_tokens_details": { "reasoning_tokens": 0 } } }
```
| ProbeResult field | Source |
|---|---|
| `answerText` | All `output[type=message].content[type=output_text].text`, joined with `\n\n` |
| `citations` | `annotations[type=url_citation]`, deduped by URL in first-seen order; `inline: true`; spans from `start_index`/`end_index` (offset by the join) |
| `sources` | Union of `output[type=web_search_call].action.sources[].url` (only returned with `include`) |
| `searchQueries` | `action.query` and `action.queries[]` from `search` actions. `open_page` and `find_in_page` actions aren't queries. |
| `usage.searchCalls` | Count of `web_search_call` items whose action type is `search` (each one is billed) |
| `modelReturned` | `model` (a dated snapshot) |
| `usage.*` | `input_tokens`, `cached_tokens`, `output_tokens`, `reasoning_tokens` |
| status `empty` | No `output_text`, or response `status` is `incomplete` with no text |

### 1.3 Pricing (**[unverified]**, must be confirmed)
| Model | Input / 1M | Cached input / 1M | Output / 1M |
|---|---|---|---|
| gpt-6.1-sol | $2.00 | $0.10 | $10.00 |
| gpt-6-astra | $10.00 | $1.00 | $50.00 |
| gpt-6-luna | $0.10 | $0.01 | $0.50 |

- Web search (`web_search`, all models): **$10 per 1K calls**, plus search content tokens billed at the model's input rate. [unverified]
- Reasoning tokens bill as output.
- Cost per call = `input × in_rate + cached × cached_rate + (output + reasoning) × out_rate + searchCalls × 0.010`.
- Rough estimate: about **$0.03–0.06 per call**.

### 1.4 Rate limits [unverified]
- Usage tiers are reportedly now Build, Launch and Grow, limited on RPM, RPD, TPM and TPD.
- Headers: `x-ratelimit-remaining-requests`, `x-ratelimit-reset-requests` (e.g. `6m0s`), `x-ratelimit-remaining-tokens`, `Retry-After`. The adapter parses the reset durations for `retryAfterMs`.
- A hard spend limit returns 429. The adapter maps a 429 whose error code is `insufficient_quota` to `quota_exhausted`.
- **Our throttle:** queue concurrency `OPENAI_CONCURRENCY` (default 4). Check the real limits under Console → Settings → Limits.

---

## 2. Perplexity: Agent API with `perplexity/sonar` and `web_search`

| Item | Value | Tag |
|---|---|---|
| Status of Sonar | "Sonar Chat Completions support ended on September 27, 2026. Synchronous and streaming requests keep working: they are being reformulated as Agent API requests." Async Sonar is gone. | [docs] |
| Endpoint | `POST https://api.perplexity.ai/v1/agent` (Open Responses-compatible: `input` in, typed `output[]` out) | [docs] |
| Auth | `Authorization: Bearer $PERPLEXITY_API_KEY` | [docs] |
| Default model | `PERPLEXITY_MODEL=perplexity/sonar` ("Perplexity's grounded search model") with `tools: [{type: "web_search"}]` | [docs] |
| Why not a preset | Presets (`fast`, `low`, …) run on third-party models; the `fast` example returns `"model": "openai/gpt-5.6-luna"`. A Perplexity probe should use Perplexity's own model. | [docs] |
| Legacy fallback | `POST /v1/sonar` with `model: "sonar-pro"` still answers (reformulated). It's kept as `PERPLEXITY_API_FLAVOR=sonar_legacy` but is not the default. | [docs] |

### 2.1 Request
```json
{
  "model": "perplexity/sonar",
  "input": "What are the best sales engagement platforms for seed-stage startups?",
  "tools": [{ "type": "web_search", "user_location": { "country": "US" } }]
}
```
- `web_search` options: `search_type` (`web` default, or `fast`), `max_results` (1–50), `filters.search_domain_filter` (up to 20 entries, allowlist or denylist), `user_location` (country, region, city, latitude, longitude). We set only `user_location.country`. [docs]
- No `instructions`, per §0.2. (Perplexity's own examples add "Search for current … information"; we don't.)

### 2.2 Response mapping
```json
{ "id": "resp_…", "model": "perplexity/sonar", "status": "completed",
  "output": [
    { "type": "search_results", "results": [ { "id": 1, "title": "…", "url": "https://…", "snippet": "…", "date": "2026-06-14", "last_updated": "2026-06-21", "source": "web" } ] },
    { "type": "message", "role": "assistant", "content": [ { "type": "output_text", "text": "… platforms[1][3] …", "annotations": [] } ] } ],
  "usage": { "input_tokens": 20, "output_tokens": 222, "cost": { "currency": "USD", "input_cost": 0.00008, "output_cost": 0.00444, "total_cost": 0.00452 } } }
```
| ProbeResult field | Source |
|---|---|
| `answerText` | `output[type=message].content[type=output_text].text`. `[n]` markers stay in the text. |
| `citations` | `output[type=search_results].results[]`, ordered by `id`. `inline: true` if `[id]` appears in the text. "The `[n]` maps to a result `id`." Any `url_citation` annotations present are merged in. [docs] |
| `sources` | All search results |
| `usage.searchCalls` | `usage.tool_calls_details.search_web` when present, otherwise the count of `search_results` items [docs] |
| `cost` | `usage.cost.total_cost` → `source: "provider"`. We also compute our own estimate. [docs] |
| `modelReturned` | `model` (store it; the Agent API can fall back between models) |

**Legacy `/v1/sonar` mapping** (fallback only): `choices[0].message.content`. Citations come from
`search_results[]` (title, url, date), with `citations[]` (URL strings) as a fallback. The docs
contradict each other on whether `citations` was removed, so parse both. Cost comes from
`usage.cost.total_cost`. [docs]

### 2.3 Pricing [docs]
- `perplexity/sonar`: **$0.25 per 1M input, $2.50 per 1M output, $0.0625 per 1M cache read**.
- `web_search` costs **$2.50 per 1K invocations**, or $1.00 with `search_type: "fast"`.
- Rough estimate: about **$0.005–0.01 per call**.
- Legacy Sonar, for reference: `sonar` costs $1/$1 per 1M tokens plus request fees of $5/$8/$12 per 1K (low/medium/high context). `sonar-pro` costs $3/$15 plus $6/$10/$14.

### 2.4 Rate limits [docs]
- Usage tiers depend on lifetime credit purchases: T0 $0, T1 $50, T2 $250, T3 $500, T4 $1,000, T5 $5,000.
- Agent API: **T0 1 QPS, T1 3 QPS, T2 8, T3 17, T4–5 33** (rolling 1-second window per organisation). A 429 means back off with jitter.
- **Our throttle:** concurrency `PERPLEXITY_CONCURRENCY` (default 1). With latency around 3–8s this stays under 1 QPS.
  Raise it once we're on T1 or above.

### 2.5 API vs the consumer app [docs]
"The API uses the same search system as the UI with differences in configuration—so their outputs
may differ," and "the underlying AI model might differ." This supports our "In our tests" framing.

---

## 3. Gemini: `generateContent` with Google Search grounding (⚠️ disabled by default)

| Item | Value | Tag |
|---|---|---|
| Endpoint | `POST https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent` | [docs] |
| Auth | `x-goog-api-key: $GEMINI_API_KEY` | [docs] |
| Tool | `tools: [{ "google_search": {} }]`. "Older models use a `google_search_retrieval` tool. For all current models, use the `google_search` tool." | [docs] |
| Default model | `GEMINI_MODEL=gemini-3.8-flash` (stable, "our most intelligent Flash model"). The cheaper option is `gemini-3.5-flash-lite`; the flagship is `gemini-4-argon`. | [docs] |
| API surface | The guides now default to the Interactions API (`/v1beta/interactions`). `generateContent` is still documented, and we use it because its grounding metadata is well specified. | [docs] |
| Free tier | Grounding is "Not available" on the free tier for Gemini 3+ models, so a billing-enabled project is required. | [docs] |

### 3.1 Request
```json
{ "contents": [{ "role": "user", "parts": [{ "text": "What are the best sales engagement platforms for seed-stage startups?" }] }],
  "tools": [{ "google_search": {} }],
  "generationConfig": { "thinkingConfig": { "thinkingLevel": "LOW" } } }
```
- `temperature`, `topP` and `topK` are "no longer supported and [are] silently ignored on Gemini 3.6 Flash and newer". We don't send them. [docs]
- `thinkingLevel` is MINIMAL, LOW, MEDIUM or HIGH (Gemini 3+). We use LOW, configurable as `GEMINI_THINKING_LEVEL`. [docs]
- There's no user-location parameter for grounding on `generateContent`. We note this in `params` as `location: "not_supported"`. [docs: absence]

### 3.2 Response mapping
```json
{ "candidates": [{ "content": { "parts": [{ "text": "…" }] }, "finishReason": "STOP",
    "groundingMetadata": {
      "webSearchQueries": ["best sales engagement platforms 2026"],
      "searchEntryPoint": { "renderedContent": "<style>…</style><div>…</div>" },
      "groundingChunks": [{ "web": { "uri": "https://vertexaisearch.cloud.google.com/grounding-api-redirect/…", "title": "outreach.io" } }],
      "groundingSupports": [{ "segment": { "partIndex": 0, "startIndex": 0, "endIndex": 85, "text": "…" },
                              "groundingChunkIndices": [0], "confidenceScores": [0.9] }] } }],
  "usageMetadata": { "promptTokenCount": 0, "candidatesTokenCount": 0, "thoughtsTokenCount": 0, "toolUsePromptTokenCount": 0 },
  "modelVersion": "…", "responseId": "…" }
```
| ProbeResult field | Source |
|---|---|
| `answerText` | Concatenated `candidates[0].content.parts[].text`, skipping thought parts |
| `citations` | `groundingChunks[].web` with `{uri, title}`, in index order. `inline: true` if any `groundingSupports` references the index. **Segment indices are in bytes, not characters**, so convert from UTF-8 bytes to UTF-16 offsets for `spans`. [docs] |
| `searchQueries` | `webSearchQueries[]` |
| `usage.searchCalls` | Count of non-empty `webSearchQueries` (the billing unit for Gemini 3+) |
| `usage` | `promptTokenCount + toolUsePromptTokenCount` → input; `candidatesTokenCount` → output; `thoughtsTokenCount` → reasoning |
| `modelReturned` | `modelVersion` |
| extra | `searchEntryPoint.renderedContent` is stored inside `rawResponse`. The ToS requires displaying it alongside grounded results. |

**Redirect URIs.** `groundingChunks[].web.uri` is a `vertexaisearch.cloud.google.com` redirect, and
`title` is usually the domain. No official doc explains how to resolve them. The terms forbid
"programmatic or automated means to collect Links". So **we do not resolve Gemini redirects
server-side**. Citation `domain` comes from `title` when it parses as a hostname, otherwise it's
`null`. Gemini citations are excluded from source-gap page fetches (hooks.md).

### 3.3 Pricing [docs]
| Model | Input / 1M | Output / 1M (incl. thinking) |
|---|---|---|
| gemini-3.8-flash | $0.75 until 2026-12-31, then **$1.50 from 2027-01-01** | $3.75, then **$7.50** |
| gemini-3.5-flash-lite | $0.30 | $2.50 |
| gemini-4-argon | $2.00 until 2026-11-30, then **$4.00 from 2026-12-01** | $10, then **$20** |

- Grounding for Gemini 3 and newer: "5,000 free search requests per month (shared across all Gemini 3 and newer models), then **$14 per 1,000** requests." The fee is charged per individual search query, so cost = `searchCalls × $0.014`.
- Gemini 2.5 is billed per grounded prompt ($35/1K) instead.
- Rough estimate: about **$0.02–0.04 per call**.

### 3.4 Rate limits [docs]
- The page gives no fixed per-model numbers; limits "can be viewed in Google AI Studio" and "are not guaranteed."
- Tier 1 (billing linked) has a $250 cap and a **$10 per 10-minute spend limit**.
- Requests per day reset at midnight Pacific. Over the limit returns 429 `RESOURCE_EXHAUSTED`.
- **Our throttle:** `GEMINI_CONCURRENCY` (default 2). We stay well under $10 per 10 minutes, since a 140-call batch costs about $4.

### 3.5 ⚠️ Terms-of-service constraint (blocks enabling Gemini)
From the Gemini API Additional Terms, "Grounding with Google Search" section (effective 2026-03-23, updated 2026-04-28) [docs]:
- You must "only display the Grounded Results with the associated Search Suggestion(s) to the end user who submitted the prompt."
- You must not "cache, frame, syndicate, resell, **analyze**, train on, or otherwise learn from Grounded Results or Search Suggestions." Limited storage of text is allowed only for narrow purposes, such as chat history or display optimisation for up to two years.
- You must not "modify, or intersperse any other content with" grounded results, or use automated means to collect Links.

**Impact.** Our core loop analyses grounded answers (extraction and scoring) and shows them to a third
party (the prospect's report). That appears to violate these terms. Options are in build-plan Q1:
1. Legal review or written permission from Google.
2. Gemini **without** grounding: a "what Gemini knows" probe. These answers aren't Grounded Results, so the general Gemini API terms apply.
3. Get Google answers through a SERP-data vendor (Google AI Overviews or AI Mode). This is already on the roadmap.
4. Drop Gemini for Phase 1.

The adapter is still built and fixture-tested. It runs only when `gemini` is in `ENGINES_ENABLED`, and
`GEMINI_GROUNDING=true|false` selects grounded or ungrounded mode.

---

## 4. Anthropic (LLM steps, plus a future Claude probe engine)

This isn't a probe engine in Phase 1, but every pipeline LLM step uses it. Client: `src/llm/client.ts`.

| Item | Value | Tag |
|---|---|---|
| Models | `claude-opus-5-5`, `claude-sonnet-5-5`, `claude-haiku-5-5`, `claude-fable-5-1`. Each alias equals its pinned ID. | [docs] |
| Structured output | GA, no beta header. Use `output_config: { format: { type: "json_schema", schema } }`. The TS SDK provides `zodOutputFormat()` (from `@anthropic-ai/sdk/helpers/zod`) with `client.messages.parse()`, returning `.parsed_output`; it throws `AnthropicError` on invalid output. `output_format` is deprecated. | [docs] |
| Schema limits | No recursion. No numeric, length or `minItems > 1` constraints (the SDK strips them, then checks them client-side). `additionalProperties: false`. At most 24 optional properties and 16 unions per request. Enum casing may vary, so compare case-insensitively. Check `stop_reason`, because `refusal` or `max_tokens` may not match the schema. | [docs] |
| Forced tools | Forced `tool_choice` (`any` or `tool`) is a 400 on Opus 5.5, Sonnet 5.5 and Fable 5.1, so use structured outputs. | [docs] |
| Thinking | Always on for Opus 5.5 and Fable 5.1. Tune it with `output_config.effort` (`low`, `medium`, `high`). Thinking tokens bill as output. | [docs] |
| Rate limits | Per model, RPM, ITPM and OTPM. Start tier: 1K RPM, 2M ITPM, 400K OTPM for Opus, Sonnet and Haiku 5.5. A 429 carries `retry-after`. A 429 with `error.details.error_code: "enforced_spend_limit_reached"` has no `retry-after`, so don't retry it. A 529 means overloaded; back off. The SDK retries 408, 409, 429 and 5xx (`maxRetries`). | [docs] |
| Batches | 50% off, up to 100K requests per batch, most finish within 1 hour, matched by `custom_id`. Structured outputs work. Not used in Phase 1 because latency matters for the review loop; it's a Phase 3 option for deep audits. | [docs] |

**Pricing per 1M tokens [docs]:**

| Model | Input | Output |
|---|---|---|
| Opus 5.5 | $4 | $20 |
| Sonnet 5.5 | $2 | $10 |
| Haiku 5.5 (prompt ≤ 100K tokens) | $0.10 | $0.50 |
| Fable 5.1 | $10 | $50 |

Cache reads cost about 5% of input. Setting `inference_geo: "us"` multiplies prices by 1.1×; we don't set it.

**Model per step** (env-overridable; see understanding-and-prompts.md):

| Step | Env var | Model | Effort |
|---|---|---|---|
| Extraction | `LLM_MODEL_EXTRACT` | `claude-haiku-5-5` | `low` |
| Categorisation, claim check, source check | `LLM_MODEL_CLASSIFY` | `claude-sonnet-5-5` | `low` |
| Understanding, prompt generation, hooks | `LLM_MODEL_REASON` | `claude-opus-5-5` | `medium` |

**Future Claude probe adapter** (not Phase 1) [docs]:
- Tool: `web_search_20260318` (also `web_search_20250305` and `web_search_20260209`), with `max_uses` and `user_location`.
- $10 per 1K searches plus tokens.
- Responses contain `web_search_tool_result` blocks with `web_search_result` entries (url, title, page_age), and text blocks carry `citations` of type `web_search_result_location` (url, title, cited_text).
- Errors come back as HTTP 200 with `web_search_tool_result_error`. Handle `stop_reason: "pause_turn"`.

---

## 5. Mock adapter (`src/engines/mock.ts`)
- `MockAdapter implements EngineAdapter` with `id = "mock"`. It can also stand in for any `EngineId`, so the scoring grid still has three columns in tests.
- Behaviour is deterministic and driven by `fixtures/engines/manifest.json`:
  - The manifest maps a `(engine, normalizedPromptText, sampleIndex)` hash to a fixture file containing a recorded raw response.
  - The adapter runs the **real** response mapper for that engine on the fixture, so mappers are tested end to end.
  - If nothing matches, it falls back to a synthetic answer generator. It's seeded by the hash and lists brands from `fixtures/brands.json`, with configurable inclusion of the prospect so tests can force absence or presence.
- Failure injection via `MOCK_ENGINE_FAILURES` (e.g. `openai:429:2` means the first two calls return 429) for retry tests.
- Costs come from the same pricing module, so the cost views are exercised.

## 6. Cost estimate for the Phase 1 acceptance run

Assumptions: 10 prospects, 1 category, 20 category prompts, 4 brand prompts each, 2 samples, OpenAI plus Perplexity (Gemini optional).

| Engine | Calls | ≈ $/call | ≈ total |
|---|---|---|---|
| OpenAI | 40 + 80 = 120 | 0.045 | $5.40 |
| Perplexity | 120 | 0.008 | $1.00 |
| Gemini (if enabled) | 120 | 0.03 | $3.60 |
| Extraction (Haiku) | 240–360 | 0.0006 | $0.20 |
| Understanding, prompts, hooks (Opus) | ~25 | 0.05 | $1.25 |

**Total: about $8 without Gemini, about $12 with it, or roughly $0.8–1.2 per prospect.** The category share is
about a third. Budget guard: `MAX_CYCLE_COST_USD` (default $30).

## Sources

Checked 2026-10-17. Loaded:
- Anthropic: https://platform.claude.com/docs/en/about-claude/models/overview.md, https://platform.claude.com/docs/en/about-claude/pricing.md, https://platform.claude.com/docs/en/build-with-claude/structured-outputs.md, https://platform.claude.com/docs/en/api/rate-limits.md, https://platform.claude.com/docs/en/api/errors.md, https://platform.claude.com/docs/en/build-with-claude/batch-processing.md, https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-search-tool.md, https://github.com/anthropics/anthropic-sdk-typescript/blob/main/helpers.md
- Perplexity: https://docs.perplexity.ai/docs/agent-api/migrate-from-sonar/overview.md, https://docs.perplexity.ai/docs/agent-api/migrate-from-sonar/how-to.md, https://docs.perplexity.ai/docs/agent-api/quickstart.md, https://docs.perplexity.ai/docs/agent-api/models.md, https://docs.perplexity.ai/docs/agent-api/presets.md, https://docs.perplexity.ai/docs/agent-api/tools/web-search.md, https://docs.perplexity.ai/docs/getting-started/pricing.md, https://docs.perplexity.ai/docs/admin/rate-limits-usage-tiers.md, https://docs.perplexity.ai/docs/resources/faq.md, https://docs.perplexity.ai/api-reference/legacy/sonar-post.md, https://docs.perplexity.ai/docs/legacy/sonar/quickstart.md
- Gemini: https://ai.google.dev/gemini-api/docs/generate-content/google-search, https://ai.google.dev/gemini-api/docs/google-search, https://ai.google.dev/gemini-api/docs/models, https://ai.google.dev/gemini-api/docs/thinking, https://ai.google.dev/api/generate-content, https://ai.google.dev/gemini-api/docs/pricing, https://ai.google.dev/gemini-api/docs/rate-limits, https://ai.google.dev/gemini-api/terms
- OpenAI (SDK source only): https://github.com/openai/openai-python, specifically `src/openai/types/responses/{web_search_tool,response_function_web_search,response_output_text,response_includable,response_usage,response_create_params,response}.py`, `src/openai/types/shared/{chat_model,responses_model,reasoning_effort}.py` and `CHANGELOG.md` (v3.33.0, 2026-10-16)
- **Not loadable from this environment** (snippets only, so values are tagged [unverified]): https://developers.openai.com/api/docs/guides/tools-web-search, https://developers.openai.com/api/docs/pricing, https://openai.com/api/pricing/, https://developers.openai.com/api/docs/guides/rate-limits
