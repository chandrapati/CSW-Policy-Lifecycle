# 03 — Enabling and Disabling Enforcement via API

The enforcement toggle is reachable via the OpenAPI. Wiring
it into your incident-response automation and CI is one of
the highest-leverage things you can do.

> **Authoritative source.**
> [Secure Workload OpenAPIs chapter](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/secure-workload-openapis.html).
> The exact endpoint paths, request bodies, and response shapes
> are release-specific. **The paths shown below are illustrative
> patterns reflecting common Tetration / Secure Workload
> conventions; verify the exact paths in the OpenAPIs chapter
> for your release before depending on them.**

---

## What the endpoints do

| Conceptual operation | Illustrative endpoint pattern | Effect |
|---|---|---|
| Enable enforcement on a workspace | `POST /openapi/v1/applications/{app_id}/enable_enforce` (verify) | UI equivalent: re-running the [Policy Enforcement Wizard](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#policy-enforcement-wizard) |
| Disable enforcement on a workspace | `POST /openapi/v1/applications/{app_id}/disable_enforce` (verify) | UI equivalent: red **Stop Policy Enforcement** button on the workspace's Policy Enforcement page |
| Revert to a previous version | (no separate endpoint per Cisco's UI flow — re-call enable-enforce with the older version) | Cisco's documented procedure is to *"follow one of the processes that are described in Enable Policy Enforcement and choose an earlier version to enforce"* — see [`../enforcement/09-rollback-and-revert.md`](../enforcement/09-rollback-and-revert.md) |

---

## Wiring disable into incident response

The single most valuable use of these endpoints is the
**emergency disable button** for on-call. The pattern:

```python
# disable.py — call from a runbook or chatops
# Illustrative skeleton: replace the import and method names
# with whatever the Python helper your CSW release ships uses
# (see api/01-authentication.md and the OpenAPIs chapter).
import os, sys
# from <your-release-helper-package> import RestClient

CLUSTER  = os.environ["CSW_CLUSTER"]
KEY      = os.environ["CSW_API_KEY"]
SECRET   = os.environ["CSW_API_SECRET"]

WORKSPACE_BY_APP = {
    "payments-api": "5f1e...",
    "crm-web":      "6a2f...",
    # … filled from the GitOps repo for accuracy
}

app    = sys.argv[1]
app_id = WORKSPACE_BY_APP[app]

client = RestClient(CLUSTER, api_key=KEY, api_secret=SECRET)
# Verify the exact path against the OpenAPIs chapter for your release.
resp   = client.post(f"/openapi/v1/applications/{app_id}/disable_enforce")
resp.raise_for_status()
print(f"Enforcement disabled for {app} (workspace {app_id}).")
```

Wrap this in a chatops command (`/csw-disable payments-api`),
a runbook step, or both. Test it ahead of time — see
[`../enforcement/10-pause-and-emergency-disable.md`](../enforcement/10-pause-and-emergency-disable.md).

Operational tips:

- **Source of truth for `WORKSPACE_BY_APP` is the GitOps repo.**
  Don't hand-maintain. Generate from the canonical config so a
  workspace ID change can't strand the script.
- **Log every invocation** with caller identity, timestamp, and
  the request ID — this is an incident-response artefact.
- **Page on use** — disable is a high-impact action; you want
  the team to see it happened in real time, not via the next
  morning's audit log review.
- **Lock down the credential** that this script uses to *only*
  `enable_enforce` / `disable_enforce` and only on the relevant
  workspaces. The blast radius of a leaked key for this script
  is "anyone can disable enforcement on these apps."

---

## Wiring enable into change-management automation

The forward path is symmetric. The exact request body shape is
release-specific (whether `mode` is a parameter, what values it
accepts, whether you supply a version ID or a workspace
identifier alone) — **consult the OpenAPIs chapter and treat
the example below as a skeleton, not a contract:**

```python
# Illustrative — verify body schema against your release.
resp = client.post(
    f"/openapi/v1/applications/{app_id}/enable_enforce",
    json_body=json.dumps({
        "version_id": "<the version you intend to enforce>",
    }),
)
```

Use this in CI/CD pipelines that gate on:

1. PR merged to the GitOps repo (the policy change is approved
   in git).
2. Quick Analysis / Policy Experiments run via API and clean.
3. Live Analysis window completed.
4. Change-management ticket in approved state.
5. **Optional staged step** — if your release exposes a
   non-enforcing pre-stage option (or you are using a separate
   non-enforcing agent type for "Simulate"), apply it first
   per [`../enforcement/04-rollout-pattern.md`](../enforcement/04-rollout-pattern.md).
6. Wait the configured pre-enforce observation window.
7. Call `enable_enforce` to commit.

Don't collapse 5 and 7 into a single step *if your release
supports a pre-stage*. The pre-stage is a guardrail. If your
release doesn't support a separate staging mode, run Live
Policy Analysis on the proposed version and only then enforce.

---

## Pre-flight checks the script should do

Before calling `enable_enforce`:

- Confirm the target workspace has the **expected** p\* as its
  current published version (read it back, compare).
- Confirm Live Analysis on the workspace is clean (or has been
  for the configured window).
- Confirm agent population health (count + status; refuse if
  too many unhealthy).
- Confirm the change ticket is in the right state (CMDB / ITSM
  integration).

Each of these is a 1–2 line check. They're cheap insurance.

---

## Error handling specific to enforcement toggle

| Error | Cause | What the script should do |
|---|---|---|
| 409 Conflict — workspace already enforcing | Idempotency gap, retry hit, or concurrent caller | Check current state; if already what you wanted, treat as success |
| 409 Conflict — workspace pending another change | A push in progress; e.g. a recent publish hasn't fully landed | Brief sleep and retry |
| 4xx with "agents not ready" | Some / all agents failing readiness | Don't auto-retry; alert humans |
| 5xx | Cluster-side issue | Retry with backoff; if persistent, escalate to TAC |

---

## Audit trail

Every API-driven enforcement toggle appears in the workspace's
Activity Log alongside UI actions. The activity entry records
the API key that made the call, which is why scoping API keys
narrowly (per [`01-authentication.md`](./01-authentication.md))
is important — *"`api-key-gitops-payments` enabled enforcement
at 14:02"* is far more useful for an audit than *"`api-key-tenant-admin`
enabled enforcement"*.

---

## See also

- [`01-authentication.md`](./01-authentication.md)
- [`02-openapi-policies.md`](./02-openapi-policies.md)
- [`05-gitops-pattern.md`](./05-gitops-pattern.md)
- [`../enforcement/03-enable-enforcement.md`](../enforcement/03-enable-enforcement.md)
- [`../enforcement/10-pause-and-emergency-disable.md`](../enforcement/10-pause-and-emergency-disable.md)
- [`../enforcement/09-rollback-and-revert.md`](../enforcement/09-rollback-and-revert.md)
