# CS-01 — Accidental Force Push Overwrites Main

> **Category:** Recovery | **Complexity:** High | **Time to Resolve:** 35 minutes
>
> **Related:** [`recovery/`](../recovery/) | [`enterprise-workflows/`](../enterprise-workflows/) | [`reset-or-revert`](../decision-guides/reset-or-revert.md)

---

## Problem

An engineer on a platform team ran `git push --force origin main` on a Friday afternoon while trying to clean up their personal fork. The command was executed in the wrong terminal window — the one connected to the production infrastructure repository. Four commits were overwritten, including two that had been deployed to the staging environment 20 minutes earlier.

---

## Environment

- **Team size:** 18 engineers
- **Repository:** Terraform monorepo for AWS infrastructure (production)
- **Branch model:** GitHub Flow — protected `main`, feature branches
- **CI/CD:** GitHub Actions — deploys on merge to `main`
- **Detection:** A senior engineer noticed the missing commits when reviewing the GitHub Actions run history

---

## Timeline

| Time | Event |
|---|---|
| 15:42 | Engineer pushes `--force` to `main` — 4 commits overwritten |
| 15:44 | Senior engineer notices gap in commit history while reviewing a CI run |
| 15:45 | Incident declared. Repository write access locked via GitHub branch protection |
| 15:48 | Engineer with local clone identified as having all 4 missing commits |
| 15:56 | Missing commits pushed back to `main` via `--force-with-lease` |
| 16:05 | Branch protection restored. Incident closed |
| 16:17 | Post-incident review scheduled |

---

## Root Cause

No controls prevented force push to `main`. The engineer had direct write access. Branch protection rules existed but did not have **"Do not allow force pushes"** enabled.

Secondary cause: multiple terminal windows with different git contexts — a common developer environment pattern with no mitigation.

---

## Diagnosis

```bash
# On a local clone (not the engineer who force-pushed)
git fetch origin

# Check the reflog for origin/main
git reflog show origin/main
# origin/main@{0}: push --force: 3f8a2b1 chore: update provider (force-pushed state)
# origin/main@{1}: commit: 7a9c4e2 fix(eks): correct node group scaling policy
# origin/main@{2}: commit: 5b8d3f1 feat(iam): add cross-account role trust policy
# origin/main@{3}: commit: 2e6c1a4 feat(vpc): add NAT gateway outputs
# origin/main@{4}: commit: 1d5b0c3 chore: bump provider versions (original force-push base)

# The 4 missing commits are visible in the reflog:
# 7a9c4e2, 5b8d3f1, 2e6c1a4, 1d5b0c3 (original, kept)
```

---

## Resolution

```bash
# Step 1: Identify the correct HEAD SHA (state before force push)
# origin/main@{1} is the last good state before the force push
git log origin/main@{1} --oneline -5
# 7a9c4e2 fix(eks): correct node group scaling policy
# 5b8d3f1 feat(iam): add cross-account role trust policy
# 2e6c1a4 feat(vpc): add NAT gateway outputs
# 1d5b0c3 chore: bump provider versions

# Step 2: Verify the commits are correct
git show 7a9c4e2
git show 5b8d3f1

# Step 3: Restore by pushing the correct SHA back to origin/main
# Using --force-with-lease for safety
git push origin origin/main@{1}:main --force-with-lease

# Step 4: Verify
git fetch origin
git log origin/main --oneline -5
# 7a9c4e2 fix(eks): correct node group scaling policy   ← restored
# 5b8d3f1 feat(iam): add cross-account role trust policy ← restored
# ...
```

---

## Prevention — Branch Protection Configuration

GitHub settings applied after this incident:

```
Repository Settings → Branches → Branch protection rules → main

✅ Require a pull request before merging
✅ Require approvals (minimum: 2)
✅ Dismiss stale pull request approvals when new commits are pushed
✅ Require status checks to pass before merging
✅ Require branches to be up to date before merging
✅ Do not allow bypassing the above settings
✅ Restrict who can push to matching branches
✅ Allow force pushes → DISABLED   ← This was the key missing control
✅ Allow deletions → DISABLED
```

---

## Verification

```bash
# Confirm all 4 commits are restored
git log origin/main --oneline | head -10

# Confirm the SHA the CI/CD pipeline deployed from is present
git show <deployed-sha>

# Confirm branch protection now prevents force push
git push --force origin main
# remote: error: GH006: Protected branch update failed for refs/heads/main.
# remote: error: Force pushes are not allowed. (exit status 128)
```

---

## Lessons Learned

1. **"Do not allow force pushes" must be enabled on all protected branches.** It was in the branch protection documentation but not applied. This single missing toggle caused the incident.

2. **The recovery was possible because another engineer had a current local clone.** If everyone had been running CI-only shallow clones, the reflog on local machines would not have contained the missing commits. Always ensure at least some engineers have full clones.

3. **Detection was by accident.** A senior engineer noticed a gap by coincidence. Proper monitoring should alert on commits disappearing from a protected branch — GitHub Audit Log webhooks can detect this.

4. **Multiple terminal windows with different git contexts is a known failure mode.** No organizational solution was implemented, but engineers adopted a practice of always running `git remote -v` before any destructive operation.

---

## Best Practices

- Enable "Do not allow force pushes" on `main` and all `release/*` branches before your first engineer joins
- Enable "Restrict who can push" — only admins should have push access to `main`
- Instrument GitHub Audit Log → webhook → Slack/PagerDuty for protected branch force push events
- Require that at least 3 engineers maintain current full local clones at all times — these are your recovery copies
- Document recovery procedures before an incident, not during one
