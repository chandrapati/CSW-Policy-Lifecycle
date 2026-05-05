# Day-2 Operations

Once a workspace is enforcing in steady state, the day-1
runbooks (discovery, analysis, enforcement) shift to day-2
concerns: drift, troubleshooting, evidence, handover, and
compliance integration.

---

## What's in this folder

| File | Purpose |
|---|---|
| [`01-policy-drift.md`](./01-policy-drift.md) | Detect when reality drifts from intent |
| [`02-version-history.md`](./02-version-history.md) | Activity logs, version retention, audit anchors |
| [`03-troubleshooting-blocked-flows.md`](./03-troubleshooting-blocked-flows.md) | A flow that should be allowed is being denied — triage path |
| [`04-evidence-and-audit.md`](./04-evidence-and-audit.md) | Per-policy evidence buckets for compliance |
| [`05-handover-runbook.md`](./05-handover-runbook.md) | Steady-state operations runbook |
| [`06-compliance-companion.md`](./06-compliance-companion.md) | How this guide pairs with [`chandrapati/CSW-Compliance-Mapping`](https://github.com/chandrapati/CSW-Compliance-Mapping) |

---

## The day-2 mindset

| Day-1 question | Day-2 question |
|---|---|
| "Is this policy correct?" | "Is this policy *still* correct?" |
| "Will this rollout break anything?" | "Did anything change since the rollout that we should know about?" |
| "What does the app actually do?" | "What does the app do *now*, vs. what it did six months ago?" |
| "Is enforcement on?" | "Is enforcement still on, on every workload, with the policy we think?" |

The signal flow is similar (Live Analysis, Conversations, agent
health), but the cadence is steady-state and the bar is
different — *catching drift early* matters more than *getting
the initial rollout right*.

---

## Daily / weekly / monthly cadence

| Cadence | Activity | Page |
|---|---|---|
| Continuous | Live Policy Analysis or Enforcement Reporting on every enforced workspace | [`../analysis/04-live-analysis.md`](../analysis/04-live-analysis.md) |
| Daily | Review new / unexpected rejected flows | [`03-troubleshooting-blocked-flows.md`](./03-troubleshooting-blocked-flows.md) |
| Weekly | Review new conversations on key workspaces | [`01-policy-drift.md`](./01-policy-drift.md) |
| Monthly | Spot-check host firewall on sample agents per workspace | [`../enforcement/06-verify-enforcement.md`](../enforcement/06-verify-enforcement.md) |
| Monthly | Pull current policy snapshot into evidence bucket | [`04-evidence-and-audit.md`](./04-evidence-and-audit.md) |
| Quarterly | End-of-quarter version pin on every primary workspace | [`02-version-history.md`](./02-version-history.md) |
| Quarterly | Cross-reference compliance mappings | [`06-compliance-companion.md`](./06-compliance-companion.md) |
| Annually | Re-baseline ADM run as a sanity check | [`../discovery/03-run-adm.md`](../discovery/03-run-adm.md) |

---

## Who owns what

A clean steady-state has explicit ownership, not a single
admin team trying to track everything:

| Role | Owns |
|---|---|
| **App owner** | Their workspace's policy intent; reviews changes proposed by ADM / discovery |
| **Platform / SRE** | Enforcement state, agent health, the rollout pattern |
| **Security** | Absolute Deny rules at parent scopes, cross-workspace consistency, the Catch-All semantics |
| **Compliance** | The evidence bucket, audit log, mappings to frameworks |
| **CSW admin** | The cluster itself, scopes, RBAC, connectors, agent install pipeline |

Each role has a runbook entry in
[`05-handover-runbook.md`](./05-handover-runbook.md).

---

## See also

- [`../enforcement/11-monitoring-after-enforcement.md`](../enforcement/11-monitoring-after-enforcement.md)
- [`../api/05-gitops-pattern.md`](../api/05-gitops-pattern.md)
- Companion repo: [`chandrapati/CSW-Compliance-Mapping`](https://github.com/chandrapati/CSW-Compliance-Mapping)
- Cisco: [Manage Policy Lifecycle](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html)
