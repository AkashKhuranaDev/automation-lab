# CS-07 — 22 GB Repository Causing 40-Minute CI Clone Times

> **Category:** Performance, LFS | **Complexity:** High | **Time to Resolve:** 2 weeks (remediation)
>
> **Related:** [`performance/`](../performance/) | [`internals/`](../internals/) | [`best-practices/`](../best-practices/)

---

## Problem

A CI/CD pipeline for a machine learning infrastructure repository was taking 40 minutes per run, with 37 of those minutes spent on `git clone`. The repository had grown to 22 GB over 3 years as engineers committed model weights, training datasets, and compiled binaries directly into Git. Cache invalidation was frequent, making the 40-minute clone a near-constant cost.

---

## Environment

- **Repository:** GitHub Enterprise (private)
- **CI system:** Jenkins with ephemeral agent nodes
- **Repository size:** 22 GB (1.1 GB legitimate code + 20.9 GB binary blobs)
- **Clone time:** 37–42 minutes per job
- **Jobs per day:** ~200 (across multiple pipelines)
- **CI cost impact:** 200 jobs × 40 min = 133 hours/day of CI runner time wasted on clone

---

## Diagnosis

```bash
# Identify the largest objects in the repository
git filter-repo --analyze
# Review: .git/filter-repo/analysis/

# Top contributors by size
head -30 .git/filter-repo/analysis/blob-shas-and-paths.txt

# Sample output:
# 2147483648  models/bert-base-uncased.bin
# 1073741824  datasets/training-2022.tar.gz
# 536870912   models/gpt2-medium.bin
# 268435456   models/gpt2-small.bin
# ...

# Count objects by extension
cat .git/filter-repo/analysis/extensions.txt

# Extension  Count  Total size
# .bin       47     14.2 GB
# .tar.gz    12     4.1 GB
# .pkl       31     1.8 GB
# .pt        23     0.8 GB
# ...

# Timeline of size growth
git log --oneline --stat | grep "Bin" | awk '{print $1}' | sort | uniq -c
```

---

## Remediation Plan

### Phase 1: Identify and Migrate Binaries to Git LFS

```bash
# Install Git LFS
brew install git-lfs  # macOS
git lfs install

# Track large binary patterns
git lfs track "*.bin"
git lfs track "*.pt"
git lfs track "*.pkl"
git lfs track "*.tar.gz"
git lfs track "*.zip"

git add .gitattributes
git commit -m "chore: configure Git LFS tracking for binary assets"
```

### Phase 2: Migrate Historical Binaries from Git History

This is the most complex step — existing blobs in history must be migrated to LFS:

```bash
# Install git-filter-repo
pip install git-filter-repo

# Create a fresh clone for the migration
git clone --mirror git@github.com:org/ml-infra.git ml-infra-migration
cd ml-infra-migration

# Migrate all .bin, .pt, .pkl files from history to LFS
# (git-filter-repo with LFS migration support)
git lfs migrate import \
  --include="*.bin,*.pt,*.pkl,*.tar.gz,*.zip" \
  --everything \
  --yes

# Verify size reduction
git count-objects -vH
# size-pack: 1.08 GiB  (was 22 GB)

# Force push all branches and tags
git push --force --all
git push --force --tags
```

> **Warning:** All existing clones become invalid after this operation. Every engineer and CI system must re-clone.

### Phase 3: CI/CD Optimization

Even with LFS migration, cloning 1.1 GB of code for each CI job is unnecessary. Use shallow + sparse checkout:

```bash
# Jenkins pipeline — optimized checkout
checkout([
  $class: 'GitSCM',
  branches: [[name: env.BRANCH_NAME]],
  extensions: [
    [$class: 'CloneOption',
     shallow: true,
     depth: 1,
     noTags: true],
    [$class: 'SparseCheckoutPaths',
     sparseCheckoutPaths: [
       [$class: 'SparseCheckoutPath', path: 'src/'],
       [$class: 'SparseCheckoutPath', path: 'ci/'],
       [$class: 'SparseCheckoutPath', path: 'requirements.txt']
     ]]
  ],
  userRemoteConfigs: [[
    url: 'git@github.com:org/ml-infra.git',
    credentialsId: 'github-ssh-key'
  ]]
])
```

Alternative — `git` native approach in shell steps:

```bash
# Shallow clone with sparse checkout
git clone \
  --depth=1 \
  --filter=blob:none \
  --sparse \
  git@github.com:org/ml-infra.git

cd ml-infra
git sparse-checkout set src/ ci/ requirements.txt

# Skip LFS download for CI jobs that don't need model files
GIT_LFS_SKIP_SMUDGE=1 git clone --depth=1 git@github.com:org/ml-infra.git

# Then download only needed LFS files
cd ml-infra
git lfs pull --include="models/bert-base-uncased.bin"
```

---

## Results

| Metric | Before | After |
|---|---|---|
| Repository size | 22 GB | 1.1 GB (code) + LFS |
| CI clone time (full) | 37–42 minutes | 4.2 minutes |
| CI clone time (sparse, no LFS) | 37–42 minutes | 28 seconds |
| CI cost (daily, 200 jobs) | 133 hours runner time | 9.3 hours runner time |
| Developer clone time | 45 minutes | 4.2 minutes |

---

## `.gitignore` Added Post-Migration

```gitignore
# Model weights and datasets — use DVC or direct S3 references
*.bin
*.pt
*.pkl
models/*.ckpt
datasets/*.tar.gz

# Python build artifacts
__pycache__/
*.pyc
*.pyo
dist/
build/
*.egg-info/

# Jupyter checkpoints
.ipynb_checkpoints/

# Compiled extensions
*.so
*.dylib
```

---

## Tools Considered

| Tool | Purpose | Decision |
|---|---|---|
| **Git LFS** | Large file versioning within Git workflow | **Used** — integrates with existing GitHub workflow |
| **DVC** | ML dataset and model versioning (S3/GCS backed) | Evaluated — requires separate toolchain, deferred to Year 2 |
| **git filter-repo** | History rewrite to remove/migrate blobs | **Used** — replaced deprecated `git filter-branch` |
| **Artifactory** | Binary artifact repository | Already in use for packages, not connected to Git |

---

## Lessons Learned

1. **`git filter-repo --analyze` before any remediation.** Do not guess what is large. The analysis output shows extensions, top blobs by size, and their paths. This took 8 minutes and shaped the entire remediation plan.

2. **LFS migration invalidates all existing clones.** Communicate clearly, give engineers a 48-hour window to wrap up work, then execute. Do not attempt a rolling migration — partial states cause more problems than a clean cutover.

3. **`GIT_LFS_SKIP_SMUDGE=1` is the key optimization for CI jobs that don't need binaries.** Most CI pipeline stages (lint, unit test, deploy config) do not need the model weights. Only the model-training pipeline stage should pull LFS objects.

4. **`--filter=blob:none` (blobless clone) is different from LFS.** It defers downloading Git-stored blobs until they are accessed, reducing initial clone size. Use this for CI alongside LFS for maximum performance.

5. **The `.gitignore` additions prevented the problem from recurring.** Without adding the binary patterns to `.gitignore`, the repository would grow back to 22 GB within 18 months.

---

## Best Practices

- Add `.gitignore` rules for binary patterns on Day 1 of a new repository — do not allow binaries to accumulate and require expensive remediation later
- Set up a CI job that alerts if any commit adds a file larger than 10 MB — this is a canary for accidental binary commits
- Use `GIT_LFS_SKIP_SMUDGE=1` in all CI jobs by default; pull specific LFS files only when the job requires them
- Consider DVC for ML datasets if the team's workflow requires dataset versioning with remote storage — LFS costs scale with bandwidth
