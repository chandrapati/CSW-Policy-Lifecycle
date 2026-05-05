# 11 — Monitoring After Enforcement

The first 30 days after enforcement is enabled on a workspace
are when most preventable issues either show up or quietly pass.
This page is the watch list.

---

## What to watch — the four-signal model

```
   1. Rejected-flow rate              ── catches policy gaps
   2. Misdrop rate                    ── catches app / infra issues
   3. Agent health across population  ── catches agent-layer regression
   4. App-side SLI / synthetic checks ── catches end-user impact
```

Track all four. They cover overlapping but distinct failure
modes; any one alone misses things.

---

## Signal 1 — Rejected-flow rate

**What it is.** Count of flows the host firewall rejected per
unit time, in this workspace's scope. Visible in the workspace's
Enforce / Enforcement Reporting view.

**Healthy.** At-or-near zero, matching what
[Live Analysis](../analysis/04-live-analysis.md) predicted in
the pre-flight window.

**Red flags.**

| Pattern | Likely cause | Action |
|---|---|---|
| Steady rejected count higher than predicted | Real flows being blocked that you didn't catch in Live Analysis | [`../operations/03-troubleshooting-blocked-flows.md`](../operations/03-troubleshooting-blocked-flows.md) |
| Sudden spike at a specific time of day | A scheduled job whose flow wasn't in the ADM window | Allow if legitimate; audit if not |
| Slowly climbing over days | Application drift — new dependency, new client, new partner | [`../operations/01-policy-drift.md`](../operations/01-policy-drift.md) |
| Spike correlated with a deployment | New dependency in the new app version | Open a change ticket, add the rule, re-publish |

---

## Signal 2 — Misdrop rate

**What it is.** Flows the policy *permitted* but the agent
observed a connection failure on (RST, no listener, etc.).

**Healthy.** Stable; should not change as a function of the CSW
rollout.

**Red flags.**

| Pattern | Likely cause | Action |
|---|---|---|
| Misdrop count climbed after enforcement | Almost always *not* CSW — but worth confirming. The host firewall's enforcement might be slightly less permissive than expected for some edge cases. | Spot-check on affected hosts; verify no host-level conflict per [`05-platform-specific.md`](./05-platform-specific.md) |
| New misdrop pattern from a specific consumer | App / infra issue at the consumer or provider | Surface to the app team; CSW telemetry is helpful but not the cause |

---

## Signal 3 — Agent health across population

**What it is.** Status / last-checkin / type for every agent in
the workspace's scope.

**Healthy.** Population stable; `Status=OK` for all; `Type=Enforcement`
for the enforced subset; recent check-ins for all.

**Red flags.**

| Pattern | Likely cause | Action |
|---|---|---|
| One agent goes unhealthy after enforcement | Local issue on that host — agent crash, kernel issue, conflicting tool | Triage; consider excluding from enforcement until fixed |
| Many agents go unhealthy after enforcement | Cluster-side push issue, or systemic incompatibility (kernel update?) | Open TAC; consider disable while investigating |
| Agent CPU / memory uplift visible | Some uplift is expected (more state to track); large uplift is unusual | Capture before / after; verify against [Compatibility Matrix](https://www.cisco.com/c/m/en_us/products/security/secure-workload-compatibility-matrix.html) |

---

## Signal 4 — App-side SLI / synthetic checks

The most important one — and the one that's *not* in CSW.

**What it is.** Whatever your apps already use to know they're
healthy: synthetic transactions, end-to-end probes, business
KPIs.

**Healthy.** Unchanged from pre-enforcement.

**Red flags.**

| Pattern | Action |
|---|---|
| Synthetic check fail correlated with the enforcement go-live | Strong indicator of policy issue. Go to [`10-pause-and-emergency-disable.md`](./10-pause-and-emergency-disable.md) if severe; otherwise [`../operations/03-troubleshooting-blocked-flows.md`](../operations/03-troubleshooting-blocked-flows.md) |
| Tail latency increase | Possibly a flow that's now retrying through a fallback path; investigate via [`../analysis/05-conversations.md`](../analysis/05-conversations.md) |
| User-reported issue with no signal in CSW | Don't dismiss. CSW signals are necessary, not sufficient. |

---

## Day-by-day cadence (first 30 days)

| Day | Watch |
|---|---|
| 0 (go-live) | All four signals at T+5, T+15, T+60 minutes |
| 1 | Daily summary of all four signals; on-call walk-through |
| 2–7 | Daily summary; review every rejected flow individually |
| 8–14 | Daily summary; weekly batch first run-through under enforcement |
| 15–30 | Twice-weekly summary; monthly batch first run-through (if applicable) |
| 30 | First-30-days review; close out the change ticket; promote to standard-change category for future similar enforcement events |

---

## Long-running monitoring (after the first 30 days)

| Cadence | Activity |
|---|---|
| Continuous | Live Analysis (or Enforcement Reporting) on the workspace; alert on anomalous rejected-flow count |
| Weekly | Review new conversations against current policy — spot drift early |
| Monthly | Spot-check sample of agents — host firewall state matches workspace policy |
| Quarterly | End-of-quarter version pin (per [`08-policy-versions.md`](./08-policy-versions.md)); compliance report run-through if applicable |
| Annually | Re-baseline ADM run as a sanity check; compare against current published policy |

---

## What to log / archive

| Artefact | Why |
|---|---|
| Workspace state at enforcement go-live | Audit anchor |
| Policy export at go-live (JSON) | Reproducible baseline |
| Live Analysis screenshots from the pre-flight window | Evidence of due diligence |
| First-30-days monitoring report | Closure of the change ticket |

See [`../operations/04-evidence-and-audit.md`](../operations/04-evidence-and-audit.md)
for the full evidence-bucket pattern.

---

## See also

- [`04-rollout-pattern.md`](./04-rollout-pattern.md)
- [`06-verify-enforcement.md`](./06-verify-enforcement.md)
- [`10-pause-and-emergency-disable.md`](./10-pause-and-emergency-disable.md)
- [`../operations/01-policy-drift.md`](../operations/01-policy-drift.md)
- [`../operations/03-troubleshooting-blocked-flows.md`](../operations/03-troubleshooting-blocked-flows.md)
- [`../operations/04-evidence-and-audit.md`](../operations/04-evidence-and-audit.md)
