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
- **Quick Analysis** against historical flows shows zero (or
  fully-explained) would-be-blocked legitimate flows.
- **Live Policy Analysis** against current flows shows zero
  (or fully-explained) would-be-blocked legitimate flows for
  ≥ 5 business days, including any periodic batch.
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
| [`03-quick-analysis.md`](./03-quick-analysis.md) | Quick Analysis (historical flow simulation) |
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
   4. Quick Analysis (file 03) — replay historical flows
      against the proposed policy
            │
            ▼
   5. Live Policy Analysis (file 04) — keep doing it as
      current flows arrive; resolve every would-be-blocked flow
            │
            ▼
   6. Publish (v* → p*) when Live Analysis is clean for ≥ 5 days
            │
            ▼
   On to enforcement/
```

---

## Quick Analysis vs. Live Analysis

These are easy to confuse, so up front:

| | Quick Analysis | Live Policy Analysis |
|---|---|---|
| Input | Historical flows from a chosen window | Currently arriving flows |
| When you'd use it | First check after ADM, after every refinement | After Quick Analysis is clean — ongoing soak test |
| Reproducible | Yes — same window → same result | No — new flows constantly |
| Does it block traffic? | No, simulation only | No, simulation only |
| Time to value | Seconds–minutes | Continuous; you watch it for days |
| Confidence level | Catches the bulk of issues | Catches the long tail (rare flows, periodic events) |

**Both** are needed before publishing. Quick Analysis is *fast
proof* the policy isn't broken. Live Analysis is *evidence over
time* that it's actually safe. Skipping Live Analysis is the
single most common cause of *"the policy passed Quick Analysis
in test, blocked something in production."*

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
