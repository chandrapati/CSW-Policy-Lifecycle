# 01 — Pre-Enforcement Checklist

The last go/no-go before enforcement is enabled on a workspace.
Treat this as a change-management gate; everything below should
have a documented "yes" before any enforcement-related setting
is changed.

---

## Workspace state

- [ ] Policy is **published** — workspace has a current p\*, not
      just a v\*. (See [`08-policy-versions.md`](./08-policy-versions.md).)
- [ ] **Live Policy Analysis** has been running for **≥ 5
      business days** on this workspace's scope.
- [ ] Live Analysis would-be-rejected count is **0** (or every
      non-zero is documented as expected).
- [ ] Live Analysis has covered any **periodic / batch event**
      the app has (weekly batch, monthly close, scheduled DR
      test).
- [ ] **Catch-All is explicit** — workspace has an authored
      Catch-All Allow or Deny; you know which and why.
- [ ] **Cross-scope rules** are reflected on **both sides** —
      the counterpart workspace has the matching rule. (See
      [`../analysis/06-policy-complexities.md`](../analysis/06-policy-complexities.md).)

---

## Agent state

- [ ] All agents in the workspace's scope are **healthy** —
      status `OK`, last check-in within the threshold (see
      [`02-agent-readiness.md`](./02-agent-readiness.md)).
- [ ] Agent versions are **supported** for enforcement on this
      OS — check the [Cisco Secure Workload Compatibility Matrix](https://www.cisco.com/c/m/en_us/products/security/secure-workload-compatibility-matrix.html).
- [ ] No workload in the scope is in **degraded** mode (agent
      reports flow only, can't enforce due to kernel or WFP
      limitation).
- [ ] **Pre-existing host firewall rules** on the target
      workloads are documented and reconciled with what the CSW
      agent will install (see [`05-platform-specific.md`](./05-platform-specific.md)).

---

## Change-management state

- [ ] Change ticket is **filed** with the right risk category
      (standard / normal / emergency — see
      [`../docs/01-prerequisites.md`](../docs/01-prerequisites.md)).
- [ ] **Approver(s)** have signed off.
- [ ] **Communications plan** is in place — app team, BU,
      relevant on-call rotations know enforcement is going live.
- [ ] **Maintenance window** is scheduled (if applicable for the
      change category).
- [ ] **Rollback path** is documented — see
      [`09-rollback-and-revert.md`](./09-rollback-and-revert.md)
      and [`10-pause-and-emergency-disable.md`](./10-pause-and-emergency-disable.md).
- [ ] **Rollback rehearsal** has been performed in a non-prod
      environment for this app (highly recommended for the
      first two production waves; see
      [`README.md`](./README.md)).
- [ ] **Incident response path** is clear — who calls what, how
      to open TAC if the cluster's involved.

---

## Observability state

- [ ] **Alerts** are wired for: agents going unhealthy, sudden
      rise in rejected flow count, sudden drop in expected flow
      count.
- [ ] **Dashboard** for the first 30 days exists (see
      [`11-monitoring-after-enforcement.md`](./11-monitoring-after-enforcement.md)).
- [ ] **App-level** SLI / SLO / synthetic checks are in place
      and currently green — so any post-enforcement issue
      surfaces from the app side as well as from CSW.

---

## Stakeholder state

- [ ] App owner has reviewed the policy and signed off.
- [ ] Security owner has reviewed the policy.
- [ ] Compliance owner has reviewed (if the app is under a
      compliance-mapping framework — see
      [`../operations/06-compliance-companion.md`](../operations/06-compliance-companion.md)).
- [ ] On-call team for the next 24 h is briefed.

---

## Documentation state

- [ ] Workspace description references the change ticket.
- [ ] Workspace description records the rollback procedure for
      this enforcement event.
- [ ] Evidence bucket (or equivalent) is set up to capture the
      enforcement-go-live artefacts (see
      [`../operations/04-evidence-and-audit.md`](../operations/04-evidence-and-audit.md)).
- [ ] Runbook for "blocked legitimate flow during the first
      week" is published and the on-call team knows where it is
      (see [`../operations/03-troubleshooting-blocked-flows.md`](../operations/03-troubleshooting-blocked-flows.md)).

---

## If any line above is unchecked

Don't enable enforcement yet. The fastest way to lose trust in
CSW (and to be told to "back it out") is to enforce on an app
that wasn't ready. Spending another week to satisfy this list
is cheap insurance.

---

## See also

- [`02-agent-readiness.md`](./02-agent-readiness.md)
- [`03-enable-enforcement.md`](./03-enable-enforcement.md)
- [`04-rollout-pattern.md`](./04-rollout-pattern.md)
- [`../docs/01-prerequisites.md`](../docs/01-prerequisites.md)
- [`../operations/04-evidence-and-audit.md`](../operations/04-evidence-and-audit.md)
