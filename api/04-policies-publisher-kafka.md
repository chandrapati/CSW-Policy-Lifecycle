# 04 — Policies Publisher (Kafka)

CSW can publish policy events to a **Kafka stream** so that
downstream systems — host firewalls outside CSW, network
controllers, custom enforcers — can consume policy as it changes
without polling the OpenAPI.

> **Cisco source.** [Manage Policy Lifecycle — Policies Publisher](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#policies-publisher).

---

## What it is

Policies Publisher is a CSW-side **Kafka producer** that emits a
structured event whenever a workspace publishes (or a policy
changes in any way that affects the published surface). External
**clients** subscribe to the stream and act on the events.

```
   CSW cluster                                  Downstream client
       │                                              │
       │  publish v17 → p4                            │
       │                                              │
       │  ─── policy update event ───►   Kafka topic  │
       │                                              │
       │                                              ▼
       │                                       reconcile local
       │                                       firewall / table
```

Common downstream consumers:

- A **non-CSW host firewall** that mirrors a subset of policy
  (e.g. for legacy systems where the CSW agent isn't deployed).
- A **network controller** (SDN, firewall manager) consuming
  policy as input to its own configuration plane.
- A **custom enforcer** for a platform CSW doesn't natively
  support.
- An **observability pipeline** that wants policy events in the
  same time-series store as flow events.

---

## Prerequisites

> **Source.** The Cisco section
> [Policies Publisher — Prerequisites](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#prerequisites)
> is the canonical source.

| Prerequisite | Notes |
|---|---|
| Kafka broker(s) reachable from CSW | Cluster-side network reachability; TLS strongly recommended |
| Kafka topic for policy events | Configurable; usually one per tenant or one per consumer group |
| **Client certificates** for the consumer | CSW issues per-consumer certs for mTLS |
| Reasonable retention on the topic | Long enough that a brief consumer outage doesn't lose state |

---

## Getting Kafka client certificates

Per
[Getting Kafka Client Certificates](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#getting-kafka-client-certificates):

1. Open the cluster's Kafka client config (UI path is
   release-dependent; consult the chapter).
2. Generate a client cert / key pair tied to the consumer.
3. Store the cert + key in your secret manager.
4. Configure the downstream client to authenticate to Kafka
   using mTLS with the issued cert.

These certs **expire** — manage their lifecycle the same way
you'd manage any other certificate (renewal cadence, rotation
plan, monitoring approaching expiry).

---

## The protobuf schema

Policy events are serialized as **Protocol Buffers**. The
schema is published with the User Guide chapter — see
[Protobuf Definition File](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#protobuf-definition-file)
and [Data Model of Secure Workload Network Policy](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#data-model-of-secure-workload-network-policy).

Key concepts:

| Field group | Carries |
|---|---|
| Header | Tenant, workspace, version (p\*), event timestamp |
| Inventory items | Workloads / sets that the policy references, with their resolved IPs at the time of publish |
| Policies | Consumer / provider / service / action / rank for each rule |
| Catch-All | The workspace-level Catch-All semantics |

A consumer doesn't need every field; project to what's relevant
for your enforcement plane.

---

## A reference consumer in outline

Cisco publishes a [Reference Implementation of Secure Workload Network Policies Client](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#reference-implementation-of-secure-workload-network-policies-client)
that's worth reading before writing your own. The skeleton:

```python
from kafka import KafkaConsumer
import policy_pb2  # generated from the protobuf definition

consumer = KafkaConsumer(
    "csw-policies",                                   # topic name
    bootstrap_servers=["kafka.example.com:9093"],
    security_protocol="SSL",
    ssl_cafile="/etc/csw/kafka/ca.crt",
    ssl_certfile="/etc/csw/kafka/client.crt",
    ssl_keyfile="/etc/csw/kafka/client.key",
    auto_offset_reset="earliest",
    group_id="my-enforcer",
)

for msg in consumer:
    event = policy_pb2.PolicyUpdate()
    event.ParseFromString(msg.value)
    reconcile_local_firewall(event)
```

`reconcile_local_firewall` is the application-specific bit —
how your downstream system applies the new policy.

---

## Reconciliation discipline

Two patterns:

### Pattern A — full snapshot per event

Every event carries a complete view of the workspace's
published policy. The consumer replaces its local view with
the event's view on each message. Simple, idempotent, robust to
missed messages (the next event will be a complete snapshot).

### Pattern B — delta-only

The event carries only the diff vs. the previous version. The
consumer applies the delta. More bandwidth-efficient but fragile
to missed messages — must reconcile from a full snapshot
periodically.

**Default to Pattern A** unless bandwidth is a real constraint.
The robustness is worth the volume.

---

## Operational concerns

| Concern | Mitigation |
|---|---|
| Consumer falls behind / disconnects | Topic retention long enough to recover; alert on lag > N seconds |
| Consumer applies an old event after a newer one | Use the event's published version (p\*) number to drop stale events |
| Kafka cert expiring | Monitor expiry; rotate ahead of time |
| Schema evolution between releases | Use protobuf's compatibility rules; consumer should ignore unknown fields gracefully |
| Sensitive data in events | Inventory items contain workload identity; treat the topic as security-relevant — restrict who can subscribe |

---

## Caveats

The [Caveats](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#caveats)
sub-section in the chapter covers known limitations of the
publisher. Read it as part of the design phase.

---

## When *not* to use Policies Publisher

- For pure observability ("we want to see policy changes in our
  log pipeline"), the Activity Log API or audit feed is usually
  enough — Kafka is heavier than needed.
- For automation that *acts on* CSW (e.g. GitOps reconciliation),
  you're calling OpenAPI in the other direction. See
  [`05-gitops-pattern.md`](./05-gitops-pattern.md).
- For one-off integrations, a polling consumer of the OpenAPI
  is simpler.

---

## See also

- [`01-authentication.md`](./01-authentication.md) (for OpenAPI;
  Kafka uses its own mTLS)
- [`02-openapi-policies.md`](./02-openapi-policies.md)
- [`05-gitops-pattern.md`](./05-gitops-pattern.md)
- Cisco: [Policies Publisher](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#policies-publisher)
