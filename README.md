# Cisco Secure Workload — Policy Lifecycle Guide

![Visitors](https://visitor-badge.laobi.icu/badge?page_id=chandrapati.CSW-Policy-Lifecycle&left_text=visitors)

A practitioner-oriented reference for the **discovery, analysis,
and enforcement** of segmentation policy on Cisco Secure Workload
(CSW) — the lifecycle that takes a customer from *"agents are
installed and reporting flows"* to *"least-privilege
micro-segmentation policy is enforced at every workload, with
evidence and rollback paths."*

Written for security engineers, platform owners, and POV teams
who already have CSW agents deployed (see the companion
[CSW-Agent-Installation-Guide](https://github.com/chandrapati/CSW-Agent-Installation-Guide))
and now need to author, validate, and roll out policy without
breaking production.

> **Status.** Draft v1. Patterns and command shapes reflect
> typical operating practice on CSW 4.0. The authoritative source
> for any specific release remains the *Cisco Secure Workload User
> Guide* — see [`docs/00-official-references.md`](./docs/00-official-references.md)
> — and your release notes; always cross-check version-specific
> details there before relying on this repository in a customer
> engagement.

> **Official Cisco Secure Workload documentation.** Every page
> in this repo cross-references the canonical Cisco source. The
> hub:
> [`docs/00-official-references.md`](./docs/00-official-references.md).
> Master chapter:
> [Manage Policy Lifecycle in Secure Workload (4.0 On-Prem)](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html)
> ·
> [SaaS 4.0](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-saas-v40.html)
> ·
> [Compatibility Matrix](https://www.cisco.com/c/m/en_us/products/security/secure-workload-compatibility-matrix.html).
> When this guide and the User Guide disagree, **the User Guide
> wins**.

---

## For executives — at a glance

> **CISO / CIO read.** This repo is the *operational* half of a CSW
> segmentation program; the business case and audit evidence live in
> the companion [CSW-Compliance-Mapping](https://github.com/chandrapati/CSW-Compliance-Mapping)
> and [CSW-Tenant-Insights](https://github.com/chandrapati/CSW-Tenant-Insights) repos.

- **Problem it solves.** Flat internal networks let one compromised
  workload reach hundreds more. This guide is the disciplined path to
  **least-privilege, east-west micro-segmentation** — limiting each
  workload to only the traffic it actually needs.
- **How risk is controlled.** Policy is never flipped straight to
  "block." The method is **Monitor → Simulate → Enforce**: observe real
  traffic, dry-run the rules against live flows to catch breakage *before*
  it happens, then enforce — with a one-click rollback path the whole way.
- **Time-to-value.** A typical scope moves from agents-reporting to
  enforced policy in roughly **30 days** (≈2 weeks flow baselining, then
  staged simulate/enforce). See [`enforcement/04-rollout-pattern.md`](./enforcement/04-rollout-pattern.md).
- **Who owns what.** Security defines intent; platform/app owners
  validate flows; change control gates each enforce step. Emergency
  disable is a documented, per-workspace action — see
  [`enforcement/10-pause-and-emergency-disable.md`](./enforcement/10-pause-and-emergency-disable.md).
- **What good looks like.** % of in-scope apps in Enforce, a falling
  trend of unexpected ("escaped") flows, low policy drift, and
  audit-ready evidence exports between assessments.
- **Status / trust.** Draft v1, CSW 4.0-oriented; the *Cisco Secure
  Workload User Guide* is authoritative for any specific release.

---

## The CSW policy lifecycle in one diagram

```
        agents installed in Visibility (this guide assumes this)
                              │
                              ▼
   ┌────────────────────────────────────────────────────────────┐
   │ 1. DISCOVERY — discovery/                                  │
   │    scope tree + inventory labels → ADM run on flow data    │
   │    → discovered (v*) policy                                │
   └────────────────────────────────────────────────────────────┘
                              │
                              ▼
   ┌────────────────────────────────────────────────────────────┐
   │ 2. ANALYSIS — analysis/                                    │
   │    review · visual map · Quick Analysis (one hypothetical  │
   │    flow) · Policy Experiments (replay past traffic) ·      │
   │    Live Analysis (current flows) · resolve conflicts       │
   │    → analyzed / published (p*) policy                      │
   └────────────────────────────────────────────────────────────┘
                              │
                              ▼
   ┌────────────────────────────────────────────────────────────┐
   │ 3. ENFORCEMENT — enforcement/                              │
   │    Monitor → Simulate → Enforce, per-workspace,            │
   │    with verification, rollback, and emergency-disable      │
   │    paths defined up front                                  │
   └────────────────────────────────────────────────────────────┘
                              │
                              ▼
   ┌────────────────────────────────────────────────────────────┐
   │ DAY 2 — operations/                                        │
   │    drift detection, version history, blocked-flow triage,  │
   │    evidence collection, handover                           │
   └────────────────────────────────────────────────────────────┘
```

The bedrock vocabulary — **scopes**, **workspaces**, **policies**
(with rank Absolute / Default / Catch-All), **clusters**,
**inventory filters**, **conversations**, **v\*** (discovery
version) and **p\*** (published version) — is explained in
[`docs/`](./docs/). Read those first if any term above is
unfamiliar.

---

## What's in this repo

- [`README.md`](./README.md) — you are here (overview + lifecycle map)
- [`INDEX.md`](./INDEX.md) — jump table by phase / by question
- [`docs/`](./docs/) — Background concepts (read first)
  - [`00-official-references.md`](./docs/00-official-references.md) — Cisco doc cross-reference (read this first)
  - [`01-prerequisites.md`](./docs/01-prerequisites.md) — scopes, labelling, agents in Visibility, flow-data window
  - [`02-segmentation-basics.md`](./docs/02-segmentation-basics.md) — what a segmentation policy actually is in CSW
  - [`03-workspaces.md`](./docs/03-workspaces.md) — workspaces are the unit of policy management
  - [`04-policy-attributes.md`](./docs/04-policy-attributes.md) — attributes, rank, inheritance, consumer / provider
  - [`05-decision-matrix.md`](./docs/05-decision-matrix.md) — discover automatically vs. author manually
- [`discovery/`](./discovery/) — Phase 1: Application Dependency Mapping (ADM)
  - [`README.md`](./discovery/README.md)
  - [`01-prepare-scope.md`](./discovery/01-prepare-scope.md) — scope tree shape, labelling, inventory filters
  - [`02-flow-collection-window.md`](./discovery/02-flow-collection-window.md) — how long to collect, what to look for
  - [`03-run-adm.md`](./discovery/03-run-adm.md) — run ADM, refine, iterate
  - [`04-clusters-and-inventory-filters.md`](./discovery/04-clusters-and-inventory-filters.md) — grouping workloads
  - [`05-flow-filters.md`](./discovery/05-flow-filters.md) — include / exclude filters
  - [`06-external-dependencies.md`](./discovery/06-external-dependencies.md) — handle external dependencies
  - [`07-f5-adm.md`](./discovery/07-f5-adm.md) — F5 load-balancer-aware ADM
  - [`08-discovery-anti-patterns.md`](./discovery/08-discovery-anti-patterns.md) — common mistakes to avoid
- [`analysis/`](./analysis/) — Phase 2: Review and Analyze Policies
  - [`README.md`](./analysis/README.md)
  - [`01-review-discovered-policies.md`](./analysis/01-review-discovered-policies.md) — interpret what ADM produced
  - [`02-policy-visual.md`](./analysis/02-policy-visual.md) — Policy Visual Representation
  - [`03-quick-analysis.md`](./analysis/03-quick-analysis.md) — Quick Analysis (single hypothetical flow) + Policy Experiments (past traffic)
  - [`04-live-analysis.md`](./analysis/04-live-analysis.md) — Live Policy Analysis (current flows)
  - [`05-conversations.md`](./analysis/05-conversations.md) — Conversations table + observations
  - [`06-policy-complexities.md`](./analysis/06-policy-complexities.md) — priorities, cross-scope, effective consumer / provider
  - [`07-policy-templates.md`](./analysis/07-policy-templates.md) — policy templates
  - [`08-import-export.md`](./analysis/08-import-export.md) — Import / Export of policies
- [`enforcement/`](./enforcement/) — Phase 3: Enforce Policies
  - [`README.md`](./enforcement/README.md)
  - [`01-pre-enforcement-checklist.md`](./enforcement/01-pre-enforcement-checklist.md)
  - [`02-agent-readiness.md`](./enforcement/02-agent-readiness.md) — Check Agent Health and Readiness to Enforce
  - [`03-enable-enforcement.md`](./enforcement/03-enable-enforcement.md) — Enable Policy Enforcement + the Enforcement Wizard
  - [`04-rollout-pattern.md`](./enforcement/04-rollout-pattern.md) — Monitor → Simulate → Enforce phased rollout
  - [`05-platform-specific.md`](./enforcement/05-platform-specific.md) — Windows WFP · Linux iptables / nftables · containers
  - [`06-verify-enforcement.md`](./enforcement/06-verify-enforcement.md) — Verify Enforcement Works as Expected
  - [`07-modify-enforced-policies.md`](./enforcement/07-modify-enforced-policies.md) — enforce new and revised policies
  - [`08-policy-versions.md`](./enforcement/08-policy-versions.md) — v* and p* versioning + Policy Diff
  - [`09-rollback-and-revert.md`](./enforcement/09-rollback-and-revert.md) — revert enforced policies to an earlier version
  - [`10-pause-and-emergency-disable.md`](./enforcement/10-pause-and-emergency-disable.md) — Pause Policy Updates · Disable Policy Enforcement
  - [`11-monitoring-after-enforcement.md`](./enforcement/11-monitoring-after-enforcement.md) — what to watch in the first 30 days
- [`api/`](./api/) — Programmatic management
  - [`README.md`](./api/README.md)
  - [`01-authentication.md`](./api/01-authentication.md) — API key + secret, OpenAPI auth
  - [`02-openapi-policies.md`](./api/02-openapi-policies.md) — create / update / list policies via OpenAPI
  - [`03-enforcement-toggle-api.md`](./api/03-enforcement-toggle-api.md) — enable / disable enforcement on a workspace via API
  - [`04-policies-publisher-kafka.md`](./api/04-policies-publisher-kafka.md) — Policies Publisher (Kafka) for downstream consumers
  - [`05-gitops-pattern.md`](./api/05-gitops-pattern.md) — managing policy as code
- [`operations/`](./operations/) — Day-2 operations specific to policy
  - [`README.md`](./operations/README.md)
  - [`01-policy-drift.md`](./operations/01-policy-drift.md)
  - [`02-version-history.md`](./operations/02-version-history.md) — Activity Logs and Version History
  - [`03-troubleshooting-blocked-flows.md`](./operations/03-troubleshooting-blocked-flows.md)
  - [`04-evidence-and-audit.md`](./operations/04-evidence-and-audit.md) — evidence buckets per policy
  - [`05-handover-runbook.md`](./operations/05-handover-runbook.md)
  - [`06-compliance-companion.md`](./operations/06-compliance-companion.md) — pairing with CSW-Compliance-Mapping

---

## How to use this guide

0. **Read [`docs/00-official-references.md`](./docs/00-official-references.md) first.**
   It cross-references the CSW 4.0 User Guides and pins the
   canonical chapters this repo is a companion to:
   *Manage Policy Lifecycle*, *Manage Inventory*, *Get Started*,
   and *Secure Workload OpenAPIs*.
1. **Read [`docs/01-prerequisites.md`](./docs/01-prerequisites.md) next.**
   Almost every failed CSW policy rollout traces back to one of:
   a half-built scope tree, missing inventory labels, agents
   still in `--visibility` so flow data is incomplete, or fewer
   than the recommended ~30 days of flow data on the workloads
   you intend to author policy against.
2. **Get fluent with the policy model in
   [`docs/02-segmentation-basics.md`](./docs/02-segmentation-basics.md),
   [`docs/03-workspaces.md`](./docs/03-workspaces.md), and
   [`docs/04-policy-attributes.md`](./docs/04-policy-attributes.md).**
   Workspaces, scope tree, policy rank (Absolute / Default /
   Catch-All), inheritance, and consumer / provider are the
   five concepts every other page in this repo assumes. For the
   common *"can CSW segment on URLs / FQDNs?"* question, see
   [`docs/06-fqdn-and-domain-policy.md`](./docs/06-fqdn-and-domain-policy.md)
   (FQDN/domain policy yes since 3.9; URL/path no).
3. **Pick a path from
   [`docs/05-decision-matrix.md`](./docs/05-decision-matrix.md).**
   Most apps go through automatic policy discovery (ADM); a few
   (HA pairs, multicast-heavy apps, very small surfaces) are
   easier to author manually. The matrix walks the choice.
4. **Run discovery →
   [`discovery/`](./discovery/README.md).** Prepare the scope,
   collect flow data, run ADM, refine the result, repeat.
5. **Analyze before enforcing →
   [`analysis/`](./analysis/README.md).** Review what ADM
   produced, use Quick Analysis to debug specific hypothetical
   flows, use Policy Experiments to replay past traffic against
   the proposed policy, run Live Policy Analysis against current
   flows, resolve conflicts and cross-scope edges, then take the
   analyzed version through the Enable Policy Enforcement
   wizard.
6. **Roll out enforcement →
   [`enforcement/`](./enforcement/README.md).** Monitor →
   Simulate → Enforce, in waves, with rollback paths defined
   up front. Going straight to Enforce on day one is the single
   most common cause of "CSW broke production" stories.
7. **Operate it →
   [`operations/`](./operations/README.md).** Drift detection,
   blocked-flow triage, evidence collection for compliance, and
   the handover runbook for steady-state.

For programmatic management (GitOps, Kafka subscribers,
external automation), see [`api/`](./api/README.md).

---

## Companion repositories

This repo is the **policy** half of the CSW practitioner toolkit.
The other halves:

- [`chandrapati/CSW-Agent-Installation-Guide`](https://github.com/chandrapati/CSW-Agent-Installation-Guide)
  — get agents deployed and reporting flow telemetry. **Read this
  first if you don't yet have agents in place** — without flow
  data there's nothing for ADM to discover.
- [`chandrapati/CSW-Compliance-Mapping`](https://github.com/chandrapati/CSW-Compliance-Mapping)
  — sixteen compliance, sector, and zero-trust frameworks mapped
  to CSW capability with runbooks and customer reports. The
  policy artefacts produced here become the evidence those
  mappings rely on. See
  [`operations/06-compliance-companion.md`](./operations/06-compliance-companion.md).

---

## Disclaimer

The patterns, command examples, sample API calls, and operational
guidance in this repository are provided for **informational and
reference purposes only**. They are not a substitute for the
official Cisco Secure Workload product documentation, your
organisation's change-management process, or qualified consulting
engagement.

> **Official Cisco Secure Workload documentation.** The full set
> of canonical pointers — User Guides (4.0
> [On-Premises](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40.html)
> and [SaaS](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-saas-v40.html)),
> [Compatibility Matrix](https://www.cisco.com/c/m/en_us/products/security/secure-workload-compatibility-matrix.html),
> [Manage Policy Lifecycle](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html)
> chapter, [Manage Inventory](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-inventory-for-secure-workload.html)
> chapter, [OpenAPIs](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/secure-workload-openapis.html),
> release notes — is consolidated in
> [`docs/00-official-references.md`](./docs/00-official-references.md).
> The
> [CSW 4.0 documentation landing page](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/landing-page/secureworkload-40-docs.html)
> is the navigation root if you need to drill into a section
> that isn't called out in this repo. **When this guide and the
> User Guide disagree, the User Guide wins.**

Specifically:

- CSW UI navigation paths and exact menu labels reflect typical
  practice at the time of authoring. Cluster-side labels do
  shift between releases; the
  [*Manage Policy Lifecycle* chapter](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html)
  is the source of truth for your specific release.
- Policy authoring and enforcement is **release-version-agnostic
  in structure** in this repo, but specific UI flows (the
  Enforcement Wizard, Live Policy Analysis, Conversations) gain
  capabilities over time. Cross-check the User Guide for your
  release.
- Sample OpenAPI calls in [`api/`](./api/README.md) are
  illustrative; tailor authentication, scope IDs, and pagination
  shape to your tenant before running in production.
- Production deployments should always start in **Monitoring**
  mode and progress to Enforcement only after the simulation
  workflow ([`enforcement/04-rollout-pattern.md`](./enforcement/04-rollout-pattern.md))
  has retired the obvious would-be-blocked flows. Going straight
  to Enforce on day one is the single most common cause of
  preventable outages during CSW rollouts.

### Questions, sizing, licensing, or anything else?

For questions about your specific deployment — release-version
specifics, customer-environment trade-offs, sizing, licensing,
Compatibility-Matrix edge cases, or anything that requires
cluster-side workflow review — **reach out to your Cisco Secure
Workload account team** (your assigned Cisco SE or partner SE).
If you don't yet have an account team, the
[Cisco Secure Workload product home page](https://www.cisco.com/c/en/us/products/security/secure-workload/index.html)
has the *Contact Cisco* / *Get a demo* / *Find a partner* paths,
or use [Cisco's general contact page](https://www.cisco.com/c/en/us/about/contact-cisco.html).
For incidents on a deployed cluster,
[open a Cisco TAC case](https://www.cisco.com/c/en/us/support/index.html).

This document should receive subject-matter-expert review before
being used to gate any production change.

---

## Step-by-Step Guides

Hands-on integration and deployment guides — follow these top to bottom to build out a deployment:

| Guide | Description | Best for |
|-------|-------------|---------|
| [📘 Agent Installation](https://github.com/chandrapati/CSW-Agent-Installation-Guide) | Deploy CSW agents on Linux / Windows / cloud | Day-1 sensor deployment |
| [📘 Policy Lifecycle](https://github.com/chandrapati/CSW-Policy-Lifecycle) | Policy discovery → enforcement workflow | Policy management |
| [📘 ISE / pxGrid](https://github.com/chandrapati/csw-ise-integration) | ISE/pxGrid: user-identity–aware microsegmentation | Identity & Zero Trust |
| [📘 AnyConnect NVM](https://github.com/chandrapati/csw-anyconnect-nvm) | Endpoint process flows + user identity via NVM | Endpoint telemetry |
| [📘 ServiceNow CMDB](https://github.com/chandrapati/csw-servicenow-integration) | ServiceNow CMDB label enrichment for workload scopes | CMDB-driven policy |
| [📘 Infoblox](https://github.com/chandrapati/csw-infoblox-integration) | Infoblox IPAM/DNS extensible-attribute label enrichment | IPAM/DNS-driven policy |
| [📘 F5 BIG-IP](https://github.com/chandrapati/csw-f5-integration) | F5 virtual-server labels, policy enforcement, IPFIX flow visibility | Load balancer segmentation |
| [📘 NetScaler ADC](https://github.com/chandrapati/csw-netscaler-integration) | NetScaler LB virtual-server labels + ACL policy enforcement | Load balancer segmentation |
| [📘 AWS Connector](https://github.com/chandrapati/csw-aws-connector) | EC2 tag ingestion + VPC flow logs + Security Group enforcement | AWS workloads |
| [📘 Azure Connector](https://github.com/chandrapati/csw-azure-connector) | Azure VM tag ingestion + VNet flow logs + NSG enforcement | Azure workloads |
| [📘 GCP Connector](https://github.com/chandrapati/csw-gcp-connector) | GCE label ingestion + VPC flow logs + firewall enforcement | GCP workloads |
| [📘 NetFlow](https://github.com/chandrapati/csw-netflow-integration) | NetFlow v9/IPFIX agentless flow ingestion from switches | Network fabric visibility |
| [📘 ERSPAN](https://github.com/chandrapati/csw-erspan-integration) | Agentless packet mirroring for legacy / OT / IoT devices | Deep agentless visibility |
| [📘 Secure Firewall](https://github.com/chandrapati/CSW-Secure-Firewall-Integration-Guide) | NSEL flow ingestion from Cisco Secure Firewall (FTD/ASA) | Firewall flow visibility |
| [📘 Splunk Integration](https://github.com/chandrapati/csw-splunk-integration) | CSW syslog alerts → Splunk SIEM | SecOps / SIEM teams |

## Resources

Learning paths, reference material, and day-2 tooling:

| Resource | Description | Best for |
|----------|-------------|---------|
| [📘 User Education](https://github.com/chandrapati/CSW-User-Education) | Onboarding guides, concept explainers, and curated video library | New CSW users |
| [📘 Compliance Mapping](https://github.com/chandrapati/CSW-Compliance-Mapping) | Map CSW controls to NIST, PCI-DSS, HIPAA, CIS | Compliance & audit |
| [📘 Tenant Insights](https://github.com/chandrapati/CSW-Tenant-Insights) | Tenant-level reporting and analytics | Visibility metrics |
| [📘 Operations Toolkit](https://github.com/chandrapati/CSW-Operations-Toolkit) | Day-2 ops scripts: health checks, reporting, policy analysis | Ongoing operations |

> **Suggested customer journey:**
> User Education → Agent Installation → Policy Lifecycle → ISE/pxGrid → ServiceNow CMDB → Infoblox → F5 BIG-IP → NetScaler ADC → Splunk Integration → Compliance Mapping → Operations Toolkit
