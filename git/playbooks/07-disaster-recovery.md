# Playbook 07: Disaster Recovery

**Risk Level:** P1 — Critical  
**Time to Execute:** 30 minutes – 4 hours  
**Affected Systems:** Repository availability, team productivity, deployment capability

---

## Situation

One or more of the following is true:
- The remote repository is unavailable (platform outage, accidental deletion, account suspension)
- A local repository is corrupt (`git status` reports errors, or `git fsck` shows missing objects)
- History has been destructively rewritten on the main branch with no backup
- A `git gc --prune=now` was run and deleted unreachable objects that were still needed

---

## Incident Classification

| Symptom | Severity | Recovery Method |
|---------|----------|-----------------|
| Remote platform is down (GitHub outage) | Operational pause | Work from local, wait |
| Repository accidentally deleted on GitHub | P1 | GitHub support recovery |
| Local `.git` directory corrupted | Developer-local | Re-clone from remote |
| Remote history rewritten, no backups | P1 | Restore from any surviving clone |
| Objects missing after aggressive `git gc` | P1 | Restore from backups |

---

## Scenario 1: Remote Platform Outage

**Indicator:** `git fetch origin` returns connection refused or 503.

```bash
# Verify it's a platform issue, not a network issue
curl -I https://github.com          # For GitHub
# OR: check https://www.githubstatus.com

# You can continue working locally
git commit -m "feat: ..."           # Commits work without remote access
git log --oneline                   # Full history is local

# When the platform is back, resume normal operations
git push origin main
```

If the outage is extended (> 2 hours) and deployment is needed:

1. Deploy from any team member's local clone or the last CI artifact
2. Tag the deployed commit in the local clone
3. Resume normal remote operations after recovery

---

## Scenario 2: Repository Accidentally Deleted on GitHub

**Immediate response time matters — GitHub can restore within 90 days if not overwritten.**

```bash
# Step 1: Do NOT create a new repository with the same name
# Creating a new repo at the same URL prevents GitHub from restoring the old one

# Step 2: Contact GitHub Support immediately
# https://support.github.com → "Repository accidentally deleted"
# Include: organization name, repository name, approximate deletion time

# Step 3: While waiting, identify which team member has the most recent local clone
git log --oneline -5               # Check the last commit date on all available machines

# Step 4: Push the best available local clone to a temporary repository
git remote add recovery git@github.com:org/repo-recovery.git
git push recovery --mirror

# Step 5: Identify any CI artifacts for recent commits
# Most CI systems cache build artifacts with the git SHA label
```

---

## Scenario 3: Corrupt Local Repository

**Indicator:** `git status` returns errors like "error: object file ... is empty" or "fatal: loose object ... is corrupt"

```bash
# Check integrity
git fsck --full 2>&1 | head -30

# If the remote is still healthy (most common case):
# The fastest fix is to re-clone
cd ..
git clone git@github.com:org/repo.git repo-fresh

# Copy over any uncommitted work from the corrupt clone
cp -r repo-corrupt/src/my-changes repo-fresh/src/
cd repo-fresh
git status                         # Should show your uncommitted changes cleanly
```

---

## Scenario 4: Severe History Rewrite — Restore from Surviving Clone

If the remote history has been destructively rewritten (mass force-push, filter-repo applied incorrectly) and team members still have local clones:

```bash
# Step 1: Identify the best surviving clone
# Ask all team members to run:
git log --oneline origin/main | head -5  # What does their remote/main show?
git reflog | head -5                     # What is their local HEAD state?

# Step 2: On the machine with the most complete history
git log --oneline -20                    # Verify completeness
git tag                                  # Verify tags are present

# Step 3: Create a mirror push back to the remote
# Temporarily disable branch protection if needed
git push origin main --force-with-lease

# Step 4: Push all tags
git push origin --tags

# Step 5: Notify team to re-fetch
git fetch --prune --tags origin
```

---

## Scenario 5: Objects Pruned by git gc

If `git gc --prune=now --aggressive` was run and dangling objects were needed:

```bash
# Check what objects remain
git fsck --lost-found 2>/dev/null | wc -l

# If objects are gone from the local repo, check the remote
git fetch origin
git log --all --oneline | head -20   # Does the remote have more history?

# If remote still has the history:
git reset --hard origin/main         # Restore from remote

# If both local and remote are affected:
# You need a third machine, backup, or CI artifact
```

---

## Recovery Readiness Checklist

Run this quarterly to ensure you are ready for a disaster before it happens:

```bash
# 1. Verify at least 2 full local clones exist among team members
git log --oneline | wc -l            # Check depth of each clone

# 2. Verify CI artifacts are being stored with commit SHAs
# Each build should tag its artifacts with the git SHA

# 3. Verify the backup mirror is current (if you have one configured)
# git clone --mirror should run weekly via cron

# 4. Verify GitHub Backup Utilities or third-party backup is configured
# Options: BackHub, Gitea, periodic mirror to S3

# 5. Document your recovery procedures and test them
# Run through Scenarios 3 and 4 in a test repository
```

---

## Related

- [Playbook 01: Force Push Recovery](01-force-push-recovery.md)
- [Production Incident P001](../production-incidents/P001-force-push-overwrites-main.md)
- [Recovery Reference](../recovery/README.md)
- [Case Study: Repository Migration](../case-studies/06-repository-migration.md)

---

[← Playbook 06: Recover Lost Commits](06-recover-lost-commits.md) | [Playbook 08: Repository Maintenance →](08-repository-maintenance.md)
