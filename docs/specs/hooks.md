# Spec: Hooks (selection, templates, evidence, verification)

Code: `src/hooks/`. Task: `hook.generate` (pipeline.md). Output: `hooks` and `hook_evidence` rows.

## 1. Principles
- **Facts first, copy second.** Candidates are computed deterministically from `ProspectMetrics`, and each
  candidate carries a `facts` object plus its evidence call IDs. The LLM only writes copy from `facts`.
  It never sees raw data it could re-interpret.
- **No evidence means no hook.** A candidate with no `engine_call_id` evidence is discarded.
  `hook_evidence` must have at least one `primary` row before a hook can reach `pending_review`.
- **Framing.** Every hook begins with or contains `In our tests on {Month D, YYYY}`. The date is the cycle's
  `test_date`, e.g. "In our tests on October 17, 2026". Hooks never claim to describe what "people see" or what the consumer
  app shows, and they never generalise beyond the tested prompts.
- **Specific and checkable.** Every number in the copy must equal a number in `facts`.

## 2. Candidate rules (priority order a → d)

Each rule yields zero or more candidates with a `strength` between 0 and 1. Selection takes the **highest-priority type
that has a qualifying candidate**, and within that type the highest strength. The other candidates are kept with
`status=candidate` so the admin can swap.

### (a) Absence
- **Qualifies when:**
  - P's mention count across included engines in `R_cat` is ≤ `ABSENCE_MAX_MENTIONS`. The default is **0**, i.e. strict absence. See Q4.
  - At least 2 competitors in S each appear in ≥ 20% of `R_cat`.
  - Per engine: `|R_cat[e]| ≥ 0.9 × expected`.
- **Engine-level variant:** if P is absent on one or more engines but present on others, the candidate covers only the absent engines. Strength is reduced to 0.8×.
- **Facts:**
  - `engines_absent[]`, `n_prompts`, `n_answers`.
  - `example_prompt` (the highest-intent `best_for` prompt).
  - `top_competitor {name, count}` and `second_competitor`.
  - `test_date`.
- **Evidence:** every call ID in `R_cat` for the absent engines, all with `claim_key=absent_in`. The 3 calls naming
  `top_competitor` most prominently are marked `primary` (`claim_key=competitor_count`).
- **Strength:** `0.6 + 0.4 × min(1, top_competitor.count / n_answers × 2)`.

### (b) Competitor capture
- **Qualifies when:** at least one brand-prompt response with intent in {alternatives, versus, fit_for_use_case} is `captured`
  (extraction-and-scoring.md §3), and the captured-by brand is a real competitor rather than a generic tool.
- **Strongest form:** a `fit_for_use_case` or `versus` answer where P isn't recommended and a named competitor is.
- **Facts:** `engine`, `prompt_text`, `recommended_brands[]`, `p_recommended=false`, the `snippet` for the
  recommendation, `capture_rate`, `n_brand_answers`.
- **Evidence:** the specific call (primary, `claim_key=capture_answer`), plus the other captured calls (supporting).
- **Strength:**
  - `0.5 + 0.3 × capture_rate`.
  - Add 0.2 if the same capture happened on both samples or on 2 or more engines.
- **Note:** an "alternatives to P" answer naming competitors is expected behaviour, not a finding. It
  only qualifies if P is also absent from, or ranked below, competitors in the category answers. That's
  checked with `MR < 0.25` or `AP > 3`.

### (c) Source gap
- **Qualifies when:**
  - A third-party domain in the top 5 cited sources is cited in ≥ 15% of `R_cat`.
  - Its fetched page (`source_checks`, fetched within 7 days) **doesn't** mention P under any alias.
  - The same page mentions at least 1 competitor.
- **Confirmation is mandatory.** The page must have been fetched successfully, with HTTP 200 and at least 500 characters of text. Pages
  that are `blocked_by_robots`, fail to fetch, or are Gemini redirects can't support this hook.
- **Facts:** `domain`, `url`, `page_title`, `cited_in` (number of answers), `n_answers`,
  `competitors_on_page[]`, `checked_at`.
- **Evidence:** the calls citing that URL (`claim_key=cited_source`), plus the `source_check_id` on the primary row.
- **Strength:** `0.4 + 0.6 × (cited_in / n_answers)`.

### (d) Inaccuracy
- **Qualifies when:** a claim about P extracted from a brand-prompt answer (`mentions.claims` where `brand=P`) is
  judged to contradict P's own site by the claim check step:
  - Model: `LLM_MODEL_CLASSIFY`, template `claim-check.v1`.
  - Input: the claim and its snippet, plus P's `key_facts` and the site page text.
  - Output: `{verdict: contradicts|supported|not_addressed, site_quote, explanation}`. `site_quote` must be a verbatim substring of the stored site text.
- Only `contradicts` with a valid `site_quote` qualifies.
- **Always** `needs_verification = true`. The admin must tick "I checked the live site" before approving
  (admin.md). Unverified inaccuracy hooks can never be exported.
- **Facts:** `engine`, `prompt_text`, `ai_claim`, `ai_snippet`, `site_quote`, `site_url`, `fetched_at`.
- **Evidence:** the call ID (primary, `claim_key=ai_claim`). The site page is linked through `facts.site_url` and `site_pages.id`.
- **Strength:** `0.5`, rising to `0.7` if the contradiction appears on 2 or more engines.

**No qualifying candidate** → prospect `stage = no_finding`. The admin can still see the report, but
nothing is exported. (Q9 asks whether to add a "position gap" hook for visible prospects.)

## 3. Copy generation (`hook.v1`)

**Model:** `LLM_MODEL_REASON`, effort `medium`.
**Input:** `{type, facts, company_name, first_name_placeholder: "{{first_name}}", test_date_text}`. Two to three
style examples per type come from `src/hooks/examples.ts`.

**System prompt:**
```
You write the opening hook of a cold email from a consultant to a B2B SaaS founder or marketer.
The hook states ONE finding from an AI-search visibility test, using ONLY the facts provided.
Rules:
- 1–2 sentences, ≤ 45 words total, plain US English, no exclamation marks, no hype, no emojis.
- Must include the exact phrase "In our tests on {test_date_text}".
- Every number must come from the facts. Do not round up, generalise, or add new claims.
- Name the engine(s) as "ChatGPT", "Perplexity" or "Gemini" (we tested via their APIs; do not
  claim this is what users see in the apps).
- Name at most two competitors, exactly as given.
- Do not mention the report link or a call to action; those are added separately.
```
**Output:** `{ hook: string, alt_hook: string }`. The alternative is offered in the admin editor.

**Templates**, used as examples and as the deterministic fallback if the LLM fails validation twice:
- (a) `In our tests on {date}, {engines} didn't mention {company} in any of the {n_answers} answers to buyer questions like "{example_prompt}". {top_competitor} came up {count} times.`
- (b) `In our tests on {date}, when we asked {engine} "{prompt_text}", it recommended {brand_1} and {brand_2} instead of {company}.`
- (c) `In our tests on {date}, {domain}'s page "{page_title}" was cited in {cited_in} of {n_answers} AI answers about {category}. It lists {competitor_1} and {competitor_2}, but not {company}.`
- (d) `In our tests on {date}, {engine} told us {company} {ai_claim_paraphrase}. Your {page} page says "{site_quote_short}".`

## 4. Validation (code, after generation and after every admin edit)
`src/hooks/validate.ts` returns `{ok, issues[]}`, stored in `hooks.validation`.
1. Contains `In our tests on {test_date_text}`, case-sensitive.
2. Length is at most 2 sentences and at most 300 characters.
3. Every number token (digits and number words one through twenty) appears in `facts`.
4. Every brand name mentioned is P or in `facts`. Brand names are found with the matcher over the category dictionary.
5. Engine names mentioned ⊆ the engines in `facts`.
6. Banned phrases are absent: `always`, `never shows`, `every time`, `all AI`, `guaranteed`,
   `users see`, `customers see`, and `in ChatGPT's app`.
7. `hook_evidence` has at least 1 primary row.

If validation fails after generation, retry once with the issues appended, then use the template fallback.
An admin edit that fails validation can't be saved, except that rule 3 can be overridden with a typed reason. The override is logged.

## 5. Verification flags and approval
- `needs_verification = true` for all `inaccuracy` hooks, and for any hook whose facts touch a mention with
  `needs_review = true` (an ambiguous alias or an extraction disagreement).
- Approval requires: validation ok, `needs_verification` either false or resolved (`verified_at`, `verified_note`),
  and a published report.
- On approval, the report snapshot is frozen (report-page.md §5), the contacts' report links are
  activated, and the prospect's `stage = approved`.
- On rejection, a `reject_reason` is required. Options: `inaccurate`, `weak`, `not_icp`, `competitor_or_client`, `other`. With
  `competitor_or_client`, a domain suppression can be added in one click.
