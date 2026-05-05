# 01 — Policy Drift

Policy that was correct on Day 0 may not be correct on Day 90.
Apps change, dependencies change, partners change. Drift is
inevitable. Detection and timely remediation is the goal.

---

## Sources of drift

| Source | Example |
|---|---|
| **Application change** | New microservice deployed; new dependency on a Redis cluster the policy doesn't allow |
| **Infrastructure change** | New monitoring tool rolled out; existing rules don't allow it |
| **Partner / external change** | Partner moves API to a new endpoint; old allow rule references the old endpoint |
| **Inventory drift** | Workloads relabelled inconsistently; inventory filter membership shifts unexpectedly |
| **Scope-tree change** | Reorganisation; workloads moved between scopes; inherited rules no longer apply |
| **People drift** | Original policy author moves teams; new owner doesn't know why a rule exists |

The first three produce *would-be-rejected* flows visible in
Live Analysis. The latter three produce *silent* drift that
only surfaces in audit or incident.

---

## Detection mechanisms

### Continuous — would-be-rejected on enforced workspace

Every enforced workspace should run [Live Policy Analysis](../analysis/04-live-analysis.md)
or its post-enforcement equivalent (Enforcement Reporting)
indefinitely. New rejected flows that aren't explained by an
existing change ticket are drift.

### Weekly — Conversations review

Open the workspace's [Conversations](../analysis/05-conversations.md)
table; filter to "new in the last 7 days." New conversations
that aren't expected are usually drift candidates.

### Monthly — agent vs. cluster reconciliation

Pick a sample of agents; verify the host firewall reflects the
workspace's current p\*. Mismatch means either agent push
issue or someone modifying host firewall out-of-band.

### Quarterly — re-baseline ADM

Run ADM on a fresh secondary workspace with a recent flow
window. Diff the discovered policy against the current
published policy. Differences are drift candidates.

### GitOps drift detection

If you're on the GitOps pattern (per [`../api/05-gitops-pattern.md`](../api/05-gitops-pattern.md)),
the reconciler does a nightly diff between git and CSW state.
Any non-zero drift gets surfaced as a PR or alert.

---

## Triage decision tree

```
   New rejected flow / new conversation
            │
            ▼
   Is it tied to a known change?
            │
            ├── Yes ──► Update policy via standard change flow
            │           ([`../enforcement/07-modify-enforced-policies.md`](../enforcement/07-modify-enforced-policies.md))
            │
            └── No  ──► Is it expected app behaviour?
                              │
                              ├── Yes ──► Add an Allow with description
                              │           tying back to the app
                              │           change. Investigate why
                              │           change went unnoticed.
                              │
                              └── No  ──► Investigate as a security
                                          concern. The policy is
                                          probably correct; the
                                          flow is suspect.
```

---

## Common drift findings and fixes

| Finding | Fix |
|---|---|
| New microservice version triggers new dependency | App owner files change ticket; new rule added; standard rollout |
| New monitoring host added with new label | Reconcile labels (should be `application=monitoring` so it joins `prod-monitoring` filter); existing rules cover it once labelling is right |
| Partner moves to a new IP / FQDN | Update inventory filter for the partner; existing rule continues to work |
| Workload relabelled inconsistently | Reconcile labels — usually a CMDB / connector issue; investigate at source |
| Scope-tree reorganisation | Move policy down to where the workloads now live; remove from where they used to be |
| Rule whose author left the team | Document the rule's purpose now (whoever inherits should be able to explain it); audit periodically |

---

## Anti-pattern: rubber-stamping drift

A trap to avoid: *"new rejected flow, must mean we need a new
Allow."* Sometimes the rejection is exactly right and the flow
is suspect. Default to *investigate*, not *allow*. The fast
path of "always rubber-stamp drift as legitimate" is how
zero-trust degrades into permissive-trust over time.

---

## Drift budget

For each workspace, define a **drift budget** — how many
unresolved drift findings are tolerable before remediation
becomes a priority:

| Workspace category | Drift budget |
|---|---|
| Critical revenue path | 0 unresolved findings older than 7 days |
| Standard production | ≤ 3 unresolved findings older than 14 days |
| Internal / non-customer-facing | ≤ 5 unresolved findings older than 30 days |
| Lab | n/a |

Tracked in your normal ticketing system; reviewed in monthly
ops review.

---

## See also

- [`02-version-history.md`](./02-version-history.md)
- [`03-troubleshooting-blocked-flows.md`](./03-troubleshooting-blocked-flows.md)
- [`../analysis/04-live-analysis.md`](../analysis/04-live-analysis.md)
- [`../analysis/05-conversations.md`](../analysis/05-conversations.md)
- [`../enforcement/11-monitoring-after-enforcement.md`](../enforcement/11-monitoring-after-enforcement.md)
- [`../api/05-gitops-pattern.md`](../api/05-gitops-pattern.md)
