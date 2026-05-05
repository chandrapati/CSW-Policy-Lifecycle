# 10 — Pause Policy Updates and Emergency Disable

Two distinct controls — easy to confuse. Both reduce the
"surface area" of a problem; they do different things.

> **Cisco source.** [Manage Policy Lifecycle — Pause Policy Updates](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#pause-policy-updates)
> and [Disable Policy Enforcement](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#disable-policy-enforcement).

---

## Side-by-side

| | **Pause Policy Updates** | **Disable Policy Enforcement** |
|---|---|---|
| What it does | Freezes the *push* of updated policy to agents | Removes enforcement; agents stop blocking based on this workspace's policy |
| Host firewall state | Whatever was last pushed before pause; **still enforcing** | Reverts to the agent's pre-enforcement / Visibility state |
| Use when | Authoring multiple changes, want to push them as a batch | Production is breaking and you need to take CSW out of the path **now** |
| Reversible | Yes — Resume Updates pushes the latest published policy | Yes — Re-enable enforcement; goes back through Simulate → Enforce per [`04-rollout-pattern.md`](./04-rollout-pattern.md) |
| Audit weight | Low | High — this is an incident-response action |

If in doubt under incident pressure: **disable**, then triage,
then re-enable through the normal phased flow.

---

## Pause Policy Updates

Use case: you're about to make several related policy edits and
you want them to land on agents as one push, not several.

### Procedure

1. Open *Workspace → Enforce → Pause Policy Updates*.
2. Make edits (multiple v\* iterations are fine).
3. Run Quick Analysis after each edit; Live Analysis as appropriate.
4. Publish the final v\* → p\*.
5. *Resume Policy Updates*. The cluster pushes the latest p\*
   to the agents.

While paused:

- Agents continue enforcing the **previously pushed** policy.
- New edits accumulate but don't reach the host.
- Health and flow telemetry are unaffected.

### When pause is the right answer

| Scenario | Why pause |
|---|---|
| Coordinated change across multiple workspaces | Pause all of them, edit, publish, resume together |
| Iterating on a single workspace during a maintenance window | Avoid intermediate-state pushes that could confuse on-call observers |
| Long-running ADM iteration where intermediate v\* shouldn't reach hosts | Already true (v\* never reaches hosts); pause is unnecessary here — useful only if the intermediate state is *published* |

### When pause is *not* the right answer

| Scenario | Better choice |
|---|---|
| Production is broken and you need to revert | **Disable** (this page) or **revert** ([`09-rollback-and-revert.md`](./09-rollback-and-revert.md)) |
| You've published a bad p\* by accident | Revert to the previous good p\* |
| Agent-level issue (push isn't reaching hosts) | Investigate agent health; pause won't help |

---

## Disable Policy Enforcement

Use case: production is breaking; the simplest correct action is
to take CSW out of the enforcement path.

### Procedure (incident path — fast version)

1. Open *Workspace → Enforce → Disable Enforcement*.
2. Confirm the population (the entire scope, or a subset).
3. Click *Disable*.

The cluster pushes the disable to agents. Agents remove the
CSW-managed firewall rules and revert to their pre-enforcement
state — typically Deep Visibility.

Push time is seconds–minutes. Application traffic that was
being blocked due to policy starts flowing again as agents
process the update.

### What disable does *not* do

- Disable does **not** delete the workspace's policy. The p\* /
  v\* history is preserved.
- Disable does **not** change the workspace's published version.
- Disable does **not** uninstall agents.
- Disable does **not** stop flow telemetry — agents continue
  reporting flows, just not enforcing.

### After disable — the calm phase

Once production is recovering:

1. Confirm with app owners / on-call that the issue's resolved.
2. Open Live Policy Analysis on the workspace — what *would*
   have been rejected if enforcement were on right now?
3. Diff the last good p\* against the bad p\* to identify which
   rule(s) caused the issue.
4. Author a fix in a new v\*.
5. Publish.
6. Re-enable through the Monitor → Simulate → Enforce flow per
   [`04-rollout-pattern.md`](./04-rollout-pattern.md). **Don't
   skip Simulate** because of post-incident pressure to "get
   back to enforcing." The whole reason this happened is that
   something wasn't caught in Simulate.

---

## API alternative — the scripted incident path

For very high-availability deployments, **wire the disable
into your incident-response automation** ahead of time:

- A button or a simple script that calls
  `/openapi/v1/applications/{app_id}/disable_enforce` for the
  affected workspace.
- A runbook that documents which workspace IDs map to which
  business apps so on-call can disable the right one without
  guessing.

See [`../api/03-enforcement-toggle-api.md`](../api/03-enforcement-toggle-api.md)
for the request shape.

---

## Rehearsal — the underrated step

Disable is the single most important control in this folder.
**Rehearse it.** In a non-production workspace:

1. Pick a Friday afternoon.
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
