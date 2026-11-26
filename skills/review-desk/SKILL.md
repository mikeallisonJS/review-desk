---
name: review-desk
description: Build, or update, a Review Desk from its recipe. A Review Desk is a pull-request desk where locked-down agents prepare reviews, fixes and merges, and nothing goes out under the owner's name until they confirm it. Use when the user wants to set up Review Desk, build a PR review watcher or dashboard from this recipe, or bring an existing Review Desk up to a newer recipe version.
---

# Review Desk

The recipe is `RECIPE.md`, beside this file, with `CHANGELOG.md` and `templates/`. Read the recipe in full before
anything else. This skill decides *how* you build from it. The recipe decides *what* you build.

## 1. New build or update?

Look for an existing Review Desk in the user's setup: a `REVIEW-DESK.md` in the build's docs, recording the recipe
version it was built from.

- **None:** a new build. Go to 2.
- **Found:** an update. Read `CHANGELOG.md` from that version to the newest. For each entry, check whether the build
  already does it, show the user the list of what's new with one line each, and ask which to take. Build the chosen
  ones with the same care as a new stage (5), then update the recorded version.

## 2. Interview

Ask the questions in the recipe's section 2, a few at a time, with the working setup's answer as the default. Also
ask:

- Which agent harness runs the agents, and whether it has scheduled tasks, webhooks, thread launch and listing, and
  per-workspace permission files. Map each agent in the recipe's section 3 to something the harness can do. If one
  can't be done (no webhooks, say), offer the fallback the recipe names, or a timer.
- Where the script should live and what language. The recipe's working setup is one Python file with no
  dependencies beyond `gh` and `git`. Match what the user maintains.
- Which machine runs it, and how the page is reached (an identity-aware proxy, or localhost).

Write the answers to `REVIEW-DESK.md` in the build's docs folder, with the recipe version, before writing code.

## 3. Hold the invariants

Read the recipe's section 1 aloud to yourself before each stage. If the harness can't enforce one (no deny rules,
for example), stop and tell the user which one and what it would cost. Never ship a reviewer that can run commands,
write files or post, and never let a full-access agent read PR or review text.

## 4. Stages

Build in the recipe's section 9 order, skipping stages the user turned off. Stage 1 alone is useful, so get it in
front of the user early.

## 5. Each stage

1. Write the code for the stage, following the recipe's sections for it.
2. Run the stage's checks from section 9 against real data where they're read-only, and with every outward call
   mocked or pointed at a test repo where they're not. Never test a merge, review post or chat send on someone
   else's real PR.
3. Show the user what it does, with a screenshot of the page where there is one, and what you checked.
4. Record the stage as done in `REVIEW-DESK.md`, with anything you built differently from the recipe and why.

## 6. Prompts and the reviewer

Start from `templates/`: `reviewer.md` (fill in the placeholders), `reviewer-permissions.json` (translate for the
harness), and `prompts.md` (the watcher, relay and fix dispatcher). Keep their rules. Every agent prompt stays
mechanical, and every "never" in them is there for a reason the recipe gives.

Test the reviewer's lockdown before its first real PR: in a throwaway thread in the reviewer's workspace, ask it to
run a shell command, write a file, fetch a URL and read a credentials file. Each must be refused by the harness, not
by the model's choice.

## 7. When it's running

Tell the user:

- which scheduled tasks and services now exist, and how to stop each one;
- where the action log is;
- how to add a person to the allowlist (and to run `seed` first, so PRs already waiting don't all start at once);
- that updating this skill (`npx skills update`, or a fresh copy of the folder) brings a newer `CHANGELOG.md`, and
  running the skill again brings the build up to date.
