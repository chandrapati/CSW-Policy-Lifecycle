# Phase 1 — Policy Discovery

Discovery is where policy is **born**. Given a scope, the inventory
within it, and a meaningful window of flow data, *Application
Dependency Mapping* (ADM) clusters workloads by behaviour and
proposes a draft policy set that describes what they actually do.

> **Cisco source.** [Manage Policy Lifecycle — Discover Policies Automatically](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#discover-policies-automatically).

---

## When you're ready for this folder

You should already have:

- ✅ Agents in **Deep Visibility**, reporting flows for ≥ 14 days
  (≥ 30 for batch-heavy apps). See
  [`../docs/01-prerequisites.md`](../docs/01-prerequisites.md).
- ✅ Inventory **labelled** on at least 90 % of in-scope workloads
  with `application`, `tier`, `environment`, plus whatever else
  your scope tree references.
- ✅ A **scope tree** that reflects how policy ownership is split
  across the org (env → BU → app, plus shared services as a
  sibling).
- ✅ A **secondary workspace** on the scope you intend to
  discover policy for, so you can run ADM without disturbing any
  primary workspace policy in place.

If any of those aren't true, finish the prerequisites first.

---

## What's in this folder

| File | Purpose |
|---|---|
| [`01-prepare-scope.md`](./01-prepare-scope.md) | Final pre-flight: scope shape, inventory filters, namespace planning |
| [`02-flow-collection-window.md`](./02-flow-collection-window.md) | How long to collect flow data, what to look for, when more is needed |
| [`03-run-adm.md`](./03-run-adm.md) | Running ADM: parameters, modes, iterating |
| [`04-clusters-and-inventory-filters.md`](./04-clusters-and-inventory-filters.md) | Interpreting ADM output: clusters vs. inventory filters |
| [`05-flow-filters.md`](./05-flow-filters.md) | Include / exclude filters to refine ADM input |
| [`06-external-dependencies.md`](./06-external-dependencies.md) | Handling flows to/from outside the scope (other apps, internet, partners) |
| [`07-f5-adm.md`](./07-f5-adm.md) | F5-aware ADM: load-balanced server pools |
| [`08-discovery-anti-patterns.md`](./08-discovery-anti-patterns.md) | Common mistakes, with diagnostic / fix |

---

## The discovery workflow at a glance

```
   Pre-flight (this folder, file 01)
          │
          ▼
   Verify flow window (file 02)
          │
          ▼
   Run ADM in secondary workspace (file 03)
          │
          ▼
   Review clusters (file 04)         ◄─── usually 2–4 ADM iterations
          │
          ▼
   Apply flow filters to refine (file 05)
          │
          ▼
   Resolve external dependencies (file 06)
          │
          ▼
   Final discovered (v*) policy → on to analysis/
```

ADM is **iterative**, not one-shot. The first ADM run on a new
workspace almost always produces clusters that are too coarse,
too fine, or fragmented in surprising ways. Plan for 2–4
iterations with refinement of inventory labels, flow filters, and
ADM parameters in between.

---

## Common questions in this phase

| Question | Page |
|---|---|
| *"How long should I collect flows before I run ADM?"* | [`02-flow-collection-window.md`](./02-flow-collection-window.md) |
| *"ADM gave me one giant cluster — what now?"* | [`08-discovery-anti-patterns.md`](./08-discovery-anti-patterns.md) |
| *"ADM gave me 47 single-host clusters — what now?"* | [`08-discovery-anti-patterns.md`](./08-discovery-anti-patterns.md) |
| *"How does it handle traffic to a service in another scope?"* | [`06-external-dependencies.md`](./06-external-dependencies.md) |
| *"My app sits behind an F5 — does ADM understand that?"* | [`07-f5-adm.md`](./07-f5-adm.md) |
| *"How do clusters become reusable inventory filters?"* | [`04-clusters-and-inventory-filters.md`](./04-clusters-and-inventory-filters.md) |

---

## When discovery is "done"

Discovery is complete when:

- The discovered policy set covers **every flow** observed in the
  collection window (no orphan flows in the workspace).
- Each cluster either represents a **meaningful workload role**
  (web, app, db, cache, ingress) or has been folded into an
  inventory filter that does.
- **External dependencies** are explicit (referenced by inventory
  filter on the other side, or accepted as Catch-All exceptions).
- **No "Unknown" providers** remain — every consumer / provider
  on a discovered rule resolves to a known set of workloads or a
  known external endpoint.

When you reach that bar, move on to
[`../analysis/`](../analysis/README.md) to validate the
discovered policy against historical and current flows before
publishing.

---

## See also

- [`../docs/05-decision-matrix.md`](../docs/05-decision-matrix.md)
  — when ADM is the right tool, and when manual authoring is
- [`../analysis/`](../analysis/README.md) — what to do with
  discovered policy before publishing
- Cisco: [Discover Policies Automatically](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#discover-policies-automatically)
