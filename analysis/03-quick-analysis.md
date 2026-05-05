# 03 — Quick Analysis

Quick Analysis replays a chosen window of **historical flow
data** against the workspace's current policy and reports what
*would have been* permitted, denied, or escaped (no matching
rule). It's the fastest way to know whether the policy you've
authored is broken.

> **Cisco source.** [Manage Policy Lifecycle — Quick Analysis](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#quick-analysis).

---

## What Quick Analysis does

Inputs:

- The current policy in the workspace (discovered or authored).
- A flow-data window (default: last 6 hours; adjustable).

Output: for every flow in the window, one of:

| Result | Meaning |
|---|---|
| **Allowed** | A policy matched and its action was *Allow* |
| **Denied** | A policy matched and its action was *Deny* (including the workspace's Catch-All Deny) |
| **Escaped** | No policy matched (rare; should be impossible with a Catch-All) |
| **Mis-matched / no policy** | A flow CSW saw but couldn't simulate against (usually means the flow's consumer/provider isn't in any inventory filter currently in scope) |

The output is **simulated only** — Quick Analysis does *not*
push policy to agents and *does not* affect production traffic.

---

## When to run Quick Analysis

| Trigger | Why |
|---|---|
| Immediately after each ADM run | Catch obvious problems before spending time refining |
| After every meaningful policy edit | Did you introduce a new gap? |
| Before promoting a secondary workspace to primary | Last sanity check before changing what the cluster considers canonical |
| Before publishing (v\* → p\*) | Last sanity check before policy becomes enforceable |

Typical cadence on an active ADM iteration: **dozens of times.**
Quick Analysis is fast (seconds–minutes), so use it liberally.

---

## Reading Quick Analysis output

Open *Workspace → Analysis → Quick Analysis*. The result table
groups by:

| Column | What it tells you |
|---|---|
| **Consumer** | Who initiated |
| **Provider** | Who received |
| **Service (proto/port)** | Which port |
| **Result** | Allowed / Denied / Escaped |
| **Matching policy** | The specific rule that decided the result |
| **Hit count** | How many flows in the window matched this row |

The first thing to look at is the **Denied** rows. Sort by hit
count (descending). For each:

- Is this flow legitimate? If yes, you have a missing or
  mis-ranked Allow rule. **Don't publish** until you've added it.
- Is this flow expected to be denied (a probe, a debug tool, a
  scanner)? Confirm the Catch-All / Deny rule is doing its job.
  Document the expected denial in the workspace description.

Then check **Escaped** rows. Any escapes mean your Catch-All
isn't covering every flow — usually a sign that consumer or
provider isn't matched by any of your filters. Resolve before
publishing.

---

## Quick Analysis vs. Live Analysis

Quick Analysis uses **historical** flows. That's good — it's
reproducible, fast, and lets you iterate. But it's only as good
as the window it's given:

- A 6-hour window won't catch a daily batch.
- A 24-hour window won't catch a weekly batch.
- A 30-day window will, but takes longer to compute.

Quick Analysis catches **the bulk** of issues. Live Policy
Analysis ([`04-live-analysis.md`](./04-live-analysis.md)) is
where you catch the **tail.**

---

## A "Quick Analysis is clean" gate

Before publishing, the team should be able to say:

- ✅ Quick Analysis run with a window that includes a complete
  business cycle (or the longest periodic event the app has).
- ✅ Zero unexpected Denied rows; every Denied row has a
  *"yes, this should be denied"* note.
- ✅ Zero Escaped rows.
- ✅ The same configuration (window, filters) reproduces the
  result on a re-run.

If those are green, proceed to Live Policy Analysis.

---

## Common Quick Analysis findings

| Finding | Cause | Fix |
|---|---|---|
| Inbound from monitoring host denied | Monitoring host not labelled / not in `prod-monitoring` filter | Label, or add to inventory filter |
| Outbound DNS denied | `prod-shared-dns` filter wasn't created or has wrong members | Fix filter at parent scope |
| One workload constantly denied talking to another in the same tier | Cross-tier-internal flow ADM didn't propose; usually clustering / replication. Add as a Default Allow. |
| Lots of *Escaped* (no policy matched) | Catch-All not authored at workspace level — fix immediately |
| Denied flow whose consumer is "Unknown" | Inventory gap on the consumer side | Label / scope the consumer workload, re-run |

---

## See also

- [`04-live-analysis.md`](./04-live-analysis.md)
- [`05-conversations.md`](./05-conversations.md)
- [`01-review-discovered-policies.md`](./01-review-discovered-policies.md)
- Cisco: [Quick Analysis](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#quick-analysis)
