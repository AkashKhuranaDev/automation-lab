# P004 — Merge Conflict Blocks Scheduled Release

> **Severity:** P2 | **Expected Recovery Time:** 20–60 minutes (depends on conflict complexity)
>
> **Related:** [`merging/`](../merging/) | [`enterprise-workflows/`](../enterprise-workflows/) | [`rebasing/`](../rebasing/)

---

## Symptoms

- Merge of a release branch to `main` fails with merge conflicts
- Automated merge job in CI fails with exit code 1 and conflict markers in output
- `git merge` exits non-zero — branch cannot be fast-forwarded
- `git status` shows files with `UU` prefix (both modified)
- Release is blocked — deployment cannot proceed until conflicts are resolved

---

## Diagnosis

```bash
# Identify conflicting files
git status --short
# UU modules/networking/main.tf
# UU modules/iam/main.tf
# M  modules/compute/main.tf   (auto-resolved, verify the result)

# See the conflict markers
grep -rn "<<<<<<" .
# modules/networking/main.tf:45:<<<<<<< HEAD
# modules/iam/main.tf:89:<<<<<<< HEAD

# Understand the full diff context for each conflict
git diff HEAD...origin/release-v2.4
# or for a specific file:
git diff HEAD...origin/release-v2.4 -- modules/networking/main.tf

# Identify which commits introduced each side
git log --oneline HEAD...origin/release-v2.4 --left-right
# < abc1234 feat(networking): add VPC peering      ← on current HEAD
# > def5678 fix(networking): update subnet ranges  ← on release branch
```

---

## Resolution: Manual Conflict Resolution

```bash
# Step 1: Start the merge (if not already in conflict state)
git merge origin/release-v2.4
# CONFLICT (content): Merge conflict in modules/networking/main.tf

# Step 2: For each conflicting file, understand the conflict
# Open the file — conflict markers look like:
# <<<<<<< HEAD (current branch)
# ... current version of the code ...
# =======
# ... incoming version of the code ...
# >>>>>>> origin/release-v2.4 (incoming branch)

# Step 3: Resolve using the correct version
# Option A: Use HEAD version (current branch wins)
git checkout --ours modules/networking/main.tf

# Option B: Use incoming version (release branch wins)
git checkout --theirs modules/networking/main.tf

# Option C: Use a merge tool for complex conflicts
git mergetool modules/networking/main.tf

# Option D: Manually edit the file to combine both changes
# (Most common for infrastructure — understand both sides, write the correct result)

# Step 4: Mark as resolved
git add modules/networking/main.tf
git add modules/iam/main.tf

# Step 5: Verify no remaining conflicts before committing
git diff --check  # Should show nothing

# Step 6: Complete the merge
git commit
# Default merge commit message is fine — do not change it
```

---

## Resolution: Using rerere (If Enabled)

`git rerere` records conflict resolutions so identical conflicts are resolved automatically in the future:

```bash
# Enable rerere globally (do this before an incident — for prevention)
git config --global rerere.enabled true

# If rerere is enabled and has seen this conflict before:
git merge origin/release-v2.4
# CONFLICT (content): Merge conflict in modules/networking/main.tf
# Recorded preimage for 'modules/networking/main.tf'  ← rerere is active

# rerere may auto-resolve based on past resolution
git status
# If the file shows as modified (not UU), rerere resolved it automatically

# Verify the auto-resolution is correct before staging
git diff modules/networking/main.tf

# If correct:
git add modules/networking/main.tf
git commit
```

---

## Resolution: Structured Approach for Large Conflicts

When the conflict is broad (many files, complex changes):

```bash
# Step 1: List all conflicting files and create a work plan
git status --short | grep "^UU" | awk '{print $2}'

# Step 2: Categorize conflicts
# Type A: Simple — one side is clearly correct (ownership is clear)
# Type B: Compound — both sides have valid changes, need merging
# Type C: Semantic — changes interact in non-obvious ways (need domain expert)

# Step 3: Assign Type C conflicts to the engineers who wrote them
# Get the authors for each side
git log --oneline --author-date-is-committer-date \
  HEAD...origin/release-v2.4 -- modules/networking/main.tf

# Step 4: Resolve in order — Type A, then B, then C
# Stage and verify after each file before moving to next
```

---

## Verification

```bash
# No remaining conflict markers
grep -rn "<<<<<<\|=======\|>>>>>>>" . --include="*.tf" --include="*.yml"
# (empty output — all conflicts resolved)

# The merge commit is clean
git log --oneline -3
# abc1234 Merge branch 'release-v2.4' into main
# def5678 ...
# ...

# Run CI checks on the resolved state before pushing
# (run your validation scripts locally)
terraform validate ./
tflint --recursive

# Push
git push origin main
```

---

## Prevention

```bash
# Merge main into the release branch regularly (not just at release time)
# This keeps the divergence small and conflicts manageable

# In the release branch:
git fetch origin
git merge origin/main --no-ff -m "chore: sync release-v2.4 with main [weekly sync]"
git push origin release-v2.4

# Set a policy: release branches must sync with main at least weekly
# Long-lived release branches with no syncs are the root cause of large conflicts
```

Enable `rerere` org-wide:

```bash
git config --global rerere.enabled true
git config --global rerere.autoupdate true
```

Consider using `git mergetool` with a visual diff tool (e.g., vimdiff, opendiff, IntelliJ):

```bash
git config --global merge.tool vimdiff
git config --global mergetool.keepBackup false
```
