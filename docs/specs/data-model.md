# Spec: Data model (Phase 1)

Postgres on Supabase. Migrations live in `supabase/migrations/`. Generated TypeScript types are in
`src/core/db.types.ts` (`supabase gen types typescript --local`).

## Conventions

- Every table has `id uuid primary key default gen_random_uuid()` and `created_at timestamptz not null default now()`.
  Mutable tables also have `updated_at`, maintained by a shared `set_updated_at()` trigger.
- Enums are Postgres `text` columns with `check` constraints rather than `create type`, so adding a value is a cheap migration.
  This doc lists the allowed values.
- Money is `numeric(12,6)` in USD. Tokens and counts are `integer`. Timestamps are `timestamptz`.
- JSON columns are `jsonb`, and each has a zod schema in `src/core/json-schemas.ts`. Code always parses on read.
- Large raw payloads (HTML, raw engine responses over 256 KB) go to Supabase Storage. The row keeps
  a `*_path` column. Smaller raw responses are stored inline in `jsonb` so evidence is queryable.
- Soft state changes use status columns. Nothing in the evidence chain (`engine_calls`, `extractions`,
  `mentions`, `citations`, `source_checks`, `hooks`, `hook_evidence`, `reports`) is ever hard-deleted.

## Entity overview

```
imports ──< contacts >── prospects >── categories ──< prompt_set_versions ──< prompts (kind=category)
                            │  │                         │
                            │  └──< prompts (kind=brand)  └──< probe_cycles ──< cycle_prospects >── prospects
                            │                                     │
                            │                                     └──< engine_calls ──< extractions ──< mentions >── brands
                            │                                              └──< citations ──> source_pages ──< source_checks
                            ├──< site_pages
                            ├──< prospect_scores
                            ├──< hooks ──< hook_evidence >── engine_calls
                            └──< reports ──< report_links >── contacts
llm_calls (any step)   suppressions   exports ──< export_rows   settings   admin_users   task_runs
```

## Tables

### `admin_users`
Holds the single admin. Checked by `is_admin()`.
| column | type | notes |
|---|---|---|
| user_id | uuid PK, FK `auth.users(id)` | |
| email | text not null unique | |

### `imports`
One row per uploaded CSV.
| column | type | notes |
|---|---|---|
| filename | text | |
| source | text | `apollo` \| `clay` \| `other` (detected from headers, editable) |
| storage_path | text | original file in bucket `imports/` |
| row_count, inserted_count, duplicate_count, invalid_count, non_us_count, suppressed_count | int | import summary |
| column_map | jsonb | CSV header → canonical field |
| status | text | `parsed` \| `committed` \| `failed` |
| errors | jsonb | per-row errors `[{row, field, message}]` |

### `prospects`
A company, keyed by its normalized domain. This is the unit of analysis.
| column | type | notes |
|---|---|---|
| domain | text not null **unique** | normalized: lowercase, no scheme, no `www.`, no path |
| name | text not null | from CSV, can be refined by understanding |
| brand_id | uuid FK `brands` | the prospect's own brand row (created at understanding) |
| category_id | uuid FK `categories` null | |
| category_assigned_by | text | `llm` \| `admin` |
| category_confidence | numeric(3,2) | from categorization |
| country_flag | text | `us` \| `non_us` \| `unknown` (see import spec in admin.md) |
| description | text | one-line description |
| sells_to | text | target customer |
| use_cases | text[] | 3–6 |
| competitors_suggested | jsonb | `[{name, domain, reason}]` from understanding (3–6) |
| understanding | jsonb | full validated LLM output, plus `schema_version` |
| stage | text not null default `imported` | `imported` → `fetching` → `understood` → `categorized` → `probing` → `scored` → `hook_ready` \| `no_finding` → `approved` \| `rejected`; plus `error` and `excluded` |
| stage_error | text | last error message, if `stage = error` |
| excluded_reason | text | e.g. `non_us`, `suppressed`, `manual` |
| import_id | uuid FK `imports` | first import that created it |

Indexes: `unique(domain)`, `(category_id, stage)`, `(stage)`.

### `contacts`
People at a prospect. This is the unit of export.
| column | type | notes |
|---|---|---|
| prospect_id | uuid FK not null | |
| import_id | uuid FK | |
| first_name, last_name, title | text | |
| email | text not null | stored lowercased and trimmed |
| email_domain | text generated | `split_part(email,'@',2)` |
| country | text null | if the CSV provides one |
| country_flag | text | `us` \| `non_us` \| `unknown` |
| raw | jsonb | the original CSV row |
| outreach_status | text not null default `new` | `new` → `exported` → `sent` → `report_viewed` → `replied` → `call_booked` → `won`; plus `lost`, `bounced`, `unsubscribed` (Phase 2 updates most of these) |
| last_exported_at | timestamptz | |

Indexes: **`unique(prospect_id, email)`**, which is the dedupe rule (domain plus email, since a
prospect is a domain); `(email)`; `(outreach_status)`.

### `categories`
| column | type | notes |
|---|---|---|
| name | text not null | e.g. "Sales engagement platforms" |
| slug | text unique not null | |
| description | text | what's in or out of the category |
| buyer | text | typical buyer persona |
| status | text | `active` \| `merged` \| `archived` |
| merged_into_id | uuid FK `categories` null | set when merged; reads follow it |
| active_prompt_set_id | uuid FK `prompt_set_versions` null | |

### `brands`
The brand dictionary used for matching. Brands are category-scoped so names don't collide across categories.
| column | type | notes |
|---|---|---|
| category_id | uuid FK null | null means global |
| name | text not null | display name |
| normalized_name | text not null | see extraction-and-scoring.md for normalization |
| domain | text null | canonical domain, used for citation-to-brand matching |
| aliases | text[] not null default `{}` | extra surface forms, normalized |
| ambiguous | boolean default false | common-word names (e.g. "Close", "Front"); stricter matching |
| source | text | `prospect` \| `understanding` \| `extraction` \| `admin` |
| review_status | text | `confirmed` \| `unreviewed` \| `merged` |
| merged_into_id | uuid FK `brands` null | |

Indexes: `unique(category_id, normalized_name)`, `(domain)`, GIN on `aliases`.

### `prompt_set_versions`
Versioned category prompt sets. A version is immutable once `status = active` or once any probe uses it.
| column | type | notes |
|---|---|---|
| category_id | uuid FK not null | |
| version | int not null | 1, 2, 3, … per category |
| status | text | `draft` \| `active` \| `archived` |
| origin | text | `llm` \| `admin_edit` |
| parent_version_id | uuid FK null | the version this was edited from |
| generator | jsonb | `{model, llm_call_id, prompt_template_version}` |
| notes | text | |

Indexes: `unique(category_id, version)`. Partial unique index `(category_id) where status='active'`
enforces one active version.

### `prompts`
Category prompts and brand prompts in one table.
| column | type | notes |
|---|---|---|
| kind | text not null | `category` \| `brand` |
| prompt_set_version_id | uuid FK null | set when `kind = category` |
| prospect_id | uuid FK null | set when `kind = brand` |
| intent | text not null | category: `best_for`, `top_tools`, `comparison`, `problem`, `alternatives_generic`, `how_to_choose`; brand: `what_is`, `alternatives`, `versus`, `fit_for_use_case`, `pricing` |
| text | text not null | the exact text sent to engines |
| template | text null | brand prompts: the template used, e.g. `Alternatives to {company}` |
| variables | jsonb | e.g. `{competitor: "Outreach"}` |
| position | int | display order |
| status | text | `active` \| `superseded` \| `disabled` |
| superseded_by_id | uuid FK `prompts` null | brand prompts are edited by creating a replacement row |

Check constraint: exactly one of `prompt_set_version_id` or `prospect_id` is set, matching `kind`.
Indexes: `(prompt_set_version_id)`, `(prospect_id, status)`.

### `probe_cycles`
One probing pass for one category. Category answers in a cycle are reused for every prospect attached to it.
| column | type | notes |
|---|---|---|
| category_id | uuid FK not null | |
| prompt_set_version_id | uuid FK not null | |
| engines | text[] not null | e.g. `{openai,perplexity,gemini}` |
| engine_config | jsonb | resolved model IDs and params per engine (snapshotted) |
| samples_per_prompt | int not null default 2 | |
| status | text | `pending` → `running` → `extracting` → `scoring` → `complete`; plus `partial`, `failed`, `cancelled` |
| budget_usd | numeric | cap for this cycle (from `MAX_CYCLE_COST_USD`) |
| estimated_cost_usd, actual_cost_usd | numeric | `actual` is maintained by the finalize task |
| test_date | date not null | the date used in "In our tests on {date}" (UTC date the cycle started) |
| started_at, completed_at | timestamptz | |
| trigger_run_id | text | parent Trigger.dev run |

Indexes: `(category_id, created_at desc)`.

### `cycle_prospects`
The prospects attached to a cycle. Their brand prompts run in that cycle, and category cost is shared among them.
| column | type | notes |
|---|---|---|
| cycle_id | uuid FK | |
| prospect_id | uuid FK | |
| joined_at | timestamptz | |
| status | text | `pending` \| `probed` \| `scored` \| `failed` |

PK `(cycle_id, prospect_id)`.

### `engine_calls`
**This is the evidence table.** One row per (prompt × engine × sample) attempt that reached a final
state. Its `id` is the "response ID" that hooks cite.
| column | type | notes |
|---|---|---|
| cycle_id | uuid FK not null | |
| prompt_id | uuid FK not null | |
| prospect_id | uuid FK null | set for brand prompts; null for category prompts (shared) |
| engine | text not null | `openai` \| `perplexity` \| `gemini` \| `mock` (later `claude`, `aio`, …) |
| sample_index | smallint not null | 1..samples_per_prompt |
| idempotency_key | text not null **unique** | `{cycle_id}:{prompt_id}:{engine}:{sample_index}` |
| status | text | `pending` \| `succeeded` \| `failed` \| `empty` (succeeded but no answer text) |
| model_requested | text | e.g. `gpt-5-mini` |
| model_returned | text | the model string the provider echoes |
| params | jsonb | normalized request params (tools, search context size, location, temperature, …) |
| request_body | jsonb | exact JSON sent, with secrets stripped |
| raw_response | jsonb null | full provider JSON when ≤ 256 KB |
| raw_response_path | text null | storage path when larger |
| provider_response_id | text | the provider's `id` / `responseId` |
| answer_text | text | normalized plain text of the answer |
| citations_raw | jsonb | provider-native citation structures |
| search_queries | text[] | queries the engine says it ran, where exposed |
| http_status | int | |
| error_code, error_message | text | for failed rows |
| attempts | smallint | Trigger.dev attempts used |
| started_at, completed_at | timestamptz | |
| latency_ms | int | provider round-trip of the successful attempt |
| input_tokens, output_tokens, reasoning_tokens, search_calls | int | from usage |
| cost_usd | numeric(12,6) not null default 0 | |
| cost_breakdown | jsonb | `{input, output, search, request_fee, source: "computed"\|"provider"}` |
| pricing_version | text | e.g. `2026-10-17` |
| adapter_version | text | bumps when mapping logic changes |

Indexes: `unique(idempotency_key)`, `(cycle_id, engine, status)`, `(prompt_id)`, `(prospect_id)`.

### `citations`
Normalized citations, one row per cited URL per engine call.
| column | type | notes |
|---|---|---|
| engine_call_id | uuid FK not null | |
| position | int | 1-based order in the provider's list |
| url | text not null | resolved final URL (Gemini redirect resolved, see engines.md) |
| url_raw | text | URL exactly as returned |
| normalized_url | text | lowercase host, no fragment, tracking params stripped |
| domain | text not null | registrable domain (eTLD+1), e.g. `g2.com` |
| title | text | |
| inline | boolean | true if referenced in the answer text (annotation or [n] marker) |
| brand_id | uuid FK null | set when `domain` is a known brand's domain |

Indexes: `(engine_call_id)`, `(domain)`, `(normalized_url)`.

### `extractions`
The structured LLM extraction of one engine call. Re-extraction with a new extractor version adds a row.
| column | type | notes |
|---|---|---|
| engine_call_id | uuid FK not null | |
| extractor_version | text not null | e.g. `extract-v1` |
| llm_call_id | uuid FK `llm_calls` | |
| output | jsonb | validated `ExtractionOutput` (see extraction-and-scoring.md) |
| answer_type | text | `ranked_list` \| `comparison` \| `explanation` \| `refusal` \| `other` |
| disagreements | jsonb | LLM vs deterministic matcher differences |
| is_current | boolean default true | |

Indexes: `unique(engine_call_id, extractor_version)`, partial `(engine_call_id) where is_current`.

### `mentions`
One row per brand mentioned in an engine call's answer (from the current extraction).
| column | type | notes |
|---|---|---|
| extraction_id | uuid FK not null | |
| engine_call_id | uuid FK not null | denormalized for queries |
| brand_id | uuid FK null | null if unmatched (`surface_name` holds the text) |
| surface_name | text not null | as written in the answer |
| position | int not null | 1-based order of first appearance among brands in the answer |
| char_offset | int | first occurrence offset in `answer_text` |
| list_rank | int null | rank in an explicit numbered or bulleted list |
| recommended | boolean | the answer recommends it (vs. merely mentions it) |
| sentiment | text | `positive` \| `neutral` \| `negative` \| `mixed` |
| snippet | text | verbatim substring of `answer_text`, ≤ 300 chars |
| claims | jsonb | `[{text, snippet}]` facts asserted about the brand |
| match_method | text | `exact` \| `alias` \| `domain` \| `fuzzy` \| `llm_only` |
| needs_review | boolean default false | |

Indexes: `(engine_call_id)`, `(brand_id, engine_call_id)`.

### `site_pages`
Prospect site snapshots (homepage, pricing, about).
| column | type | notes |
|---|---|---|
| prospect_id | uuid FK | |
| kind | text | `home` \| `pricing` \| `about` |
| url | text | final URL after redirects |
| fetched_at | timestamptz | |
| http_status | int | |
| fetcher | text | `native` \| `jina` \| `firecrawl` |
| html_path | text | storage `page-snapshots/…` |
| text | text | readability-extracted text (capped at 50k chars) |
| text_sha256 | text | |

Indexes: `(prospect_id, kind, fetched_at desc)`.

### `source_pages` and `source_checks`
Cited pages fetched to confirm source gaps.
`source_pages`: `normalized_url unique`, `url`, `domain`, `fetched_at`, `http_status`, `fetcher`,
`html_path`, `text`, `text_sha256`, `fetch_error`.
`source_checks`: `source_page_id FK`, `prospect_id FK`, `cycle_id FK`,
`mentions_prospect boolean`, `matched_alias text null`, `prospect_snippet text null`,
`competitors_mentioned jsonb` (`[{brand_id, snippet}]`), `checked_at`. Unique
`(source_page_id, prospect_id, cycle_id)`.

### `prospect_scores`
| column | type | notes |
|---|---|---|
| prospect_id, cycle_id | uuid FK | unique pair per `scoring_version` |
| scoring_version | text | e.g. `score-v1` |
| summary_score | int | 0–100 |
| metrics | jsonb | the full `ProspectMetrics` object (extraction-and-scoring.md) |
| mention_rate, share_of_voice, avg_position | numeric | denormalized for list views |
| computed_at | timestamptz | |

### `hooks`
| column | type | notes |
|---|---|---|
| prospect_id, cycle_id | uuid FK | |
| type | text | `absence` \| `competitor_capture` \| `source_gap` \| `inaccuracy` |
| is_selected | boolean | the chosen candidate; other candidates are kept for swapping |
| rank | int | position among candidates after priority and strength ordering |
| strength | numeric | 0–1, see hooks.md |
| facts | jsonb | the structured facts the copy was written from |
| text_generated | text | LLM-written hook |
| text_final | text null | admin-edited text (validated again) |
| needs_verification | boolean | always true for `inaccuracy` |
| verified_at, verified_note | | the admin's verification record |
| status | text | `candidate` \| `pending_review` \| `approved` \| `rejected` \| `superseded` |
| reject_reason | text | |
| llm_call_id | uuid FK | |
| validation | jsonb | results of the post-generation checks |

Partial unique index: `(prospect_id, cycle_id) where is_selected`.

### `hook_evidence`
Links each hook claim to stored evidence. **A hook with zero rows here can't leave `candidate`.**
| column | type | notes |
|---|---|---|
| hook_id | uuid FK | |
| engine_call_id | uuid FK not null | |
| role | text | `primary` (shown first) \| `supporting` |
| claim_key | text | which fact this supports, e.g. `absent_in`, `competitor_count`, `source_page` |
| mention_id, citation_id, source_check_id | uuid FK null | finer-grained pointers |

PK `(hook_id, engine_call_id, claim_key)`.

### `reports` and `report_links`
`reports`: `prospect_id`, `cycle_id`, `hook_id`, `status` (`draft` \| `published` \| `revoked`),
`snapshot jsonb` (frozen render data, see report-page.md), `snapshot_version`, `test_date date`,
`published_at`, `revoked_at`.
`report_links`: `report_id FK`, `contact_id FK null` (null for the admin's generic link), `token
text unique not null` (22-char base62 from 16 random bytes = 128 bits), `view_count int default 0`,
`first_viewed_at`, `last_viewed_at`, `revoked_at`. Index `unique(token)`.

### `suppressions`
| column | type | notes |
|---|---|---|
| kind | text | `email` \| `domain` |
| value | text not null | lowercased email or registrable domain |
| reason | text | `unsubscribed` \| `bounced` \| `replied_no` \| `manual` \| `competitor` \| `existing_client` |
| source | text | `admin` \| `instantly_webhook` \| `csv` |
| note | text | |

Index `unique(kind, value)`. The check function `is_suppressed(email text)` returns true if the
email or its domain is present.

### `exports` and `export_rows`
`exports`: `kind` (`instantly_csv` \| `instantly_api`), `row_count`, `skipped_count`, `skipped
jsonb` (`[{contact_id, reason}]`), `storage_path`, `campaign_name`, `filters jsonb`.
`export_rows`: `export_id`, `contact_id`, `hook_id`, `report_link_id`, `row jsonb` (the exact CSV
row). Unique `(export_id, contact_id)`.

### `llm_calls`
Every Anthropic call, whatever the step.
| column | type | notes |
|---|---|---|
| purpose | text | `understand` \| `categorize` \| `generate_category_prompts` \| `generate_brand_prompts` \| `extract` \| `source_check` \| `claim_check` \| `hook` |
| model | text | |
| prompt_template_version | text | e.g. `understand-v1` |
| prospect_id, category_id, cycle_id, engine_call_id | uuid null | attribution (whichever applies) |
| request | jsonb | messages plus params (system prompt by template version, inputs inline) |
| response | jsonb | raw API response |
| parsed | jsonb | validated output |
| status | text | `succeeded` \| `failed` \| `invalid_output` |
| input_tokens, output_tokens, cache_read_tokens, cache_write_tokens | int | |
| latency_ms | int | |
| cost_usd | numeric(12,6) | |
| pricing_version | text | |
| batch_id | text null | if sent through the Message Batches API |

Indexes: `(purpose, created_at)`, `(prospect_id)`, `(cycle_id)`.

### `task_runs`
A thin mirror of Trigger.dev runs, for the admin Runs screen and cross-linking.
`task_id`, `trigger_run_id unique`, `entity_type` (`prospect` \| `cycle` \| `category` \| `import`),
`entity_id`, `status`, `error`, `started_at`, `finished_at`.

### `settings`
Key/value config the admin can edit without a deploy. `key text PK`, `value jsonb`. Phase 1 keys:
`cal_link`, `cycle_reuse_days`, `max_cycle_cost_usd`, `report_footer_note`, `absence_max_mentions`, `engine_paused:{engine}`.
Env vars give the defaults; this table overrides them.

## Cost views

- `v_cost_by_engine_call`: `engine_calls` plus `llm_calls` with purpose `extract`, joined per engine call.
- `v_cycle_costs`: per cycle, the sum of category-prompt engine calls plus their extraction LLM calls, plus the
  cycle's `source_check` calls.
- `v_prospect_costs`: per (prospect, cycle) `own_cost` (brand-prompt engine calls plus their
  extractions, plus understanding, brand prompt generation, claim check and hook LLM calls attributed to the
  prospect), `allocated_category_cost` (`cycle category cost / count(cycle_prospects)`), and `total`.
- Category prompt generation cost is reported per category, not allocated to prospects.

## Row-level security

- RLS is **enabled on every table**, with no exceptions. `alter table … force row level security` is not needed.
- `is_admin()`: `select exists(select 1 from admin_users where user_id = auth.uid())`, defined as
  `security definer stable` with `search_path = ''`.
- One policy per table: `for all to authenticated using (is_admin()) with check (is_admin())`.
- **The `anon` role gets no policies**, so it can read nothing. Public report pages and the view beacon go
  through Next.js server code with the **service role key** (bypasses RLS), scoped by token lookup:
  - RPC `get_report_by_token(token text)` is `security definer`, returns the snapshot only when the
    link and report are not revoked, and executes for `service_role` only.
  - RPC `record_report_view(token text)` atomically increments `view_count`, sets
    `first_viewed_at = coalesce(first_viewed_at, now())` and `last_viewed_at = now()`, and returns
    nothing. It executes for `service_role` only.
- Trigger.dev tasks use the service role key (server-only env var, never `NEXT_PUBLIC_`).
- Signups are disabled in Supabase Auth settings. The admin is created once with `scripts/create-admin.ts`.
- Storage buckets `imports`, `page-snapshots`, `raw-responses` and `exports` are **private**. Their policies
  allow authenticated admins; downloads use short-lived signed URLs.
- Supabase advisors (`get_advisors`, security and performance) must report clean before the foundation PR merges.
