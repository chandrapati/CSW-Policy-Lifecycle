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

Output: for every flow as it arrives, one of:

| Result | Meaning |
|---|---|
| **Permitted** | Would be allowed by current policy |
| **Rejected (would be)** | Would be denied if policy were enforced |
| **Misdropped (would be)** | A flow that *was* permitted but for which the agent later observed a connection failure (helps spot near-misses) |

Crucially: **the agent does *not* block the flow.** Live Analysis
sits alongside Visibility (or Simulate, which is the same thing
expressed in enforcement terms — see
[`../enforcement/04-rollout-pattern.md`](../enforcement/04-rollout-pattern.md)).

---

## When and how to run Live Analysis

Open *Workspace → Analysis → Live Policy Analysis* and *Start*.
The cluster begins evaluating arriving flows against the
workspace's current policy.

Recommended timeline:

| Day | Action |
|---|---|
| Day 0 | Quick Analysis is clean. Start Live Policy Analysis. |
| Days 1–3 | Triage every "would-be-rejected" flow. Refine policy or accept the denial. |
| Days 4–5 | Verify trend: zero unexpected rejections for 48 h, including overnight. |
| Day 5+ (if app has a weekly batch) | Continue until the batch has run cleanly under simulation. |
| Day 30 (if app has a monthly batch) | Continue until the monthly batch has run cleanly under simulation. |

Don't rush. *"It's been clean for 24 hours"* is not enough for
production.

---

## What to do with would-be-rejected flows

Each would-be-rejected flow is one of:

| Category | Action |
|---|---|
| Legitimate flow we forgot to allow | Add an Allow rule. Re-run Quick Analysis. Keep Live Analysis running. |
| Misconfigured client — never should have been talking to that endpoint | Fix at source. Verify the rejection becomes 0 hits. |
| Periodic / rare legitimate flow we want to allow | Add an Allow rule with a description that explains the periodicity. |
| Truly malicious / unauthorised | The policy was right to deny. Document; alert if appropriate. |

Repeat until "would-be-rejected count = 0 (or fully explained)"
holds across a full business cycle.

---

## What about the *misdropped* category?

Misdropped is the more subtle case: the **policy permits the
flow, but the connection failed anyway.** That tells you:

- The agent saw a flow it would allow, but the receiving side
  reset / refused / had no listener.
- This is *not* a CSW policy issue — it's an application or
  infra issue.
- But it's *visible* through CSW's flow telemetry, which is
  unusually useful for debugging during a rollout.

When troubleshooting *"the policy is right but something is
broken,"* check misdropped before assuming policy.

---

## Live Analysis exit criteria — when to enable enforcement

The team should be able to assert all of:

- ✅ Live Policy Analysis has been running for **at least 5
  business days** on this workspace's scope.
- ✅ The window includes any weekly batch / scheduled jobs.
- ✅ Would-be-rejected count is at zero (or every non-zero is
  explicitly accepted).
- ✅ Misdropped count is steady — i.e., no new misdrops
  introduced by recent policy changes (or the changes have been
  accounted for).
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
| Dismissed weekly batch as noise | Day 7 surprise: massive batch flow is would-be-rejected | Wait at least one weekly cycle before declaring clean |
| Confused Live Analysis (simulation) with Simulate enforcement | Different things; both don't block traffic, but they're different stages | Use Live Analysis pre-publish; Simulate enforcement post-publish, pre-Enforce — see [`../enforcement/04-rollout-pattern.md`](../enforcement/04-rollout-pattern.md) |
| Stopped Live Analysis after enforcement | Lost ongoing drift signal | Leave it running; or rely on Enforcement Reporting in [`../operations/01-policy-drift.md`](../operations/01-policy-drift.md) |

---

## See also

- [`03-quick-analysis.md`](./03-quick-analysis.md)
- [`05-conversations.md`](./05-conversations.md)
- [`../enforcement/04-rollout-pattern.md`](../enforcement/04-rollout-pattern.md)
- [`../operations/01-policy-drift.md`](../operations/01-policy-drift.md)
- Cisco: [Live Policy Analysis](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#live-policy-analysis)
