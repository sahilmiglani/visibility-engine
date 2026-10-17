---
description: Wrap up the session by updating docs/STATUS.md and docs/DECISIONS.md from this session's work
---

Wrap up this session. Do the following in order and don't skip steps.

1. **Work out what this session did.** Review the conversation, plus `git status`, `git diff` and
   `git log` for commits on this branch that aren't on the base branch
   (`git log --oneline origin/main..HEAD`). List the features or specs built or changed, files
   touched, tests added, and anything left half-done.

2. **Update `docs/STATUS.md`.**
   - Set `_Last updated:` to today's date.
   - **Built:** add finished items. Be concrete about the feature, where it lives and whether it's tested.
   - **In progress:** what's partly done, the branch it's on and what's left.
   - **Next:** the next 1–5 concrete steps, in order.
   - **Blockers:** anything missing (keys, decisions, access), stated exactly. Remove blockers that were resolved.
   - **Users and validation:** any real usage, feedback or validation results.
   - Remove items that are no longer true. STATUS describes the present, not a history.

3. **Update `docs/DECISIONS.md`.** Append one line per decision made this session, in the format
   `- YYYY-MM-DD — <decision> — <why>`. Include decisions that changed a spec, picked a library or
   model, or set a threshold. Never rewrite or delete earlier entries. If a decision reverses an
   earlier one, add a new line that says "supersedes <date> <short ref>".

4. **Check the specs.** If the code now differs from anything in `docs/specs/`, update the spec
   (or list the mismatch under Next in STATUS).

5. **Commit** the doc updates with the message `docs: wrap session — update STATUS and DECISIONS`
   and push to the current branch.

6. Reply with a 5-line summary: what was built, what's next, and any blockers.

$ARGUMENTS
