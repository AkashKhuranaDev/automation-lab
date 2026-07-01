# Playbook 05: Recover a Deleted Branch

**Risk Level:** Medium  
**Time to Execute:** 5–15 minutes  
**Affected Systems:** Developer's local workflow, open PRs on the deleted branch

---

## Situation

A branch was deleted — either accidentally by a developer or automatically by the "delete branch on merge" GitHub setting — but work had not been fully merged or the branch needs to be referenced again.

The key insight: **Git does not delete the commit objects when a branch is deleted.** The branch label (ref) is removed, but commits remain in the object database until garbage collection runs. On GitHub, deleted branch refs are preserved for 30 days and can be restored from the UI.

---

## Prerequisites

- [ ] Know approximately when the deletion happened (within the last 30 days for GitHub restore)
- [ ] Have the branch name, or the last known commit SHA

---

## Method 1: Restore via GitHub UI (Fastest)

If the branch was deleted on GitHub within the last 30 days:

1. Navigate to the repository on GitHub
2. Click **Pull Requests** tab
3. Click **Closed** filter
4. Find the PR associated with the branch
5. At the bottom of the PR page: **"Restore branch"** button

This is the fastest path and requires no CLI.

---

## Method 2: Recover from Local Reflog

If you had the branch checked out locally before it was deleted:

```bash
# Your reflog tracks all ref movements, including branch deletions
git reflog show --all | grep <branch-name>

# Example output:
# abc1234 refs/remotes/origin/feature/vpc-peering@{3}: fetch prune: .../feature/vpc-peering

# Recreate the branch from the SHA
git checkout -b feature/vpc-peering abc1234
```

---

## Method 3: Search All Local Commit Objects

If you don't have the branch name but you remember something about the work:

```bash
# List all dangling commits (commits with no reachable ref)
git fsck --lost-found 2>/dev/null | grep "dangling commit"

# Example output:
# dangling commit 9b1e4a7d...

# View each one
git log --oneline -10 9b1e4a7d

# If it's what you want, create a branch
git checkout -b recovered/my-branch 9b1e4a7d
```

---

## Method 4: Find the SHA from GitHub Events

If no local clone had the branch:

```bash
# GitHub API: list recent pushes to the repository
curl -s "https://api.github.com/repos/ORG/REPO/events" \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  | python3 -c "
import sys, json
events = json.load(sys.stdin)
for e in events:
    if e.get('type') == 'PushEvent':
        print(e['created_at'], e['payload'].get('ref'), e['payload'].get('head')[:8] if e['payload'].get('head') else '')
" | grep <branch-name>
```

Take the commit SHA from the output and use `git checkout -b <branch-name> <sha>`.

---

## Step: Push the Recovered Branch

After recovering locally:

```bash
git push origin feature/vpc-peering

# Confirm
git ls-remote origin | grep vpc-peering
```

---

## Verification

```bash
# Confirm the branch is fully restored
git log --oneline feature/vpc-peering | head -10

# Compare against the PR diff to confirm all expected commits are present
git diff main...feature/vpc-peering --stat
```

---

## Prevention

```bash
# 1. Enable "Allow users to restore deleted branches" in GitHub Settings
#    Settings → General → "Allow users to restore deleted branches"

# 2. For critical long-lived branches, create regular remote tags as backups
git tag backup/vpc-peering-$(date +%Y%m%d) feature/vpc-peering
git push origin backup/vpc-peering-$(date +%Y%m%d)

# 3. Increase the reflog expiry window
git config --global gc.reflogExpire "120 days"
git config --global gc.reflogExpireUnreachable "60 days"
```

---

## Related

- [Case Study: Deleted Branch Recovery](../case-studies/02-deleted-branch-recovery.md)
- [Recovery Reference](../recovery/README.md)
- [Lab 01: Recover Deleted Branch](../labs/01-recover-deleted-branch.md)

---

[← Playbook 04: Clean Large Binary Files](04-clean-large-binary-files.md) | [Playbook 06: Recover Lost Commits →](06-recover-lost-commits.md)
