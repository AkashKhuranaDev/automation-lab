# Playbook 03: Repository Migration Checklist

**Risk Level:** High  
**Time to Execute:** 2–8 hours (depends on repository size and history)  
**Affected Systems:** All developers' local clones, CI/CD pipelines, deployment automation, issue trackers

---

## Situation

You need to move a repository from one platform or organization to another. Examples:
- GitHub Organization A → GitHub Organization B
- Bitbucket → GitHub
- Self-hosted GitLab → GitHub Enterprise
- Monolith repository split into multiple repositories

This is not a simple `git clone` + `git push`. Proper migration preserves all history, tags, branches, and leaves team members with working local clones.

---

## Prerequisites

- [ ] Write access to the source repository
- [ ] Admin access to create a repository on the destination platform
- [ ] Scheduled maintenance window communicated to the team
- [ ] CI/CD pipeline credentials for the new repository (prepare in advance)
- [ ] `git filter-repo` installed if rewriting history during migration

---

## Phase 1: Prepare the Source Repository

### 1.1 Audit the repository

```bash
# Size check
git count-objects -vH

# Branch count
git branch -r | wc -l

# Tag count
git tag | wc -l

# Largest files in history (potential LFS candidates)
git rev-list --objects --all \
  | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
  | sort -k3 -rn \
  | head -20
```

### 1.2 Decide what to migrate

- All branches? Or only `main` + long-lived branches?
- All tags? Or only release tags?
- Full history? Or a shallow clone from a cutoff date?
- Binary files? Or convert to Git LFS during migration?

Document your decision.

### 1.3 Freeze the source repository

Communicate to the team: no pushes to `main` during the migration window.

Enable branch protection on the source to enforce this if possible.

---

## Phase 2: Clone the Source Repository (Mirror)

```bash
# A mirror clone includes ALL branches, tags, and refs — not just the default branch
git clone --mirror git@github.com:old-org/old-repo.git old-repo.git

cd old-repo.git
git log --oneline -5               # Verify history is complete
git branch -r | head -10           # Verify all branches are present
git tag | head -10                 # Verify tags are present
```

---

## Phase 3: (Optional) Rewrite History

Only perform this step if you need to:
- Remove large binary files
- Remove accidentally committed secrets
- Strip internal paths or sensitive metadata

```bash
# Install git filter-repo (required — do not use git filter-branch)
pip install git-filter-repo

# Example: remove files larger than 10MB from history
git filter-repo --strip-blobs-bigger-than 10M

# Example: remove a specific directory
git filter-repo --path secret-configs/ --invert-paths

# Example: replace a secret string everywhere in history
git filter-repo --replace-text <(echo "OLD_SECRET==>REDACTED")
```

**Warning:** After rewriting, all commit SHAs change. Coordinate with the team before proceeding.

---

## Phase 4: Create the Destination Repository

1. Create a new empty repository on the destination platform
2. **Do not** initialize with README or .gitignore — that creates a commit that will conflict
3. Note the new remote URL

---

## Phase 5: Push to Destination

```bash
cd old-repo.git                    # The mirror clone directory

# Set the new remote
git remote set-url origin git@github.com:new-org/new-repo.git

# Push everything: all branches, all tags
git push --mirror

# Verify the push
git ls-remote origin | head -20
```

---

## Phase 6: Update CI/CD Pipelines

For each pipeline (GitHub Actions, Jenkins, CircleCI, etc.):

```bash
# Update the repository URL in:
# - Workflow files (.github/workflows/*.yml)
# - Webhook configurations
# - Deploy keys
# - Service account permissions
```

**GitHub Actions specifically:**
- Re-create deploy keys: `ssh-keygen -t ed25519 -C "github-actions-deploy"`, add public key to destination repo's deploy keys, add private key as Actions secret
- Update `GITHUB_TOKEN` references if cross-repo
- Test at least one workflow run manually before cutting over

---

## Phase 7: Update Team Local Clones

After the push succeeds, send this to all team members:

```bash
# Update existing clone to point to the new remote
git remote set-url origin git@github.com:new-org/new-repo.git

# Fetch to confirm connection
git fetch origin

# Verify
git remote -v
```

If there are 20+ team members, create a migration script:

```bash
#!/bin/bash
# run-migration.sh
OLD_URL="git@github.com:old-org/old-repo.git"
NEW_URL="git@github.com:new-org/new-repo.git"

CURRENT=$(git remote get-url origin 2>/dev/null)
if [[ "$CURRENT" == "$OLD_URL" ]]; then
    git remote set-url origin "$NEW_URL"
    echo "Updated to new remote: $NEW_URL"
else
    echo "Skipping: remote is $CURRENT (not matching old URL)"
fi
```

---

## Phase 8: Verification

```bash
# On the destination repository
git log --oneline -10                # Check history
git branch -r | wc -l               # Branch count matches source
git tag | wc -l                      # Tag count matches source
git diff <sha-from-source> HEAD      # Should be empty for same commit

# Trigger a CI run and confirm it passes on the new platform
```

---

## Phase 9: Deprecate the Source Repository

After 48 hours of successful operation on the new platform:

1. Archive the source repository (do not delete — references may still exist)
2. Add a notice to the source README: "**This repository has been moved to [new URL]**"
3. Configure GitHub redirect if moving between organizations (available in some plans)

---

## Rollback Strategy

If migration fails or CI is broken:

1. Un-archive the source repository
2. Remove the migration notice from its README
3. Notify team to `git remote set-url origin` back to the old URL
4. Investigate the failure before retrying

---

## Related

- [Case Study: Repository Migration](../case-studies/06-repository-migration.md)
- [Playbook 04: Clean Large Binary Files](04-clean-large-binary-files.md)

---

[← Playbook 02: Production Hotfix](02-production-hotfix.md) | [Playbook 04: Clean Large Binary Files →](04-clean-large-binary-files.md)
