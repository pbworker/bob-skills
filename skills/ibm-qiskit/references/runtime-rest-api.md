# Qiskit Runtime REST API (when you're not in Python)

> The **IBM Quantum System API** — the REST surface behind `qiskit-ibm-runtime`,
> for other languages, thin services, or curl. Verified against the provided
> OpenAPI spec (**IBM Quantum System API, version `2025-08-15`**) and the IBM
> Quantum REST setup docs. The Python SDK is still the primary, higher-level
> interface — reach for REST only when you can't run Qiskit itself.
>
> **Verification status:** the auth flow (IAM token → `Authorization: Bearer` +
> `Service-CRN`), base URL, and the **read** endpoints `GET /v1/backends` and
> `GET /version` are **live-confirmed** against a real instance (both HTTP 200).
> The **`POST /v1/jobs`** body below (serializing ISA circuits into `params`) is
> from the OpenAPI spec, **not** a live round-trip — build and submit jobs with
> the SDK and use REST mainly to list/poll/manage, as noted under "Practical
> guidance."

## Base URL, versioning, auth

- **Base URL:** `https://quantum.cloud.ibm.com/api`
- **Versioning:** date-based via the **`IBM-API-Version`** header (e.g.
  `2025-08-15`). Omit it to default to the latest; `GET /version` and
  `GET /versions` report what's available. `YYYY-MM-DD` format (the old `YYYYMMDD`
  is deprecated).
- **Auth: IAM Bearer token (JWT).** The same flow as other IBM Cloud services:
  exchange your **API key** for a short-lived IAM access token, then send it plus
  your instance **CRN**.

```bash
# 1) API key -> IAM bearer token (token lasts ~1h)
TOKEN=$(curl -s -X POST 'https://iam.cloud.ibm.com/identity/token' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d "grant_type=urn:ibm:params:oauth:grant-type:apikey&apikey=$IQP_API_TOKEN" \
  | python3 -c "import sys,json;print(json.load(sys.stdin)['access_token'])")

# 2) Call the API: Bearer token + Service-CRN (instance) header
curl -s -X GET 'https://quantum.cloud.ibm.com/api/v1/usage' \
  -H 'accept: application/json' \
  -H "Authorization: Bearer $TOKEN" \
  -H "Service-CRN: $INSTANCE_CRN"
```
- Store the API key in an env var (`IQP_API_TOKEN`) — never hardcode/commit it.
  One API key works for either region.
- **401 Unauthorized** almost always means you sent the **raw API key** where a
  **Bearer token** is required (or the token expired) — re-exchange it.

## Endpoints (from the spec)

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/v1/backends` | List available backends (QPUs/simulators) |
| `GET` | `/v1/backends/{backend-name}` | Backend details |
| `GET` | `/v1/backends/{backend-name}/configuration` | Static configuration |
| `GET` | `/v1/backends/{backend-name}/properties` | Calibration/error properties |
| `GET` | `/v1/backends/{backend-name}/lanes` | Lanes configuration |
| `GET` | `/v1/jobs` | List your jobs |
| `POST` | `/v1/jobs` | **Run a job** — invoke a Qiskit Runtime primitive |
| `GET` | `/v1/jobs/{job-id}` | Job status / details |
| `DELETE` | `/v1/jobs/{job-id}` | Delete a job |
| `POST` | `/v1/jobs/{job-id}/cancel` | Cancel a running job |
| `GET` | `/version`, `/versions` | Latest / supported API versions |

(`/v1/usage` for account usage is also exposed; see the REST setup docs.)

## Running a primitive over REST (`POST /v1/jobs`)

Headers: `Authorization: Bearer <token>`, `Service-CRN: <instance CRN>`,
optionally `IBM-API-Version: 2025-08-15`.

The request body is a **`RunJobBody`** with fields:
`backend`, `program_id`, `params`, `id`, `log_level`, `storage`, `timeout_secs`.

- **`program_id`** selects the primitive: `"sampler"` or `"estimator"` (also
  `"noise-learner"`).
- **`backend`** is the backend name (e.g. `"ibm_brisbane"`).
- **`params`** carries the PUBs and options — **the circuits must already be ISA
  (transpiled) and serialized** (the SDK uses its `RuntimeEncoder`/QPY/OpenQASM 3
  under the hood). Hand-building `params` is involved; if you can run Python at
  all, build and submit with the SDK and only use REST to poll/manage.

```bash
curl -s -X POST 'https://quantum.cloud.ibm.com/api/v1/jobs' \
  -H "Authorization: Bearer $TOKEN" -H "Service-CRN: $INSTANCE_CRN" \
  -H 'Content-Type: application/json' \
  -d '{
        "program_id": "sampler",
        "backend": "ibm_brisbane",
        "params": { "...": "PUBs + options (ISA circuits, serialized)" }
      }'
# -> returns a job id; then poll:
curl -s "https://quantum.cloud.ibm.com/api/v1/jobs/<job-id>" \
  -H "Authorization: Bearer $TOKEN" -H "Service-CRN: $INSTANCE_CRN"
```

## Practical guidance
- **Prefer the SDK.** REST is for non-Python callers or orchestration layers. The
  hard part over REST is serializing ISA circuits/observables into `params`; the
  SDK does this for you.
- **The four-step pattern still applies.** You must transpile to the backend's ISA
  *before* submitting — there's no server-side magic that fixes a non-ISA circuit.
- **Same constraints as the SDK:** Sampler circuits need measurements, Estimator
  circuits don't; jobs queue and cost; mind shot counts.
- Backend `configuration` + `properties` over REST give you the `basis_gates`,
  `coupling_map`, and current error rates if you're transpiling outside Qiskit.

> See the OpenAPI spec in `REFERENCE-Quantum/latest` for exact request/response
> schemas, and the live docs at
> https://quantum.cloud.ibm.com/docs/api/runtime for the full Runtime REST
> reference (this System API is a subset focused on backends + jobs).
