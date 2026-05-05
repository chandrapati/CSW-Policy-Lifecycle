# 08 — Discovery Anti-Patterns

A field guide to the most common ways an ADM run goes off the
rails, and the fix.

---

## "One giant cluster" — ADM lumps everything together

**Symptom.** ADM run completes; there's one cluster containing
all the workloads in scope.

**Diagnosis.**

- Inventory is missing `tier` labels (or labels are inconsistent).
- All workloads talk to roughly the same external dependencies
  (DNS, AD, monitoring), so 5-tuple-only behaviour looks similar.

**Fix.**

1. Open the cluster in the workspace UI.
2. Sample 5–10 workloads — confirm what role each actually plays.
3. Add or correct `tier` labels (`tier=web`, `tier=app`,
   `tier=db`, etc.) — see
   [`../docs/01-prerequisites.md`](../docs/01-prerequisites.md).
4. Re-run ADM.

---

## "47 micro-clusters" — ADM splits everything

**Symptom.** ADM produces dozens of small clusters, often with
1–2 workloads each.

**Diagnosis.**

- Workloads in the same role have inconsistent extra labels
  (e.g. `node_type=m5.large` vs. `m5.xlarge`) that ADM is
  treating as distinguishing.
- Genuine drift — workloads in the same intended role are
  actually behaving differently because of misconfiguration.
- ADM's *Cluster Granularity* parameter is set too aggressive.

**Fix.**

1. Open two micro-clusters that should be one — diff their
   member workloads' labels.
2. If labels are accidentally distinguishing them (a label that
   is metadata, not behaviour), normalise the label values so
   they match where they should.
3. If there's a genuine behavioural difference (e.g. one
   workload is talking to an endpoint the others aren't),
   investigate the real difference before deciding whether to
   keep them separate.
4. If neither — adjust ADM's *Cluster Granularity* parameter
   toward fewer, larger clusters and re-run.

---

## "Unknown" providers in discovered policy

**Symptom.** Discovered rules reference providers shown as
"Unknown" or as bare IPs / CIDRs.

**Diagnosis.** The destination workload isn't in CSW's
inventory, isn't labelled, or is in inventory but not in any
scope.

**Fix.** See [`06-external-dependencies.md`](./06-external-dependencies.md)
for the categories. In summary:

- If the destination *is* a CSW-managed workload, fix its
  scope / labels.
- If the destination is third-party / SaaS / partner, create an
  explicit inventory filter naming the IP / CIDR with a
  description.
- If it's an internet flow you'll allow, accept it explicitly
  with a description.
- If it's an internet flow you'll block, don't add an Allow —
  delete from the proposed set.

**Don't publish a workspace with Unknown providers.**

---

## "Why does ADM propose this rule?" — the policy doesn't match my expectations

**Symptom.** ADM proposes a rule that surprises you.

**Diagnosis.** Either the flow really happened (and you didn't
know about it), or ADM misattributed.

**Fix.** Open the policy in the discovered set, click through to
[Conversations](../analysis/05-conversations.md). The
Conversations view shows the raw flows that drove the proposal:
when, between which workloads, on which port, with what
volume. From there:

- The flow is real and unexpected → investigate the app; the
  policy is correct, your understanding was wrong.
- The flow is real but undesired (a misconfigured client, a
  forgotten test fixture) → fix at source, exclude via flow
  filter, re-run.
- The flow is real but rare (one-off backup, scanner sweep) →
  decide whether to encode as policy or accept as exception.

---

## ADM proposes policy for an old version of the app

**Symptom.** Discovered rules reference workloads / flows that
belong to a deprecated version of the app.

**Diagnosis.** Old version is still in scope and still talking
in the flow window.

**Fix.**

- Decommission the old version; re-run ADM after at least the
  collection-window time has passed since decommission.
- *Or* exclude old-version workloads via a flow filter
  ([`05-flow-filters.md`](./05-flow-filters.md)) keyed on a
  `version=` label.
- Don't author policy that references the old version — it'll
  be dead code that future operators have to reason about.

---

## ADM "forgets" the month-end batch

**Symptom.** A monthly batch flow is missing from the discovered
policy. Week 3 enforcement breaks the batch.

**Diagnosis.** The collection window didn't include the batch.

**Fix.**

- Don't enforce until you've re-run ADM with a window that
  *does* include the batch.
- Or, author the batch flow rules manually and add them to the
  workspace before publishing.
- Update the workspace description to flag the dependency, so a
  future re-run after a process change includes it deliberately.

---

## ADM is run in the *primary* workspace by accident

**Symptom.** Existing published policy gets disturbed.

**Diagnosis.** ADM was run in the primary instead of a secondary.

**Fix.**

- Roll back to the previous published version: see
  [`../enforcement/09-rollback-and-revert.md`](../enforcement/09-rollback-and-revert.md).
- Re-run in a secondary next time. Always.

---

## Repeated ADM runs produce different cluster boundaries

**Symptom.** Run-to-run instability — cluster membership shifts
between consecutive runs.

**Diagnosis.**

- The flow window is rolling and a flow that's near-borderline
  drifts in / out.
- Inventory labels are changing during the runs (someone is
  re-tagging in parallel).

**Fix.**

- **Pin the time window** to a fixed start / end date when
  iterating — see [`02-flow-collection-window.md`](./02-flow-collection-window.md).
- Pause inventory relabelling during ADM iteration; finalise
  labels first, then iterate ADM.

---

## "Cross-scope rule" auto-accepted by mistake

**Symptom.** Workspace contains rules to scopes you didn't
intend to touch.

**Diagnosis.** *Auto-Accept Outgoing Policies* was on for an
ADM run on a sensitive workspace.

**Fix.**

- Remove the offending rules manually before publishing.
- Turn off auto-accept for this workspace's future runs.
- For shared-services workspaces, treat this with extra care —
  inadvertently auto-accepting a cross-scope rule into a
  shared service can broaden the surface unexpectedly.

---

## See also

- [`03-run-adm.md`](./03-run-adm.md)
- [`04-clusters-and-inventory-filters.md`](./04-clusters-and-inventory-filters.md)
- [`05-flow-filters.md`](./05-flow-filters.md)
- [`06-external-dependencies.md`](./06-external-dependencies.md)
- [`../analysis/05-conversations.md`](../analysis/05-conversations.md)
- [`../operations/03-troubleshooting-blocked-flows.md`](../operations/03-troubleshooting-blocked-flows.md)
- Cisco: [Discover Policies Automatically](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#discover-policies-automatically)
