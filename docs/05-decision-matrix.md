# Decision Matrix — Discover Automatically vs. Author Manually

CSW gives you two ways to populate a workspace with policy:

1. **Automatic Policy Discovery** (ADM) — let the platform
   cluster workloads and propose policy from observed flows.
2. **Manual authoring** — write rules directly, optionally on
   top of policy templates.

Most apps go through ADM. A meaningful minority are easier
authored manually. This page is the decision aid.

> **Cisco source.** [Manage Policy Lifecycle — Best Practices for Creating Policies](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#best-practices-for-creating-policies),
> [Manually Create Policies](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#manually-create-policies),
> and [Discover Policies Automatically](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#discover-policies-automatically).

---

## TL;DR

| Use ADM when | Use manual authoring when |
|---|---|
| You have ≥ 14 days of representative flow data | The flow surface is small (<10 distinct flows) |
| Communication patterns aren't fully documented anywhere | The app has a precise, documented API contract |
| The workload set is non-trivial (10+ workloads or 3+ tiers) | The "app" is one or two appliances with known protocol |
| You want consumer / provider semantics inferred from observation | You're authoring shared-services policy (DNS, AD, NTP) — pattern is well-known and consistent across all apps |
| The app's owners can't enumerate all current dependencies | Compliance requires the policy to come from a documented requirement, not from observation |

In practice most teams **use both**: ADM to bootstrap and
discover the unexpected, manual authoring to add or refine
specific rules and to write the Catch-All semantics.

---

## When ADM shines

| Scenario | Why ADM is the right call |
|---|---|
| Brownfield app whose flows aren't fully documented | ADM observes what's actually happening; the team can review and accept |
| 3-tier app with a long tail of dependencies (DNS, AD, monitoring, log shippers, vuln scanners) | The long tail is exactly what humans miss when authoring; ADM catches it |
| App being modernised — old and new versions running side by side | ADM adapts as flow patterns shift |
| A whole BU's set of apps in a POV — need to bootstrap quickly | ADM gives you a credible draft to refine, much faster than starting from blank |
| Cross-app dependencies you didn't know about | ADM surfaces them in the discovered policy and in Conversations |

---

## When manual authoring is faster / safer

| Scenario | Why manual is the right call |
|---|---|
| **Shared-services policy** (DNS, AD, NTP, monitoring, syslog) | The pattern is well-known: every app needs to reach DNS / AD / NTP / monitoring on a fixed set of ports. ADM rediscovers this for every workspace; author once at a parent scope and inherit. |
| **HA / DR pairs with planned-failover-only flows** | The flow only happens during DR test or failover; ADM run *after* a failover catches it, but a stable manual rule is more trustworthy. |
| **Multicast-heavy apps** (vMotion, clustering, some HFT) | Multicast and broadcast aren't always cleanly clustered by ADM; manual is sometimes clearer. |
| **Compliance-required denies** | "Workloads carrying PII MUST NOT egress to the internet" is a documented obligation, not an observation. Author as Absolute Deny. |
| **Net-new app being deployed greenfield** | There are no flows to observe yet. Author from the API contract and adjust as the app comes online. |
| **Very small surface** | A two-VM appliance with 3 well-known flows is faster to author than to ADM. |

---

## What about templates?

CSW ships [policy templates](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#policy-templates)
for common patterns (e.g. Active Directory, Microsoft Exchange,
common databases). Treat templates as *manual authoring with
training wheels*: they pre-populate the well-known flow set so
you don't have to remember every Active Directory port. Use them
liberally for shared services. See
[`analysis/07-policy-templates.md`](../analysis/07-policy-templates.md).

---

## A typical hybrid pattern

For a real app rollout, the pragmatic flow is:

1. **Author the Absolute Deny rules at parent scopes manually.**
   These are policy obligations, not observations.
2. **Use templates for shared services** (DNS, AD, NTP, the
   monitoring stack) at the appropriate parent scope so every
   app inherits.
3. **Run ADM on each application workspace** to discover
   app-specific flows.
4. **Review and refine** what ADM produced — add descriptions,
   collapse over-specific rules, expand consumer / provider where
   the inferred filter is too narrow.
5. **Author the Catch-All explicitly** (don't rely on ADM to
   produce a sensible Catch-All; decide deliberately whether the
   default is *Deny* or *Allow*).
6. **Publish (v → p)** and run Live Policy Analysis before
   enabling enforcement.

---

## The decision in one diagram

```
            ┌──────────────────────────────────┐
            │  Need policy for this scope?     │
            └──────────────┬───────────────────┘
                           │
                           ▼
       ┌───────────────────────────────────────────────┐
       │ Is it a shared service (DNS, AD, NTP,         │
       │ monitoring, syslog)?                          │
       └───────┬───────────────────────────────┬───────┘
               │ yes                           │ no
               ▼                               │
        Manual + template                      │
        at parent scope, inherit               │
                                               ▼
                                ┌─────────────────────────────┐
                                │ Compliance/regulatory rule  │
                                │ that's a documented "MUST"? │
                                └───────┬─────────────────────┘
                                        │ yes        │ no
                                        ▼            │
                                Manual Absolute Deny │
                                at relevant scope    │
                                                     ▼
                                       ┌──────────────────────────┐
                                       │ Have ≥14d flow data, or  │
                                       │ ≥10 workloads / 3+ tiers?│
                                       └──┬─────────────────┬─────┘
                                          │ yes             │ no
                                          ▼                 ▼
                                   ADM                 Manual or template
                                   then refine
```

---

## See also

- [`discovery/03-run-adm.md`](../discovery/03-run-adm.md) —
  running ADM
- [`analysis/07-policy-templates.md`](../analysis/07-policy-templates.md)
  — using policy templates
- [`docs/04-policy-attributes.md`](./04-policy-attributes.md) —
  rank, inheritance, consumer / provider
- Cisco: [Best Practices for Creating Policies](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#best-practices-for-creating-policies)
