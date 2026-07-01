# Git — Automation Lab | BuildWithAkash

A production reference for Infrastructure Engineers, Platform Engineers, SREs, and DevOps practitioners. Not a beginner tutorial. Every section covers Git under real conditions: large teams, time pressure, regulated environments, multiple remotes, long-lived branches.

If you need a beginner introduction, start with the [Pro Git book](https://git-scm.com/book/en/v2). Return here when you need production depth.

---

## Tools Required

| Tool | Purpose | Install |
|---|---|---|
| Git 2.35+ | Core (SSH signing, `--staged` stash) | `brew install git` |
| gitleaks | Secrets scanning in hooks and CI | `brew install gitleaks` |
| git-filter-repo | Safe history rewriting | `pip install git-filter-repo` |
| pre-commit | Team-wide hook distribution | `pip install pre-commit` |
| git-sizer | Large repository diagnosis | `brew install git-sizer` |
| GPG | Commit and tag signing | `brew install gnupg` |
| GitHub CLI (`gh`) | Branch/PR automation in aliases | `brew install gh` |

> Cross-references: Each section links to related sections. `hooks/` links to `security/` for secrets scanning; `recovery/` links to `internals/` for reflog mechanics; `enterprise-workflows/` links to all four of the above.

---

## Engineering Philosophy

Git is not a backup system. It is an engineering communication tool. The commit history is a conversation between the engineer who wrote the code and every engineer who will maintain it after them — including the author six months later.

| Principle | What it means in practice |
|---|---|
| Every commit should be deployable | A broken commit on a shared branch blocks the entire team |
| The message explains *why*, the diff explains *what* | Future engineers read messages to understand decisions, not code |
| The branch model must match the deployment model | Trunk-based works for daily deploys; release branches work for versioned products |
| Rebase cleans history, merge preserves it | Choose based on audit requirements, not personal preference |
| Force push to shared branches is a production incident | It rewrites history that others have based work on |
| The reflog is your recovery mechanism | Almost nothing is permanently lost in Git |
| Tags are immutable release references | Never move a tag that has been deployed against |
| Branch protection is not optional | In any regulated or team environment, protect `main` |

---

## Folder Map and Reading Order

```
git/
├── README.md                ← You are here
│
├── internals/               ← Object model, packfiles, GC, SHA mechanics
├── fundamentals/            ← Three trees, HEAD, reset, staging granularity
│
├── branching/               ← Branch models: GitFlow, trunk-based, GitHub Flow
├── merging/                 ← Merge strategies, conflicts, rerere
├── rebasing/                ← Interactive rebase, golden rule, autosquash
├── cherry-pick/             ← Surgical commit application, backports, hotfixes
├── stash/                   ← Named stashes, partial stash, stash-to-branch
├── tags/                    ← Annotated tags, semver, git describe, release mgmt
│
├── hooks/                   ← pre-commit, commit-msg, pre-push, pre-commit framework
├── security/                ← Secrets detection, GPG signing, SSH, CODEOWNERS, audit
├── performance/             ← LFS, sparse checkout, partial clone, git maintenance
├── recovery/                ← Reflog playbook, 6 recovery scenarios, git bisect
│
├── best-practices/          ← Commit standards, gitignore, gitattributes, hygiene
├── enterprise-workflows/    ← GitOps, release branching, compliance, monorepos
├── troubleshooting/         ← Field guide: 15 production failure scenarios
│
├── decision-guides/         ← Mermaid flowcharts: merge-or-rebase, reset-or-revert, and 4 more
├── case-studies/            ← 9 enterprise case studies with root cause and resolution
└── production-incidents/    ← 5 P1/P2 incident playbooks: diagnosis, commands, prevention
```

---

## Recommended Study Tracks

### Track 1 — Understanding Git (before you can reason about anything else)
1. [`internals/`](internals/) — What Git actually stores
2. [`fundamentals/`](fundamentals/) — Three trees, HEAD, reset mechanics

### Track 2 — Daily workflow mastery
3. [`branching/`](branching/) — Branch models and naming
4. [`merging/`](merging/) — Merge strategies and conflict resolution
5. [`rebasing/`](rebasing/) — Interactive rebase, autosquash
6. [`stash/`](stash/) — Context switching without committing

### Track 3 — Production operations
7. [`cherry-pick/`](cherry-pick/) — Backports and hotfixes
8. [`tags/`](tags/) — Release management and git describe
9. [`hooks/`](hooks/) — Automated quality enforcement
10. [`recovery/`](recovery/) — Getting back everything you thought was lost

### Track 4 — Enterprise and security
11. [`security/`](security/) — Secrets, signing, CODEOWNERS, supply chain
12. [`performance/`](performance/) — LFS, sparse checkout, git maintenance
13. [`enterprise-workflows/`](enterprise-workflows/) — GitOps, compliance, incident response
14. [`best-practices/`](best-practices/) — Standards, gitignore, gitattributes
15. [`troubleshooting/`](troubleshooting/) — 15 production failure scenarios

### Track 5 — Decision frameworks and real-world scenarios
16. [`decision-guides/`](decision-guides/) — Flowcharts for merge-or-rebase, reset-or-revert, branching strategy selection, and 3 more daily decisions
17. [`case-studies/`](case-studies/) — 9 enterprise scenarios with problem, root cause, commands, and lessons learned
18. [`production-incidents/`](production-incidents/) — 5 P1/P2 incident playbooks ready to use during active incidents

---

## References

| Resource | URL |
|---|---|
| Pro Git (official book) | https://git-scm.com/book/en/v2 |
| Git Reference Manual | https://git-scm.com/docs |
| Conventional Commits | https://www.conventionalcommits.org |
| Semantic Versioning | https://semver.org |
| GitHub Flow | https://docs.github.com/en/get-started/quickstart/github-flow |
| Trunk Based Development | https://trunkbaseddevelopment.com |
| GitOps Principles | https://opengitops.dev |
| gitleaks | https://github.com/gitleaks/gitleaks |
| pre-commit framework | https://pre-commit.com |
| Git LFS | https://git-lfs.com |
