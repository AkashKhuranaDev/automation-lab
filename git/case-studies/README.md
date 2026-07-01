# Case Studies — Git in Enterprise Infrastructure Environments

> **Navigation:** [`← Back to git/`](../) | [`Decision Guides →`](../decision-guides/) | [`Production Incidents →`](../production-incidents/)

These case studies document real patterns encountered in enterprise infrastructure environments. Names and specific details are generalized, but the scenarios, root causes, and resolutions are representative of actual production situations.

Each case study follows a consistent structure so you can quickly locate the section relevant to your current situation.

---

## Case Study Index

| # | Title | Primary Topic | Complexity |
|---|---|---|---|
| [CS-01](01-force-push-recovery.md) | Accidental Force Push Overwrites Main | Recovery, reflog | High |
| [CS-02](02-deleted-branch-recovery.md) | Production Fix Branch Deleted Before Merge | Recovery, branching | Medium |
| [CS-03](03-secrets-in-commit-history.md) | AWS Credentials Committed to Public Repository | Security, filter-repo | High |
| [CS-04](04-emergency-hotfix-workflow.md) | Critical Vulnerability During Release Freeze | Hotfix, cherry-pick | High |
| [CS-05](05-production-rollback.md) | Terraform Module Deployment Causes Outage | Revert, rollback | High |
| [CS-06](06-repository-migration.md) | Migrating 8-Year Monolith from SVN to Git | Migration, history | Very High |
| [CS-07](07-large-repository-optimization.md) | 22 GB Repository Causing 40-Minute Clone Times | LFS, performance | High |
| [CS-08](08-merge-strategy-migration.md) | Team Migrating from GitFlow to Trunk-Based | Strategy change | Medium |
| [CS-09](09-monorepo-sparse-checkout.md) | Monorepo Checkout Slowing CI to 18 Minutes | Sparse checkout | Medium |

---

## How to Use These Case Studies

Each case study describes the environment, problem, and resolution in a format that mirrors real incident reports. Read the complete case study when you are facing a similar situation. Extract the commands and adapt them to your environment.

The **Lessons Learned** and **Best Practices** sections in each case study are the highest-value content — they represent what the team would do differently if starting again.
