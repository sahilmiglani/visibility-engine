# Brief: Visibility Engine (Studio5)

## The idea

Studio5 ([studio5.in](https://studio5.in)) is an AI services practice offering fractional CTO and
forward-deployed engineering work. It's adding an **AI search visibility** offering (AEO/GEO,
answer-engine and generative-engine optimisation) on its own landing page under Studio5.

To win clients, the outbound pitch is **a demonstration, not a claim**. It shows a prospect, with
real data, how ChatGPT, Perplexity and Gemini treat them compared with their competitors.

Visibility Engine does two jobs with one core:

1. **Outreach mode (Phases 1–2).** For each prospect, find one specific, true, data-backed finding
   and turn it into a cold-email hook plus a shareable report page.
2. **Client mode (Phase 3).** The same engine, run deeper, becomes the paid audit and the monthly
   tracking report for retainer clients.

## The user

- **The operator:** Sahil, the single admin user. He imports prospect lists, reviews findings and
  hooks, approves leads and exports them to Instantly.
- **The audience:** founders, heads of marketing and growth leads at **US B2B SaaS companies from
  seed to Series B**. They receive a cold email with one concrete finding and a link to their
  report. The first campaigns target 2–3 categories.

## Positioning

- Proof first: "In our tests on {date}, ChatGPT didn't mention you in any of 20 'best {category}'
  answers. It named {Competitor} 14 times." The report link holds the raw answers as evidence.
- Honest framing: results are "in our tests", run through official APIs on a stated date. API
  answers can differ from what people see in the consumer apps, and we say so.
- The report ends with three concrete fixes and a call-booking button. The audit and retainer are the upsell.

## Decisions already made

- Sending happens in **Instantly**. This app never sends email itself.
- Report pages live on a Studio5 subdomain. The default is `visibility.studio5.in` and the base URL is configurable.
- Cold email goes out from **separate sending domains**, never from studio5.in. That's set up in
  Instantly, outside this codebase.
- Phase 1 targets the US only.

## Phases

- **Phase 1 (MVP):** import, company understanding, categories and prompt generation; probing on
  three engines; extraction and scoring; hooks and report pages; admin (prospects, categories and
  prompts, runs, approval queue, costs); Instantly CSV export.
- **Phase 2:** Instantly API push, webhooks (report views, replies, bounces, unsubscribes),
  Telegram alerts and a funnel dashboard.
- **Phase 3:** the paid deep audit (100+ prompts, more runs, an exportable recommendations doc) and
  monthly tracking for retainer clients.
- **Out of scope:** list scraping, our own sending infrastructure, X/social automation, multiple users or teams, and billing.

**Phase 1 is done when** a 10-prospect CSV in one category runs end to end with real API keys,
produces report pages and an Instantly CSV, and the admin shows cost per prospect.
