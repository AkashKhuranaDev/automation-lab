# Lab 06: Repository Cleanup

**Difficulty:** Advanced  
**Time:** 45 minutes  
**Skills:** `git filter-repo`, `git gc`, pack optimization, `.gitignore` enforcement

---

## Objective

Take a bloated repository (simulated with intentionally large files and bad commits), diagnose the problem, clean it using `git filter-repo`, and verify the size reduction.

---

## Prerequisites

```bash
# git filter-repo is not included with git — install it first
pip install git-filter-repo
# OR: brew install git-filter-repo (macOS)

git filter-repo --version           # Verify installation
```

---

## Setup: Create a Bloated Repository

```bash
cd ~/git-labs
mkdir lab-06-repo-cleanup && cd lab-06-repo-cleanup
git init

# Healthy application code
cat > app.py << 'EOF'
def main():
    print("Application started")

if __name__ == "__main__":
    main()
EOF
git add app.py && git commit -m "feat: initial application"

# Simulate an engineer who accidentally committed a large build artifact
dd if=/dev/urandom of=build-artifact.zip bs=1M count=5 2>/dev/null
git add build-artifact.zip && git commit -m "chore: temporary build output"

# More application commits
git commit --allow-empty -m "feat: add configuration parser"
git commit --allow-empty -m "fix: handle empty config"
git commit --allow-empty -m "test: add unit tests"

# Another engineer commits a secrets file
cat > .env.production << 'EOF'
DATABASE_URL=postgres://admin:P@$$w0rd123@prod-db.internal:5432/myapp
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=<YOUR_AWS_SECRET_KEY_HERE>
STRIPE_SECRET_KEY=<YOUR_STRIPE_SECRET_KEY_HERE>
EOF
git add .env.production && git commit -m "chore: add production environment template"

# More commits
git commit --allow-empty -m "feat: add health check endpoint"
git commit --allow-empty -m "chore: update dependencies"

# Someone removes the files but the history still has them
git rm build-artifact.zip .env.production
git commit -m "chore: remove accidentally committed files"

echo ""
echo "Current repository size:"
git count-objects -vH
echo ""
echo "Total commits: $(git log --oneline | wc -l | tr -d ' ')"
```

---

## Step 1: Diagnose the Problem

```bash
# Check total size
git count-objects -vH

# Find large objects in history
echo "=== Large objects in history ==="
git rev-list --objects --all \
  | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
  | awk '{ if ($3 > 100000) printf "%s MB\t%s\n", int($3/1024/1024), $4 }' \
  | sort -rn

# Find commits that contain secrets
echo ""
echo "=== Files with 'secret' or 'key' in name ==="
git log --all --diff-filter=A --name-only --format='' | sort -u | grep -iE 'secret|key|password|env|credential'
```

---

## Step 2: Back Up Before Cleaning

```bash
cd ~/git-labs
git clone --mirror lab-06-repo-cleanup lab-06-backup.git
echo "Backup size: $(du -sh lab-06-backup.git)"
```

---

## Step 3: Clean with git filter-repo

`git filter-repo` requires a fresh mirror clone to operate on. It refuses to work on a regular clone with a configured remote.

```bash
# Work in the original directory (which has no remote)
cd ~/git-labs/lab-06-repo-cleanup

# Remove the large binary file from all history
git filter-repo --path build-artifact.zip --invert-paths

echo "Size after removing build artifact:"
git count-objects -vH
```

```bash
# Remove the secrets file from all history
git filter-repo --path .env.production --invert-paths

echo "Size after removing secrets file:"
git count-objects -vH
```

```bash
# Alternatively, remove all files matching a pattern
# git filter-repo --path-glob '*.zip' --invert-paths
# git filter-repo --path-glob '.env.*' --invert-paths
```

---

## Step 4: Replace Secret Values in History

If you want to keep the file but scrub the secret values:

```bash
# Create a replacements file
cat > /tmp/replacements.txt << 'EOF'
wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY==>REDACTED_SECRET_KEY
AKIAIOSFODNN7EXAMPLE==>REDACTED_ACCESS_KEY_ID
<YOUR_STRIPE_SECRET_KEY_HERE>==>REDACTED_STRIPE_KEY
EOF

# Apply replacements (this modifies all blobs in history)
git filter-repo --replace-text /tmp/replacements.txt
rm /tmp/replacements.txt
```

---

## Step 5: Force Garbage Collection

After filter-repo, run gc to actually free the disk space:

```bash
git gc --aggressive --prune=now

echo ""
echo "=== Size comparison ==="
echo "Before cleanup:"
du -sh ~/git-labs/lab-06-backup.git
echo ""
echo "After cleanup:"
git count-objects -vH
```

---

## Step 6: Update .gitignore

Prevent the files from being re-added:

```bash
cat > .gitignore << 'EOF'
# Build artifacts
*.zip
*.jar
*.war
dist/
build/

# Environment and secrets files — never commit these
.env
.env.*
!.env.example
*.pem
*.key
credentials/
EOF

git add .gitignore
git commit -m "chore: add comprehensive .gitignore to prevent recurrence"
```

---

## Step 7: Verify

```bash
# Confirm removed files are gone from all history
git log --all --diff-filter=A --name-only --format='' | grep -E 'build-artifact|\.env\.production'
# Should return nothing

# Confirm application code is intact
git log --oneline
cat app.py                            # Should still exist

# Confirm size is reduced
git count-objects -vH
```

---

## Step 8: Notify Team

After cleaning history, all existing clones are incompatible. Send this message:

> "Repository history was rewritten to remove large files and secrets committed by accident. All team members must re-clone. Run: `cd ~ && git clone git@github.com:org/repo.git repo-fresh`"

---

## Key Concepts

1. **Removing a file in a commit does not remove it from history.** The file content is still in every pack file created before the removal commit. `git filter-repo` rewrites every commit to remove the object entirely.
2. **All SHAs change after filter-repo.** This is unavoidable. Every commit's SHA is a hash of its content plus its parent SHA. Changing any ancestor changes all descendants.
3. **Backup first.** `git clone --mirror` creates a full mirror that can be pushed back if anything goes wrong.
4. **`git filter-repo` over `git filter-branch`.** Filter-branch is deprecated, slow, and has security footguns. Always use filter-repo.
5. **Running gc is required.** Filter-repo rewrites the commits but the old object files remain until gc prunes them.

---

## Cleanup

```bash
cd ~/git-labs
rm -rf lab-06-repo-cleanup lab-06-backup.git
```

---

[← Lab 05: Cherry-Pick Backport](05-cherry-pick-backport.md) | [Lab 07: Git Worktree →](07-git-worktree.md)
