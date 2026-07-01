# Playbook 02: Production Hotfix

**Risk Level:** P1 — Critical  
**Time to Execute:** 30–90 minutes (depends on fix complexity)  
**Affected Systems:** Production deployment pipeline, `main`, release tags

---

## Situation

A critical bug, security vulnerability, or data integrity issue exists in production. The normal development cycle (feature branch → PR → review → merge → deploy) is too slow. You need to ship a fix to production within 60 minutes.

**Hotfix criteria — meet at least one:**
- Production is actively degraded or data loss is occurring
- Security vulnerability is being actively exploited
- Regulatory or contractual SLA breach is imminent
- Revenue impact exceeds the cost of the expedited review process

---

## Prerequisites

- [ ] Incident declared and incident commander assigned
- [ ] The commit in production is known: `git tag` or check deployment tags
- [ ] The fix is understood and scoped — not exploratory
- [ ] At least one other engineer available to review (async is acceptable)
- [ ] CI/CD pipeline functional (if it's also broken, see [Playbook 07: Disaster Recovery](07-disaster-recovery.md))

---

## Step 1: Identify the Production Commit

```bash
# Check the deployed version
git fetch --tags origin
git log --oneline v2.3.0          # If using semantic versioning tags

# OR check your deployment system (Kubernetes, ECS, Heroku) for the git SHA
# It should be visible in the pod/container labels or environment variables

# Record this SHA
PROD_SHA=$(git rev-parse v2.3.0)
echo "Production is at: $PROD_SHA"
```

---

## Step 2: Create the Hotfix Branch

Branch from the production tag or SHA — **not** from `main`, which may contain unreleased changes:

```bash
git checkout -b hotfix/INC-$(date +%Y%m%d)-<description> v2.3.0
# Example: hotfix/INC-20240315-rds-connection-leak

git log --oneline -5               # Verify you're at the right starting point
```

---

## Step 3: Apply the Fix

Make the smallest possible change that fixes the issue. This is not the time for refactoring or "while I'm here" changes.

```bash
# Apply the fix
# ... edit files ...

git add -p                          # Review every hunk before staging
git status                          # Confirm nothing extra is staged

git commit -m "fix(INC-0000): describe the exact issue in one line

Root cause: [one sentence]
Fix: [one sentence]
Risk: [low/medium — what could this change break?]"
```

---

## Step 4: Expedited Review

This step is non-negotiable even in emergencies. A second pair of eyes has caught production-breaking bugs in "obvious" fixes many times.

```bash
git push origin hotfix/INC-20240315-rds-connection-leak
```

Options for expedited review:
1. **Async:** Push the branch, ping the on-call reviewer, review via GitHub while you continue
2. **Synchronous:** Share your screen, walk the reviewer through the diff live (15 minutes)
3. **Post-incident review:** If situation is extreme (data loss in progress), document intent and review within 30 minutes post-fix

Do not merge without at least one approval, even if you have to temporarily adjust branch protection settings.

---

## Step 5: Merge the Hotfix

```bash
# After review approval, merge via PR
# OR if you have direct merge rights and CI is too slow:

git checkout main
git merge --no-ff hotfix/INC-20240315-rds-connection-leak -m "fix(INC-0000): production hotfix - rds connection leak"

# Also merge to develop/release branches if they exist
git checkout develop
git merge --no-ff hotfix/INC-20240315-rds-connection-leak
```

---

## Step 6: Tag the Hotfix Release

```bash
git checkout main
git tag -a v2.3.1 -m "Hotfix v2.3.1: fix RDS connection leak (INC-0000)"
git push origin main --follow-tags
```

---

## Step 7: Deploy and Verify

Follow your deployment procedure. After deployment:

```bash
# Verify the fix is deployed
git describe --tags HEAD             # Should show v2.3.1

# Smoke test the specific scenario that was broken
# Monitor error rates for 15 minutes post-deploy
```

---

## Step 8: Post-Incident

Within 24 hours:

```bash
# Clean up the hotfix branch
git push origin --delete hotfix/INC-20240315-rds-connection-leak
git branch -d hotfix/INC-20240315-rds-connection-leak

# Schedule the post-incident review (PIR) meeting
```

Document in the PIR:
- Root cause
- Why it reached production (what process gap?)
- How to prevent recurrence
- Action items with owners and deadlines

---

## Rollback Strategy

If the hotfix makes things worse:

```bash
# Revert the hotfix commit on main
git revert v2.3.1                   # Creates a new commit that undoes v2.3.1
git tag -a v2.3.2 -m "Revert hotfix v2.3.1: caused [issue]"
git push origin main --follow-tags
# Deploy v2.3.2
```

Do not `git reset` or `git rebase` on `main`. Always use `git revert` for published commits.

---

## Checklist Summary

```
□ Production commit SHA identified
□ Hotfix branch created from production tag/SHA (not main)
□ Smallest possible fix applied
□ Commit message references incident number
□ At least one reviewer has approved
□ Merged to main with --no-ff
□ Merged to develop/release branches
□ Hotfix tagged as patch release
□ Tags pushed (git push --follow-tags)
□ Deployed and smoke tested
□ Hotfix branch deleted
□ PIR scheduled
```

---

## Related

- [Case Study: Emergency Hotfix Workflow](../case-studies/04-emergency-hotfix-workflow.md)
- [Production Rollback](05-produce-rollback.md) ← if hotfix doesn't work
- [Tags Reference](../tags/README.md)

---

[← Playbook 01: Force Push Recovery](01-force-push-recovery.md) | [Playbook 03: Repository Migration →](03-repository-migration-checklist.md)
