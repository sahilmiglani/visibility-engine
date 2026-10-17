# Spec: Report page

Route: `app/r/[token]/page.tsx`, served at `{REPORT_BASE_URL}/r/{token}`. The default base URL is
`https://visibility.studio5.in`. It's a public, server-rendered page. Data comes from the frozen
snapshot only, through `get_report_by_token` with the service role (data-model.md §RLS).

## 1. URL and access
- `token` is 22 base62 characters from 16 bytes of `crypto.randomBytes`, so 128 bits. Each contact gets
  its own token for the same report. The admin also has a contact-less preview token.
- An unknown, revoked or draft token returns a 404 with the generic "This report isn't available" copy. It never
  reveals whether a token once existed.
- The admin preview is `/admin/reports/[id]`, which renders the same component from the draft snapshot with a
  "DRAFT" banner and no view tracking.
- The page is static apart from the beacon. Response headers: `Cache-Control: private, no-store`.

## 2. Indexing and privacy
- `<meta name="robots" content="noindex, nofollow, noarchive">` through `generateMetadata`, **plus** the response header
  `X-Robots-Tag: noindex, nofollow, noarchive` (set in `middleware.ts` for `/r/*`).
- `/r/` is **not** disallowed in `robots.txt`. Crawlers have to be able to fetch the page to see the noindex.
  The page isn't in the sitemap.
- `Referrer-Policy: no-referrer`, so clicking a cited source never leaks the token. All outbound links use
  `rel="noopener noreferrer nofollow"` and `target="_blank"`.
- No third-party scripts, fonts or analytics on report pages. Self-host fonts with `next/font`.
- Open Graph: a generic Studio5 title and image with no prospect data, so link previews in email or Slack don't leak findings.

## 3. Sections (in order)

1. **Header.**
   - Studio5 logo and "AI Search Visibility Report".
   - "Prepared for {Company}".
   - "Tested on {test_date}" and "Engines: ChatGPT (OpenAI API), Perplexity (API)[, Gemini (API)]".
2. **Summary score.**
   - A 0–100 dial with its band label.
   - Three stat tiles: Mention rate, Share of voice and Average position, each with a per-engine mini breakdown.
   - The hook sentence as the headline finding, with an "evidence" link that scrolls to the primary answers.
3. **Prompt × engine grid.**
   - Rows are the category prompts, grouped by intent. Columns are the engines.
   - Cells: ● mentioned in 2/2, ◐ 1/2, ○ not mentioned, — no data. Cells show position on hover or focus.
   - Clicking a cell opens that cell's raw answers (§3.6).
   - A collapsible second block shows the brand prompts.
4. **Competitor leaderboard.**
   - A table of brands in S: mentions, mention rate, average position and per-engine mention rate.
   - The prospect's row is highlighted.
   - Bars use a single hue. The prospect gets the accent colour.
5. **Top cited sources.**
   - The top 10 domains: answers citing, per-engine counts and a label (third-party / competitor-owned / yours).
   - For checked pages: "Mentions you: yes/no" and "Mentions: {competitors}", with the check date.
6. **Three recommended fixes.**
   - Built in code from the findings, chosen and ordered by `src/report/fixes.ts` rules.
   - The LLM may lightly polish wording from fixed templates. It can't add claims.
   - Rule examples:
     - **Source gap:** "Get included on {domain}'s '{page_title}'. It was cited in {n} answers."
     - **Absence on best-for prompts:** "Publish a comparison page for '{use_case}' that answers '{example_prompt}' directly."
     - **Competitor capture:** "Create a '{company} vs {competitor}' page with honest, structured comparisons."
     - **Inaccuracy:** "Correct '{claim}' on your {page} page and in your structured data."
     - **Generic fallback:** "Add an llms.txt and FAQ schema covering {use_cases}."
   - Always exactly 3, each with a one-sentence "why", linked to the evidence behind it.
7. **Raw answers (proof).**
   - An accordion per prompt. Inside it, per engine and sample: the answer text (rendered markdown, sanitised with
     `rehype-sanitize`) with the prospect and competitor names highlighted, the citations list, the model ID, and the call
     timestamp in UTC.
   - Each answer shows its short evidence ID (the first 8 characters of `engine_calls.id`), which the hook cites.
   - If Gemini grounded answers are ever shown, `searchEntryPoint.renderedContent` must be rendered next to
     each one (Gemini terms; see engines.md §3.5). This is blocked by Q1.
8. **Method and framing.**
   - Copy: "In our tests on {date}, we asked each engine {n} buyer-style questions through its official API,
     twice each, with web search on and a US location. API answers can differ from what people see in the
     ChatGPT, Perplexity or Gemini apps, and answers vary from run to run. This is a snapshot, not a guarantee."
   - Also shows: prompt set version, engines and models, number of runs, and the date.
9. **CTA.**
   - "Book a 20-minute walkthrough" opens `CAL_LINK` (settings, falling back to env).
   - It's prefilled with `?name=&email=` **only** when the link has a contact. That data comes from our DB, not from the URL.
   - Booking clicks aren't tracked in Phase 1. A Cal.com webhook could cover them in Phase 2.
10. **Footer.** "Prepared by Studio5 · studio5.in". Contact email, plus a "Remove me" mailto link that leads to a manual suppression.

Copy rules:
- Every number on the page is rendered from the snapshot.
- There are no sentences claiming more than was tested.
- "In our tests" framing appears in the header subtitle, next to the hook and in the method section.

## 4. View tracking
- The client component `ViewBeacon` waits for `document.visibilityState === "visible"` for 3
  continuous seconds, then sends `navigator.sendBeacon('/api/r/{token}/view')` once per page load.
  A `sessionStorage` flag `rv:{token}` dedupes reloads; it's wrapped in try/catch.
- `app/api/r/[token]/view/route.ts` (POST):
  - Ignores requests whose `User-Agent` matches a bot or scanner list (`isbot` plus known mail scanners).
  - Calls the `record_report_view(token)` RPC.
  - Always returns 204, so the response doesn't reveal whether the token is valid.
  - **It stores no IP, user agent, referrer or cookie.** The only data kept is `view_count`, `first_viewed_at` and `last_viewed_at` per link.
- Rate limit: at most 1 counted view per token per 60s, enforced in the RPC by comparing `last_viewed_at`.
- Phase 2 hook: the RPC also inserts a row into `report_view_events(report_link_id, viewed_at)`, which feeds the
  Telegram alert. The table is created in Phase 2 and still holds no personal data.

## 5. Snapshot
- `reports.snapshot` (`ReportSnapshot`, versioned with zod) holds everything the page renders:
  - company name and domain, test date, engines and models, the score object, grid cells with call IDs,
    leaderboard, sources, fixes, hook text, and the raw answers.
  - Raw answers are stored as the answer text plus citations for each evidence call. **The page never queries live tables.**
- The snapshot is built as a `draft` by `report.build` and re-built if the hook changes before approval.
- It's **frozen at approval** (`status=published`). Later re-scoring creates a new draft report and leaves the published one alone.
- Size budget: about 1.5 MB. If larger, raw answers are truncated to the first 4,000 characters, with a "full answer on
  request" note. In Phase 1 sizes are about 40 prompts × 3 × 2 × 3 KB ≈ 700 KB.

## 6. Design
- One clean, printable page. Light theme by default, plus `prefers-color-scheme` dark tokens.
- Works at 375px width. Tables scroll horizontally inside their own container, never the page.
- Accessibility: grid cells carry text labels (`aria-label="Mentioned in 2 of 2 runs, position 3"`), and colour is never the only signal.
- Print stylesheet: expands accordions and hides the CTA and beacon.
