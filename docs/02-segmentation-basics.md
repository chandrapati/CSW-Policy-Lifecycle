# Segmentation Policy Basics

This page is the high-level mental model: *what is a segmentation
policy in CSW, what does it actually do at the host, and what
does the platform give you that a vanilla host firewall does not.*

> **Cisco source.** [Manage Policy Lifecycle in Secure Workload — Segmentation Policy Basics](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#segmentation-policy-basics).

---

## What CSW segmentation policy is

A CSW policy rule is the same logical shape as any
host-firewall rule — *consumer talks to provider, on protocol /
port, action allow or deny* — but with three things attached
that a vanilla host firewall doesn't have:

1. **Identity, not addresses.** Consumer and provider are
   *inventory filters* (label expressions like
   `environment=prod AND application=payments-api AND tier=web`)
   rather than IP addresses. The cluster keeps the IP-to-label
   mapping current as workloads move, scale, or get re-IP'd.
2. **Hierarchy / inheritance.** Policy lives in a **workspace**
   on a **scope**, and scopes form a tree. A policy on the
   `Production` scope is inherited by every child scope. This
   is how you express "deny inbound from non-prod to prod" once,
   not 800 times.
3. **Lifecycle state.** A policy is born as **discovered** (v\*)
   from ADM, gets reviewed and edited, then **published** (p\*)
   into a versioned, enforceable artefact. The version history
   is preserved; you can diff and revert.

The host enforcement is delivered to the workload's local
firewall — Linux iptables / nftables, Windows Filtering Platform
(WFP), or for Kubernetes the configured CNI / iptables — by the
CSW software agent. The cluster never sits in the data path.

---

## Five concepts that everything else builds on

| Concept | One-line | Lives in |
|---|---|---|
| **Scope** | A node in the inventory tree; the unit of RBAC | [`docs/01-prerequisites.md`](./01-prerequisites.md) |
| **Workspace** | The unit of policy management; sits on a scope | [`docs/03-workspaces.md`](./03-workspaces.md) |
| **Policy** | A single allow/deny rule with a consumer, provider, port/proto | [`docs/04-policy-attributes.md`](./04-policy-attributes.md) |
| **Inventory filter** | A label expression that resolves to a set of workloads (used as consumer or provider) | [`discovery/04-clusters-and-inventory-filters.md`](../discovery/04-clusters-and-inventory-filters.md) |
| **Cluster** | A workspace-local group of workloads ADM identified as behaving together (often becomes the basis for an inventory filter) | [`discovery/04-clusters-and-inventory-filters.md`](../discovery/04-clusters-and-inventory-filters.md) |

If any of those terms are unfamiliar, this is the page to bookmark
and the next four pages in [`docs/`](./) to read in order.

---

## What a policy looks like (logically)

```
┌──────────────────────────────────────────────────────────────┐
│  Policy                                                      │
│                                                              │
│  CONSUMER       (label expression — e.g. tier=web)           │
│       │                                                      │
│       │  protocol=tcp, port=8443                             │
│       ▼                                                      │
│  PROVIDER       (label expression — e.g. tier=app)           │
│                                                              │
│  ACTION:  ALLOW                                              │
│  RANK:    Default                                            │
│  WORKSPACE / SCOPE: payments-api on Production / BU-Retail   │
│  VERSION: v17 (discovered)  →  p4 (published)                │
└──────────────────────────────────────────────────────────────┘
```

In CSW UI nomenclature this is rendered as
*Consumer → Provider, tcp/8443, Allow* with the workspace and
rank attributes shown alongside.

The same rule expressed against IP addresses would have to be
rewritten any time a web tier instance scales out, gets replaced,
or the app is re-IP'd. The label-based form survives all of those.

---

## Where the rule actually executes

CSW does **not** sit in-line. The cluster:

1. Computes the **concrete IP-port-protocol rule set** that the
   policy resolves to *for each workload* (so workload `web-01`
   and workload `web-02` get rule sets that talk about the
   specific app-tier IPs that match `tier=app` *right now*).
2. Pushes that concrete rule set to the workload's **CSW agent**.
3. The agent installs the rules into the **local host firewall**
   — `iptables`/`nftables` on Linux, **WFP** on Windows, or the
   pod-level mechanism on Kubernetes.
4. The agent reports flow telemetry — including any **rejected**
   flows when in Enforce mode, or any **would-have-been-rejected**
   flows when in Simulate / Live Analysis — back to the cluster.

When labels change (a workload gets re-tagged, scales, moves),
the cluster re-resolves the policy and pushes the updated
concrete rule set to the agents. This recomputation is what
makes label-based policy work in practice.

> **Operational implication.** The host firewall is the
> enforcement plane. If the agent is unhealthy, the host firewall
> may reflect a stale rule set. Agent health monitoring is part
> of the day-2 picture — see
> [`enforcement/02-agent-readiness.md`](../enforcement/02-agent-readiness.md)
> and [`operations/03-troubleshooting-blocked-flows.md`](../operations/03-troubleshooting-blocked-flows.md).

---

## What CSW policy is *not*

To set expectations correctly:

- **Not a network firewall.** CSW does not sit in-line; it doesn't
  replace your perimeter, DC core, or zone-edge firewalls. It is
  east-west host-level segmentation, complementary to (not a
  replacement for) network firewalls.
- **Not a WAF / IPS.** CSW operates on flow tuple (5-tuple +
  process / package metadata where the agent reports it). It does
  not inspect payload, decrypt TLS, or detect L7 attacks. Use the
  appropriate Cisco tooling (Secure Firewall, etc.) for that.
- **Not a DLP control on its own.** CSW can enforce *who can
  reach what*, which is a strong DLP precondition, but the
  decision about whether the data flowing on an allowed channel
  is sensitive is made elsewhere.
- **Not real-time.** Policy push is fast (seconds to minutes
  depending on workspace size), but it is *not* hot-path. Don't
  treat CSW as a sub-second control loop.

---

## See also

- [`docs/03-workspaces.md`](./03-workspaces.md) — the unit of
  policy management
- [`docs/04-policy-attributes.md`](./04-policy-attributes.md) —
  policy attributes, rank, inheritance, consumer / provider
- [`discovery/`](../discovery/README.md) — how policy is born
  (ADM)
- [`analysis/`](../analysis/README.md) — how policy is reviewed
  before publishing
- [`enforcement/`](../enforcement/README.md) — how policy gets
  to the host firewall
- Cisco: [Segmentation Policy Basics](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#segmentation-policy-basics)
