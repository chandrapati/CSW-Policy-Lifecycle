# 08 — Import / Export

CSW can export a workspace's policy to **JSON** or **CSV** and
re-import the same shape. This is the bridge between the cluster
UI and external tooling: review handoff, GitOps, audit
artefacts.

> **Cisco source.** [Manage Policy Lifecycle — Import / Export](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#import-export).

---

## What you can export

From a workspace:

| Export | Format | Common use |
|---|---|---|
| **Policy set** (current state) | JSON | Programmatic post-processing, GitOps |
| **Policy set** (current state) | CSV | Human review, spreadsheet handoff |
| **Inventory filters** | JSON | Reusing filter definitions across tenants / labs |
| **Clusters** | JSON | Documentation snapshot of an ADM run |
| **Conversations** | CSV | Audit / review hand-off |

The **published version** (p\*) is what you want to export for
audit; the **discovered version** (v\*) is what you want for
"this is what ADM proposed" snapshots.

---

## What you can import

Import is the inverse — a JSON / CSV in the documented schema
gets ingested into a workspace as proposed policy. Use cases:

| Import for | Why |
|---|---|
| Migrating policy across tenants (POV → prod, lab → prod) | Lab-tested rule set lifts cleanly |
| GitOps — versioning policy in git | Source of truth in git, reconcile to cluster |
| Bulk authoring — load a spreadsheet of "obvious" rules | Faster than UI clicking for 200+ rules |
| External policy generation tools | E.g., a compliance team produces policy artefacts in a separate workflow |

> Imported policies become **proposed v\*** in the target
> workspace — they are *not* automatically published or enforced.
> You still go through review, analysis, and publishing.

---

## A typical export → review → import workflow

```
   Workspace (current p*)
        │
        │ Export JSON
        ▼
   Git repo:  policies/payments-api/p17.json
        │
        │ PR + review (human / linter)
        ▼
   policies/payments-api/p18.json (with proposed changes)
        │
        │ Import JSON
        ▼
   Workspace (now has v_next reflecting p18.json's intent)
        │
        │ Quick Analysis · Live Analysis · Publish
        ▼
   Workspace at p18
```

This gives you a reviewable diff in git, a rollback path that's
just `git revert + import`, and reproducibility across
environments. See [`../api/05-gitops-pattern.md`](../api/05-gitops-pattern.md)
for the full pattern.

---

## CSV vs. JSON

Use **CSV** when a human or spreadsheet is the primary audience,
you need a flat representation, or you're handing to a tool that
consumes CSV (compliance reports, ticketing systems).

Use **JSON** when you're round-tripping (export → edit →
re-import), you need to preserve filter references and metadata
exactly, or you're integrating with code (GitOps, linters).

JSON is the lossless format. CSV is for human ergonomics.

---

## Things that don't round-trip cleanly

- **Inventory filters referenced by name.** If the target tenant
  doesn't have the same filters, the import will fail or create
  them automatically — neither is universally desired. Resolve
  by either pre-creating the filters or treating filter
  definitions as part of the JSON.
- **Cluster references.** Clusters are workspace-local; exporting
  and reimporting into a different workspace will not preserve
  cluster identity. Promote to inventory filters first.
- **Workspace identity.** Importing into a different workspace
  reassigns the policies; importing into the same workspace
  replaces / merges based on the documented import behaviour.
  Read the chapter for the exact semantics in your release.
- **Version numbers.** v\* and p\* are cluster-local; imported
  policy gets a new v\*. Don't expect "p17.json" imported to
  appear as p17.

---

## Validating an import

Before importing into a primary workspace:

1. Import into a **secondary** workspace first.
2. Run [Quick Analysis](./03-quick-analysis.md).
3. Compare to the previous published version using
   [Policy Diff](../enforcement/08-policy-versions.md).
4. Resolve any unexpected diff.
5. Promote / publish.

Skipping (1)–(4) and importing straight into the primary is the
fastest way to publish broken policy.

---

## OpenAPI vs. UI import / export

Most teams use the OpenAPI for automation (the
`/openapi/v1/policies/import` and `/export` family of
endpoints) and reserve the UI for ad-hoc / one-off cases. See
[`../api/02-openapi-policies.md`](../api/02-openapi-policies.md)
for request shapes.

---

## See also

- [`../enforcement/08-policy-versions.md`](../enforcement/08-policy-versions.md)
- [`../api/02-openapi-policies.md`](../api/02-openapi-policies.md)
- [`../api/05-gitops-pattern.md`](../api/05-gitops-pattern.md)
- Cisco: [Import / Export](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#import-export)
