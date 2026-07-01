# CS-08 — Migrating a Team from GitFlow to Trunk-Based Development

> **Category:** Strategy Change | **Complexity:** Medium | **Time to Resolve:** 8-week migration
>
> **Related:** [`branching/`](../branching/) | [`decision-guides/branching-strategy`](../decision-guides/branching-strategy.md) | [`enterprise-workflows/`](../enterprise-workflows/)

---

## Problem

An infrastructure team of 22 engineers was using GitFlow with 3-week release cycles. Long-lived feature branches regularly diverged 200–400 commits from `develop`, making merges painful (4–8 hours each). CI pipeline coverage was low on feature branches. Production defect rate was high because integration testing only happened at release time.

---

## Environment

- **Team:** 22 infrastructure engineers, 4 teams
- **Previous model:** GitFlow — `main`, `develop`, `feature/*`, `release/*`, `hotfix/*`
- **Release cadence:** Every 3 weeks
- **Target model:** Trunk-Based Development (TBD) — all commits to `main`, feature flags for incomplete work
- **CI system:** GitHub Actions
- **Feature flag system:** LaunchDarkly (pre-existing, not previously used for infrastructure)

---

## Why GitFlow Was Failing

The team's post-mortem on 6 months of data revealed:

| Metric | Value |
|---|---|
| Average feature branch lifespan | 18 days |
| Average merge conflict time | 5.4 hours |
| Integration defects caught before production | 34% |
| Integration defects caught in production | 66% |
| Time spent on merge coordination | ~15% of engineering time |

Root cause: 3-week branches accumulate enough divergence that merging becomes a separate engineering project.

---

## Migration Plan

### Phase 1 (Weeks 1–2): Infrastructure Preparation

```bash
# Set up branch protection on main for TBD
# GitHub → Settings → Branches → main:
# ✅ Require status checks (CI must pass)
# ✅ Require PR reviews (minimum 1)
# ✅ Require branches to be up to date
# ✅ Do not allow bypassing (including admins)

# Target: PRs open < 48 hours, branches live < 3 days

# Set up feature flag infrastructure
# LaunchDarkly SDK already in use — add infrastructure feature flag context
# Example: LD_FEATURE_NEW_NETWORKING_STACK=false
```

```yaml
# GitHub Actions — CI on every PR (did not exist before)
name: CI
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Terraform validate
        run: terraform validate ./
      - name: Terraform plan (non-destructive)
        run: terraform plan -lock=false
      - name: tflint
        run: tflint --recursive
      - name: tfsec
        run: tfsec .
```

### Phase 2 (Weeks 3–4): Pilot Team

Four engineers on the networking team ran TBD for 2 weeks while others continued GitFlow. Key practices:

```bash
# Short-lived branch workflow
git checkout -b feat/VPC-441-add-nat-gateway-outputs
# Work for 1–2 days max
git push origin feat/VPC-441-add-nat-gateway-outputs
# Open PR
# Get review (same day)
# Merge to main
# Delete branch immediately after merge

# Feature flag for incomplete changes
# In Terraform:
variable "enable_new_vpc_flow_logs" {
  type    = bool
  default = false  # Off by default — code is merged but inactive
}

# In main.tf
resource "aws_flow_log" "new_vpc_logs" {
  count  = var.enable_new_vpc_flow_logs ? 1 : 0
  # ...
}
```

### Phase 3 (Weeks 5–6): Full Team Migration

```bash
# Close all long-lived feature branches (critical step)
# List branches older than 7 days
git for-each-ref refs/remotes/origin \
  --format='%(refname:short) %(creatordate:relative)' | \
  awk '$2 >= 7 && $3 == "days" {print $1}'

# For each long-lived branch, team meeting to decide:
# A. Split into < 3-day chunks and merge incrementally
# B. Add feature flag and merge incomplete work
# C. Defer the feature entirely

# 14 branches were assessed:
# - 6 merged in chunks (weeks 5-6)
# - 5 wrapped in feature flags and merged
# - 3 deferred (non-critical, team agreed to drop)
```

### Phase 4 (Weeks 7–8): Establish Norms

Team norms documented and added to CONTRIBUTING.md:

```markdown
## Branch Norms

- Maximum branch lifespan: 3 days. If your branch will take longer,
  split the work or use a feature flag.
- Branch naming: `feat/TICKET-123-short-description`
  or `fix/TICKET-456-short-description`
- PR size target: < 400 lines changed. If larger, split into multiple PRs.
- PR review SLO: Respond to review requests within 4 business hours.
- Feature flags: Incomplete Terraform resources should default to `count = 0`
  or use a boolean variable gated by environment.
- No force pushes to main.
- No direct commits to main (requires PR).
```

---

## Results After 8 Weeks

| Metric | GitFlow (Before) | TBD (After) |
|---|---|---|
| Average branch lifespan | 18 days | 1.8 days |
| Merge conflict rate | 71% of merges | 8% of merges |
| Average conflict resolution time | 5.4 hours | 22 minutes |
| Integration defects caught in CI | 34% | 78% |
| Releases per week | 0.33 (every 3 weeks) | 4.2 (daily possible) |
| Time on merge coordination | ~15% of time | ~2% of time |

---

## What Did Not Work

1. **Skipping the pilot phase.** The first attempt was a big-bang migration. 14 engineers switching simultaneously caused chaos in week 1. Restarted with a 4-person pilot.

2. **Merging long-lived branches without feature flags.** Three branches with incomplete work were merged "to get them off the branch" without feature flags. This caused partial features to appear in production configuration. Required 3 rollback commits.

3. **PR size limits without tooling.** The 400-line limit was frequently violated in the first 3 weeks. Added a GitHub Actions check that comments a warning (not a block) on PRs exceeding 600 lines.

---

## Lessons Learned

1. **The merge conflict rate drops immediately.** The most common complaint during GitFlow was merge pain. Within 2 weeks of the pilot team switching to TBD, they reported the 8% conflict rate — confirming the hypothesis.

2. **Feature flags require design discipline.** Adding a `count = 0` default to a Terraform resource is simple. But engineers need to be disciplined about removing the flag after the feature is complete. A follow-up issue should always be opened for "remove feature flag after X date."

3. **3-week release cycles were a symptom, not a cause.** The team was releasing every 3 weeks because merges were painful — they batch-fixed everything in a big release. Once merges became trivial, daily deployment became natural.

4. **The pilot team becomes the internal advocates.** The 4 engineers who ran TBD for 2 weeks became evangelists. Their firsthand experience was more persuasive than any management directive or process document.

---

## Best Practices

- Run a 2-week pilot with a small team before migrating everyone — pick engineers who are receptive to process change
- Close or migrate all long-lived branches before the full team switch — do not allow GitFlow branches to coexist with TBD for more than 2 weeks
- Feature flags are not optional for TBD in infrastructure — incomplete infrastructure code merged to main will apply partially, not just sit dormant
- Track and share the before/after metrics with the team — data is more persuasive than arguments
