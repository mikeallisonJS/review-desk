# Review Desk: the recipe

Recipe version: **2026-11-27** (see `CHANGELOG.md`)

This is a recipe for your agent, not a product to install. Hand it this folder (the `review-desk` skill: this
file, `SKILL.md`, `CHANGELOG.md` and `templates/`), and it builds a Review Desk shaped to how you work: which
people, which repos, which chat, which agent harness, and whether it runs inside the harness or as a small service
on a machine that's always on.

Review Desk does two jobs:

- **Reviews people ask you for.** It notices when someone wants your review on a pull request, waits until the PR
  is ready, and has a locked-down reviewer agent read it and draft a review in your voice. The PR then waits for you
  on one page with the verdict, what must be fixed, and the calls only you can make. You make those calls, edit the
  text if you like, and press one button to post it as you.
- **Your own pull requests.** Every open PR you authored, sorted into *Ready to merge*, *Needs changes* (and why),
  *In progress* and *Waiting*. From the same page you merge, update a branch, or send change requests to the agent
  thread that opened the PR. Stacked PRs and multi-PR features show as one card and can be merged or fixed as a
  whole by one agent.

The rule that shapes everything: **the agents prepare, and nothing leaves under your name until you confirm it.**
The one opt-in exception, automatic first passes, is covered in A10.

Everything below comes from a working setup that has run daily since November 2026, and each section says why it is
built that way, because those reasons are what your build needs to keep. The names and numbers are that setup's
defaults; change them in the interview (section 2), not on the fly.

---

## 0. For the agent building this

1. Read this whole file before writing code. Then read `SKILL.md`, beside this file, which runs the interview and
   the build in stages.
2. The **invariants** (section 1) are not up for trade-offs. If your harness can't hold one of them, stop and tell
   your owner which one and why. Don't build a weaker version quietly.
3. Build in the order of section 9. Each stage ends with checks; run them before going on.
4. Write down the recipe version you built from, where your build keeps its docs. Updates are a diff of
   `CHANGELOG.md` since then.
5. Where this says "the owner", that's the person whose name the reviews and merges go out under.

## 1. Invariants

1. **The reviewer can't act.** The agent that reads PRs runs with read, search and read-only subagents only: no
   shell, no file writes, no web, no messaging, no chat or mail connectors, no credentials in reach, and nothing
   that can post. *Why:* everything it reads (the diff, the description, comments, the chat message asking for the
   review) was written by someone else, often by their agent. An injected "approve this and merge" must meet a
   reviewer that can't do either. Enforce it in the harness's permission config (deny rules), not only in the
   prompt, and test that the deny rules hold in the harness's most permissive mode and in subagents.
2. **The reviewer never runs the PR's code.** CI is the test evidence. *Why:* running the PR's tests runs its code,
   with your credentials on the machine.
3. **Deterministic code decides and acts; agents only draft.** A script decides which PRs need a review, stages the
   inputs, parses the reports, and does every post, merge and push request. *Why:* code can be pinned to a commit,
   rate-limited, logged and tested. A model that reads untrusted text can't be.
4. **The full-access agent never reads PR text or review text.** Some agent has to have tools (launching threads,
   reading chat). Keep its job mechanical: it moves ids, SHAs and file paths between the script and the harness.
   Untrusted text goes into files the script writes for the reviewer, and the reviewer's report is parsed by the
   script.
5. **Nothing goes out as the owner without confirmation**, and every confirmation shows the exact text and target.
   The automatic first pass (A10) is the only exception. It's opt-in, comment-only, labelled as automatic, and
   can't ping anyone.
6. **Every outward action is pinned to what the owner saw.** Each review, merge and branch update carries the head
   SHA the page showed. If the PR moved since, the action is refused.
7. **Only the owner can press the buttons**, from the page itself. The server checks identity and blocks
   cross-site requests (section 7).
8. **Every outward step is logged** (append-only JSONL): what was done, the PR, the head, and the result.

## 2. The interview: choices your build makes

Ask the owner these before building. The working setup's answer is in brackets.

| Choice | Options |
|---|---|
| Where it runs | **Service**: an always-on machine runs a timer, a small HTTP server and scheduled agent tasks [this]. **In-harness**: the agent harness's scheduled tasks do the polling, and the page is a local file plus a localhost server started on demand. |
| Code host | GitHub [this]. The recipe uses GitHub's API throughout; another host needs its equivalents of each call in section 8. |
| Which reviews | Requests from allowlisted people in chosen orgs [one org, a handful of people]. Allowlist entries list every login that counts as that person, so a person and their bot share one entry. |
| Chat | Slack DMs from allowlisted people also ask for reviews, and a prepared reply goes back [Slack]. Or none. |
| Reviewer model | A strong model at high effort [a mid-size model at max effort; the largest model for auth, tokens or payments]. |
| First passes (A10) | On or off [on]. |
| Your own PRs (part B) | On or off [on]; which repos to leave out (fixtures, archived, probes). |
| Fix dispatch (B6) | Send change requests to the agent thread that opened the PR, else a new thread in that repo [this], or only copy text for the owner to paste. |
| Page access | Behind an identity-aware proxy that sets a user header (Tailscale Serve, Cloudflare Access, oauth2-proxy) [Tailscale Serve], or bound to localhost. |
| Merge defaults | Use GitHub's `viewerDefaultMergeMethod` and only the methods the repo allows [this]. |
| Agents on the page (C6) | On or off [on]. |

## 3. Architecture

```
            timer (1 min)                         scheduled agent task (30 min, or a "Check now" webhook)
                 |                                              |
   script: harvest reports, own PRs, render        watcher agent (full access, mechanical):
   page + data.json, post due first passes            reads chat DMs -> inbox.json (verbatim, no shell)
                 |                                     runs `script poll` -> {launch, send}
                 v                                     launches / messages reviewer threads, records ids
   page (static HTML + data.json, refresh 1 min)                     |
                 |  buttons (POST, guarded)                          v
                 v                                  reviewer threads (locked down, read-only)
   script serve: post review | merge | update branch   read staged files, write reports in a fixed format
                 | queue chat reply | queue fix | ask review | check now
                 v
   outbox/, fixes/ queues --webhook--> relay agent (sends confirmed chat text unchanged)
                                   --> fix dispatcher agent (delivers confirmed change requests to a thread)
```

One script with subcommands does all the deterministic work. In the working setup that's about 2,300 lines of
Python with no dependencies beyond `gh` and `git`:

| Subcommand | Does |
|---|---|
| `poll [--chat inbox.json]` | Finds PRs that need a review (A1–A4), stages them, prints `{launch: [...], send: [...]}` for the watcher agent. Stands down if another poll ran in the last 5 minutes. |
| `record <thread> <pr>...` | Notes which thread holds which PRs and heads. |
| `dashboard` | Harvests reports (A6), builds the own-PR data (part B), writes `index.html` and `data.json`. |
| `first-pass [--dry-run]` | Posts due first passes (A10). Run before each render. |
| `serve` | The page and its buttons (section 7). |
| `outbox claim/sent/failed`, `fixes claim/sent/failed` | The queues the relay and fix-dispatcher agents work through. |
| `install` | Installs the reviewer's instructions and permission file into its workspace. |
| `seed` | Marks PRs already waiting as handled, when you first turn it on or add a person. |

Files:

- `allowlist.json`: who may ask, with each person's logins, chat user id, DM channel id and default repo (for a
  bare `#123`).
- `state.json`: tracked PRs (head last reviewed, thread id, verdicts per head, open or closed).
- `actions.jsonl`: every button press and outward step.
- `prs/<repo>-<n>/`: staged inputs per PR (A3). `outbox/`, `fixes/`: queued items, one JSON file each, moving
  `queued/ -> claimed/ -> sent/ or failed/`.
- `mine.json`: last known mergeability per PR head (B1).

Agents:

| Agent | Access | Started by |
|---|---|---|
| Watcher | Full: chat read, launch and message threads, run the script | A schedule (30 min) and a webhook (*Check now*) with the **same prompt** |
| Reviewer | Read-only, locked down (invariant 1) | The watcher, one thread per author per run |
| Chat relay | Chat send, run the script's `outbox` subcommands, nothing else | A webhook the server calls after the owner confirms a reply |
| Fix dispatcher | List, launch and message agent threads, run `fixes` subcommands | A webhook the server calls after the owner confirms change requests |

Prompt templates for all four are in `templates/`. Keep every agent prompt short and mechanical, with explicit
"never" rules. Each one reads some untrusted words, so the less judgment it exercises the better.

## 4. Part A: reviews people ask you for

### A1. Triggers

A PR is a candidate when any of these holds:

- **Requested on GitHub:** open, not a draft, in a chosen org, authored by an allowlisted login, with the owner in
  its requested reviewers. A re-request after new commits counts again.
- **A ping after your review:** the head moved since the owner's last review, and the author commented after that
  review.
- **Asked in chat:** a DM to the owner from an allowlisted person (sent by that person, under 48 hours old) that
  names a PR, as a full URL or a bare `#N` read against their default repo. A line with "no review", "FYI" or
  "merged" is context, not an ask.
- **After a first pass:** any new push to a PR that got an automatic first pass (A10).
- **The owner asked**, from the page (B7).

Never review the same head twice. Wait while the last commit is under 5 minutes old, since it may still be moving.

Chat text crosses into the script as a JSON file the watcher writes with its file tool, never as a shell argument.
The script parses it, and it reaches the reviewer only as quoted data.

### A2. The readiness gate

Only PRs that look ready get reviewed (*why:* a review of an unready PR is wasted and comes back stale). A PR waits,
shown as *Not ready yet* with the reason and checked again every run, when it:

- is a draft, or has merge conflicts;
- has a label matching `wip|work in progress|do-not-merge|dnm|on hold|hold|blocked|stale`, or a title starting with
  `WIP`, `[DNM]` or `Draft`;
- has no checks at all, or the **latest run** of any check failed, was cancelled, or is still running.

Use the latest run per check (`gh pr checks --json name,bucket,workflow`), because GitHub's `statusCheckRollup`
keeps a superseded failure even after a re-run passed.

### A3. Staging

For each PR, write `prs/<repo>-<n>/`. Nothing in it is ever run.

- `checkout/`: a detached worktree at the PR head, from a bare cache clone per repo. Fetch
  `refs/pull/<n>/head` and the base branch. If the fetched head isn't the head GitHub reported, the PR is still
  moving: skip it this round.
- `diff.patch`: `--stat` then the full diff, against the **merge base** with the base branch. For a stacked PR the
  base is the PR below it, so the diff holds only what this PR adds.
- `delta.patch` (re-reviews): last reviewed head to the new head. Before fetching, keep the old head reachable with
  a ref (`refs/reviewed/<n>`), so a force-push doesn't lose it.
- `commits.txt`; `pr.md` (title, author, base, head, size, why it was staged, CI per check, files, then the
  description fenced as `<pr-body>` and labelled untrusted); `comments.md` (reviews and comments, each fenced and
  labelled untrusted); `request.md` (the chat asks, fenced).

Delete the checkout when the PR closes. They're big: a monorepo's can be 250 MB or more each.

### A4. Threads

- **One reviewer thread per author per run**, holding all of that author's new PRs. The reviewer sees stacks and
  contradictions between them, and drafts one chat reply for the batch. (*Considered:* one thread per PR, which
  misses cross-PR problems and leaves you five threads to read; one thread per run, which mixes people whose replies
  go to different DMs.)
- **A re-review goes back to the thread that last reviewed that PR**, with the delta, so it can say which earlier
  points are fixed.
- **A replacement PR goes to the earlier thread too:** one that shares a closing issue or a branch with a PR already
  reviewed, so it's reviewed against the earlier findings.
- Hide review threads from the harness's sidebar (snooze or the equivalent): the page is where you look.
  **Never archive them**, because an archived thread may refuse messages, and archiving a running one stops it.

### A5. The reviewer's contract

`templates/reviewer.md` is the reviewer's instructions. Install it as the reviewer workspace's agent instructions
file. `templates/reviewer-permissions.json` is the lockdown, in Claude Code's settings format; translate it for
other harnesses. The parts the rest of the system depends on:

- **The heading line**, exactly: `## <repo>#<n>: <title> — <Approve | Approve once fixed | Comment, needs work |
  Do not merge>`. The script finds reports by it.
- Numbered lists under `**Must fix before merge**`, `**Should fix**` and `**For you to decide**`. The page shows
  their counts.
- Each *For you to decide* item opens with its question in bold, then options as `- **(a) <label>.** <cost>`
  bullets, then `- **Recommendation:** <letter and why>`. The page turns these into picks.
- `### Prepared GitHub review (as <owner>)` and `### Prepared <chat> reply to <first name>`: the texts the buttons
  post.
- For a batch, one report per PR as each finishes, then a batch message with a table and the batch's chat reply.

A reviewer that changes the heading's shape drops off the page (its row stays *Reviewing*, then *No report yet*)
but not out of the review. Make that state visible, so a broken format gets noticed.

### A6. Reading the reports

The script reads the reviewer threads' **transcripts** (for example Claude Code's JSONL session files in the
reviewer workspace's project folder), not the threads through the harness API, and matches report messages by the
heading line. It keeps per PR and head: the verdict, the counts, the decisions, and the prepared texts. Store the
prepared texts on disk, and serve them only when the owner opens a button. The page's data carries only verdicts
and counts. *Why:* the watcher agent never sees review text (invariant 4), and the page stays light.

Report the time a thread has been reviewing. Over about 3 hours without a report, show *No report yet*.

### A7. Lanes

Sort every tracked PR by whose move it is:

- **Needs you:** reviewed, and the owner hasn't reviewed that head on GitHub yet. The page's top line counts the
  decisions waiting there.
- **With the reviewer:** *Queued*, *Reviewing*, *No report yet*.
- **Waiting on the author:** the owner's latest review asked for changes or commented; *New commits* (moved since
  the review, with no re-request or ping yet); *Not ready yet* (with the reason).
- **Approved:** the owner approved the latest head, not merged yet.
- **Done:** nobody's move: marked handled, then merged or closed PRs for a week.

### A8. Posting a review

**Post review…** shows the prepared review, editable, and asks which event to post: comment, approve, or request
changes. The verdict's suggestion is picked first: *Approve* approves, *Approve once fixed* and *Comment, needs
work* comment, *Do not merge* requests changes. On confirm the server posts through the API **pinned to the
reviewed commit** (`commit_id`). It refuses if the head moved, and refuses the same event twice on one commit.

### A9. Decisions go back to the reviewer

When a report has *For you to decide* items, the button reads **Decide and post review…** and shows each one first:
the question, the options to pick (the recommended one marked), the reviewer's notes, and a box for your own words.
If you answer any:

1. The server writes your calls, and any edits you made to the review text (between `<owner-edit>` tags), to
   `prs/<repo>-<n>/calls.md`, then wakes the watcher.
2. The watcher sends the review thread **only that file's path**. Neither your words nor the reviewer's questions
   pass through the full-access agent.
3. The reviewer writes a new report for the same commit with your calls worked in. Each call becomes a must-fix or
   should-fix item, or a point in the review in your voice, not an appendix. *Why:* appending your answers to the
   prepared text left points that contradicted them.
4. The PR waits under *With the reviewer* ("Your calls are with the reviewer"), then returns to *Needs you*. A
   revised report never posts as a first pass.

If you answer none, it posts as it stands.

### A10. Automatic first passes (opt-in)

*Why:* the closer the author gets the PR to ready before your attention, the better. A report qualifies when it has
must-fix items (or should-fix items, on the PR's first round only), **no** *For you to decide* items, and a verdict
other than *Do not merge*. Within a minute, the script posts its prepared review as a **COMMENT** review on the
reviewed commit, from your account, under a line such as *"Automated first pass from <owner>'s review setup.
<Owner> hasn't read this yet; they review once these points are addressed."* Then:

- Any new push is re-reviewed without the author asking (A1). The loop runs between author and reviewer until
  nothing must be fixed. Then the PR waits for you.
- Should-fix items alone post once per PR, not every round. An author may fairly leave some, and reposting them
  would keep the PR from ever reaching you.
- An author who re-requests your review without pushing sends it to you as it is.
- The poster is the script, never the reviewer. It never approves or requests changes, and posts once per commit.
  It skips a commit you already reviewed or one the author has pushed past. It breaks every `@` before a word
  character (insert a zero-width space), so model-written text can't ping anyone.
- Never on your own PRs.

The cost, which you accept when you turn it on: text a model wrote after reading untrusted PR content appears on
authors' PRs under your name before you read it.

### A11. The chat reply

**Chat reply…** shows the prepared reply, editable. On confirm, the server writes it to `outbox/queued/` with the
person's DM channel id from the allowlist, never from the report, and calls the relay's webhook. The relay agent
claims the queue and sends each text **unchanged** to its channel. The page shows queued, sending, sent or failed.
(*Why an agent:* in the working setup the machine has no chat token of its own, and the harness's chat connector
can send. With a bot token, have the script send directly and drop the relay.)

### A12. Check now

A button that starts the watcher without waiting for its schedule, through a second webhook-triggered task with the
**same prompt** (keep the two identical). Two overlapping polls would stage the same PRs and send the same message
twice, so `poll` stands down within 5 minutes of another poll, and the server refuses a press inside that window.

## 5. Part B: your own pull requests

### B1. One query per render

One GraphQL search, `is:pr is:open author:<you> archived:false`, `first: 100`, run every render (every minute in the
working setup; about 9 rate-limit points). Fields:

```graphql
id number title url isDraft mergeable mergeStateStatus reviewDecision headRefOid headRefName baseRefName
updatedAt createdAt viewerCanUpdateBranch additions deletions changedFiles
stack { number size } stackEntry { position }          # GitHub's own stacked PRs, where available
repository { nameWithOwner squashMergeAllowed mergeCommitAllowed rebaseMergeAllowed viewerDefaultMergeMethod
  defaultBranchRef { name } }
closingIssuesReferences(first: 5) { nodes { number title url blockedBy(first: 20) { nodes { number } }
  parent { number title url parent { number title url parent { number title url } } } } }
latestOpinionatedReviews(first: 10) { nodes { author { login } state } }
reviewThreads(first: 100) { nodes { id isResolved } }
commits(last: 1) { nodes { commit { committedDate statusCheckRollup { contexts(first: 80) { nodes { __typename
  ... on CheckRun { name status conclusion startedAt checkSuite { workflowRun { workflow { name } } } }
  ... on StatusContext { context state createdAt } } } } } } }
```

Send the header `Accept: application/vnd.github.merge-info-preview+json`.

- **Checks:** keep only each check's latest run, by start time per `workflow / name` (B3).
- **Mergeability is often `UNKNOWN` on a cold read.** Cache the last known `mergeable` and `mergeStateStatus` per PR
  **head**, and use it while GitHub works it out. Show "GitHub is still working out whether it can merge" when
  there's no cached value.

### B2. Statuses and reasons

- **Ready to merge:** not a draft, `mergeable == MERGEABLE`, `mergeStateStatus` in `CLEAN` or `HAS_HOOKS`, and no
  reason below.
- **Needs changes**, each reason listed on the row:
  - conflicts (`CONFLICTING` or `DIRTY`);
  - failing checks (latest run only);
  - changes requested (by whom);
  - the reviewer's must-fix items, if you asked for a review (B7);
  - behind its base (`BEHIND`);
  - **unanswered review threads:** unresolved, with no comment from you, whoever opened them. Review bots such as
    CodeRabbit only comment, so GitHub calls the PR clean while their threads sit unanswered. Count them per
    author, noting how many are on code that has since changed (`isOutdated`). Fetch the comments in a **second
    query**, `nodes(ids: [...])` on just the unresolved thread ids, with `... on PullRequestReviewThread { id
    isOutdated comments(first: 30) { nodes { author { login } } } }`. *Why:* asking for every thread's comments
    inside the search costs about 50 points a run instead of 1.
- **In progress:** changes were sent to an agent and nothing has been pushed since; the reviewer is on it; or CI is
  running and nothing else is wrong.
- **Waiting:** draft, a review needed, a branch rule, or GitHub still deciding.

### B3. CI running must be obvious

When checks are running on the head, show a chip on the row (and on its stack card) with a spinner, how many checks
are running, and since when, with the names on hover. Respect `prefers-reduced-motion`.

### B4. Merge

- One press merges with GitHub's default for the repo (`viewerDefaultMergeMethod`: the repo's default, or your last
  method there). A menu beside it offers the other methods **the repo allows**, plus "More options…" (keep the
  branch).
- The server re-reads the PR. It refuses unless the head is the one the page showed, and GitHub would merge it now
  (mergeable, clean, not a draft).
- Merge through the **asynchronous merge API**: `PUT /repos/{o}/{r}/pulls/{n}/merge-async` with `merge_method` and
  `sha`. It answers `pending` with a `uuid`. Poll `GET .../merge-async/{uuid}` every few seconds for up to a minute,
  until `merged`, `enqueued` or `failed`. A 409 that carries a uuid means a merge is already running: poll it.
  *Why:* `gh pr merge` and the synchronous API refuse a PR in one of GitHub's stacks ("must be merged using the
  asynchronous merge REST API"). Merging a PR in a GitHub stack also merges the open PRs below it.
- After a merge, delete the merged branches, except one another open PR uses as its base or head. GitHub retargets
  that PR.
- **One merge at a time per repo.** Hold the other merge buttons in a repo for two minutes after a merge there,
  while GitHub re-checks the rest against the new base, and while an agent is merging a stack there (B8). Say why on
  the page, and refuse on the server too.

### B5. Update branch

Show **Update branch** when `viewerCanUpdateBranch` is true and the branch doesn't conflict. One press merges the
base in (GitHub's own button does the same). Its menu offers a rebase, **left out when another open PR is stacked
on the branch**, since rewriting it would strand that PR. Use GraphQL:

```graphql
mutation($id: ID!, $head: GitObjectID!, $how: PullRequestBranchUpdateMethod!) {
  updatePullRequestBranch(input: {pullRequestId: $id, expectedHeadOid: $head, updateMethod: $how}) {
    pullRequest { headRefOid } } }
```

Then poll up to 30 seconds for the new head. The button keeps a loading state until the head changes (or a few
minutes pass). Not while a merge is under way in the repo, or an agent is working on the branch.

A conflicting branch gets **Resolve conflicts** instead: a *Send changes* (B6) with fixed instructions: rebase onto
the base, resolve, run the repo's checks, push, and stop on a conflict that needs a decision. Its menu lets you edit
the instructions first.

### B6. Send changes

**Send changes…** opens a notes box, prefilled from the reasons (one instruction line per reason) and, if the
reviewer looked at it, its prepared review. On confirm, the server queues the item in `fixes/` and calls the fix
dispatcher's webhook. The dispatcher sends the message **unchanged** to:

1. the last thread that worked on this PR (or group), else
2. the agent thread that opened the PR (search the harness's thread listings, on every machine it can reach, for a
   thread whose PRs include it), else
3. a new thread in that repo's harness project, in a worktree on the PR's branch.

The message tells the agent: work in a worktree, check out the PR, make the changes, run the repo's checks, push to
the same branch, open no new PR, and if it's stacked, rebase the PRs above it onto the new head
(`git rebase --onto <new head> <old head> <branch>`, push with `--force-with-lease`). For unanswered review threads,
it says to fix each comment that's right and reply on its thread, reply with the reason to one that's wrong, then
resolve it (`gh pr view` doesn't list them; query `reviewThreads`).

The row shows *Changes sent* and a link to the thread until the next push. (*Why a thread, not a PR comment:*
agent threads don't see PR comments, and GitHub won't let you request changes on your own PR.)

### B7. Review on request

A **Review** button runs the reviewer on one of your own PRs, on request only. It skips the allowlist and the
readiness gate, since you asked, and the reviewer writes its prepared review as change requests for the agent. No
first pass ever posts on your PRs. (*Why on request:* with dozens of open PRs, mostly bots' dependency bumps,
reviewing all of them costs a run each.)

### B8. Stacks and features

Group your PRs per repo:

- A **stack**: a PR opened against another open PR's branch.
- A **feature**: PRs whose closing issues share a top-level parent issue (GitHub sub-issues, up to three levels).
- A **dependency** also joins a group: a PR closing an issue that is *blocked by* an issue another open PR closes.

Show each group as one card, named by the parent issue, collapsible to a one-line tally.

- **Order** (the page and the agent use the same one): each PR after everything it needs (the PR it's stacked on,
  and the PRs closing its blocking issues). Among the PRs free to go next, a PR stacked on the one just placed comes
  first, so the stack reads as a tree. After that, lowest issue number, then lowest PR number. Number the rows.
- **Indent** each stacked PR one step per level under the PR it's built on, with an elbow line pointing back up to
  it. Label rows *stacked on #N* or *after #N*.
- **A stacked PR has no merge button of its own.** Merging it alone would land it in the branch below, not the
  default branch. Nor does a PR whose blocking PRs are still open. The exception is GitHub's own stacks: merging an
  upper PR there merges the ones below it, so once everything below it is ready, it gets *Squash and merge with
  #N, #M*.
- **Merge the stack** (method asked; offered once every PR merges cleanly into its own base) hands the group to one
  agent through the fix dispatcher. It merges bottom-up with the async merge API. Between merges it retargets the
  next PR to the default branch, rebases it onto the new base (dropping commits that already landed), and waits for
  its checks. It stops and reports on anything that needs a decision. It never passes `--admin` or bypass flags,
  never force-pushes a branch outside the list, and never merges a PR outside the list.
- While a stack merge is queued or running, the repo is **locked**: other merge buttons there wait. The lock lifts
  when its PRs are all closed, after two hours with no change to any of them, or when you press *It has stopped:
  allow merges*.
- **Send changes to the stack…** sends one set of notes for the whole group, worked bottom-up, restacking as it
  goes.

## 6. Part C: the page

### C1. Shape

- A static `index.html` with its data inlined, plus `data.json`, rewritten every minute. An open page refetches
  `data.json` every minute and re-renders in place, keeping open panels, focus and scroll.
- **Tabs**: *All* (the default on each load), *Needs you*, *Yours*, *With the reviewer*, *Waiting on the author*,
  *Approved*, *Done*, each with its count. The counts in the top line open their tab, and `#section` links open
  on their tab.
- **Filters**: organization and repository (narrowing everything, including the top line), person, and sort (most
  urgent, newest, oldest, last pushed, smallest, largest, by repo). Remember them in `localStorage` (wrapped in
  try/catch).
- **Plain text only.** Every PR title, review text and name is set as text, never as HTML. Escape `<` in the inlined
  JSON.
- A row: status pip and label, title link, meta (repo #n, size, stacked on, GitHub stack position), the reasons,
  running CI, the verdict and counts, then the buttons. On phones, the buttons wrap under the text with no
  horizontal scroll.
- **Loading states** on every button that waits on GitHub (spinner, `aria-busy`), and an optimistic status
  (*Merging…*, *Changes sent*) that holds until the data catches up, with a few quick refreshes after an action.

### C2. Light and dark

Define colors as tokens and support `prefers-color-scheme`. Check contrast on the status colors in both themes.

### C3. Agents on the page (optional)

If you want an agent in your browser to be able to use the page:

- Name every control for its row: "Squash and merge api #953", not "Squash and merge". No two controls share a
  name.
- Expose them through WebMCP (`navigator.modelContext` / `document.modelContext`, where the browser has it) and a
  `window` object for the console. The working setup uses [UI SDK](https://ui-sdk.dev)'s Web Components and agent
  bridge (`createAgent`, `registerTools`), built from pinned versions and served by the server itself, **never from
  a CDN**: any script on this page can press buttons that act as you.
- Mark every outward control with the sentence it would do ("Merges api #950 on GitHub as <owner> (squash and
  merge), deleting the branch"). The bridge's allow hook asks you in a **modal dialog hidden from the agent**
  (excluded from its snapshot, plain HTML buttons it can't press). *Don't allow* stops it. A disabled control is
  refused without asking.
- Gotchas found in practice: close open menus before showing the dialog, because an open menu swallows the next
  outside click. Run a menu item's action after its menu has closed, because re-rendering an open menu leaves the
  agent's press hanging.
- Ship a headless-browser check that loads the page with every API call mocked and verifies: every reachable
  control can be found by name, no two share a name, every outward control asks first, the agent can't see Allow,
  *Don't allow* stops a merge, Allow lets it through, a menu choice takes one Allow, tabs switch, and the console
  shows no errors.

## 7. The server

- Bind to `127.0.0.1`. Put it behind an identity-aware proxy (Tailscale Serve sets `Tailscale-User-Login`), and
  accept button calls only when the identity header is the owner's.
- Guard against other sites pressing buttons in your browser: require a custom header (`X-Review-Desk: 1`) and a JSON
  body, which force a CORS preflight that the server never answers, and check `Origin` on every POST.
- Send `X-Frame-Options: DENY`, `Content-Security-Policy: frame-ancestors 'none'`, `X-Content-Type-Options:
  nosniff`, `Referrer-Policy: no-referrer`, and `Cache-Control: no-store` on data.
- Handle one action at a time. Cap request bodies (64 KB). Return plain error messages the page can show, and never
  a stack trace.
- Routes: `GET /api/actions` (the action log, folded per PR), `GET /api/prepared?key=` (a report's prepared texts),
  and `POST /api/review | calls | merge | update | fix | group | ask | chat | check | release`.
- Static assets the page needs (a UI bundle) are served from the repo with a content hash in the URL and an
  immutable cache header.
- In-harness variant: run `serve` on demand on localhost with a random token in the URL fragment, which the page
  sends as a header. Same checks otherwise.

## 8. GitHub gotchas

- **`gh api -F` turns all-digit values into numbers.** A GraphQL ID or SHA that happens to be all digits then fails
  with `{"data": null}` and no error. Pass string variables with `-f`, and treat `data: null` as a failure.
- **`statusCheckRollup` keeps superseded runs.** Use the latest run per check.
- **`mergeable` / `mergeStateStatus` are `UNKNOWN` on a cold read.** Cache per head (B1).
- **Stacked PRs need `merge-async`** (B4). `gh pr merge` refuses them.
- **Review bots' comments don't change `reviewDecision`.** Count unresolved threads yourself (B2).
- **Nested connections multiply rate-limit cost.** Check `rateLimit(dryRun: true) { cost }` for every query you
  add, and fetch details for the few nodes that need them in a second query by id.
- **A big search sometimes comes back cut short** (`gh`: "unexpected end of JSON input"), about one reply in fifteen
  for B1's query. Retry it, and if it still fails, keep the last good list (with the time it was fetched, so a run of
  failures can't keep it forever; 10 minutes in the working setup). Never render an empty list for a failed query: it
  reads as "nothing open".
- **Post reviews pinned to `commit_id`**, so a review can't land on a head you didn't see.
- **Turning merge commits off** (`PATCH /repos/{o}/{r} allow_merge_commit=false`) makes squash the default in the
  merge button's eyes. Only do it in repos you own or have been asked to change.

## 9. Build order and checks

Build and check one stage at a time. Each stage is usable on its own.

1. **Your PRs, read-only** (B1–B3, C1–C2): the page lists your open PRs with statuses and reasons, refreshing every
   minute. *Check:* every open PR appears once, reasons match GitHub's page, nothing runs over the rate limit
   (`rateLimit` before and after an hour).
2. **The server and its guards** (section 7), then **Merge** and **Update branch** (B4–B5). *Check:* a request
   without the identity header, from another origin, or with a stale head is refused; a merge of a stale head is
   refused; every press is in the action log.
3. **The reviewer** (A1–A7): allowlist, staging, the locked-down reviewer, the watcher task, reports on the page.
   *Check:* the reviewer's denied tools really are denied (ask it to run a shell command and to write a file in a
   test thread); a staged PR's files hold fenced, untrusted-labelled text; a second poll within 5 minutes stands
   down; a head is never reviewed twice.
4. **Posting** (A8–A9, A11–A12). *Check:* post refuses a moved head and a repeat event; calls round-trip through
   `calls.md` and come back as a revised report; a chat reply goes only to the allowlisted DM.
5. **First passes** (A10), if chosen. *Check:* `first-pass --dry-run` on real reports; no approve or request-changes
   event is ever sent; `@` is defanged; a revised (calls) report never qualifies.
6. **Send changes and stacks** (B6–B8). *Check:* the dispatcher finds the opening thread; a stacked PR has no
   lone merge button; the repo lock holds and lifts.
7. **Agents on the page** (C3), if chosen. *Check:* the headless check in C3 passes.

## 10. Costs

- Rate limit: about 10 GraphQL points per render for your PRs (once a minute is about 600 an hour), plus one
  `gh pr view` and `gh pr checks` per candidate PR per poll.
- Tokens: one reviewer run per author batch, per re-review, and per push after a first pass. The watcher, relay and
  dispatcher runs are small and mechanical.
- Disk: a bare clone per repo, plus a checkout per open PR under review.
