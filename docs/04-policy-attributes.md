# Policy Attributes — Rank, Inheritance, Consumer / Provider

The four most-asked-about attributes of a CSW policy:
**rank**, **inheritance**, **consumer**, and **provider**. Get
these right and almost every "why is this flow allowed / blocked"
question becomes self-evident; get them wrong and CSW behaviour
will feel mysterious.

> **Cisco source.** [Manage Policy Lifecycle — About Policies](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#about-policies).

---

## Policy attributes overview

A CSW policy carries:

| Attribute | What it means |
|---|---|
| **Action** | `Allow` or `Deny` |
| **Consumer** | The set of workloads (inventory filter) that initiates the flow |
| **Provider** | The set of workloads (inventory filter) that receives the flow |
| **Service** | Protocol + port(s) — e.g. `tcp/443`, `tcp/8443,udp/53` |
| **Rank** | `Absolute`, `Default`, or `Catch-All` (controls evaluation order) |
| **Workspace** | The container the policy lives in (which sits on a scope) |
| **Version** | `v*` (discovered) or `p*` (published) |
| **Description / metadata** | Free-text — *please* fill this in for any non-obvious rule |

Reference: [Policy Attributes](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#policy-attributes).

---

## Rank — Absolute, Default, Catch-All

Rank determines **evaluation order** when multiple rules could
match the same flow. It is the single most important policy
attribute to understand.

| Rank | Purpose | Evaluation order | When you'd use it |
|---|---|---|---|
| **Absolute** | Highest-priority — overrides everything below | Evaluated **first** | Org-wide *deny* rules ("never allow non-prod to reach prod"), regulatory non-negotiables ("PII never traverses to internet") |
| **Default** | The bulk of policy — typical Allow rules | Evaluated **next** | Most application-tier policy: web → app → db, web → DNS, web → AD, etc. |
| **Catch-All** | Last-resort — what to do if no Absolute or Default rule matched | Evaluated **last** | Scope-level "deny all not explicitly allowed" or, in early stages of a rollout, "allow all not explicitly denied" |

```
   Flow arrives at host
          │
          ▼
   ┌──────────────────────┐
   │   Absolute rules     │  ← evaluated first; if match, decision is final
   └──────────┬───────────┘
              │ no match
              ▼
   ┌──────────────────────┐
   │   Default rules      │  ← evaluated next
   └──────────┬───────────┘
              │ no match
              ▼
   ┌──────────────────────┐
   │   Catch-All rule     │  ← always matches; defines the "default action"
   └──────────────────────┘
```

> **Practical pattern.** Most production tenants run with a
> *Default Catch-All = Deny* on each in-scope leaf, with Default
> *Allow* rules for the legitimate flows the app actually needs.
> Absolute *Deny* rules at parent scopes (e.g. Production)
> backstop the model: if anyone accidentally adds a Default
> Allow that crosses an environment boundary, the Absolute Deny
> wins.

Reference: [Policy Rank: Absolute, Default, and Catch-All](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#policy-rank).

---

## Inheritance through the scope tree

Policies on a parent scope are **inherited** by every descendant
scope. This is what makes CSW policy *manageable at scale*: you
don't write the same Catch-All Deny on every app workspace, you
write it once at the environment scope.

```
Production              ─── Absolute Deny: non-prod → prod
   │
   └── BU-Retail        ─── Default Allow: BU-Retail backbone
         │
         └── payments-api ─── Default Allow: web→app, app→db
                              Catch-All Deny
```

Workload `payments-api / web-01` evaluates, in order:

1. Absolute rules from the workspace itself, *and* Absolute
   rules inherited from `BU-Retail` and `Production`.
2. Default rules from the workspace, *and* Default rules
   inherited from parents.
3. The Catch-All on the workspace's own scope (typically the
   leaf), *or* an inherited Catch-All if no leaf-level Catch-All
   is defined.

> **Practical implication.** Authoring policy as deep in the
> tree as the rule's actual blast radius means parent-scope
> changes don't disrupt local flow. Only put a rule at a parent
> scope when it genuinely applies to *every* descendant.

Reference: [Policy Inheritance and the Scope Tree](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#policy-inheritance).

---

## Consumer and provider

The single most common source of *"my policy doesn't seem to
work"* is consumer / provider being swapped. CSW's convention:

- **Consumer = the side that initiates the connection.** For
  a web-tier workload calling an app-tier workload over
  `tcp/8443`, the **web tier is the consumer** (it starts the
  TCP handshake).
- **Provider = the side that receives.** The **app tier is the
  provider** in the same example.

```
       Consumer (initiator)              Provider (receiver)
   ┌──────────────────────┐  TCP SYN  ┌──────────────────────┐
   │  tier=web            │ ─────────►│  tier=app            │
   │                      │           │  port=8443/tcp       │
   └──────────────────────┘           └──────────────────────┘
```

Common gotchas:

- **Bidirectional flows.** A typical client-server flow is
  one-directional from CSW's POV (the SYN initiator is the
  consumer). The reply traffic on the same connection rides on
  the same allowed flow; no separate "return" rule is needed.
- **Health checks.** A monitoring system polling a service is
  the consumer; the service is the provider. If you see a "deny
  monitor → service" in observed flows, your policy is missing
  the monitoring consumer.
- **Database-initiated calls.** Replication / change-data-capture
  flows often have the *database* as the consumer (it initiates
  the connection to a downstream sink). This is a common surprise
  — verify with flow data, don't guess.

Reference: [About Consumer and Provider in Policies](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#consumer-provider).

---

## Putting it together — a worked example

Imagine `payments-api` with web, app, db tiers, and a Production
Catch-All Deny inherited from the parent scope.

| Rank | Action | Consumer | Provider | Service | Lives on |
|---|---|---|---|---|---|
| Absolute | Deny | `environment=non-prod` | `environment=prod` | `any` | Production |
| Default | Allow | `tier=web AND application=payments-api` | `tier=app AND application=payments-api` | `tcp/8443` | payments-api |
| Default | Allow | `tier=app AND application=payments-api` | `tier=db AND application=payments-api` | `tcp/5432` | payments-api |
| Default | Allow | `tier=any AND application=payments-api` | `application=dns AND environment=prod` | `udp/53,tcp/53` | payments-api |
| Default | Allow | `tier=any AND application=payments-api` | `application=ad AND environment=prod` | `tcp/389,tcp/636,tcp/88` | payments-api |
| Catch-All | Deny | `any` | `application=payments-api` | `any` | payments-api |

A flow from `non-prod / app-99` to `payments-api / db-01` is
denied by the Absolute rule (highest priority); a flow from
`crm-web` to `payments-api / db-01` is denied by the Catch-All
(no Default Allow matches); a flow from `payments-api / web-01`
to `payments-api / app-01` on tcp/8443 is allowed by the second
rule above.

---

## See also

- [`docs/05-decision-matrix.md`](./05-decision-matrix.md) —
  whether to author this manually or discover it via ADM
- [`discovery/04-clusters-and-inventory-filters.md`](../discovery/04-clusters-and-inventory-filters.md)
  — how the inventory filters used as consumer / provider are
  built
- [`analysis/06-policy-complexities.md`](../analysis/06-policy-complexities.md)
  — what happens when consumer and provider are in different
  scopes
- [`enforcement/08-policy-versions.md`](../enforcement/08-policy-versions.md)
  — how v\* and p\* version numbers work
- Cisco: [About Policies](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#about-policies)
