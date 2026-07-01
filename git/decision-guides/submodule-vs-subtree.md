# Decision Guide: Submodule vs Subtree

For managing dependencies that live in separate Git repositories — shared libraries, shared infrastructure code, or vendor dependencies.

---

## Quick Decision

```
Do you want the dependency updates to happen explicitly (not automatically)?
└─ Yes → Git Submodule (version-pinned, manual update)

Do you want contributors to work without knowing about the dependency structure?
└─ Yes → Git Subtree (transparent to most contributors)

Do you own the dependency repository and modify it frequently from the parent?
└─ Yes → Git Subtree (easier bidirectional sync)

Do you consume a third-party library with infrequent updates?
└─ Yes → Git Submodule (pin to a specific commit)

Are there > 5 dependencies to manage?
└─ Consider a proper package manager instead (pip, npm, Go modules, etc.)
```

---

## The Core Difference

**Git Submodule:** The parent repository stores a _reference_ to a specific commit in the dependency repository. The dependency's files are not stored in the parent — only the pointer. Cloning the parent does not automatically clone the dependency.

**Git Subtree:** The dependency's files are _copied into_ the parent repository, with their history merged into the parent's history. There is no external reference — it's all in one repository. Cloning the parent gives you everything.

---

## Git Submodules

### How it works

```bash
# Add a submodule
git submodule add git@github.com:org/shared-lib.git libs/shared

# This creates:
# .gitmodules — stores the submodule URL and path
# libs/shared/ — a checked-out clone of the dependency at a specific commit
# The parent repo stores: "libs/shared points to commit abc1234 in org/shared-lib"

cat .gitmodules
# [submodule "libs/shared"]
#     path = libs/shared
#     url = git@github.com:org/shared-lib.git
```

### Cloning a repository with submodules

```bash
# Without --recurse: libs/shared/ is empty
git clone git@github.com:org/parent-repo.git

# Fix empty submodule
git submodule update --init --recursive

# OR: clone and initialize submodules in one step
git clone --recurse-submodules git@github.com:org/parent-repo.git
```

### Updating to a new version of the dependency

```bash
cd libs/shared
git pull origin main               # Update to latest
cd ../..
git add libs/shared               # Stage the pointer update (new SHA)
git commit -m "chore: update shared-lib to v2.1.0"
```

### Pushing changes to the dependency

```bash
cd libs/shared                    # You're now inside the dependency clone
# Make changes
git commit -m "fix: ..."
git push origin main              # Push to the dependency's repository
cd ../..
git add libs/shared               # Update the parent's pointer to the new commit
git commit -m "chore: update shared-lib pointer"
```

---

## Git Subtree

### How it works

```bash
# Add a subtree (copies the dependency's history into the parent repository)
git subtree add --prefix=libs/shared git@github.com:org/shared-lib.git main --squash

# --prefix: where in the parent to put the files
# --squash: compress the dependency's full history into a single commit (recommended for vendors)
```

The dependency's files are now part of the parent repository. There is no `.gitmodules`. Contributors see a `libs/shared/` directory but have no indication it came from another repository.

### Updating to a new version

```bash
git subtree pull --prefix=libs/shared git@github.com:org/shared-lib.git main --squash
```

This creates a new commit in the parent that applies the changes from the dependency.

### Pushing changes upstream

If you modify files in `libs/shared/` within the parent and want to contribute them back:

```bash
git subtree push --prefix=libs/shared git@github.com:org/shared-lib.git main
```

Git extracts the commits that touched `libs/shared/` and pushes them to the dependency's repository.

---

## Comparison Matrix

| Criteria | Submodule | Subtree |
|----------|-----------|---------|
| Files in parent repo | No (pointer only) | Yes (full copy) |
| Clone requires extra step | Yes (`--recurse-submodules`) | No |
| Contributor awareness | Must know about submodules | Transparent |
| Version pinning | Explicit (commit SHA) | Implicit (after each pull) |
| `git diff` shows dependency changes | No | Yes |
| Bidirectional sync complexity | Medium | Low |
| Repository size impact | Small (pointer only) | Larger (full history) |
| CI/CD setup | Extra step needed | Works out of the box |
| Vendor/third-party updates | Well-suited | Well-suited |
| Active shared development | Difficult | Easier |
| `git log` shows dependency commits | Separate repo | Mixed into parent (unless --squash) |

---

## Submodule Pain Points

1. **New team members forget `--recurse-submodules`.** The `libs/shared/` directory is empty until they run `git submodule update --init`. This causes confusing build failures.
2. **Two-step push required.** Changes to the dependency must be pushed to the dependency repo AND the pointer update committed to the parent. Forgetting either step breaks CI.
3. **Detached HEAD in submodule.** When you `cd libs/shared`, you are in detached HEAD state by default. Making commits here is fine, but you must remember to `git checkout main` first.
4. **Branch tracking is not automatic.** The submodule pointer tracks a specific SHA, not a branch name. If the dependency releases v2.1.0, nothing happens in the parent until you explicitly update the pointer.

```bash
# Fix: configure a branch to track
git submodule set-branch --branch main libs/shared
git config -f .gitmodules submodule.libs/shared.branch main
```

---

## Subtree Pain Points

1. **History pollution.** Without `--squash`, the dependency's full commit history is merged into the parent's `git log`. With 500 commits in the dependency, `git log main` becomes noisy.
2. **Subtree push is slow.** `git subtree push` scans all commits to find those that touched the subtree prefix. On large repositories, this can take minutes.
3. **No explicit version pinning.** There is no file that records "we are using shared-lib at commit abc1234." You must rely on commit messages or tags to track the version.
4. **Conflicts are harder to diagnose.** When `git subtree pull` conflicts, the conflict markers are in the normal working tree — there is no indication they came from the dependency.

---

## Neither is Right: Use a Package Manager

For most cases, a package manager is the correct answer:

| Language | Tool |
|----------|------|
| Python | `pip` + `requirements.txt` or `pyproject.toml` |
| JavaScript/TypeScript | `npm` or `yarn` + `package-lock.json` |
| Go | Go modules (`go.mod`) |
| Terraform | Terraform module registry + version constraints |
| Infrastructure (shared) | Internal package registry (Artifactory, GitHub Packages) |

Use submodules or subtrees only when:
- The dependency is not publishable to a package registry (internal tooling, configuration templates)
- You need to make changes to the dependency as part of the same workflow as the parent
- The dependency is language-agnostic (shared config files, documentation templates)

---

## Summary Recommendation

| Use case | Tool |
|---------|------|
| Consuming versioned third-party code | Package manager (not Git) |
| Shared internal library, rarely modified from parent | Git Submodule |
| Shared internal code, frequently modified from parent | Git Subtree |
| Vendor code copied for customization | Git Subtree with `--squash` |
| Configuration templates shared across services | Git Subtree |
| Team of > 3 people with the dependency | Git Submodule (explicit versioning) |

---

## Related

- [Enterprise Workflows](../enterprise-workflows/README.md)
- [Performance Reference](../performance/README.md) — large repositories and dependency size
- [Decision Guides Index](README.md)

---

[← GitFlow vs Trunk-Based](gitflow-vs-trunk-based.md) | [← Decision Guides Index](README.md)
