# 07 — F5-Aware ADM (Load-Balanced Server Pools)

When workloads sit behind an F5 BIG-IP load balancer, naive ADM
sees flows from clients to the **VIP**, not to the actual server
pool members. CSW has explicit support for resolving F5 VIPs and
pool members so the discovered policy expresses the right
consumer / provider semantics.

> **Cisco source.** [Manage Policy Lifecycle — Automated Load Balancer Config for Automatic Policy Discovery (F5 Only)](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#automated-load-balancer-config-for-automatic-policy-discovery-f5-only).

---

## The problem F5-aware ADM solves

Without F5 awareness, observed flows look like:

```
   Client (consumer) ──tcp/443──► F5 VIP 10.20.0.10
                                     │
                                     │ (LTM SNAT or transparent)
                                     ▼
                                  Pool members
                                  payments-api/web-01..web-12 (tcp/8443)
```

ADM, looking at workload-side flow records, sees the *client →
F5 VIP* leg as an external dependency, and the *F5 → pool
member* leg as if the F5 itself were the consumer. The
discovered policy ends up with:

- A rule from `client` to a CIDR that contains the VIP IP — too
  abstract.
- A rule from `f5-snat-pool-cidr` to `tier=web` — mechanically
  correct but operationally meaningless (the *real* consumer is
  the client, not the F5).

F5-aware ADM fixes this by **resolving VIPs to their pool
members** during discovery, so the rule reads correctly:

```
   Discovered: client (consumer)  ───tcp/443──► VIP "payments-vip"
   Discovered: client (consumer)  ───tcp/443──► payments-api/tier=web
                                                 (pool members of the VIP)
```

---

## Prerequisites

- An F5 BIG-IP estate with management API access
- A **Secure Workload Ingest Appliance** running the
  [F5 connector](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/configure-and-manage-connectors-for-secure-workload.html)
  for AppFlow / IPFIX flow ingest. (Ingest is for *flow* — see
  [`chandrapati/CSW-Agent-Installation-Guide → agentless`](https://github.com/chandrapati/CSW-Agent-Installation-Guide/tree/main/agentless).)
- The F5 **load balancer config** also imported into CSW —
  this is the F5-aware ADM piece, distinct from the flow ingest.
  See the [Cisco section](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#automated-load-balancer-config-for-automatic-policy-discovery-f5-only)
  for the import procedure.

---

## How F5-aware ADM works

CSW imports from the F5:

| Object | What CSW does with it |
|---|---|
| **Virtual Server (VIP)** | Created as a logical entity in inventory, with the VIP's IP, ports, and SNI / SNI-pool relationships |
| **Pool** | Members reconciled against existing inventory; ADM uses the pool→member mapping to rewrite flow attribution |
| **iRules / SNAT pools** | Used to determine whether the client IP visible to the pool member is original or NAT'd |

During ADM, instead of producing rules with the F5 SNAT pool as
consumer, CSW substitutes the resolved upstream client identity
(consumer-side workload, where it can identify it).

---

## Limitations and edge cases

| Limitation | Workaround |
|---|---|
| **F5-only.** Other LBs (NetScaler, HAProxy, AVI, AWS NLB / ALB) are not yet supported by *automated config import*. | Author the equivalent rules manually; use ADM only to confirm member-side flow shape. |
| **TLS-terminating VIPs** can lose visibility of the original client app | Use header-based identification (e.g. `X-Forwarded-For` if your apps log it, or label workloads consistently and rely on cluster-side inventory) |
| **iRule-driven pool selection** at L7 may not be reflected if the iRule is dynamic | Manual review of the discovered rule set is essential |
| **VIPs not in the imported config** | Re-run the LB import; check API connectivity to the F5 |

---

## Operational tips

- **Re-import LB config when it changes.** A new pool member
  added on the F5 won't appear in CSW's resolution until config
  is re-imported.
- **Tag pool members with the right `application`/`tier` label**
  *before* running F5-aware ADM. The whole point of the
  resolution is to attribute flows to a meaningful identity, and
  it can only do that against a correctly labelled inventory.
- **Document VIP-to-application mapping** in the workspace
  description so reviewers can sanity-check ADM output.

---

## When F5-aware ADM is overkill

If the F5 sits behind a small, stable app and the relevant
flows are already well-understood, **manual policy authoring**
covers it more cleanly than F5-aware ADM. The automated import
is most valuable when:

- You have many VIPs in front of many applications.
- Pool membership changes frequently (autoscaling, blue/green).
- Operational ownership of the F5 and the apps is split — the
  app team and the LB team need a single source of truth that
  doesn't depend on either reading the other's config.

---

## See also

- [`03-run-adm.md`](./03-run-adm.md)
- [`06-external-dependencies.md`](./06-external-dependencies.md)
- Cisco: [Automated Load Balancer Config for Automatic Policy Discovery (F5 Only)](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#automated-load-balancer-config-for-automatic-policy-discovery-f5-only)
- Cisco: [Configure and Manage Connectors](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/configure-and-manage-connectors-for-secure-workload.html)
- Companion repo:
  [`chandrapati/CSW-Agent-Installation-Guide → agentless`](https://github.com/chandrapati/CSW-Agent-Installation-Guide/tree/main/agentless)
