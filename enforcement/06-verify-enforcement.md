# 06 — Verify Enforcement Works as Expected

After enabling enforcement, you need to confirm — not assume —
that the host firewall on every workload reflects the published
policy *and* is correctly affecting traffic.

> **Cisco source.** [Manage Policy Lifecycle — Verify Enforcement Works as Expected](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#verify-enforcement-works-as-expected).

---

## The two layers to verify

```
   ┌──────────────────────────────────────────────────────┐
   │ Layer 1 — Cluster says "policy is enforcing"         │
   │   Workspace state:  Enforcing                        │
   │   Agent type column: Enforcement                     │
   │   Live Analysis / Enforcement Reporting clean        │
   └─────────────────────────┬────────────────────────────┘
                             │ but separately…
                             ▼
   ┌──────────────────────────────────────────────────────┐
   │ Layer 2 — Host actually reflects the policy          │
   │   Host firewall contains expected rules              │
   │   Probes confirm allowed flows succeed               │
   │   Probes confirm not-allowed flows are dropped       │
   └──────────────────────────────────────────────────────┘
```

Layer 1 is a cluster-side claim. Layer 2 is the truth on the
host. Both must agree.

---

## Layer 1 — Cluster-side verification

In the workspace's *Enforce* / *Policy Enforcement* view, confirm:

| What | Healthy |
|---|---|
| Workspace enforcement state | `Enforcing` (or `Simulating` if mid-rollout) |
| Workloads in scope | Count matches expected |
| Agent type for each workload | `Enforcement` (was `Deep Visibility`) |
| Last policy push time | Recent — within the last few minutes |
| Rejected flows trend | At-or-near zero, matching what Live Analysis predicted |
| Allowed-but-dropped count (policy permits, network drops) | Stable; not climbing — a rise after go-live can indicate non-policy issues (host firewall, NAT, app crash) |
| Health alerts | None active for this scope |

In *Investigate → Flows*, sample a window from the last hour
and verify:

- Expected high-volume flows (web → app, app → db, etc.) appear
  as permitted.
- Deny rules (Catch-All Deny on this workspace, Absolute Denies
  inherited from parent scopes) appear as rejected for any
  flow that matches them.
- No surprise rejections.

---

## Layer 2 — Host-side verification

Sample 3–5 agents at random from the population. On each:

### Linux

```bash
# Confirm the agent's chains are installed and populated
sudo iptables -L -n -v
sudo nft list ruleset

# Look for CSW-managed chains (names depend on agent version)
sudo iptables -L -n -v | grep -i 'tet\|csw'

# Check rule counts on the agent's chains — should match
# the policy's rule count (allowing for chain wrapping rules)

# Service health
sudo service tet-sensor status
sudo journalctl -u tet-sensor --since "30 minutes ago"
```

### Windows

```powershell
# Find the CSW-managed filters (admin)
Get-NetFirewallRule |
  Where-Object { $_.DisplayName -like "*tet*" -or $_.DisplayName -like "*csw*" } |
  Select-Object DisplayName, Direction, Action, Enabled

# Or via WFP for the full picture
netsh wfp show filters file=C:\temp\wfp.xml

# Service health
Get-Service tet-sensor
Get-EventLog -LogName Application -Source tet-sensor -Newest 50
```

### Kubernetes

```bash
# DaemonSet rollout
kubectl -n tetration get ds
kubectl -n tetration get pods -o wide

# Per-node iptables (exec into the agent pod or onto the node)
kubectl -n tetration exec -it <agent-pod> -- iptables -L -n -v
```

### Probe-based verification

For a few workloads, **actively probe** the policy:

| Probe | Expected result if policy is correct |
|---|---|
| From web tier → app tier on tcp/8443 | Permitted (the well-known allow rule) |
| From web tier → db tier on tcp/5432 | **Denied** (Catch-All — web is not allowed direct DB access) |
| From an out-of-scope host → app tier on tcp/8443 | **Denied** unless explicitly allowed |
| From a non-prod host → prod tier on any port | **Denied** by Absolute Deny |

Use safe probing — `nc` / `curl` / similar from a host you
control. Don't probe from production-critical paths.

---

## What "verified" looks like

Document, attached to the change ticket:

- ✅ Workspace state confirmed `Enforcing` at time T
- ✅ Agent count in `Enforcement` mode matches expected
- ✅ Sample of N agents — host firewall contains expected rules
- ✅ Active probe results — known-allowed flow permitted,
     known-denied flow blocked
- ✅ Live Analysis / Enforcement Reporting trend reviewed —
     rejected-flow count behaves as predicted
- ✅ App synthetic checks remain green at T+15, T+60

If any of those is missing, you don't have verified enforcement.
Don't move to the next wave until you do.

---

## Common gaps found at verification

| Gap | Likely root cause | Action |
|---|---|---|
| Cluster says enforcing, host firewall shows no CSW chains | Agent is in unhealthy / degraded state, didn't apply rules | Repair agent (per [`02-agent-readiness.md`](./02-agent-readiness.md)) before continuing |
| Cluster says enforcing, host firewall has stale rules | Push not propagated yet, or agent stuck on an older policy version | Force re-push from cluster; check agent connectivity |
| Probe to "should-allow" flow is denied | Inventory filter on the rule's consumer / provider doesn't match the prober's labels | Check labels; reconcile filter or relabel |
| Probe to "should-deny" flow is allowed | A wider Allow rule (often inherited from parent scope) covers the flow | Tighten the parent's rule, or add a workspace-level Deny |

---

## See also

- [`03-enable-enforcement.md`](./03-enable-enforcement.md)
- [`05-platform-specific.md`](./05-platform-specific.md)
- [`11-monitoring-after-enforcement.md`](./11-monitoring-after-enforcement.md)
- [`../operations/03-troubleshooting-blocked-flows.md`](../operations/03-troubleshooting-blocked-flows.md)
- Cisco: [Verify Enforcement Works as Expected](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#verify-enforcement-works-as-expected)
