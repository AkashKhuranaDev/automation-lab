# P001 — Force Push Overwrites Protected Branch

> **Severity:** P1 | **Expected Recovery Time:** 10–20 minutes
>
> **Related:** [`CS-01`](../case-studies/01-force-push-recovery.md) | [`recovery/`](../recovery/) | [`enterprise-workflows/`](../enterprise-workflows/)

---

## Symptoms

- Commit history on `main` (or a release branch) is shorter than expected
- Commits that were deployed to staging/production are no longer visible in `git log`
- GitHub shows a sudden reduction in commit count on a protected branch
- CI/CD pipeline references a commit SHA that no longer exists in the branch history
- GitHub Audit Log shows a `protected_branch.policy_override` or force push event

---

## Immediate Actions (Do These First)

```bash
# 1. Lock the repository — prevent any further writes
# GitHub → Settings → Branches → Branch protection rules → main
# → Add "Restrict who can push" → Remove all access
# OR: Enable "Lock branch" (read-only mode)

# 2. Do NOT run git gc, git prune, or git maintenance
# These will permanently delete the dangling commits you need to recover
```

---

## Diagnosis

```bash
# Fetch origin to get the current state
git fetch origin

# Check the reflog for origin/main — shows recent history including force pushes
git reflog show origin/main | head -20
# origin/main@{0}: push --force: 3f8a2b1 chore: some commit (post-force state)
# origin/main@{1}: commit: 7a9c4e2 fix(something): critical fix ← last good commit

# Identify the SHA of the last known-good state
# This is typically origin/main@{1} or whichever entry precedes the force push

# Verify the missing commits are reachable
git show <SHA-of-missing-commit>
# If this succeeds, the commit is still in the repository (just not on the branch)
```

---

## Recovery

### Path A: Recovery from Local Clone (Fastest)

The recovering engineer must have a local clone that was fetched AFTER the last good commits but BEFORE the force push. Check fetch timestamps first.

```bash
# Find a local clone with the missing commits
git log --oneline | head -10
# If you see the expected commits, this clone has what you need

# Push the correct state back to origin
# origin/main@{N} points to the state before the force push
git push origin origin/main@{1}:refs/heads/main --force-with-lease
# Verify
git fetch origin && git log origin/main --oneline | head -10
```

### Path B: Recovery from Reflog (If Path A Unavailable)

```bash
# The commits are dangling objects — still in the repository, just unreachable
# Find the commit SHA from reflog
git reflog show origin/main | grep "last-known-good-description"
# or review timestamps to find the commit before the force push

# Recover using the SHA directly
GOOD_SHA="<sha-from-reflog>"
git push origin $GOOD_SHA:refs/heads/main --force-with-lease
```

### Path C: Recovery from GitHub API

GitHub retains commit objects for 90 days after they become unreachable.

```bash
# Get the SHA of the PR that contained the overwritten commits
curl -H "Authorization: token $GITHUB_TOKEN" \
  "https://api.github.com/repos/$ORG/$REPO/pulls?state=closed" | \
  jq '.[] | {number: .number, sha: .merge_commit_sha, title: .title}' | \
  head -20

# Find the correct merge commit SHA
MERGE_SHA="<merge-sha-from-api>"
git fetch origin $MERGE_SHA
git push origin $MERGE_SHA:refs/heads/main --force-with-lease
```

---

## Verification

```bash
# Confirm the branch is back to the correct state
git fetch origin
git log origin/main --oneline -10

# Confirm the SHA that was deployed still exists in history
git show <deployed-sha>

# Confirm the CI/CD pipeline can reference the expected commits
# Check your CI system's last successful run SHA

# Test that force push is now blocked
git push --force origin main
# remote: error: GH006: Protected branch update failed...
# remote: error: Force pushes are not allowed.
# (This should fail — if it succeeds, branch protection is not configured)
```

---

## Prevention

Enable these settings on all protected branches before closing the incident:

```
GitHub → Settings → Branches → Branch protection rules → main:
✅ Do not allow force pushes
✅ Do not allow deletions
✅ Restrict who can push to matching branches
✅ Do not allow bypassing the above settings
```

Set up GitHub Audit Log alerting:

```bash
# GitHub supports Audit Log streaming to Splunk/Datadog/S3
# Alert on: protected_branch.policy_override, git.push (with force: true)
# GitHub → Settings → Audit log → Log streaming
```
