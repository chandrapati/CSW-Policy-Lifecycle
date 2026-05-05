# 05 — Conversations

The Conversations table is the **raw flow evidence** behind
every discovered or proposed policy. It's the answer to *"why
does this rule exist?"* and to *"what flow does this rule
actually represent?"*.

> **Cisco source.** [Manage Policy Lifecycle — Conversations](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#conversations).

---

## What a Conversation is

A *Conversation* is an aggregated communication between a
consumer and a provider on a specific service over a time
window. Where individual flow records are noisy and high-volume,
a Conversation is one row per `(consumer, provider, port,
proto)` tuple with rolled-up metrics.

A Conversation row carries:

| Field | Meaning |
|---|---|
| Consumer | Initiator workload (or workloads, when the row aggregates) |
| Provider | Receiver workload |
| Service | protocol/port |
| Total bytes / packets | Volume in the window |
| Flow count | Number of distinct flows |
| Time range | Period the aggregation covers |
| Tags / process metadata | Where the agent reported it |

The Conversations view is reachable from any policy in the
workspace via *"View Conversations"*, and stand-alone via
*Workspace → Analysis → Conversations*.

---

## When to use Conversations

| Use case | Why |
|---|---|
| *"Why did ADM propose this rule?"* | Conversations show the flows that drove the proposal |
| Triage a suspect would-be-rejected flow in Live Analysis | The conversation row tells you who, what, when, and how much |
| Confirm a flow is legitimate before allowing | If the volume / cadence makes sense for a real app interaction, it's probably legitimate |
| Confirm a flow is malicious before deny is permanent | If the volume is tiny and sporadic from an unexpected consumer, it's worth investigating before allowing |
| Reconcile two interpretations of the same rule | The raw evidence is unambiguous |

---

## Reading Conversations

### Volume tells you the rule's importance

| Volume profile | What it means |
|---|---|
| Steady high volume | Production-critical path; *don't* break this rule lightly |
| Steady low volume | Health checks, heartbeats, monitoring polls |
| Bursty high volume | Periodic batch / scheduled job |
| Single sporadic event | A scanner sweep, a one-off, or a real but rare partner exchange |

### Cadence tells you whether ADM saw enough

| Cadence | Implication |
|---|---|
| Continuous (every minute) | ADM saw plenty; high confidence |
| Periodic (daily, weekly, monthly) | Confirm the ADM window includes the period |
| One observation in the window | Don't author broad policy from a single observation; review |

### Process / package tells you who's actually talking

When the agent reports it, you'll see the source process
(`java`, `nginx`, `cmd.exe`, etc.) and sometimes the binary
hash. Useful for:

- Distinguishing a legitimate app process from a misconfigured
  tool that happens to be running on the same workload.
- Spotting unauthorised software (a binary nobody recognises
  initiating egress).
- Confirming SSH sessions are coming from the bastion's
  managed `ssh` process and not a worm.

---

## Common Conversation-driven decisions

### "ADM proposed an Allow for tcp/22; should I keep it?"

Look at the conversation. If consumer is the bastion's known
inventory filter, source process is `ssh`, volume is small and
human-paced — yes, keep, narrow the consumer to the bastion
filter. If consumer is "Unknown" and the source process is
something opaque, no.

### "Live Analysis is rejecting tcp/445 from a workload to a server"

Open the conversation. If the source process is something like
`smbclient` from a known monitoring agent, allow with
`prod-monitoring → fileserver tcp/445`. If the source process is
something you don't recognise and the volume is very low, you
have a potential discovery — don't allow until investigated.

### "The same rule has 100k flows per day on weekdays, 0 on weekends"

This is normal for most business apps. The rule is fine. The
data point is useful for *capacity planning* and for
*recognising abnormal behaviour later*.

---

## Conversations as drift signal

Once enforcement is on, a *new* conversation (one not previously
seen) often signals legitimate app drift:

- New version deployed; new dependency.
- New batch onboarded; existing rule too narrow.
- New partner integration.

Periodically diffing the Conversations view (or sampling new
conversations) is a low-cost drift detector. See
[`../operations/01-policy-drift.md`](../operations/01-policy-drift.md).

---

## Limitations

- **Process metadata depends on agent capability.** Containerised
  workloads, kernel-restricted environments, and some Windows
  configurations report less. Don't assume process identification
  is always available.
- **Aggregation hides detail.** A conversation row is a roll-up;
  if you need exact flow records (5-tuple, timing), use *Investigate
  → Flows* with the same filter.
- **Conversations don't include flow that wasn't observed.** A
  flow blocked by a network firewall before reaching the
  workload won't appear; CSW only sees flows that reach the
  agent's observation point.

---

## See also

- [`01-review-discovered-policies.md`](./01-review-discovered-policies.md)
- [`03-quick-analysis.md`](./03-quick-analysis.md)
- [`04-live-analysis.md`](./04-live-analysis.md)
- [`../discovery/05-flow-filters.md`](../discovery/05-flow-filters.md)
- Cisco: [Conversations](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#conversations)
