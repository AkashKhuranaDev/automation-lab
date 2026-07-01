# Git Tags — Release Management and Versioning

## Overview

Tags are permanent references to specific commits. Unlike branches, they do not move when new commits are added. They are the correct mechanism for marking releases, milestones, and auditable snapshots of your codebase.

---

## Why Tagging Matters

| Without tags | With tags |
|---|---|
| "Deploy the commit from last Friday" — which one? | `git checkout v2.4.1` — precise, unambiguous |
| Rollback requires digging through log history | `git checkout v2.3.0` — immediate |
| Release audit trail is reconstructed from commit messages | Release audit trail is a first-class artifact |
| CI/CD pipelines trigger on branch — hard to version | CI/CD pipelines trigger on `v*` tags — clean version semantics |

---

## Tag Types

### Lightweight tags

A pointer to a commit. Stored as a ref file. No metadata.

```bash
git tag v1.0.0
```

Do not use lightweight tags for releases. They carry no author, date, or message information beyond the commit they point to.

### Annotated tags

A full Git object with author, date, message, and optionally a GPG signature. Use these for all releases.

```bash
git tag -a v1.0.0 -m "Release v1.0.0 — initial production release"
```

```bash
git show v1.0.0
# tag v1.0.0
# Tagger: Akash Khurana <akash@example.com>
# Date:   Tue Jul 1 18:00:00 2025 +0000
#
# Release v1.0.0 — initial production release
#
# commit abc1234...
```

---

## Semantic Versioning

Tags should follow [Semantic Versioning](https://semver.org): `MAJOR.MINOR.PATCH`

| Segment | Increment when |
|---|---|
| `MAJOR` | Breaking change — consumers must update their code |
| `MINOR` | New backward-compatible functionality |
| `PATCH` | Backward-compatible bug fixes |

```
v1.0.0     — initial stable release
v1.1.0     — new feature added
v1.1.1     — bug fix in v1.1.0
v2.0.0     — breaking API change
v2.0.0-rc.1 — release candidate
v2.0.0-beta.1 — beta
```

---

## Tagging Workflow

### Tag the current HEAD

```bash
git checkout main
git pull origin main
git tag -a v1.2.0 -m "Release v1.2.0

Changes:
- feat: add VPC module for production [INFRA-1042]
- fix: correct IAM role trust policy [SEC-114]
- chore: update Terraform provider to 5.x"
```

### Tag a specific commit

```bash
git tag -a v1.1.1 abc1234 -m "Hotfix v1.1.1 — memory leak in worker pool"
```

### Push tags to remote

```bash
# Push a specific tag
git push origin v1.2.0

# Push all local tags not yet on remote
git push origin --tags
```

---

## Listing and Searching Tags

```bash
# List all tags
git tag

# List with pattern
git tag -l "v1.*"
# v1.0.0
# v1.1.0
# v1.1.1
# v1.2.0

# List tags sorted by version
git tag -l --sort=version:refname "v*"

# Show tag details
git show v1.2.0
```

---

## Signed Tags — For Regulated Environments

Signed tags use GPG to cryptographically verify the tagger's identity. Required in some compliance frameworks.

```bash
# Configure signing key
git config --global user.signingkey <GPG-KEY-ID>

# Create a signed tag
git tag -s v1.2.0 -m "Release v1.2.0"

# Verify a signed tag
git tag -v v1.2.0
```

---

## Deleting Tags

```bash
# Delete locally
git tag -d v1.0.0-beta.1

# Delete from remote
git push origin --delete v1.0.0-beta.1
```

Do not delete tags that have been used in deployments or referenced in audit records.

---

## Expected Output

```bash
$ git tag -a v1.2.0 -m "Release v1.2.0"

$ git show v1.2.0
tag v1.2.0
Tagger: Akash Khurana <akash@example.com>
Date:   Tue Jul 1 18:00:00 2025 +0000

Release v1.2.0

commit 3f8a2b1c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f90 (tag: v1.2.0, HEAD -> main)
Author: Akash Khurana <akash@example.com>
Date:   Tue Jul 1 17:45:00 2025 +0000

    chore: update Terraform provider to 5.x

$ git log --oneline --decorate | head -5
3f8a2b1 (HEAD -> main, tag: v1.2.0, origin/main) chore: update Terraform provider to 5.x
```

---

## CI/CD Integration

Tags drive release pipelines. GitHub Actions example:

```yaml
on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Get version
        run: echo "VERSION=${GITHUB_REF#refs/tags/}" >> $GITHUB_ENV
      - name: Deploy
        run: ./scripts/deploy.sh $VERSION
```

This triggers only on version tags, not on every commit. The pipeline gets the version from the tag name.

---

## Real Enterprise Use Cases

**Infrastructure module versioning**

Terraform modules are versioned with annotated tags. Downstream consumers pin to `v1.2.0` in their module source. Upgrading is a conscious, tested decision.

**Compliance audit trail**

A regulated financial environment requires that every production deployment be traceable to a specific commit. Tags provide the immutable reference. CI/CD refuses to deploy if the artifact was not built from a tagged commit.

**Multi-environment promotion**

`v1.2.0-rc.1` deploys to staging. After approval, the same commit is tagged `v1.2.0` and deployed to production. The history is clear.

---

## Common Mistakes

| Mistake | Consequence |
|---|---|
| Using lightweight tags for releases | No author/date/message — not useful for audit |
| Forgetting to push tags | Tags exist locally only — CI/CD never sees them |
| Tagging from a dirty or non-`main` branch | Tag points to an unreviewed or incorrect state |
| Deleting or moving tags that were deployed | Breaks reproducibility and audit trails |
| Not following semver | Consumers cannot determine upgrade impact |

---

## Best Practices

- Always use annotated tags for releases
- Always push tags explicitly after creation
- Follow semantic versioning consistently
- Include a meaningful message — at minimum, what changed and what tickets it addresses
- Protect tags in your repository settings (prevent deletion and overwriting)
- Automate tagging in your release pipeline — do not rely on humans to remember

---

## Troubleshooting

### "Tag exists locally but CI doesn't see it"

```bash
git push origin v1.2.0
```

### "I tagged the wrong commit"

```bash
# Move the tag (before it has been used for a deployment)
git tag -d v1.2.0
git tag -a v1.2.0 <correct-commit> -m "Release v1.2.0"
git push origin --delete v1.2.0
git push origin v1.2.0
```

### "I need to check out the exact state at a tag"

```bash
git checkout v1.2.0
# This puts you in detached HEAD state
# To work from this point, create a branch:
git checkout -b investigate/v1.2.0 v1.2.0
```

---

## References

| Resource | URL |
|---|---|
| Git Tagging | https://git-scm.com/book/en/v2/Git-Basics-Tagging |
| git tag | https://git-scm.com/docs/git-tag |
| Semantic Versioning | https://semver.org |
| GitHub — Managing releases | https://docs.github.com/en/repositories/releasing-projects-on-github |
