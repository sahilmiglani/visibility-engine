# Status

_Last updated: 2026-10-17_

## Built

- Repo structure: `CLAUDE.md`, `docs/BRIEF.md`, `docs/STATUS.md`, `docs/DECISIONS.md`, and the `/wrap` command.
- Phase 1 specs in `docs/specs/`: data model, engines (verified against provider docs on
  2026-10-17), pipeline, understanding and prompts, extraction and scoring, hooks, report page,
  admin, testing, and the build plan.
- `.env.example` listing every environment variable the specs reference.

## In progress

- Spec review by Sahil (PR open). **No application code until the specs are approved.**

## Next

1. Resolve the open questions at the top of `docs/specs/build-plan.md`.
2. Foundation (one worker): scaffold, Supabase migrations, Trigger.dev setup, shared types, the
   adapter interface plus a mock adapter, the LLM client plus a mock, and CI.
3. Parallel workstreams W1–W4 as laid out in `build-plan.md`, then W5 integration and the live Phase 1 acceptance run.

## Blockers

- **Gemini grounding terms** (build-plan Q1): the terms appear to prohibit analysing grounded results and showing
  them to anyone other than the requester. Gemini stays disabled by default until this is decided.
- **OpenAI docs unreachable from the build environment** (build-plan Q5): `developers.openai.com` and
  `openai.com` fail DNS or return a proxy 403, so OpenAI pricing and rate limits are unverified. Either allowlist those hosts
  or confirm the figures from the OpenAI console before the live run.
- Accounts and API keys are needed before W5 (live acceptance): OpenAI, Perplexity, Google AI
  Studio (Gemini), Anthropic, a Supabase project, a Trigger.dev project, a Vercel project, DNS for
  `visibility.studio5.in`, and a Cal.com booking link. None are needed for the foundation or W1–W4,
  which run on fixtures and mocks.

## Users and validation

- No users yet. First validation target: a 10-prospect run in one category, reviewed by Sahil for
  hook accuracy before anything is exported to Instantly.
