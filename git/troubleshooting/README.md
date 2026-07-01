# Git Troubleshooting — Diagnosing and Recovering from Common Failures

## Overview

This is a field guide for Git problems in production environments. Every scenario here has been encountered in real engineering teams. The goal is not to explain theory — it is to give you the exact commands to diagnose and recover.

---

## Diagnostic First Steps

Before anything else:

```bash
git status          # What state is the working tree in?
git log --oneline --graph --all | head -20  # What does the branch topology look like?
git reflog | head -20  # What have HEAD movements been recently?
```

These three commands answer 80% of questions.

---

## Scenario 1 — Detached HEAD State

### Symptoms

```bash
$ git status
HEAD detached at abc1234
```

### Cause

You checked out a commit, tag, or remote branch directly instead of a local branch. `HEAD` points to a commit, not a branch ref.

### Recovery

```bash
# Option 1: Go back to your branch
git checkout main

# Option 2: Keep the work you did in detached HEAD
git checkout -b recover/detached-work
# Now you are on a named branch — commit and push safely
```

### If you made commits in detached HEAD state

```bash
git log --oneline | head -5
# Note the SHA of the work you want to keep

git checkout main
git cherry-pick <sha-of-commit-from-detached-head>
```

---

## Scenario 2 — Accidentally Committed to the Wrong Branch

### Situation A: Committed to `main` instead of feature branch (before push)

```bash
git log --oneline | head -3
# abc1234 feat: my accidental commit on main
# def5678 previous commit

# Create the feature branch from current state
git branch feature/INFRA-correct-branch

# Remove the commit from main
git reset --hard HEAD~1

# Switch to the feature branch
git checkout feature/INFRA-correct-branch
```

### Situation B: Pushed the wrong commit to `main`

```bash
# Create a revert commit (safe for shared branches)
git revert abc1234
git push origin main
```

Never `git reset --hard` a shared branch after it has been pushed.

---

## Scenario 3 — Recovering Lost Commits

### "I ran git reset --hard and lost work"

```bash
git reflog
# HEAD@{0}: reset: moving to HEAD~2
# HEAD@{1}: commit: feat: vpc module complete
# HEAD@{2}: commit: feat: subnet configuration
# HEAD@{3}: checkout: moving from main to feature/vpc

# Restore the commit before the reset
git reset --hard HEAD@{1}
```

The reflog is a local record of every HEAD movement. It is retained for 90 days by default.

### "I deleted a branch without merging"

```bash
git reflog | grep "feature/lost-branch"
# HEAD@{12}: checkout: moving from feature/lost-branch to main

# The SHA just before checkout is the tip of the deleted branch
git checkout -b feature/lost-branch HEAD@{12}
```

---

## Scenario 4 — Merge Conflict Resolution

### Step by step

```bash
git merge feature/INFRA-1042-vpc-module
# CONFLICT (content): Merge conflict in modules/vpc/main.tf

# See all conflicted files
git diff --name-only --diff-filter=U

# Resolve each file (edit to correct state, remove conflict markers)
# Then:
git add modules/vpc/main.tf
git merge --continue
```

### Abandon the merge entirely

```bash
git merge --abort
# Restores to pre-merge state
```

### Use git rerere to auto-resolve repeated conflicts

If your workflow produces the same conflict repeatedly (long-lived branches, frequent rebases):

```bash
git config rerere.enabled true
# Git records conflict resolutions and replays them automatically
```

---

## Scenario 5 — Bisect — Finding Which Commit Introduced a Bug

`git bisect` performs a binary search through commit history to find the commit that introduced a regression.

```bash
git bisect start

git bisect bad          # Current state is broken
git bisect good v1.0.0  # v1.0.0 was working

# Git checks out a middle commit
# Test whether the bug exists, then mark it:
git bisect good   # or
git bisect bad

# Repeat until Git identifies the first bad commit:
# abc1234 is the first bad commit
# fix: revert abc1234 or investigate it

git bisect reset   # Returns to original HEAD
```

### Automate bisect with a script

```bash
git bisect run ./scripts/test-for-regression.sh
# Git runs the script at each step and uses exit code 0 (good) or non-zero (bad)
```

---

## Scenario 6 — Force Push Disaster Recovery

Someone force-pushed to `main` and overwrote commits.

```bash
# Check reflog on your local clone
git reflog show origin/main
# origin/main@{0}: 3f8a2b1 — the force-pushed state
# origin/main@{1}: abc1234 — the original state before force push

# Restore origin/main to the correct state
git push origin abc1234:main --force-with-lease
```

If no local clone has the original state, contact GitHub support — they retain reflog server-side for a limited time.

---

## Scenario 7 — Large File Accidentally Committed

```bash
git log --all --full-history -- path/to/large-file.bin | head -5
# Find the commit that added it

# Remove from history (requires force-push after — coordinate with team)
git filter-branch --tree-filter 'rm -f path/to/large-file.bin' HEAD
# Or use git-filter-repo (preferred):
pip install git-filter-repo
git filter-repo --path path/to/large-file.bin --invert-paths
```

---

## Scenario 8 — Diagnosing Slow Git Operations

```bash
# Check repository size
git count-objects -vH

# Find large objects
git rev-list --objects --all \
  | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
  | awk '/^blob/ {print substr($0,6)}' \
  | sort -k2 -rn \
  | head -10
```

---

## Scenario 9 — Authentication Failures

```bash
# Test SSH connectivity
ssh -T git@github.com
# Hi AkashKhuranaDev! You've successfully authenticated

# Test correct key is being used
ssh -vT git@github.com 2>&1 | grep "Offering public key"

# For multiple accounts (see git/fundamentals for SSH config)
ssh -T git@github-akashdev
```

---

## Scenario 10 — "Permission denied to AnotherUser"

This means the wrong credential is being sent. macOS Keychain is sending the wrong stored token.

```bash
# List stored credentials
git credential-osxkeychain get <<EOF
protocol=https
host=github.com
EOF

# Erase cached credential
git credential-osxkeychain erase <<EOF
protocol=https
host=github.com
EOF

# Next push will prompt for fresh credentials
```

Or use SSH aliases to avoid credential conflicts entirely. See `git/branching/README.md`.

---

## Quick Reference — Git Emergency Commands

| Situation | Command |
|---|---|
| Undo last commit (keep changes) | `git reset --soft HEAD~1` |
| Undo last commit (discard changes) | `git reset --hard HEAD~1` |
| Undo a pushed commit safely | `git revert HEAD` |
| See recent HEAD movements | `git reflog` |
| Recover deleted branch | `git checkout -b branch HEAD@{N}` |
| Abort in-progress merge | `git merge --abort` |
| Abort in-progress rebase | `git rebase --abort` |
| See what is staged | `git diff --staged` |
| Find which commit broke something | `git bisect start && git bisect bad && git bisect good <tag>` |
| Remove a file from last commit | `git reset HEAD~1 -- file && git commit --amend` |

---

## References

| Resource | URL |
|---|---|
| git reflog | https://git-scm.com/docs/git-reflog |
| git bisect | https://git-scm.com/docs/git-bisect |
| git filter-repo | https://github.com/newren/git-filter-repo |
| Undoing Things | https://git-scm.com/book/en/v2/Git-Basics-Undoing-Things |
| Debugging with Git | https://git-scm.com/book/en/v2/Git-Tools-Debugging-with-Git |
