# Claims Review

A Claude Code skill that reviews a pull request against its own description.

AI review is good at the trees and bad at the forest. Point a model at a diff and it will find a missing null check, a slow loop, a naming nit. It won't tell you that the PR solves the wrong problem, or that half the diff has nothing to do with the ticket. This skill asks two questions in order: does the description describe the change the system needs, and does the code do what the description says?

## Install

As a plugin:

```
/plugin marketplace add anurag27397/claims-review
/plugin install claims-review@claims-review
```

Or drop the skill in by hand:

```bash
git clone https://github.com/anurag27397/claims-review.git
cp -r claims-review/skills/claims-review ~/.claude/skills/
```

Then run `/claims-review`, or ask Claude to review a PR or branch against its description. It reads PRs with `gh` and merge requests with `glab`.

## What it does

**It looks at the forest first.** Before any code review, it compares the PR description with the linked ticket or RFC, related PRs, and whatever notes the repo keeps on how the system is built. If the goal is unclear, the description contradicts the ticket, or the diff carries changes that serve no part of the goal, it stops and drafts questions for the author instead of reviewing code.

**It turns the description into claims.** Each claim is one change and the outcome it promises, with the files that implement it. Claims come from the description and the ticket, never from the diff. Code that does something no claim mentions is a finding on its own.

**A panel checks the claims.** Separate reviewers each own one angle: security, failure modes, architecture, data integrity, performance, UX. How many run depends on the risks and the areas the change touches. A docs fix gets none; a payment flow might get five.

**"Clean" has a bar.** A reviewer can only call its part clean if it shows what the code takes and returns, every caller it checked, why it holds in production (tenants, caches, concurrency, deploy order), and a test that would fail if the promised outcome broke. Anything short of that is reported as not proven.

**Every comment points back to a claim.** The report is grouped by claim, so you know what each comment is about and why it came up. You get fewer comments, and you can trust them more.

## License

MIT. Fork it and change the parts you disagree with.
