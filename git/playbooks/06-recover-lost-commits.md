# Playbook 06: Recover Lost Commits

**Risk Level:** Medium  
**Time to Execute:** 5–20 minutes  
**Affected Systems:** Developer's local working state

---

## Situation

Commits have "disappeared" — typically after:
- `git reset --hard HEAD~N` (went too far)
- `git rebase` that went wrong
- `git checkout -f` overwriting uncommitted changes
- A misunderstood `git clean -fd`

The reflog is your primary recovery tool. Unless you explicitly ran `git gc` or the reflog has expired (default: 90 days for reachable, 30 days for unreachable commits), the commits are still in the object database.

**What IS recoverable:** committed work (even after `git reset --hard`)  
**What is NOT recoverable:** changes that were never committed (`git checkout -f`, `git clean -fd`)

---

## Prerequisites

- [ ] Git reflog has not been manually cleared (`git reflog expire --expire=now --all` would destroy this)
- [ ] You are working on the same machine where the commits were made

---

## Step 1: Find the Lost Commits in Reflog

```bash
# The reflog records every position HEAD has been at
git reflog

# Example output:
# a3f2c11 (HEAD -> main) HEAD@{0}: reset: moving to HEAD~3
# 9b1e4a7 HEAD@{1}: commit: fix: add retry logic to RDS connection pool
# 8d0c2f3 HEAD@{2}: commit: feat: add circuit breaker to API gateway
# 3c1e5a9 HEAD@{3}: commit: chore: update terraform provider version
# ...

# HEAD@{1} through HEAD@{3} are the commits that were "lost" by the reset
```

```bash
# If you want to filter by date
git reflog --since="2 hours ago"

# If you want to see the full commit details for a reflog entry
git show HEAD@{2}
```

---

## Step 2: Identify the Exact Target SHA

```bash
# Look at the commits around the point you want to restore to
git log --oneline HEAD@{1}~3..HEAD@{1}

# The SHA you want is the one where the branch was before the bad operation
# In the example above: 9b1e4a7 (the last commit before the reset)
```

---

## Step 3: Recover

### Option A: Move the branch back to the pre-reset state

This is the simplest recovery if you want to restore exactly the state before the bad operation:

```bash
# Reset the branch to the SHA where it was before
git reset --hard HEAD@{1}
# OR using the explicit SHA:
git reset --hard 9b1e4a7

# Verify
git log --oneline -5
```

### Option B: Create a recovery branch (non-destructive)

If you want to examine the lost commits before committing to restoring them:

```bash
git checkout -b recovery/lost-commits 9b1e4a7

# Review the commits
git log --oneline

# If they look right, merge back or cherry-pick:
git checkout main
git cherry-pick <sha1> <sha2> <sha3>

# Clean up
git branch -d recovery/lost-commits
```

### Option C: Cherry-pick individual commits

If you only want some of the lost commits:

```bash
# Pick specific commits from the reflog
git cherry-pick HEAD@{2}              # One commit
git cherry-pick HEAD@{3}..HEAD@{1}   # A range
```

---

## Step 4: Recover Dangling Commits (Advanced)

If the reflog doesn't show the lost commits (e.g., after a failed rebase):

```bash
# Find all commits not reachable from any branch or tag
git fsck --lost-found 2>/dev/null | grep "dangling commit" | awk '{print $3}' | while read sha; do
    echo "---"
    git log --oneline -1 "$sha"
done
```

The output file `lost-found/commit/<sha>` contains the tree at that state. Pick the SHA you need and check it out.

---

## Step 5: Recover Lost Stash

If you accidentally `git stash drop`ped or `git stash clear`ed:

```bash
# Stash entries are also in the object database
git fsck --lost-found 2>/dev/null | grep "dangling commit" | awk '{print $3}' | while read sha; do
    # Stash commits have a specific parent structure
    PARENT_COUNT=$(git cat-file -p "$sha" | grep -c "^parent")
    if [[ $PARENT_COUNT -eq 3 ]]; then
        echo "Possible stash: $sha"
        git log --oneline -1 "$sha"
    fi
done

# Restore the stash
git stash apply <sha>
```

---

## Verification

```bash
# After recovery, confirm the commits are correct
git log --oneline -10

# Run tests to confirm the recovered state is functional
make test                            # or npm test, pytest, etc.

# If the branch was pushed before the loss:
git diff origin/main...HEAD          # Should show your expected changes
```

---

## Prevention

```bash
# 1. Never use git reset --hard on shared branches
# 2. Create safety branches before destructive operations:
git branch backup/$(git branch --show-current)-$(date +%s)

# 3. Extend the reflog window
git config --global gc.reflogExpire "180 days"
git config --global gc.reflogExpireUnreachable "90 days"

# 4. Alias dangerous commands to require confirmation
# Add to ~/.gitconfig:
# [alias]
#   reset-hard = "!f() { echo 'About to git reset --hard -- are you sure? (y/N)'; read r; [ \"$r\" = 'y' ] && git reset --hard \"$@\"; }; f"
```

---

## Related

- [Recovery Reference](../recovery/README.md)
- [Lab 01: Recover Deleted Branch](../labs/01-recover-deleted-branch.md)
- [Playbook 05: Recover Deleted Branch](05-recover-deleted-branch.md)

---

[← Playbook 05: Recover Deleted Branch](05-recover-deleted-branch.md) | [Playbook 07: Disaster Recovery →](07-disaster-recovery.md)
