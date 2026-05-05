# 06 — Compliance Companion

This repo (`CSW-Policy-Lifecycle`) covers *how* CSW policy is
authored, analysed, enforced, and operated. Its sibling repo
[`chandrapati/CSW-Compliance-Mapping`](https://github.com/chandrapati/CSW-Compliance-Mapping)
covers *which compliance controls those policy practices satisfy*.

This page describes how the two are intended to be used together.

---

## What lives where

| Concern | This repo | Compliance-Mapping repo |
|---|---|---|
| How to discover policy from flow data | Yes (`discovery/`) | No |
| How to analyse policy before enforcement | Yes (`analysis/`) | No |
| The Monitor → Simulate → Enforce pattern | Yes (`enforcement/`) | No |
| Day-2 ops, drift, audit | Yes (`operations/`) | No |
| OpenAPI / Kafka programmatic management | Yes (`api/`) | No |
| Mapping CSW capabilities to NIST 800-53 controls | No | Yes |
| Mapping CSW capabilities to PCI DSS, HIPAA, SOC 2, etc. | No | Yes |
| Asset / capability tables for individual CSW features | No | Yes |
| Audit-ready evidence templates per control | No | Yes (templates) — paired with `operations/04-evidence-and-audit.md` here |

---

## How to use them together

### Onboarding a new application

1. **Compliance** — open the compliance repo's frameworks
   relevant to the app (e.g. PCI DSS for a payments app).
   Identify which control requirements segmentation satisfies.
2. **Lifecycle** — follow this repo's onboarding flow:
   - [`discovery/`](../discovery/README.md)
   - [`analysis/`](../analysis/README.md)
   - [`enforcement/04-rollout-pattern.md`](../enforcement/04-rollout-pattern.md)
3. **Evidence** — capture per [`04-evidence-and-audit.md`](./04-evidence-and-audit.md).
   Each artefact ties back to a control mapping in the compliance
   repo.
4. **Cross-reference** — the workspace's evidence bucket
   includes a snapshot reference (commit hash) of the compliance
   repo at the time of onboarding, so the framework version is
   unambiguous.

### Annual / cyclical compliance audit

1. **Compliance** — open the compliance repo for the framework
   under audit. Pull the control list.
2. **Lifecycle** — for each control:
   - Identify the relevant workspaces / policies.
   - Pull the relevant Activity Log entries
     ([`02-version-history.md`](./02-version-history.md)).
   - Pull the relevant evidence bucket snapshots
     ([`04-evidence-and-audit.md`](./04-evidence-and-audit.md)).
3. **Auditor package** — assemble per the compliance repo's
   templates. The lifecycle repo provides the *operational*
   evidence; the compliance repo provides the *control mapping*
   that frames that evidence.

### When a control changes

E.g. a new framework version introduces a tighter requirement.

1. **Compliance** — the compliance repo is updated to reflect
   the new control language. PR-reviewed.
2. **Lifecycle** — if the new control implies a CSW practice
   change (e.g. shorter retention, additional audit step), the
   relevant page in this repo is updated. PR-reviewed.
3. **Operations** — the change-management process pushes the
   updated practice through the existing workspaces:
   [`../enforcement/07-modify-enforced-policies.md`](../enforcement/07-modify-enforced-policies.md)
   for policy changes, [`05-handover-runbook.md`](./05-handover-runbook.md)
   for procedural changes.

---

## What to put in evidence — the cross-walk

For each major control category, here's how lifecycle artefacts
map to evidence:

| Control category | Lifecycle artefact | Where in this repo |
|---|---|---|
| Network segmentation between zones | Workspace policy with cross-scope rules; Catch-All Deny | [`docs/04-policy-attributes.md`](../docs/04-policy-attributes.md), [`analysis/06-policy-complexities.md`](../analysis/06-policy-complexities.md) |
| Restricted access to sensitive systems | Inventory filter for sensitive class; targeted Allow rules; Default Deny everywhere else | [`discovery/04-clusters-and-inventory-filters.md`](../discovery/04-clusters-and-inventory-filters.md) |
| Change management for security-relevant changes | GitOps PR history; Activity Log; change tickets | [`api/05-gitops-pattern.md`](../api/05-gitops-pattern.md), [`02-version-history.md`](./02-version-history.md) |
| Continuous monitoring of policy effectiveness | Live Policy Analysis | [`analysis/04-live-analysis.md`](../analysis/04-live-analysis.md) |
| Periodic review of policy | Quarterly review process | [`05-handover-runbook.md`](./05-handover-runbook.md), [`01-policy-drift.md`](./01-policy-drift.md) |
| Audit logging | Activity Log + SIEM forwarding | [`02-version-history.md`](./02-version-history.md) |
| Roles and responsibilities | RBAC + ownership in workspace metadata | [`docs/03-workspaces.md`](../docs/03-workspaces.md), [`05-handover-runbook.md`](./05-handover-runbook.md) |
| Incident response | Disable / revert procedures, runbooks | [`enforcement/10-pause-and-emergency-disable.md`](../enforcement/10-pause-and-emergency-disable.md), [`enforcement/09-rollback-and-revert.md`](../enforcement/09-rollback-and-revert.md) |

The compliance repo turns each row's left column into the exact
language of NIST / PCI / HIPAA / etc.

---

## Anti-patterns

| Anti-pattern | Why it fails |
|---|---|
| Maintaining compliance mappings in this repo | Duplicates the compliance-mapping repo; will drift |
| Maintaining policy lifecycle guidance in the compliance repo | Same problem in reverse |
| Capturing evidence without referencing a control | "We did it" without "why" — audits ask "why" |
| Capturing the control without referencing concrete CSW evidence | "We had a control" without "we operated it" — audits ask "operate it" |

The pairing — operational evidence here, control mapping there
— keeps both repos focused and reviewable.

---

## See also

- [`04-evidence-and-audit.md`](./04-evidence-and-audit.md)
- [`02-version-history.md`](./02-version-history.md)
- [`05-handover-runbook.md`](./05-handover-runbook.md)
- Companion repo: [`chandrapati/CSW-Compliance-Mapping`](https://github.com/chandrapati/CSW-Compliance-Mapping)
- Companion repo: [`chandrapati/CSW-Agent-Installation-Guide`](https://github.com/chandrapati/CSW-Agent-Installation-Guide)
- Cisco: [Manage Policy Lifecycle](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html)
