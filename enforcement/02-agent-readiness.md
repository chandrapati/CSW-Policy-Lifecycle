# 02 — Agent Readiness for Enforcement

Before flipping the enforcement bit on a workspace, every agent
on every workload in the workspace's scope must be **healthy
and capable of enforcing**. CSW will surface issues but does
not refuse to enable enforcement on top of unhealthy agents —
that's your job.

> **Cisco source.** [Manage Policy Lifecycle — Check Agent Health and Readiness to Enforce](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#check-agent-health-and-readiness-to-enforce).

---

## What "ready to enforce" means

| Dimension | Healthy state |
|---|---|
| **Agent process** | Running on the host; PID stable; not flapping |
| **Cluster registration** | Last check-in within the threshold (default ~5 min) |
| **Flow telemetry** | Reporting flows continuously over the past 24 h |
| **Configuration** | Agent type / mode is *Deep Visibility* (about to be promoted to Enforcement) |
| **OS / kernel** | On the [Compatibility Matrix](https://www.cisco.com/c/m/en_us/products/security/secure-workload-compatibility-matrix.html) for the agent version |
| **Host firewall capability** | Linux: `iptables` / `nftables` available and not heavily contested; Windows: WFP healthy |
| **No conflicting host-FW rules** | Pre-existing rules don't conflict with what CSW will install |

The CSW UI exposes a *Readiness to Enforce* indicator on each
workload — green / amber / red. Red workloads should be excluded
from enforcement until fixed.

---

## Where to look in the UI

*Manage → Agents → Software Agents.* Sort or filter on:

| Column | Healthy |
|---|---|
| **Status** | `OK` |
| **Type** | `Deep Visibility` (Visibility before enforcement; will become `Enforcement` after enable) |
| **Last Checkin** | Within the threshold |
| **Software Version** | Supported by the [Compatibility Matrix](https://www.cisco.com/c/m/en_us/products/security/secure-workload-compatibility-matrix.html) for both your CSW release and the host OS |
| **Enforcement Readiness** *(where present in the release)* | `Ready` / Green |
| **Health** | No active alerts |

If your release doesn't expose every column listed, the
[Cisco section](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#check-agent-health-and-readiness-to-enforce)
shows the equivalents.

---

## Common readiness blockers

### Linux

| Blocker | Symptom | Fix |
|---|---|---|
| Heavy pre-existing iptables / nftables rules | Agent reports degraded enforcement capability | Reconcile pre-existing rules; CSW agent generally manages its own chains, but very contested setups need analysis |
| Container-host with bespoke iptables management (legacy Docker, custom CNI) | Conflicts with agent's chain insertion | Verify the agent's container-mode behaviour; see [`05-platform-specific.md`](./05-platform-specific.md) |
| Old kernel without nftables / netfilter features the agent expects | Agent runs but enforcement capability limited | Either upgrade kernel (per [Compatibility Matrix](https://www.cisco.com/c/m/en_us/products/security/secure-workload-compatibility-matrix.html)) or skip enforcement on those hosts |

### Windows

| Blocker | Symptom | Fix |
|---|---|---|
| Group Policy pushing conflicting WFP rules | Agent reports degraded enforcement capability | Coordinate with AD / GPO team; isolate CSW's WFP filters from GPO-managed ones |
| Third-party AV / EDR with deep WFP integration | Possible stack conflicts | Vendor-specific reconciliation; check [Compatibility Matrix](https://www.cisco.com/c/m/en_us/products/security/secure-workload-compatibility-matrix.html) for known interactions |
| Older Windows Server SKUs out of support window | Agent runs visibility but not enforcement | Upgrade or skip |

### Kubernetes

| Blocker | Symptom | Fix |
|---|---|---|
| CNI not on the supported list | Agent visibility works but enforcement is not validated | Confirm against [Compatibility Matrix](https://www.cisco.com/c/m/en_us/products/security/secure-workload-compatibility-matrix.html); see [`05-platform-specific.md`](./05-platform-specific.md) |
| Agent DaemonSet not deployed to all nodes | Some nodes lack agents → policy not enforced on pods scheduled there | Reconcile DaemonSet rollout |

---

## A 5-minute pre-enforcement readiness check

For a workspace `payments-api`, before enabling enforcement:

1. Filter the *Software Agents* page by scope `payments-api`.
2. Verify total agent count matches expected workload count
   (within ±5 %; missing agents are blockers).
3. Confirm all `Status` = `OK`, `Last Checkin` recent.
4. Spot-check 5 random workloads — drill into agent details,
   confirm version is supported on the host OS per the
   [Compatibility Matrix](https://www.cisco.com/c/m/en_us/products/security/secure-workload-compatibility-matrix.html).
5. Confirm no agents in *Degraded* / amber / red.
6. Sample 3 agents; verify the host's actual firewall state
   (`iptables -L -n` on Linux, `Get-NetFirewallProfile` on
   Windows) — confirm there are no contested chains / filters
   the CSW agent's enforcement would conflict with.

If all five pass, agent layer is ready. Move to
[`03-enable-enforcement.md`](./03-enable-enforcement.md).

---

## What if some workloads aren't ready

Three options:

1. **Defer enforcement** until the unready workloads are
   remediated. Most conservative; keeps the rollout coherent.
2. **Carve out the unready workloads** via a sub-scope or
   inventory filter and exclude from this workspace's
   enforcement. Continue with the ready set.
3. **Push enforcement anyway** and accept that some workloads
   will be in inconsistent state. **Don't do this.** It's the
   path to incidents that take days to diagnose.

Option 1 is almost always right.

---

## See also

- [`01-pre-enforcement-checklist.md`](./01-pre-enforcement-checklist.md)
- [`03-enable-enforcement.md`](./03-enable-enforcement.md)
- [`05-platform-specific.md`](./05-platform-specific.md)
- Companion repo: [`chandrapati/CSW-Agent-Installation-Guide`](https://github.com/chandrapati/CSW-Agent-Installation-Guide)
- Cisco: [Check Agent Health and Readiness to Enforce](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#check-agent-health-and-readiness-to-enforce)
- Cisco: [Compatibility Matrix](https://www.cisco.com/c/m/en_us/products/security/secure-workload-compatibility-matrix.html)
