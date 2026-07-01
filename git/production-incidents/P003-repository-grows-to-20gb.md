# P003 — Repository Grows to 20+ GB

> **Severity:** P2 | **Expected Recovery Time:** 2–4 hours
>
> **Related:** [`CS-07`](../case-studies/07-large-repository-optimization.md) | [`performance/`](../performance/) | [`internals/`](../internals/)

---

## Symptoms

- `git clone` takes more than 10 minutes
- CI pipeline times dominated by `git clone` step
- `du -sh .git/` returns > 5 GB
- Developer workstations report slow `git status`, `git log`, or `git pull`
- `git count-objects -vH` shows `size-pack` in the GiB range
- Storage quota alerts from GitHub

---

## Diagnosis

```bash
# Step 1: How large is the repository?
git count-objects -vH
# count: 2048
# size: 12.34 MiB
# in-pack: 3872954
# packs: 4
# size-pack: 19.42 GiB    ← problem is here
# prune-packable: 0
# garbage: 0

# Step 2: Analyze what is taking the space
pip install git-filter-repo
git filter-repo --analyze
# Creates .git/filter-repo/analysis/ directory

# Review top blobs by size
head -30 .git/filter-repo/analysis/blob-shas-and-paths.txt
# Format: size  sha1  path

# Review by extension
cat .git/filter-repo/analysis/extensions.txt
# Shows which file types consume the most space

# Step 3: Find when the growth happened
git log --oneline --stat | grep -E "Bin [0-9]+ ->" | head -20
# Shows binary file additions in commit history
```

---

## Recovery

### Option A: Remove Specific Files from History (Surgical)

Use this when a small number of large files are the primary cause.

```bash
# Work on a fresh mirror clone
git clone --mirror git@github.com:$ORG/$REPO.git repo-clean
cd repo-clean

# Remove specific files from all history
git filter-repo --path path/to/large-file.bin --invert-paths
git filter-repo --path datasets/training-2022.tar.gz --invert-paths

# Multiple files at once via a file listing
cat > paths-to-remove.txt << 'EOF'
models/bert-base.bin
models/gpt2-medium.bin
datasets/
EOF
git filter-repo --paths-from-file paths-to-remove.txt --invert-paths

# Verify size reduction
git count-objects -vH | grep size-pack

# Force push
git push --force --all
git push --force --tags
```

### Option B: Strip All Files Over a Size Threshold (Broad)

Use this when many files across many commits are the problem.

```bash
# Remove all blobs larger than 50 MB from history
git filter-repo --strip-blobs-bigger-than 50M

# Verify
git count-objects -vH | grep size-pack
```

### Option C: Migrate Large Files to Git LFS (Going Forward)

After removing large files from history, configure LFS to handle them in the future:

```bash
# In the working repository (not the mirror)
git lfs install
git lfs track "*.bin"
git lfs track "*.tar.gz"
git lfs track "*.pt"
git lfs track "*.pkl"
git add .gitattributes
git commit -m "chore: configure Git LFS tracking for binary assets"
```

---

## All-Engineer Re-Clone After History Rewrite

```
ACTION REQUIRED — All engineers must re-clone [REPO-NAME]

A repository size remediation was performed. Your local clone
contains old history that is incompatible with the updated remote.

  cd ..
  rm -rf [repo-directory]
  git clone git@github.com:[ORG]/[REPO].git

Estimated clone time with new history: ~2 minutes
```

---

## Verification

```bash
# Confirm size reduction
git count-objects -vH | grep size-pack
# Should now show MiB, not GiB

# Confirm specific large files are gone
git log --all --full-history -- models/bert-base.bin
# (empty output — good)

# Test clone time
time git clone --depth=1 git@github.com:$ORG/$REPO.git /tmp/clone-test
# Should complete in < 3 minutes

# Run git fsck to verify repository integrity
git fsck --full
# Should show no errors
```

---

## Prevention

```bash
# Add a CI check that fails if any file in a commit exceeds 10 MB
# .github/workflows/file-size-check.yml

name: File Size Check
on: [pull_request]
jobs:
  check-file-sizes:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2
      - name: Check for large files
        run: |
          git diff --name-only HEAD~1 HEAD | while read file; do
            if [ -f "$file" ]; then
              size=$(stat -c%s "$file" 2>/dev/null || stat -f%z "$file")
              if [ "$size" -gt 10485760 ]; then  # 10 MB
                echo "ERROR: $file is $(($size / 1048576)) MB — exceeds 10 MB limit"
                echo "Use Git LFS for large binary files."
                exit 1
              fi
            fi
          done
```

```gitignore
# .gitignore additions to prevent large file accumulation
*.bin
*.tar.gz
*.zip
*.pkl
*.pt
*.ckpt
build/
dist/
target/
*.class
__pycache__/
```
