# Lab 08: Git LFS

**Difficulty:** Advanced  
**Time:** 45 minutes  
**Skills:** `git lfs track`, pointer files, LFS migration, `.gitattributes`

---

## Objective

Configure Git LFS for large binary assets, understand how LFS stores files, migrate existing large files into LFS, and verify that LFS pointers replace blobs in the object database.

---

## Prerequisites

```bash
# Install Git LFS
# macOS: brew install git-lfs
# Ubuntu: sudo apt install git-lfs
# All platforms: https://git-lfs.com

git lfs version                        # Verify: git-lfs/X.Y.Z ...
```

---

## Setup

```bash
cd ~/git-labs
mkdir lab-08-git-lfs && cd lab-08-git-lfs
git init
git lfs install                         # Install LFS hooks in this repository

echo "Git LFS status:"
git lfs env | head -5
```

---

## Part 1: Track File Types with LFS

```bash
# Tell LFS which file extensions to track
git lfs track "*.png"
git lfs track "*.jpg"
git lfs track "*.mp4"
git lfs track "*.psd"
git lfs track "*.zip"

# This creates/updates .gitattributes
cat .gitattributes
```

Output:
```
*.png filter=lfs diff=lfs merge=lfs -text
*.jpg filter=lfs diff=lfs merge=lfs -text
*.mp4 filter=lfs diff=lfs merge=lfs -text
*.psd filter=lfs diff=lfs merge=lfs -text
*.zip filter=lfs diff=lfs merge=lfs -text
```

```bash
# ALWAYS commit .gitattributes first
git add .gitattributes
git commit -m "chore: configure Git LFS tracking for binary assets"
```

---

## Part 2: Add Files to LFS

```bash
# Create a simulated large binary (5MB)
dd if=/dev/urandom of=design-mockup.png bs=1M count=5 2>/dev/null
dd if=/dev/urandom of=product-video.mp4 bs=1M count=10 2>/dev/null

# Add and commit — LFS intercepts automatically
git add design-mockup.png product-video.mp4
git status
# Both files should show as modified (tracked by LFS)

git commit -m "assets: add design mockup and product video"
```

---

## Part 3: Inspect LFS Pointers

LFS replaces the actual binary content with a tiny pointer file. The binary is stored in the LFS store, not in the Git object database.

```bash
# View the LFS pointer (what's stored in git, not the actual file)
git cat-file -p HEAD:design-mockup.png

# Output (example):
# version https://git-lfs.github.com/spec/v1
# oid sha256:4f5b6c7d...
# size 5242880

# The actual file on disk is the real binary (LFS downloads it transparently)
ls -lh design-mockup.png              # Shows 5MB

# LFS stored objects
git lfs ls-files
# Output:
# abc123 * design-mockup.png
# def456 * product-video.mp4

# Size breakdown
git lfs status
```

---

## Part 4: Understand the Size Difference

```bash
# Repository object size (tiny — only pointers)
git count-objects -vH

# LFS local cache size
du -sh $(git lfs env | grep LocalWorkingDir | cut -d= -f2)
```

The Git object database stores only the pointer files (a few hundred bytes each). The actual binary data is in the LFS store, separate from the object database. When you clone a repository with LFS:

```bash
# Clone with LFS (downloads binaries)
git clone --local . /tmp/lab-lfs-clone
ls -lh /tmp/lab-lfs-clone/*.png       # Full binary is available

# Clone without downloading LFS objects (faster, for CI that doesn't need binaries)
GIT_LFS_SKIP_SMUDGE=1 git clone --local . /tmp/lab-lfs-clone-no-lfs
ls -lh /tmp/lab-lfs-clone-no-lfs/*.png    # Shows pointer file (small)
cat /tmp/lab-lfs-clone-no-lfs/design-mockup.png  # Shows LFS pointer text
```

---

## Part 5: Migrate Existing Large Files to LFS

If a repository already has large files committed without LFS, you need to rewrite history to put them in LFS. This changes all commit SHAs.

```bash
# Create a repository with a pre-existing large file (simulating a repo you inherited)
cd ~/git-labs
mkdir lab-08-legacy && cd lab-08-legacy
git init

dd if=/dev/urandom of=legacy-archive.zip bs=1M count=8 2>/dev/null
git add legacy-archive.zip
git commit -m "chore: add legacy archive"

git commit --allow-empty -m "feat: feature A"
git commit --allow-empty -m "feat: feature B"

echo "Before migration:"
git count-objects -vH
```

```bash
# Install LFS and migrate
git lfs install

# Migrate the file from all history into LFS
git lfs migrate import --include="*.zip" --everything

echo "After migration:"
git count-objects -vH

# Verify the pointer is in place
git cat-file -p HEAD:legacy-archive.zip | head -5
# Should show LFS pointer format, not binary garbage

# Verify the file still works
ls -lh legacy-archive.zip             # Should show 8MB (LFS fetches it)
```

---

## Part 6: Locking Files (for non-mergeable binaries)

For files like Photoshop documents where merge conflicts are impossible to resolve:

```bash
cd ~/git-labs/lab-08-git-lfs

# Track with locking enabled
git lfs track "*.psd" --lockable
cat .gitattributes | grep psd

# Lock a file before editing (prevents others from editing simultaneously)
# Note: requires a remote — simulate the concept
# git lfs lock design-mockup.psd
# git lfs locks                       # See active locks
# git lfs unlock design-mockup.psd   # Release when done

# Without locking, the file is read-only on disk
ls -l design-mockup.png             # Note: may be read-only (-r--r--r--)
```

---

## Key Concepts

1. **LFS separates pointer from content.** The Git object database stores a text pointer (≈130 bytes). The binary is stored in the LFS backend. This is why clone size is small even for large-asset repositories.
2. **`.gitattributes` must be committed first.** If you commit the binary before `.gitattributes`, Git will store the binary in the object database, not LFS. The order matters.
3. **Migration is destructive.** `git lfs migrate import --everything` rewrites all commit SHAs. All existing clones become incompatible — the whole team must re-clone.
4. **`GIT_LFS_SKIP_SMUDGE=1`** is the standard CI optimization. CI pipelines that only run code tests don't need the binary files — skip LFS download to speed up checkout.
5. **LFS backends are separate from Git hosting.** GitHub, GitLab, and Bitbucket all provide LFS storage, but it is billed separately. Self-hosted alternatives include Gitea with LFS or a dedicated LFS server.

---

## Cleanup

```bash
cd ~/git-labs
rm -rf lab-08-git-lfs lab-08-legacy /tmp/lab-lfs-clone /tmp/lab-lfs-clone-no-lfs
```

---

[← Lab 07: Git Worktree](07-git-worktree.md) | [⌂ Labs Index](README.md)
