# 03 — Running ADM (Automatic Policy Discovery)

This is the core discovery action: *take a scope and a flow window,
get a draft policy set out the other side.*

> **Cisco source.** [Manage Policy Lifecycle — Discover Policies Automatically](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#discover-policies-automatically).

---

## Where in the UI

*Defend → Segmentation → Workspaces → \[your secondary
workspace\] → Automatic Policy Discovery → Run.*

The Run dialog asks for the time window, flow filters, and
ADM-specific parameters described below.

---

## ADM parameters that matter

CSW exposes a number of knobs. The handful that materially
affect output:

| Parameter | What it does | Default |
|---|---|---|
| **Time Window** | Range of flow data ADM considers | Last 14 days (override per [`02-flow-collection-window.md`](./02-flow-collection-window.md)) |
| **External Dependencies** | Whether to discover policy for flows that cross out of this scope | On (almost always wanted) |
| **Cluster Granularity** | How aggressive the clustering is — fewer / larger clusters vs. more / smaller | Default; only adjust after at least one run |
| **Auto-accept Outgoing Policies** | Auto-accept policy where this scope is the consumer to a known external scope | On for first pass — easier to delete than to add |
| **Member Workloads Required for Cluster** | Minimum workloads that must share behaviour to form a cluster | Default ≈ 2; raise for very large scopes if you're getting too many tiny clusters |
| **Confidence Threshold** | Minimum confidence ADM needs before it proposes a rule | Default; raise if the discovered set has too many low-confidence noise rules |

> **First-run rule of thumb.** Run with defaults. Look at what
> ADM produced. *Then* iterate parameters based on the kind of
> output you got. Tuning blind ahead of the first run is
> guesswork.

---

## Running ADM and reading the result

When ADM completes, the workspace will show:

| Section | What to look at first |
|---|---|
| **Clusters** | Did ADM produce ~ the number of clusters you expected? Tier-by-tier (web/app/db) is typical. See [`04-clusters-and-inventory-filters.md`](./04-clusters-and-inventory-filters.md). |
| **Discovered Policies** | The proposed allow rules. Each has consumer / provider / port / proto / confidence. |
| **External Dependencies** | Flows ADM saw that crossed out of this scope. Each is a candidate cross-scope rule. |
| **Conversations** | The raw observed flows that drove the discovered policy. Useful for "why did ADM propose this?" questions. See [`../analysis/05-conversations.md`](../analysis/05-conversations.md). |

The discovered policy is now **v1** in the workspace's version
history. Subsequent ADM runs produce v2, v3, etc. *Nothing is
yet published or enforceable* — these are draft proposals.

---

## The iteration loop

Almost every workspace goes through 2–4 ADM iterations. The
typical iteration:

```
   ADM run N
      │
      ▼
   Review clusters and discovered policies
      │
      ▼
   Spot a problem (one of the patterns below)
      │
      ▼
   Apply fix:
     - relabel inventory  (if labels were the issue)
     - tweak scope membership  (if scope was wrong)
     - add or refine inventory filters  (shared services)
     - add a flow filter  (if a noisy flow shouldn't drive policy)
     - adjust an ADM parameter  (last resort, not first)
      │
      ▼
   ADM run N+1
```

Common iteration triggers:

| Symptom | Most likely fix |
|---|---|
| One giant cluster covering all tiers | Inventory missing `tier` labels — add them, re-run |
| Web tier split into 3 mini-clusters | Some web nodes carry an extra label that's distinguishing them; reconcile labels |
| Database cluster contains a monitoring host | Monitoring host is missing `application=monitoring` label — add it |
| Proposed rule includes a one-off load-test flow | Add flow filter excluding the load tester |
| External dependencies includes "Unknown" | The remote workload is unmanaged or not labelled — add it to inventory or carve out a manual rule |

---

## When ADM is converged

ADM has converged on a usable result when:

- The cluster set matches your mental model of the app (or
  your mental model has been updated by something ADM showed
  you that was real).
- Every discovered rule has a sensible consumer and provider —
  no "Unknown" providers.
- External dependencies are either represented as cross-scope
  rules to known counterpart filters, or have been deliberately
  excluded.
- A second consecutive ADM run with the same parameters produces
  effectively the same output (modulo flow window roll).

When you reach that bar, move to
[`../analysis/`](../analysis/README.md) — review and analyse
before publishing.

---

## ADM in the API

The same ADM run can be triggered via OpenAPI for automated
pipelines. See
[`../api/02-openapi-policies.md`](../api/02-openapi-policies.md)
for the request shape. Common use case: nightly ADM run as a
sanity check on whether observed flow has drifted from published
policy.

Reference: [Secure Workload OpenAPIs](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/secure-workload-openapis.html).

---

## See also

- [`04-clusters-and-inventory-filters.md`](./04-clusters-and-inventory-filters.md)
  — interpret what ADM produced
- [`05-flow-filters.md`](./05-flow-filters.md) — refine ADM's
  input
- [`06-external-dependencies.md`](./06-external-dependencies.md)
  — handle cross-scope flows
- [`08-discovery-anti-patterns.md`](./08-discovery-anti-patterns.md)
  — what to do when ADM's first run is a mess
- Cisco: [Discover Policies Automatically](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#discover-policies-automatically)
