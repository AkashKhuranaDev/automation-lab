# CS-03 — AWS Credentials Committed to Public Repository

> **Category:** Security | **Complexity:** High | **Time to Resolve:** 4 hours (initial response) + 1 week (full remediation)
>
> **Related:** [`security/`](../security/) | [`hooks/`](../hooks/) | [`best-practices/`](../best-practices/)

---

## Problem

An infrastructure engineer committed a `.env` file containing AWS access keys to a public GitHub repository. The repository was a personal fork of a company Terraform module. The credentials had `PowerUserAccess` on the production AWS account. GitHub secret scanning triggered a notification 4 minutes after the push.

---

## Environment

- **Repository:** Public GitHub (personal fork, meant to be private)
- **Credentials exposed:** AWS IAM user with `PowerUserAccess` on production account
- **Detection:** GitHub Secret Scanning alert + AWS GuardDuty alert (API calls from unknown IP)
- **Blast radius:** Full read/write access to all S3 buckets, EC2 instances, and RDS databases in the production account

---

## Timeline

| Time | Event |
|---|---|
| T+0 | Engineer pushes `.env` file containing `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` |
| T+4 | GitHub Secret Scanning fires — email notification to repository owner |
| T+6 | AWS GuardDuty fires — API calls from external IP to list S3 buckets |
| T+9 | Security team receives both alerts simultaneously |
| T+12 | AWS IAM credentials rotated (key invalidated) |
| T+15 | AWS GuardDuty API calls stop — attacker's session invalidated |
| T+22 | Repository set to private |
| T+35 | `git filter-repo` history rewrite completed |
| T+55 | All engineers re-cloned — force push propagated |
| T+4h | AWS CloudTrail audit of all API calls during exposure window completed |

---

## Root Cause

No secrets scanning hook or CI check existed. The `.env` file was not in `.gitignore`. The engineer believed the fork was private — it was public by default because the source repository was public.

---

## Immediate Response (Do This in Order)

> **Order matters.** Rotate credentials before fixing Git history. Assume the secret was read the moment it was pushed.

```bash
# Step 1: Rotate credentials FIRST — do this before touching Git
# AWS Console → IAM → Users → engineer-user → Security credentials
# → Access keys → Make inactive → Delete

# Step 2: Verify GuardDuty findings — understand blast radius
aws guardduty list-findings --detector-id <id> \
  --finding-criteria '{"Criterion": {"createdAt": {"Gte": <timestamp>}}}'

# Step 3: Review CloudTrail for what the attacker did
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=AKIAIOSFODNN7EXAMPLE \
  --start-time "2024-07-01T15:00:00Z" \
  --end-time "2024-07-01T16:00:00Z" \
  --query 'Events[*].[EventTime,EventName,SourceIPAddress]' \
  --output table

# Step 4: Set repository to private immediately
# GitHub → Settings → Change repository visibility → Private

# Step 5: Determine if any downstream systems still hold the credential
# (CI systems, other engineers' machines, Secrets Manager — none in this case)
```

---

## Removing the Secret from History

After rotating credentials (critical prerequisite — rotating is the security fix, history removal is housekeeping):

```bash
pip install git-filter-repo

# Clone a fresh copy for the filter operation
git clone git@github.com:username/repo.git repo-clean
cd repo-clean

# Remove the .env file from all history
git filter-repo --path .env --invert-paths

# Verify the file is gone from all history
git log --all --full-history -- .env
# (no output — good)

# Add .env to .gitignore and commit
echo ".env" >> .gitignore
git add .gitignore
git commit -m "security: add .env to gitignore [post-incident remediation]"

# Force push (team was notified and stopped all work)
git push origin --force --all
git push origin --force --tags

# All engineers must re-clone or reset
git fetch --all
git reset --hard origin/main
```

---

## Team Re-Clone Process

After the force push, all existing clones have the old history with the secret:

```bash
# Each engineer runs:
cd ..
rm -rf repo
git clone git@github.com:username/repo.git
```

Attempting to pull without re-cloning will cause conflicts between local and remote history.

---

## Prevention — Implemented After This Incident

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.4
    hooks:
      - id: gitleaks

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: detect-private-key
      - id: check-added-large-files
```

```gitignore
# Added to all repository templates
.env
.env.*
*.env
secrets/
credentials
*.pem
*.key
```

GitHub repository settings applied:
- Secret scanning enabled (already on for public repos)
- Push protection enabled (blocks push if secret pattern detected)
- Repository set to private until reviewed

---

## Lessons Learned

1. **Rotate first.** Every minute spent on Git cleanup before credential rotation is a minute the attacker has valid credentials. The Git history cleanup is a 4-hour task. Credential rotation is a 30-second task.

2. **GitHub sets fork visibility to match the parent.** Engineers who believe they're working privately in a fork of a public repo are wrong. Verify repository visibility before pushing anything sensitive.

3. **A 4-minute exposure window is enough.** Automated bots continuously monitor public GitHub for credential patterns. GuardDuty fired because the attacker's tooling triggered within 6 minutes of the push. The credential was read before the engineer noticed the Secret Scanning alert.

4. **CloudTrail saved the post-incident analysis.** Because the AWS account had CloudTrail logging, every API call with the compromised key was recorded. This allowed the team to confirm that only S3 list operations were performed — no data was exfiltrated.

5. **`git filter-repo` is non-negotiable.** The deprecated `git filter-branch` would have taken 45 minutes for this repository. `git filter-repo` took 8 seconds.

---

## Best Practices

- Enable pre-commit secrets scanning in every repository on day one — not after an incident
- Enable GitHub push protection (blocks pushes containing known secret patterns before they reach GitHub)
- Add a `.gitignore` template to every repository that explicitly excludes `.env`, `*.pem`, `credentials/`
- Train engineers that "public fork of a public repo" means the fork is public — it is not a private copy
- Document the incident response procedure for compromised credentials and test it annually
