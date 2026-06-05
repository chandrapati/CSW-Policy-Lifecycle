# FQDN / Domain-Based Policy — and why "URL segmentation" is a different thing

A question that comes up in almost every evaluation: *"Can
Secure Workload segment on URLs or FQDNs?"* The honest, precise
answer is:

- **FQDN / domain — yes.** CSW has **domain-based policy
  enforcement** (introduced in **release 3.9**). You can write
  Allow/Deny policy against a domain such as `*.amazonaws.com`.
- **URL (scheme + host + path) — no.** CSW matches on the
  **domain name only**. It does **not** parse HTTP and cannot
  enforce on a path like `https://host/api/v1/orders` vs
  `/admin`. True URL / web-category filtering is a proxy / SWG /
  NGFW function — pair CSW with **Cisco Secure Access / Umbrella**
  or **Secure Firewall** for that.

> **Cisco source.** [Secure Workload 3.9 — domain-based policy
> enforcement](https://blogs.cisco.com/security/cisco-secure-workload-3-9-delivers-stronger-security-and-greater-operational-efficiency)
> and [Manage Inventory — Create a Domain
> Filter](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/3_9/cisco-secure-workload-user-guide-on-prem-v39/manage-inventory-for-secure-workload.html).

---

## What "domain-based policy" actually is

A **Domain Filter** is a special kind of inventory filter whose
object type is `DOMAIN` (instead of the usual `INVENTORY`). You
then use that filter as the **consumer** or **provider** of a
policy — most commonly the **provider** of an *egress* Allow rule
to an external service the workload legitimately needs, or a
**Deny** rule to a known-bad domain.

| | `INVENTORY` filter | `DOMAIN` filter |
|---|---|---|
| Matches | Workloads, services, pods, IP addresses | Domain names only |
| Built from | Labels / annotations / subnets | One or more domain names |
| Typical use | East-west workload-to-workload policy | Egress allow-list / deny to external FQDNs |
| Facet available | All labels | `domain name` only |

---

## How it enforces (the important mechanic)

Enforcement is still **IP-based at the kernel firewall** — the
FQDN is the *intent*, and the agent keeps the IP set current by
watching DNS:

```
   Workload resolves api.example.com
            │  (plaintext DNS query / response)
            ▼
   ┌────────────────────────────┐
   │  CSW agent snoops DNS       │   Windows: ETW session CSW_MonDns
   │  captures A / AAAA records  │   Linux:   enforcer DNS snoop
   └─────────────┬──────────────┘
                 │ resolved IPs
                 ▼
   ┌────────────────────────────┐
   │  Agent programs host FW     │   Linux: iptables/ip6tables + ipset
   │  with the resolved IPs      │   Windows: WFP filters
   └────────────────────────────┘
```

Because the agent re-learns IPs from every DNS response, the
policy survives CDN / SaaS IP churn — which is exactly why a
static IP allow-list doesn't work for those destinations.

> **Operational consequence — encrypted DNS breaks it.** The
> agent must *see* the DNS resolution in cleartext. If the
> workload uses DoH / DoT (encrypted DNS), the snooper is blind
> and FQDN matching will not work. Confirm the workloads in
> scope use standard DNS before relying on domain policy.

---

## Rules and caveats (from the 3.9 guide — quote these in a POV)

| Area | Rule |
|---|---|
| **Wildcards** | Allowed on the **first label only**: `*.amazon.com` ✓ · `aws.*.com` ✗ · `aws*.com` ✗ (no regex-style mixing) |
| **Wildcard scope** | `*.yahoo.com` matches `finance.yahoo.com`, `web.finance.yahoo.com`, etc. — **but not** the apex `yahoo.com` |
| **`www`** | Treated as a subdomain: `google.com` and `www.google.com` are **distinct** domains |
| **Minimum form** | Must include at least a second-level domain (`*.cisco.com`, even `*.com`) |
| **Filter logic** | A domain facet may combine with another facet using **OR only**, never **AND** (e.g. `domain name=*.google.com OR hostname contains mach` is valid; the same with `AND` is not) |
| **Scoping** | Domain filters **cannot be restricted to a scope**, and they return **no matching inventory items** |
| **TCP behavior** | When a domain is blocked, the **first packet may pass**, but the connection is dropped before the handshake completes |
| **Proxy support** | In conversation mode, only certain proxies (HTTP proxy, TCP) are supported for domain enforcement |

---

## When to use it (and when not to)

| Use case | Right tool |
|---|---|
| East-west, workload-to-workload microsegmentation | CSW label / scope-based IP policy — the core of this repo |
| Egress allow-list to known SaaS / API FQDNs (`*.amazonaws.com`, `*.windowsupdate.com`) | **CSW domain filter** |
| Block known-malicious domains / threat-intel IPs | **CSW domain filter** + integrated threat-intel feeds (3.9+) |
| Allow/deny by **URL path** (`/admin` vs `/api`) | Secure Firewall / Secure Access — **not** CSW |
| Web category / SWG / TLS inspection | Umbrella SIG / Secure Access — **not** CSW |

---

## How to create one (UI)

1. **Organize → Inventory Filters** (or, inside a workspace,
   **Manage Policies → Filters → Inventory Filters**).
2. **Create Filter / Add Inventory Filter.**
3. Check the **Domain Filter** box.
4. Enter a name and a query — e.g. `domain name = *.amazonaws.com`
   — then **Next**.
5. Review and **Create**.
6. Use the new domain filter as the **provider** (egress) or
   **consumer** of an Allow/Deny policy, then publish and enforce
   via the normal lifecycle (see
   [`enforcement/03-enable-enforcement.md`](../enforcement/03-enable-enforcement.md)).

---

## See also

- [`docs/04-policy-attributes.md`](./04-policy-attributes.md) —
  rank, consumer / provider, how a domain filter slots into a
  policy
- [`discovery/06-external-dependencies.md`](../discovery/06-external-dependencies.md)
  — how ADM surfaces the external destinations you'd turn into
  domain policy
- [`docs/00-official-references.md`](./00-official-references.md)
  — the canonical Cisco User Guide links
- Cisco: [Create a Domain Filter (3.9)](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/3_9/cisco-secure-workload-user-guide-on-prem-v39/manage-inventory-for-secure-workload.html)
- Cisco: [Enforce Policies with Agents (4.0)](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/deploy-software-agents.html)
