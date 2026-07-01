# Git Decision Guides

> **Navigation:** [`← Back to git/`](../) | [`Case Studies →`](../case-studies/) | [`Production Incidents →`](../production-incidents/)

Decision fatigue is real in Git. The same operation has multiple implementations, and choosing the wrong one in production has real consequences. These guides remove the ambiguity.

Each guide is a flowchart you can follow under pressure — during a hotfix, during an incident, during a code review. No theory. Just the decision and the command.

---

## Guides

| Guide | When to Use |
|---|---|
| [Merge or Rebase?](merge-or-rebase.md) | Integrating branch changes — most common daily decision |
| [Reset or Revert?](reset-or-revert.md) | Undoing changes — the most dangerous decision in Git |
| [Cherry-Pick or Merge?](cherry-pick-or-merge.md) | Moving specific commits across branches |
| [Stash or Branch?](stash-or-branch.md) | Handling interrupted work |
| [Squash or Not?](squash-or-not.md) | Commit history cleanup before merge |
| [Which Branching Strategy?](branching-strategy.md) | Choosing a team workflow model |

---

## How to Use These Guides

These guides are not academic. They are decision frameworks for production engineers.

Each guide:
- Opens with the question you are actually asking
- Provides a flowchart you can follow
- Explains the consequence of each path
- Gives you the exact command
- Tells you when to reverse course

If you find yourself asking "should I merge or rebase?" open the guide, follow the chart, run the command.

---

## Engineering Notes

The most common Git mistakes happen when engineers reach for the wrong tool not because they don't know it exists, but because they haven't internalized when to use it. These guides exist to close that gap.

Keep these bookmarked. The decision of "merge vs rebase" will come up hundreds of times in your career. It should take 10 seconds to answer, not 10 minutes of debate in a PR comment thread.
