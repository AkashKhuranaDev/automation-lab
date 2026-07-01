# Git Operational Playbooks

Playbooks are step-by-step operational procedures for production Git scenarios. Unlike [case studies](../case-studies/README.md) (which document past incidents) or [production incidents](../production-incidents/README.md) (which document ongoing incident response), playbooks are **proactive procedures you follow before, during, and after planned operations**.

Read them once. Practice them in a lab environment. Reference them under pressure.

---

## Index

| # | Playbook | Scenario | Risk Level |
|---|----------|----------|------------|
| 01 | [Force Push Recovery](01-force-push-recovery.md) | Someone force-pushed over shared history | P1 — Critical |
| 02 | [Production Hotfix](02-production-hotfix.md) | Emergency fix needed on production right now | P1 — Critical |
| 03 | [Repository Migration](03-repository-migration-checklist.md) | Move repository between organizations or platforms | High |
| 04 | [Clean Large Binary Files](04-clean-large-binary-files.md) | Repository has grown to unacceptable size | High |
| 05 | [Recover Deleted Branch](05-recover-deleted-branch.md) | A branch was deleted before merging | Medium |
| 06 | [Recover Lost Commits](06-recover-lost-commits.md) | Commits disappeared after reset or rebase | Medium |
| 07 | [Disaster Recovery](07-disaster-recovery.md) | Repository is corrupt or remote is unavailable | P1 — Critical |
| 08 | [Repository Maintenance](08-repository-maintenance.md) | Scheduled maintenance for repository health | Low |
| 09 | [Safe Branch Cleanup](09-safe-branch-cleanup.md) | Remove stale branches without losing work | Low |

---

## How to Use These Playbooks

1. **Before the operation:** Read the full playbook, including the rollback strategy
2. **Prerequisite check:** Verify every item in the Prerequisites section before starting
3. **One step at a time:** Do not skip steps, even if they seem redundant
4. **Verify after each phase:** Confirmation steps are not optional
5. **Rollback triggers:** Know the rollback condition before you start — if X happens, stop and revert

---

## Relationship to Other Sections

- **Case Studies** (`../case-studies/`) — What went wrong and why, documented after the fact
- **Production Incidents** (`../production-incidents/`) — Active incident response procedures by type
- **Decision Guides** (`../decision-guides/`) — Which approach to choose before acting
- **Labs** (`../labs/`) — Practice these scenarios safely before you need them under pressure

---

## Navigation

[⌂ Git Index](../README.md)
