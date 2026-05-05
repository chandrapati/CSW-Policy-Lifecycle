# 06 — External Dependencies

Almost no application is fully self-contained. ADM identifies
flows that **cross out of** (or into) the workspace's scope and
surfaces them as **external dependencies**. How you handle them
determines whether your policy will be operationally correct.

> **Cisco source.** [Manage Policy Lifecycle — Discover Policies Automatically](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#discover-policies-automatically)
> *(External Dependencies sub-section)* and
> [Address Policy Complexities](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#address-policy-complexities).

---

## What an external dependency is

A flow is external if **either the consumer or the provider is
not a member of the workspace's scope**. Examples for a workspace
on `payments-api` scope:

| Flow | External? | Why |
|---|---|---|
| `payments-api / web-01` → `payments-api / app-02` | No | Both ends in scope |
| `payments-api / web-01` → `prod-shared / dns-01` | Yes | Provider is in `Production / Shared-Services / DNS` scope |
| `crm-web / web-01` → `payments-api / api-01` | Yes | Consumer is in another app's scope |
| `payments-api / batch-01` → `partner-bank-api.partner.com` (internet) | Yes | Provider is unmanaged |

ADM identifies all four kinds and asks you what to do with the
last three.

---

## Categories of external dependency, in order of preference

### 1. Cross-scope to a known counterpart filter (best)

The provider is a workload in another scope, that scope has an
inventory filter for it (e.g. `prod-shared-dns`), and the rule
naturally references that filter.

| | |
|---|---|
| Discovered policy | `payments-api / any` → `prod-shared-dns` on `udp/53,tcp/53` |
| Counterpart workspace | The DNS workspace already has a Default Allow for inbound DNS from any prod app |
| Action | Accept the rule. Both sides have what they need. |

For this to work, **the counterpart inventory filter must exist
before ADM runs.** This is why
[`01-prepare-scope.md`](./01-prepare-scope.md) emphasises
shared-service filters at parent scopes ahead of ADM.

### 2. Cross-scope to a workload that isn't a known filter

The provider is in another scope but no clean filter exists for
it. ADM proposes the rule against an ad-hoc filter (often
inventory-list-based or a CIDR).

| | |
|---|---|
| Discovered policy | `payments-api / app` → `(individual workloads list)` on `tcp/1521` |
| Action | Either (a) create a proper inventory filter on the counterpart scope and re-run ADM, or (b) replace the ad-hoc filter with the new one in the discovered policy before publishing. |

### 3. Internet / unmanaged provider

The provider is on the internet or isn't reporting flow
telemetry to CSW.

| | |
|---|---|
| Discovered policy | `payments-api / batch-01` → `partner-bank-api.partner.com` on `tcp/443` |
| Action | Decide explicitly: allow as a named external dependency (with a description), or block at egress with a more authoritative tool. CSW host firewall *can* enforce this, but FQDN-based egress policy is generally better expressed at network egress (Secure Firewall, secure web gateway). |

### 4. Cross-scope where one side is a *consumer* of yours

ADM, by default, will also discover *inbound* external
dependencies — other apps reaching into this workspace's scope.
These appear as candidate cross-scope rules where the consumer
is in another scope.

| | |
|---|---|
| Discovered policy | `crm-web / web` → `payments-api / api` on `tcp/8443` |
| Action | This rule needs to **also exist on `crm-web`'s workspace** (where `crm-web / web` is the consumer). Cross-scope policy is symmetric — see [`../analysis/06-policy-complexities.md`](../analysis/06-policy-complexities.md). |

---

## Auto-accept of cross-scope rules

The ADM run dialog has an *Auto-Accept Outgoing Policies* option.
What it does:

| Setting | Effect |
|---|---|
| **On** | When ADM proposes a cross-scope rule where this scope is the consumer (outgoing), and a counterpart filter / scope is found, accept it automatically. |
| **Off** | All cross-scope rules require explicit acceptance. |

For the **first** ADM run on a workspace, *On* is reasonable —
you can prune what shouldn't have been accepted. For
subsequent / production runs, *Off* gives more control.

---

## When external dependencies aren't recognised

Sometimes ADM marks an external endpoint as **Unknown**:

| Cause | Fix |
|---|---|
| Workload exists in inventory but isn't labelled into a scope | Add it to the appropriate scope by labelling it correctly |
| Workload genuinely isn't in CSW's view (third-party SaaS, partner host) | Create an explicit inventory filter that matches the IP / CIDR, or accept it as an internet flow |
| Workload was offline during the ADM window | Re-run ADM after the workload returns |
| The label on the remote workload is misspelled or inconsistent | Reconcile labels |

Don't publish policy that contains "Unknown" providers — every
unknown should resolve to either a known filter or a deliberate
internet-flow exception.

---

## Documenting external dependencies

In the workspace description for the discovered v\*, include a
section like:

```
External dependencies (this v*):

  prod-shared-dns         (Production / Shared-Services / DNS)        ─ accepted
  prod-shared-ad          (Production / Shared-Services / AD)         ─ accepted
  prod-monitoring         (Production / Shared-Services / Monitoring) ─ accepted
  prod-shared-syslog      (Production / Shared-Services / Syslog)     ─ accepted
  partner-bank-api.partner.com (internet, tcp/443)                    ─ accepted, see CR-1234
  Unknown 10.40.7.0/24                                                ─ deferred,
                                                                        labelling in progress
```

This is the kind of context that makes a policy reviewable and
auditable months later.

---

## See also

- [`03-run-adm.md`](./03-run-adm.md)
- [`04-clusters-and-inventory-filters.md`](./04-clusters-and-inventory-filters.md)
- [`../analysis/06-policy-complexities.md`](../analysis/06-policy-complexities.md)
  — cross-scope policy mechanics (effective consumer / provider)
- Cisco: [Address Policy Complexities](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#address-policy-complexities)
