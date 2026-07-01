# Lab 04: Git Bisect

**Difficulty:** Intermediate  
**Time:** 30 minutes  
**Skills:** `git bisect`, binary search, automated bisect with test scripts

---

## Objective

Use `git bisect` to find the exact commit that introduced a regression, first manually and then with an automated test script. Understand why bisect is faster than reading commits.

---

## Setup

```bash
cd ~/git-labs
mkdir lab-04-git-bisect && cd lab-04-git-bisect
git init

# Create a "working" baseline
cat > calculator.py << 'EOF'
def add(a, b):
    return a + b

def subtract(a, b):
    return a - b

def multiply(a, b):
    return a * b
EOF
git add calculator.py
git commit -m "feat: initial calculator implementation"

# Simulate 15 commits of normal development
# One of them introduces a bug in the multiply function

COMMITS=(
  "chore: update documentation"
  "feat: add logging to add function"
  "fix: handle float inputs in subtract"
  "chore: run formatter"
  "feat: add type hints to add"
  "refactor: extract validation logic"
  "chore: update .gitignore"
  "BUGGY_COMMIT"
  "chore: remove unused import"
  "fix: improve error message"
  "feat: add divide function"
  "chore: update README"
  "test: add edge case for negative numbers"
  "fix: handle zero inputs"
  "chore: final cleanup"
)

for msg in "${COMMITS[@]}"; do
  if [[ "$msg" == "BUGGY_COMMIT" ]]; then
    # This is the bug: multiply now returns wrong result
    cat > calculator.py << 'EOF'
def add(a, b):
    return a + b

def subtract(a, b):
    return a - b

def multiply(a, b):
    return a * b + 1   # Bug: off-by-one error introduced here
EOF
    git add calculator.py
    git commit -m "refactor: simplify multiply implementation"
  else
    git commit --allow-empty -m "$msg"
  fi
done

echo ""
echo "Repository created with $(git log --oneline | wc -l | tr -d ' ') commits"
echo "One commit introduced a bug in the multiply function"
echo ""
echo "History:"
git log --oneline
```

---

## Scenario

Your CI is reporting that `multiply(3, 4)` is returning `13` instead of `12`. You have 16 commits in the history and don't know which one broke it. Rather than reading each commit manually, you will use `git bisect` to find the culprit in at most 4 steps (log₂ of 16).

---

## Part 1: Manual Bisect

```bash
# Start bisect
git bisect start

# Mark the current state as bad (the bug exists here)
git bisect bad HEAD

# Mark a known-good state (the initial commit definitely didn't have the bug)
git bisect good HEAD~16
# OR use the SHA of the first commit:
# git bisect good $(git log --oneline | tail -1 | awk '{print $1}')
```

Git now checks out the middle commit. For each checkout, test the function and tell Git the result:

```bash
# Test the function at the current commit
python3 -c "import calculator; result = calculator.multiply(3, 4); print(f'multiply(3,4) = {result}'); exit(0 if result == 12 else 1)"

# If the result is 12 (correct):
git bisect good

# If the result is 13 (buggy):
git bisect bad
```

Repeat until Git reports: `<sha> is the first bad commit`

---

## Part 2: Automated Bisect

Automated bisect is faster and removes human error. Write a test script and let Git run it at each step.

```bash
# Create a test script
cat > /tmp/test_multiply.sh << 'EOF'
#!/bin/bash
python3 -c "
import sys
sys.path.insert(0, '.')
try:
    import calculator
    result = calculator.multiply(3, 4)
    sys.exit(0 if result == 12 else 1)
except Exception as e:
    print(f'Error: {e}')
    sys.exit(1)
"
EOF
chmod +x /tmp/test_multiply.sh

# Reset bisect from the manual run
git bisect reset

# Start automated bisect
git bisect start
git bisect bad HEAD
git bisect good HEAD~16

# Run the automated search
git bisect run /tmp/test_multiply.sh

# Git will test each midpoint and find the bad commit without any prompting
```

---

## Walkthrough

After bisect completes, you'll see output like:
```
<sha> is the first bad commit
commit <sha>
Author: ...
Date:   ...

    refactor: simplify multiply implementation
```

```bash
# View the exact change
git show $(git bisect terms | head -1 | awk '{print $NF}' 2>/dev/null) \
  || git show HEAD

# The diff will show the bug:
# -    return a * b
# +    return a * b + 1   # Bug: off-by-one error introduced here
```

---

## Part 3: Bisect with a Non-Obvious Bug

Practice bisecting when you can't easily test programmatically. Simulate a performance regression:

```bash
# Reset
git bisect reset

# Create a scenario where the issue is a config value, not a code error
git bisect start
git bisect bad HEAD

# You can skip commits where the test is inconclusive
git bisect skip HEAD~3            # This commit doesn't apply cleanly — skip it

# Git will continue bisecting around the skipped commit
# Result will be: "There are only 'skip'ped commits left to test"
# or: "The first bad commit could be any of: <list>"
```

---

## Verification

```bash
# After finding the bad commit
git bisect reset                   # Return to HEAD

# Verify you found the right commit
git log --oneline | grep "simplify multiply"

# Confirm the fix
sed -i.bak 's/return a \* b + 1/return a * b/' calculator.py && rm calculator.py.bak
python3 -c "import calculator; print(calculator.multiply(3,4))"   # Should print 12
```

---

## Key Concepts

1. **Binary search is O(log n).** 1000 commits = 10 steps. 1,000,000 commits = 20 steps. Always bisect before manually reading commit history.
2. **The test script must exit 0 for good, non-zero for bad.** This is standard Unix convention. Any non-zero exit code means "bad."
3. **`git bisect skip`** is for commits that are untestable (compilation errors, unrelated broken tests). Git works around skipped commits.
4. **`git bisect log`** shows the bisect session history — useful for documentation or restarting a session.
5. **`git bisect reset`** always returns you to your original branch. Run it when done.

---

## Cleanup

```bash
git bisect reset                   # Ensure we're off bisect mode
cd ~/git-labs
rm -rf lab-04-git-bisect
rm -f /tmp/test_multiply.sh
```

---

[← Lab 03: Interactive Rebase](03-interactive-rebase.md) | [Lab 05: Cherry-Pick Backport →](05-cherry-pick-backport.md)
