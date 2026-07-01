# Lab 02: Resolve Merge Conflicts

**Difficulty:** Beginner  
**Time:** 25 minutes  
**Skills:** `git merge`, conflict markers, `git rerere`, merge strategies

---

## Objective

Resolve three types of merge conflicts — content conflict, delete/modify conflict, and binary conflict — and understand how `rerere` automates recurring resolutions.

---

## Setup

```bash
cd ~/git-labs
mkdir lab-02-merge-conflicts && cd lab-02-merge-conflicts
git init
git config rerere.enabled true         # Enable rerere for this lab

# Create a shared base file
cat > config.yaml << 'EOF'
database:
  host: localhost
  port: 5432
  name: myapp_db
  pool_size: 10

cache:
  host: localhost
  port: 6379

log_level: info
EOF

git add config.yaml
git commit -m "initial: add base config"
```

---

## Scenario A: Content Conflict (Most Common)

Two engineers modify the same line in different branches.

```bash
# Engineer A's branch: increases pool size for performance
git checkout -b feat/performance-tuning
sed -i.bak 's/pool_size: 10/pool_size: 25/' config.yaml && rm config.yaml.bak
echo "" >> config.yaml
echo "performance:" >> config.yaml
echo "  query_timeout: 30s" >> config.yaml
git add config.yaml
git commit -m "perf: increase db pool size and add query timeout"

# Engineer B's branch: changes pool size for cost reduction
git checkout main
git checkout -b feat/cost-reduction
sed -i.bak 's/pool_size: 10/pool_size: 5/' config.yaml && rm config.yaml.bak
echo "  connection_timeout: 10s" >> config.yaml
git add config.yaml
git commit -m "cost: reduce db pool size for lower connection costs"

# Now merge feat/performance-tuning into main
git checkout main
git merge feat/performance-tuning
git merge feat/cost-reduction          # This will conflict
```

---

## Walkthrough A: Resolving a Content Conflict

```bash
# 1. Check what conflicts exist
git status
# Both modified:   config.yaml

# 2. View the conflict
cat config.yaml
```

You will see conflict markers:
```
database:
  host: localhost
  port: 5432
  name: myapp_db
<<<<<<< HEAD
  pool_size: 25
=======
  pool_size: 5
>>>>>>> feat/cost-reduction
```

The format:
- `<<<<<<< HEAD` — your current branch (HEAD) version
- `=======` — separator
- `>>>>>>> feature-branch` — the incoming version

```bash
# 3. Edit the file to resolve: keep performance tuning's value, add cost comment
cat > config.yaml << 'EOF'
database:
  host: localhost
  port: 5432
  name: myapp_db
  pool_size: 25   # Performance: 25 connections; review for cost if > 5 environments

cache:
  host: localhost
  port: 6379

log_level: info

performance:
  query_timeout: 30s
  connection_timeout: 10s
EOF

# 4. Stage the resolved file
git add config.yaml

# 5. Complete the merge (use the pre-populated merge commit message)
git commit
```

---

## Scenario B: Delete/Modify Conflict

```bash
# Reset to a clean state for this scenario
git checkout main
mkdir lab-b && cd lab-b
git init

echo "LEGACY_CONFIG=true" > legacy.env
git add legacy.env && git commit -m "add legacy env file"

# Branch A: modifies the file
git checkout -b feat/update-legacy
echo "LEGACY_CONFIG=true" > legacy.env
echo "LEGACY_VERSION=2" >> legacy.env
git add legacy.env && git commit -m "update legacy config"

# Branch B: removes the file (it's not needed anymore)
git checkout main
git checkout -b feat/remove-legacy
git rm legacy.env
git commit -m "chore: remove legacy env file"

# Merge update-legacy first, then try to merge remove-legacy
git checkout main
git merge feat/update-legacy
git merge feat/remove-legacy         # Conflict: modified vs deleted
```

```bash
# Git will report: CONFLICT (modify/delete): legacy.env deleted in feat/remove-legacy
# and modified in HEAD

# Two valid resolutions:
# Option 1: Accept the deletion (the removal wins)
git rm legacy.env
git commit

# Option 2: Keep the file (the update wins)
git add legacy.env
git commit

# This decision requires human judgment about the intent of both changes
cd .. && rm -rf lab-b
```

---

## Scenario C: Using rerere

`rerere` (Reuse Recorded Resolution) records conflict resolutions and replays them automatically when the same conflict recurs.

```bash
# Continue in the original lab-02 directory
cd ~/git-labs/lab-02-merge-conflicts

# Enable rerere globally
git config --global rerere.enabled true

# Create a recurring conflict scenario
git checkout main
git checkout -b branch-a
echo "timeout: 30" >> config.yaml
git add config.yaml && git commit -m "set timeout to 30"

git checkout main
git checkout -b branch-b
echo "timeout: 60" >> config.yaml
git add config.yaml && git commit -m "set timeout to 60"

# Merge branch-a first
git checkout main
git merge branch-a

# Merge branch-b — conflict
git merge branch-b

# Resolve the conflict manually
# Edit config.yaml: choose timeout: 45  (or any resolved value)
echo "timeout: 45" >> config.yaml
git add config.yaml
git commit

# rerere has now recorded this resolution
# Next time the same conflict appears, rerere will auto-apply the resolution
git rerere status         # Shows recorded resolutions
```

---

## Key Concepts

1. **Conflict markers are not code.** They are Git's notation. Never commit a file containing `<<<<<<<` markers.
2. **`git diff --check`** detects leftover conflict markers before committing.
3. **The resolution choice matters.** Git cannot decide which change wins — that is an engineering decision. Read both changes and understand the intent.
4. **`git merge --abort`** is always available. If the conflict is complex, abort, discuss with the author, then retry.
5. **`rerere.enabled true` is a safety net** for rebases. If you rebase a branch with many commits, each commit may introduce the same conflict. `rerere` applies your previous resolution automatically.

---

## Verification

```bash
# After completing Scenario A:
git log --oneline --graph main

# Check no conflict markers remain
grep -r '<<<<<<<' . && echo "ERROR: conflict markers found" || echo "Clean"

# Verify both changes are present in the merged result
grep "pool_size" config.yaml
grep "query_timeout" config.yaml
```

---

## Cleanup

```bash
cd ~/git-labs
rm -rf lab-02-merge-conflicts
```

---

[← Lab 01: Recover Deleted Branch](01-recover-deleted-branch.md) | [Lab 03: Interactive Rebase →](03-interactive-rebase.md)
