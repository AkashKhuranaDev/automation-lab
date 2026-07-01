# CS-09 — Monorepo Sparse Checkout Reduces CI from 18 Minutes to 90 Seconds

> **Category:** Performance, Sparse Checkout | **Complexity:** Medium | **Time to Resolve:** 3 days
>
> **Related:** [`performance/`](../performance/) | [`internals/`](../internals/) | [`enterprise-workflows/`](../enterprise-workflows/)

---

## Problem

A platform engineering team maintained a monorepo containing Terraform modules, Ansible playbooks, Helm charts, and Kubernetes manifests for 14 product teams. The monorepo approach was deliberate (shared modules, atomic cross-service changes), but CI pipelines for any individual team checked out the entire repository — 8.4 GB and 18+ minutes — even when they only needed 200 MB of their team's directory.

---

## Environment

- **Repository:** Private GitHub monorepo
- **Repository size:** 8.4 GB (including Git LFS objects for Helm chart archives)
- **Teams using the repo:** 14 product teams, each owning a subdirectory
- **CI system:** GitHub Actions (self-hosted runners on EKS)
- **Problem:** Every CI job regardless of which team triggered it ran a full clone

---

## Repository Structure

```
automation-monorepo/
├── teams/
│   ├── team-payments/
│   │   ├── terraform/
│   │   ├── helm/
│   │   └── kubernetes/
│   ├── team-auth/
│   ├── team-platform/
│   │   └── ... (shared modules used by all)
│   └── ... (14 teams total)
├── shared-modules/
│   ├── terraform/
│   └── ansible/
└── docs/
```

Each team's CI job needed:
- Their own directory (`teams/team-X/`)
- The shared modules (`shared-modules/`)
- Root config files (`.github/`, `Makefile`)

---

## Diagnosis

```bash
# Measure clone time for various strategies
time git clone git@github.com:org/automation-monorepo.git full-clone
# real 18m14s

time git clone --depth=1 git@github.com:org/automation-monorepo.git shallow-clone
# real 12m33s  (LFS objects still downloaded)

time GIT_LFS_SKIP_SMUDGE=1 git clone --depth=1 \
  git@github.com:org/automation-monorepo.git no-lfs-clone
# real 4m17s

time GIT_LFS_SKIP_SMUDGE=1 git clone --depth=1 --filter=blob:none --sparse \
  git@github.com:org/automation-monorepo.git sparse-clone
# real 28s
```

The combination of `--filter=blob:none` (blobless) + `--sparse` + `GIT_LFS_SKIP_SMUDGE=1` dropped clone from 18 minutes to 28 seconds.

---

## Solution: Per-Team Sparse Checkout in CI

```yaml
# .github/workflows/team-payments-ci.yml
name: Team Payments CI

on:
  push:
    paths:
      - 'teams/team-payments/**'
      - 'shared-modules/**'
  pull_request:
    paths:
      - 'teams/team-payments/**'
      - 'shared-modules/**'

jobs:
  validate:
    runs-on: [self-hosted, linux, x64]
    steps:
      - name: Sparse checkout for team-payments
        uses: actions/checkout@v4
        with:
          sparse-checkout: |
            teams/team-payments
            shared-modules
            Makefile
            .github
          sparse-checkout-cone-mode: true
          fetch-depth: 1
          lfs: false  # Download specific LFS files below if needed

      - name: Download specific LFS files (Helm charts only)
        run: |
          git lfs pull --include="teams/team-payments/helm/*.tgz"

      - name: Validate Terraform
        run: make validate-terraform TEAM=team-payments

      - name: Lint Helm charts
        run: make lint-helm TEAM=team-payments
```

---

## Path Filtering (CI Runs Only When Relevant Files Change)

```yaml
# This is the key efficiency unlock — CI for team-payments only runs
# when team-payments/ or shared-modules/ files change.
# Not every commit to the monorepo triggers every team's CI.

on:
  push:
    paths:
      - 'teams/team-payments/**'
      - 'shared-modules/**'
  pull_request:
    paths:
      - 'teams/team-payments/**'
      - 'shared-modules/**'
```

Teams that do not own the changed files do not pay the CI cost.

---

## Developer Workstation Sparse Checkout

```bash
# One-time setup for a developer on team-payments
git clone --filter=blob:none --sparse \
  git@github.com:org/automation-monorepo.git
cd automation-monorepo

# Set cone mode (faster pattern matching)
git sparse-checkout init --cone

# Check out only the files needed
git sparse-checkout set teams/team-payments shared-modules docs

# Download LFS objects for your team only
git lfs pull --include="teams/team-payments/**"

# Result: 200 MB checked out (of 8.4 GB total)

# Update sparse checkout if you need to work on shared-modules temporarily
git sparse-checkout add teams/team-platform
# Work on shared-modules
git sparse-checkout set teams/team-payments shared-modules docs
# Return to original checkout
```

---

## Results

| Operation | Before | After |
|---|---|---|
| Full clone (CI) | 18 min 14s | N/A (not done) |
| Sparse clone (CI) | N/A | 28 seconds |
| With LFS files for team | N/A | 90 seconds total |
| Developer workstation clone | 18 min | 45 seconds + team LFS pull |
| CI runs triggered unnecessarily | 100% (all teams) | 0% (only affected teams) |
| Daily CI runner hours (200 jobs) | 60 hours | 5 hours |

---

## Sparse Checkout Gotchas

```bash
# Gotcha 1: Cone mode requires directories — file patterns are not supported
# Wrong:
git sparse-checkout set 'teams/team-payments/*.tf'  # Does not work in cone mode

# Right (use non-cone mode for file patterns):
git sparse-checkout init  # no --cone
git sparse-checkout set 'teams/team-payments/'
echo 'teams/team-payments/*.tf' >> .git/info/sparse-checkout

# Gotcha 2: After sparse checkout, git status shows no missing files
# You cannot tell if a file exists but is excluded without checking
git ls-tree -r HEAD --name-only | grep "teams/team-auth"
# (no output if sparse-checked-out correctly)

# Gotcha 3: git merge and git rebase across sparse checkout boundaries
# If a PR touches files outside your sparse checkout, checkout the missing dirs first
git sparse-checkout add teams/team-auth
git merge origin/main
git sparse-checkout set teams/team-payments shared-modules docs
```

---

## Lessons Learned

1. **`--filter=blob:none` (blobless clone) is the most impactful single flag.** It defers downloading all non-tree Git objects until they are accessed. Combined with sparse checkout, files outside your cone are never downloaded.

2. **LFS and blobless clone interact differently.** `--filter=blob:none` affects regular Git objects. LFS objects are separate — you must also use `GIT_LFS_SKIP_SMUDGE=1` or `lfs: false` to skip them, then pull only what you need.

3. **Path filtering in CI is as important as sparse checkout.** Sparse checkout reduces what is downloaded. Path filtering reduces when CI runs at all. Together, they address both the cost and the noise from unrelated changes.

4. **Cone mode is faster but less flexible.** Cone mode uses directory prefixes (fast because Git can skip entire tree entries). Non-cone mode allows arbitrary patterns but is slower. Use cone mode unless you have specific per-file patterns.

5. **Sparse checkout does not affect `git log`.** You can still see the full commit history. It only affects the working tree. This means `git blame` on excluded files still works if you add their directory to sparse checkout temporarily.

---

## Best Practices

- Establish sparse checkout patterns for each team's working context on Day 1 of a new monorepo — do not wait until clone times become painful
- Use `actions/checkout@v4` with `sparse-checkout` parameter in GitHub Actions — the newer versions support this natively without manual setup
- Add path filters to CI workflows so only affected teams' pipelines run — unnecessary CI runs are a significant cost in large monorepos
- Document the team-specific `git sparse-checkout set` commands in each team's README — developers should not need to figure this out themselves
