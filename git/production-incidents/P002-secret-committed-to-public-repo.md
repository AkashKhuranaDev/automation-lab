# P002 — Secret Committed to Public Repository

> **Severity:** P1 | **Expected Recovery Time:** 25–45 minutes (initial response) + 4–8 hours (full remediation)
>
> **Related:** [`CS-03`](../case-studies/03-secrets-in-commit-history.md) | [`security/`](../security/) | [`hooks/`](../hooks/)

---

## Symptoms

- GitHub Secret Scanning alert received (email or GitHub Security tab)
- AWS GuardDuty, Splunk, or SIEM alert on credentials used from unknown IP
- Engineer reports accidentally committing a file containing credentials
- `.env`, `credentials`, `config.json`, or similar file appears in a push

---

## Critical Ordering Rule

> **ROTATE FIRST. CLEAN GIT SECOND.**
>
> Credential rotation takes 30 seconds. Git history cleanup takes 4 hours.
> Every second spent on Git before rotating is a second with a live credential.

---

## Immediate Actions (Do in Order)

```bash
# 1. ROTATE THE CREDENTIAL — FIRST ACTION, NO EXCEPTIONS
# AWS IAM: Console → IAM → Users → <user> → Security credentials → Make inactive → Delete
# AWS CLI:
aws iam delete-access-key --user-name <user> --access-key-id AKIA...

# 2. Set repository to private (if it is public)
# GitHub → Settings → Change repository visibility → Make private
# (Or use the API:)
curl -X PATCH \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Content-Type: application/json" \
  "https://api.github.com/repos/$ORG/$REPO" \
  -d '{"private": true}'

# 3. Revoke any active sessions using the compromised credential
# AWS: Attach DenyAll policy to the IAM user temporarily
aws iam put-user-policy \
  --user-name <user> \
  --policy-name EmergencyDeny \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Deny","Action":"*","Resource":"*"}]}'
```

---

## Diagnosis

```bash
# Identify the commit containing the secret
git log --all --oneline --diff-filter=A -- .env
# or search by content
git log --all -S "AWS_SECRET_ACCESS_KEY" --oneline

# Find which branches/tags contain the commit
COMMIT_SHA="<commit-sha-from-above>"
git branch -a --contains $COMMIT_SHA
git tag --contains $COMMIT_SHA

# Determine the blast radius window
git show $COMMIT_SHA --format="%ci" | head -1
# 2024-07-01 15:42:03 +0000
# This is when the secret was exposed — review SIEM/CloudTrail from this time
```

---

## Blast Radius Assessment

```bash
# AWS: List all API calls made with the exposed key
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=<exposed-key-id> \
  --start-time "<exposure-time>" \
  --query 'Events[*].[EventTime,EventName,SourceIPAddress,Username]' \
  --output table

# Check if any resources were created/modified/deleted
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=<exposed-key-id> \
  --query 'Events[?contains(`["CreateBucket","PutObject","RunInstances","CreateUser"]`, EventName)]'
```

---

## Git History Remediation

After rotating credentials and assessing blast radius:

```bash
# Install git-filter-repo
pip install git-filter-repo

# Work on a fresh mirror clone (not your working directory)
git clone --mirror git@github.com:$ORG/$REPO.git repo-clean
cd repo-clean

# Remove the specific file from all history
git filter-repo --path .env --invert-paths
# or if the secret was in a config file that you want to keep (but cleaned):
git filter-repo --replace-text <(echo "AKIA...==>REDACTED")

# Verify the file/string is gone from all history
git log --all --oneline -- .env  # Should return nothing
git log --all -S "AKIA..." --oneline  # Should return nothing

# Force push all branches and tags
git push --force --all
git push --force --tags
```

---

## Engineer Re-Clone Notification

After force push, all existing clones have the old (polluted) history:

```
ACTION REQUIRED — All engineers must re-clone [REPO-NAME]

A security-related history rewrite was performed on [REPO-NAME] at [TIME].
Your existing local clone contains the old history.

To update your local clone:
  cd ..
  rm -rf [repo-directory]
  git clone git@github.com:[ORG]/[REPO].git

Do NOT run `git pull` on your existing clone — this will fail.

If you have uncommitted local work, save it first:
  git stash
  git stash show -p > /tmp/my-wip-patch.diff
```

---

## Verification

```bash
# Confirm the secret is gone from the public history
git log --all -S "AKIA" --oneline  # empty

# Confirm the file is gone
git log --all --full-history -- .env  # empty

# Confirm AWS credential is invalid
aws sts get-caller-identity --profile <profile-using-old-key>
# An error occurred (InvalidClientTokenId) ... (expected — key is deleted)

# Confirm GitHub Secret Scanning shows resolved
# GitHub → Security tab → Secret scanning → Mark alert as resolved
```

---

## Prevention

```yaml
# Add to all repositories — .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.4
    hooks:
      - id: gitleaks

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: detect-private-key
```

```gitignore
# Add to .gitignore template in all repos
.env
.env.*
*.env
credentials
*.pem
*.key
secrets/
config/local.yml
```

GitHub organization settings:
- Enable **Secret scanning** for all repositories
- Enable **Push protection** — blocks pushes containing known secret patterns before they reach GitHub
- Enable **Secret scanning alerts** routing to the security team
