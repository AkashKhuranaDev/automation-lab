# Production Incidents — Git Operations

> **Navigation:** [`← Back to git/`](../) | [`Case Studies →`](../case-studies/) | [`Decision Guides →`](../decision-guides/)

These playbooks document Git-related production incidents with the structure, diagnosis steps, and recovery commands needed to resolve them quickly. Each playbook is written to be used during an active incident — commands are immediately executable, decisions are pre-made.

---

## Incident Index

| ID | Title | Severity | Time to Recover (P50) |
|---|---|---|---|
| [P001](P001-force-push-overwrites-main.md) | Force Push Overwrites Protected Branch | P1 | 15 min |
| [P002](P002-secret-committed-to-public-repo.md) | Secret Committed to Public Repository | P1 | 25 min |
| [P003](P003-repository-grows-to-20gb.md) | Repository Grows to 20+ GB | P2 | 2–4 hours |
| [P004](P004-merge-conflict-blocks-release.md) | Merge Conflict Blocks Scheduled Release | P2 | 30 min |
| [P005](P005-ci-pipeline-breaks-on-git-operations.md) | CI Pipeline Breaks on Git Operations | P2 | 45 min |

---

## How to Use These Playbooks

1. **Identify the incident pattern** from the index above
2. **Open the matching playbook**
3. **Start at Symptoms** — verify you have the right playbook
4. **Run Diagnosis commands** to confirm root cause
5. **Execute Recovery steps** in order
6. **Run Verification** to confirm resolution
7. **Implement Prevention** steps before closing the incident

---

## Severity Definitions

| Level | Description | Response SLO |
|---|---|---|
| P1 | Production affected, data at risk, or security breach | Respond in < 15 minutes |
| P2 | Production degraded, release blocked, or CI broken | Respond in < 1 hour |
| P3 | Developer workflow impacted, no production effect | Respond in < 1 business day |
