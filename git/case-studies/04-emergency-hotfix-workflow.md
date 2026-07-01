# CS-04 — Emergency Hotfix During Release Freeze

> **Category:** Hotfix, Cherry-Pick | **Complexity:** High | **Time to Resolve:** 47 minutes
>
> **Related:** [`cherry-pick/`](../cherry-pick/) | [`tags/`](../tags/) | [`enterprise-workflows/`](../enterprise-workflows/)

---

## Problem

During a scheduled release freeze (a 72-hour window where no planned changes are allowed to production), a critical vulnerability was identified in an IAM role configuration. The role had `sts:AssumeRole` with a wildcard principal — any AWS account could assume it. This required an emergency fix to production immediately, bypassing the change freeze.

---

## Environment

- **Team:** Platform Engineering, 12 engineers
- **Branch model:** GitFlow — `main` (production), `develop`, `release/*`, `hotfix/*`
- **Release state:** Release freeze for quarterly compliance review
- **Production tag:** `v2024-q3.0` (deployed 18 hours before the discovery)
- **CI/CD:** GitHub Actions — deploys on tag push to production environment

---

## Timeline

| Time | Event |
|---|---|
| 09:15 | Security scanner identifies wildcard principal in `modules/iam/cross-account-role.tf` |
| 09:18 | Platform Lead declares emergency change — change freeze exception granted |
| 09:22 | Hotfix branch cut from production tag |
| 09:31 | Fix committed, peer-reviewed by security team |
| 09:34 | Fix tagged as `v2024-q3.1`, CI pipeline triggered |
| 09:47 | Hotfix deployed to production — IAM role updated |
| 09:49 | Security scanner re-run — issue resolved |
| 09:52 | Fix cherry-picked to `develop` for inclusion in next release |
| 10:15 | Post-incident review scheduled |

---

## Hotfix Procedure

```bash
# Step 1: Branch from the PRODUCTION TAG — not from main or develop
# main and develop may have unreleased work
git fetch origin --tags
git checkout -b hotfix/SEC-447-iam-wildcard-principal v2024-q3.0

# Step 2: Apply the minimal fix
# Edit modules/iam/cross-account-role.tf:
# Change: principals = ["*"]
# To:     principals = ["arn:aws:iam::123456789012:root"]

git add modules/iam/cross-account-role.tf
git commit -m "fix(iam): restrict cross-account role to specific principal [SEC-447]

The sts:AssumeRole trust policy had a wildcard principal (*) allowing
any AWS account to assume this role. Restricted to the specific
trusted account ARN.

Severity: Critical
Incident: INC-8847
Reviewed-by: security-team
Change-exception: CE-2024-Q3-001"

# Step 3: Push hotfix branch and open PR (even under pressure — get one reviewer)
git push -u origin hotfix/SEC-447-iam-wildcard-principal

# Step 4: After approval, merge to main and tag
git checkout main
git merge --no-ff hotfix/SEC-447-iam-wildcard-principal \
  -m "fix: merge hotfix for IAM wildcard principal [SEC-447]"

# Step 5: Tag the patch release
git tag -a v2024-q3.1 -m "Hotfix v2024-q3.1 — SEC-447 IAM wildcard principal

Emergency patch for critical IAM misconfiguration.
Change exception: CE-2024-Q3-001
Incident: INC-8847"

git push origin main
git push origin v2024-q3.1

# CI/CD deploys automatically on tag push
```

---

## Back-Merging to Develop

Critical: The fix must reach `develop` or it will be absent from the next release cycle.

```bash
# Step 6: Get the hotfix commit SHA
git log main --oneline | head -3
# abc1234 fix: merge hotfix for IAM wildcard principal [SEC-447]
# def5678 ...previous commit

# Identify the actual fix commit (not the merge commit)
git log main --oneline | grep "SEC-447"
# 789abcd fix(iam): restrict cross-account role to specific principal [SEC-447]

# Cherry-pick the fix to develop
git checkout develop
git cherry-pick 789abcd -e
# Add to message:
# Backport-of: 789abcd (main)
# Part-of: hotfix/SEC-447

git push origin develop
```

Alternatively, merge the hotfix branch directly to develop:

```bash
git checkout develop
git merge --no-ff hotfix/SEC-447-iam-wildcard-principal \
  -m "fix: back-merge SEC-447 hotfix to develop"
git push origin develop
```

---

## Verification

```bash
# Verify the role trust policy is updated in production
aws iam get-role --role-name cross-account-assume-role \
  --query 'Role.AssumeRolePolicyDocument'

# Verify no wildcard principals exist in the module
grep -r '"*"' modules/iam/

# Verify fix is in both main and develop
git log main --oneline | grep "SEC-447"
git log develop --oneline | grep "SEC-447"

# Verify the tag is correct
git show v2024-q3.1 | head -15
```

---

## Hotfix Checklist (Laminate and Post at Desks)

```
□ Change exception granted before starting
□ Hotfix branch cut from production TAG — not from develop or main
□ Fix is MINIMAL — no refactoring, no unrelated changes
□ Commit message includes incident reference, reviewer, exception number
□ At least one reviewer (security team if security-related)
□ CI passes on hotfix branch before merge
□ Merge to main with --no-ff
□ Tag with patch version (MAJOR.MINOR.PATCH)
□ Deploy and verify in production
□ Back-merge or cherry-pick to develop
□ Delete hotfix branch
□ Update incident ticket with resolution details
□ Schedule post-incident review
```

---

## Lessons Learned

1. **Branch from the production tag, not from main.** `main` at the time of this incident had 3 unreleased features in the release freeze queue. Branching from `main` would have included those changes in the hotfix deployment.

2. **Even during emergencies, get one reviewer.** The security team reviewed the 3-line fix in 4 minutes. The alternative — deploying unreviewed changes to production — is a worse risk than the 4-minute delay.

3. **The back-merge step is the one teams forget.** It is the last thing done under pressure when people want to close the incident. Without it, the bug reappears in the next major release. Put it on the checklist.

4. **Minimal fix only.** The engineer originally wanted to refactor the entire IAM module while they had it open. This was rejected — the hotfix PR should contain the minimum change that resolves the vulnerability. Refactoring waits for a regular release.

---

## Best Practices

- Maintain a written emergency change procedure that defines: who approves, who reviews, what tags to use
- Practice the hotfix procedure quarterly with a non-production scenario — the first time running it should not be during a production incident
- Configure GitHub to allow one named role (Security Lead) to merge during a freeze, but require their explicit approval
- Alert on any `sts:AssumeRole` with wildcard principals in automated security scanning — this should have been caught in CI, not by a quarterly scanner
