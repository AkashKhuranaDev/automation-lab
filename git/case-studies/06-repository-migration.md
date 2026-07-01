# CS-06 — Migrating an 8-Year SVN Monolith to Git

> **Category:** Migration | **Complexity:** Very High | **Time to Resolve:** 6 weeks (planned migration)
>
> **Related:** [`internals/`](../internals/) | [`best-practices/`](../best-practices/) | [`enterprise-workflows/`](../enterprise-workflows/)

---

## Problem

A platform team was tasked with migrating a 400,000-commit SVN repository containing 8 years of infrastructure automation code to Git. The repository had 47 projects in a single SVN tree, multiple release branches, and contributors who had never used Git. The migration needed to preserve full commit history, maintain continuous development during the cutover, and complete within a 6-week window.

---

## Environment

- **Source:** Apache SVN, self-hosted
- **Repository size:** 400,000 commits, 18 GB of repository data
- **Projects:** 47 separate infrastructure projects in a monorepo structure
- **Active contributors:** 63 engineers across 8 teams
- **Target:** GitHub Enterprise (on-premises)
- **Git version required:** 2.35+ on all developer workstations

---

## Migration Architecture

```
SVN Repository (monolith)
└── trunk/
│   ├── project-a/
│   ├── project-b/
│   └── ... (47 projects)
└── branches/
│   ├── release/2022/
│   └── release/2023/
└── tags/

↓ git svn + git filter-repo

Git (2 strategies evaluated):
Option A: Single Git monorepo (preserves cross-project history)
Option B: 47 separate Git repositories (cleaner ownership)
```

Decision: **Option A (monorepo) with sparse checkout** — cross-project dependencies were too tightly coupled for immediate separation. Repository split deferred to Year 2.

---

## Week-by-Week Plan

### Week 1–2: Migration Infrastructure and Dry Run

```bash
# Install git-svn (may need to install separately on macOS)
brew install git-svn

# Initial clone with author mapping
# First: generate author map from SVN
svn log -q "svn+ssh://svn.company.com/infra" | \
  grep -E "^r[0-9]+" | \
  awk '{print $3}' | sort | uniq > svn-authors.txt

# Create authors.txt mapping file
# Format: svn_username = Full Name <email@company.com>
cat svn-authors.txt | while read user; do
  echo "$user = $user <$user@company.com>"
done > authors.txt

# Edit authors.txt manually to fill in real names and emails

# Clone from SVN with author mapping
git svn clone \
  svn+ssh://svn.company.com/infra \
  --authors-file=authors.txt \
  --no-metadata \
  --prefix=svn/ \
  --stdlayout \
  infra-migration

# This takes 12–24 hours for a large repository
```

### Week 3: History Cleanup

```bash
cd infra-migration

# Remove large binary files that accumulated over 8 years
pip install git-filter-repo

# Identify large files
git filter-repo --analyze
cat .git/filter-repo/analysis/blob-shas-and-paths.txt | sort -k1 -rn | head -30

# Remove files over 50MB that should not be in version control
git filter-repo --strip-blobs-bigger-than 50M

# Remove build artifacts that were accidentally committed
git filter-repo --path-glob '**/*.class' --invert-paths
git filter-repo --path-glob '**/target/' --invert-paths
git filter-repo --path-glob '**/*.zip' --invert-paths

# Result: 18 GB → 4.2 GB after cleanup
```

### Week 4: Staging Validation

```bash
# Push to GitHub Enterprise staging instance
git remote add origin https://github-enterprise.company.com/infra/automation.git
git push origin --all
git push origin --tags

# Validate:
# 1. Commit count matches (within tolerance — some SVN metadata commits dropped)
git log --oneline | wc -l

# 2. Recent commit history looks correct
git log --oneline -20

# 3. Tags are present
git tag -l | wc -l

# 4. All 47 project directories present
ls -d */ | wc -l
```

### Week 5: Engineer Training and Parallel Running

```bash
# Set up sparse checkout for teams working on specific projects
git clone --filter=blob:none --sparse \
  https://github-enterprise.company.com/infra/automation.git
cd automation

# Team A works on project-a and project-b only
git sparse-checkout set project-a/ project-b/ docs/

# Team B works on project-networking
git sparse-checkout set project-networking/ shared-modules/
```

Training sessions conducted:
- Git fundamentals for SVN users (2 hours)
- PR workflow vs SVN commit-to-trunk (1 hour)
- Conflict resolution in Git (1 hour)
- Company branching standards (30 minutes)

### Week 6: Cutover

```bash
# Freeze SVN on Friday at 17:00
# Take final SVN clone with any last commits

# Final sync from SVN
cd infra-migration
git svn fetch --all

# Push final state to GitHub Enterprise
git push origin main --force
git push origin --tags --force

# Update repository to mark as read-only in SVN
# (SVN admin task — revoke commit access)

# Point CI/CD systems to GitHub Enterprise
# (Update Jenkins job configurations, Ansible Tower SCM)

# Update developer documentation with new clone URLs
```

---

## SVN Concepts Mapped to Git

| SVN Concept | Git Equivalent | Notes |
|---|---|---|
| Revision number (r1234) | Commit SHA | SHA is not sequential — use `git log --grep` |
| `svn commit` | `git add` + `git commit` + `git push` | 3 separate steps in Git |
| `svn update` | `git pull` | |
| `svn switch` | `git checkout` | Switching branches |
| `svn copy` | `git branch` or `git tag` | Branches and tags are lightweight |
| `svn merge --reintegrate` | `git merge` or `git rebase` | Different semantics |
| Externals | `git submodule` or path copying | Avoid externals pattern in Git |
| Global revision as timestamp | Commit date + SHA | |

---

## Metrics: Before vs After

| Metric | SVN | Git |
|---|---|---|
| Full checkout time | 45 minutes | 8 minutes (sparse) / 22 minutes (full) |
| Branch creation | 2 minutes (server copy) | < 1 second (local pointer) |
| Code review process | Email + Trac tickets | GitHub PR with inline comments |
| CI trigger | Trac post-commit hook | GitHub Actions (PR + merge) |
| Repository size | 18 GB | 4.2 GB (after cleanup) |
| Merge conflict rate | N/A (trunk-only) | 4% of PRs had conflicts (acceptable) |

---

## Lessons Learned

1. **Author mapping is the most time-consuming preparation step.** SVN usernames (often Windows domain names like `CORP\jsmith`) do not map cleanly to Git identity. Budget a full week to collect real names and email addresses from 63 engineers.

2. **The `--no-metadata` flag removes SVN revision numbers from commit messages.** This is usually desired (cleaner history) but means you cannot look up SVN revision `r94231` in Git. Maintain a mapping file if traceability is required.

3. **Large binary file cleanup is critical before migration.** Going from 18 GB to 4.2 GB was the single most impactful quality improvement. Engineers who built over 8 years accumulated `.war` files, database dumps, and compiled binaries in version control. Clean this before migration, not after.

4. **Parallel running (Week 5) exposed more problems than testing.** When engineers actually used the system with real work, they discovered 14 issues that testing had not caught — mostly around CI/CD webhook configuration and Windows line ending (`CRLF`) differences between SVN and Git.

5. **Sparse checkout was non-negotiable for a 4.2 GB repository.** Without it, developer workstation clone times were 22 minutes. With sparse checkout, team-specific clones took under 3 minutes.

---

## Best Practices

- Start with a dry run migration 4 weeks before the real migration — plan for it to fail and teach you things
- Generate author mappings from `svn log` and validate them with HR records
- Run `git filter-repo --analyze` before migration to understand what you are importing
- Train engineers using the actual migrated repository, not exercises — real URLs, real PRs, real CI
- Maintain SVN in read-only mode for 90 days post-migration (do not delete it) — engineers will need to look up old revisions
