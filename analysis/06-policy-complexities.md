# 06 — Policy Complexities: Priorities, Cross-Scope, Effective Consumer / Provider

This page covers the three things that most often produce
*"why is this flow being decided like that?"* questions:

1. **Priorities** when multiple rules could apply
2. **Cross-scope** rules where consumer and provider live in
   different scopes
3. **Effective Consumer / Effective Provider** — what the rule
   *actually* resolves to

> **Cisco source.** [Manage Policy Lifecycle — Address Policy Complexities](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#address-policy-complexities).

---

## 1. Priorities — what wins when rules overlap

CSW applies rules in a defined order, both by *rank* and by
*scope tree depth*.

### Rank order (recap from [`../docs/04-policy-attributes.md`](../docs/04-policy-attributes.md))

```
   Absolute  ──►  Default  ──►  Catch-All
   (highest)                       (lowest)
```

A flow matched by an Absolute rule is decided by it, even if
Default rules disagree. A flow matched by a Default rule is
decided by it, unless an Absolute also matched. The Catch-All
runs last.

### Scope tree depth

Within rank, **policies inherited from a parent scope apply
alongside those at the workspace's own scope**. The
[Address Policy Complexities → Policy Priorities](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#policy-priorities)
section is the authoritative description of how multiple
matching rules at the same rank are resolved (typically
allow-by-default at parent for shared services, with workspace-level
deny taking precedence — confirm against the chapter for the
exact resolution model in your release).

### Practical pattern

```
Production       (Absolute Deny: non-prod → prod)
    │
    ├── Shared-Services (Default Allow: prod → dns, prod → ad,
    │                      prod → monitoring, prod → ntp)
    │
    └── BU-Retail
          │
          └── payments-api (workspace)
                ├── Default Allow:  web → app tcp/8443
                ├── Default Allow:  app → db tcp/5432
                └── Catch-All Deny

Resolution for a flow web → app tcp/8443:
  1. No Absolute matches (Production's Absolute is non-prod→prod;
     this flow is prod→prod, doesn't match)
  2. Default match: payments-api Default Allow → ALLOW
```

Resolution for a flow `payments-api/web-01 → 10.7.7.7:445` with
no rule explicitly allowing it:

```
  1. No Absolute match
  2. No Default match
  3. payments-api Catch-All Deny → DENY
```

> **Tip.** When a "wrong" decision happens, sort the rules that
> matched by rank, and check the inheritance chain. The answer
> is almost always one of: a parent Absolute you forgot about,
> a workspace Catch-All that's broader than you intended, or a
> rank that should be Absolute but isn't.

---

## 2. Cross-scope policy

A cross-scope rule is one where consumer and provider live in
different scopes — for example,
*"`crm-web / web` → `payments-api / api` on tcp/8443"*.

### Where the rule "lives"

Cross-scope rules need to be **expressed on both sides**:

| Side | What it needs |
|---|---|
| Consumer side (`crm-web`) | A rule allowing the consumer to talk **outbound** to the counterpart provider |
| Provider side (`payments-api`) | A rule allowing **inbound** from the counterpart consumer |

If only one side has the rule, the agent on the workload that
*lacks* the rule will block (when enforcing) — the policy at
each end controls that workload's host firewall, and both must
permit the flow.

### How CSW helps

When ADM proposes a cross-scope rule, it can be marked as
*"depends on the counterpart workspace also having…"* and
shown in the External Dependencies view. The cluster does
*not* automatically push the same rule into the counterpart
workspace — that's a deliberate choice by the counterpart owner.

### A common workflow

1. ADM in `payments-api` discovers `crm-web → payments-api/api tcp/8443`.
2. The proposed rule is reviewed and accepted in `payments-api`.
3. In parallel, the same rule is added to `crm-web`'s workspace
   as `crm-web/web → payments-api/api tcp/8443` (consumer-side).
4. Both workspaces are published.
5. Live Analysis on **both** workspaces shows the flow as
   permitted before either is enforced.

Reference: [When Consumer and Provider Are in Different Scopes — Policy Options](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#when-consumer-and-provider-are-in-different-scopes).

---

## 3. Effective Consumer / Effective Provider

When a rule references inventory filters, the **effective**
consumer / provider is the actual set of workloads the filter
resolves to *at evaluation time*.

This matters because:

| Scenario | Surprise |
|---|---|
| Filter `tier=app` resolves to 12 workloads at policy authoring; later 18 at evaluation (autoscale) | Effective consumer / provider just changed — usually transparent, occasionally surprising |
| Filter intersects with an unrelated label that drifted | Membership can broaden silently |
| Filter referenced by multiple workspaces | Changing the filter affects every consuming workspace |

CSW's UI exposes the effective resolution at any time — open
the rule, click *"View effective consumer / provider"* (or the
equivalent in your release).

### Effective vs. authored — examples

| Authored | Effective at evaluation | Comment |
|---|---|---|
| Consumer = `application=payments-api AND tier=web AND environment=prod` | `web-01..web-12` | Ordinary; filter is doing what it should |
| Consumer = `tier=web` (no `application`) | All web workloads in scope across many apps | Probably too broad — narrow the filter |
| Provider = `prod-shared-dns` | `dns-01..dns-04` | Fine, as long as `prod-shared-dns` filter is correctly defined |

### When to inspect effective resolution

- After any inventory label change touching the filter's
  expression.
- After any scope membership change.
- During incident response — *"this rule should be allowing the
  flow but isn't"* — confirm the effective set actually contains
  the workload you think it does.
- Before publishing — sample a few rules and confirm the
  effective sets look correct.

Reference: [Effective Consumer or Effective Provider](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#effective-consumer-or-effective-provider).

---

## Putting it together — diagnosing a "wrong" decision

When you see a flow being decided unexpectedly, walk this list:

1. **Which rules matched?** Open the flow's evaluation in
   Conversations / Quick Analysis / Live Analysis.
2. **What ranks did they have?** Absolute beats Default beats
   Catch-All.
3. **What scopes did they come from?** Inherited from parent or
   defined locally?
4. **What did the inventory filters resolve to?** The effective
   consumer / provider.
5. **Is one of the rules from a counterpart cross-scope
   workspace?** Misalignment is the #1 cause of "this should
   work, doesn't."

90 % of "wrong" decisions resolve at step 1 or 2. The remaining
10 % resolve at step 4 or 5.

---

## See also

- [`../docs/04-policy-attributes.md`](../docs/04-policy-attributes.md)
- [`../discovery/06-external-dependencies.md`](../discovery/06-external-dependencies.md)
- [`../enforcement/08-policy-versions.md`](../enforcement/08-policy-versions.md)
- [`../operations/03-troubleshooting-blocked-flows.md`](../operations/03-troubleshooting-blocked-flows.md)
- Cisco: [Address Policy Complexities](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#address-policy-complexities)
