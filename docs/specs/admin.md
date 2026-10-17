# Spec: Admin

Routes are under `app/(admin)/admin/**`, protected by `middleware.ts`:
- There must be a valid Supabase session, and `is_admin()` must be true. Otherwise redirect to `/admin/login`.
- The login page uses Supabase email magic link. Sign-ups are disabled.
- Server actions re-check `is_admin()`. Mutations go through server actions using the user's session client (RLS-enforced).
  Triggering Trigger.dev tasks uses `tasks.trigger` from the server.

UI: Next.js server components, plus small client components for tables, forms and realtime progress. Use
shadcn/ui-style primitives and TanStack Table for lists. No global state library.

## Screens and actions

### 1. Dashboard (`/admin`)
- Counts by prospect stage: imported, understood, categorized, probing, hook_ready, no_finding,
  approved, rejected, excluded and error.
- Costs: today, last 7 days and all time, split by engine and LLM. Running cycles show progress bars (realtime `metadata.progress`).
- Alerts:
  - Paused engines (`quota_exhausted`), with a resume button.
  - Failed cycles.
  - Prospects in `error`.
  - The "check category" list.
  - Unreviewed brands.

### 2. Imports (`/admin/imports`)
- **Upload a CSV** (≤ 5 MB, ≤ 5,000 rows):
  1. Parse it with `papaparse` in a server action.
  2. Detect the source (Apollo or Clay) from the headers and auto-map columns:
     - `company` ← `Company`, `Company Name`, `Organization Name`, `company_name`
     - `domain` ← `Website`, `Company Domain`, `Domain`, `company_website` (normalized)
     - `first_name`, `last_name`, `title` ← `Title` / `Job Title`
     - `email` ← `Email`, `Work Email`
     - `country` ← `Country`, `Person Country`, `Company Country`
  3. A mapping editor lets the admin fix the map.
- **Preview table with per-row flags:**
  - `invalid` (missing email or domain, bad email syntax).
  - `duplicate_in_file`.
  - `duplicate_existing` (same prospect domain plus email).
  - `non_us`: the country column isn't US, or the email or domain TLD is a country-code TLD (other than `.us`, `.io`, `.ai`, `.co` or `.app`).
  - `unknown_country`.
  - `suppressed` (the email or its domain is on the suppression list).
  - `email_domain_mismatch` (the email domain differs from the company domain; a warning only).
- **Commit:** inserts prospects and contacts and records counts on `imports`. New prospects with a US or unknown flag go into the pipeline.
  Non-US rows are stored but `excluded` (prospect) or skipped at export (contact), with a per-row "include anyway" override.
- **Import history:** counts, errors, and a link to each import's prospects.

### 3. Prospects (`/admin/prospects`, `/admin/prospects/[id]`)
- **List:**
  - Filters: stage, category, country flag, has hook, and import.
  - Columns: company, domain, category, stage, score, hook type, contacts, cost, and updated.
- **Detail tabs:**
  - **Overview:** the understanding (one-liner, sells-to, use cases, key facts with quotes, and competitors), the site pages fetched (with links to the stored text), and the cost breakdown (own plus allocated).
  - **Category:** the current assignment with confidence and rationale. Actions:
    - **Reassign** (picker).
    - **Create new category from this prospect.**
    - **Re-categorize.** Reassigning moves the prospect to `categorized` and triggers attach once the new category has an active prompt set.
  - **Brand prompts:** a list with edit (which creates a replacement), disable, and add custom.
  - **Results:** the score summary, the grid (the same component as the report), the leaderboard, and the raw answers with mentions highlighted and `needs_review` marks.
  - **Hooks:** the selected hook plus other candidates, with facts and evidence links.
  - **Report:** preview and links per contact, with view counts.
  - **Contacts:** a list with outreach status and suppress/unsuppress.
- **Actions:**
  - Re-run understanding.
  - Re-probe, which attaches the prospect to a new cycle.
  - Exclude or include.
  - Edit name and aliases (this edits the prospect's brand).

### 4. Categories and prompts (`/admin/categories`, `/admin/categories/[id]`)
- **List:** name, prospects, active prompt set version, last cycle date and cost.
- **Detail:**
  - Edit the name, description and buyer.
  - **Merge into…**, with a confirmation that shows the effects.
  - Archive.
- **Prompt sets** for the category:
  - A version list (status, origin and created).
  - Viewing a version shows its prompts by intent.
  - **Edit:** inline editing of the text and intent, reorder, add and delete. Saving creates a new draft version from the current one.
  - **Diff vs active.**
  - **Activate:** the version must have 15–20 prompts that pass the code checks. Activating archives the previous version and triggers attach for waiting prospects.
  - **Regenerate with LLM:** creates a new draft.
- **Brands:** the dictionary for the category.
  - Name, domain, aliases (editable chips), the ambiguous flag, and review status.
  - Merge suggestions (fuzzy duplicates).
  - Confirm or merge actions for unreviewed brands found by extraction.
  - Edits trigger re-scoring of affected prospects (extraction-and-scoring.md §4).

### 5. Runs (`/admin/runs`, `/admin/runs/[cycleId]`)
- **Cycle list:**
  - Category, status, prompt set version, engines, prospects attached, and test date.
  - Calls succeeded, failed and empty per engine.
  - Estimated and actual cost, and duration.
- **Cycle detail:**
  - Realtime progress.
  - A per-engine success-rate bar with the 90% line.
  - A failed-calls table with error codes.
  - Actions:
    - **Retry failed cells.**
    - **Cancel.**
    - **Confirm over-budget start.**
  - Each cell links to its `engine_calls` row (raw request and response JSON viewer, latency and cost).
- **Task runs:** the `task_runs` list with links to the Trigger.dev dashboard.

### 6. Approval queue (`/admin/queue`)
- **The core review screen.** One card per prospect with `stage=hook_ready`, sorted by hook strength.
- **Card contents:**
  - Company, category, score and band, and hook type.
  - **The hook text** (editable). Live validation results appear as you type.
  - **Evidence panel:** the primary evidence answers inline, with the relevant snippet highlighted, plus a link to all evidence.
  - For source-gap hooks: the checked page URL, a snippet of the page text, and "mentions you: no".
  - For inaccuracy hooks: the AI claim next to the site quote, plus a **"I checked the live site — the contradiction is real"** checkbox and a note. **Required.**
  - Alternative candidates: swap to another hook type or candidate.
  - Report preview link, and the contacts the hook will be exported for (with suppressed or non-US contacts struck through).
- **Actions:**
  - **Approve:** validation must pass and verification must be done. This freezes the report.
  - **Edit and approve.**
  - **Reject** with a reason (plus an optional domain suppression).
  - **Regenerate copy.**
- **Keyboard:** `j`/`k` to move, `a` approve, `r` reject, `e` edit.

### 7. Exports (`/admin/exports`)
- **New Instantly CSV:**
  - Choose approved prospects: all not yet exported, or a selection. Set a campaign name (a label only).
  - **Pre-flight**, which runs `is_suppressed()` on every contact plus checks that the contact is US, the hook is approved and the report is published. It shows included and skipped counts with reasons.
  - **Generate:** builds the CSV in `src/export/instantly.ts`, uploads it to the `exports/` bucket, stores `export_rows`, sets the contacts to `outreach_status=exported`, and offers a download through a signed URL.
- **CSV columns** (UTF-8, header row):

  | Column | Kind | Value |
  |---|---|---|
  | `email` | standard | |
  | `first_name` | standard | |
  | `last_name` | standard | |
  | `company_name` | standard | |
  | `website` | standard | |
  | `job_title` | standard | |
  | `hook` | custom variable | the approved hook text, single line |
  | `report_url` | custom variable | the contact's report link |
  | `test_date` | custom variable | e.g. "October 17, 2026" |
  | `category` | custom variable | |
  | `top_competitor` | custom variable | |
  | `score` | custom variable | |
  | `engine_tested` | custom variable | e.g. "ChatGPT and Perplexity" |

  - Instantly CSV rules: email is the only required column. Other columns become custom variables named after their header, used as `{{header}}` (case-sensitive). Headers must be at most 20 characters with no duplicates, at most 50 variables, UTF-8, and must not reuse a predefined field name. Re-uploading an existing lead skips it. *[search-only; the help pages were blocked, so confirm on first upload]* The API schema's standard fields are `email, first_name, last_name, company_name, website, phone, job_title, personalization`.
  - Values are escaped per RFC 4180. Newlines in the hook are replaced with spaces. A leading `=`, `+`, `-` or `@` gets a `'` prefix to block CSV injection.
- **Export history:** download again, and see rows and skipped rows.
- Phase 2 replaces download with **Push to Instantly** (`POST /api/v2/leads/add`, ≤ 1,000 leads per call,
  `campaign_id`, `custom_variables` flat values, `skip_if_in_campaign: true`), after the same pre-flight.

### 8. Suppression list (`/admin/suppressions`)
- Table of email and domain entries with reason, source, note and date.
- Add a single entry, or paste or upload a CSV of emails and domains.
- Remove an entry, with a confirmation and an audit note.
- Reasons: unsubscribed, bounced, replied_no, manual, competitor, existing_client. Phase 2 webhooks add entries automatically.

### 9. Costs (`/admin/costs`)
- **Per engine:** calls, average cost per call, total, average latency, and the success rate over the date range.
- **Per cycle (category run):** category-prompt engine cost, extraction cost, source-check cost, and total.
- **Per prospect:** own cost (brand probes, understanding, hooks, extraction) plus allocated category share, and total. This is the Phase 1 acceptance view.
- **LLM by purpose:** understand, categorize, prompts, extract, claim check and hook. Tokens and cost.
- CSV download of each table. Prices shown with `pricing_version`.

### 10. Settings (`/admin/settings`)
- Cal.com link, cycle reuse days, max cycle cost, absence threshold, enabled engines (read-only display of
  env, with paused-engine toggles), and report footer note.
- `REPORT_BASE_URL` is shown read-only, since it's env-only.
