# Git Interview Guide

For Infrastructure Engineers, Platform Engineers, DevOps Engineers, and SREs.

This guide covers what interviewers actually ask — from basics expected of all engineers to architecture-level questions asked at senior/staff/principal levels. Each answer is written as you should speak it, not as a textbook definition.

---

## How to Use This Guide

- **Junior / Mid roles:** Master the Beginner and Intermediate sections
- **Senior roles:** Beginner through Advanced + all Scenario-Based questions
- **Staff / Principal / Architect roles:** Everything, with architecture and design trade-offs
- **15-minute interview prep:** Read the Quick Reference at the end

---

## Table of Contents

1. [Beginner Questions](#beginner-questions)
2. [Intermediate Questions](#intermediate-questions)
3. [Advanced Questions](#advanced-questions)
4. [Scenario-Based Questions](#scenario-based-questions)
5. [Architecture and Design Questions](#architecture-and-design-questions)
6. [Common Mistakes to Avoid](#common-mistakes-to-avoid)
7. [Quick Reference — 15-Minute Prep](#quick-reference)

---

## Beginner Questions

### Q1: What is the difference between `git fetch` and `git pull`?

**What they want to hear:** You understand the two-step nature of synchronization.

> `git fetch` downloads new commits, branches, and tags from the remote but does not touch your working tree or current branch. Your local state is completely unchanged. You can then inspect what arrived with `git log HEAD..origin/main` before deciding to merge.
>
> `git pull` is `git fetch` followed by `git merge` (or `git rebase` if configured). It modifies your current branch immediately.
>
> In practice, I prefer `git fetch` + `git merge --ff-only` in automated scripts because it fails explicitly rather than silently creating a merge commit when fast-forward is not possible.

---

### Q2: What is the difference between `git reset` and `git revert`?

**What they want to hear:** Safety awareness — reset rewrites history, revert does not.

> `git reset` moves the branch pointer to a different commit. With `--hard`, it also resets the working tree. This rewrites history — if the reset commits were already pushed, other people's clones will have diverged history and `git push` will be rejected.
>
> `git revert` creates a new commit that applies the inverse of an existing commit. It preserves the original commit in history. This is safe to push to shared branches.
>
> Rule: Use `git revert` on any commit that has been pushed. Use `git reset` only for local-only work.

---

### Q3: What is a detached HEAD state?

**What they want to hear:** You understand what HEAD actually is.

> HEAD is normally a pointer to a branch name (e.g., `HEAD → main`). A branch name, in turn, points to a commit. In detached HEAD state, HEAD points directly to a commit SHA instead of a branch name.
>
> You enter detached HEAD by checking out a tag, a specific SHA, or a remote branch directly: `git checkout v2.3.0` or `git checkout abc1234`.
>
> The danger is that any commits you make are unreachable once you checkout another branch — there is no branch label advancing with your work. If you want to keep commits made in detached HEAD, create a branch first: `git checkout -b my-branch`.

---

### Q4: Explain the three states of a file in Git.

**What they want to hear:** You know the staging area, not just "tracked vs untracked."

> Git tracks file state in three areas:
> 1. **Working tree:** The actual files on disk. Modifications here are not yet recorded.
> 2. **Staging area (index):** A pre-commit snapshot. `git add` promotes changes from working tree to staging. Only staged changes go into the next commit.
> 3. **Repository (object database):** Committed history. Once committed, the state is permanent unless you rewrite history.
>
> `git status` shows the differences between all three: working tree vs staging, and staging vs last commit.

---

### Q5: What is the difference between a merge commit and a fast-forward merge?

**What they want to hear:** You know when each happens and which is better for audit trails.

> A fast-forward merge happens when the current branch has no divergent commits — the target branch is directly ahead. Git simply moves the branch pointer forward. No new commit is created. `git log` shows a linear history.
>
> A merge commit is created when both branches have divergent commits. Git creates a new commit with two parents, recording that the histories were joined at this point.
>
> In enterprise settings, `--no-ff` (no fast-forward) is often enforced so that merges always create a merge commit, making PR merge points visible in `git log --graph` as clear integration points.

---

### Q6: What does `git stash` do, and when would you use it?

**What they want to hear:** Practical use case, not just a definition.

> `git stash push` saves your current working tree and staging area state to a stack and reverts both to HEAD. This is useful when you need to switch context quickly — for example, you're in the middle of a feature and you need to check out another branch to answer a question or apply a hotfix.
>
> `git stash pop` restores the most recent stash and removes it from the stack. `git stash apply` restores without removing.
>
> I prefer `git stash push -m "description"` over bare `git stash` because stash entries without descriptions become confusing when you have multiple of them. For anything that will be saved more than 30 minutes, I create a WIP branch instead — stash is for short interruptions only.

---

## Intermediate Questions

### Q7: Explain the difference between `git rebase` and `git merge`. When would you choose each?

**What they want to hear:** Clear understanding of trade-offs, not a dogmatic preference.

> Both `rebase` and `merge` integrate changes from one branch into another, but they do it differently:
>
> `git merge` creates a new merge commit that has two parents. The original history of both branches is preserved exactly. `git log --graph` shows the branches visually.
>
> `git rebase` replays the commits from the feature branch on top of the target branch. The result is a linear history with new SHAs for each replayed commit. The merge point is invisible in `git log`.
>
> I choose based on context:
> - **Rebase:** updating a personal feature branch to stay current with main, cleaning up local commits before opening a PR, squashing WIP commits before review
> - **Merge:** integrating a completed, reviewed feature PR into main, particularly when you want the integration point visible in history (GitFlow, release branches, hotfix merges)
>
> The key constraint: **never rebase a branch that has been pushed to a shared remote**, unless you own that branch exclusively.

---

### Q8: What is `git cherry-pick` and when would you use it in practice?

**What they want to hear:** Practical production use case, especially backporting.

> `git cherry-pick <sha>` applies the changes introduced by a specific commit onto the current branch as a new commit. The new commit has a different SHA but identical content changes.
>
> The primary production use case is **backporting**: a bug is fixed on `main`, and you need the same fix applied to `release/v2.x` without merging all of `main`. You cherry-pick the specific fix commit.
>
> Use `git cherry-pick -x <sha>` for backports — the `-x` flag appends `(cherry picked from commit <sha>)` to the commit message, providing traceability.
>
> Cherry-pick should be used sparingly. If you find yourself cherry-picking the same commits across many branches, it's often a sign that your branching strategy needs to be simplified.

---

### Q9: What is `git bisect` and how do you use it?

**What they want to hear:** You've actually used it, not just read about it.

> `git bisect` performs a binary search through commit history to find the commit that introduced a bug. Given a known-good commit and a known-bad commit, Git checks out the midpoint, you test it, report `git bisect good` or `git bisect bad`, and Git narrows the range. In $\log_2(n)$ steps, you find the exact bad commit.
>
> The automated form is more powerful: `git bisect run ./test.sh`. The test script exits 0 for good and non-zero for bad. Git runs the full bisect without human intervention.
>
> In practice, I use bisect when an engineer reports "it worked last week but not today." I find the last-known-good release tag and run bisect. It consistently narrows down 100+ commits to one in under 10 minutes.

---

### Q10: Explain how `git reflog` differs from `git log`.

**What they want to hear:** You understand the reflog as a safety net.

> `git log` shows commits reachable from the current branch. When a branch is deleted or a commit is reset, those commits disappear from `git log`.
>
> `git reflog` shows every position HEAD has been at on the local machine — including commits that are no longer reachable from any branch. It is maintained locally and not synced to the remote. The default expiry is 90 days for reachable entries and 30 days for unreachable.
>
> The reflog is the primary recovery tool for "I lost my commits after a reset." Find the SHA in `git reflog`, then `git checkout -b recovery-branch <sha>`.

---

### Q11: What is the difference between `git reset --soft`, `--mixed`, and `--hard`?

**What they want to hear:** You can explain the three trees model.

> `git reset` moves the branch pointer to a specified commit. The three flags control what happens to the staging area and working tree:
>
> - `--soft`: Move branch pointer only. Staging area and working tree are unchanged. The reverted commits' changes end up staged, ready to recommit. Use this for "uncommit but keep staged."
> - `--mixed` (default): Move branch pointer and reset staging area. Working tree is unchanged. Changes end up as unstaged modifications. Use this for "uncommit and unstage, but don't touch files."
> - `--hard`: Move branch pointer, reset staging area, and reset working tree. All changes in the reverted commits are discarded from disk. Use this only when you genuinely want to throw away the changes.
>
> `--hard` is the dangerous one. Everything is recoverable from the reflog, but running `git gc` after `--hard` can permanently destroy the data.

---

### Q12: How does branch protection work and why is it important?

**What they want to hear:** You've configured it in practice, not just "it protects branches."

> Branch protection rules on GitHub (or equivalents on GitLab/Bitbucket) define what actions are allowed on specific branches.
>
> Key settings I configure on `main` for any production repository:
> - **Require pull request before merging**: prevents direct push
> - **Require status checks**: CI must pass — specific job names listed, not just "any"
> - **Require conversation resolution**: no merge until all review comments are addressed
> - **Restrict who can push**: limits direct push to specific teams even in emergency
> - **Do not allow force pushes**: prevents history rewrite on the protected branch
> - **Require linear history**: enforces squash or rebase merge (no merge commits), useful for clean history
>
> I also add `CODEOWNERS` to require approval from specific teams for specific directories — e.g., platform team must approve any change to `.github/workflows/`.

---

## Advanced Questions

### Q13: Explain Git's object model. What are the four object types?

**What they want to hear:** Accurate technical description, not a high-level metaphor.

> Git stores everything as objects in the object database (`.git/objects/`). Each object is identified by its SHA-1 (or SHA-256 in newer repositories) hash. There are four types:
>
> **Blob**: Stores the contents of a file, with no filename or metadata. Two files with identical content share one blob object.
>
> **Tree**: Stores a directory listing — a list of (mode, type, SHA, name) tuples pointing to blobs (files) and other trees (subdirectories). Trees are snapshots of directory state.
>
> **Commit**: Stores the repository state at a point in time. Contains a pointer to a root tree, pointers to parent commits (one for regular commits, two for merge commits), author/committer metadata, and the commit message.
>
> **Tag**: An annotated tag object. Points to a commit with a tagger name, date, and message. (Lightweight tags are just refs — no object.)
>
> You can inspect any object with `git cat-file -p <sha>`. This model is why everything in Git is content-addressed — if the content doesn't change, the SHA doesn't change, and objects are automatically deduplicated.

---

### Q14: What are packfiles and why does Git use them?

**What they want to hear:** Understanding of compression and delta encoding.

> Initially, Git stores objects as loose files in `.git/objects/`. This works for small repositories but becomes inefficient at scale — millions of small files is slow for filesystems.
>
> Packfiles (`.git/objects/pack/`) bundle many objects into a single file, using delta compression: instead of storing each version of a file completely, the packfile stores one complete version and the deltas (differences) between versions. A 10MB file with 100 revisions might be stored as 10MB + 100 small deltas, rather than 1GB.
>
> Repacking happens automatically during `git gc` or explicitly with `git repack`. The `pack-*.idx` index file allows Git to find any object in the pack without reading the entire pack file.
>
> For large repositories, `git clone --depth=1` (shallow clone) and partial clone (`git clone --filter=blob:none`) reduce packfile download size by not fetching all objects.

---

### Q15: What is `git rerere` and when is it useful?

**What they want to hear:** You understand the specific problem it solves (not just "it re-uses resolutions").

> `rerere` stands for "reuse recorded resolution." When enabled (`git config rerere.enabled true`), Git records how you resolve a merge conflict. If the same conflict recurs — same diff content on both sides — Git automatically applies the recorded resolution.
>
> The practical use case is rebasing a long-lived feature branch. If `main` has moved significantly and your branch has 20 commits, rebasing means resolving conflicts at each conflicting commit. If the same file section conflicts repeatedly, `rerere` applies your resolution automatically starting from the second occurrence.
>
> It is also useful during integration branches: if you regularly merge multiple feature branches into a `develop` branch, `rerere` automates the conflict resolution for recurring conflicts.
>
> The recordings are in `.git/rr-cache/` and are not shared via the remote.

---

### Q16: How does `git filter-repo` work and when would you use it?

**What they want to hear:** You've used it in a real scenario.

> `git filter-repo` rewrites Git history by applying transformations to every commit. Because Git commits are content-addressed (each commit SHA is derived from its content and parent SHA), changing any commit changes all its descendants. After `filter-repo`, every commit in the affected history has a new SHA.
>
> Production use cases:
> - **Secret removal**: a credentials file was committed to a public repository. `git filter-repo --path .env.production --invert-paths` removes it from all history.
> - **Large file removal**: a 500MB build artifact was committed. `git filter-repo --strip-blobs-bigger-than 100M` removes all large blobs.
> - **Repository split**: extract a subdirectory into a standalone repository: `git filter-repo --subdirectory-filter src/payment-service/`
> - **Repository merge**: merge two repositories while preserving history: `git filter-repo --to-subdirectory-filter payment-service/`
>
> Because it rewrites history, all existing clones become incompatible. Treat it as a destructive operation: back up with `git clone --mirror` first, notify all team members to re-clone after.

---

### Q17: Explain commit signing. What is the difference between GPG and SSH signing?

**What they want to hear:** You've configured both and understand the key management difference.

> Commit signing cryptographically attests that a commit was authored by the holder of a specific private key. It prevents commit author spoofing — without signing, `git commit --author="Linus Torvalds <torvalds@linux-foundation.org>"` would show a false author.
>
> **GPG signing**: `git config gpg.program gpg && git config commit.gpgsign true`. Requires a GPG key pair. The signing key is referenced by key ID. Verification requires the signer's public key in a keyring.
>
> **SSH signing**: `git config gpg.format ssh && git config user.signingkey /path/to/id_ed25519.pub`. Uses an SSH key pair that most engineers already have. The public key file path is specified (not a key ID — this is a common mistake). Verification requires an `allowed_signers` file.
>
> GitHub supports both and marks signed commits with a "Verified" badge. SSH signing is easier to set up for most engineers because it reuses existing SSH infrastructure. GPG signing provides a more mature ecosystem for key distribution via keyservers.

---

## Scenario-Based Questions

### S1: "You force-pushed over `main` by accident. What do you do?"

> First, stop and notify the team — no one should push until this is resolved.
>
> Then find the pre-force-push state. If another team member had fetched recently, their `origin/main` ref still points to the old history. Ask everyone to run `git reflog show origin/main` and share the SHA they see at `HEAD@{1}` (the previous position before your force push overwrote their ref).
>
> With the old SHA, restore it: `git push origin <old-sha>:main --force-with-lease`. This restores the remote to the pre-incident state.
>
> After recovery, fix the root cause: enable branch protection with "Require pull request before merging" which prevents all direct pushes. Aliases that force `--force-with-lease` help but don't address the root: the branch should not allow force pushes at all.

---

### S2: "A developer committed AWS credentials to a public repository 3 days ago. Walk me through the response."

> Immediate actions (within 5 minutes):
> 1. Rotate the credentials — invalidate the old ones in AWS IAM before removing them from Git. Assume they have already been harvested.
> 2. Check CloudTrail for unauthorized API calls using the compromised key
>
> Repository remediation (within the hour):
> 1. Make the repository private temporarily (Settings → Change visibility)
> 2. `git clone --mirror <repo>` for backup
> 3. `git filter-repo --path-glob '*.env' --invert-paths` to remove the file from history
> 4. `git push --mirror` to push the rewritten history back
> 5. GitHub secret scanning may have already flagged it — check the Security tab
>
> Post-incident:
> 1. Install `gitleaks` pre-commit hook on all team member machines
> 2. Enable GitHub secret scanning and push protection (prevents commits with known secret patterns)
> 3. Add `.env*` and `credentials/` to `.gitignore`
> 4. Write an incident report

---

### S3: "How would you set up Git for a team of 50 engineers working on a monorepo?"

> The challenges at 50 engineers on a monorepo are: clone time, CI time, merge conflicts due to high commit velocity, and CODEOWNERS routing.
>
> **Clone optimization:**
> - Partial clone: `git clone --filter=blob:none` — don't download file contents until needed
> - Sparse checkout: `git sparse-checkout set <paths>` — only check out the directories relevant to the engineer's team
>
> **CI optimization:**
> - Shallow clone in CI: `git clone --depth=1` — don't need full history for a build
> - Path-based CI triggers: only run tests for changed directories
>
> **Merge conflict reduction:**
> - Trunk-based development with short-lived feature branches (< 2 days)
> - CODEOWNERS for per-directory ownership and automatic review routing
> - Merge queue (GitHub's merge queue or Mergify) to serialize merges and prevent integration failures
>
> **Governance:**
> - Branch protection on `main` with required CODEOWNERS review
> - Separate CI check jobs per domain — payment team's CI doesn't block a docs change

---

### S4: "You need to move a repository from GitHub to GitLab without losing history, branches, or tags."

> The procedure uses a mirror clone, which captures everything:
>
> ```
> git clone --mirror git@github.com:org/repo.git repo.git
> cd repo.git
> git remote set-url origin git@gitlab.com:org/repo.git
> git push --mirror
> ```
>
> Before pushing, I audit the repository for any GitHub-specific Actions workflows that reference `actions/checkout`, GitHub Secrets, or GitHub Packages — these need to be replaced with GitLab CI equivalents.
>
> After pushing, I verify branch count and tag count match: `git ls-remote origin | wc -l` should be the same on both platforms.
>
> I communicate to team members the new remote URL. They run `git remote set-url origin git@gitlab.com:org/repo.git && git fetch origin` — they do not need to re-clone.

---

## Architecture and Design Questions

### A1: Design a branching strategy for an infrastructure team that deploys to 4 environments (dev, staging, canary, production).

> The key constraint for infrastructure teams is that code and infrastructure state must be kept in sync, and deployment is triggered by Git state. My design:
>
> **Long-lived branches:**
> - `main` → production
> - `canary` → canary environment (20% of production traffic)
> - `staging` → staging environment
> - `develop` → dev environment
>
> **Feature branches:** `feat/INFRA-NNN-description` branch off `develop`, merge back to `develop` via PR. Each merge to `develop` triggers a deploy to dev.
>
> **Promotion flow:** dev → staging via PR, staging → canary via PR, canary → main via PR after canary metrics confirm health. Each promotion is a git merge, creating a clear audit trail of when each change reached each environment.
>
> **Hotfix flow:** branch from `main`, apply fix, merge to `main` (deploys to production), then back-merge to `canary`, `staging`, and `develop` to keep them current.
>
> **Tagging:** every merge to `main` triggers a tag — `infra-v20240315-a3f2c11` format (date + short SHA) for traceability without semantic versioning overhead.

---

### A2: What is the trade-off between Trunk-Based Development and GitFlow for a team of 20 engineers shipping daily?

> For 20 engineers shipping daily, GitFlow adds overhead that does not pay off:
>
> GitFlow's `develop`, `release/X.Y`, `hotfix/` branch structure was designed for scheduled release cycles (weekly or monthly). With daily deployment, the release branch is almost never needed — by the time you cut a release branch, you're ready to deploy.
>
> TBD with short-lived feature branches (< 24 hours) and feature flags is a better fit for daily deployment because:
> - Fewer merge conflicts (branches stay short-lived)
> - Simpler mental model — `main` is always deployable
> - No `develop` branch to maintain and keep in sync
> - Hotfixes go directly to `main` (which is production anyway)
>
> The trade-off: TBD requires feature flags for any work that takes more than 1 day. If the engineering culture doesn't invest in feature flag infrastructure, TBD leads to long-lived branches anyway, which negates the model's advantages.
>
> See the full comparison: [`decision-guides/gitflow-vs-trunk-based.md`](../decision-guides/gitflow-vs-trunk-based.md)

---

### A3: How would you enforce commit message standards across an organization of 200 engineers?

> There are three enforcement points, and I recommend using all three for defense in depth:
>
> **Local (pre-commit hook + commitlint):**
> `commitlint` enforces Conventional Commits format (`feat:`, `fix:`, `docs:`, etc.) at commit time. Distribute via a shared `pre-commit` config in a `.pre-commit-config.yaml` checked into each repository. Engineers install it via `pre-commit install`.
>
> **CI (server-side validation):**
> A GitHub Actions workflow that runs `commitlint` on all commits in a PR. This catches commits that bypass the local hook (e.g., commits made via the GitHub UI or GitHub Codespaces without the hook installed).
>
> **Repository templates:**
> A GitHub repository template that includes the `.pre-commit-config.yaml`, `commitlint.config.js`, and a `CONTRIBUTING.md` explaining the standard. New repositories created from the template inherit all enforcement.
>
> The cultural challenge is as important as the tooling: publish the standard in the engineering wiki, include it in onboarding, and explain the *why* (automated changelogs, semantic versioning, searchable history).

---

## Common Mistakes to Avoid

| Mistake | Correct Approach |
|---------|-----------------|
| "I'd just Google it during the interview" | Practice the commands, not just the concepts |
| Saying "I know git pull, git commit, git push" | Demonstrate depth: reflog, bisect, filter-repo |
| Conflating `reset` and `revert` | Clear rule: `revert` for shared history, `reset` for local only |
| "I always use `--force`" | `--force-with-lease` is the correct default |
| Mentioning `git filter-branch` | It's deprecated; always say `git filter-repo` |
| Saying "merge is better than rebase" or vice versa | Context-dependent — demonstrate understanding of both |
| No mention of security when discussing commit signing | Connect signing to supply chain security and SLSA |
| Describing GitFlow without mentioning its downsides | Show you understand the trade-offs, not just the pattern |

---

## Quick Reference

**15-Minute Interview Prep**

```
fetch vs pull        → fetch downloads, pull downloads + merges
reset vs revert      → reset rewrites, revert creates new commit (safe for shared)
merge vs rebase      → merge preserves history, rebase linearizes it
HEAD~1 vs HEAD^1     → same for single parents; ^ disambiguates merge parents
reflog               → local history of every HEAD position (safety net)
cherry-pick -x       → backport with source traceability
bisect               → binary search for regressions
filter-repo          → history rewrite (not filter-branch)
--force-with-lease   → safe force push (fails if remote has changed)
4 object types       → blob, tree, commit, tag
packfiles            → delta-compressed object bundles
rerere               → automated conflict re-resolution
```

---

[⌂ Git Index](../README.md) | [Labs →](../labs/README.md) | [Playbooks →](../playbooks/README.md)
