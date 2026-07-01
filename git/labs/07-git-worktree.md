# Lab 07: Git Worktree

**Difficulty:** Intermediate  
**Time:** 25 minutes  
**Skills:** `git worktree`, parallel branch work, hotfix while mid-feature

---

## Objective

Use `git worktree` to work on a production hotfix while keeping your in-progress feature branch untouched. Understand when worktree is more appropriate than stashing.

---

## Setup

```bash
cd ~/git-labs
mkdir lab-07-git-worktree && cd lab-07-git-worktree
git init

git commit --allow-empty -m "initial commit"
git commit --allow-empty -m "feat: add user service"
git commit --allow-empty -m "feat: add payment service"
git tag v2.5.0 -m "Release v2.5.0"

git commit --allow-empty -m "feat: add recommendation engine"
git commit --allow-empty -m "feat: train initial model"

echo ""
echo "Current state:"
git log --oneline
echo ""
echo "Current branch:"
git branch
```

---

## Scenario

You are in the middle of a large refactoring on `main`. A production bug is reported: `v2.5.0` has a security issue in the payment service that needs an immediate hotfix. You cannot commit your in-progress work — it's half-done and the tests don't pass yet.

**Options:**
1. `git stash` — Risk: stash is fragile, and you'll need to re-set up your editor state
2. `git worktree` — Create a separate working directory for the same repository, on a different branch, without disturbing your current work

---

## Part 1: Worktree for Hotfix

```bash
# List current worktrees (just one — the main working directory)
git worktree list

# Create a new worktree for the hotfix
# This creates a NEW directory alongside your main working directory
# WITHOUT touching your current branch
git worktree add ../lab-07-hotfix hotfix/payment-security -b hotfix/payment-security

# Note: -b creates the new branch; without it, the branch must already exist
```

```bash
# Your current work is still here, untouched
ls                                    # Your current files
git branch                            # Still on main

# Switch to the hotfix worktree
cd ../lab-07-hotfix
ls                                    # Same repository, different branch
git branch                            # On hotfix/payment-security
git log --oneline -5                  # Starts from the commit you were on
```

```bash
# Apply the hotfix
cat > payment-fix.txt << 'EOF'
Security fix: added input validation to payment endpoint
Prevents SQL injection via unescaped user input in payment reference field
EOF
git add payment-fix.txt
git commit -m "fix(security): add input validation to payment endpoint

Affected version: v2.5.0
CVE: CVE-2024-XXXX
Severity: High"

# Tag the hotfix
git tag v2.5.1 -m "Hotfix v2.5.1: payment security fix"

echo "Hotfix committed and tagged"
```

---

## Part 2: Return to Feature Work

```bash
# Return to your main worktree
cd ../lab-07-git-worktree

# Your feature work is exactly where you left it
git log --oneline -3
git status                            # Shows your in-progress changes (none in this lab)

# The hotfix is on a separate branch — you can merge it to main when ready
git merge hotfix/payment-security
```

---

## Part 3: Multiple Worktrees

```bash
# You can have multiple worktrees simultaneously
git worktree add ../lab-07-docs docs-review -b docs-review

# Now you have three working directories:
# lab-07-git-worktree/  → main branch
# lab-07-hotfix/        → hotfix/payment-security
# lab-07-docs/          → docs-review

git worktree list
# Output:
# /path/to/lab-07-git-worktree  <sha> [main]
# /path/to/lab-07-hotfix        <sha> [hotfix/payment-security]
# /path/to/lab-07-docs          <sha> [docs-review]
```

---

## Part 4: Comparing Worktree vs Stash

```bash
# Practice with stash for comparison
git stash push -m "in-progress feature work"

# Stash limitations:
git stash list                        # Hidden — you must remember it's there
# No separate directory — you must restore before switching contexts
# Stash applies to working tree, not a branch

# Restore from stash
git stash pop

# When to use stash: quick context switch (5-30 minutes), no parallel editing needed
# When to use worktree: parallel, ongoing work; two branches need simultaneous access
```

---

## Part 5: Worktree for Reviewing PRs

```bash
# Common real-world pattern: reviewing a PR while keeping your feature branch active
git worktree add ../lab-07-pr-review pr/review-branch

cd ../lab-07-pr-review

# Make review comments in a file (simulating code review)
cat > REVIEW_NOTES.txt << 'EOF'
Reviewed PR #42
- Logic looks correct
- Suggest adding error handling on line 47
- Missing test for edge case where input is null
EOF

# Your main feature branch in lab-07-git-worktree is unaffected
cd ../lab-07-git-worktree
git log --oneline -3                  # Still on your feature work
```

---

## Cleanup: Removing Worktrees

```bash
# IMPORTANT: Remove worktrees before deleting their directories
# Simply deleting the directory leaves a stale worktree reference

# Clean way to remove:
git worktree remove ../lab-07-hotfix
git worktree remove ../lab-07-docs
git worktree remove ../lab-07-pr-review

# OR if the directory was deleted already:
git worktree prune                    # Removes references to missing worktrees

# Final list should show only the main worktree
git worktree list
```

---

## Key Concepts

1. **Worktrees share the same `.git` directory.** All worktrees see the same branches, commits, tags, and stashes. A commit in one worktree is immediately visible in all others.
2. **One branch per worktree.** You cannot check out the same branch in two worktrees simultaneously — Git prevents this.
3. **Worktrees are not clones.** There is no extra network overhead or disk usage for objects. Only the working tree (files) is duplicated.
4. **Path matters.** Worktree paths are relative to the filesystem, not the repository. Use sibling directories (same parent) to keep them organized.
5. **Prune after path changes.** If you move or delete a worktree directory without using `git worktree remove`, use `git worktree prune` to clean up the stale references.

---

## Cleanup

```bash
cd ~/git-labs
rm -rf lab-07-git-worktree lab-07-hotfix lab-07-docs lab-07-pr-review
```

---

[← Lab 06: Repository Cleanup](06-repository-cleanup.md) | [Lab 08: Git LFS →](08-git-lfs.md)
