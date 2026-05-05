# Programmatic Management — OpenAPI and Kafka

Most of this repo describes UI-driven workflow. The same lifecycle
— discovery, analysis, enforcement, version management — is
fully programmable via CSW's **OpenAPIs**, and the
**Policies Publisher (Kafka)** stream lets downstream systems
subscribe to policy events as they happen.

> **Cisco source.** [Secure Workload OpenAPIs (4.0 On-Prem)](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/secure-workload-openapis.html)
> and [Manage Policy Lifecycle — Policies Publisher](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#policies-publisher).

---

## When to use the API path

| Use case | Why API |
|---|---|
| **GitOps** — policy as code, reviewed in PRs, reconciled to cluster | Single source of truth in git; auditable history; same review workflow as application code |
| **Bulk operations** — onboarding many apps at once | Click-by-click in UI doesn't scale to 200 workspaces |
| **Incident-response automation** — scripted disable / revert | The disable button is faster when wrapped in a runbook script |
| **Integration with external tools** — CMDB, change-management, ticketing | Policy events flow alongside other infra events |
| **Custom analytics / reporting** — pulling policy state for compliance dashboards | Don't screen-scrape the UI; use OpenAPI |
| **Subscribe to policy events** (Kafka) | Downstream systems get policy updates pushed, not polled |

For one-off authoring, the UI is faster and lower-risk. The API
path is for workflows that **repeat** or that need **audit
weight**.

---

## What's in this folder

| File | Purpose |
|---|---|
| [`01-authentication.md`](./01-authentication.md) | API key + secret; OpenAPI authentication mechanics |
| [`02-openapi-policies.md`](./02-openapi-policies.md) | Create / update / list / publish policies via OpenAPI |
| [`03-enforcement-toggle-api.md`](./03-enforcement-toggle-api.md) | Enable / disable enforcement on a workspace via API |
| [`04-policies-publisher-kafka.md`](./04-policies-publisher-kafka.md) | Subscribe to policy events via the Kafka publisher |
| [`05-gitops-pattern.md`](./05-gitops-pattern.md) | The end-to-end GitOps pattern for CSW policy as code |

---

## A note on the OpenAPI surface

The OpenAPI is **large** (~366 KB / 18,000+ rendered lines in
the User Guide chapter). This folder doesn't try to mirror it
— that would just rot. It covers the patterns that come up
repeatedly during a CSW policy programme:

- Auth (every script needs it).
- Policy CRUD on a workspace.
- Enforcement toggle (incident response + GitOps reconcile).
- Kafka subscription (downstream integration).
- The end-to-end GitOps loop.

For specific endpoints not covered here, the
[OpenAPIs chapter](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/secure-workload-openapis.html)
is the authoritative reference.

---

## Code samples

The samples in this folder use a mix of `curl` (for clarity)
and Python (for realistic scripted use). Translate to your
preferred language; CSW supplies a Python SDK that wraps the
OpenAPI in conveniences.

---

## See also

- [`../enforcement/`](../enforcement/README.md) — UI-driven flows
- [`../analysis/08-import-export.md`](../analysis/08-import-export.md)
  — JSON / CSV import-export, the foundation for GitOps
- [`../operations/`](../operations/README.md) — day-2 ops, much
  of which is API-friendly
- Cisco: [Secure Workload OpenAPIs](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/secure-workload-openapis.html)
- Cisco: [Policies Publisher](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#policies-publisher)
