# Git Cherry-Pick — Selective Commit Application

## Overview

Cherry-pick applies one or more specific commits from one branch to another, creating new commits with the same changes but different SHAs. It is a surgical tool for situations where you need a specific change without bringing along everything else in a branch.

---

## Why Cherry-Pick Matters

| Use case | Why cherry-pick is the right tool |
|---|---|
| Applying a production hotfix to an older release branch | The release branch should not receive all of `main`, just the fix |
| Backporting a security patch | Same reason — only the security fix, nothing else |
| Salvaging one good commit from an abandoned branch | Merge brings everything; cherry-pick brings one commit |
| Moving a commit that was applied to the wrong branch | Applied accidentally to `develop`, needed on `main` |

---

## When to Use It

- Hotfix applied to `main` must also reach `release/2024-q3`
- Security patch needs to land on multiple long-lived release branches simultaneously
- A feature is split across two PRs but one commit needs to ship first
- Recovering specific work from an abandoned branch

## When NOT to Use It

- When the full branch history needs to be integrated — use merge instead
- As a habit to avoid dealing with merge strategy decisions — it creates history debt
- When the same commit is cherry-picked into multiple branches repeatedly — this creates duplicate history that is painful to reconcile later

---

## How Cherry-Pick Works Internally

Cherry-pick computes the diff introduced by a commit (comparing it to its parent) and applies that diff as a new commit on the current branch.

New commit has:
- Same author, message, and diff as the original
- A new SHA (because the parent is different)
- A new committer entry (you, at this moment)

---

## Practical Examples

### Cherry-pick a single commit

```bash
git checkout release/2024-q3
git cherry-pick abc1234
```

### Cherry-pick with a custom message

```bash
git cherry-pick abc1234 -e
# Opens editor — you can add backport reference
```

### Cherry-pick multiple commits

```bash
git cherry-pick abc1234 def5678 789abcd
# Applied in order
```

### Cherry-pick a range of commits

```bash
# Apply commits from abc1234 up to and including def5678
git cherry-pick abc1234..def5678
# Note: abc1234 itself is excluded

# Include abc1234:
git cherry-pick abc1234^..def5678
```

### Cherry-pick without committing (stage only)

```bash
git cherry-pick --no-commit abc1234
# Applies the diff to the staging area without creating a commit
# Useful when you want to combine multiple cherry-picks into one commit
```

---

## Expected Output

```bash
$ git cherry-pick abc1234
[release/2024-q3 3f8a2b1] fix(security): rotate compromised API key
 Date: Tue Jul 1 18:30:00 2025 +0000
 1 file changed, 2 insertions(+), 2 deletions(-)
```

### When a conflict occurs

```bash
$ git cherry-pick abc1234
error: could not apply abc1234... fix(security): rotate compromised API key
hint: After resolving the conflicts, mark them with
hint: "git add/rm <pathspec>", then run
hint: "git cherry-pick --continue"

# Edit conflicting files, then:
git add config/credentials.tf
git cherry-pick --continue

# Or abort entirely:
git cherry-pick --abort
```

---

## Hotfix Workflow — Full Example

Scenario: A critical security issue is fixed on `main` and must reach the current production release (`release/2024-q3`) within the hour.

```bash
# Identify the fix commit on main
git log main --oneline | grep "SEC-220"
# abc1234 fix(iam): remove wildcard from production role [SEC-220]

# Switch to the release branch
git checkout release/2024-q3
git pull origin release/2024-q3

# Apply the fix
git cherry-pick abc1234

# If the commit message needs a backport marker:
git cherry-pick abc1234 -e
# Add: Backport-of: abc1234 (main)
# Resolves: SEC-220

# Push
git push origin release/2024-q3

# Open a PR or notify the release manager, depending on your process
```

---

## Backport Convention

In teams that maintain multiple release branches, document cherry-picks clearly:

```
fix(iam): remove wildcard from production role [SEC-220]

Backport of: abc1234 from main
Release: 2024-q3
Resolves: SEC-220
Reviewed-by: security-team
```

This makes future `git log` searches fast and makes audit trails complete.

---

## Real Enterprise Use Cases

**Security Operations team**

CVE identified in a library used across 4 release branches. The fix commit from `main` is cherry-picked to each release branch in sequence. Each cherry-pick commit includes the CVE reference and a link to the security advisory.

**SRE hotfix process**

Production is on `release/2024-q3`. A memory leak fix is committed to `main` via standard PR. SRE on-call cherry-picks the commit to `release/2024-q3`, triggers the deployment pipeline, and closes the incident.

**Feature flagging split**

A large feature PR is broken into two: infrastructure commits that can ship now, and application logic commits blocked on a dependency. Cherry-pick the infrastructure commits to `main` directly. The rest stays in the feature branch.

---

## Common Mistakes

| Mistake | Consequence |
|---|---|
| Cherry-picking a merge commit without `-m` | Git doesn't know which parent to use for the diff |
| Cherry-picking to many branches manually | Creates duplicate history — automate with a script |
| Not noting cherry-picks in commit messages | Future engineers can't tell why the same fix appears in multiple branches |
| Cherry-picking instead of fixing a broken branch strategy | Masks the root cause |

### Handling merge commits

```bash
# -m 1 means "use the first parent as the mainline"
# For a standard PR merge commit, -m 1 is almost always correct
git cherry-pick -m 1 <merge-commit-sha>
```

---

## Best Practices

- Always note the source commit SHA in the cherry-pick commit message
- Use `--no-commit` when combining multiple cherry-picks into a single logical commit
- For recurring backport patterns, write a script instead of doing it manually
- After cherry-picking, run tests on the target branch before pushing
- Prefer cherry-pick for single commits; for multiple sequential commits, consider merging a dedicated backport branch

---

## Troubleshooting

### "Cherry-pick applied but the change doesn't look right"

The diff was applied but the context around it changed since the original commit. Review the resulting file carefully.

```bash
git show HEAD  # Review what was actually committed
git diff HEAD~1  # Compare to the state before cherry-pick
```

### "I cherry-picked the wrong commit"

```bash
# Before pushing:
git reset --hard HEAD~1

# After pushing (creates a revert commit):
git revert HEAD
```

### "Cherry-pick says 'nothing to commit'"

The change from that commit is already present on the current branch (from a previous merge or cherry-pick).

```bash
git cherry-pick --skip
```

---

## References

| Resource | URL |
|---|---|
| git cherry-pick | https://git-scm.com/docs/git-cherry-pick |
| Advanced Merging | https://git-scm.com/book/en/v2/Git-Tools-Advanced-Merging |
| Revert vs Cherry-pick | https://git-scm.com/docs/git-revert |
