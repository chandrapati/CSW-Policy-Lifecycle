# 04 — Live Policy Analysis

Live Policy Analysis is the **continuous simulation** of the
workspace's policy against currently arriving flows. It does
not block traffic. It tells you, in near real time, *"if this
policy were enforced right now, this is what would happen."*

It is the gold standard for "is this policy safe to enforce?".

> **Cisco source.** [Manage Policy Lifecycle — Live Policy Analysis](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#live-policy-analysis).

---

## What Live Analysis does

Inputs:

- The current published or about-to-be-published policy in the
  workspace.
- The **live** stream of flow data from agents in the workspace's
  scope.

For every flow that arrives, the cluster computes a **disposition**
(what the network actually did to the flow — `ALLOWED` or
`DROPPED`) and applies the analyzed policy. The combination
yields one of three result categories:

| Result | Cisco definition | What it tells you |
|---|---|---|
| **Permitted** | Allowed by the network *and* by the analyzed policies | Intended state — no action |
| **Escaped** | Allowed by the network, but *would have been dropped* by the analyzed policies | **The actionable signal** — flow that would be rejected if you enabled enforcement now |
| **Rejected** | Dropped by the network *and* would also have been denied by the analyzed policies | Network or another layer is already blocking it; policy agrees |

> **Cisco source.** [Flow Disposition](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#flow-disposition).

Crucially: **the agent does *not* block the flow.** Live Analysis
sits alongside Visibility (or Simulate, which is the same thing
expressed in enforcement terms — see
[`../enforcement/04-rollout-pattern.md`](../enforcement/04-rollout-pattern.md)).

**Escaped is the category that matters most pre-enforcement.**
Every Escaped flow is something the workspace would block when
enforcement turns on — so it's either a policy gap to fix or
an unwanted flow you'll let enforcement reject.

---

## When and how to run Live Analysis

Open *Workspace → Analysis → Live Policy Analysis* and *Start*.
The cluster begins evaluating arriving flows against the
workspace's current policy.

Recommended timeline:

| Day | Action |
|---|---|
| Day 0 | Quick Analysis is clean. Start Live Policy Analysis. |
| Days 1–3 | Triage every **Escaped** flow. Refine policy or accept the would-be denial. |
| Days 4–5 | Verify trend: zero unexpected Escaped flows for 48 h, including overnight. |
| Day 5+ (if app has a weekly batch) | Continue until the batch has run cleanly under simulation. |
| Day 30 (if app has a monthly batch) | Continue until the monthly batch has run cleanly under simulation. |

Don't rush. *"It's been clean for 24 hours"* is not enough for
production.

---

## What to do with Escaped flows

Each Escaped flow (allowed by the network, would-be-denied by
your policy) is one of:

| Category | Action |
|---|---|
| Legitimate flow we forgot to allow | Add an Allow rule. Re-run Quick Analysis. Keep Live Analysis running. |
| Misconfigured client — never should have been talking to that endpoint | Fix at source. Verify the rejection becomes 0 hits. |
| Periodic / rare legitimate flow we want to allow | Add an Allow rule with a description that explains the periodicity. |
| Truly malicious / unauthorised | The policy was right to deny. Document; alert if appropriate. |

Repeat until **Escaped count = 0** (or every non-zero is fully
explained) holds across a full business cycle.

---

## What about flows that policy permits but the network drops?

Cisco's three-category model (Permitted / Escaped / Rejected)
doesn't have an explicit name for the **disposition=DROPPED +
policy=ALLOW** corner — flows the policy *would* allow but
that the network is dropping anyway. These typically show up
in the Conversations or Flows views with a `DROPPED` disposition
and don't appear in the Escaped count of Live Analysis.

Common causes (none of which Live Analysis can fix on its own):

- The receiving side reset / refused / had no listener.
- A host firewall outside CSW's control (GPO, hand-edited
  iptables) is blocking it.
- A network ACL or security group upstream is blocking it.
- A stateful network device in the path is unhealthy.

When troubleshooting *"the policy is right but something is
still broken,"* this is the corner to investigate — but the
diagnostic flow lives in [`05-conversations.md`](./05-conversations.md)
and [`../operations/03-troubleshooting-blocked-flows.md`](../operations/03-troubleshooting-blocked-flows.md),
not in the Live Analysis Permitted/Escaped/Rejected output itself.

---

## Live Analysis exit criteria — when to enable enforcement

The team should be able to assert all of:

- ✅ Live Policy Analysis has been running for **at least 5
  business days** on this workspace's scope.
- ✅ The window includes any weekly batch / scheduled jobs.
- ✅ **Escaped** count is at zero (or every non-zero Escaped
  flow is explicitly accepted as "we'll let enforcement reject
  this").
- ✅ Workspace is **published** (p\*) and the published version
  matches what Live Analysis is evaluating.
- ✅ Rollback path is documented (per
  [`../docs/01-prerequisites.md`](../docs/01-prerequisites.md)
  and [`../enforcement/09-rollback-and-revert.md`](../enforcement/09-rollback-and-revert.md)).

If all green, proceed to enforcement —
[`../enforcement/04-rollout-pattern.md`](../enforcement/04-rollout-pattern.md).

---

## Live Analysis is not a one-shot

Live Analysis is left running indefinitely as a **drift detector**.
After enforcement is on, Live Analysis (or its post-enforcement
sibling, Enforcement Reporting) shows new flows that *would*
have been rejected by the existing policy — typically a sign of
new application behaviour that needs a policy update. See
[`../operations/01-policy-drift.md`](../operations/01-policy-drift.md).

---

## Common Live Analysis pitfalls

| Pitfall | Symptom | Fix |
|---|---|---|
| Started Live Analysis before publishing | Result reflects the discovered (v\*) version, which isn't what enforcement will use | Publish first, then start |
| Dismissed weekly batch as noise | Day 7 surprise: massive batch flow shows up as Escaped | Wait at least one weekly cycle before declaring clean |
| Confused Live Analysis (simulation) with Simulate enforcement | Different things; both don't block traffic, but they're different stages | Use Live Analysis pre-publish; Simulate enforcement post-publish, pre-Enforce — see [`../enforcement/04-rollout-pattern.md`](../enforcement/04-rollout-pattern.md) |
| Stopped Live Analysis after enforcement | Lost ongoing drift signal | Leave it running; or rely on Enforcement Reporting in [`../operations/01-policy-drift.md`](../operations/01-policy-drift.md) |

---

## See also

- [`03-quick-analysis.md`](./03-quick-analysis.md)
- [`05-conversations.md`](./05-conversations.md)
- [`../enforcement/04-rollout-pattern.md`](../enforcement/04-rollout-pattern.md)
- [`../operations/01-policy-drift.md`](../operations/01-policy-drift.md)
- Cisco: [Live Policy Analysis](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#live-policy-analysis)
