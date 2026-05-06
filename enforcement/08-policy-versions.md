# 08 — Policy Versions (v\* and p\*) and Policy Diff

CSW maintains policy versions per workspace. Two version types
exist (**v\*** and **p\***), and the **p\*** type itself has
two parallel meanings depending on context. Understanding the
exact semantics matters for rollback, auditing, and GitOps.

> **Cisco source.**
> [Manage Policy Lifecycle — About Policy Versions (v\* and p\*)](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#about-policy-versions-v-and-p),
> [Managing Published (p\*) Versions](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#managing-published-p-versions),
> and [Comparison of Policy Versions: Policy Diff](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#comparison-of-policy-versions-policy-diff).

---

## v\* — Discovered Policy Version

Per Cisco's [Policy Discovery Version (v\*)](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#policy-discovery-version-v):

> *"Each time you automatically discover policies for a
> workspace, the version (v\*) increments."*
>
> *"The first time you automatically discover policies, version
> 1 is generated, and all modifications after that run, such as
> editing or approving clusters (but not a rerun), are also
> grouped under version 1. When you subsequently automatically
> discover policies, a new version is generated (unless
> discovery failed)."*
>
> *"The v\* version also increments if you import policies."*

So a v\* is the workspace's snapshot at one Automatic Policy
Discovery run, plus any subsequent edits to that run's output
(clusters, manual rule edits) that don't trigger a new run.

v\* is **never directly enforced** — it's the discovered /
authored state.

---

## p\* — Published Policy Version (two contexts)

Cisco's documentation is precise here, and worth quoting
directly because it's easy to get wrong:

> *"The term 'published' policy version (p\*) for a workspace
> can refer to **either**: The version of the policies that was
> **analyzed**, or The version of the policies that was
> **enforced**. These are two separate but parallel versions
> that depend on the context."*
>
> [About Policy Versions (v\* and p\*) — Published Policy Version (p\*)](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#about-policy-versions-v-and-p)

### p\* for analysis — increments on each analyze

> *"Each time you analyze policies in a workspace, or click
> **Analyze Latest Policies** after making a change, the system
> takes a snapshot of all the clusters and policies that are
> defined in that workspace, and the 'published' policy version
> (p\*) number for analysis increments."*

The latest Live Policy Analysis version is shown at the
top-left of the **Policy Analysis** tab of the primary
workspace.

### p\* for enforcement — IS the v\* number you chose

This is the part most people get backwards (including an
earlier version of this page):

> *"Each time you enable enforcement of the policies in a
> workspace, or enable enforcement again after making changes,
> the 'published' policy version (p\*) for enforcement
> **becomes the number of the analyzed version that you choose
> in the enforcement wizard**. So, if you enforce analyzed
> version 5, the enforced version is also version 5, even if it
> is, for example, the first time policy has been enforced for
> the workspace."*

Two consequences worth absorbing:

1. **The enforcement-p\* number is not a separate sequence.**
   It mirrors the v\* / analyzed version you picked in the
   wizard. Enforce v17 → enforced version shown as 17.
2. **Going backward is normal.** If you re-enforce v15 after
   having enforced v17 (i.e. revert), the enforced version
   simply becomes 15 again. No `p18 = same as p15` is
   synthesised.

The current Enforced Policy Version is shown at the top-left of
the **Enforcement** tab of the primary workspace.

---

## Putting it together

```
   Discovery / authoring stream                    p* for analysis stream
   ─────────────────────────────                   ────────────────────────
   v1 ─ v2 ─ v3 ─ v4 ─ v5 ─ v6 ─ ...               (separate counter,
       │                   │                        increments on each
       │ analyze v3       │ analyze v6              "Analyze Latest")
       ▼                   ▼
       p_analysis_n      p_analysis_(n+1)


   p* for enforcement stream
   ─────────────────────────
   When you enforce v5: enforced version = 5
   When you enforce v6: enforced version = 6
   When you revert to v5: enforced version = 5 again
```

### Quick reference

| Question | Answer |
|---|---|
| What does v\* increment from? | An ADM run, or a policy import |
| What does p\* (analysis) increment from? | An *Analyze Latest Policies* click (or the equivalent) |
| What is p\* (enforcement)? | The v\* number you selected in the Enable Policy Enforcement wizard |
| Can a v\* be enforced without being analyzed first? | The wizard offers analyzed versions; running analysis is part of the lifecycle. See the wizard documentation for your release. |
| Is the enforcement-p\* always the latest v\*? | Not necessarily — it's whichever v\* you chose to enforce. Reverting to an older v\* is a normal operation. |

---

## Limits and retention

Per Cisco:

> *"Published policy versions (p\*) are limited to 100 total.
> Once this limit is reached, you must delete old versions."*
>
> [Managing Published (p\*) Versions](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#managing-published-p-versions)

Auto-deletion (per [Automatic Deletion of Old Policy Versions](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#automatic-deletion-of-old-policy-versions)):

> *"Every week, the following are automatically deleted:
> Workspace versions that have not been accessed for six months
> and policy experiments that have not been accessed in the
> last 30 days."*

Practical implication: **periodically open** versions you want
to keep alive (or use the API to delete unwanted ones to stay
under 100), and **export** versions you want indefinitely (out
of cluster). See [`../analysis/08-import-export.md`](../analysis/08-import-export.md)
for the export pattern.

---

## Policy Diff

CSW provides a structured diff between any two policy versions
(v\* vs. v\*, analyzed-p\* vs. analyzed-p\*, or enforced-p\*
vs. enforced-p\*). Per Cisco's
[Comparison of Policy Versions: Policy Diff](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#comparison-of-policy-versions-policy-diff):

The diff groups changes by **rank** — Absolute, Default, and
Catch-All — and supports filtering by facet (Priority, Action,
Consumer, Provider, Port, Protocol) and by diff type (ADDED,
REMOVED, UNCHANGED).

The diff CSV output includes the columns:

| Column | Notes |
|---|---|
| Rank | `ABSOLUTE`, `DEFAULT`, or `CATCH_ALL` |
| Diff | `ADDED`, `REMOVED`, or `UNCHANGED` |
| Priority | The priority number (e.g. 100) |
| Action | `ALLOW` or `DENY` |
| Consumer Name + ID | The consumer cluster's identity |
| Provider Name + ID | The provider cluster's identity |
| Protocol | E.g. `TCP` |
| Port | E.g. 80 |

Use Policy Diff:

| When | Why |
|---|---|
| Reviewing before publish (current v\* vs. previously enforced version) | Confirm the change is what you intended |
| Reviewing before re-enforce (proposed version vs. currently enforced) | Last sanity check |
| Forensic / incident analysis | Pinpoint the rule that broke a flow |
| Audit ("what changed in this quarter?") | Diff end-of-quarter version vs. start |
| Rollback decision ("what would reverting to v15 actually do?") | Understand the blast radius |

---

## Activity Logs and Enforcement History

Two related-but-distinct artefacts:

| Feature | What's recorded | Cisco reference |
|---|---|---|
| **Activity Log** (per workspace) | Workspace-level events: ADM runs, edits, cluster approvals, version creates | [Activity Logs and Version History](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#activity-logs-and-version-history) |
| **Enforcement History** | List of changes to which workspaces are enforced, and which version each is enforcing | [Enforcement History](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#enforcement-history) |

For audit and post-incident review, you typically want both.
Operationally, see [`../operations/02-version-history.md`](../operations/02-version-history.md).

---

## See also

- [`07-modify-enforced-policies.md`](./07-modify-enforced-policies.md)
- [`09-rollback-and-revert.md`](./09-rollback-and-revert.md)
- [`../analysis/08-import-export.md`](../analysis/08-import-export.md)
- [`../operations/02-version-history.md`](../operations/02-version-history.md)
- [`../operations/04-evidence-and-audit.md`](../operations/04-evidence-and-audit.md)
- Cisco: [About Policy Versions (v\* and p\*)](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#about-policy-versions-v-and-p)
- Cisco: [Managing Published (p\*) Versions](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#managing-published-p-versions)
- Cisco: [Comparison of Policy Versions: Policy Diff](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#comparison-of-policy-versions-policy-diff)
- Cisco: [Activity Logs and Version History](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#activity-logs-and-version-history)
- Cisco: [Enforcement History](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#enforcement-history)
