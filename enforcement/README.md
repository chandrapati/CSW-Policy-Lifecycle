# Phase 3 — Policy Enforcement

You arrive here with a **published** (p\*) policy in a primary
workspace, with Live Policy Analysis clean for ≥ 5 business
days including any periodic batch. The job in this phase is to
take that policy from *"would do the right thing if enforced"*
to *"is enforcing"* — without breaking anything.

> **Cisco source.** [Manage Policy Lifecycle — Enforce Policies](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#enforce-policies).

---

## The single most important thing on this page

**Don't go straight to Enforce.** The phased rollout —
Monitor → Simulate → Enforce — exists for a reason: it is the
single most effective control against accidentally breaking
production with a CSW change. Skipping it is the #1 cause of
preventable outages during CSW rollouts.

The phased rollout is documented in
[`04-rollout-pattern.md`](./04-rollout-pattern.md). Read that
file before configuring anything.

---

## What's in this folder

| File | Purpose |
|---|---|
| [`01-pre-enforcement-checklist.md`](./01-pre-enforcement-checklist.md) | Final go/no-go before flipping anything |
| [`02-agent-readiness.md`](./02-agent-readiness.md) | Check Agent Health and Readiness to Enforce |
| [`03-enable-enforcement.md`](./03-enable-enforcement.md) | Enabling enforcement on a workspace + the Enforcement Wizard |
| [`04-rollout-pattern.md`](./04-rollout-pattern.md) | The Monitor → Simulate → Enforce phased rollout (read first) |
| [`05-platform-specific.md`](./05-platform-specific.md) | Windows WFP · Linux iptables / nftables · containers |
| [`06-verify-enforcement.md`](./06-verify-enforcement.md) | Verifying enforcement actually does what the workspace says |
| [`07-modify-enforced-policies.md`](./07-modify-enforced-policies.md) | Editing policy that's already enforced |
| [`08-policy-versions.md`](./08-policy-versions.md) | v\* and p\* version semantics, Policy Diff, version retention |
| [`09-rollback-and-revert.md`](./09-rollback-and-revert.md) | Reverting an enforced workspace to a previous p\* |
| [`10-pause-and-emergency-disable.md`](./10-pause-and-emergency-disable.md) | Pause Policy Updates · Disable Policy Enforcement (incident path) |
| [`11-monitoring-after-enforcement.md`](./11-monitoring-after-enforcement.md) | What to watch in the first 30 days after enforcement |

---

## The enforcement workflow

```
   Pre-flight (file 01)
          │
          ▼
   Confirm agent readiness (file 02)
          │
          ▼
   Enable enforcement (file 03 + 04)
      │
      ├─► Phase 1: Monitor
      │       (no host firewall change; agents continue
      │        to report; differences from the policy
      │        appear as would-be-rejected events)
      │
      ├─► Phase 2: Simulate
      │       (host firewall is updated to match policy
      │        BUT actions are not enforced — the same
      │        signal as Live Policy Analysis, now driven
      │        by the agent rather than by the cluster)
      │
      └─► Phase 3: Enforce
              (host firewall enforces; flows that don't
               match an Allow are dropped per Catch-All)
          │
          ▼
   Verify (file 06)
          │
          ▼
   Steady state — modify, version, rollback as needed
   (files 07–11)
```

Each phase has explicit exit criteria; don't progress until
they're met. Detail in
[`04-rollout-pattern.md`](./04-rollout-pattern.md).

---

## A note on order of waves

Don't enforce all your apps at once, even with the phased
rollout per app. Recommended cadence:

| Wave | Population | Why |
|---|---|---|
| Wave 0 | Lab / pre-prod | Confirm the entire pipeline works end-to-end |
| Wave 1 | One **non-critical** prod app | Smallest blast radius; verify the org's change-management muscle works for CSW changes |
| Wave 2 | Two or three more prod apps with similar shape | Confirm the pattern holds |
| Wave 3+ | Roll-out plan in earnest | Adjusted by what was learned in waves 1–2 |

For the first two production waves, **plan for explicit
rollback rehearsal** — actually exercise the disable path
(file 10) on a Friday afternoon and verify the steady-state
returns. If you don't rehearse it, you don't have it.

---

## See also

- [`../analysis/`](../analysis/README.md) — Live Policy Analysis
  must be clean before this folder
- [`../operations/`](../operations/README.md) — what happens
  after the dust settles
- Cisco: [Enforce Policies](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#enforce-policies)
- Cisco: [Modify Enforced Policies](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#modify-enforced-policies)
