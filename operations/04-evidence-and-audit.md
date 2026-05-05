# 04 — Evidence and Audit

For each policy-relevant change you make in CSW, an auditor
will eventually ask: *"who, what, when, why, and how do I know
the policy was actually enforced?"*. This page is the discipline
that makes those answers cheap.

---

## The evidence bucket

For every workspace, maintain an **evidence bucket** — a
structured folder, S3 prefix, or git path where you accumulate
the artefacts that prove the workspace's policy lifecycle was
followed.

```
evidence/
└── <tenant>/
    └── <scope>/
        └── <workspace>/
            ├── 2026/
            │   ├── Q1/
            │   │   ├── policy-export-p4.json
            │   │   ├── live-analysis-clean-window.png
            │   │   ├── change-ticket-CR-1234.pdf
            │   │   └── enforcement-go-live-screenshot.png
            │   ├── Q2/
            │   └── ...
            └── README.md     ← workspace owner, scope, intent
```

What lives in each evidence bucket:

| Artefact | When generated |
|---|---|
| Policy export (JSON) | Each new published p\* |
| Activity Log excerpt | Each change event in the workspace |
| Live Analysis screenshot of clean window | Before each enforcement enable |
| Change ticket / PR link | Each change |
| Enforcement state screenshot at go-live | Each enable / disable / revert |
| First-30-days monitoring summary | After each enforcement enable |
| Compliance mapping snapshot | Quarterly |
| Pinned p\* references | Per the pinning policy in [`02-version-history.md`](./02-version-history.md) |

---

## Five questions an auditor will ask

For each, the corresponding evidence:

| Question | Evidence |
|---|---|
| 1. *Who authored the policy that's currently enforcing?* | Activity Log + change tickets + (if GitOps) PR history |
| 2. *Was it reviewed before being enforced?* | Change ticket approval state; PR review record |
| 3. *Was it tested before being enforced?* | Quick Analysis output; Live Analysis clean window screenshot |
| 4. *Is it actually being enforced right now?* | Enforcement state screenshot; sample of agents showing host firewall reflects p\* |
| 5. *How would you roll back if it broke something?* | The pinned previous p\*; the documented disable / revert procedure |

If you can't produce the answer to any of these inside an hour
for any production workspace, the evidence bucket needs work.

---

## Generation pattern — automate where you can

Manual evidence collection rots. Drive these from CI:

```python
# evidence_capture.py — runs on every successful publish/enforce
def capture_evidence(workspace_id, action):
    quarter = current_quarter()
    bucket = f"evidence/{tenant}/{scope}/{workspace}/{quarter}"

    if action == "publish":
        export = client.export_workspace(workspace_id)
        write(f"{bucket}/policy-export-{export.version}.json", export.body)
        write_activity_log(bucket, workspace_id, since=last_publish_time)

    if action == "enforce":
        screenshot_enforcement_state(workspace_id, bucket)
        capture_live_analysis_window(workspace_id, bucket)
        write_change_ticket_ref(bucket, ticket=os.environ["CHANGE_TICKET"])

    if action == "revert" or action == "disable":
        capture_event_record(workspace_id, bucket, action)
```

Wire it into the GitOps reconciler (per [`../api/05-gitops-pattern.md`](../api/05-gitops-pattern.md))
or the equivalent in your stack.

---

## Manual events still need evidence

Some changes happen via the UI, not via GitOps — emergency
fixes, on-call action, etc. They still need evidence:

- The Activity Log captures *what* happened automatically.
- The change ticket should capture *why* — even an emergency
  ticket opened after the fact.
- A short screenshot or text note in the evidence bucket links
  the two.

For routine UI edits in non-GitOps environments, run a daily
job that:

1. Diffs the current workspace state against last known good.
2. Captures the diff as an evidence artefact.
3. Files an open evidence ticket if the change isn't tied to an
   approved change request.

This forces eventual reconciliation.

---

## Retention

| Evidence type | Retention |
|---|---|
| Per-change artefacts | At least the audit cycle (typically 7 years for regulated industries) |
| Quarterly snapshots | At least the audit cycle |
| Pinned p\* | Indefinite (or aligned to retention policy) |
| Activity Log forwarding to SIEM | SIEM's retention policy |

Match your enterprise retention regime; CSW cluster retention is
typically *shorter* than what you'll need for audit, which is
why out-of-cluster archival matters (per [`02-version-history.md`](./02-version-history.md)).

---

## Cross-reference to compliance mappings

Each policy ties back to one or more **control objectives** in
your compliance regime. The companion repo
[`chandrapati/CSW-Compliance-Mapping`](https://github.com/chandrapati/CSW-Compliance-Mapping)
documents that mapping for the major frameworks.

The evidence bucket should include, per change:

- The control(s) the change satisfies / impacts.
- The mapping repo's commit hash at the time (so the framework
  version is unambiguous).

This connects "we changed rule X" to "control Y was thereby
maintained / strengthened / amended."

See [`06-compliance-companion.md`](./06-compliance-companion.md).

---

## Anti-patterns

| Anti-pattern | Why it fails |
|---|---|
| Evidence "in someone's email" | Not query-able, not preserved on departure, not auditable |
| Evidence "in CSW only" | Cluster retention may be shorter than audit cycle; tenant migration loses it |
| Per-change evidence but no quarterly snapshot | Reconstructing "state at a moment in time" requires replaying all events |
| Evidence without ownership | When the auditor calls, no-one knows where to look |

---

## See also

- [`02-version-history.md`](./02-version-history.md)
- [`06-compliance-companion.md`](./06-compliance-companion.md)
- [`../analysis/08-import-export.md`](../analysis/08-import-export.md)
- [`../api/05-gitops-pattern.md`](../api/05-gitops-pattern.md)
- Companion repo: [`chandrapati/CSW-Compliance-Mapping`](https://github.com/chandrapati/CSW-Compliance-Mapping)
