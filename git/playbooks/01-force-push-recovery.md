# Playbook 01: Force Push Recovery

**Risk Level:** P1 — Critical  
**Time to Execute:** 15–45 minutes  
**Affected Systems:** Shared branches, open PRs, team history

---

## Situation

Someone ran `git push --force` (without `--force-with-lease`) on a shared branch — typically `main` or `develop`. This overwrites the remote history with the pusher's local state. Commits that existed on the remote but not in the pusher's local branch are now unreachable via normal `git log`.

**Symptoms:**
- Team members get `rejected: non-fast-forward` on next pull
- `git log origin/main` shows commits that do not match expected history
- Open PRs show incorrect diffs or impossible conflicts
- CI triggered on the wrong commits

---

## Prerequisites

- [ ] Write access to the repository
- [ ] GitHub repository admin access (to temporarily disable branch protection if needed)
- [ ] Another team member's local clone that has the pre-force-push state
- [ ] The SHA of the last known-good commit (get from GitHub PR history, CI logs, or deployment tags)

---

## Step 1: Stop the Bleeding

Immediately notify the team. No one should push to the affected branch until this is resolved.

```bash
# Post in the team channel immediately:
# "DO NOT PUSH to [branch] — history incident in progress"
```

If you have admin access, enable branch protection to block further pushes:

1. GitHub → Settings → Branches → Edit rule for `main`
2. Check "Require pull request before merging"
3. This blocks direct push while you recover

---

## Step 2: Locate the Lost Commits

Find a machine that has the pre-force-push state in its reflog or local branch:

```bash
# On any team member's machine that has the branch checked out
git reflog show origin/main

# Output example:
# a3f2c11 (HEAD -> main, origin/main) HEAD@{0}: pull: Fast-forward
# 9b1e4a7 HEAD@{1}: commit: feat: add vpc peering config
# 8d0c2f3 HEAD@{2}: commit: fix: correct subnet cidr
# ...
```

```bash
# Alternatively, use the GitHub API to find the last push SHA
# GitHub → Repository → Commits → check the commit before the incident
```

If no local clone has the data, check:
- GitHub PR "Files Changed" tab (shows the diff even after force push)
- CI logs (the SHA is usually in the build output)
- GitHub Audit Log (Settings → Audit Log → search for `git.push`)

---

## Step 3: Restore the Lost History

### Option A: Restore from team member's local clone

```bash
# On the machine that has the old history
git log --oneline origin/main          # Verify old state is still here
git rev-parse HEAD                     # Note this SHA

# Push the old state back, temporarily bypassing branch protection if needed
git push origin main --force-with-lease

# --force-with-lease is still safer than --force even here
# It will fail if the remote has changed again since you fetched
```

### Option B: Cherry-pick lost commits onto current state

If the current remote state has commits you want to keep:

```bash
# Get both states
git fetch origin

# Find the common ancestor between old and new
git merge-base <old-sha> <new-sha>

# Cherry-pick the lost commits from the old branch onto the new one
git checkout -b recovery/force-push-restore
git cherry-pick <first-lost-sha>^..<last-lost-sha>

# Verify
git log --oneline

# Merge via PR — do NOT force push this back
```

---

## Step 4: Repair Team Member Local Clones

After the correct history is restored to the remote, team members must update their local clones:

```bash
# On each affected team member's machine
git fetch origin

# If their local branch has diverged from the restored remote:
git checkout main
git reset --hard origin/main           # Destructive: discards local-only commits

# OR, safer: create a new branch from origin/main and manually apply local work
git checkout -b temp/my-wip origin/main
git cherry-pick <my-local-commits>
```

---

## Step 5: Fix Affected Pull Requests

For each open PR that targets the affected branch:

1. Check if the PR base commits are still in history: `git log --oneline origin/main | grep <pr-base-sha>`
2. If yes: the PR is intact, just re-run CI
3. If no: re-base the PR branch onto the restored history

```bash
# For each affected PR branch
git checkout pr-branch-name
git rebase origin/main

# Force-push the PR branch (this is safe — single-author branch)
git push --force-with-lease origin pr-branch-name
```

---

## Step 6: Verification

```bash
# Confirm the expected commits are in history
git log --oneline origin/main | head -20

# Confirm no commits are missing
git log --oneline <expected-sha>..origin/main  # Should be empty or show only new commits

# Confirm CI is green on the restored commit
# GitHub Actions → check the latest run on main

# Confirm all open PRs show correct diffs
```

---

## Rollback Strategy

If Step 3 makes things worse:

```bash
# Do NOT push again. Stop.
# Take a snapshot of every local SHA you have:
git log --oneline -20

# Wait for a team member with the correct state to arrive
# Coordinate before pushing anything
```

---

## Prevention

1. **Branch protection:** Enable `main` protection with no force push allowed
2. **`--force-with-lease`:** Add a git alias: `alias gpf='git push --force-with-lease'`
3. **Policy:** Communicate in writing that `--force` on shared branches is never allowed
4. **`receive.denyNonFastForwards`:** On self-hosted repositories, set this server-side config

---

## Related

- [Case Study: Force Push Overwrites Main](../case-studies/01-force-push-recovery.md)
- [Production Incident P001](../production-incidents/P001-force-push-overwrites-main.md)
- [Recovery Reference](../recovery/README.md)

---

[← Playbooks Index](README.md) | [Playbook 02: Production Hotfix →](02-production-hotfix.md)
