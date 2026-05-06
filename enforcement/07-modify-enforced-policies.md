# 07 — Modifying Enforced Policies

Once a workspace is enforcing, future changes follow the same
lifecycle as the initial rollout — but with the additional
constraint that **incorrect changes can immediately impact
production**. This page covers safe ways to modify enforced
policy.

> **Cisco source.** [Manage Policy Lifecycle — Modify Enforced Policies / Enforce New and Revised Policies](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#modify-enforced-policies).

---

## When you'd modify enforced policy

| Trigger | Typical change |
|---|---|
| App release with a new dependency | Add an Allow rule for the new flow |
| App release retiring an old dependency | Remove (or narrow) the obsolete rule |
| New shared service onboarded | Add Allow at the appropriate parent scope |
| Compliance requirement updated | Add an Absolute Deny |
| Refactor of the scope tree | Update inventory filters, possibly broader policy reshape |
| Discovery of unintended permissive rule | Tighten consumer / provider |

For app-team workflows that require frequent policy change,
GitOps-driven flows (see
[`../api/05-gitops-pattern.md`](../api/05-gitops-pattern.md)) are
the right pattern.

---

## The safe modify-while-enforcing flow

```
   Enforced workspace (currently enforcing v_n)
            │
            ▼
   Edit policy in workspace (creates v_(n+1))
            │
            ▼
   Quick Analysis (single hypothetical flow tests)
   Policy Experiments (replay past traffic against v_(n+1))
            │
            ▼
   Live Analysis on v_(n+1) (continuous, pre-enforce)
            │
            ▼
   Re-run Enable Policy Enforcement wizard,
     - select v_(n+1) on the Select Version page
     - confirm Policy Diff vs. v_n
            │
            ▼
   (Optional) "Simulate" via a non-enforcing agent type
   if your release supports one  ── 24–72 h depending on risk
            │
            ▼
   Enforce v_(n+1)
            │
            ▼
   Verify (file 06)
```

> **Naming reminders.** Per
> [About Policy Versions (v\* and p\*)](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#about-policy-versions-v-and-p),
> when you enforce v_(n+1) the **enforced version number is
> n+1** — it mirrors the v\* number you picked, not a separate
> p\* sequence. See
> [`08-policy-versions.md`](./08-policy-versions.md).
>
> **Quick Analysis** tests a single hypothetical flow. **Policy
> Experiments** replays past traffic. They're separate features
> — see [`../analysis/03-quick-analysis.md`](../analysis/03-quick-analysis.md).

Key points:

- **Always re-run Quick Analysis (and Policy Experiments)** after
  the edit — fast, cheap, catches the bulk of issues.
- **Don't skip the pre-enforce analysis window** for production
  changes, even small ones. The workspace is already enforcing;
  the cost of a bad change is real.
- **Compare via Policy Diff** — see [`08-policy-versions.md`](./08-policy-versions.md).
  A diff with surprises is the strongest signal that the
  proposed change isn't what you intended.

---

## "Safe" vs. "high-risk" changes

| Change type | Risk | Recommended approach |
|---|---|---|
| Adding a new Allow rule | Low | Quick Analysis + Live Analysis briefly; can move quickly |
| Narrowing an Allow rule | Medium | Simulate phase; watch for would-be-rejected flows that the previous broader rule covered |
| Removing an Allow rule | High | Simulate phase; the flow is now subject to the Catch-All — confirm that's intended |
| Adding a Deny rule | High | Simulate phase; possibly multiple days |
| Reshape of inventory filter referenced by many rules | High | Re-run Quick + Live Analysis; consider phased rollout per consumer-set |
| Change to Catch-All semantics (Allow → Deny or vice versa) | Critical | Treat as a fresh enforcement event; Monitor → Simulate → Enforce in full |

A change that "feels small" but is a Catch-All flip is
*never* small. Treat it like a new enforcement.

---

## Batching multi-change updates

A common ask: *"can we hold a batch of edits in one workspace
and push them to agents together?"* Worth being precise about
what CSW supports:

- **v\* edits don't push to agents.** Editing the workspace
  creates new v\* iterations. The host firewall doesn't change
  until you re-run the **Enable Policy Enforcement** wizard with
  the new version. So in practice, batching is the natural
  default — author all the edits in the workspace, analyze, then
  do **one** wizard run when ready.
- **There is no per-workspace "Pause Policy Updates" feature.**
  CSW's documented [Pause Policy Updates](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#pause-policy-updates)
  is **global** (all scopes / all workloads) and **site-admin
  only** — it's not for routine batching. Use it only when you
  need to halt updates *cluster-wide*. See
  [`10-pause-and-emergency-disable.md`](./10-pause-and-emergency-disable.md).

For routine multi-change updates: edit, analyze, enforce — once.

---

## Deleting policies

The [About Deleting Policies](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#about-deleting-policies)
section is the canonical reference. Operationally:

| Type of delete | Safety |
|---|---|
| Deleting a discovered (v\*) draft rule before publish | Safe — never went to host |
| Deleting a published (p\*) rule before that p\* is enforced | Safe — never went to host |
| Deleting an enforced rule | **Same risk profile as removing an Allow rule (medium–high)** — flow loses its specific allow, falls through to Catch-All |

Always treat deletion as "removing an Allow rule" risk-wise,
even when the intent is "this rule was always wrong." The
deletion still takes effect at the host firewall.

---

## Versioning hygiene during modifications

Each modification produces a new v\* and (after publish) a new
p\*. Best practice:

- **One change-ticket → one v\* → one p\*.** Don't bundle
  unrelated changes into one published version. The diff is
  easier to review, and the rollback is cleaner.
- **Annotate the workspace** with the change-ticket reference
  on each publish.
- **Keep the v\* / p\* history.** CSW retains versions for some
  time (release-dependent); you may want to export periodically
  for long-term audit. See [`08-policy-versions.md`](./08-policy-versions.md).

---

## See also

- [`04-rollout-pattern.md`](./04-rollout-pattern.md)
- [`08-policy-versions.md`](./08-policy-versions.md)
- [`09-rollback-and-revert.md`](./09-rollback-and-revert.md)
- [`10-pause-and-emergency-disable.md`](./10-pause-and-emergency-disable.md)
- [`../api/05-gitops-pattern.md`](../api/05-gitops-pattern.md)
- [`../operations/02-version-history.md`](../operations/02-version-history.md)
- Cisco: [Modify Enforced Policies](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#modify-enforced-policies)
- Cisco: [About Deleting Policies](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#about-deleting-policies)
