# 02 — Policy CRUD via OpenAPI

The policy lifecycle's core operations — list, create, update,
delete, publish — are all available as OpenAPI calls. This is
the building block for everything in
[`05-gitops-pattern.md`](./05-gitops-pattern.md).

> **Cisco source.** [Secure Workload OpenAPIs — Application Workspaces / Policies](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/secure-workload-openapis.html).

---

## Endpoint families

> **Exact paths, parameters, body schemas, and field names are
> release-specific. Treat the patterns below as a *map* of which
> capabilities exist, not as a contract for paths.** Always
> consult the [OpenAPIs chapter](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/secure-workload-openapis.html)
> for your release before depending on a specific path.

| Family | Path pattern (illustrative — verify) | Use |
|---|---|---|
| Workspaces | `/openapi/v1/applications` (list, create, get, delete) | Manage workspaces themselves |
| Policies on a workspace | `/openapi/v1/applications/{app_id}/policies` (list, create, update, delete) | Policy CRUD |
| Versions | `/openapi/v1/applications/{app_id}/versions` | List discovered (v\*) and analyzed/enforced (p\*) versions |
| Analyze | An operation that snapshots the current policy and increments p\* (analysis context) | UI equivalent: *Analyze Latest Policies* |
| Enforcement | `/openapi/v1/applications/{app_id}/enable_enforce` and `/disable_enforce` (verify) | See [`03-enforcement-toggle-api.md`](./03-enforcement-toggle-api.md) |
| Inventory filters | `/openapi/v1/filters/inventories` | Reusable label expressions |
| Scopes | `/openapi/v1/app_scopes` | Inventory tree |
| Import / Export | `/openapi/v1/policies/import` and `/export` | JSON / CSV round-trip |

---

## A typical request

List workspaces (Python SDK):

```python
client = RestClient(CLUSTER, api_key=KEY, api_secret=SECRET)
resp = client.get("/openapi/v1/applications")
resp.raise_for_status()
for ws in resp.json():
    print(ws["id"], ws["name"], ws["primary"])
```

Get a specific workspace's policies:

```python
app_id = "5f1e..."
resp = client.get(f"/openapi/v1/applications/{app_id}/policies")
policies = resp.json()
for p in policies:
    print(p["id"], p["consumer_filter_id"], p["provider_filter_id"],
          p["action"], p["rank"])
```

Create a new policy:

```python
new_policy = {
    "consumer_filter_id":  "filter-prod-payments-api-web",
    "provider_filter_id":  "filter-prod-payments-api-app",
    "action":              "ALLOW",
    "rank":                "DEFAULT",
    "l4_params": [
        {"proto": 6, "port": [8443, 8443]},
    ],
    "description": "web → app — owner=platform-eng, ticket=CR-1234",
}
resp = client.post(
    f"/openapi/v1/applications/{app_id}/policies",
    json_body=json.dumps(new_policy),
)
resp.raise_for_status()
print("created policy:", resp.json()["id"])
```

The exact JSON shape (field names, enum values for `action`,
`rank`, the protocol numbering for `l4_params`) is documented
in the
[OpenAPIs chapter](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/secure-workload-openapis.html).

---

## Idempotency and concurrency

CSW OpenAPI is not transactional in the database sense.
Multiple writers can produce surprising states. Defensive
patterns:

1. **Single writer per workspace.** If GitOps is the canonical
   author, lock out the UI for that workspace via RBAC.
2. **Use any version / revision field the API exposes for the
   resource you're editing.** The principle is read-modify-write
   with a version check: read the current version; pass it back
   on write; the server rejects mismatches. *However*, CSW's
   OpenAPI doesn't broadly advertise per-policy ETag-style
   optimistic concurrency for individual rule edits in the
   [OpenAPIs chapter](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/secure-workload-openapis.html)
   — confirm what your release exposes and lean harder on
   patterns 1, 3, and 4 where it doesn't.
3. **Always read back** after a write to confirm the resulting
   state.
4. **Wrap writes in idempotency keys** at the script level —
   keep a record of "I already created this rule, no need to
   retry" so retries on transient errors don't double-create.

---

## Pagination

List endpoints are paginated. Don't assume the first response
contains all results:

```python
def list_all_policies(client, app_id):
    out = []
    cursor = None
    while True:
        params = {"limit": 200}
        if cursor:
            params["cursor"] = cursor
        resp = client.get(
            f"/openapi/v1/applications/{app_id}/policies",
            params=params,
        )
        body = resp.json()
        out.extend(body["results"])
        cursor = body.get("next_cursor")
        if not cursor:
            break
    return out
```

(Field names — `cursor` / `next_cursor` / `results` — vary by
endpoint; check the chapter.)

---

## Publishing (v\* → p\*)

Publishing is a separate operation:

```python
resp = client.post(f"/openapi/v1/applications/{app_id}/publish",
                   json_body=json.dumps({
                       "version_id": "v_id_to_publish",
                       "comment":    "Published by GitOps for ticket CR-1234",
                   }))
```

Publishing creates a new p\*. Subsequent enforcement (or revert)
is a separate operation — see [`03-enforcement-toggle-api.md`](./03-enforcement-toggle-api.md).

---

## Validation before write

Always run Quick Analysis (or its API equivalent) on the v\*
before publishing in an automated flow. The Cisco doc covers
the analysis-side endpoints; treat publish without prior
analysis as an unverified change.

---

## Error handling

| HTTP | Meaning | Action |
|---|---|---|
| 200 / 201 | Success | Log; continue |
| 400 | Validation error | Inspect message; the input was malformed (probably a schema issue in your script) |
| 401 | Auth failed | Check key + secret + signing; check key hasn't expired or been revoked |
| 403 | Authorized but not entitled | Capabilities or RBAC scope is wrong; right-size in API Keys |
| 404 | Resource not found | Stale ID; re-list to refresh |
| 409 | Conflict — concurrent edit | Re-read, merge, retry with new ETag/version |
| 429 | Rate-limited | Back off; CSW exposes its rate limits per release |
| 5xx | Cluster-side issue | Retry with backoff; alert if persistent |

---

## See also

- [`01-authentication.md`](./01-authentication.md)
- [`03-enforcement-toggle-api.md`](./03-enforcement-toggle-api.md)
- [`05-gitops-pattern.md`](./05-gitops-pattern.md)
- [`../analysis/08-import-export.md`](../analysis/08-import-export.md)
- Cisco: [Secure Workload OpenAPIs](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/secure-workload-openapis.html)
