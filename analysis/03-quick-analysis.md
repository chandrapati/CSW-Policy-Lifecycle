# 03 — Quick Analysis and Policy Experiments

Two distinct fast-iteration tools sit at this stage of the
lifecycle, and they're easy to confuse:

| Tool | What it does | Use when |
|---|---|---|
| **Quick Analysis** | Tests a single **hypothetical flow** (consumer / provider / port / protocol) against the current policy and reports the verdict and matching rule. | "If host A talked to host B on tcp/8443, would policy permit it? Which rule decides?" |
| **Policy Experiments** | Replays a **window of past traffic** against current policies and reports what *would have been* permitted, escaped, or rejected. | "Across the last 6 hours of real flows, what would my current policy do?" |

Both are simulation only — neither pushes anything to agents.

> **Cisco source.** [Manage Policy Lifecycle — Quick Analysis](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#quick-analysis)
> and [Manage Policy Lifecycle — Run Policy Experiments to Test Current Policies Against Past Traffic](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#run-policy-experiments-to-test-current-policies-against-past-traffic).

---

## Quick Analysis — single hypothetical flow

Cisco's definition (paraphrased from the User Guide):

> Quick analysis enables testing a hypothetical flow against
> all the policies in the current workspace and all other
> relevant policies from other workspaces. Quick analysis
> facilitates debugging and experimentation with different
> security policies, without the need to run live policy
> analysis for the workspace.

**Cisco-stated limitations:**

- **Primary workspaces only** — you cannot run Quick Analysis
  on a secondary workspace.
- **Not currently supported on flows from Kubernetes services.**

### Procedure

Open the workspace and click the **Run Quick Analysis** tab on
the right navigation pane. In the dialog, specify the
hypothetical flow:

| Field | Notes |
|---|---|
| Consumer | Workload, address, or filter expression |
| Provider | Workload, address, or filter expression |
| Service | Protocol + port (e.g. `tcp/8443`) |

Quick Analysis returns:

- The **verdict** (Allow / Deny).
- The **matching policy** (which specific rule decided the
  result, including its rank — Absolute / Default / Catch-All).
- Cross-workspace rules that affect the verdict, when relevant.

### When to use Quick Analysis

| Trigger | Why |
|---|---|
| Triaging a denied-flow ticket | "Why is A → B on port X failing?" — Quick Analysis names the rule responsible |
| Reviewing a proposed rule change | "If I add this Allow, does it actually fire for the flow I care about?" |
| Sanity check after editing a filter | "Did changing `prod-web` filter membership change which rule wins?" |
| Cross-workspace debugging | "A flow crosses two workspaces. Which workspace's rule is deciding it?" |

It is **fast** (sub-second) and **per-flow**. Use it liberally
during authoring.

---

## Policy Experiments — replay past traffic

Cisco's User Guide describes this as a separate tool from Quick
Analysis. It runs the workspace's currently-loaded policies
against a historical window of flow data and reports the
disposition of each flow.

### Procedure

From the workspace's Policy Analysis page, **Run Policy
Experiments to Test Current Policies Against Past Traffic**.
You'll be asked for:

| Parameter | Notes |
|---|---|
| Experiment name | Free text |
| Duration | The size of the past window to replay (e.g. last 6 hours) |

The job takes a few minutes depending on duration. When it
finishes, the result appears in the policy selector menu and
the time-series chart updates with that experiment's flow
categories.

> **Cisco source.** [Run Policy Experiments to Test Current Policies Against Past Traffic](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#run-policy-experiments-to-test-current-policies-against-past-traffic).

### Reading Policy Experiment output

Per Cisco's [Flow Disposition](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#flow-disposition)
model, each flow is categorised:

| Result | Cisco definition |
|---|---|
| **Permitted** | Allowed by the network *and* by the analyzed policies |
| **Escaped** | Allowed by the network, but *would have been dropped* by the analyzed policies |
| **Rejected** | Dropped by the network *and* would also have been denied by the analyzed policies |

**Escaped flows are the actionable signal** — they're flows
that would break if you enforced the current policy. Cisco's
guide ([Focus on Escaped Flows initially](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#focus-on-escaped-flows-initially))
recommends triaging them first.

### When to use Policy Experiments

| Trigger | Why |
|---|---|
| After each ADM iteration | Catch obvious would-block-real-traffic problems before refining further |
| Before publishing a v\* | Sanity check that the policy you're about to publish doesn't have a wide swath of Escaped flows |
| Comparing two versions | Run an experiment with each version's policy on the same window |
| Quarterly or after big inventory shifts | Drift sanity check |

### Retention note

Per Cisco: *"Every week, the following are automatically
deleted: Workspace versions that have not been accessed for six
months and **policy experiments that have not been accessed in
the last 30 days**."*
([Automatic Deletion of Old Policy Versions](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#automatic-deletion-of-old-policy-versions))
If a particular experiment is evidence you want to keep, export
it before the 30-day clock runs out.

---

## How they fit together

```
   Authoring loop
   ──────────────────────────────────────────────────────
   Edit policy in workspace
        │
        ▼
   Quick Analysis ◄── per-flow, sub-second
        │           "what does the rule say about flow X?"
        ▼
   Policy Experiments ◄── window of past flows, minutes
        │           "what would my policy do across N hours?"
        ▼
   Live Policy Analysis ◄── continuous, against live flows
        │           (covered in 04-live-analysis.md)
        ▼
   Publish (v* → p*) → Enforcement
```

Quick Analysis is for **point queries**. Policy Experiments is
for **batch evaluation against past traffic**. Live Policy
Analysis is for **continuous evaluation against arriving
flows**. Each is the right tool for a different question.

---

## A "ready to move on to Live Analysis" gate

Before promoting from Policy Experiments to Live Policy
Analysis, the team should be able to assert:

- ✅ Policy Experiments run with a window that includes a
  complete business cycle (or the longest periodic event the
  app has).
- ✅ Escaped flows are at zero, or every non-zero Escaped flow
  has a *"yes, this should be denied"* note.
- ✅ The same configuration (window) reproduces the result on
  a re-run.
- ✅ Quick Analysis has been used to confirm individual
  questionable rules.

If green, proceed to [Live Policy Analysis](./04-live-analysis.md).

---

## Common findings (and what they usually mean)

| Finding | Usual cause | Fix |
|---|---|---|
| Inbound from monitoring host appears as Escaped | Monitoring host not in `prod-monitoring` filter | Label, or fix the filter |
| Outbound DNS Escaped | `prod-shared-dns` filter wasn't created or has wrong members | Fix filter at parent scope |
| One workload Escaped talking to peer in same tier | Cross-tier-internal flow ADM didn't propose; usually clustering / replication | Add an explicit Default Allow |
| Many Escaped from a "Unknown" consumer | Inventory gap on the consumer side | Label / scope the consumer workload, re-run |
| Quick Analysis says "no matching policy" for a flow you expected to allow | Catch-All hasn't been authored; add it | See [`../docs/04-policy-attributes.md`](../docs/04-policy-attributes.md) |

---

## See also

- [`04-live-analysis.md`](./04-live-analysis.md)
- [`05-conversations.md`](./05-conversations.md)
- [`01-review-discovered-policies.md`](./01-review-discovered-policies.md)
- Cisco: [Quick Analysis](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#quick-analysis)
- Cisco: [Run Policy Experiments to Test Current Policies Against Past Traffic](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#run-policy-experiments-to-test-current-policies-against-past-traffic)
- Cisco: [Flow Disposition](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#flow-disposition)
- Cisco: [Automatic Deletion of Old Policy Versions](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#automatic-deletion-of-old-policy-versions)
