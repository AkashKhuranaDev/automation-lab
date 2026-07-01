# Git Hands-On Labs

Labs are structured practice exercises. Each lab creates an isolated, disposable environment so you can make mistakes safely. Run them in `/tmp` or a dedicated `~/git-labs/` directory.

**Completion time for all labs:** ~4 hours  
**Target audience:** Engineers who want to go beyond reading and develop muscle memory for production scenarios

---

## Index

| # | Lab | Skills Practiced | Difficulty | Time |
|---|-----|-----------------|------------|------|
| 01 | [Recover Deleted Branch](01-recover-deleted-branch.md) | reflog, fsck, branch recreation | Beginner | 20 min |
| 02 | [Resolve Merge Conflicts](02-resolve-merge-conflict.md) | merge, conflict resolution, rerere | Beginner | 25 min |
| 03 | [Interactive Rebase](03-interactive-rebase.md) | rebase -i, squash, fixup, reorder | Intermediate | 30 min |
| 04 | [Git Bisect](04-git-bisect.md) | bisect, regression hunting, automated bisect | Intermediate | 30 min |
| 05 | [Cherry-Pick Backport](05-cherry-pick-backport.md) | cherry-pick, conflict resolution, versioning | Intermediate | 25 min |
| 06 | [Repository Cleanup](06-repository-cleanup.md) | filter-repo, gc, pack optimization | Advanced | 45 min |
| 07 | [Git Worktree](07-git-worktree.md) | worktree, parallel branches, hotfix workflow | Intermediate | 25 min |
| 08 | [Git LFS](08-git-lfs.md) | lfs track, migrate, pointer files | Advanced | 45 min |

---

## Setup

```bash
# Create a labs directory
mkdir -p ~/git-labs
cd ~/git-labs

# Each lab creates its own isolated repository
# You can reset any lab by deleting its directory and restarting
```

---

## Lab Conventions

Each lab follows this structure:
1. **Objective** — What you will learn
2. **Setup** — Create the lab environment (all commands provided)
3. **Scenario** — The situation you are in
4. **Your Task** — What you need to do (try on your own first)
5. **Walkthrough** — Step-by-step solution with explanations
6. **Verification** — How to confirm you succeeded
7. **Cleanup** — Remove the lab environment

---

## Navigation

[⌂ Git Index](../README.md) | [Playbooks →](../playbooks/README.md)
