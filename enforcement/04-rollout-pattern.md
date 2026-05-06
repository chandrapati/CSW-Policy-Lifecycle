# 04 — The Monitor → Simulate → Enforce Rollout Pattern

The single most important pattern in this entire repo. Read it
before configuring any enforcement setting.

> **Naming caveat.** "Monitor → Simulate → Enforce" is a
> **methodology** for safe rollout, not a Cisco-named workflow
> or a built-in three-state machine. It composes Cisco features
> that *are* documented:
>
> - **Monitor** = agents in
>   [Deep Visibility](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html)
>   + cluster-side
>   [Live Policy Analysis](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#live-policy-analysis).
> - **Simulate** = analyze a published version *without* enabling
>   enforcement (or via a non-enforcing agent type if your release
>   exposes one — verify the agent-type names available in your
>   release).
> - **Enforce** = the documented
>   [Enable Policy Enforcement](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#enable-policy-enforcement)
>   flow with the
>   [Policy Enforcement Wizard](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#policy-enforcement-wizard).
>
> The pattern is what disciplined teams do; the building blocks
> are Cisco's. If your release exposes different agent-mode names
> (e.g. variants of "visibility-only" or a preview state), use
> those — the methodology still applies.

---

## Why this pattern exists

CSW enforcement is a change to host-firewall configuration on
every workload in scope. Like every other host-firewall change,
the failure mode is "legitimate flow gets blocked, app breaks."

The Monitor → Simulate → Enforce pattern exists to **catch
those failure cases before they cause damage** by progressively
exposing the policy to real traffic without yet acting on it.

Each phase produces high-fidelity evidence that the policy is
safe to take to the next phase. Skipping a phase loses that
evidence.

---

## Phase 1 — Monitor

**What it is.** The pre-enforcement state. The agent reports
flows, the cluster runs Live Policy Analysis, but no host
firewall change occurs yet. This is essentially the
[`../analysis/04-live-analysis.md`](../analysis/04-live-analysis.md)
state, viewed through the enforcement lens.

| | |
|---|---|
| Agent mode | `Deep Visibility` (no enforcement) |
| Host firewall | Unchanged |
| Cluster behaviour | Evaluates policy against current flows; reports would-be-rejected |
| Risk to traffic | None |

**Exit criteria.** Live Policy Analysis is clean — would-be-rejected
count is 0 (or every non-zero is explicitly accepted) for ≥ 5
business days, including any periodic event.

---

## Phase 2 — Simulate

**What it is.** The agent installs the host firewall rules
matching the published policy, but enforcement actions are
suppressed. Both sides — cluster and agent — now have a
consistent view, and the agent can report local "would-be-rejected"
events in addition to cluster-level signals.

| | |
|---|---|
| Agent mode | A non-enforcing mode that pre-stages rules where your release supports it (names vary — verify in your agent documentation); otherwise stay in Deep Visibility while running cluster-side analysis on the published version |
| Host firewall | **Updated** to reflect the policy *only if your release's agent supports a pre-stage mode*; otherwise unchanged |
| Cluster behaviour | Evaluates flows, reports would-be-rejected |
| Agent behaviour | Reports local would-be-rejected from the agent's perspective (when in a pre-stage mode) |
| Risk to traffic | None — agent does not block, only reports |

**Why this phase exists separately from Monitor.** Monitor
catches issues the cluster sees; Simulate catches issues the
agent sees. The two views are not identical — agents see
flows the cluster might not (due to ingest sampling, cluster-side
gaps, etc.). Simulate is the first time the **host firewall's
own evaluation** can be observed without consequences.

**Exit criteria.**

- ≥ 48–72 h of Simulate with rejected-flow count at zero (or
  fully explained), spanning at least one normal business
  daily cycle.
- Agent status remains OK across the population.
- Application synthetic checks remain green.
- Spot-check confirms host firewall reflects expected rules
  (`iptables -L -n` / `Get-NetFirewallRule`).

---

## Phase 3 — Enforce

**What it is.** The agent enforces the host firewall rules.
Flows that don't match an Allow are dropped per the Catch-All.

| | |
|---|---|
| Agent mode | `Enforcement` |
| Host firewall | Enforces rules |
| Cluster behaviour | Evaluates and reports actual rejected flows |
| Agent behaviour | Drops non-matching flows; reports rejection events |
| Risk to traffic | Real — non-matching flows are blocked |

**Exit criteria.** Same as the steady-state monitoring criteria
in [`11-monitoring-after-enforcement.md`](./11-monitoring-after-enforcement.md).

---

## A typical timeline

For a single non-trivial production app on first enforcement:

| T | Step |
|---|---|
| Day –30 | Agents in Visibility, flow data collecting |
| Day –14 | Pre-flight + ADM run (see [`../discovery/`](../discovery/README.md)) |
| Day –10 | Refinement passes |
| Day –5 | Publish (v\* → p\*); start Live Policy Analysis |
| Day 0 | Live Analysis clean for 5+ days, including any batch — pre-flight checklist green |
| Day 0 | **Begin Simulate phase** (agents go to Simulate mode) |
| Day 0 → Day +3 | Simulate; agent-side rejected-flow count must trend to zero |
| Day +3 | **Move to Enforce on the lowest-risk tier first** (e.g. `tier=web`) |
| Day +4 | If clean for 24 h, **Enforce on `tier=app`** |
| Day +5 | If clean for 24 h, **Enforce on `tier=db`** |
| Day +5 → Day +30 | Steady-state monitoring per [`11-monitoring-after-enforcement.md`](./11-monitoring-after-enforcement.md) |
| Day +30 | First-30-days review; either close out the change or address findings |

For very high-risk apps (revenue-critical, single point of
failure, regulatory), extend each phase. For lab / non-prod
apps, compress the timeline.

---

## Tier-by-tier sequencing within Enforce

Within Phase 3, don't enforce all tiers at once even on a
single app. The recommended order:

```
   Web tier first      ─── (highest connectivity, most permissive,
                            simplest to revert without DB damage)
        │
        ▼
   App tier next       ─── (middle tier; ride-along monitoring of
                            web→app already established)
        │
        ▼
   DB tier last        ─── (highest risk; smallest population;
                            most observable from app tier)
```

Reasoning: web-tier issues are easiest to detect (synthetic
checks), easiest to revert (kick a stateless instance), and
have the smallest blast radius. DB-tier issues can corrupt or
stall many things; they're worth the most caution.

---

## Per-phase rollback paths

| Phase | Rollback action | Effect |
|---|---|---|
| Monitor | None needed (no change made) | n/a |
| Simulate | Revert agent mode to Deep Visibility | Host firewall is rolled back to pre-Simulate state; takes seconds–minutes |
| Enforce | Revert agent mode to Simulate (or Visibility); or revert workspace to a previous p\* | Documented in [`09-rollback-and-revert.md`](./09-rollback-and-revert.md) and [`10-pause-and-emergency-disable.md`](./10-pause-and-emergency-disable.md) |

For incident-grade rollback (production breaking), the path
is in [`10-pause-and-emergency-disable.md`](./10-pause-and-emergency-disable.md)
— go directly to that on production-impacting issues.

---

## What "ready to move to next phase" actually means

The mistake is reading these criteria as a checkbox. They're
not — they're **questions you must be able to answer "yes" to,
honestly, with evidence.**

| Criterion | Evidence |
|---|---|
| Live Analysis is clean | Screenshot / export of the Live Analysis dashboard for the relevant window, attached to the change ticket |
| Periodic events covered | The window includes the actual event (look at the timestamps) |
| Agents healthy | Agent inventory query showing `Status=OK` count = expected |
| App synthetic checks green | App team confirms |
| Rollback rehearsed | Write-up of the rehearsal (in non-prod) attached to the change ticket |

If you can't produce the evidence, you don't have the criterion
met. Don't progress.

---

## See also

- [`01-pre-enforcement-checklist.md`](./01-pre-enforcement-checklist.md)
- [`03-enable-enforcement.md`](./03-enable-enforcement.md)
- [`06-verify-enforcement.md`](./06-verify-enforcement.md)
- [`09-rollback-and-revert.md`](./09-rollback-and-revert.md)
- [`10-pause-and-emergency-disable.md`](./10-pause-and-emergency-disable.md)
- [`11-monitoring-after-enforcement.md`](./11-monitoring-after-enforcement.md)
- [`../analysis/04-live-analysis.md`](../analysis/04-live-analysis.md)
