# 03 — Enable Policy Enforcement (and the Enforcement Wizard)

This page is the mechanical "click here, set this, confirm
that" guide for enabling enforcement on a workspace once the
[pre-enforcement checklist](./01-pre-enforcement-checklist.md)
and [agent readiness](./02-agent-readiness.md) are green.

> **Cisco source.** [Manage Policy Lifecycle — Enable Policy Enforcement](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#enable-policy-enforcement)
> and [Policy Enforcement Wizard](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#policy-enforcement-wizard).

---

## Where in the UI

Open the workspace's **Enforcement** page. Per Cisco's
[Enable Policy Enforcement](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#enable-policy-enforcement),
two flows exist:

- The **per-workspace** wizard (more detailed; what's described
  below).
- A **multi-scope** wizard (less detailed; for enforcing
  multiple scopes simultaneously).

Click *Enable Enforcement* (label may vary by release) to
launch the per-workspace wizard.

---

## The Policy Enforcement Wizard — Cisco's documented steps

Per Cisco's [Policy Enforcement Wizard](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#policy-enforcement-wizard),
the wizard's purpose is to:

> *"Review policies before they are implemented on the
> workloads … Download policy changes for review … Compare
> policy versions … Choose which analyzed version of the
> workspace to enforce … Roll back policies to a previous
> version."*

The documented steps are:

### Step 1 — Select Policy Updates

> *"You can select which version of policies to be enforced on
> the workloads. The difference between the currently enforced
> policies and policies in the selected version is displayed."*

This is where rollback happens too: pick an earlier analyzed
version on this step instead of the latest. Per Cisco, the
diff is filterable and downloadable (CSV) much like the
standalone [Policy Diff](./08-policy-versions.md).

What to check:

- The shown analyzed version is the one your team reviewed.
- The diff vs. the currently-enforced version is what you
  expect — no surprise rules.

### Step 2 — Impacted Workloads

> *"This step shows the workloads that will be affected by the
> new firewall rules generated from the selected policy
> changes."*

Cisco notes:

> *"The actual impacted workloads might be smaller due to other
> factors such as agent config intents."*

This is the wizard *displaying* the impacted set; it is not a
selector. Sub-population selection (e.g. "enforce only on web
tier first") is configured separately via **Agent Config
Profiles / Agent Config Intents**, *not* in this wizard. See
[`02-agent-readiness.md`](./02-agent-readiness.md).

### Step 3 — Impacting Policies

> *"Policies from the ancestor workspaces may impact workloads
> in the current workspace. Therefore, you should make sure the
> desired allow policies from ancestor workspaces are
> enforced."*

Read the ancestor list carefully. A common gotcha: an ancestor
scope's Catch-All Deny that the workspace owner had assumed
*didn't* apply.

### Step 4 — Review & Accept

> *"This final step summarizes the policy changes to be
> enforced, the number of potentially impacted workloads, and
> the catch-all action that will be enforced. When you click
> **Accept and Enforce**, the policies in the workspace will
> be used to calculate the new firewall rules that will be
> configured on the relevant workloads."*

You can supply a name, description, and reason for action — do
this for every enforcement event, including rollback (where
you supply only the reason; name and description for a past
version cannot be changed).

---

## What "enforcement mode" actually means in CSW — and where it's set

There is **no Visibility / Simulate / Enforce selector inside
the Policy Enforcement Wizard.** Agent behaviour (whether the
agent is reporting flows only, applying the host firewall, or
both) is configured separately via the **Agent Config
Profile** that the agent participates in.

The Enforcement Wizard's job is "which version of policy
should be enforced for this workspace." The agent's role —
deep visibility, enforcement, etc. — comes from agent config.
See [`02-agent-readiness.md`](./02-agent-readiness.md) and the
[Software Agents](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/deploy-software-agents.html)
chapter for agent configuration details.

The phased rollout described in [`04-rollout-pattern.md`](./04-rollout-pattern.md)
composes both: agent config (for what the agent does) and
workspace-level enforcement (for which version is canonical).

---

## What "policy is now enforcing" looks like

Within seconds–minutes:

| Indicator | Healthy |
|---|---|
| Workspace's *Enforcement* page | Shows the new Enforced Policy Version (top-left) |
| Agent → cluster status | Still `OK` for every in-scope agent |
| Agent type | `Enforcement` for agents whose Agent Config makes them enforcing |
| Host-side firewall | Contains CSW-managed chains / WFP filters with rules from the enforced version |
| Flow telemetry | Continues uninterrupted |
| **Rejected flow count** | At-or-near zero, matching what Live Policy Analysis predicted before enforcement |

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
