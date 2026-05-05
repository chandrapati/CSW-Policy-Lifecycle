# 03 — Troubleshooting Blocked Flows

The most common day-2 ticket: *"flow X should be allowed but
something is blocking it; is it CSW?"*. This page is the triage
flow.

---

## Step 0 — Confirm the symptom

Don't assume CSW. Confirm:

- The flow is actually being attempted from the consumer.
- The flow isn't reaching the provider, or is being reset.
- The failure is recent (correlate to any recent change).

If the consumer isn't even attempting the flow, the issue is
upstream (DNS, auth, app config). CSW won't be involved.

---

## Step 1 — Is CSW enforcing on the relevant workload?

| Check | Where |
|---|---|
| Workspace covering the consumer is enforcing | *Workspace → Enforce* state |
| Workspace covering the provider is enforcing | Same, on the other workspace |
| Agent on consumer is in `Enforcement` mode | *Manage → Agents* |
| Agent on provider is in `Enforcement` mode | Same |

If neither end is enforcing, CSW is not blocking. Investigate
network path, host-side firewall, app config.

---

## Step 2 — What does CSW say about this flow?

Open *Investigate → Flows* with a filter on the specific
consumer / provider / port. Each flow record has a status:

| Status | Meaning |
|---|---|
| **Permitted** | Policy allowed; flow should be reaching the provider |
| **Rejected (consumer side)** | Consumer's host firewall blocked outbound |
| **Rejected (provider side)** | Provider's host firewall blocked inbound |
| **Misdrop** | Allowed by policy but failed at TCP / app layer |
| **No record** | Flow not observed; either not attempted, or sampling gap |

Branch from here:

### "Rejected" — CSW is blocking

Open the flow in detail. CSW should show **which rule** caused
the rejection (typically a Catch-All Deny on the rejecting
side). Branch by which rule:

| Rule that rejected | Likely cause |
|---|---|
| Workspace's Catch-All Deny | No specific Allow matched — rule missing or filter wrong |
| Inherited Absolute Deny | Cross-environment / regulatory deny — confirm whether the flow should be allowed at all |
| Inherited Default Deny | Parent-scope rule too broad — investigate which scope owns it |
| Specific Default Deny | Authored rule that thinks this flow is bad — confirm intent |

### "Permitted but app reports failure" — Misdrop

Not a CSW policy issue. Could be:

- Provider not listening on the port.
- TCP connect succeeded but app-layer auth failed.
- Provider's app crashed / restarted.
- A stateful network device in between (NAT, LB) in unhealthy
  state.

Use [Conversations](../analysis/05-conversations.md) and the
agent's process metadata to confirm the provider's process is
listening and healthy.

### "No record" — flow not seen

Three possibilities:

- Flow never attempted — investigate the consumer.
- Flow blocked **before** reaching the agent (network firewall,
  cloud security group). Check those layers.
- Sampling gap — rare; spot-check by tcpdump / packet capture
  on the consumer if you have the access.

---

## Step 3 — If CSW is blocking, why?

Two flavours of "CSW is blocking but it shouldn't be":

### Inventory filter doesn't resolve to the workload

The rule says "allow `tier=app` → `tier=db` tcp/5432" but the
new app-tier workload is missing the `tier=app` label, so the
filter doesn't include it.

**Fix.** Add the label. Filter membership refreshes automatically;
push to agents follows. No policy change needed.

Confirm via *Effective Consumer / Effective Provider* on the
rule (per [`../analysis/06-policy-complexities.md`](../analysis/06-policy-complexities.md)).

### Rule genuinely doesn't allow the flow

The rule allows `tcp/5432` but the flow is `tcp/5433` (test
port).

**Fix.** Either bring the flow back to standard ports, or amend
the rule to allow the additional port. The latter requires the
standard modify-while-enforced flow per
[`../enforcement/07-modify-enforced-policies.md`](../enforcement/07-modify-enforced-policies.md).

### Cross-scope rule on only one side

The flow is `crm-web → payments-api`. `payments-api`'s workspace
allows the inbound; `crm-web`'s workspace doesn't allow the
outbound. Consumer-side host firewall blocks.

**Fix.** Add the outbound rule on the consumer-side workspace.
See [`../analysis/06-policy-complexities.md`](../analysis/06-policy-complexities.md).

---

## Step 4 — If you can't figure it out fast, mitigate

If the blocked flow is causing customer impact and the root
cause isn't immediate:

| Option | When |
|---|---|
| Add a temporary specific Allow rule with a short expiry comment | The flow is legitimate, you can identify consumer + provider, but root cause needs more time |
| Pause Policy Updates on one side while you investigate | If a recent push is suspected and you want to halt further changes |
| Disable enforcement on the affected workspace | High-impact production issue, time-bound investigation needed |

Prefer the smallest mitigation that resolves the impact.
Disable is the heaviest hammer; reach for it last among these.

---

## Step 5 — Document the resolution

Whatever the fix, capture:

| Field | Value |
|---|---|
| Symptom | "Flow A → B on tcp/X failing since 14:02" |
| Root cause | "Workload missing `tier=app` label" |
| Where the bug was | "Inventory labelling pipeline didn't apply on rebuild" |
| Fix | "Re-labelled; verified rule effective consumer now contains the workload" |
| Prevention | "Add label-presence assertion to inventory pipeline" |

This ends up in the postmortem and feeds drift detection
([`01-policy-drift.md`](./01-policy-drift.md)).

---

## Common pre-tested scenarios

| Scenario | Where blocked | Resolution |
|---|---|---|
| New app version uses a new port | Consumer or provider Catch-All | Standard modify; add allow rule |
| Workload migrated cloud regions, IP changed, label forgot to update | Provider workspace's filters don't include it | Reconcile labels |
| Policy edit by a peer team broke a cross-scope flow | Counterpart workspace | Coordinate edit; add the rule on both sides |
| GPO push removed a host-firewall exception CSW expected | Host firewall | Reconcile GPO with CSW; see [`../enforcement/05-platform-specific.md`](../enforcement/05-platform-specific.md) |
| Backup window starts; flow not in policy | Workspace | Add allow with description; capture for next ADM iteration |

---

## See also

- [`01-policy-drift.md`](./01-policy-drift.md)
- [`../analysis/05-conversations.md`](../analysis/05-conversations.md)
- [`../analysis/06-policy-complexities.md`](../analysis/06-policy-complexities.md)
- [`../enforcement/06-verify-enforcement.md`](../enforcement/06-verify-enforcement.md)
- [`../enforcement/07-modify-enforced-policies.md`](../enforcement/07-modify-enforced-policies.md)
- [`../enforcement/10-pause-and-emergency-disable.md`](../enforcement/10-pause-and-emergency-disable.md)
