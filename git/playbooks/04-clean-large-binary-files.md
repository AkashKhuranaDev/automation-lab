# Playbook 04: Clean Large Binary Files from History

**Risk Level:** High  
**Time to Execute:** 1–4 hours  
**Affected Systems:** All existing local clones become incompatible after history rewrite

---

## Situation

The repository has grown to an unacceptable size because large binary files were committed directly — build artifacts, compiled binaries, media assets, database dumps, or dependency archives. Common triggers:

- `git clone` takes more than 2 minutes
- Repository size exceeds 1GB
- CI checkout time exceeds 5 minutes
- GitHub shows "this repository is over the recommended size" warning

---

## Prerequisites

- [ ] All team members notified: **existing clones will become incompatible**
- [ ] Maintenance window scheduled
- [ ] `git filter-repo` installed: `pip install git-filter-repo`
- [ ] Repository fully backed up: `git clone --mirror <url> backup.git`
- [ ] Admin access to the remote repository

---

## Step 1: Diagnose

Identify what is taking up space:

```bash
# Total repository size
git count-objects -vH

# Largest objects in history
git rev-list --objects --all \
  | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
  | awk '{ if ($3 > 1048576) print int($3/1048576) "MB", $4 }' \
  | sort -rn \
  | head -30
```

Categorize the results:
- **Build artifacts** (`*.jar`, `*.war`, `*.zip`, `dist/`) → delete from history
- **Dependencies** (`node_modules/`, `vendor/`) → delete from history + add to `.gitignore`
- **Media assets** (`*.mp4`, `*.psd`, large `*.png`) → consider Git LFS
- **Database dumps** (`*.sql`, `*.dump`) → delete from history, store in S3

---

## Step 2: Back Up

```bash
cd ..
git clone --mirror git@github.com:org/repo.git repo.git.backup
echo "Backup size:"
du -sh repo.git.backup
```

Keep this backup until all team members have re-cloned.

---

## Step 3: Clean with git filter-repo

**Important:** `git filter-repo` requires a fresh, clean mirror clone. It will refuse to operate on a working clone with a remote configured.

```bash
git clone --mirror git@github.com:org/repo.git repo-clean.git
cd repo-clean.git

# Remove specific paths from all history
git filter-repo --path dist/ --invert-paths
git filter-repo --path '*.jar' --invert-paths
git filter-repo --path 'node_modules/' --invert-paths

# OR remove by file size (files > 10MB in history)
git filter-repo --strip-blobs-bigger-than 10M

# Verify the result
git count-objects -vH                    # Should be significantly smaller
git log --all --oneline | head -5        # History should still be present
```

---

## Step 4: Update `.gitignore`

Prevent the files from being re-added:

```bash
# In your working directory (not the mirror clone)
cat >> .gitignore << 'EOF'
# Build artifacts — do not commit
dist/
build/
*.jar
*.war
target/

# Dependencies — do not commit
node_modules/
vendor/

# Database dumps — store in S3 or encrypted blob storage
*.sql
*.dump
*.backup
EOF

git add .gitignore
git commit -m "chore: add build artifacts and dependencies to .gitignore"
```

---

## Step 5: Push the Rewritten History

```bash
cd repo-clean.git

# Force-push all branches and tags
git push --mirror
```

GitHub may block this if branch protection is enabled. Temporarily disable force push protection:

1. GitHub → Repository Settings → Branches → Edit rule
2. Uncheck "Require linear history" and related options temporarily
3. Push
4. Re-enable protection

---

## Step 6: Verify

```bash
# Remote size (GitHub API)
curl -s https://api.github.com/repos/ORG/REPO | python3 -c "import sys,json; d=json.load(sys.stdin); print(f'{d[\"size\"]/1024:.0f} MB')"

# Local clone size after clone
cd /tmp
git clone git@github.com:org/repo.git test-clone
du -sh test-clone/.git
rm -rf test-clone
```

---

## Step 7: Re-onboard Team

Send team members this command:

```bash
# Everyone must re-clone — existing clones cannot be rebased onto rewritten history
cd ~
git clone git@github.com:org/repo.git repo

# Recover any uncommitted local work first:
# git stash push -m "save before migration"
# After re-clone: git stash show -p > patch.diff  (apply manually)
```

Do **not** tell them to `git pull` — the history is incompatible and they will create a mess.

---

## Rollback Strategy

If the push succeeded but something is wrong:

```bash
# Push the backup mirror back to restore the original state
cd repo.git.backup
git remote set-url origin git@github.com:org/repo.git
git push --mirror                    # This restores the original history
```

---

## Alternative: Git LFS

If the large files are genuinely needed in the repository (design assets, ML model weights), consider Git LFS instead of deletion:

```bash
# Convert to LFS during migration
git lfs install
git lfs track "*.psd"
git lfs track "*.png" --lockable     # For files that need exclusive editing

git add .gitattributes
git commit -m "chore: track large assets with Git LFS"

# Migrate existing history to LFS
git lfs migrate import --include="*.psd" --everything
git push --force-with-lease
```

See [Lab 08: Git LFS](../labs/08-git-lfs.md) for a hands-on walkthrough.

---

## Related

- [Case Study: Large Repository Optimization](../case-studies/07-large-repository-optimization.md)
- [Production Incident P003](../production-incidents/P003-repository-grows-to-20gb.md)
- [Performance Reference](../performance/README.md)

---

[← Playbook 03: Repository Migration](03-repository-migration-checklist.md) | [Playbook 05: Recover Deleted Branch →](05-recover-deleted-branch.md)
