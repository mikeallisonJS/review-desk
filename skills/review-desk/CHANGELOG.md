# Changelog

What changed in the recipe, newest first. Each entry is dated, and that date is the recipe version. To bring a
build up to date, read the entries after the version it records, and take the ones you want (the `review-desk`
skill walks through it). Section numbers refer to `RECIPE.md` as of that entry.

## 2026-11-25

First published version. It covers:

- **Reviews people ask you for (4):** GitHub, chat and ping triggers; the readiness gate; staging into fenced
  files; one locked-down reviewer thread per author per run; the report format; lanes; posting pinned to the
  reviewed commit; decisions sent back to the reviewer as `calls.md`; opt-in automatic first passes; the chat reply
  through a relay; *Check now*.
- **Your own pull requests (5):** one GraphQL search a minute; ready, needs changes, in progress and waiting, with
  reasons. These include unanswered review threads (review bots such as CodeRabbit only comment), found by a second
  query by thread id, which keeps the rate-limit cost low. Also: CI running made obvious; merge through
  `merge-async`; Update branch and Resolve conflicts; Send changes through a fix dispatcher to the thread that opened
  the PR; review on request.
- **Stacks and features (5, B8):** grouping by base branch, parent issue and issue dependencies; tree order with
  each stacked PR indented under the PR it's built on; no lone merge for a stacked PR, except in GitHub's own stacks;
  *Merge the stack* through one agent; one merge at a time per repo.
- **The page (6):** tabs with *All* as the default, filters, plain-text rendering, loading states, light and dark;
  optional agents on the page through WebMCP, with a confirmation dialog the agent can't see.
- **The server (7)**, **GitHub gotchas (8)**, **build order and checks (9)**.
