# 08 — Policy Versions (v\* and p\*) and Policy Diff

CSW maintains two distinct version streams per workspace:
**v\*** (discovered) and **p\*** (published). Understanding the
difference is essential for anything involving rollback,
auditing, GitOps, or change attribution.

> **Cisco source.** [Manage Policy Lifecycle — About Policy Versions (v\* and p\*)](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#about-policy-versions)
> and [Comparison of Policy Versions: Policy Diff](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#comparison-of-policy-versions-policy-diff).

---

## v\* — Discovered (or Authored) Policy Version

| | |
|---|---|
| Created when | An ADM run completes, or when a policy edit is made |
| Represents | "This is what the workspace's policy *could* be" |
| Cardinality | One per ADM run / edit; v1, v2, v3, … |
| Enforceable? | **No** — v\* is never directly pushed to agents |
| Lifespan | Retained for some time (release-dependent); older versions auto-deleted by retention policy |

You iterate freely on v\* — every ADM iteration, every manual
tweak, every rule cleanup is a new v\*.

---

## p\* — Published Policy Version

| | |
|---|---|
| Created when | You explicitly *Publish* a v\* | 
| Represents | "This is the canonical policy for this workspace" |
| Cardinality | Strictly increasing; p1, p2, p3, … |
| Enforceable? | **Yes** — only p\* versions can be set as the enforced version |
| Lifespan | Retained per the workspace's retention policy; pinned versions can be kept indefinitely |

Publishing is the deliberate act that elevates a v\* to
canonical status. Each publish should correspond to a real
intent (a change ticket, an ADM run reviewed and accepted, a
rollback to an earlier baseline).

---

## The relationship in one diagram

```
   (ADM runs, manual edits)
   v1 → v2 → v3 → v4 → v5 → v6 → v7 → ...
                     │              │
                     │ publish      │ publish
                     ▼              ▼
                     p1             p2
                     │              │
                     │ enforce      │ enforce p2
                     ▼              ▼
                  enforced       enforced
                  (briefly)      (current)
```

Notes:

- **Most v\* never become p\*.** That's normal — many ADM /
  edit iterations don't result in a new published version.
- **v\* numbering and p\* numbering are independent.** You may
  publish v3 as p1 and v17 as p2. The p\* number is its
  position in the published sequence.
- **Only one p\* is the *current enforced* version at a time.**
  Older p\* versions are retained for diff and rollback.

---

## Policy Diff

CSW provides a structured diff between any two policy versions
(v\* vs. v\*, p\* vs. p\*, or v\* vs. p\*). Open *Workspace →
Policies → \[version selector\] → Compare*.

A diff highlights:

| Diff type | What it shows |
|---|---|
| Added | Rules in version B that aren't in A |
| Removed | Rules in version A that aren't in B |
| Modified | Same rule (same consumer / provider / port / proto) but with changed action, rank, or description |
| Identical | Same in both — collapsed by default |

Use Policy Diff:

| When | Why |
|---|---|
| Reviewing before publish (v\* vs. previous p\*) | Confirm the change is what you intended |
| Reviewing before enforce (this p\* vs. currently enforced p\*) | Last sanity check |
| Forensic / incident analysis ("what changed between p11 and p12?") | Pinpoint the rule that broke the flow |
| Audit ("what changed in this quarter?") | Diff p\* at end of quarter vs. start |
| Rollback decision ("what would reverting from p12 to p11 actually do?") | Understand the blast radius |

---

## Pinning and retention

CSW retains policy versions per the workspace / tenant
retention policy. **Periodically pin** important versions:

| Version | Pin? |
|---|---|
| The very first published version (the policy baseline) | Yes |
| Each end-of-quarter / end-of-year published version | Yes (compliance audit anchor) |
| Each version associated with a major audit or compliance assessment | Yes |
| The version preceding a controversial / high-stakes change | Yes (rollback insurance) |
| Each ADM iteration's discovered v\* | Usually no — too many; export the interesting ones |

Pinning prevents the retention process from auto-deleting the
version. Pin sparingly; an unpinned-but-meaningful version can
be exported (see [`../analysis/08-import-export.md`](../analysis/08-import-export.md))
for permanent off-cluster archival.

Reference: [Managing Published (p\*) Versions](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#managing-published-p-versions)
and [Automatic Deletion of Old Policy Versions](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#automatic-deletion-of-old-policy-versions).

---

## Activity Logs and Version History

Every workspace also has an **Activity Log** that records *who
did what, when* — version creates, publishes, enforcement
toggles, etc. This is your audit trail for the workspace.

The [Activity Logs and Version History](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#activity-logs-and-version-history)
section is the reference. Operationally, see
[`../operations/02-version-history.md`](../operations/02-version-history.md).

---

## See also

- [`07-modify-enforced-policies.md`](./07-modify-enforced-policies.md)
- [`09-rollback-and-revert.md`](./09-rollback-and-revert.md)
- [`../analysis/08-import-export.md`](../analysis/08-import-export.md)
- [`../operations/02-version-history.md`](../operations/02-version-history.md)
- [`../operations/04-evidence-and-audit.md`](../operations/04-evidence-and-audit.md)
- Cisco: [About Policy Versions (v\* and p\*)](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#about-policy-versions)
- Cisco: [Comparison of Policy Versions — Policy Diff](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#comparison-of-policy-versions-policy-diff)
