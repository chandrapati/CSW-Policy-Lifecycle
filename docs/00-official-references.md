# Official Cisco Documentation — Cross-Reference

**Read this page first.** Every other page in this repo is a
companion to one or more of the Cisco-published documents listed
below. When the two disagree, **the Cisco docs win**.

This page is intentionally comprehensive — keep it as a single
hub so customer-facing engagements can quote canonical URLs.

---

## Master User Guide chapters (Release 4.0)

The CSW User Guide is published in two flavours that share the
same chapter set; pick the one that matches your tenant
deployment model.

| Chapter | 4.0 On-Premises | 4.0 SaaS |
|---|---|---|
| Documentation landing page | [Secure Workload 4.0 docs](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/landing-page/secureworkload-40-docs.html) | (same) |
| User Guide root | [On-Prem 4.0 User Guide](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40.html) | [SaaS 4.0 User Guide](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-saas-v40.html) |
| **Manage Policy Lifecycle** *(this repo's primary reference)* | [On-Prem](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html) | (same chapter, SaaS variant) |
| **Manage Inventory for Secure Workload** *(scopes, labels, filters)* | [On-Prem](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-inventory-for-secure-workload.html) | (same chapter, SaaS variant) |
| **Get Started with Cisco Secure Workload** | [On-Prem](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/get-started-with-cisco-secure-workload.html) | (same chapter, SaaS variant) |
| **Secure Workload OpenAPIs** *(used by [`api/`](../api/README.md))* | [On-Prem](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/secure-workload-openapis.html) | (same chapter, SaaS variant) |
| Software Agents *(referenced from [companion install guide](https://github.com/chandrapati/CSW-Agent-Installation-Guide))* | [On-Prem](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/software-agents.html) | (same chapter, SaaS variant) |
| Configure and Manage Connectors | [On-Prem](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/configure-and-manage-connectors-for-secure-workload.html) | (same chapter, SaaS variant) |

> If your cluster is on a different release (3.x or earlier 4.x)
> the chapter URLs change predictably — replace `4_0` with `3_10`
> (etc.) in the path. Version-specific behaviour for any
> ambiguous detail in this repo should be confirmed against your
> running release's User Guide.

---

## Sections of the Manage Policy Lifecycle chapter mapped to this repo

The chapter is large (~250 KB / 6,000+ rendered lines). The most
useful map is which section of this repo corresponds to which
Cisco section.

| Cisco section | This repo |
|---|---|
| Segmentation Policy Basics | [`docs/02-segmentation-basics.md`](./02-segmentation-basics.md) |
| Use Workspaces to Manage Policies | [`docs/03-workspaces.md`](./03-workspaces.md) |
| About Policies — Attributes / Rank / Inheritance / Consumer & Provider | [`docs/04-policy-attributes.md`](./04-policy-attributes.md) |
| Create and Discover Policies — Best Practices | [`discovery/01-prepare-scope.md`](../discovery/01-prepare-scope.md) |
| Manually Create Policies | [`docs/05-decision-matrix.md`](./05-decision-matrix.md) |
| Discover Policies Automatically (ADM) | [`discovery/03-run-adm.md`](../discovery/03-run-adm.md) |
| Policies for Specific Purposes | [`analysis/07-policy-templates.md`](../analysis/07-policy-templates.md) |
| Policy Templates | [`analysis/07-policy-templates.md`](../analysis/07-policy-templates.md) |
| Grouping Workloads — Clusters and Inventory Filters | [`discovery/04-clusters-and-inventory-filters.md`](../discovery/04-clusters-and-inventory-filters.md) |
| Address Policy Complexities — Priorities | [`analysis/06-policy-complexities.md`](../analysis/06-policy-complexities.md) |
| Address Policy Complexities — Cross-scope | [`analysis/06-policy-complexities.md`](../analysis/06-policy-complexities.md) |
| Effective Consumer or Effective Provider | [`analysis/06-policy-complexities.md`](../analysis/06-policy-complexities.md) |
| About Deleting Policies | [`enforcement/07-modify-enforced-policies.md`](../enforcement/07-modify-enforced-policies.md) |
| Review and Analyze Policies — Review Discovered | [`analysis/01-review-discovered-policies.md`](../analysis/01-review-discovered-policies.md) |
| Policy Visual Representation | [`analysis/02-policy-visual.md`](../analysis/02-policy-visual.md) |
| Quick Analysis | [`analysis/03-quick-analysis.md`](../analysis/03-quick-analysis.md) |
| Live Policy Analysis | [`analysis/04-live-analysis.md`](../analysis/04-live-analysis.md) |
| Enforce Policies — Check Agent Health | [`enforcement/02-agent-readiness.md`](../enforcement/02-agent-readiness.md) |
| Enable Policy Enforcement + Wizard | [`enforcement/03-enable-enforcement.md`](../enforcement/03-enable-enforcement.md) |
| Enforcement on Containers | [`enforcement/05-platform-specific.md`](../enforcement/05-platform-specific.md) |
| Verify Enforcement Works as Expected | [`enforcement/06-verify-enforcement.md`](../enforcement/06-verify-enforcement.md) |
| Modify Enforced Policies / Enforce New and Revised | [`enforcement/07-modify-enforced-policies.md`](../enforcement/07-modify-enforced-policies.md) |
| Revert / Disable / Pause | [`enforcement/09-rollback-and-revert.md`](../enforcement/09-rollback-and-revert.md), [`enforcement/10-pause-and-emergency-disable.md`](../enforcement/10-pause-and-emergency-disable.md) |
| About Policy Versions (v\* and p\*), Policy Diff, Activity Logs | [`enforcement/08-policy-versions.md`](../enforcement/08-policy-versions.md), [`operations/02-version-history.md`](../operations/02-version-history.md) |
| Conversations | [`analysis/05-conversations.md`](../analysis/05-conversations.md) |
| Automated Load Balancer Config (F5 ADM) | [`discovery/07-f5-adm.md`](../discovery/07-f5-adm.md) |
| Policies Publisher (Kafka) | [`api/04-policies-publisher-kafka.md`](../api/04-policies-publisher-kafka.md) |
| Import / Export | [`analysis/08-import-export.md`](../analysis/08-import-export.md) |

---

## Software Agents — install pages (companion repo references)

These are referenced when the policy lifecycle interacts with
the agent layer (readiness checks, enforcement modes, host-firewall
behaviour). Full deployment runbooks live in
[`chandrapati/CSW-Agent-Installation-Guide`](https://github.com/chandrapati/CSW-Agent-Installation-Guide);
the Cisco canonicals:

- [Software Agents — chapter root](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/software-agents.html)
- [Deploy Software Agents on a Linux Host](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/software-agents.html#concept_deploy-software-agents-on-a-linux-host)
- [Deploy Software Agents on a Windows Host](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/software-agents.html#concept_deploy-software-agents-on-a-windows-host)
- [Deploy Kubernetes / OpenShift Software Agents](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/software-agents.html#concept_deploy-software-agents-kubernetes-openshift)
- [Cisco Secure Workload Compatibility Matrix](https://www.cisco.com/c/m/en_us/products/security/secure-workload-compatibility-matrix.html)
  *— the only authoritative source for OS / kernel support*

---

## OpenAPIs (used by `api/`)

The full OpenAPI reference is large (~366 KB / 18,000+ rendered
lines). The key sub-sections this repo touches:

| OpenAPI sub-section | This repo |
|---|---|
| Authentication (API key + secret) | [`api/01-authentication.md`](../api/01-authentication.md) |
| Application Workspaces / Policies | [`api/02-openapi-policies.md`](../api/02-openapi-policies.md) |
| Enable / Disable Enforcement on a Workspace | [`api/03-enforcement-toggle-api.md`](../api/03-enforcement-toggle-api.md) |
| Inventory & Scopes | (cross-referenced from [`docs/01-prerequisites.md`](./01-prerequisites.md)) |

Full OpenAPI page:
[Secure Workload OpenAPIs (4.0 On-Prem)](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/secure-workload-openapis.html).

---

## Release notes and downloads

- [Secure Workload Release Notes](https://www.cisco.com/c/en/us/support/security/tetration/products-release-notes-list.html)
- [Secure Workload Software Downloads](https://software.cisco.com/download/home/286309796) *(requires CCO login)*

---

## Where to file bugs / questions / TAC

- For questions about your specific deployment — release-version
  specifics, customer-environment trade-offs, sizing, licensing —
  reach your Cisco Secure Workload account team (your assigned
  Cisco SE or partner SE).
- [Cisco Secure Workload product home page](https://www.cisco.com/c/en/us/products/security/secure-workload/index.html)
  has the *Contact Cisco* / *Get a demo* / *Find a partner* paths.
- [Cisco general contact page](https://www.cisco.com/c/en/us/about/contact-cisco.html).
- [Open a Cisco TAC case](https://www.cisco.com/c/en/us/support/index.html)
  for incidents on a deployed cluster.

---

## See also

- This repo's high-level overview: [`../README.md`](../README.md)
- Quick index: [`../INDEX.md`](../INDEX.md)
- Companion repo for agent install:
  [`chandrapati/CSW-Agent-Installation-Guide`](https://github.com/chandrapati/CSW-Agent-Installation-Guide)
- Companion repo for compliance framework mappings:
  [`chandrapati/CSW-Compliance-Mapping`](https://github.com/chandrapati/CSW-Compliance-Mapping)
