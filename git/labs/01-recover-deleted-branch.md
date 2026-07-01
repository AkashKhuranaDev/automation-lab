# Lab 01: Recover a Deleted Branch

**Difficulty:** Beginner  
**Time:** 20 minutes  
**Skills:** `git reflog`, `git fsck`, `git checkout -b`

---

## Objective

Recover a deleted branch using three different methods. Understand why recovery is possible and when each method applies.

---

## Setup

```bash
cd ~/git-labs
mkdir lab-01-deleted-branch && cd lab-01-deleted-branch
git init

# Create some history
git commit --allow-empty -m "initial commit"
git commit --allow-empty -m "feat: add user authentication"
git commit --allow-empty -m "feat: add dashboard"

# Create a feature branch with unique commits
git checkout -b feature/payment-gateway
git commit --allow-empty -m "feat: add payment provider config"
git commit --allow-empty -m "feat: implement checkout flow"
git commit --allow-empty -m "fix: handle declined card edge case"

# Note the tip SHA before deleting
BRANCH_TIP=$(git rev-parse HEAD)
echo "Branch tip SHA: $BRANCH_TIP"

# Now simulate the accident: switch to main and delete the branch
git checkout main
git branch -D feature/payment-gateway

echo ""
echo "The branch is now deleted. 'git branch' output:"
git branch
```

---

## Scenario

You are working on the `feature/payment-gateway` branch. A teammate ran `git branch -D feature/payment-gateway` on your machine while you were at lunch. The branch is gone. The work was not committed to any other branch.

---

## Your Task

Try to recover the branch before looking at the walkthrough. You have:
- The knowledge that reflog tracks all position changes
- `git fsck` which can find unreachable commits
- The approximate time the branch was last active

**Hint:** `git reflog` is your first tool.

---

## Walkthrough

### Method 1: Reflog Recovery (most common)

```bash
# The reflog shows every HEAD movement and branch operation
git reflog

# Example output:
# c4a2b18 (HEAD -> main) HEAD@{0}: checkout: moving from feature/payment-gateway to main
# c4a2b18 HEAD@{1}: commit: fix: handle declined card edge case
# 9e3f7a2 HEAD@{2}: commit: feat: implement checkout flow
# 4b8c1d5 HEAD@{3}: commit: feat: add payment provider config
# ...

# The commit at HEAD@{1} is the tip of the deleted branch
# Its SHA is c4a2b18 in this example (yours will differ)

# Recreate the branch from that SHA
git checkout -b feature/payment-gateway HEAD@{1}
# OR using the SHA directly:
# git checkout -b feature/payment-gateway c4a2b18

git log --oneline
# Should show:
# c4a2b18 fix: handle declined card edge case
# 9e3f7a2 feat: implement checkout flow
# 4b8c1d5 feat: add payment provider config
# ...
```

### Method 2: Using the SHA you noted

```bash
# If you noted the SHA before the deletion (good practice)
git checkout -b feature/payment-gateway $BRANCH_TIP

git log --oneline -5
```

### Method 3: fsck for dangling commits (when you don't know the SHA)

```bash
# Reset the lab first to practice this method
git checkout main
git branch -D feature/payment-gateway 2>/dev/null || true

# fsck finds objects that have no reachable path from any ref
git fsck --lost-found 2>/dev/null | grep "dangling commit"

# Example output:
# dangling commit c4a2b18d...
# dangling commit 9e3f7a26...
# dangling commit 4b8c1d5a...

# Check each dangling commit to find your work
git log --oneline c4a2b18d
# fix: handle declined card edge case

# Recreate the branch from the most recent one
git checkout -b feature/payment-gateway c4a2b18d
```

---

## Verification

```bash
# Confirm the branch exists
git branch

# Confirm all 3 payment commits are present
git log --oneline | grep -E "payment|checkout|card"

# Output should show:
# fix: handle declined card edge case
# feat: implement checkout flow
# feat: add payment provider config
```

---

## Key Concepts

1. **Git never immediately destroys commit objects.** When you delete a branch, the commits become "dangling" — unreachable from any ref, but still in the object database.
2. **Reflog is local.** The reflog only exists on your machine. It does not sync to the remote. If someone else deletes the branch on GitHub, you need their local reflog or the GitHub UI restore feature.
3. **Garbage collection is not immediate.** By default, Git prunes unreachable objects only after 30 days (`gc.reflogExpireUnreachable`) or 90 days (`gc.reflogExpire`). You have time.
4. **`-D` vs `-d`:** `git branch -d` refuses to delete unmerged branches. `git branch -D` forces deletion regardless. The recovery procedure is the same for both.

---

## Cleanup

```bash
cd ~/git-labs
rm -rf lab-01-deleted-branch
```

---

[← Labs Index](README.md) | [Lab 02: Resolve Merge Conflicts →](02-resolve-merge-conflict.md)
