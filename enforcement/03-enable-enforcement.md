# 03 — Enable Policy Enforcement (and the Enforcement Wizard)

This page is the mechanical "click here, set this, confirm
that" guide for enabling enforcement on a workspace once the
[pre-enforcement checklist](./01-pre-enforcement-checklist.md)
and [agent readiness](./02-agent-readiness.md) are green.

> **Cisco source.** [Manage Policy Lifecycle — Enable Policy Enforcement](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#enable-policy-enforcement)
> and [Policy Enforcement Wizard](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#policy-enforcement-wizard).

---

## Where in the UI

*Defend → Segmentation → Workspaces → \[your primary workspace\]
→ Enforce* (or *Policy Enforcement* depending on release) →
*Enable Enforcement*.

This launches the **Enforcement Wizard**, which walks the
specific settings for this enforcement event.

---

## The Enforcement Wizard step by step

### Step 1 — Choose the published version (p\*)

Confirm which p\* will be enforced. This is normally the
latest published version, but for rollback or controlled
experiments you may want an earlier version. The wizard shows
the [Policy Diff](./08-policy-versions.md) between the current
state and the chosen p\*.

Verify:

- The shown p\* is the one your team reviewed.
- The diff vs. the previous enforced p\* (if any) is what you
  expect — no surprise rules.

### Step 2 — Choose enforcement mode

The mode applied to the agents in scope:

| Mode | What the agent does | When |
|---|---|---|
| **Enforced** | Applies the host firewall rules; drops non-matching flows | Final state |
| **Simulate** *(some releases call this "Preview" or related)* | Updates the host firewall rules but takes no enforcement action; reports would-be-rejected flows from the agent's perspective | Bridge between published policy and full enforcement; complements [Live Analysis](../analysis/04-live-analysis.md) |
| **Visibility** | No enforcement; flow telemetry only | Pre-enforcement state |

For a first-time enable, **start in Simulate (or
Monitor-equivalent)** for at least 48–72 h before moving to
Enforced. See [`04-rollout-pattern.md`](./04-rollout-pattern.md)
for the full Monitor → Simulate → Enforce sequence.

### Step 3 — Choose the population

The wizard offers granular control of which workloads in the
scope will be enforced:

| Selector | Use when |
|---|---|
| **Entire scope** | Standard enforcement on a small / coherent app |
| **Inventory filter subset** | Phased rollout — e.g. enforce on `tier=web` first, then `tier=app`, then `tier=db` |
| **Specific workloads** | Canary on 1–2 hosts before broadening |

For a first-time production enforcement on a non-trivial app,
prefer a **subset selector** — start with the lowest-risk tier
(typically web), confirm clean for 24 h, then broaden.

### Step 4 — Confirm and apply

The wizard summarises:

- Workspace + scope
- p\* version being enforced
- Mode (Enforced / Simulate / Visibility)
- Population (all / subset / specific)
- Estimated workload count

Click *Apply*. The cluster pushes configuration to the agents
in the population.

---

## What "policy is now enforcing" looks like

Within seconds–minutes:

| Indicator | Healthy |
|---|---|
| Workspace's *Enforce* page state | `Enforcing` (or `Simulating`, etc., as set) |
| Agent → cluster status | Still `OK` for every in-scope agent |
| Agent type | Now reads `Enforcement` (was `Deep Visibility`) |
| Host-side firewall | Contains CSW-managed chains / WFP filters with rules from p\* |
| Flow telemetry | Continues uninterrupted |
| **Rejected flow count** *(if Enforced mode)* | At-or-near zero, matching what Live Analysis predicted |

If any of those are wrong — agents going unhealthy en masse,
flow telemetry stopping, rejected-flow count exploding — go
straight to [`10-pause-and-emergency-disable.md`](./10-pause-and-emergency-disable.md).

---

## After applying — first-hour checks

| At T+ | Check |
|---|---|
| 5 min | Agent statuses still OK across the population |
| 15 min | Live Analysis or Enforcement Reporting still shows expected flow counts; rejected-flow count is small and explainable |
| 30 min | Application-level synthetic checks (the app's own SLI dashboards) still green |
| 60 min | Quick spot-check of host-side firewall on 2–3 agents — `iptables -L -n` or `Get-NetFirewallRule` shows expected CSW-installed rules |

If first-hour checks are clean, broaden the population and
proceed to the next phase per
[`04-rollout-pattern.md`](./04-rollout-pattern.md).

---

## API alternative

The same enforcement enable can be performed via the
`/openapi/v1/applications/{app_id}/enable_enforce` family of
endpoints. See [`../api/03-enforcement-toggle-api.md`](../api/03-enforcement-toggle-api.md).
Useful for automated change-management pipelines and for
scripted rollback.

---

## See also

- [`04-rollout-pattern.md`](./04-rollout-pattern.md)
- [`06-verify-enforcement.md`](./06-verify-enforcement.md)
- [`08-policy-versions.md`](./08-policy-versions.md)
- [`10-pause-and-emergency-disable.md`](./10-pause-and-emergency-disable.md)
- Cisco: [Enable Policy Enforcement](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#enable-policy-enforcement)
- Cisco: [Policy Enforcement Wizard](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#policy-enforcement-wizard)
