# Quick Index — by Phase, by Concept, by Question

> See [`README.md`](./README.md) for the high-level overview and
> the policy-lifecycle diagram. This index is for jumping
> directly to the page that answers a specific question.

---

## By lifecycle phase

| Phase | Folder | Start here |
|---|---|---|
| 0. Background — concepts and prerequisites | [`docs/`](./docs/) | [`docs/00-official-references.md`](./docs/00-official-references.md) |
| 1. Discovery — Application Dependency Mapping | [`discovery/`](./discovery/README.md) | [`discovery/01-prepare-scope.md`](./discovery/01-prepare-scope.md) |
| 2. Analysis — review, simulate, resolve | [`analysis/`](./analysis/README.md) | [`analysis/01-review-discovered-policies.md`](./analysis/01-review-discovered-policies.md) |
| 3. Enforcement — Monitor → Simulate → Enforce | [`enforcement/`](./enforcement/README.md) | [`enforcement/01-pre-enforcement-checklist.md`](./enforcement/01-pre-enforcement-checklist.md) |
| Day-2 operations | [`operations/`](./operations/README.md) | [`operations/03-troubleshooting-blocked-flows.md`](./operations/03-troubleshooting-blocked-flows.md) |
| Programmatic / GitOps | [`api/`](./api/README.md) | [`api/01-authentication.md`](./api/01-authentication.md) |

---

## By concept

| If you're asking about… | Go to |
|---|---|
| **Workspaces** — the unit of policy management | [`docs/03-workspaces.md`](./docs/03-workspaces.md) |
| **Scopes** — the inventory tree | [`docs/01-prerequisites.md`](./docs/01-prerequisites.md) and [Manage Inventory (Cisco)](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-inventory-for-secure-workload.html) |
| **Policy rank** — Absolute, Default, Catch-All | [`docs/04-policy-attributes.md`](./docs/04-policy-attributes.md) |
| **Policy inheritance** through the scope tree | [`docs/04-policy-attributes.md`](./docs/04-policy-attributes.md) |
| **Consumer and provider** semantics | [`docs/04-policy-attributes.md`](./docs/04-policy-attributes.md) |
| **ADM** — Application Dependency Mapping | [`discovery/03-run-adm.md`](./discovery/03-run-adm.md) |
| **Clusters** vs **Inventory Filters** | [`discovery/04-clusters-and-inventory-filters.md`](./discovery/04-clusters-and-inventory-filters.md) |
| **Conversations** | [`analysis/05-conversations.md`](./analysis/05-conversations.md) |
| **Quick Analysis** vs **Live Analysis** | [`analysis/03-quick-analysis.md`](./analysis/03-quick-analysis.md) and [`analysis/04-live-analysis.md`](./analysis/04-live-analysis.md) |
| **Effective Consumer / Effective Provider** | [`analysis/06-policy-complexities.md`](./analysis/06-policy-complexities.md) |
| **Cross-scope policy** options | [`analysis/06-policy-complexities.md`](./analysis/06-policy-complexities.md) |
| **Policy versions** v\* and p\* | [`enforcement/08-policy-versions.md`](./enforcement/08-policy-versions.md) |
| **Policy Diff** | [`enforcement/08-policy-versions.md`](./enforcement/08-policy-versions.md) |
| **Enforcement Wizard** | [`enforcement/03-enable-enforcement.md`](./enforcement/03-enable-enforcement.md) |
| **Pause Policy Updates** | [`enforcement/10-pause-and-emergency-disable.md`](./enforcement/10-pause-and-emergency-disable.md) |
| **Policies Publisher (Kafka)** | [`api/04-policies-publisher-kafka.md`](./api/04-policies-publisher-kafka.md) |
| **F5 ADM** — load-balancer-aware policy discovery | [`discovery/07-f5-adm.md`](./discovery/07-f5-adm.md) |

---

## By question

| If you're asking… | Start here |
|---|---|
| *"Where is the official Cisco documentation?"* | [`docs/00-official-references.md`](./docs/00-official-references.md) — links the 4.0 On-Prem and SaaS User Guides plus the Manage Policy Lifecycle, Manage Inventory, OpenAPIs, and Compatibility Matrix pages |
| *"My agents are installed; what do I do next to get to a policy?"* | [`docs/01-prerequisites.md`](./docs/01-prerequisites.md) → [`discovery/01-prepare-scope.md`](./discovery/01-prepare-scope.md) |
| *"How do I structure my scope tree?"* | [`discovery/01-prepare-scope.md`](./discovery/01-prepare-scope.md) |
| *"How long should I collect flow data before running ADM?"* | [`discovery/02-flow-collection-window.md`](./discovery/02-flow-collection-window.md) |
| *"How do I run ADM and what knobs matter?"* | [`discovery/03-run-adm.md`](./discovery/03-run-adm.md) |
| *"ADM produced too many clusters / one giant cluster — now what?"* | [`discovery/08-discovery-anti-patterns.md`](./discovery/08-discovery-anti-patterns.md) |
| *"How do I tell whether my discovered policy is safe to publish?"* | [`analysis/03-quick-analysis.md`](./analysis/03-quick-analysis.md) and [`analysis/04-live-analysis.md`](./analysis/04-live-analysis.md) |
| *"Two workspaces overlap on the same workload — what wins?"* | [`analysis/06-policy-complexities.md`](./analysis/06-policy-complexities.md) |
| *"My consumer and provider are in different scopes."* | [`analysis/06-policy-complexities.md`](./analysis/06-policy-complexities.md) |
| *"How do I check that an agent is ready to enforce?"* | [`enforcement/02-agent-readiness.md`](./enforcement/02-agent-readiness.md) |
| *"How do I roll out enforcement without breaking production?"* | [`enforcement/04-rollout-pattern.md`](./enforcement/04-rollout-pattern.md) |
| *"Enforcement is on but a flow is blocked that shouldn't be."* | [`operations/03-troubleshooting-blocked-flows.md`](./operations/03-troubleshooting-blocked-flows.md) |
| *"How do I revert to an earlier policy version?"* | [`enforcement/09-rollback-and-revert.md`](./enforcement/09-rollback-and-revert.md) |
| *"Production incident — turn enforcement off NOW."* | [`enforcement/10-pause-and-emergency-disable.md`](./enforcement/10-pause-and-emergency-disable.md) |
| *"How do I manage policy as code (GitOps)?"* | [`api/05-gitops-pattern.md`](./api/05-gitops-pattern.md) |
| *"How do I subscribe to policy events from CSW (Kafka)?"* | [`api/04-policies-publisher-kafka.md`](./api/04-policies-publisher-kafka.md) |
| *"How do I capture evidence for a compliance audit?"* | [`operations/04-evidence-and-audit.md`](./operations/04-evidence-and-audit.md) |
| *"How does this guide relate to CSW-Compliance-Mapping?"* | [`operations/06-compliance-companion.md`](./operations/06-compliance-companion.md) |

---

## Disclaimer

Everything in this index points to draft v1 documentation. The
authoritative source for any specific CSW release remains the
*Cisco Secure Workload User Guide* and your release notes; always
cross-check before relying on this repository for a customer
engagement. See [`README.md`](./README.md) for the full
disclaimer.
