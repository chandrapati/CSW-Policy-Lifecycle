# 10 — Pause Policy Updates and Disable Policy Enforcement

Two distinct controls that practitioners often confuse. Both
reduce the impact of a problem; they do **very** different
things. The corrections in this page reflect the actual Cisco
documentation — earlier framing of "Pause" as a per-workspace
batching tool was incorrect.

> **Cisco source.**
> [Pause Policy Updates](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#pause-policy-updates)
> and [Disable Policy Enforcement](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#disable-policy-enforcement).

---

## Side-by-side

| | **Pause Policy Updates** | **Disable Policy Enforcement** |
|---|---|---|
| Scope of effect | **GLOBAL** — pauses rule updates for ALL workloads in ALL scopes | Per workspace (or via multi-scope wizard) |
| Required role | **Site admin or customer-support** privileges | Workspace owner / scope admin |
| What changes on the host | Nothing immediately — agents keep enforcing whatever was last pushed | Agents revert to pre-enforcement state (typically Deep Visibility); CSW-managed firewall rules removed |
| Use when | You need to halt all CSW-driven firewall changes across the entire cluster (e.g. cluster-side investigation, suspected propagation issue) | Production is breaking and you need to take CSW out of the path on a specific workspace |
| Reversibility | Yes — toggle Policy Updates back on and the latest published policy is pushed | Yes — re-enable enforcement; goes back through the standard analyze-then-enforce flow |
| Audit weight | High — global change | High — incident-response action |

If in doubt under incident pressure: **disable the affected
workspace**, then triage, then re-enable through the normal
flow.

---

## Pause Policy Updates — what it actually is

Per Cisco's [Pause Policy Updates](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#pause-policy-updates),
this is a **cluster-wide** (not per-workspace) control with the
warning:

> *"This option pauses policy updates for ALL workloads in ALL
> scopes."*
> *"This feature requires site admin or customer support
> privileges."*

### When pause is the right answer

| Scenario | Why a global pause is right |
|---|---|
| Suspected cluster-side propagation issue affecting many workspaces | Halt rule updates everywhere while you investigate |
| Maintenance on the cluster's enforcement plane | Avoid in-flight pushes mid-maintenance |
| Multi-scope coordinated change with an unusually wide blast radius | Pause everything; do the change; resume |

### When pause is *not* the right answer

| Scenario | Better choice |
|---|---|
| You want to batch several edits on **one** workspace | There's no per-workspace pause. Just author the edits in the workspace; v\* iterations don't push to agents until you publish + enforce |
| Production is broken on one workspace | Disable that workspace's enforcement (this page) or revert it ([`09-rollback-and-revert.md`](./09-rollback-and-revert.md)) |
| Agent-level issue (push isn't reaching some hosts) | Investigate agent health; pause won't help |

### Procedure

Per Cisco:

1. Navigate to *Defend → Enforcement*.
2. Click the status indicator beside **Policy Updates**.
3. Read and accept the caution dialog.

Site admin / customer-support role required. Use only when you
genuinely need to halt updates for the whole tenant.

---

## Disable Policy Enforcement

Use case: production is breaking; the simplest correct action is
to take CSW out of the enforcement path on the affected
workspace.

### Procedure (single workspace — the incident path)

Per Cisco's [Disable Policy Enforcement — single scope](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#disable-policy-enforcement):

> *"Navigate to the Policy Enforcement page for the scope's
> primary workspace and click the red **Stop Policy Enforcement**
> button. This writes new firewall rules to assets in the scope
> based on enforced policies in ancestor workspaces. A Label
> Flag with an 'x' will be created on the time series chart."*

So for a single workspace under incident pressure:

1. Open the affected primary workspace.
2. Go to its Policy Enforcement page.
3. Click the **red Stop Policy Enforcement button**.

The cluster pushes the change to agents. Two important
behaviours:

- The agent doesn't go back to "no rules" — it falls back to
  whatever rules **ancestor scopes' enforced workspaces**
  define. If an ancestor workspace has Catch-All Deny rules
  applied to descendants, those still apply.
- A Label Flag with an 'x' appears on the workspace's time
  series chart so the action is visible in subsequent triage.

### Procedure (multiple scopes simultaneously)

Per Cisco:

> *"Follow the procedure for enforcing policy in multiple scopes
> simultaneously, as described in Enable Policy Enforcement.
> On the Select Version page of the wizard, click Select a
> version and choose **Disable enforcement**."*

So multi-scope disable is a special case of the multi-scope
Enable wizard — pick "Disable enforcement" instead of a
version.

### What disable does *not* do

- **Not** delete the workspace's policy. The version history is
  preserved.
- **Not** uninstall agents.
- **Not** stop flow telemetry — agents continue reporting flows,
  just not enforcing this workspace's policy.
- **Not** override ancestor-scope enforcement — ancestor rules
  still apply (see above).

### After disable — the calm phase

Once production is recovering:

1. Confirm with app owners / on-call that the issue's resolved.
2. Open Live Policy Analysis on the workspace — what *would*
   have been rejected if enforcement were on right now?
3. Diff the last good version against the bad one to identify
   which rule(s) caused the issue (Policy Diff — see
   [`08-policy-versions.md`](./08-policy-versions.md)).
4. Author a fix in a new v\*.
5. Analyze (Quick Analysis, Policy Experiments, Live Policy
   Analysis).
6. Re-enable through the rollout pattern in
   [`04-rollout-pattern.md`](./04-rollout-pattern.md).

---

## API alternative — the scripted incident path

For very high-availability deployments, **wire enforcement
disable into your incident-response automation** ahead of time.
The OpenAPI exposes enforcement enable / disable endpoints —
exact paths depend on your release; consult the
[Secure Workload OpenAPIs chapter](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/secure-workload-openapis.html)
for your release. See [`../api/03-enforcement-toggle-api.md`](../api/03-enforcement-toggle-api.md)
for the pattern.

The pre-built script needs:

- A button or chatops command (`/csw-disable <workspace>`).
- A runbook entry mapping business apps to workspace IDs.
- A narrowly-scoped API key with only the enforcement-toggle
  capability (per [`../api/01-authentication.md`](../api/01-authentication.md)).

---

## Rehearsal — the underrated step

Disable is the single most important control on this page.
**Rehearse it.** In a non-production workspace:

1. Pick a maintenance window.
2. Walk on-call through the procedure (UI path + API).
3. Time how long from "we need to disable" to "production is
   recovering."
4. Document any surprises.

Until on-call has done this once, you don't have working
disable. The first time can't be in a real incident.

---

## See also

- [`04-rollout-pattern.md`](./04-rollout-pattern.md)
- [`07-modify-enforced-policies.md`](./07-modify-enforced-policies.md)
- [`09-rollback-and-revert.md`](./09-rollback-and-revert.md)
- [`11-monitoring-after-enforcement.md`](./11-monitoring-after-enforcement.md)
- [`../api/03-enforcement-toggle-api.md`](../api/03-enforcement-toggle-api.md)
- [`../operations/05-handover-runbook.md`](../operations/05-handover-runbook.md)
- Cisco: [Pause Policy Updates](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#pause-policy-updates)
- Cisco: [Disable Policy Enforcement](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#disable-policy-enforcement)
