# 02 — Policy Visual Representation

The Policy Visual Representation is the workspace's **graph
view** — workloads / clusters / inventory filters as nodes,
allowed flows as edges. It is the single fastest way to sanity-check
policy topology against your mental model of the app.

> **Cisco source.** [Manage Policy Lifecycle — Policy Visual Representation](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#policy-visual-representation).

---

## When to use it

| Use the visual when | Skip it when |
|---|---|
| You want to see "does this *look* like a 3-tier app?" at a glance | You're hunting a specific rule (use the table view + filters) |
| You're explaining policy to an app owner who doesn't think in firewall rules | You're triaging a specific blocked flow (use [`05-conversations.md`](./05-conversations.md)) |
| You're spotting cross-scope edges that shouldn't be there | You're auditing — the table view is the audit primitive, not the graph |
| You're sanity-checking that the right Catch-All ended up in place | You need an exact rule-by-rule comparison (use [Policy Diff](../enforcement/08-policy-versions.md)) |

---

## What "good" looks like

For a typical 3-tier app:

```
   ┌──────┐                ┌──────┐                ┌──────┐
   │ web  │ ───tcp/8443──► │  app │ ───tcp/5432──► │  db  │
   └──┬───┘                └──┬───┘                └──┬───┘
      │                       │                       │
      │  shared-DNS, shared-AD, monitoring, syslog    │
      └───────────────────────┴───────────────────────┘
```

A clean graph has:

- **Few inbound edges to internal tiers.** Web tier has inbound
  from clients (or an external LB); app tier has inbound from
  web only; db tier has inbound from app only.
- **Per-tier shared-service edges.** Every tier reaches DNS, AD,
  NTP, monitoring, syslog through their inventory filters.
  These edges look uniform across the app.
- **Cross-scope edges, where they exist, point to known
  counterparts** — never to "Unknown" or arbitrary CIDRs.
- **No edges from non-prod nodes to prod nodes.** If you see one,
  *that's the bug* — the Absolute Deny needs to live higher in
  the scope tree, or the inventory labelling has a stowaway.

---

## What "bad" looks like

| Pattern | Likely cause | Action |
|---|---|---|
| Edge from web tier directly to db tier (skipping app) | A misconfigured client, a debug tool that bypasses the app tier, or a flow filter wasn't applied | Investigate via [Conversations](./05-conversations.md); usually exclude the source |
| Many edges to a CIDR provider, no filter name | Inventory filter wasn't created for that destination | Create the filter, re-run ADM, refine |
| One node with edges to *everything* | Probably a monitoring / management host that wasn't excluded | Exclude or label as `application=monitoring` so it gets a clean shared-service filter |
| Bidirectional edges between two nodes that should be one-way | Either real (the relationship is bidirectional), or ADM mis-attributed; check Conversations for the SYN-initiator side |
| Cross-scope edge from a Production app to a Non-Production app | Almost certainly a label / scope error — fix immediately, do not publish |

---

## Filtering the view

The graph supports filtering by:

| Filter | Useful for |
|---|---|
| Tier | "Show only the app-tier nodes and their edges" |
| Inventory filter | "Show only the prod-shared-dns relationships" |
| Direction | Inbound only / outbound only |
| Action | Allow only / Deny only |
| Rank | Absolute / Default / Catch-All |
| Confidence | Low-confidence rules — sanity check edges before publishing |

For very large workspaces, filter aggressively before opening
the visual — full-graph rendering on a 500-rule workspace gets
unwieldy.

---

## Limitations

- The graph shows **policy-as-authored**, not flow-as-observed.
  An edge is an authored allow rule, not necessarily an actively
  used flow. To see actual flow, use [Conversations](./05-conversations.md).
- The graph does **not** simulate enforcement — for that, use
  [Quick Analysis](./03-quick-analysis.md) or
  [Live Policy Analysis](./04-live-analysis.md).
- The graph view's layout can shift between releases; export to
  CSV / JSON for reproducibility if you need a fixed artefact
  for review.

---

## Using the visual in customer reviews

Two practical tips:

1. **Open the visual at the start of any review meeting.** It
   sets shared visual context faster than walking through the
   table. Even people who can't read firewall rules can read a
   topology graph.
2. **If a node "doesn't look right" to the app owner, pause.**
   The fastest way to catch a labelling or membership bug is the
   app owner saying *"that node shouldn't be there."*

---

## See also

- [`01-review-discovered-policies.md`](./01-review-discovered-policies.md)
- [`05-conversations.md`](./05-conversations.md)
- [`06-policy-complexities.md`](./06-policy-complexities.md)
- Cisco: [Policy Visual Representation](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#policy-visual-representation)
