# 02 — Flow Collection Window

How long should agents have been collecting flow telemetry
before you run ADM?

The honest answer: *long enough for ADM to have observed every
flow that legitimately exists in steady state, plus all
periodic / scheduled flows.* Concretely:

| Application profile | Minimum window | Notes |
|---|---|---|
| Steady-state stateless web app | 7 days | Captures weekly variation |
| Typical 3-tier business app | 14 days | Default for most apps |
| App with weekly batch / reconciliation | 21 days | Must include at least one weekly batch |
| App with monthly close, quarterly batch | **30+ days** | Must include the largest batch window |
| HA / DR pair (active/passive) | Long enough to include at least one planned failover | Or run a DR test specifically to capture flows |

If you don't have enough flow window yet, **leave the workload
in Visibility, do labelling work in parallel, and come back in
2–4 weeks.** Skipping the batch window is the #1 cause of
*"the policy looked fine in test, then blocked the month-end
batch in production."*

---

## What "good" flow data looks like

Open *Investigate → Flows* with the time range you intend ADM
to use. You're looking for:

| What to check | Healthy | Warning |
|---|---|---|
| **Flow count per day** | Roughly constant or follows expected business cycle | Sharp drop on certain days = agent / ingestion gap |
| **Per-workload flow presence** | Every workload reports flows daily | A workload missing for >24 h was probably offline; investigate before including |
| **Periodic events visible** | Weekly batch / monthly close clearly visible as flow spikes | Flat = either no batch happened, or batch ran from a host you didn't include |
| **Process / package metadata** | Populated on Linux + Windows | Mostly missing = agent enriched view isn't working; ADM still works on 5-tuple, but loses precision |

If flow data is sparse, fix the root cause (agent disconnection,
ingest-side outage, label drift) before running ADM. Garbage in,
garbage out.

---

## Edge cases

### The DR / failover application

The app's flows during steady state include only the active node
talking to its peers. The flows during failover (or DR test) are
*different* — they include the standby suddenly becoming
primary, replication in the reverse direction, etc.

ADM run only against steady-state flow data will not produce
policy that survives a failover. Two options:

1. **Wait for a planned event.** If you have a regular DR test
   cadence (e.g. quarterly), schedule ADM after the next test.
2. **Run a controlled DR test** specifically to populate flow
   data, then run ADM against the window that includes it.

In both cases, describe the included event in the workspace's
description so future operators know why the policy includes
the flows it does.

### The "rare external partner" flow

A partner connection that fires once a week (or once a month)
might or might not appear in the ADM window. If you know about
it, three options:

1. **Extend the window** to ensure it's captured.
2. **Author the rule manually** and exclude that specific flow
   from ADM via [`05-flow-filters.md`](./05-flow-filters.md).
3. **Accept that it'll be a gap** and surface it in
   [Conversations](../analysis/05-conversations.md) for explicit
   review.

### Greenfield apps with no flow yet

Don't run ADM. There's nothing to discover. Author manually from
the API contract — see
[`../docs/05-decision-matrix.md`](../docs/05-decision-matrix.md).

### Apps mid-modernisation

Old and new versions running side-by-side produce a flow profile
that's the union of both. ADM will discover flows for both. This
is usually fine — the discovered policy will be the one that
covers the migration period. Plan to re-run ADM after the
migration completes and the old version is retired.

---

## Setting the time window in the UI

When you click *Run ADM* in the workspace, you're prompted for:

| Field | Recommendation |
|---|---|
| **Start date / End date** (custom range) | Prefer this. Be explicit. Document the choice in the workspace description. |
| **Last N days** | Use only for quick first passes; less reproducible |
| **Time zone** | UTC or the operating time zone of the app, consistently — don't mix |

Reproducibility matters: if a colleague needs to re-run ADM and
get a comparable result, they need to know exactly which window
was used the first time.

---

## Beyond the window — flow filters

The window is one dimension of ADM input control. The other is
**flow filters** — include / exclude expressions that further
refine what ADM is allowed to consider.

A common pattern is *use a 14-day window, but exclude flows from
the load test on day 9*. Flow filters cover this; see
[`05-flow-filters.md`](./05-flow-filters.md).

---

## See also

- [`01-prepare-scope.md`](./01-prepare-scope.md)
- [`03-run-adm.md`](./03-run-adm.md)
- [`05-flow-filters.md`](./05-flow-filters.md)
- Cisco: [Discover Policies Automatically](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#discover-policies-automatically)
