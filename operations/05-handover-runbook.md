# 05 — Handover Runbook

The steady-state operational runbook for a CSW deployment in
production. Use as a handover document when a new
admin / on-call / app team takes over CSW responsibilities.

---

## What you're inheriting

A list of things the receiver should be able to point to:

- [ ] **Tenant URL** and how to access it (SSO group, MFA).
- [ ] **API key inventory** — which keys exist, what they can do,
      where their secrets live, when they expire.
- [ ] **Connectors inventory** — vCenter, F5, AnyConnect, ISE,
      cloud connectors. Each with credentials location and
      health.
- [ ] **Scope tree** — top-level scopes, who owns each.
- [ ] **Workspace inventory** — every primary workspace, the
      app it covers, the owner, and where it sits in the
      Monitor → Simulate → Enforce rollout (see
      [`../enforcement/04-rollout-pattern.md`](../enforcement/04-rollout-pattern.md)
      for the methodology and how it maps to Cisco's
      named features).
- [ ] **Inventory filter inventory** — the reusable filters,
      what they mean.
- [ ] **GitOps repo location** (if applicable), branch protection
      status.
- [ ] **Evidence bucket location** (per [`04-evidence-and-audit.md`](./04-evidence-and-audit.md)).
- [ ] **Compliance mapping repo** ([`chandrapati/CSW-Compliance-Mapping`](https://github.com/chandrapati/CSW-Compliance-Mapping))
      and the active framework version pinned to.
- [ ] **Cisco TAC contract** and account-team contact.

---

## Day-to-day

| Frequency | Activity | Owner |
|---|---|---|
| Continuous | Live Policy Analysis on enforced workspaces | Platform / SRE |
| Daily | Scan Live Analysis for new rejected flows | Platform / SRE |
| Daily | Triage drift findings older than the workspace's drift budget | Platform / SRE |
| Weekly | Review new Conversations on key workspaces | App owners (with platform support) |
| Monthly | Spot-check host firewall on sample agents | Platform / SRE |
| Monthly | Capture evidence bucket snapshot | CSW admin / compliance |
| Quarterly | Pin end-of-quarter p\* on every primary workspace | CSW admin |
| Quarterly | Cross-check compliance mappings | Compliance |
| Annually | Re-baseline ADM run | App owners + platform |

---

## Incident response

### "A flow is broken and we think it's CSW" — see [`03-troubleshooting-blocked-flows.md`](./03-troubleshooting-blocked-flows.md)

```
   1. Confirm the symptom (Step 0).
   2. Confirm CSW is enforcing on the path (Step 1).
   3. Look at the flow in *Investigate → Flows* (Step 2).
   4. Identify the blocking rule (Step 3).
   5. Mitigate as needed (Step 4).
   6. Document (Step 5).
```

### "Production is broken and we don't have time to debug" — see [`../enforcement/10-pause-and-emergency-disable.md`](../enforcement/10-pause-and-emergency-disable.md)

```
   1. Open the affected workspace's Enforce view.
   2. Click Disable Enforcement.
   3. Confirm.
   4. Wait for application recovery.
   5. Engage triage with full info — only after recovery.
```

If you have the API-driven disable wired in (per [`../api/03-enforcement-toggle-api.md`](../api/03-enforcement-toggle-api.md)),
the runbook step 1–3 collapses to one chatops command.

### "A bad policy was published" — see [`../enforcement/09-rollback-and-revert.md`](../enforcement/09-rollback-and-revert.md)

```
   1. Identify the last good p* (Activity Log + Policy Diff).
   2. Workspace → Enforce → Revert to Earlier Version.
   3. Confirm; verify; document.
```

### "An agent is misbehaving" — see [`../enforcement/02-agent-readiness.md`](../enforcement/02-agent-readiness.md)

```
   1. Manage → Agents → drill into the host.
   2. Check status, last check-in, version.
   3. Pull agent logs.
   4. Open TAC if needed; consider removing from enforcement
      (toggle agent type to Visibility) while investigating.
```

---

## Common changes

### "We're onboarding a new application"

```
   1. Confirm scope tree placement and ownership.
   2. Install agents on the workloads (per
      [`chandrapati/CSW-Agent-Installation-Guide`](https://github.com/chandrapati/CSW-Agent-Installation-Guide)).
   3. Validate flow telemetry for ≥ 1 representative business
      cycle.
   4. Run ADM in a secondary workspace ([`../discovery/`](../discovery/README.md)).
   5. Iterate on the policy ([`../analysis/`](../analysis/README.md)).
   6. Roll out via Monitor → Simulate → Enforce
      ([`../enforcement/04-rollout-pattern.md`](../enforcement/04-rollout-pattern.md)).
   7. Capture evidence ([`04-evidence-and-audit.md`](./04-evidence-and-audit.md)).
```

### "We're modifying an enforced policy" — see [`../enforcement/07-modify-enforced-policies.md`](../enforcement/07-modify-enforced-policies.md)

```
   1. Edit in workspace (creates new v*).
   2. Quick Analysis on the change.
   3. Live Analysis for ≥ 4h.
   4. Publish (v* → p*).
   5. Re-enforce in Simulate first (if change is non-trivial).
   6. Then Enforce.
   7. Verify ([`../enforcement/06-verify-enforcement.md`](../enforcement/06-verify-enforcement.md))
      and capture evidence.
```

### "We're decommissioning an application"

```
   1. Disable enforcement on the workspace.
   2. Wait the configured cool-down period.
   3. Confirm no flows from / to the application's workloads.
   4. Uninstall agents.
   5. Delete the workspace (after evidence-bucket archival).
   6. Tidy up inventory filters that no longer have any users.
```

---

## Critical contacts

Maintain — and keep up to date — the contact list:

| Role | Person | Reachable how |
|---|---|---|
| Cisco account team | (name) | (email / phone) |
| Cisco TAC primary contract | (number) | https://www.cisco.com/c/en/us/support/index.html |
| CSW admin (this team) | (name) | (chatops channel) |
| Each app's owner | maintained per workspace | (each workspace's README) |
| Security / compliance lead | (name) | (chatops channel) |
| Platform / SRE on-call | rotation | (paging system) |

---

## What to read first when picking this up

In order:

1. The top-level [`README.md`](../README.md) — repo orientation.
2. [`../INDEX.md`](../INDEX.md) — quick-reference by question.
3. [`../docs/02-segmentation-basics.md`](../docs/02-segmentation-basics.md)
   — the mental model.
4. [`../docs/03-workspaces.md`](../docs/03-workspaces.md) and
   [`../docs/04-policy-attributes.md`](../docs/04-policy-attributes.md)
   — how policy is structured.
5. [`../enforcement/04-rollout-pattern.md`](../enforcement/04-rollout-pattern.md)
   — the safe-rollout pattern, the heart of the practice.
6. [`../enforcement/10-pause-and-emergency-disable.md`](../enforcement/10-pause-and-emergency-disable.md)
   and [`../enforcement/09-rollback-and-revert.md`](../enforcement/09-rollback-and-revert.md)
   — the controls you want to know cold before an incident.
7. This file (`05-handover-runbook.md`).

---

## See also

- [`../README.md`](../README.md)
- [`../INDEX.md`](../INDEX.md)
- [`01-policy-drift.md`](./01-policy-drift.md)
- [`03-troubleshooting-blocked-flows.md`](./03-troubleshooting-blocked-flows.md)
- [`04-evidence-and-audit.md`](./04-evidence-and-audit.md)
- [`06-compliance-companion.md`](./06-compliance-companion.md)
- Companion repo: [`chandrapati/CSW-Agent-Installation-Guide`](https://github.com/chandrapati/CSW-Agent-Installation-Guide)
- Companion repo: [`chandrapati/CSW-Compliance-Mapping`](https://github.com/chandrapati/CSW-Compliance-Mapping)
