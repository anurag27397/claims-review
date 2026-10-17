---
name: claims-review
description: Review a PR/MR by first checking that its description states the change the system needs, turning that description into numbered claims, then having a panel of single-angle reviewers verify the code against those claims. Every comment points back to a claim. Use when the user says /claims-review, asks to review a PR or MR or branch "against its description", or wants fewer, more trustworthy review comments.
---

# /claims-review

AI review is good at the trees and bad at the forest. This review answers two questions, in order:

1. Does the PR description describe the change the system needs?
2. Does the code do what the description says it does?

Don't move to question 2 until question 1 has a clear yes.

## Inputs

Collect these before reading any code for quality:

- **PR/MR description**: `gh pr view <n> --json title,body,closingIssuesReferences` or `glab mr view <n>`. With no PR, ask the user for the intent. Don't infer it from the diff.
- **Ticket or RFC**: linked issues, or a link in the description.
- **Related PRs and code**: other PRs the description mentions, plus any repos or services the change touches.
- **System notes**: how the system is built. Look in `CLAUDE.md`, `ARCHITECTURE.md`, `docs/`, ADRs, and `memory/`.
- **The diff**: `git diff <base>...HEAD` (use the merge base, not the tip of base).

## Step 1: Look at the forest

Compare the description with the ticket, related PRs and code, and the system notes. **Stop** if any of these is true:

- The goal, or the reason for it, is unclear.
- The description contradicts the ticket.
- The diff contains changes that serve no part of the stated goal (unrelated refactors, drive-by fixes, extra features).

When you stop, don't review the code. Draft numbered questions for the author so that everything becomes clear: what is unclear, what looks unrelated, and what would resolve it. Give the questions to the user and end there.

## Step 2: Turn the description into claims

Rewrite the description as a numbered list. Each claim has:

```
C1. <change>: what this part of the PR does
    Outcome: what it promises will be true afterward
    Files: where it is implemented
```

Claims come from the description and the ticket. Use the diff only to split or locate them, never to invent intent the author didn't state. Show the list to the user before continuing.

**Code that does something no claim mentions is a finding by itself.** Tag it `UNCLAIMED`.

## Step 3: Check the claims with a panel

Spawn separate reviewer subagents in parallel, each owning one angle. Pick angles from what Step 1 found, the risks, and the areas the code touches. Don't use a fixed set:

| Angle | Use when the change touches |
|---|---|
| security | auth, permissions, user input, secrets, data exposure |
| failure modes | I/O, retries, timeouts, partial failure, empty or null inputs |
| architecture | public contracts, cross-module coupling, API compatibility |
| data integrity | schemas, migrations, validation, deploy ordering |
| performance | hot paths, loops over queries, unbounded work |
| UX | user-visible behavior, copy, accessibility |

A one-line docs fix needs none. A payment-flow change might need five. Every reviewer gets the full claims list, the diff, and read access to the repo. Tell each reviewer to stay on its angle and to cite a claim ID (or `UNCLAIMED`) on every finding.

## Step 4: The bar for "clean"

The bar is the same for every PR size. Before a reviewer can call its part of the change clean, it has to show:

- **What the code takes and returns**, or what it mutates.
- **That it found every caller.** Grep for each changed function, class, endpoint and config key, then list the call sites it checked. Count the callers that tests reach as well.
- **That it works in prod context**: multiple tenants, caches (stale entries, key shape), concurrent requests, and deploy order between services, migrations and feature flags. If none of these apply, say why.
- **That a test actually exercises the outcome**: name the test, and confirm it would fail if the claimed outcome broke. A test that only runs the code path doesn't count.

If a reviewer can't show all four, the result is "not proven", not "clean". Report it as such.

## Step 5: Report

Group everything by claim, then dedupe across reviewers and drop anything you can't tie to specific lines.

```
## Review: <APPROVE | REQUEST CHANGES | NEEDS HUMAN REVIEW>

Goal: <one sentence>

C1. <change> → <outcome>: verified | issues | not proven
  - path/to/file.py:42 [HIGH, failure modes] <what's wrong, why it breaks the outcome, fix>

UNCLAIMED
  - path/to/other.py:10 [MEDIUM] <behavior the description doesn't mention>

Not proven clean
  - path/to/file.py (security): no test exercises the outcome; callers in jobs/ not checked
```

Status rules:
- Any CRITICAL or HIGH finding, or any UNCLAIMED change, means REQUEST CHANGES.
- Otherwise, anything not proven means NEEDS HUMAN REVIEW.
- Otherwise APPROVE, with any lower-severity comments listed.

Every comment says which claim it is about and why it came up. Aim for fewer comments you can trust, not more.
