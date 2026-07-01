# Git Architecture Reference

This folder contains deep technical documentation on how Git works internally. The target audience is engineers who need to reason about Git behavior at the system level — diagnosing corruption, building tooling, evaluating trade-offs, or preparing for principal/staff-level interviews.

**Prerequisite:** Read the [Internals overview](../internals/README.md) first. This folder builds on those concepts with more detail and diagrams.

---

## Index

| Document | Topic |
|----------|-------|
| [Objects](objects.md) | Blob, tree, commit, and tag object formats; SHA computation; deduplication |
| [Refs](refs.md) | How branches, tags, HEAD, ORIG_HEAD, and remote refs work |
| [Storage](storage.md) | Loose objects, packfiles, the index (staging area), and working tree |
| [Commit Graph](commit-graph.md) | DAG structure, garbage collection, reachability, and the commit-graph file |

---

## Why This Matters for Infrastructure Engineers

Understanding Git's internals is not an academic exercise. It has direct operational consequences:

- **Diagnosing corruption:** `git fsck` output is only interpretable if you understand what the objects are
- **Performance troubleshooting:** Repository slowness is often packfile fragmentation or excessive loose objects
- **History rewriting:** `git filter-repo` works by rewriting objects; understanding object types helps you predict which operations are safe
- **Security:** Knowing that Git is content-addressed explains why you can verify the integrity of any commit without trusting the server
- **Tooling:** GitHub Actions, GitLab CI, and most Git automation reads refs and objects directly; understanding the format means understanding the tool

---

## Navigation

[⌂ Git Index](../README.md) | [Internals Overview →](../internals/README.md)
