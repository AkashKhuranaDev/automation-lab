# CS-02 — Production Fix Branch Deleted Before Merge

> **Category:** Recovery, Branching | **Complexity:** Medium | **Time to Resolve:** 18 minutes
>
> **Related:** [`recovery/`](../recovery/) | [`branching/`](../branching/)

---

## Problem

An engineer deleted a feature branch containing a critical fix after mistakenly believing it had been merged. The branch had been approved but was waiting for a scheduled deployment window. The branch name was `fix/k8s-liveness-probe-timeout` and it had been open for 4 days.

---

## Environment

- **Repository:** Private GitHub — Kubernetes deployment manifests
- **Team:** 8 engineers
- **Branch lifespan:** 4 days (pending deployment window)
- **Commits on branch:** 3 (the fix + two review-feedback iterations)
- **Detection:** Senior engineer noticed the open PR was now pointing to a deleted branch

---

## Timeline

| Time | Event |
|---|---|
| 14:02 | Engineer runs `git branch -d` on local branch and `git push origin --delete fix/k8s-liveness-probe-timeout` |
| 14:08 | Senior engineer opens the PR to check deployment status — "Branch was deleted" warning appears |
| 14:10 | Engineer who deleted the branch confirms it was intentional — believed it was merged |
| 14:11 | Recovery started using reflog on the engineer's local machine |
| 14:22 | Branch recreated and pushed — PR restored |
| 14:29 | Incident closed, deployment window maintained |

---

## Root Cause

The engineer confused this PR with a different one that had just been merged. Both had similar names and were by the same author. GitHub's branch deletion prompt does not warn that a PR is still open.

---

## Recovery: Local Reflog (Fast Path)

The engineer who deleted the branch still had the commits in their local reflog:

```bash
# On the engineer's local machine
git reflog show --all | grep "fix/k8s-liveness-probe-timeout"
# a3f7b21 refs/heads/fix/k8s-liveness-probe-timeout@{0}: commit: fix(k8s): increase liveness probe timeout

# The SHA a3f7b21 is the tip of the deleted branch

# Recreate the branch from the reflog SHA
git checkout -b fix/k8s-liveness-probe-timeout a3f7b21

# Verify the commits
git log --oneline
# a3f7b21 fix(k8s): increase liveness probe timeout
# 8b2c4d0 fix(k8s): address review feedback — add initialDelaySeconds
# 5f1e3a2 fix(k8s): initial liveness probe timeout fix

# Push the recreated branch
git push origin fix/k8s-liveness-probe-timeout
```

GitHub automatically re-attaches the open PR to the restored branch.

---

## Recovery: GitHub API (If No Local Clone Available)

GitHub retains the commits for at least 90 days after branch deletion. They are accessible via the API:

```bash
# Get the SHA of the deleted branch tip from the closed PR events
# GitHub API: GET /repos/owner/repo/events
curl -H "Authorization: token $GITHUB_TOKEN" \
  "https://api.github.com/repos/org/repo/pulls/247/commits" | \
  jq '.[-1].sha'
# "a3f7b21..."

# Recreate using the SHA
git fetch origin a3f7b21
git checkout -b fix/k8s-liveness-probe-timeout FETCH_HEAD
git push origin fix/k8s-liveness-probe-timeout
```

---

## Recovery: GitHub Web UI

For branches deleted via the GitHub UI:

1. Go to the repository on GitHub
2. Click on the **Pull requests** tab
3. Open the PR with the deleted branch
4. A banner appears: "The branch was deleted. Restore branch?"
5. Click **Restore branch**

This option is available for up to 90 days after deletion.

---

## Prevention

```bash
# Add to team's git aliases — shows branches with open PRs before delete
git config --global alias.safe-delete '!f() { 
  echo "Checking for open PRs..."; 
  gh pr list --head "$1" --state open; 
  echo "Branch: $1"; 
  read -p "Delete? (y/N): " confirm; 
  [[ "$confirm" == "y" ]] && git branch -d "$1"; 
}; f'

# Use the alias
git safe-delete fix/k8s-liveness-probe-timeout
```

GitHub branch protection: enable **"Do not allow deletion of matching branches"** on `main`, `release/*`, and any long-lived branches.

---

## Lessons Learned

1. **GitHub retains commits from deleted branches.** Branches are lightweight pointers — deleting the pointer does not delete the commits. They are recoverable via reflog, GitHub API, or the restore UI for up to 90 days.

2. **The GitHub PR restore button exists and works.** Many engineers do not know this feature exists. Document it in your team's runbook before someone needs it under pressure.

3. **Similar branch names on the same author cause confusion.** A naming convention that includes ticket numbers (`fix/INC-223-k8s-liveness-probe`) would have made the two branches distinguishable at a glance.

---

## Best Practices

- Include ticket/issue numbers in branch names to make them uniquely identifiable
- Before deleting a branch, verify with `gh pr list --head <branch-name> --state open`
- Add a team-wide Git alias that checks for open PRs before allowing branch deletion
- Document branch recovery options in your team runbook — the GitHub restore-branch feature is the fastest path
