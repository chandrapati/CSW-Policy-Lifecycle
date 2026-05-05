# 01 — Reviewing Discovered Policies

A discovered policy set is a **proposal**, not a finished
product. Every rule needs a human read before publishing.

> **Cisco source.** [Manage Policy Lifecycle — Review Automatically Discovered Policies](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#review-automatically-discovered-policies).

---

## The review checklist (per rule)

For each discovered rule, answer:

| Question | What to look for |
|---|---|
| **Does the consumer make sense?** | Inventory filter resolves to the workloads you'd expect. Not "Unknown." |
| **Does the provider make sense?** | Same — and isn't a single-workload-list when it should be a tier filter. |
| **Are the ports right?** | Are these the *minimal* ports the flow actually needs, or did ADM include a noisy non-standard port that should be excluded? |
| **Is the action right?** | ADM proposes Allow rules (with denies expressed via Catch-All); confirm. |
| **Is the rank right?** | ADM places discovered rules at Default rank by default; verify nothing is mis-ranked. |
| **Is there a description?** | If not, *add one* before publishing. *"Discovered v3 — payments-api web → app, see CR-1234"* is enough. |
| **Is this rule covered by another rule?** | Watch for redundancy — see *Coverage and overlap* below. |

---

## Reviewing the policy in the UI

The workspace's Policies view shows discovered rules in a
tabular layout with sortable columns. Practical workflow:

1. **Sort by Confidence** — review the lowest-confidence rules
   first. Anything below the workspace threshold deserves a
   manual sanity check against
   [Conversations](./05-conversations.md).
2. **Filter by Provider** — group all rules where the same
   provider filter is referenced. Often you'll find rules that
   could be merged (same consumer, same provider, multiple
   ports → consolidate).
3. **Filter by Consumer** — same exercise, from the consumer
   side.
4. **Look for Unknowns** — open the filter for `consumer = Unknown`
   and `provider = Unknown`. Resolve every one before
   publishing — see [`../discovery/06-external-dependencies.md`](../discovery/06-external-dependencies.md).
5. **Look for the Catch-All** — verify a Catch-All exists at
   the workspace level and its action is what you intend.

---

## Coverage and overlap

ADM may produce rules that overlap. Two flavours:

### Redundant rules (same logical effect)

| Rule A | Rule B | Action |
|---|---|---|
| `tier=web` → `tier=app` on tcp/8443 | `payments-api/web-01,web-02` → `tier=app` on tcp/8443 | Rule B is a strict subset of Rule A; delete B |
| `tier=any` → `prod-shared-dns` on udp/53 | `tier=web` → `prod-shared-dns` on udp/53 | Rule B is subsumed by Rule A; delete B |

The CSW UI flags many of these explicitly via the *Policy
Compression* / overlap indicators; the
[Manage Policy Lifecycle chapter — Address Policy Complexities](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#address-policy-complexities)
covers the underlying concept.

### Conflicting rules (one allow, one deny)

| Rule A | Rule B | Resolution |
|---|---|---|
| Default Allow `tier=web` → `tier=app` tcp/8443 | Absolute Deny `environment=non-prod` → `environment=prod` any | Absolute beats Default; B wins for any cross-environment subset of A. |
| Default Allow `tier=web` → `tier=app` tcp/8443 | Default Deny `application=test-tooling` → `tier=app` any | Both Default; the implicit ordering and inventory-filter overlap rules apply — see [`06-policy-complexities.md`](./06-policy-complexities.md). |

For confusing conflicts, use the [Policy Visual Representation](./02-policy-visual.md)
to see the resolved effect.

---

## Refining vs. removing rules

Some rules look redundant but aren't:

- A *narrower* rule with a description that explains *why* the
  narrowness matters can be more valuable than the *broader*
  rule that subsumes it. ("Allow `bastion → mongo-cluster
  tcp/22`, narrower than `bastion → tier=db tcp/22`, because
  only the mongo cluster permits SSH from bastion in this
  environment — see SR-9876.")
- An ADM rule from a noisy day's flow filter may *correctly*
  cover a rare legitimate flow you'd otherwise miss. Don't
  delete because "it's only one flow" without checking
  Conversations.

When in doubt, keep the rule and add a description.

---

## Common cleanup patterns

| Pattern | Cleanup |
|---|---|
| Multiple rules from the same consumer to the same provider on adjacent ports | Merge into one rule with a port-range or comma-list |
| Multiple rules from one consumer to several DNS resolvers | Replace per-resolver rules with one rule referencing `prod-shared-dns` |
| Rules referencing individual workload IPs / names | Replace with a label-based inventory filter |
| Rules with `tier=` mismatching how the rest of your policy talks about tiers | Reconcile label vocabulary first |
| A rule that exists only because of ADM's auto-accept | Delete unless you've confirmed it independently |

---

## Documenting your review

Before publishing, the workspace description (or the policy
versioning notes for the imminent p\*) should record:

- **What was reviewed** — discovered v\*N, time window used.
- **Who reviewed it** — author + at least one reviewer name.
- **What was changed from raw ADM output** — added Catch-All,
  merged duplicate rules, removed Unknown providers, etc.
- **What deferred decisions remain** — *"the partner-bank
  egress flow is allowed pending change request CR-2345"*.

This is the audit trail. It is *the* answer when, six months
from now, someone asks "why does this rule exist?".

---

## See also

- [`02-policy-visual.md`](./02-policy-visual.md)
- [`03-quick-analysis.md`](./03-quick-analysis.md)
- [`05-conversations.md`](./05-conversations.md)
- [`06-policy-complexities.md`](./06-policy-complexities.md)
- Cisco: [Review Automatically Discovered Policies](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#review-automatically-discovered-policies)
