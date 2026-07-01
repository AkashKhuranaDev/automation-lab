# Playbook 08: Repository Maintenance

**Risk Level:** Low  
**Time to Execute:** 30–60 minutes  
**Frequency:** Monthly for active repositories, quarterly for stable ones

---

## Purpose

Repositories accumulate noise over time: stale branches, unnecessary objects, large pack files, outdated configurations. This playbook covers the monthly maintenance run that keeps a repository healthy, fast, and auditable.

---

## Prerequisites

- [ ] Team notified of maintenance window (operations continue normally — this is non-destructive)
- [ ] Admin access to the repository
- [ ] 30 minutes of uninterrupted time
- [ ] Current working tree is clean: `git status`

---

## Phase 1: Local Repository Health

```bash
# 1. Check repository integrity
git fsck --quiet 2>&1 | grep -v "^Checking"
# Zero output is correct. Any "error" or "warning" lines need investigation.

# 2. Check repository size
git count-objects -vH

# 3. Run garbage collection (safe — only prunes unreachable objects > 2 weeks old)
git gc
git count-objects -vH               # Compare to before

# 4. Repack for performance
git repack -Ad --depth=50 --window=250
# -A: convert unreachable loose objects to pack (don't prune)
# -d: remove redundant packs
# --depth/--window: higher values = better compression, slower operation
```

---

## Phase 2: Branch Cleanup

### Identify stale branches

```bash
# Branches merged into main
git branch --merged main | grep -v '^\*\|main\|develop'

# Remote branches merged into main
git branch -r --merged origin/main | grep -v 'origin/main\|origin/develop\|origin/HEAD'

# All remote branches with their last commit date
git for-each-ref --sort='-creatordate' \
  --format='%(creatordate:relative) %(refname:short)' \
  refs/remotes/origin \
  | grep -v 'HEAD' \
  | head -40
```

### Delete merged local branches

```bash
# Review the list first
git branch --merged main | grep -v '^\*\|main\|develop'

# Delete them (one at a time initially, then automate)
git branch -d feature/old-branch-1 feature/old-branch-2

# OR delete all merged local branches at once
git branch --merged main | grep -v '^\*\|main\|develop' | xargs -r git branch -d
```

### Prune stale remote-tracking refs

```bash
# Remove local references to remote branches that no longer exist
git fetch --prune origin

# Verify: this should show nothing
git branch -r | grep deleted
```

### Archive (don't delete) important stale branches

Before deleting a branch that may still be referenced:

```bash
# Tag it with the date
git tag archive/$(basename feature/old-important-branch)-$(date +%Y%m%d) feature/old-important-branch
git push origin archive/$(basename feature/old-important-branch)-$(date +%Y%m%d)

# Now safe to delete the branch
git push origin --delete feature/old-important-branch
```

---

## Phase 3: Tag Hygiene

```bash
# List all tags
git tag --sort=-creatordate | head -30

# Identify lightweight tags (should be annotated for releases)
git for-each-ref --format='%(objecttype) %(refname:short)' refs/tags \
  | awk '$1 == "commit" { print "lightweight:", $2 }'
# Lightweight tags on release versions should be converted to annotated

# Remove test/temporary tags that were pushed accidentally
git tag | grep 'test\|temp\|wip\|draft' | while read tag; do
    echo "Candidate for deletion: $tag"
done
# Review each, then: git push origin --delete <tag-name>
```

---

## Phase 4: Security Audit

```bash
# Scan for secrets in recent commits (adjust the date range)
gitleaks detect --since="$(git log --format=%aI -1 origin/main 3 months ago)" 2>/dev/null \
  || echo "Run: pip install gitleaks"

# Check for large files recently added
git log --all --since="90 days ago" \
  --diff-filter=A -- \
  "*.pem" "*.key" "*.env" "*credentials*" "*secret*" \
  --name-only \
  | grep -v '^commit\|^Author\|^Date\|^$'

# Verify branch protection is still configured correctly
# GitHub → Settings → Branches → verify rules
```

---

## Phase 5: Configuration Review

```bash
# Check remote URLs are correct (especially after any organizational changes)
git remote -v

# Verify no stale or wrong remotes
git remote show origin

# Check for outdated git config settings
git config --list --local

# Verify signing is still configured (if used)
git config commit.gpgsign
git config user.signingkey
```

---

## Phase 6: Update and Document

```bash
# Fetch and prune
git fetch --all --prune --tags

# Update main
git checkout main
git pull --ff-only

# Record the maintenance run
# (In CHANGELOG or team wiki)
# Date: YYYY-MM-DD
# Repository size before: X MB
# Repository size after: Y MB
# Branches deleted: N
# Tags cleaned: N
# Issues found: none / describe
```

---

## Monthly Maintenance Output

After each run, note these metrics (track in a spreadsheet or wiki):

| Metric | Command | Expected Trend |
|--------|---------|----------------|
| Pack size | `git count-objects -vH` | Stable or slowly growing |
| Loose objects | `git count-objects -v` | Near zero after gc |
| Branch count | `git branch -r \| wc -l` | Decreasing or stable |
| Tag count | `git tag \| wc -l` | Growing at release cadence |

---

## Related

- [Performance Reference](../performance/README.md)
- [Playbook 09: Safe Branch Cleanup](09-safe-branch-cleanup.md)
- [Case Study: Large Repository Optimization](../case-studies/07-large-repository-optimization.md)

---

[← Playbook 07: Disaster Recovery](07-disaster-recovery.md) | [Playbook 09: Safe Branch Cleanup →](09-safe-branch-cleanup.md)
