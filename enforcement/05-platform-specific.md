# 05 — Platform-Specific Enforcement Notes

CSW abstracts policy authoring above the host firewall, but the
**enforcement primitive** still depends on the platform: Linux
iptables / nftables, Windows Filtering Platform (WFP), or the
container / Kubernetes equivalent. This page covers the
platform-specific things that can bite you.

> **Cisco source.** [Manage Policy Lifecycle — Enforcement on Containers](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#enforcement-on-containers)
> and [Software Agents](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/software-agents.html).

> **Compatibility.** What's supported depends on agent + host
> OS version. The
> [Cisco Secure Workload Compatibility Matrix](https://www.cisco.com/c/m/en_us/products/security/secure-workload-compatibility-matrix.html)
> is the authoritative source.

---

## Linux

### How the agent enforces

The CSW agent installs its own **chains** in iptables /
nftables (typically named with a `tetration_` or `csw_` prefix
in the relevant table). These chains contain the rules
resolved from the workspace's policy and are inserted such that
they take priority over user-defined chains for the flow types
they cover.

| Element | Where |
|---|---|
| Inbound rules | `INPUT` chain via the agent's hook |
| Outbound rules | `OUTPUT` chain via the agent's hook |
| Forward rules (containers) | `FORWARD` chain |

### Things that cause friction

| Issue | Symptom | Mitigation |
|---|---|---|
| Pre-existing busy iptables setup (e.g. heavy-handed bastion) | Agent reports degraded enforcement capability | Reconcile pre-existing rules; document which chains the host operator owns vs. the agent owns |
| nftables vs. iptables-legacy mixed setup | Some rules are visible in one tool, some in the other | Standardise on one; agent handles both but verifying state requires the right tool |
| `firewalld` / `ufw` running in parallel | Two managers fighting over the same kernel state | Disable the host-side firewall manager; let the CSW agent own iptables / nftables |
| SELinux denying agent operations | Agent install or enforcement failures | Ensure the agent's SELinux policy module is installed; check `audit.log` |

### Verifying on the host

```bash
# what's installed
sudo iptables -L -n -v
sudo nft list ruleset

# CSW-managed chains visible (names depend on agent version)
sudo iptables -L -n -v | grep -i 'tet\|csw'

# agent-side health
sudo service tet-sensor status
sudo journalctl -u tet-sensor --since "1 hour ago"
```

---

## Windows

### How the agent enforces

The CSW agent uses **Windows Filtering Platform (WFP)** —
specifically the inbound / outbound transport layer filters.
The agent registers callout drivers and install filters that
match the resolved rules.

| Element | Where |
|---|---|
| Inbound rules | WFP inbound transport layer filters |
| Outbound rules | WFP outbound transport layer filters |
| Listener-bind blocks | When policy denies bind on a port, may also use ALE bind layer |

### Things that cause friction

| Issue | Symptom | Mitigation |
|---|---|---|
| Group Policy pushing conflicting Windows Firewall rules | GPO rules and CSW rules both in WFP, contention | Coordinate with AD / GPO team; CSW manages its own filter set |
| Third-party AV / EDR with deep WFP integration | Possible stack conflicts; agent may report degraded | Vendor-specific reconciliation; check the [Compatibility Matrix](https://www.cisco.com/c/m/en_us/products/security/secure-workload-compatibility-matrix.html) for known interactions |
| Older Windows Server SKUs (out of support) | Agent supports visibility, may not support enforcement | Upgrade or skip enforcement on those hosts |
| WSL / containers on Windows host | Host firewall behaviour around namespaced traffic depends on configuration | Test in lab before enforcing |

### Verifying on the host

```powershell
# WFP filter inventory (requires admin)
Get-NetFirewallRule | Where-Object { $_.DisplayName -like "*tet*" -or $_.DisplayName -like "*csw*" }

# tighter view via WFP tooling
netsh wfp show filters file=C:\temp\wfp.xml

# agent-side health
Get-Service tet-sensor
Get-EventLog -LogName Application -Source tet-sensor -Newest 50
```

---

## Containers and Kubernetes

### How the agent enforces

Two deployment modes:

1. **Host-level agent** on each Kubernetes node. Enforces using
   iptables / nftables on the node, against pod-side traffic.
   Pod identity is associated through CNI / kubelet integration.
2. **Sidecar / namespace-aware agent** patterns (release-dependent).

The relevant Cisco section is
[Enforcement on Containers](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#enforcement-on-containers).

### Things that cause friction

| Issue | Symptom | Mitigation |
|---|---|---|
| CNI not on the [supported list](https://www.cisco.com/c/m/en_us/products/security/secure-workload-compatibility-matrix.html) | Visibility works; enforcement not validated | Stick to supported CNIs for enforcement, or accept visibility-only |
| iptables locking conflicts with CNI's own iptables management (Calico, Cilium with iptables backend, etc.) | Race conditions, contested chains | Read agent + CNI release notes for known interactions; test rollout on a node group before broadening |
| `NetworkPolicy` resources in the cluster | Two policy stacks (NetworkPolicy + CSW) on the same flows | Decide who owns which surface; usually CSW owns east-west, NetworkPolicy stays for application-team workflows |
| DaemonSet rollout incomplete | Some nodes lack agents → unenforced pods if scheduled there | Enforce DaemonSet completeness before enabling CSW enforcement |
| ServiceMesh sidecars (Istio / Linkerd) intercepting traffic | Flows look different to the agent than expected | Plan around — typically CSW enforces between pods/nodes, mesh handles intra-app concerns; verify in lab |

### Verifying

```bash
# DaemonSet rollout status
kubectl -n tetration get ds
kubectl -n tetration get pods -o wide

# per-node agent health
kubectl -n tetration logs -l app=tet-sensor --tail=200
```

---

## Cross-platform: pre-existing host firewall reconciliation

For all platforms, *what was on the host before the CSW agent
started enforcing* is part of the operational picture:

| Pre-existing rule type | What CSW does | What you should do |
|---|---|---|
| Host-defined Allow rules (admin / dev SSH from bastion, etc.) | CSW typically inserts its rules without removing host-defined ones, but ordering matters | Verify post-enforcement that the bastion / admin paths still work |
| Host-defined Deny rules (deny inbound from a specific CIDR, etc.) | Coexist; the most-restrictive wins on the rule set's overall semantics | Consolidate into CSW policy where reasonable; or document why the host-side Deny stays |
| Tools-installed rules (AV, EDR, monitoring) | Coexist; may need explicit accommodation | Coordinate with the tool's owning team |

The first time enforcement goes live on an agent, **inspect the
host firewall state on a sample of agents** within 60 minutes
to confirm no conflicts. See
[`06-verify-enforcement.md`](./06-verify-enforcement.md).

---

## See also

- [`02-agent-readiness.md`](./02-agent-readiness.md)
- [`06-verify-enforcement.md`](./06-verify-enforcement.md)
- Companion repo: [`chandrapati/CSW-Agent-Installation-Guide`](https://github.com/chandrapati/CSW-Agent-Installation-Guide)
- Cisco: [Software Agents](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/software-agents.html)
- Cisco: [Enforcement on Containers](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#enforcement-on-containers)
- Cisco: [Compatibility Matrix](https://www.cisco.com/c/m/en_us/products/security/secure-workload-compatibility-matrix.html)
