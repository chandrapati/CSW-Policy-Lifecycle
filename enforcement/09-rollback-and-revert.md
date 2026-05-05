# 09 — Rollback and Revert

CSW supports reverting an enforced workspace to an earlier
published version. This is the primary policy-level rollback
mechanism — distinct from emergency-disable (covered in
[`10-pause-and-emergency-disable.md`](./10-pause-and-emergency-disable.md)).

> **Cisco source.** [Manage Policy Lifecycle — Revert Enforced Policies to an Earlier Version](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#revert-enforced-policies-to-an-earlier-version).

---

## When to use revert vs. disable

| Scenario | Use |
|---|---|
| Recent change introduced an issue; want to go back to a known-good policy | **Revert** to the previous good p\* |
| Production is broken right now, no time to figure out which version is right | **Disable** enforcement — see [`10-pause-and-emergency-disable.md`](./10-pause-and-emergency-disable.md) |
| A rule edit was wrong but only one rule needs reverting | **Edit** in workspace + new publish + Simulate-then-Enforce — see [`07-modify-enforced-policies.md`](./07-modify-enforced-policies.md) |
| Need to entirely back out a planned change before its window closes | **Revert** if the change has been published and enforced |

Revert preserves enforcement (workspace stays in Enforce mode);
the *content* of what's enforced rolls back. Disable removes
enforcement entirely.

---

## How revert works

```
   currently enforced: p_n  (the bad version)
            │
            │ revert to p_n-1
            ▼
   enforced: p_n-1  (the previously known-good version)
```

CSW records the revert as a new event in the workspace's
Activity Log (it doesn't create a "p_n+1 = same as p_n-1"; the
version stream still shows p_n existed and was rolled away from).
Subsequent forward changes will produce p_n+1, p_n+2, etc.

---

## The revert procedure

### Step 1 — Pick the target version

Open *Workspace → Policies → Versions*. Sort by p\*. Identify
the most recent **good** p\*:

| Indicator of "good" | Where to find it |
|---|---|
| Was enforced cleanly for a meaningful window | Activity Log |
| Live Analysis was clean during its enforcement | Live Analysis history |
| Doesn't include the rule(s) you suspect caused the issue | [Policy Diff](./08-policy-versions.md) vs. current p\* |
| Was associated with a known-good change ticket | Workspace description / activity log |

If you can't pick a "known good" within ~5 minutes during an
incident, **don't try** — go to disable instead. Trying to
pick the right revert target while production is down wastes
time.

### Step 2 — Confirm the diff

Open Policy Diff between current p_n and the chosen p_(n-k).
Confirm the diff is what you'd expect — only the rules whose
removal is the goal are leaving.

### Step 3 — Revert

In the workspace's Enforce / Policy Enforcement view:
*Revert to Earlier Version → \[selected p\*\] → Confirm*.

The cluster pushes the older policy to all agents in scope.
Push time is typically seconds–minutes depending on workspace
size.

### Step 4 — Verify the revert took effect

Same checks as [`06-verify-enforcement.md`](./06-verify-enforcement.md):

- Workspace state still `Enforcing`.
- Agent type still `Enforcement`.
- Host firewall on a sample of agents now reflects the older p\*.
- Active probes confirm the older policy's behaviour.
- Application synthetic checks recover.

### Step 5 — Record what happened

The change ticket / runbook gets updated with:

- The version reverted *from* and *to*.
- The reason (rule X caused issue Y; pinpoint via Policy Diff).
- The next forward step (fix in a v_n+m, publish as p_n+1, go
  through Simulate before Enforce again).

---

## After a revert — don't immediately re-publish forward

A revert is "we went backward to a safe state." The natural
next instinct is "let's fix the issue and re-publish forward."
That's correct, but **not in a hurry**:

- Re-run [Quick Analysis](../analysis/03-quick-analysis.md) on
  the proposed forward fix.
- Run [Live Analysis](../analysis/04-live-analysis.md) for at
  least a few hours.
- Use Simulate phase before Enforce, even if the change is small.

The same pre-flight discipline that should have caught the
original issue is the one that prevents the next iteration from
hitting it again.

---

## When revert isn't enough

| Symptom | Beyond-revert action |
|---|---|
| Reverting to the previous p\* doesn't restore application function | The issue isn't (only) policy — investigate the application or another infra layer; consider disable while you do |
| The "previous good p\*" is also bad in light of new information | Walk back further; or disable enforcement and rebuild policy fresh |
| Revert succeeds but agents still report the old (bad) policy on the host | Agent push didn't propagate; verify agent connectivity, restart the agent if needed |
| You can't find a good version to revert to | Disable enforcement; rebuild from a published-version export ([`08-policy-versions.md`](./08-policy-versions.md)) |

---

## API alternative

The same revert can be invoked via OpenAPI for scripted
incident-response. See [`../api/03-enforcement-toggle-api.md`](../api/03-enforcement-toggle-api.md)
for the relevant endpoints.

---

## See also

- [`08-policy-versions.md`](./08-policy-versions.md)
- [`10-pause-and-emergency-disable.md`](./10-pause-and-emergency-disable.md)
- [`06-verify-enforcement.md`](./06-verify-enforcement.md)
- [`../operations/02-version-history.md`](../operations/02-version-history.md)
- Cisco: [Revert Enforced Policies to an Earlier Version](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#revert-enforced-policies-to-an-earlier-version)
