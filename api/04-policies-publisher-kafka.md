# 04 — Policies Publisher (Kafka)

CSW exposes published network policies on a **Kafka topic** so
that third-party enforcement code can consume policy as it
changes — typically to drive enforcement on appliances (load
balancers, firewalls) where CSW doesn't have an agent.

> **Cisco source.** [Manage Policy Lifecycle — Policies Publisher](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#policies-publisher).

---

## What it is — exactly

Direct quote from the User Guide:

> *"Policies Publisher is an advanced Cisco Secure Workload
> feature allowing third-party vendor to implement their own
> enforcement algorithms that are optimized for network
> appliances such as load balancers or firewalls. This feature
> is realized by publishing defined policies to a Kafka instance
> residing within Secure Workload cluster and by providing
> customers with Kafka client certificates, which allows
> third-party vendor code to retrieve policies from Kafka and
> to translate them into their network appliances configuration
> appropriately."*

Two key facts to internalise:

1. The Kafka instance lives **inside** the Secure Workload
   cluster — it's not a Kafka you provide. You connect to CSW's
   Kafka.
2. Cisco's documentation describes the procedure with **Java
   on Linux**. The reference implementation is in Java
   (see below).

```
   CSW cluster                                  Downstream client
   ┌──────────────┐                                   │
   │  Internal    │                                   │
   │  Kafka       │ ─── policy update event ─►        │
   │  (CSW-       │                                   ▼
   │   hosted)    │                            translate to
   └──────────────┘                            appliance config
```

---

## Common downstream consumers

- A **load balancer** (e.g. F5) that programs equivalent ACLs
  from CSW policy.
- A **network firewall** outside the CSW agent footprint that
  mirrors a subset of policy.
- A **custom enforcer** for a platform CSW doesn't natively
  support.
- An **observability pipeline** that wants policy events in the
  same store as flow events. (Often overkill — see "When not to
  use" below.)

---

## Granularity — per root scope, not per workspace

Per Cisco:

> *"Each policy contains all the items belonging to one root
> scope."*

So a downstream consumer subscribes to **the policy stream for
a root scope** — not per individual workspace. The published
message contains *all* enforced policy for that root scope as a
single coherent snapshot.

If no workspace with enforcement enabled exists for the root
scope, Cisco emits a **default ALLOW** policy:

> *"If no workspace with enforcement enabled exists for the
> root scope, a default policy of ALLOW is written to the
> produced policy."*

---

## Prerequisites

Per Cisco's [Policies Publisher prerequisites](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#policies-publisher),
the documented stack is Java on Linux:

| Component | Cisco-stated version (verify against your release) |
|---|---|
| [Apache Kafka Clients](https://kafka.apache.org/) | `kafka-clients-1.0.0.jar` |
| [Protocol Buffers Core](https://github.com/protocolbuffers/protobuf) | `protobuf-java-3.4.1.jar` |

Check the chapter for your release; specific JAR versions may
have moved.

---

## Getting Kafka client certificates

Per Cisco's exact procedure ([Getting Kafka Client Certificates](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#getting-kafka-client-certificates)):

1. **Enable enforcement on the workspace first.** Per Cisco:
   *"Perform policies enforcement as described in Enforce
   Policies. This first step is necessary as it creates a Kafka
   topic that is associated with active scope."* Without
   enforcement enabled, there's no topic for you to subscribe
   to.
2. Navigate to the **Data Taps** tab in the cluster's
   management UI.
3. Download Kafka client certificates by clicking the download
   button under the **Actions** column.
4. **Select "Java Keystore" format** in the download dialog.
5. Configure your client to authenticate to Kafka using the
   keystore.

These certs **expire** — manage their lifecycle the same way
you'd manage any other certificate (renewal cadence, rotation
plan, monitoring approaching expiry).

---

## The protobuf data model

Per Cisco's [Data Model of Secure Workload Network Policy](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#data-model-of-secure-workload-network-policy):

> *"A Secure Workload Network Policy as modeled in protobuf
> consists of a list of **InventoryGroups**, a list of
> **Intents** and a **CatchAll** policy."*

| protobuf concept | What it represents |
|---|---|
| `InventoryGroup` | A named group of `InventoryItem`s. Each `InventoryItem` represents a CSW entity (server, appliance) by its network address — singular IP, subnet, or address range. |
| `Intent` | A rule: action (ALLOW or DENY) when a network flow matches the given consumer's `InventoryGroup`, the provider's `InventoryGroup`, and the network protocol/port. |
| `CatchAll` | The catch-all action defined for the root scope. |
| `TenantNetworkPolicy.ScopeInfo` | Metadata about which root scope this policy is for (only present in the first fragment if a message is split — see below). |

The schema and full field reference are in
[Protobuf Definition File](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#protobuf-definition-file).
Generate language bindings from it for your client language.

---

## How updates arrive — `KafkaUpdate` semantics

Per Cisco:

> *"When an enforcement is triggered by the users or by a
> change of inventory groups, Secure Workload backend sends a
> **full snapshot** of defined network policies to Kafka as a
> sequence of messages that are represented as `KafkaUpdate`s."*

So **every update is a full snapshot** of the root scope's
policy. There is no native delta protocol — your consumer
replaces its local view on each `KafkaUpdate`.

### Message fragmentation

> *"In case `KafkaUpdate` message size is greater than 10MB,
> Secure Workload backend splits this message into multiple
> fragments, each of size 10MB. If there is multiple fragments,
> only the first fragment has the `ScopeInfo` field of
> `TenantNetworkPolicy`. The `ScopeInfo` will be set to nil in
> the remaining fragments of `KafkaUpdate` message."*

So a consumer **must**:

- Detect multi-fragment messages (presence/absence of
  `ScopeInfo`).
- Reassemble fragments before applying the policy.
- Refer to comments in the `tetration_network_policy.proto`
  file for the exact reassembly logic.

---

## Reference implementation (Java)

Per Cisco's [Reference Implementation of Secure Workload
Network Policies Client](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#reference-implementation-of-secure-workload-network-policies-client),
Cisco maintains a reference implementation **in Java** that
handles Kafka subscription and protobuf parsing for you:

- **Repository:**
  [`tetration-exchange/pol-client-java`](https://github.com/tetration-exchange/pol-client-java)
  (URL exact at time of writing — confirm in the chapter for
  your release).
- **Plug-in interface:**
  [`PolicyEnforcementClient`](https://github.com/tetration-exchange/pol-client-java/blob/master/src/main/java/com/tetration/network_policy/enforcement/PolicyEnforcementClient.java)
  is what you implement to translate `Intent`s into your
  appliance's configuration.

**Recommendation:** start by reading and (where possible)
extending the reference implementation rather than writing a
client from scratch. The fragment-reassembly and reconnect
logic is already correct in the reference.

---

### A non-Java client outline (illustrative)

If you must write the client in another language, you generate
language bindings from the protobuf and use a native Kafka
client. The following Python skeleton is **illustrative only**
— Cisco's documented stack is Java, the schema bindings come
from the `.proto` file, and fragment reassembly is your
responsibility:

```python
from kafka import KafkaConsumer
import tetration_network_policy_pb2  # generated from the .proto

consumer = KafkaConsumer(
    "<topic-name-from-Data-Taps>",
    bootstrap_servers=["<broker-from-Data-Taps>"],
    security_protocol="SSL",
    ssl_cafile="/etc/csw/kafka/ca.crt",
    ssl_certfile="/etc/csw/kafka/client.crt",
    ssl_keyfile="/etc/csw/kafka/client.key",
    auto_offset_reset="earliest",
    group_id="my-enforcer",
)

# WARNING: this skeleton does NOT implement fragment reassembly.
# Real clients MUST handle KafkaUpdate fragments per the .proto comments.
for msg in consumer:
    update = tetration_network_policy_pb2.KafkaUpdate()
    update.ParseFromString(msg.value)
    if update.HasField("scope_info"):
        # first fragment of a (possibly multi-fragment) snapshot
        ...
    else:
        # continuation fragment
        ...
```

The reference Java implementation is the safer starting point.

---

## Reconciliation discipline

Because every `KafkaUpdate` is a full snapshot, the natural
pattern is **idempotent replace**:

1. Receive (and reassemble fragments of) a `KafkaUpdate`.
2. Build the desired local config from `InventoryGroups +
   Intents + CatchAll`.
3. **Diff** vs. current local config; apply the minimum set of
   changes to converge.
4. Record what was applied.

If the consumer is restarted or falls behind, it just consumes
the next snapshot and converges — no separate recovery path
needed.

Use the timestamp / version fields in the message to drop stale
events if the topic delivers out-of-order across restart, or to
sequence multiple snapshots received during catch-up.

---

## Operational concerns

| Concern | Mitigation |
|---|---|
| Consumer falls behind / disconnects | Topic retention long enough to recover; alert on lag > N seconds |
| Consumer applies an old snapshot after a newer one | Use the `KafkaUpdate` ordering / version fields to drop stale events |
| Kafka cert expiring | Monitor expiry; rotate ahead of time; the cert is downloaded from the Data Taps tab |
| Schema evolution between releases | Use protobuf's compatibility rules; consumer should ignore unknown fields |
| Multi-fragment messages dropped between fragments | Track ScopeInfo + reassembly state; resync on a new ScopeInfo-bearing fragment |
| Sensitive data in events | InventoryGroups carry workload identity; restrict topic subscription accordingly |

---

## Caveats

The [Caveats](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#caveats)
sub-section in the chapter lists known limitations of the
publisher (e.g. behaviour on first enforcement, default-policy
edge cases). Read it during design.

---

## When *not* to use Policies Publisher

- For pure observability ("we want to see policy changes in our
  log pipeline"), the Activity Log / Enforcement History APIs
  are usually enough — Kafka is heavier than needed.
- For automation that *acts on* CSW (e.g. GitOps
  reconciliation), you call OpenAPI in the other direction. See
  [`05-gitops-pattern.md`](./05-gitops-pattern.md).
- For one-off integrations, a polling consumer of the OpenAPI
  is simpler.

---

## See also

- [`01-authentication.md`](./01-authentication.md) (for
  OpenAPI; Kafka uses its own mTLS via Data Taps)
- [`02-openapi-policies.md`](./02-openapi-policies.md)
- [`05-gitops-pattern.md`](./05-gitops-pattern.md)
- Cisco: [Policies Publisher](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#policies-publisher)
- Cisco: [Getting Kafka Client Certificates](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#getting-kafka-client-certificates)
- Cisco: [Data Model of Secure Workload Network Policy](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#data-model-of-secure-workload-network-policy)
- Cisco: [Reference Implementation of Secure Workload Network Policies Client](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#reference-implementation-of-secure-workload-network-policies-client)
- GitHub (verify URL for your release): [tetration-exchange/pol-client-java](https://github.com/tetration-exchange/pol-client-java)
