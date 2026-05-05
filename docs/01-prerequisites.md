# Prerequisites

Before a single policy is authored, six things must be true.
Almost every failed CSW rollout traces back to one of these
being missing or wrong.

> **Cisco source.** [Manage Inventory for Secure Workload (4.0 On-Prem)](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-inventory-for-secure-workload.html)
> for scopes, labels, and inventory filters.
> [Get Started with Cisco Secure Workload (4.0 On-Prem)](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/get-started-with-cisco-secure-workload.html)
> for the platform overview.

---

## 1. Agents are installed and reporting

Every workload you intend to author policy *for* must have a CSW
software agent installed and reporting **flow telemetry** to the
cluster. Without flow data there is nothing for ADM to discover
and nothing for Live Policy Analysis to evaluate against.

| Check | How to verify |
|---|---|
| Agent process is running | host-side: `service tet-sensor status` (Linux) / *Get-Service* `tet-sensor` (Windows) |
| Agent registered with cluster | UI: *Manage → Agents → Software Agents* — agent appears with status `OK` |
| Agent is in Visibility (Deep Visibility) mode | UI: *Manage → Agents* — `Type` column reads *Deep Visibility* (not yet enforcing) |
| Flow records are arriving | UI: *Investigate → Flows* — flows from this workload appear with the last 5 minutes window |

If any of these are *not* true, finish the agent install first.
Companion runbooks:
[`chandrapati/CSW-Agent-Installation-Guide`](https://github.com/chandrapati/CSW-Agent-Installation-Guide)
— specifically the verification chapter
[`linux/08-verification.md`](https://github.com/chandrapati/CSW-Agent-Installation-Guide/blob/main/linux/08-verification.md)
or [`windows/02-csw-generated-powershell.md`](https://github.com/chandrapati/CSW-Agent-Installation-Guide/blob/main/windows/02-csw-generated-powershell.md).

---

## 2. Inventory is labelled

Policy in CSW is written against **labels** (key/value pairs
attached to inventory). A scope tree built only on IP-CIDR is
brittle and won't survive a re-IP. Get to label-based inventory
before authoring policy of any consequence.

Recommended baseline label keys:

| Key | Example values | Why |
|---|---|---|
| `environment` | `prod`, `non-prod`, `dev`, `test`, `dr` | Almost every Catch-All policy is environment-scoped |
| `application` | `payments-api`, `crm-web`, `mongo-cluster` | The unit of micro-segmentation |
| `tier` | `web`, `app`, `db`, `cache`, `lb`, `ingress` | Drives the typical 3-tier policy template |
| `os` | `linux`, `windows`, `kubernetes` | Platform-specific policies |
| `data_classification` | `pii`, `phi`, `pci`, `internal`, `public` | Required for compliance-mapped reports |
| `business_unit` / `cost_center` | `bu-retail`, `cc-1234` | Useful for scope-tree shape and chargeback |
| `region` / `site` | `aws-us-east-1`, `dc-pri`, `dc-dr` | Geo-aware policy and DR pairing |

Sources of label data, in order of preference:

1. **Connectors** — vCenter, AWS, Azure, GCP, Kubernetes, ISE.
   See [Configure and Manage Connectors](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/configure-and-manage-connectors-for-secure-workload.html).
2. **CMDB import / API push** — preferred for the long term;
   makes inventory tagging the single source of truth.
3. **CSV upload** — pragmatic for a POV or initial labelling pass.
4. **Manual UI tagging** — for outliers and one-off corrections.

> Don't author policy until **at least 90 % of the workloads in
> the scope** carry the keys you intend to write rules against.
> A workload missing the `application` label will fall through
> the scope tree and into a parent scope's Catch-All, with
> surprising results. Inventory labelling is the single highest-leverage
> activity in a CSW rollout.

---

## 3. Scope tree is shaped to match the org

The scope tree is the inventory hierarchy CSW uses to decide
which policies apply to which workloads, and which users can
edit them.

A workable starting shape:

```
Tenant
├── Production
│   ├── BU-Retail
│   │   ├── App: payments-api
│   │   ├── App: crm-web
│   │   └── App: ...
│   ├── BU-Wholesale
│   └── Shared-Services
│       ├── DNS
│       ├── AD
│       └── Monitoring
├── Non-Production
│   ├── Dev
│   ├── Test
│   └── Staging
└── DR
```

Why this shape works:

- **Environment at the top** — Catch-All "deny inbound from
  non-prod to prod" lives once, at the Production scope.
- **BU mid-tier** — gives BU owners the right RBAC blast radius
  without giving them everything.
- **App leaf** — each application owns its own workspace, can
  be discovered, simulated, and enforced independently.
- **Shared-Services as a sibling** — DNS / AD / monitoring
  policies are written once and inherited.

Anti-patterns:

| Don't | Why |
|---|---|
| Make the tree match your *cloud* hierarchy (region → VPC → subnet) | Cloud topology shifts; org structure shifts more slowly. Use cloud info as labels, not scope hierarchy. |
| Have more than ~5 levels of depth | RBAC and Catch-All inheritance get hard to reason about. |
| Put every team's apps in one giant flat scope | Defeats RBAC; one team's policy bug breaks everyone else's apps. |

Reference: [Manage Inventory — Scopes](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-inventory-for-secure-workload.html).

---

## 4. Sufficient flow data has been collected

ADM is a behaviour-based clustering algorithm. It needs
representative traffic to discover representative policy.

| Application profile | Recommended flow window |
|---|---|
| Steady-state stateless web app | 7 days |
| Typical 3-tier business app (web → app → db) | 14 days |
| App with weekly batch / reconciliation jobs | 21 days, ideally including a month-end |
| App with monthly close, quarterly batch | **30+ days, including the largest batch window** |
| HA / DR pair (active/passive failover) | Long enough to include at least one planned failover, or capture flows during a DR test |

If you don't have enough flow data yet, leave the workload in
Visibility, do the labelling work in parallel, and come back to
discovery in 2–4 weeks. **Don't shorten the window.** Skipping
the batch window is the #1 cause of a policy that "looks good
in test, blocks the month-end batch in production."

More detail in
[`discovery/02-flow-collection-window.md`](../discovery/02-flow-collection-window.md).

---

## 5. RBAC is set up before policy authors start work

Policy authoring is a privileged operation. Define roles before
the first workspace is created:

| Role | Permissions |
|---|---|
| **Tenant Owner / Site Admin** | Full tenant config — only 2–3 people |
| **Scope Owner** (per BU or shared-service scope) | Author / review policy for own scope; cannot enforce on someone else's |
| **Policy Author** | Author policy on assigned scope; cannot enforce or publish |
| **Policy Reviewer** | Read-only across scope; can run analysis, can comment |
| **Read-only / Auditor** | Read inventory, read policy, read flow data |

Reference: [User Roles and Permissions](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/get-started-with-cisco-secure-workload.html).

---

## 6. Change management is wired up *before* the first enforcement

Policy enforcement is a change to host-firewall configuration.
Treat it that way:

- **Standard change** (low risk): Monitor → Simulate → Enforce
  for an app that has been in Live Analysis with zero would-be-blocked
  flows for 5+ business days, in non-prod and prod. Pre-approved
  category.
- **Normal change** (medium risk): first enforcement of an app
  in production; first enforcement of a Catch-All; major scope-tree
  reshape.
- **Emergency change**: rolling back enforcement after an incident;
  use [`enforcement/10-pause-and-emergency-disable.md`](../enforcement/10-pause-and-emergency-disable.md).

Define the **rollback path** for every enforcement change *before*
that change is approved. The rollback paths CSW supports are
documented in [`enforcement/09-rollback-and-revert.md`](../enforcement/09-rollback-and-revert.md)
and [`enforcement/10-pause-and-emergency-disable.md`](../enforcement/10-pause-and-emergency-disable.md).

---

## Quick checklist (copy into your rollout plan)

- [ ] Agents installed, registered, in Deep Visibility, reporting flows
- [ ] Inventory labelled — `environment`, `application`, `tier`, `os`, `data_classification` on ≥ 90 % of in-scope workloads
- [ ] Scope tree shaped (env → BU → app, plus Shared-Services sibling)
- [ ] At least 14 days of flow data (30+ for batch-heavy apps)
- [ ] RBAC defined: Tenant Owner, Scope Owner, Policy Author, Reviewer, Read-only
- [ ] Change-management category mapped (standard / normal / emergency)
- [ ] Rollback path documented per enforcement wave

When all six are green, proceed to
[`discovery/01-prepare-scope.md`](../discovery/01-prepare-scope.md).

---

## See also

- [`docs/02-segmentation-basics.md`](./02-segmentation-basics.md)
  — the policy model itself
- [`docs/03-workspaces.md`](./03-workspaces.md) — the unit of
  policy management
- Cisco: [Manage Inventory for Secure Workload](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-inventory-for-secure-workload.html)
- Cisco: [Get Started with Cisco Secure Workload](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/get-started-with-cisco-secure-workload.html)
