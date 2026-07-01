# Git Best Practices — Enterprise Standards for Production Teams

## Overview

Best practices in Git are not opinions — they are the difference between a codebase that engineers can trust and debug, and one that they fear touching. Every standard in this document comes from the pain of not following it.

---

## Commit Messages — The Most Underrated Engineering Standard

A commit message is engineering documentation written in the moment when context is highest. Six months later, the diff shows *what* changed. Only the message explains *why*.

### Conventional Commits format

```
<type>(<scope>): <short description>

[optional body]

[optional footer(s)]
```

| Type | When to use |
|---|---|
| `feat` | New feature or capability |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `refactor` | Code restructuring, no behavior change |
| `test` | Adding or updating tests |
| `chore` | Maintenance — deps, config, build |
| `ci` | Changes to CI/CD configuration |
| `perf` | Performance improvement |
| `revert` | Reverting a previous commit |
| `security` | Security-related change |

### Good vs bad commit messages

Bad:
```
fix stuff
update config
WIP
changes
```

Good:
```
fix(terraform): correct CIDR block overlap in prod VPC [INFRA-1098]

The prod VPC CIDR 10.0.0.0/16 was overlapping with the transit gateway
attachment subnet 10.0.255.0/24. Corrected to 10.0.0.0/20.

Resolves: INFRA-1098
Reported-by: network-team
Tested: plan + apply in staging, verified no route conflicts
```

### Rules

- Subject line: max 72 characters
- Imperative mood: "add feature", not "added feature"
- No period at the end of the subject
- Blank line between subject and body
- Body explains *why*, not *what* (the diff shows *what*)
- Footer references tickets: `Resolves: INFRA-1098`

---

## Atomic Commits

Each commit should represent one logical change that can be understood, reviewed, and reverted independently.

```bash
# Bad: one massive commit with unrelated changes
git add .
git commit -m "lots of stuff"

# Good: stage and commit by logical group
git add modules/vpc/
git commit -m "feat(vpc): add multi-AZ subnet configuration"

git add .github/workflows/
git commit -m "ci: add Terraform plan workflow for PRs"
```

Use `git add -p` to stage specific hunks from a file:

```bash
git add -p modules/vpc/main.tf
# Presents each changed hunk, y/n to stage
```

---

## .gitignore — What Should Never Be Committed

Create a `.gitignore` from the start. These should never reach version control:

```gitignore
# Terraform
.terraform/
*.tfstate
*.tfstate.backup
*.tfvars
.terraform.lock.hcl   # Commit this one — it locks provider versions

# Secrets
*.pem
*.key
*.env
.env*
secrets/
credentials

# OS
.DS_Store
Thumbs.db

# Editor
.vscode/
.idea/
*.swp
*.swo

# Python
__pycache__/
*.pyc
.venv/

# Ansible
*.retry
```

Use [gitignore.io](https://www.toptal.com/developers/gitignore) for language-specific templates.

### If a secret was already committed

```bash
# Remove the file from tracking
git rm --cached secrets/credentials.json
echo "secrets/" >> .gitignore
git commit -m "security: remove accidentally committed credentials"

# Rotate the secret immediately — assume it was already read
# If it was pushed, rotate before fixing the git history
```

---

## Security Practices

| Practice | Implementation |
|---|---|
| Never commit secrets | `.gitignore` + pre-commit hooks |
| Scan every commit for secrets | [`gitleaks`](https://github.com/gitleaks/gitleaks), [`detect-secrets`](https://github.com/Yelp/detect-secrets) |
| Sign commits | `git config commit.gpgsign true` |
| Sign tags | `git tag -s` |
| Require signed commits on protected branches | GitHub branch protection → require signed commits |
| Audit repository access | GitHub → Settings → Collaborators |

### Pre-commit hook for secret scanning

```bash
# .git/hooks/pre-commit
#!/bin/sh
gitleaks detect --staged --no-git
if [ $? -ne 0 ]; then
    echo "ERROR: Secrets detected in staged files. Commit aborted."
    exit 1
fi
```

```bash
chmod +x .git/hooks/pre-commit
```

For team-wide enforcement, use [pre-commit framework](https://pre-commit.com):

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
```

---

## Branch Protection — Minimum Standards

Every production repository should have these branch protection rules on `main`:

| Rule | Why |
|---|---|
| Require pull request before merging | Enforces code review |
| Require at least 1 approval | No self-merging |
| Dismiss stale reviews on push | A new commit invalidates old approvals |
| Require status checks to pass | CI must pass before merge |
| Require branches to be up to date | Prevents stale merges |
| Do not allow force pushes | Protects shared history |
| Do not allow deletions | Prevents accidental main deletion |

---

## Repository Hygiene

```bash
# Clean up merged branches regularly
git fetch --prune
git branch -vv | grep ': gone]' | awk '{print $1}' | xargs git branch -d

# Verify repository health
git fsck

# Compact the object store
git gc --aggressive

# Check repository disk usage
git count-objects -vH
```

---

## Commit Frequency and PR Size

| Anti-pattern | Impact |
|---|---|
| Giant PRs (500+ lines) | Reviewers cannot meaningfully review — rubber-stamping |
| Commits every 3 days | Context is lost, conflicts accumulate |
| WIP commits on main | CI breaks for everyone |

**Target**: PRs under 400 lines of meaningful change. One logical unit per PR.

---

## .gitattributes — Line Endings and Diff Behavior

```gitattributes
# Normalize line endings to LF on checkout
* text=auto

# Force LF for shell scripts
*.sh text eol=lf

# Force CRLF for Windows batch files
*.bat text eol=crlf

# Binary files — no diff, no merge
*.png binary
*.pdf binary
*.zip binary

# Terraform files — use HCL diff
*.tf diff=terraform
```

---

## Large Repository Handling

```bash
# Shallow clone (for CI — get only recent history)
git clone --depth=1 https://github.com/org/repo.git

# Sparse checkout (only checkout specific directories)
git clone --no-checkout https://github.com/org/monorepo.git
cd monorepo
git sparse-checkout init --cone
git sparse-checkout set modules/vpc modules/eks
git checkout main

# Git LFS for large binary files
git lfs install
git lfs track "*.bin"
git lfs track "diagrams/*.png"
git add .gitattributes
```

---

## References

| Resource | URL |
|---|---|
| Conventional Commits | https://www.conventionalcommits.org |
| Pre-commit framework | https://pre-commit.com |
| Gitleaks | https://github.com/gitleaks/gitleaks |
| Git LFS | https://git-lfs.com |
| GitHub Branch Protection | https://docs.github.com/en/repositories/configuring-branches-and-merges |
| gitignore.io | https://www.toptal.com/developers/gitignore |
