# Phase 2 — Policy Analysis

You arrive here with a **discovered** (v\*) policy in a secondary
workspace. The job in this phase is to make sure that policy
is **safe to publish and enforce** — that it covers the legitimate
flows, doesn't contain rules that would block them, and resolves
the inevitable cross-scope and priority complexities cleanly.

> **Cisco source.** [Manage Policy Lifecycle — Review and Analyze Policies](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#review-and-analyze-policies).

---

## What "ready to publish" looks like

When this phase is complete:

- Every discovered rule has been reviewed and either kept,
  refined, or removed by a human.
- **Quick Analysis** has been used to check key hypothetical
  flows against the proposed policy and the matching rule is
  what you intended.
- **Policy Experiments** replayed past traffic against the
  proposed policy with zero (or fully-explained) Escaped flows.
- **Live Policy Analysis** against current flows shows zero
  (or fully-explained) Escaped flows for a window that includes
  a complete business cycle (a recommended baseline is ≥ 5
  business days, including any periodic batch — your team's
  threshold may differ).
- All cross-scope rules are reflected on **both sides** (this
  workspace and the counterpart).
- Catch-All semantics have been **explicitly authored** (you
  decided whether the default action is Allow or Deny — you
  didn't inherit it by accident).
- The workspace has been **published (p\*)** and is ready for
  the [`../enforcement/`](../enforcement/README.md) phase.

---

## What's in this folder

| File | Purpose |
|---|---|
| [`01-review-discovered-policies.md`](./01-review-discovered-policies.md) | How to read what ADM produced, line by line |
| [`02-policy-visual.md`](./02-policy-visual.md) | The Policy Visual Representation: when to use it, what to look for |
| [`03-quick-analysis.md`](./03-quick-analysis.md) | Quick Analysis (single hypothetical flow) and Policy Experiments (replay past traffic) — two distinct Cisco features |
| [`04-live-analysis.md`](./04-live-analysis.md) | Live Policy Analysis (current flow simulation, the gold standard) |
| [`05-conversations.md`](./05-conversations.md) | Conversations table — the raw flow evidence behind every rule |
| [`06-policy-complexities.md`](./06-policy-complexities.md) | Priorities, cross-scope, effective consumer / provider |
| [`07-policy-templates.md`](./07-policy-templates.md) | Policy templates for common patterns (AD, Exchange, databases) |
| [`08-import-export.md`](./08-import-export.md) | Import / export of policy (JSON / CSV); used for review handoff and GitOps |

---

## The analysis workflow

```
   Discovered (v*) policy in secondary workspace
            │
            ▼
   1. Walk the discovered policy (file 01)
            │
            ▼
   2. Visualise (file 02) — sanity check the topology
            │
            ▼
   3. Resolve complexities (file 06)
      - cross-scope rules — counterpart consistency
      - effective consumer / provider — make sure each rule says
        what you mean
      - priorities — Absolute / Default / Catch-All correctness
            │
            ▼
   4. Quick Analysis + Policy Experiments (file 03) —
      hypothetical flow checks and historical replay against
      the proposed policy
            │
            ▼
   5. Live Policy Analysis (file 04) — keep doing it as
      current flows arrive; resolve every Escaped flow
            │
            ▼
   6. Analyze Latest Policies, then enforce via the Policy
      Enforcement Wizard when Live Analysis is clean for the
      window your team has agreed on (e.g. ≥ 5 business days
      including any periodic batch)
            │
            ▼
   On to enforcement/
```

---

## Quick Analysis vs. Policy Experiments vs. Live Analysis

Three distinct Cisco-named tools at this stage. They're easy to
confuse:

| | Quick Analysis | Policy Experiments | Live Policy Analysis |
|---|---|---|---|
| Input | A single **hypothetical flow** you specify | A window of past traffic | Currently arriving flows |
| Cisco source | [Quick Analysis](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#quick-analysis) | [Policy Experiments](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#run-policy-experiments-to-test-current-policies-against-past-traffic) | [Live Policy Analysis](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#live-policy-analysis) |
| Scope | Primary workspace only; not on Kubernetes service flows | Per workspace; bounded by selected duration | Per workspace; continuous |
| Time to value | Sub-second per query | Minutes (depending on duration) | Continuous; days of observation |
| Reproducible | Yes — same query → same answer | Yes — same window → same result | No — new flows constantly |
| Does it block traffic? | No (simulation only) | No (simulation only) | No (simulation only) |
| What it tells you | "Which rule decides flow X?" | "What would my policy do across the past N hours?" | "What is my policy doing right now, continuously?" |

All three sit *before* enforcement. Use them in combination:
Quick Analysis for point-debugging, Policy Experiments for
batch evaluation, Live Analysis for the soak test.

Detail in [`03-quick-analysis.md`](./03-quick-analysis.md) and
[`04-live-analysis.md`](./04-live-analysis.md).

---

## See also

- [`../discovery/`](../discovery/README.md) — where the policy
  came from
- [`../enforcement/`](../enforcement/README.md) — where the
  policy is going next
- [`../docs/04-policy-attributes.md`](../docs/04-policy-attributes.md)
  — rank, inheritance, consumer / provider
- Cisco: [Review and Analyze Policies](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#review-and-analyze-policies)
