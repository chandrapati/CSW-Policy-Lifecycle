# 07 — Policy Templates

Policy templates are **prepackaged sets of rules** for common
patterns — Active Directory, Microsoft Exchange, common
database engines, etc. They save you from rediscovering and
manually authoring well-known port/protocol combinations every
time.

> **Cisco source.** [Manage Policy Lifecycle — Policies for Specific Purposes / Policy Templates](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#policy-templates).

---

## What templates do

A template is a named bundle of *(consumer, provider, ports,
proto, action)* tuples for a specific common use case. Apply it
to a workspace, fill in the inventory filters for your
environment, and you've got a baseline policy set without
having to remember every Active Directory port.

| Template (typical examples) | Use case |
|---|---|
| **Active Directory** | All clients to AD: TCP/UDP 88, 135, 389, 445, 464, 636, 3268, 3269, dynamic RPC |
| **Microsoft Exchange** | Client → Exchange roles, server → server replication |
| **Common databases** | Allowed inbound to db tier on engine-default ports (MySQL 3306, PostgreSQL 5432, Oracle 1521, MSSQL 1433/1434, MongoDB 27017, Redis 6379) |
| **Windows file/print** | SMB / print spool for member servers |
| **Backup tooling** | Common backup-agent flows |
| **Monitoring** | Common SNMP / NRPE / WMI / Prometheus exporter flows |

Templates available depend on release; see the [Policy Templates section](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#policy-templates)
for the current set in your version.

---

## When to use a template

| Use a template when | Don't bother when |
|---|---|
| You're authoring shared-services policy (DNS, AD, monitoring) at a parent scope | You have a small bespoke app whose flows are nothing like the templates |
| You're new to CSW and need a reasonable baseline | The template ports are wrong for your specific deployment (always verify) |
| You're modernising and want to enforce a known-good standard from day 1 | The actual flows differ enough that you'd over-allow |

Templates are **starting points**, not finished products.
Treat them as drafts: apply, then refine.

---

## A typical template-driven flow

For Active Directory at the Production scope:

1. *Defend → Segmentation → Workspaces → \[`Production`
   workspace\] → Templates → Active Directory*.
2. Apply the template to the workspace.
3. The template proposes consumer = "any prod client" and
   provider = "AD servers." Replace the placeholders with your
   inventory filters: consumer = `tier=any AND environment=prod`,
   provider = `prod-shared-ad`.
4. Review the proposed rules — port lists, ranks (most templates
   propose Default Allow).
5. Run [Quick Analysis](./03-quick-analysis.md) — does any
   currently-observed flow get rejected? If yes, tighten or
   expand the template's port list or filters.
6. Publish at the Production scope so all child app workspaces
   inherit.

Net effect: every prod app's web, app, and db tiers can talk to
AD without each app's owner having to think about Kerberos
ports.

---

## Limits of templates

- Templates encode **the vendor's documented ports**, not the
  ports your specific deployment uses. A custom AD with
  non-standard schedules / ports will need additional rules.
- Templates don't know about your **scope tree shape**. They
  apply at whichever workspace you put them in; place them
  thoughtfully (usually as high in the tree as the rule applies).
- Templates **don't replace ADM** for app-specific flows. Use
  templates for the shared / well-known surfaces; use ADM for
  the app's own internal communication.

---

## Common pitfalls

| Pitfall | Symptom | Fix |
|---|---|---|
| Applying the AD template at the wrong scope | Some apps inherit the rule, others don't | Move the template-applied rules higher in the tree |
| Leaving template placeholder filters in place | Rules with consumer / provider = "Any" — way too broad | Replace placeholders with real inventory filters before publishing |
| Treating templates as authoritative | False sense of completeness; missing app-specific flows | Always combine templates with ADM and Live Analysis |
| Not reviewing template port lists | Allowing ports the template includes "for completeness" but your app doesn't use | Trim template's proposed ports to the ones your environment actually uses |

---

## See also

- [`01-review-discovered-policies.md`](./01-review-discovered-policies.md)
- [`03-quick-analysis.md`](./03-quick-analysis.md)
- [`../docs/05-decision-matrix.md`](../docs/05-decision-matrix.md)
- Cisco: [Policies for Specific Purposes / Policy Templates](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#policy-templates)
