# Workspaces — the Unit of Policy Management

Workspaces are where policy lives, gets reviewed, and gets
published. Almost every UI flow in the *Manage Policy Lifecycle*
chapter starts with "open the workspace for the scope you care
about."

> **Cisco source.** [Manage Policy Lifecycle — Use Workspaces to Manage Policies](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#use-workspaces-to-manage-policies).

---

## Workspaces vs. scopes vs. policies

| Object | What it is | Cardinality |
|---|---|---|
| **Scope** | A node in the inventory tree (e.g. `Production / BU-Retail / payments-api`). Membership = label expression. | Many per tenant; structured as a tree. |
| **Workspace** | A container for a working set of policies on one scope. | One **primary** workspace per scope; optionally **secondary** workspaces for what-if / experiments. |
| **Policy** | A single allow / deny rule. | Many per workspace. |

A scope can have only one *primary* workspace at a time, but it
can have additional *secondary* workspaces — typically for
modelling a proposed change without disturbing the canonical
policy set.

---

## Primary vs. secondary workspaces

| | Primary | Secondary |
|---|---|---|
| Used for | The canonical policy set for the scope | What-if modelling, experiments, draft branches |
| Counts toward enforced policy | Yes | No |
| Can be enforced | Yes | No (must first be promoted to primary, replacing the existing primary) |
| Limit | One per scope | Several per scope |
| Typical lifespan | Indefinite | Short — promoted, discarded, or merged |

The mental model: *primary = main branch, secondary = feature
branch.* You usually do ADM into a secondary workspace, refine
it there, then promote it to primary when you're ready to
publish (p\*) and enforce.

---

## Anatomy of a workspace

A workspace contains:

1. **Policies** — the rules themselves, with their consumer /
   provider, port / proto, action, rank.
2. **Clusters** — workspace-local groups of workloads identified
   by ADM as behaving together (web-01..web-12 → "web cluster").
   Clusters are *workspace-local*; they don't exist outside
   their workspace. See
   [`discovery/04-clusters-and-inventory-filters.md`](../discovery/04-clusters-and-inventory-filters.md).
3. **Inventory filters** — label expressions that resolve to
   sets of workloads. Usually shared at the scope level so they
   can be referenced from many policies.
4. **ADM run history** — every time you run automatic discovery,
   the result becomes a new discovered (v\*) version.
5. **Published versions** — every time you publish, you get a
   new p\* version. Publishing is the act that makes a workspace's
   policy enforceable.
6. **Enforcement state** — Monitor / Simulate / Enforce, plus
   whether updates are paused.

---

## Lifecycle of a workspace

```
   Create (manual or auto from ADM run)
          │
          ▼
   Author / Discover  ──► v1 (discovered) ──► v2 ──► v3 …
          │                       refine, iterate
          ▼
   Review and analyze  (Quick Analysis · Live Analysis)
          │
          ▼
   Publish  ──────────► p1 (published)
          │
          ▼
   Enforce (Monitor → Simulate → Enforce)
          │
          ├──► policy modified ──► v(n+1) ──► publish ──► p2 …
          │
          └──► rollback to earlier p* if needed
```

The discovered (v\*) and published (p\*) version streams are
distinct. v\* tracks what *could* be enforced; p\* tracks what
*has been* published as the canonical policy. Both streams retain
history; both can be diffed; published versions can be reverted.
See [`enforcement/08-policy-versions.md`](../enforcement/08-policy-versions.md).

---

## Creating a workspace

The Cisco UI walks the create flow at *Defend → Segmentation →
Workspaces → Create*. Practical guidance:

| Field | Recommendation |
|---|---|
| **Name** | `<application>-<purpose>` — e.g. `payments-api-primary`, `payments-api-whatif-2026q2` |
| **Scope** | The scope that owns this app — usually the leaf scope, not a parent |
| **Description** | Owner email + change ticket reference; saves a lot of triage time later |
| **Primary / Secondary** | Default to *Secondary* for ADM and what-if work. Promote to primary deliberately. |

> **Tip.** Keep workspace names mechanical and predictable.
> Multiple parallel POVs and audits work better when a name
> reveals scope, app, and intent without needing to open the
> workspace.

---

## RBAC and workspace access

A user's access to a workspace is determined by their role on
the **scope** the workspace lives on. A Scope Owner on
`Production / BU-Retail / payments-api` can do everything in any
workspace on that scope but cannot author policy in
`Production / BU-Retail / crm-web`.

> **Implication for shared-services apps.** Shared services
> (DNS, AD, monitoring) typically live on a separate sibling
> scope so the appropriate platform team owns their workspaces,
> while application teams remain unable to edit them.

Reference: [Controlling User Access to Workspaces](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#controlling-user-access-to-workspaces).

---

## Common workspace operations (quick reference)

| Operation | Where in UI (verify against your release) | Notes |
|---|---|---|
| Create | *Defend → Segmentation → Workspaces → Create* | Pick scope + primary/secondary |
| Rename | Workspace settings | Cosmetic; doesn't affect policy |
| Delete | Workspace settings | Only secondary; primary deletion requires demotion first |
| Run Automatic Policy Discovery | Workspace → *Automatic Policy Discovery* | See [`../discovery/03-run-adm.md`](../discovery/03-run-adm.md) |
| Quick Analysis | Workspace → *Run Quick Analysis* (right nav pane on the Policies page) | Tests a **single hypothetical flow** against current policy ([Cisco doc](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#quick-analysis)) |
| Policy Experiments | Workspace → *Policy Analysis* | Replays past traffic against current policies ([Cisco doc](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#run-policy-experiments-to-test-current-policies-against-past-traffic)) |
| Live Policy Analysis | Workspace → *Policy Analysis* | Continuous evaluation against arriving flows |
| Analyze Latest Policies | Workspace → *Policy Analysis* | Increments the analysis-context p\* version |
| Enable Policy Enforcement | Workspace → *Enforcement* (Policy Enforcement Wizard) | See [`../enforcement/03-enable-enforcement.md`](../enforcement/03-enable-enforcement.md) |
| Stop Policy Enforcement (single scope) | Workspace's Policy Enforcement page → red **Stop Policy Enforcement** button | Reverts to ancestor-scope enforced policy; agents stop applying this workspace's enforcement |
| Pause Policy Updates | *Defend → Enforcement* → status indicator beside Policy Updates | **Cluster-wide and site-admin only** — pauses rule updates for ALL scopes / workloads |

---

## See also

- [`docs/04-policy-attributes.md`](./04-policy-attributes.md) —
  the rules that live inside workspaces
- [`docs/05-decision-matrix.md`](./05-decision-matrix.md) —
  whether to populate the workspace via ADM or manual authoring
- [`discovery/`](../discovery/README.md) — populate the workspace
- [`analysis/`](../analysis/README.md) — review the workspace
- [`enforcement/`](../enforcement/README.md) — enforce the
  workspace
- Cisco: [Use Workspaces to Manage Policies](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#use-workspaces-to-manage-policies)
