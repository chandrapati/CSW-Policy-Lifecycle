# 01 — Prepare the Scope for ADM

The single biggest determinant of ADM result quality is the
**state of the scope** before you press *Run*. Spend the time
here; it's far cheaper than re-running ADM 5 times trying to
clean up garbage output.

> **Cisco source.** [Manage Policy Lifecycle — Best Practices for Creating Policies](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#best-practices-for-creating-policies)
> and [Manage Inventory for Secure Workload](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-inventory-for-secure-workload.html).

---

## Pre-flight checklist

Before you click *Run ADM*:

- [ ] Scope membership reflects the actual app boundary (no
      stragglers, no missing workloads)
- [ ] All in-scope workloads have `application`, `tier`,
      `environment` labels
- [ ] Workloads have agent → cluster status `OK` for all of the
      flow window you intend to use
- [ ] Inventory filters for shared services (DNS, AD, NTP) exist
      at the parent scope and are usable
- [ ] You're working in a **secondary** workspace, not the
      primary
- [ ] You know what *time window* you want ADM to consider

The rest of this page goes through these one at a time.

---

## Scope membership

Open *Organize → Scopes → \[your scope\]* and look at the membership
list. Common things to fix:

| Symptom | Fix |
|---|---|
| Workloads from another app appearing in the scope | Refine the scope's membership filter; add a label that uniquely identifies this app (`application=payments-api`) and use it in the filter |
| Workloads belonging to this app *missing* from the scope | Either the workload is missing the label, or the label value is inconsistent (`payments_api` vs `payments-api`). Reconcile and re-tag. |
| Decommissioned workloads still in the scope | Remove from inventory before running ADM, or add a `lifecycle=retired` label and exclude in the scope filter |

Cleaning the scope first prevents ADM from clustering ghost
workloads as their own group.

---

## Inventory labels — the ones ADM uses

ADM has access to **all** labels on a workload, but the labels
that materially affect output are:

| Label | How ADM uses it |
|---|---|
| `application` | Workloads with the same `application` value cluster together more readily |
| `tier` | The natural cluster boundary in most 3-tier apps |
| `environment` | Helps separate prod from non-prod traffic in the same scope |
| `os` | Sometimes useful for splitting Windows / Linux behaviour |
| `role` (free-form) | Useful for outlier workloads (e.g. `role=batch-runner`) that legitimately behave differently |

If two workloads share `application` and `tier` and ADM puts
them in different clusters, that's usually a real signal — they
*are* behaving differently. Investigate before forcing them
together.

> **Tip.** If your inventory only has `application` and
> `environment` (no `tier`), ADM will cluster mainly on observed
> behaviour and you'll need to add `tier` labels manually
> afterward. Adding `tier` labels *before* ADM is far less work
> than relabelling clusters after.

---

## Inventory filters for shared services

Shared services (DNS, AD, NTP, monitoring, syslog, vuln scanners)
should be expressed as **inventory filters at a parent scope**
*before* running ADM, not discovered fresh on every workspace.

Pattern:

| Filter name | Lives at | Filter expression |
|---|---|---|
| `prod-dns` | Production scope | `application=dns AND environment=prod` |
| `prod-ad` | Production scope | `application=ad AND environment=prod` |
| `prod-monitoring` | Production scope | `application=monitoring AND environment=prod` |
| `prod-syslog-targets` | Production scope | `role=syslog-target AND environment=prod` |

When ADM runs on `payments-api` and sees a flow to a DNS server,
it will resolve that flow's provider against the existing
`prod-dns` filter and produce a clean rule
*"`payments-api / any` → `prod-dns` on udp/53,tcp/53"* — not
*"`payments-api / web-01` → `10.0.0.53` on udp/53"*.

---

## Working in a secondary workspace

ADM is **iterative**. You will run it multiple times and refine
between runs. Doing this in the *primary* workspace risks
disturbing already-published policy.

```
Recommended flow:

  Primary workspace            ──── (existing policy, untouched)
  Secondary workspace (new)    ◄─── ADM runs here
       │
       │ refine, iterate
       ▼
  Promote to primary  ──────── (replaces the previous primary
                                only when you're satisfied)
```

Create the secondary with a name that signals intent:
`payments-api-adm-2026q2`, `payments-api-rerun-after-relabel`,
etc. See [`../docs/03-workspaces.md`](../docs/03-workspaces.md).

---

## Time window for ADM

ADM uses a configurable historical window of flow data. Set it
explicitly:

| Window | When |
|---|---|
| Last **7 days** | Quick first pass on a stable app you're already familiar with |
| Last **14 days** | Default for most 3-tier apps |
| Last **30 days** | Required for any app with a weekly or monthly batch |
| **Custom range** that includes a known event | Use this when you specifically want to capture a DR test, a known maintenance window, or a known peak |

Verify the window has clean flow data before running:

- *Investigate → Flows* with the same time range — confirm the
  workloads you expect are reporting flows for the entire range.
- Watch for gaps that indicate agent disconnection or
  cluster-side ingestion problems; those gaps will make ADM
  miss legitimate dependencies.

Detail in [`02-flow-collection-window.md`](./02-flow-collection-window.md).

---

## A "ready to run ADM" smoke test

Five-minute check before pressing *Run*:

1. *Organize → Scopes → \[scope\]* — workload count is what you
   expect (within ±5 %), no obvious strangers.
2. *Investigate → Inventory* with a filter on the scope — random
   sample 10 workloads; confirm `application`, `tier`,
   `environment` are populated and consistent.
3. *Investigate → Flows* with the time range you'll use — flows
   from this scope are present every day in the range.
4. *Defend → Segmentation → Inventory Filters* — shared-service
   filters (`prod-dns`, `prod-ad`, etc.) exist and resolve to
   workloads.
5. *Defend → Segmentation → Workspaces → \[your secondary
   workspace\]* — workspace exists, attached to the right
   scope, currently empty.

If all five pass, proceed to [`03-run-adm.md`](./03-run-adm.md).

---

## See also

- [`02-flow-collection-window.md`](./02-flow-collection-window.md)
- [`03-run-adm.md`](./03-run-adm.md)
- [`05-flow-filters.md`](./05-flow-filters.md) — for refining
  what ADM consumes
- Cisco: [Best Practices for Creating Policies](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#best-practices-for-creating-policies)
