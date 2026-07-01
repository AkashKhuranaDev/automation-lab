# Playbook 09: Safe Branch Cleanup

**Risk Level:** Low  
**Time to Execute:** 15–30 minutes  
**Frequency:** Weekly for active repositories, monthly for stable ones

---

## Situation

The repository has accumulated stale, merged, or abandoned branches. This creates operational noise:
- `git branch -r` returns 50+ branches, making navigation difficult
- `git fetch` pulls down dozens of refs that are no longer active
- CI runs on lingering branches from months ago

---

## Prerequisites

- [ ] Team notified: "Branch cleanup happening today — if you have a branch you want to keep, let me know"
- [ ] All active PRs identified (open PRs are never touched)
- [ ] GitHub admin access (for remote deletion)

---

## Step 1: Identify Candidates for Deletion

```bash
# === SAFE TO DELETE: branches merged into main ===
git fetch --prune origin
git branch -r --merged origin/main | grep -v 'origin/main\|origin/HEAD\|origin/develop\|origin/release/'

# === POTENTIALLY SAFE: branches with no activity in 30+ days ===
git for-each-ref \
  --sort=creatordate \
  --format='%(creatordate:iso) %(refname:short) %(authorname)' \
  refs/remotes/origin \
  | grep -v 'origin/main\|origin/HEAD\|origin/develop' \
  | awk -v cutoff="$(date -v-30d '+%Y-%m-%d' 2>/dev/null || date -d '30 days ago' '+%Y-%m-%d')" '$1 < cutoff'

# === DO NOT DELETE: these are always protected ===
# origin/main, origin/master, origin/develop, origin/release/*, origin/hotfix/*
```

---

## Step 2: Verify Each Candidate

For each branch that appears in the deletion list:

```bash
# Does it have an open PR?
gh pr list --head <branch-name> --state open
# If this returns anything: skip this branch

# Has it been merged?
git log origin/main..<branch-name> | head -5
# If empty: no unique commits — safe to delete

# What was the last commit?
git log --oneline -1 origin/<branch-name>
git show --stat origin/<branch-name>     # Brief view of what the branch contains
```

---

## Step 3: Delete Merged Branches

```bash
# Delete one at a time during first cleanup (safety)
git push origin --delete feature/old-branch-name

# After you are confident in the list, you can batch:
BRANCHES_TO_DELETE=(
  "feature/add-vpc-peering"
  "fix/rds-connection-pool"
  "chore/update-tf-providers"
)

for branch in "${BRANCHES_TO_DELETE[@]}"; do
  git push origin --delete "$branch" && echo "Deleted: $branch" || echo "FAILED: $branch"
done
```

---

## Step 4: Delete Merged Local Branches

```bash
# Clean up local branches that track deleted remotes
git fetch --prune origin              # Remove stale remote-tracking refs

# Delete local branches where the remote is gone
git branch -vv | grep '\[origin/.*: gone\]' | awk '{print $1}'
# Review the list

git branch -vv | grep '\[origin/.*: gone\]' | awk '{print $1}' | xargs -r git branch -d
```

---

## Step 5: Handle Stale Unmerged Branches

For branches with unmerged commits that are 30+ days old:

1. **Identify the owner:** `git log --format="%an <%ae>" -1 origin/<branch-name>`
2. **Contact the owner:** "Hey, your branch `feature/X` has been inactive for 30 days. Should I archive or delete it?"
3. **If no response in 1 week:**

```bash
# Archive via tag
BRANCH="feature/abandoned-work"
AUTHOR=$(git log --format="%an" -1 origin/$BRANCH)
git tag archive/$BRANCH-$(date +%Y%m%d) origin/$BRANCH
git push origin archive/$BRANCH-$(date +%Y%m%d)

# Delete the branch
git push origin --delete $BRANCH

echo "Archived $BRANCH by $AUTHOR as tag archive/$BRANCH-$(date +%Y%m%d)"
```

---

## Step 6: Verify

```bash
# Confirm only expected branches remain
git branch -r | grep -v 'HEAD\|main\|develop\|release/' | sort

# Count should match your expectation
git branch -r | wc -l

# Fetch to confirm remote is clean
git fetch --prune origin
git remote show origin | grep "stale"   # Should show nothing
```

---

## Automation Option

For repositories with many contributors, add this to a weekly scheduled GitHub Action:

```yaml
# .github/workflows/branch-cleanup.yml
name: Branch Cleanup
on:
  schedule:
    - cron: '0 9 * * 1'   # Every Monday at 09:00 UTC
  workflow_dispatch:

jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: List merged branches (dry run)
        run: |
          echo "=== Branches merged into main ==="
          git branch -r --merged origin/main \
            | grep -v 'origin/main\|origin/HEAD\|origin/develop\|origin/release/' \
            | sed 's|origin/||'
          echo "Review this list before enabling deletion"
```

Add actual deletion after validating the dry-run output for several weeks.

---

## Related

- [Playbook 08: Repository Maintenance](08-repository-maintenance.md)
- [Branching Reference](../branching/README.md)

---

[← Playbook 08: Repository Maintenance](08-repository-maintenance.md) | [⌂ Playbooks Index](README.md)
