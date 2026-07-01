# Lab 03: Interactive Rebase

**Difficulty:** Intermediate  
**Time:** 30 minutes  
**Skills:** `git rebase -i`, squash, fixup, reorder, drop, edit, split

---

## Objective

Clean up a messy commit history using interactive rebase. Transform a series of WIP commits into a professional, reviewable commit history.

---

## Setup

```bash
cd ~/git-labs
mkdir lab-03-interactive-rebase && cd lab-03-interactive-rebase
git init

git commit --allow-empty -m "initial commit"

# Simulate a developer working with messy, real-world commit habits
git checkout -b feat/user-profile

cat > user.py << 'EOF'
class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email
EOF
git add user.py && git commit -m "wip"

echo "    def get_name(self):" >> user.py
echo "        return self.name" >> user.py
git add user.py && git commit -m "wip cont"

echo "    def get_email(self):" >> user.py
echo "        return self.email" >> user.py
git add user.py && git commit -m "WIP add email getter"

cat > profile.py << 'EOF'
class Profile:
    def __init__(self, user, bio=""):
        self.user = user
        self.bio = bio
EOF
git add profile.py && git commit -m "add profile class maybe"

echo "    def update_bio(self, bio):" >> profile.py
echo "        self.bio = bio" >> profile.py
git add profile.py && git commit -m "forgot the update method"

echo "    def display(self):" >> profile.py
echo "        return f'{self.user.get_name()}: {self.bio}'" >> profile.py
git add profile.py && git commit -m "add display... fix"

cat > tests.py << 'EOF'
from user import User
from profile import Profile

def test_user_creation():
    u = User("Alice", "alice@example.com")
    assert u.get_name() == "Alice"

def test_profile_display():
    u = User("Bob", "bob@example.com")
    p = Profile(u, "Engineer")
    assert p.display() == "Bob: Engineer"
EOF
git add tests.py && git commit -m "add tests"
git commit --allow-empty -m "typo" # Oops, commit with typo in message

echo "" >> user.py
git add user.py && git commit -m "fix whitespace"
```

Your history now looks like this:
```
fix whitespace
typo
add tests
add display... fix
forgot the update method
add profile class maybe
WIP add email getter
wip cont
wip
initial commit
```

---

## Scenario

You are about to open a PR. The commit history is unprofessional and would clutter the main branch history. You need to clean it up into 3 logical commits:
1. `feat(user): add User class with name and email getters`
2. `feat(profile): add Profile class with bio and display`
3. `test: add unit tests for User and Profile`

---

## Walkthrough

### Start the interactive rebase

```bash
# Rebase the last 9 commits interactively
# Count your commits: git log --oneline | head -10
git rebase -i HEAD~9
```

This opens your editor with the rebase instruction file:

```
pick abc1234 wip
pick def5678 wip cont
pick ghi9012 WIP add email getter
pick jkl3456 add profile class maybe
pick mno7890 forgot the update method
pick pqr1234 add display... fix
pick stu5678 add tests
pick vwx9012 typo
pick yza3456 fix whitespace
```

### Edit the instructions

Change the file to:

```
reword abc1234 wip
fixup def5678 wip cont
fixup ghi9012 WIP add email getter
pick jkl3456 add profile class maybe
fixup mno7890 forgot the update method
fixup pqr1234 add display... fix
pick stu5678 add tests
fixup vwx9012 typo
fixup yza3456 fix whitespace
```

Commands used:
- `reword` — keep this commit but edit the message
- `fixup` — squash into the previous commit, discard this commit's message
- `pick` — keep as-is (we'll reword separately)

### Reword the commit messages

When the editor opens for `reword`:
- Change `wip` to: `feat(user): add User class with name and email getters`

When the editor opens for the profile commit:
- Change `add profile class maybe` to: `feat(profile): add Profile class with bio and display`

When the editor opens for the tests commit:
- Change `add tests` to: `test: add unit tests for User and Profile`

---

## Verification

```bash
# Check the resulting history
git log --oneline

# Should show exactly 3 commits:
# a1b2c3d (HEAD -> feat/user-profile) test: add unit tests for User and Profile
# e4f5g6h feat(profile): add Profile class with bio and display
# i7j8k9l feat(user): add User class with name and email getters

# Confirm the code still works (no accidental loss of changes)
cat user.py
cat profile.py
cat tests.py
```

---

## Bonus: Reorder Commits

```bash
# Practice reordering: what if you want tests to come before profile?
git rebase -i HEAD~3

# In the editor, swap the order:
# pick i7j8k9l feat(user): ...
# pick a1b2c3d test: add unit tests ...       ← moved up
# pick e4f5g6h feat(profile): ...             ← moved down

# Note: this may fail if test commits depend on profile commits
# Git will pause at the conflict and let you resolve
```

---

## Bonus: Split a Commit

If a single commit contains two logical changes:

```bash
# Start the rebase
git rebase -i HEAD~3

# Mark the commit you want to split with 'edit':
# edit i7j8k9l feat(user): add User class with name and email getters

# When Git pauses:
git reset HEAD~1                   # Unstage the commit but keep the changes
git add user.py                    # Stage only user class creation
git commit -m "feat(user): add User class"

git add user.py                    # Stage only the getters
git commit -m "feat(user): add name and email getters"

git rebase --continue
```

---

## Key Concepts

1. **Interactive rebase rewrites history.** After a rebase, all SHAs change. Never rebase commits that have been pushed to a shared branch.
2. **`fixup` vs `squash`:** `fixup` discards the commit message. `squash` combines the messages and asks you to edit the result. Use `fixup` for minor corrections, `squash` when you want to write a combined message.
3. **Order matters.** When reordering commits, Git applies them sequentially. If commit B depends on commit A, you cannot move B before A.
4. **`git rebase --abort`** at any point restores the original state. Use it freely if something goes wrong.
5. **`GIT_SEQUENCE_EDITOR`** can override the editor for the instruction file: `GIT_SEQUENCE_EDITOR="sed -i 's/^pick/fixup/'" git rebase -i HEAD~5`

---

## Cleanup

```bash
cd ~/git-labs
rm -rf lab-03-interactive-rebase
```

---

[← Lab 02: Resolve Merge Conflicts](02-resolve-merge-conflict.md) | [Lab 04: Git Bisect →](04-git-bisect.md)
