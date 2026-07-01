# Lab 05: Cherry-Pick Backport

**Difficulty:** Intermediate  
**Time:** 25 minutes  
**Skills:** `git cherry-pick`, `-x` flag, conflict resolution, version branches

---

## Objective

Practice backporting a security fix from the main development branch to two older release branches using cherry-pick. Handle a conflict that arises because the code structure differs between versions.

---

## Setup

```bash
cd ~/git-labs
mkdir lab-05-cherry-pick-backport && cd lab-05-cherry-pick-backport
git init

# Simulate a codebase with three active versions
# v1.x: 2 years old, minimal dependencies
# v2.x: 1 year old, current stable
# main: development

cat > auth.py << 'EOF'
def authenticate(username, password):
    # Simple hash check
    return hash(password) == get_stored_hash(username)

def get_stored_hash(username):
    db = {"alice": hash("password123"), "bob": hash("secret")}
    return db.get(username, -1)
EOF

git add auth.py && git commit -m "initial: basic authentication"

# Create v1.x branch at this point
git checkout -b release/v1.x
git checkout main

# Add more features to main before branching v2.x
cat > auth.py << 'EOF'
import hashlib

def authenticate(username, password):
    hashed = hashlib.sha256(password.encode()).hexdigest()
    return hashed == get_stored_hash(username)

def get_stored_hash(username):
    db = {
        "alice": hashlib.sha256("password123".encode()).hexdigest(),
        "bob": hashlib.sha256("secret".encode()).hexdigest()
    }
    return db.get(username)

def logout(username):
    print(f"Logged out: {username}")
EOF

git add auth.py && git commit -m "feat: upgrade to SHA-256 hashing"

# Create v2.x branch
git checkout -b release/v2.x
git checkout main

# Add more development on main
cat > auth.py << 'EOF'
import hashlib
import hmac
import os

PEPPER = os.environ.get("AUTH_PEPPER", "default-pepper")

def authenticate(username, password):
    hashed = _hash_password(password)
    return hmac.compare_digest(hashed, get_stored_hash(username))

def _hash_password(password):
    return hashlib.sha256((password + PEPPER).encode()).hexdigest()

def get_stored_hash(username):
    db = {
        "alice": _hash_password("password123"),
        "bob": _hash_password("secret")
    }
    return db.get(username)

def logout(username):
    print(f"Logged out: {username}")

def change_password(username, old_password, new_password):
    if authenticate(username, old_password):
        print(f"Password changed for: {username}")
        return True
    return False
EOF

git add auth.py && git commit -m "feat: add HMAC comparison and pepper"
git commit --allow-empty -m "chore: update CI config"
git commit --allow-empty -m "feat: add rate limiting"

# Discover a security vulnerability in all versions: no timing-safe comparison
# Fix it on main first
cat > auth.py << 'EOF'
import hashlib
import hmac
import os

PEPPER = os.environ.get("AUTH_PEPPER", "default-pepper")

def authenticate(username, password):
    stored = get_stored_hash(username)
    if stored is None:
        # Dummy comparison to prevent timing-based username enumeration
        hmac.compare_digest("dummy", "value")
        return False
    hashed = _hash_password(password)
    return hmac.compare_digest(hashed, stored)

def _hash_password(password):
    return hashlib.sha256((password + PEPPER).encode()).hexdigest()

def get_stored_hash(username):
    db = {
        "alice": _hash_password("password123"),
        "bob": _hash_password("secret")
    }
    return db.get(username)

def logout(username):
    print(f"Logged out: {username}")

def change_password(username, old_password, new_password):
    if authenticate(username, old_password):
        print(f"Password changed for: {username}")
        return True
    return False
EOF

git add auth.py
git commit -m "fix(security): prevent timing-based username enumeration in authenticate

Without this fix, an attacker could determine valid usernames by measuring
response time: existing users trigger a hash comparison, non-existing users
return immediately. The fix adds a dummy comparison on the None path to
equalize response time regardless of whether the username exists.

CVE: CVE-2024-XXXX
Severity: Medium
Affected: v1.x, v2.x, main"

# Note the SHA of this security fix
FIX_SHA=$(git rev-parse HEAD)
echo ""
echo "Security fix committed at: $FIX_SHA"
echo "Now backport it to release/v2.x and release/v1.x"
```

---

## Your Task

Backport the security fix to both release branches. Do it yourself before reading the walkthrough.

**Challenge:** The fix uses `PEPPER` and `_hash_password` which don't exist in v1.x. You'll need to adapt it.

---

## Walkthrough

### Backport to release/v2.x (clean cherry-pick)

```bash
git checkout release/v2.x
git log --oneline -5                   # See what's on this branch

git cherry-pick -x $FIX_SHA
# -x: appends "(cherry picked from commit <sha>)" to the commit message
#     This is important for traceability when doing backports
```

The `-x` flag adds a reference to the original commit, making the provenance clear in git log:

```
fix(security): prevent timing-based username enumeration in authenticate
...
(cherry picked from commit abc1234)
```

```bash
# If it applies cleanly:
git log --oneline -3          # Verify the cherry-pick is there

# Verify the fix is correct
cat auth.py | grep "dummy"    # Should show the timing fix
```

### Backport to release/v1.x (conflict cherry-pick)

```bash
git checkout release/v1.x
git log --oneline -5          # v1.x has the old hash() implementation

git cherry-pick -x $FIX_SHA  # This will conflict
# CONFLICT (content): Merge conflict in auth.py
```

The conflict occurs because v1.x uses `hash()` and doesn't have `PEPPER` or `_hash_password`. The fix as written doesn't apply cleanly.

```bash
# View the conflict
cat auth.py
```

You need to adapt the fix to the v1.x code style:

```bash
# Resolve by writing an equivalent fix for v1.x's code structure
cat > auth.py << 'EOF'
def authenticate(username, password):
    stored = get_stored_hash(username)
    if stored is None:
        # Dummy comparison to prevent timing-based username enumeration
        _ = (hash("dummy") == 0)
        return False
    return hash(password) == stored

def get_stored_hash(username):
    db = {"alice": hash("password123"), "bob": hash("secret")}
    return db.get(username)
EOF

git add auth.py
git cherry-pick --continue
# The commit message will be pre-populated with the original + (cherry picked from...)
# Optionally append: "Adapted for v1.x (no HMAC/pepper): equivalent timing protection"
```

---

## Verification

```bash
# Check both release branches have the fix
git checkout release/v1.x
git log --oneline | grep "timing"

git checkout release/v2.x
git log --oneline | grep "timing"

# Verify the -x reference is present
git log -1 --format="%B" | grep "cherry picked"

# Verify main is unaffected (the cherry-pick doesn't modify source)
git checkout main
git log --oneline | head -5
```

---

## Key Concepts

1. **Cherry-pick creates a new SHA.** The commit content is identical but the parent is different, so the SHA changes. This is why `--force-with-lease` is always needed on release branches.
2. **`-x` is essential for backports.** Without it, there is no traceability between the release branch fix and the main branch fix.
3. **Conflicts require adaptation.** The v1.x example shows a real scenario: the fix logic is correct but the surrounding code structure is different. You must understand the intent and re-implement, not just force-apply.
4. **Cherry-pick is for single commits.** For a range of commits, use `git cherry-pick A..B` or consider `git rebase --onto`.
5. **`--abort`** always exits cleanly: `git cherry-pick --abort`

---

## Cleanup

```bash
cd ~/git-labs
rm -rf lab-05-cherry-pick-backport
```

---

[← Lab 04: Git Bisect](04-git-bisect.md) | [Lab 06: Repository Cleanup →](06-repository-cleanup.md)
