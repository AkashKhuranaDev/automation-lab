# CS-05 — Production Rollback After Terraform Deployment Outage

> **Category:** Rollback, Revert | **Complexity:** High | **Time to Resolve:** 22 minutes (Git) + 35 minutes (infrastructure)
>
> **Related:** [`recovery/`](../recovery/) | [`tags/`](../tags/) | [`decision-guides/reset-or-revert`](../decision-guides/reset-or-revert.md)

---

## Problem

A Terraform change to security group rules for the production RDS cluster blocked inbound traffic from the application tier. The deployment completed successfully from Terraform's perspective (no plan errors) but caused a connectivity outage. All application instances returned `connection refused` on database connections within 90 seconds of the deployment.

---

## Environment

- **Infrastructure:** AWS RDS PostgreSQL cluster, ECS application tier
- **Repository:** Terraform monorepo (private GitHub)
- **Deployment trigger:** Merge to `main` → GitHub Actions → `terraform apply`
- **Production tag at deployment:** `v2024.07.15-1400`
- **Duration of outage:** 22 minutes

---

## Timeline

| Time | Event |
|---|---|
| 14:00 | Deployment pipeline starts — merge to main triggers GitHub Actions |
| 14:03 | `terraform apply` completes — security group rules updated |
| 14:04 | Application health checks start failing — HTTP 500 on all endpoints |
| 14:05 | On-call engineer paged — database connectivity errors in logs |
| 14:06 | RCA begins — recent Terraform deployment identified as suspect |
| 14:09 | Decision: rollback via revert (not forward-fix) — insufficient time to diagnose root cause |
| 14:14 | Revert commit created, approved, merged |
| 14:17 | Rollback `terraform apply` completes |
| 14:18 | Security group rules restored — application connectivity recovering |
| 14:22 | All health checks passing — incident closed |

---

## Rollback Procedure

### Option A: Git Revert (Used — Preserves History)

Revert creates a new commit that undoes the changes. The broken commit remains in history, which is correct for shared branches.

```bash
# Find the merge commit SHA that triggered the bad deployment
git log main --oneline | head -5
# 7f3a8c2 Merge pull request #412: update rds security group rules
# a4b9e1d feat(networking): add VPC peering to dev account
# ...

# Revert the merge commit
# -m 1 = revert to first parent (main branch, not the PR branch)
git revert -m 1 7f3a8c2 -e
# Commit message:
# revert: undo RDS security group update [INCIDENT-9923]
#
# Reverts: 7f3a8c2
# Reason: Security group change blocked application-to-RDS connectivity
# Incident: INCIDENT-9923
# On-call: engineering@company.com

# Push (the commit was peer-reviewed by second engineer while the first drafted it)
git push origin main

# CI/CD triggers terraform apply automatically on push to main
```

### Option B: Direct Terraform State Rollback (Alternative — Faster but Higher Risk)

```bash
# If the revert-and-apply would take too long, restore from state backup
# List state snapshots in S3
aws s3 ls s3://company-terraform-state/production/ | tail -10

# Download the pre-deployment state
aws s3 cp s3://company-terraform-state/production/terraform.tfstate.backup \
  /tmp/terraform-rollback.tfstate

# Apply the previous state configuration — DO NOT do this without locking
terraform state push /tmp/terraform-rollback.tfstate

# This approach is dangerous — only for P1 incidents where revert takes too long
```

### Option C: AWS Console Emergency (Fastest — No Git Required)

During active P1 incidents, if Git processes would take longer than 5 minutes:

```bash
# Restore the security group rules directly via AWS CLI
aws ec2 revoke-security-group-ingress \
  --group-id sg-0abc123def456789 \
  --protocol tcp --port 5432 \
  --source-group sg-0xyz987 

# Then add the correct rules back
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc123def456789 \
  --protocol tcp --port 5432 \
  --source-group sg-0application456

# Then create a Terraform import to reconcile state
terraform import aws_security_group_rule.rds_app_ingress <resource-id>
```

---

## Root Cause Analysis

```bash
# Diff the problematic commit against its parent
git diff 7f3a8c2^..7f3a8c2 -- modules/networking/security-groups.tf

# The diff shows:
# -  from_port   = 5432
# -  to_port     = 5432
# -  protocol    = "tcp"
# -  source_security_group_id = aws_security_group.app.id
# +  from_port   = 5432
# +  to_port     = 5432
# +  protocol    = "tcp"
# +  cidr_blocks = ["10.0.0.0/8"]   # Wrong — too broad AND wrong type
```

The engineer changed from a security group reference to a CIDR block. The ECS tasks are in a different /16 than the specified /8, so they were not included. Additionally, mixing `cidr_blocks` and `source_security_group_id` is a Terraform resource schema violation — only one can be present per rule.

---

## Why Revert, Not Forward-Fix

| Option | Time Required | Risk |
|---|---|---|
| Forward-fix | 15–25 minutes (diagnose, implement, review, apply) | Engineer under pressure — higher error probability |
| Revert | 5–8 minutes | Low — undoes a known commit with known state |
| Direct AWS console | 2–3 minutes | Manual drift from Terraform state — requires reconciliation |

Decision principle: **During an active outage, minimize time-to-recovery, not elegance.** The revert is cleanest because it restores a known-good state that was previously verified by CI.

---

## Post-Incident: Fix Forward

After the outage, the team implemented the correct fix:

```bash
# Create a new branch from the reverted main
git checkout -b fix/INCIDENT-9923-rds-security-group-rules

# Implement the correct Terraform configuration
# Rule uses source_security_group_id (not cidr_blocks)
# Add terraform plan output to PR description

git add modules/networking/security-groups.tf
git commit -m "fix(networking): restore correct RDS security group rules [INCIDENT-9923]

Use source_security_group_id reference to application security group.
The previous change incorrectly used cidr_blocks which excluded ECS tasks.

Tested with:
  - terraform plan shows 2 changes
  - Manual connectivity test in staging: OK

Closes: INCIDENT-9923"
```

---

## Lessons Learned

1. **Terraform `apply` returning 0 does not mean the application works.** Infrastructure changes can be syntactically and plan-level correct while causing application-level failures. Add integration smoke tests after every production deployment.

2. **Security group rule type changes (SG reference → CIDR) are high-risk.** The semantic meaning changes completely. Flag these in PR review.

3. **The revert pattern requires understanding `-m 1` for merge commits.** Without the `-m 1` flag, `git revert` on a merge commit will fail. Document this in your team's incident runbook.

4. **Having the Terraform state in S3 with versioning enabled is not optional.** Without versioned state files, Option B (direct state rollback) is unavailable. The team confirmed S3 versioning was already enabled — this was the right default decision made before the incident.

---

## Best Practices

- Add post-deployment integration smoke tests that verify application-layer connectivity (not just `terraform apply` exit code)
- Flag security group rule type changes (SG reference ↔ CIDR) in PR review checklists
- Document the `git revert -m 1` pattern for merge commits in your incident runbook — engineers forget the `-m 1` flag under pressure
- Enable S3 versioning on all Terraform state buckets — treat state files as production data
- Define rollback SLO: if an incident is not resolved in N minutes, escalate to the rollback path automatically (do not keep diagnosing while production is down)
