# 02 — Version History and Activity Logs

CSW retains version history and an activity log for every
workspace. This is the audit anchor and the recovery substrate.
Treat it deliberately.

> **Cisco source.** [Manage Policy Lifecycle — Activity Logs and Version History](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#activity-logs-and-version-history)
> and [Automatic Deletion of Old Policy Versions](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#automatic-deletion-of-old-policy-versions).

---

## What's retained

| Artefact | Retained how long |
|---|---|
| **v\* (discovered)** versions | Per workspace retention; older auto-deleted unless pinned |
| **p\* (published)** versions | Per workspace retention; older auto-deleted unless pinned |
| **Activity Log** | Cluster-wide; configurable retention |
| **Pinned versions** | Indefinitely |
| **Exported JSON / CSV** | Wherever you put them (see [`../analysis/08-import-export.md`](../analysis/08-import-export.md)) |

Pinning is the lever for keeping versions you want to keep. The
auto-deletion behaviour is documented in
[Automatic Deletion of Old Policy Versions](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#automatic-deletion-of-old-policy-versions).

---

## What to pin (and why)

| Version | Pin? | Reason |
|---|---|---|
| **First published p\*** for a workspace (the baseline) | Yes | Ground truth for "what was originally rolled out" |
| **End-of-quarter / end-of-year p\*** on every workspace | Yes | Compliance audit anchor |
| **Pre-incident p\*** before any controversial change | Yes | Insurance for rollback |
| **The p\* an audit was conducted against** | Yes | Audit reproducibility |
| **The p\* associated with a major scope-tree reshape** | Yes | Reference point for any future "before / after" review |
| **Routine ADM iteration v\*** | No | Too many; export to git if interesting |

Pin sparingly — the goal is "I can always recover any version
that mattered," not "every version is preserved forever."

---

## Beyond pinning — out-of-cluster archival

Pinning protects against auto-deletion *within the cluster*.
For long-term archival (multi-year retention, off-cluster DR,
or just defence in depth):

1. **Quarterly export** of every primary workspace's current
   p\* to JSON.
2. **Commit to** a separate audit-archive git repo (or push to
   object storage) with a deterministic path:
   `csw-archive/<tenant>/<scope>/<workspace>/<YYYY-Qn>/p<NN>.json`.
3. **Sign the archive bundle** with your enterprise signing key
   if your compliance regime needs evidence of integrity.
4. **Restorable** — practised once a year — that you could
   re-import the archive into a fresh tenant.

The archive is the answer to "in 7 years, can we still produce
the policy that was in force on this date?"

---

## Activity Log — what's in it, what to do with it

The Activity Log is a cluster-side audit feed of policy events:

| Event type | Captured |
|---|---|
| Workspace created / renamed / deleted | Yes |
| ADM run started / completed | Yes |
| Policy created / updated / deleted | Yes |
| Publish (v\* → p\*) | Yes |
| Enforcement enabled / disabled | Yes |
| Pause / Resume policy updates | Yes |
| Revert | Yes |

For each event: who (user or API key), when (timestamp), what
(the operation + delta).

Operational uses:

- **Audit response** — *"who enabled enforcement on payments-api
  on 2026-04-15?"* — Activity Log answers in seconds.
- **Incident root cause** — *"what changed in the hour before
  the outage?"* — filter by workspace and time.
- **Compliance evidence** — periodic export of the Activity Log
  for the audit archive.
- **Detection** — alerts on unusual events (enforcement disabled
  on a critical workspace, a pause that's been open too long).

---

## Activity Log retention vs. SIEM forwarding

The cluster keeps the Activity Log per its retention. For
durable, query-able, long-term audit:

- **Forward to SIEM / log warehouse.** CSW supports syslog /
  webhook integrations for activity events; route them into
  your enterprise audit pipeline.
- **Keep the cluster's view authoritative for "recent" queries**;
  use the SIEM for "historical" queries.

---

## Version History — where in the UI

*Workspace → Policies → Versions* shows the v\* / p\* timeline
with publish times, authors, and quick links to:

- Policy Diff between any two versions ([`../enforcement/08-policy-versions.md`](../enforcement/08-policy-versions.md))
- The Activity Log entries for that version
- Export JSON / CSV
- Pin / unpin

Spend a few minutes here as part of the monthly ops review;
you'll catch retention surprises before they become audit
surprises.

---

## See also

- [`../enforcement/08-policy-versions.md`](../enforcement/08-policy-versions.md)
- [`04-evidence-and-audit.md`](./04-evidence-and-audit.md)
- [`../analysis/08-import-export.md`](../analysis/08-import-export.md)
- [`../api/05-gitops-pattern.md`](../api/05-gitops-pattern.md)
- Cisco: [Activity Logs and Version History](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#activity-logs-and-version-history)
- Cisco: [Automatic Deletion of Old Policy Versions](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#automatic-deletion-of-old-policy-versions)
