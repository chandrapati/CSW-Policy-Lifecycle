# 05 — Flow Filters: Refining ADM Input

Flow filters are **include / exclude expressions** applied to
the flow data ADM consumes. They let you say *"discover policy
based on these flows, ignore those flows."*

> **Cisco source.** [Manage Policy Lifecycle — Discover Policies Automatically](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#discover-policies-automatically)
> (filters are configured in the ADM run dialog).

---

## Why use flow filters

ADM is observational: it discovers policy from what it sees. If
what it sees includes flow you don't want to encode as policy
(load tests, accidental misconfiguration, scanner noise), the
discovered policy will encode that too. Flow filters are the
clean way to keep the noise out of the discovered set.

Common scenarios:

| Scenario | Filter |
|---|---|
| One-off load test on day 9 of the window | Exclude flows where `consumer.ip == load-tester-ip` |
| Vulnerability scanner sweeping the scope | Exclude flows where `consumer.application == vuln-scanner` |
| Noisy probe / health-check from a tool you'll remove | Exclude flows where `consumer.process_name == legacy-probe` |
| Old version of the app being decommissioned | Exclude flows where `provider.label.version == v1` |
| Flow from a migration tool that won't be there post-cutover | Exclude flows from `consumer.application == data-migration-tool` |

Inverse pattern (include filters):

| Scenario | Filter |
|---|---|
| Only learn from prod traffic, ignore dev flows in this scope | Include only `consumer.label.environment == prod` and `provider.label.environment == prod` |
| Capture only traffic from a specific data centre | Include only `consumer.label.site == dc-pri` |

---

## How filters compose

ADM applies, in order:

```
   All flow records in the time window
            │
            ▼
   Apply scope filter   (= members of this workspace's scope)
            │
            ▼
   Apply include filter (if set)
            │
            ▼
   Apply exclude filter (if set)
            │
            ▼
   Surviving flows  ──►  ADM clustering and policy proposal
```

If both include and exclude are set, exclude wins on conflicts
(safer default). If only include is set, the survivor set is
"everything matching include." If only exclude is set, the
survivor set is "everything except the excluded."

---

## Writing filter expressions

CSW filter expressions reference flow attributes:

| Attribute family | Examples |
|---|---|
| 5-tuple | `consumer.ip`, `provider.ip`, `protocol`, `provider.port` |
| Inventory | `consumer.label.application`, `provider.label.tier`, `consumer.scope` |
| Process / package (where reported) | `consumer.process_name`, `provider.process_name`, `consumer.user` |
| Workload | `consumer.os`, `provider.os` |
| Flow metric | `byte_count`, `packet_count`, `flow_duration` |

Combinators:

- `AND`, `OR`, `NOT`
- `==`, `!=`, `IN (...)`, `MATCHES <regex>`
- Subnet matches: `consumer.ip IN 10.50.0.0/16`

Examples:

```text
# exclude vuln scanner sweeps
consumer.label.application == "vuln-scanner"

# include only prod-to-prod
consumer.label.environment == "prod" AND provider.label.environment == "prod"

# exclude one-off DR test traffic
consumer.label.role == "dr-test-client"

# exclude internet-bound flows from ADM, leave for manual decision
NOT (provider.scope == "internet")
```

---

## Documenting your filters

Filters used in an ADM run **change which flows the discovered
policy reflects**. Months later, the question "why doesn't this
policy cover the load-test flow?" should be answerable without
guesswork. Document filters in:

- The **workspace description** when ADM runs
- A short comment in the policy versioning notes when a v\*
  is created from a filtered run

---

## When to use filters vs. fix labels

Filters and labels overlap. If a flow can be excluded by a
filter expression, it can usually be made to behave correctly
by labelling differently. Rule of thumb:

| Use a filter when | Fix labels when |
|---|---|
| The flow source is **temporary** (one-off load test) | The flow source is **permanent** but should behave differently (a monitoring host) |
| You're about to retire the flow source | The source isn't going anywhere |
| Labels can't usefully distinguish (5-tuple-only signal) | Adding a label makes the distinction reusable across workspaces |

Filters are tactical; labels are strategic. Lean on labels
unless you have a specific tactical reason for a filter.

---

## See also

- [`02-flow-collection-window.md`](./02-flow-collection-window.md)
- [`03-run-adm.md`](./03-run-adm.md)
- [`08-discovery-anti-patterns.md`](./08-discovery-anti-patterns.md)
- Cisco: [Discover Policies Automatically](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#discover-policies-automatically)
