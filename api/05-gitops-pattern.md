# 05 — GitOps Pattern for CSW Policy

The end-to-end pattern for managing CSW policy as code, with
git as the source of truth and CSW as the runtime.

> **Cisco source.** [Secure Workload OpenAPIs](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/secure-workload-openapis.html)
> + [Manage Policy Lifecycle — Import / Export](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/manage-policy-lifecycle-in-secure-workload.html#import-export).

---

## What "GitOps for CSW" looks like

```
                            ┌──────────────────────┐
                            │  Application teams   │
                            │  open PRs against    │
                            │  policies/<app>.json │
                            └──────────┬───────────┘
                                       │
                                       ▼
                            ┌──────────────────────┐
                            │  Reviewers approve   │
                            │  PR merge            │
                            └──────────┬───────────┘
                                       │
                                       ▼
   ┌───────────────────────┐    ┌──────────────────────┐
   │  policies/             │    │  CI runs:            │
   │  ├── payments-api.json │    │  1. lint + validate   │
   │  ├── crm-web.json      │───►│  2. import as v_next  │
   │  ├── shared-dns.json   │    │  3. Quick Analysis    │
   │  └── inventories/      │    │  4. comment on PR     │
   └───────────────────────┘    └──────────┬───────────┘
                                            │
                                            ▼ on merge to main
                                ┌──────────────────────┐
                                │  Reconciler runs:    │
                                │  1. apply to CSW     │
                                │  2. publish v→p      │
                                │  3. enable_enforce   │
                                │     SIMULATE first   │
                                │  4. wait window      │
                                │  5. enable_enforce   │
                                │     ENFORCE          │
                                └──────────────────────┘
```

The repo holds policy in a structured format (JSON ergonomic
for humans + machines). Application teams propose changes via
PRs. CI validates and *previews* against CSW. Merge to main
triggers reconciliation through the Monitor → Simulate →
Enforce flow.

---

## Repo layout

```
csw-policy/
├── README.md
├── inventories/
│   ├── prod-shared-dns.json
│   ├── prod-shared-ad.json
│   └── ...
├── workspaces/
│   ├── payments-api.json
│   ├── crm-web.json
│   └── shared-services.json
├── tools/
│   ├── lint.py
│   ├── reconcile.py
│   └── disable.py
└── .github/workflows/
    ├── pr-preview.yml
    └── reconcile.yml
```

Conventions:

- One JSON file per workspace; one per shared inventory filter.
- Filenames mirror workspace names verbatim. Resist drift.
- `tools/` contains the small set of scripts CI uses; everything
  is reproducible from a clone.
- A single `reconcile.py` is the only thing that talks to the
  cluster's mutating API in production.

---

## Workspace JSON shape

The shape mirrors the OpenAPI's import/export format closely so
that round-tripping is lossless. Illustrative skeleton:

```jsonc
{
  "workspace": {
    "name": "payments-api",
    "scope": "Production/BU-Retail/payments-api",
    "primary": true,
    "description": "Owner: platform-eng. Mapped to compliance: PCI-DSS, HIPAA."
  },
  "inventory_filters": [
    { "ref": "prod-payments-api-web" },
    { "ref": "prod-payments-api-app" },
    { "ref": "prod-payments-api-db" }
  ],
  "policies": [
    {
      "consumer":  "prod-payments-api-web",
      "provider":  "prod-payments-api-app",
      "service":   "tcp/8443",
      "action":    "ALLOW",
      "rank":      "DEFAULT",
      "description": "web → app — owner=platform-eng, ticket=CR-1234"
    },
    {
      "consumer":  "any",
      "provider":  "prod-payments-api-app",
      "service":   "any",
      "action":    "DENY",
      "rank":      "CATCHALL",
      "description": "Workspace catch-all"
    }
  ]
}
```

The reconciler maps these higher-level shapes to the raw
OpenAPI fields, keeps the source-of-truth in git readable, and
hides cluster-side IDs from the policy authors.

---

## CI on PR — preview without enforcing

Pseudo-workflow:

```yaml
# .github/workflows/pr-preview.yml
name: csw-pr-preview
on: pull_request
jobs:
  preview:
    steps:
      - uses: actions/checkout@v4
      - run: ./tools/lint.py policies/
      - name: Apply to CSW as a secondary workspace
        env:
          CSW_API_KEY:    ${{ secrets.CSW_API_KEY_PREVIEW }}
          CSW_API_SECRET: ${{ secrets.CSW_API_SECRET_PREVIEW }}
        run: ./tools/reconcile.py --mode=preview --workspace=secondary
      - name: Run Quick Analysis
        run: ./tools/reconcile.py --mode=analyze
      - name: Comment on PR
        run: ./tools/comment-on-pr.py
```

The PR preview imports the proposed policy into a **secondary**
workspace named after the PR (e.g. `payments-api-pr-1234`),
runs Quick Analysis, and writes the result back as a PR comment
showing the diff vs. the current p\* and any would-be-blocked
flows.

> **Important.** The preview key has *only* the capability to
> create / modify secondary workspaces — it cannot publish or
> enforce. The blast radius of leaking it is bounded.

---

## CI on merge — reconcile through the rollout pattern

```yaml
# .github/workflows/reconcile.yml
name: csw-reconcile
on:
  push:
    branches: [main]
jobs:
  reconcile:
    steps:
      - uses: actions/checkout@v4
      - run: ./tools/reconcile.py --mode=apply
      - run: ./tools/reconcile.py --mode=publish
      - run: ./tools/reconcile.py --mode=enable-simulate
      - name: Wait for Simulate window
        run: sleep 86400  # 24h; tune per app
      - run: ./tools/reconcile.py --mode=enable-enforce
```

In practice:

- **Per-workspace cadence** (some workspaces want a longer
  Simulate window than others) — drive from per-workspace
  metadata in the JSON, not a global wait.
- **Quick Analysis gate** before publish is non-negotiable.
- **Live Analysis gate** before `enable-enforce` is ideal but
  hard to fully automate; common compromise is "Simulate window
  with rejected-flow telemetry assertion built into the
  reconciler."
- **Manual approval** before `enable-enforce` for high-risk
  workspaces — gate the workflow on a CMDB / ITSM ticket state.

---

## What lives where

| Concern | In git | In CSW |
|---|---|---|
| **Intent** of policy (what the rules should be) | Yes | No |
| **Cluster-side IDs** (workspace ID, policy ID) | Cached / regenerated | Source of truth |
| **Inventory** (workloads, labels, scope membership) | No (lives in CSW / inventory source) | Yes |
| **Flow telemetry** | No | Yes |
| **Activity log / audit trail** | Synthesized from PR history | Source of truth |
| **Published versions** (p\* history) | Tagged in git per p\* | Source of truth |

Git is great at *intent*; CSW is great at *runtime*. Don't
mix them.

---

## Drift detection

The reconciler periodically (e.g. nightly) reads the cluster's
current state and **diffs it against git**. Drift sources:

| Drift | Likely cause | Action |
|---|---|---|
| Cluster has rules not in git | Someone edited via UI | Reconciler comments on the workspace, opens a PR to either accept (and codify) or revert |
| Git has rules not in cluster | Reconciler failed to apply | Re-run reconciler; alert if persistent |
| Different rule contents | Manual edit to a rule | Same as above; figure out which is right |
| Workspace doesn't exist in cluster | Renamed / deleted via UI | Hard problem — out-of-band action defeats GitOps; alert and manually reconcile |

Surface drift to the team daily. The whole value proposition
of GitOps is "git is the truth" — drift erodes that promise.

---

## Disaster recovery

The repo plus the reconciler is your DR plan. From scratch:

1. Restore git repo (it's already in your normal SCM backups).
2. Provision a new CSW tenant.
3. Run `reconcile.py --bootstrap` to recreate inventory filters,
   workspaces, and policies.
4. Replay the Monitor → Simulate → Enforce flow per workspace.

Test this once. The first time can't be in a real DR.

---

## See also

- [`01-authentication.md`](./01-authentication.md)
- [`02-openapi-policies.md`](./02-openapi-policies.md)
- [`03-enforcement-toggle-api.md`](./03-enforcement-toggle-api.md)
- [`../analysis/08-import-export.md`](../analysis/08-import-export.md)
- [`../enforcement/04-rollout-pattern.md`](../enforcement/04-rollout-pattern.md)
- [`../operations/01-policy-drift.md`](../operations/01-policy-drift.md)
- [`../operations/04-evidence-and-audit.md`](../operations/04-evidence-and-audit.md)
