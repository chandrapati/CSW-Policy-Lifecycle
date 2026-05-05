# 01 — API Authentication

CSW OpenAPI uses **API key + secret** authentication with
HMAC-signed requests. Every script that talks to the cluster
needs a key, scoped narrowly, rotated regularly, and stored
outside source control.

> **Cisco source.** [Secure Workload OpenAPIs — Authentication](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/secure-workload-openapis.html#authentication).

---

## Creating an API key

*Manage → API Keys → Create New Key.*

The dialog asks for:

| Field | Recommendation |
|---|---|
| **Description** | What this key is for and who owns it (e.g. `gitops-reconciler-payments-api / owner=platform-eng`) |
| **Capabilities / Roles** | The minimum set of capabilities the script needs — never assign tenant-admin to a script |
| **Expiry** | Set a real expiry; treat keys as short-lived |

CSW returns:

- An **API key** (short identifier).
- An **API secret** (longer string).

The secret is **only shown once**. Capture it into your secret
store at creation time.

---

## Capability scoping

Don't attach more capabilities than the script needs:

| Capability area | Use sparingly |
|---|---|
| `app_policy_management` | Policy CRUD per workspace |
| `flow_inventory_query` | Read-only access to flow / inventory data — useful for analytics scripts |
| `sensor_management` | Agent administration — typically *not* needed for policy work |
| `external_integration` | For tools writing into CSW from outside |
| `app_data_upload` | Inventory / annotation upload — narrow to specific upload pipelines |
| `developer` / tenant-admin equivalents | Reserve for human operators, not scripts |

A GitOps reconciler reading and writing policy on one workspace
should have only `app_policy_management` scoped to that scope's
RBAC, not tenant-wide rights.

---

## Storing the key + secret

**Never** in source control. Acceptable storage:

| Where | When |
|---|---|
| Secret manager (Vault, AWS Secrets Manager, Azure Key Vault, GCP Secret Manager) | Production / CI |
| Environment variables sourced from secret manager at runtime | Production / CI |
| `~/.config/csw/credentials.json` (file mode 0600) | Developer laptop |
| `.env` file *gitignored* | Local dev only — and gitignored at the repo level |

If a secret is ever committed, **rotate immediately** — revoke
in the UI, generate a new one, scrub the git history, force-push
to the affected branches, and audit cluster activity logs since
the secret was exposed.

---

## Signing requests

The OpenAPI signs each request with HMAC-SHA256 over a canonical
string built from the request path, query, body hash, and
timestamp. The Cisco-supplied Python SDK (and the curl helpers
in the docs) handle the canonicalization.

A minimal Python example using the official SDK pattern:

```python
from tetpyclient import RestClient

CLUSTER = "https://csw.example.com"
API_KEY_ID = os.environ["CSW_API_KEY"]
API_KEY_SECRET = os.environ["CSW_API_SECRET"]

client = RestClient(
    CLUSTER,
    api_key=API_KEY_ID,
    api_secret=API_KEY_SECRET,
    verify=True,  # cluster cert; set CA bundle if internal CA
)

resp = client.get("/openapi/v1/app_scopes")
resp.raise_for_status()
print(resp.json())
```

Three things to never skip:

1. `verify=True` (or a CA bundle path) — never disable TLS
   verification. If your cluster has a private CA, set
   `requests.Session.verify` to the CA bundle path.
2. **Don't log** the API key or secret. Add a redaction filter
   to your logger.
3. **Set a reasonable timeout** on every call. Default `requests`
   timeout is *None* — that's a hang waiting to happen.

---

## Example: minimal raw-curl request

For ad-hoc shell work where the SDK isn't an option:

```bash
TS=$(date -u +%Y-%m-%dT%H:%M:%S+0000)
METHOD=GET
URI=/openapi/v1/app_scopes
BODY_HASH=$(printf '' | openssl dgst -sha256 -binary | base64)
TO_SIGN="${METHOD}\n${URI}\n${BODY_HASH}\n${TS}"
SIG=$(printf "${TO_SIGN}" |
  openssl dgst -sha256 -hmac "${CSW_API_SECRET}" -binary |
  base64)

curl -sS \
  --cacert /path/to/cluster-ca.crt \
  -H "Timestamp: ${TS}" \
  -H "Id: ${CSW_API_KEY}" \
  -H "Authorization: ${SIG}" \
  "https://csw.example.com${URI}"
```

The exact canonical string and header names depend on your
release; the
[OpenAPIs Authentication section](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/secure-workload-openapis.html#authentication)
is the source of truth — don't crib from this README in
production.

---

## Rotation

Set a calendar reminder per the agreed cadence (every 90 days
is reasonable). Rotation flow:

1. Create the new key with the same capabilities.
2. Push the new key + secret to the secret manager.
3. Roll callers (CI/CD, scripts, integrations) to the new
   credential.
4. Verify all callers are healthy on the new key.
5. Revoke the old key in the UI.

---

## Audit and detection

Every API call appears in CSW's activity / audit log. Things
to alert on:

- A key being used from an IP / source you didn't expect.
- A key making calls outside its capability set (CSW will reject
  these, but the *attempt* itself is signal).
- A key making calls outside the scoped scope.
- A burst of calls suggesting a leaked credential being scanned.

---

## See also

- [`02-openapi-policies.md`](./02-openapi-policies.md)
- [`03-enforcement-toggle-api.md`](./03-enforcement-toggle-api.md)
- [`05-gitops-pattern.md`](./05-gitops-pattern.md)
- Cisco: [Secure Workload OpenAPIs — Authentication](https://www.cisco.com/c/en/us/td/docs/security/workload_security/secure_workload/user-guide/4_0/cisco-secure-workload-user-guide-on-prem-v40/secure-workload-openapis.html#authentication)
