# API Design Rules Catalog

Each rule has an ID, a default severity, what to check, why it matters, the
fix, and a source. Severity is a default: adjust it to the domain (see
"Understand the context" in SKILL.md).

**Sources.** RFCs define the HTTP protocol and are normative. OWASP API
Security Top 10 (2023) is the reference for API security risks. Google AIP,
Microsoft REST API Guidelines and Zalando RESTful API Guidelines are widely
used *conventions*, not rules. Present them that way.

## Index

| Category | Rules |
|---|---|
| [SEC: Security](#sec-security) | SEC-001 to SEC-005 |
| [IDM / CON / ASY: Reliability](#idm--con--asy-reliability) | IDM-001, CON-001, ASY-001 |
| [STS: Status codes](#sts-status-codes) | STS-001 to STS-006 |
| [ERR: Errors](#err-errors) | ERR-001 to ERR-003 |
| [MET: Methods](#met-methods) | MET-001 to MET-003 |
| [PAG: Pagination](#pag-pagination) | PAG-001 to PAG-003 |
| [DAT: Data types](#dat-data-types) | DAT-001 to DAT-004 |
| [VER: Versioning](#ver-versioning) | VER-001, VER-002 |
| [NAM: Naming](#nam-naming) | NAM-001 to NAM-003 |

---

## SEC: Security

### SEC-001 · Object access without ownership check (BOLA)
**Severity:** 🔴 Critical
**Check:** Any operation that takes an object id from the client
(`/orders/{id}`, `?accountId=`, an id in the body) and returns or changes that
object. In code: is the lookup filtered by the authenticated user, tenant or
permission? In a spec only: report as **Verify**, since ownership checks are
invisible there.
**Why:** the most common and most exploited API vulnerability. Any logged-in
user changes the id and reads or edits someone else's data. Sequential ids make
it trivial.
**Fix:** authorize every access at the object level, not only at the route
level. Load by `id AND owner/tenant`, or check permission after loading.
Return `404` (not `403`) when the caller should not know the object exists.
Random ids (UUID) reduce enumeration but are **not** a fix on their own.
```
❌ GET /invoices/1002   (caller owns invoice 1001)  →  200 OK + someone else's invoice
✅ GET /invoices/1002   (caller owns invoice 1001)  →  404 Not Found
```
**Source:** OWASP API Security Top 10 2023, API1: Broken Object Level Authorization.

### SEC-002 · Mass assignment / writable sensitive properties
**Severity:** 🔴 Critical
**Check:** request bodies that accept fields the client should never set
(`id`, `role`, `isAdmin`, `balance`, `status`, `ownerId`, `createdAt`), or
code that binds the request straight into a database entity. Also responses
that return sensitive fields the client does not need (password hashes,
internal flags, other users' data).
**Why:** a client adds `"role": "admin"` to a profile update and becomes an
admin. Over-exposed responses leak data even when the UI hides it.
**Fix:** use separate input and output schemas. Mark server-owned fields
`readOnly: true` in OpenAPI and reject or ignore them on input. Return only the
fields the client needs.
```yaml
# ✅ OpenAPI
User:
  properties:
    id:        { type: string, readOnly: true }
    role:      { type: string, readOnly: true }
    name:      { type: string }
```
**Source:** OWASP API Security Top 10 2023, API3: Broken Object Property Level Authorization.

### SEC-003 · No rate limiting signaled
**Severity:** 🟠 Major (🔴 on login, OTP, password reset, or costly operations)
**Check:** no `429` documented, no rate-limit headers, especially on
authentication endpoints and expensive operations (search, export, report,
sending SMS/e-mail).
**Why:** brute force on login, cost explosion on paid third-party calls, one
client degrading the service for everyone.
**Fix:** enforce limits per client/user/IP and respond `429 Too Many Requests`
with `Retry-After`. Document it in every affected operation.
```
✅ 429 Too Many Requests
   Retry-After: 30
   Content-Type: application/problem+json
```
**Source:** RFC 6585 §4 (429); RFC 9110 §10.2.3 (Retry-After); OWASP API4:2023 Unrestricted Resource Consumption.

### SEC-004 · Sensitive data in the URL
**Severity:** 🔴 Critical for credentials/tokens; 🟠 Major for personal data
**Check:** tokens, API keys, passwords, national ids (CPF, SSN), e-mails or
health data in the path or query string (`/users?email=`, `?token=`).
**Why:** URLs are stored in server logs, proxy logs, browser history,
analytics and `Referer` headers. They leak by design.
**Fix:** credentials go in the `Authorization` header. For lookups by
sensitive data, use `POST /users/search` with the value in the body, or an
opaque id.
```
❌ GET /patients?cpf=12345678900
✅ POST /patients/search   { "cpf": "12345678900" }
```
**Source:** RFC 9110 §17.9 (Disclosure of Sensitive Information in URIs).

### SEC-005 · Authentication not declared per operation
**Severity:** 🟠 Major
**Check:** OpenAPI without `securitySchemes`/`security`, or operations where it
is unclear whether auth is required. Administrative operations
(`/admin/...`, bulk deletes, role changes) without a distinct permission.
**Why:** undocumented auth produces accidentally public endpoints. Admin
functions protected only by "the UI hides the button" are exploitable.
**Fix:** declare a global `security` requirement and opt out explicitly
(`security: []`) on public operations. Document required scopes/roles on
privileged operations and check them server side.
**Source:** OWASP API2:2023 Broken Authentication; OWASP API5:2023 Broken Function Level Authorization.

---

## IDM / CON / ASY: Reliability

### IDM-001 · Non-idempotent side effects without an idempotency key
**Severity:** 🔴 Critical for payments, orders, transfers, bookings, messages
sent to people; 🟡 Minor for harmless creations
**Check:** `POST` (or `PATCH`) operations that trigger effects that must not
happen twice, with no `Idempotency-Key` header or equivalent client-generated
id.
**Why:** networks fail after the server processed the request but before the
client got the answer. The client retries, and the customer is charged twice.
**Fix:** accept an `Idempotency-Key` header. Store the key with the first
response for a defined window (e.g. 24h). A retry with the same key and same
payload returns the stored response; same key with a different payload
returns `422` (or `409`).
```
✅ POST /payments
   Idempotency-Key: 8e03978e-40d5-43e8-bc93-6894a57f9324
   { "amount": "150.00", "currency": "BRL" }
```
**Source:** IETF draft-ietf-httpapi-idempotency-key-header; RFC 9110 §9.2.2 (Idempotent Methods).

### CON-001 · Lost updates on shared resources
**Severity:** 🟠 Major (🔴 when overwrites affect money, stock or clinical data)
**Check:** `PUT`/`PATCH` on resources that more than one user or process can
edit, with no `ETag`/`If-Match` or version field.
**Why:** two users load the same record, both save, and the second silently
erases the first one's changes.
**Fix:** return an `ETag` on reads. Require `If-Match` on writes; respond
`412 Precondition Failed` when it does not match, and `428 Precondition
Required` when it is missing (if you make it mandatory).
```
GET /appointments/7          →  200 OK, ETag: "v3"
PUT /appointments/7
If-Match: "v3"               →  200 OK (or 412 if someone saved "v4" meanwhile)
```
**Source:** RFC 9110 §8.8.3 (ETag), §13.1.1 (If-Match), §15.5.13 (412); RFC 6585 §3 (428).

### ASY-001 · Long-running work in a synchronous request
**Severity:** 🟠 Major
**Check:** operations that generate reports, exports, imports, batch
processing or call slow third parties, and respond only when finished.
**Why:** gateway and client timeouts (often 30 to 60s) cut the request while
the work continues. The client retries and the work runs twice, or the result
is lost.
**Fix:** respond `202 Accepted` with a status resource. The client polls it
(or receives a webhook) until it points to the result.
```
POST /reports              →  202 Accepted
                              Location: /report-jobs/91
GET  /report-jobs/91       →  200 { "status": "running" }
GET  /report-jobs/91       →  200 { "status": "done", "result": "/reports/55" }
```
**Source:** RFC 9110 §15.3.3 (202 Accepted).

---

## STS: Status codes

### STS-001 · Creation without 201 and Location
**Severity:** 🟡 Minor
**Check:** `POST` that creates a resource and returns `200`, or `201` without a
`Location` header.
**Why:** `201` tells clients a resource was created, and `Location` tells them
where. Generic HTTP clients, caches and generated SDKs rely on it.
**Fix:**
```
❌ POST /appointments  →  200 OK  { "id": 42 }
✅ POST /appointments  →  201 Created
                          Location: /appointments/42
                          { "id": 42, ... }
```
**Source:** RFC 9110 §15.3.2 (201), §10.2.2 (Location).

### STS-002 · Errors returned with 200
**Severity:** 🟠 Major
**Check:** responses like `200 { "success": false, "error": "..." }`, or a
generic wrapper where the status is always `200`.
**Why:** monitoring, retries, caches, gateways and every HTTP client decide by
status code. With `200` errors, alerts stay silent, failed responses get
cached, and every client must parse the body to know if the call worked.
**Fix:** use the right 4xx/5xx code with an error body (see ERR-001).
```
❌ 200 OK  { "success": false, "message": "Insufficient balance" }
✅ 422 Unprocessable Content  (application/problem+json)
```
**Source:** RFC 9110 §15 (Status Codes).

### STS-003 · 401 vs 403 mixed up
**Severity:** 🟡 Minor
**Check:** `403` for missing/invalid credentials, `401` for a valid user
lacking permission, or `401` without `WWW-Authenticate`.
**Why:** clients use `401` to trigger re-login or token refresh. Using it for
"permission denied" causes refresh loops; using `403` for expired tokens
prevents refresh.
**Fix:** `401` = not authenticated (missing, invalid or expired credentials),
with `WWW-Authenticate`. `403` = authenticated but not allowed. When the
caller must not learn the resource exists, use `404` (see SEC-001).
**Source:** RFC 9110 §15.5.2 (401), §15.5.4 (403), §11.6.1 (WWW-Authenticate).

### STS-004 · Wrong client error code
**Severity:** 🟡 Minor
**Check:** every client error returned as `400`, or `500` for client mistakes
(invalid input, not found, duplicates).
**Why:** clients cannot react correctly: retry, fix the input, or show "already
exists". A `500` for bad input also pollutes error dashboards and pages the
on-call team.
**Fix:**
| Situation | Code |
|---|---|
| Malformed request (invalid JSON, wrong type) | `400 Bad Request` |
| Well-formed but fails business validation | `422 Unprocessable Content` |
| Conflicts with current state (duplicate, invalid state transition) | `409 Conflict` |
| Resource does not exist | `404 Not Found` |
| Unexpected server failure | `500 Internal Server Error` |

Choosing between `400` and `422` for validation is a convention. Pick one and
use it consistently.
**Source:** RFC 9110 §15.5.1 (400), §15.5.5 (404), §15.5.10 (409), §15.5.21 (422).

### STS-005 · Only the success response is documented
**Severity:** 🟠 Major
**Check:** OpenAPI operations that document only `200`/`201`.
**Why:** client developers discover error cases in production. Generated SDKs
cannot type errors.
**Fix:** document every error the operation can return, with its schema
(typically `400`/`422`, `401`, `403`, `404`, `409`, `429`). Reuse them via
`components/responses`.
**Source:** OpenAPI Specification 3.1, Responses Object.

### STS-006 · Body sent with 204
**Severity:** 🟡 Minor
**Check:** `204 No Content` with a response body, or clients expected to read
data from it.
**Why:** a `204` response cannot contain content. Some clients and proxies drop
or mishandle it.
**Fix:** return `200` with the body, or `204` with nothing.
**Source:** RFC 9110 §15.3.5 (204).

---

## ERR: Errors

### ERR-001 · No standard error format
**Severity:** 🟠 Major
**Check:** errors as plain strings, different shapes per endpoint, or no
documented error schema.
**Why:** every client writes custom parsing for every endpoint. A standard
format allows shared error handling and tooling.
**Fix:** use Problem Details, `Content-Type: application/problem+json`.
```json
{
  "type": "https://api.example.com/problems/insufficient-balance",
  "title": "Insufficient balance",
  "status": 422,
  "detail": "Balance is 30.00 BRL, transfer requires 150.00 BRL.",
  "instance": "/transfers/abc123"
}
```
**Source:** RFC 9457 (Problem Details for HTTP APIs).

### ERR-002 · Internal details leaked in errors
**Severity:** 🔴 Critical
**Check:** stack traces, exception class names, SQL, file paths, internal
hostnames or framework versions in error bodies or headers (`X-Powered-By`,
`Server`).
**Why:** gives attackers a map of the stack and sometimes the query structure
needed for injection.
**Fix:** generic `detail` for 5xx with a correlation id (`instance` or a trace
id) that points to the full error in server logs.
**Source:** OWASP API8:2023 Security Misconfiguration.

### ERR-003 · Validation errors without field details
**Severity:** 🟡 Minor
**Check:** validation failures returned as one generic message, or only the
first error.
**Why:** the client cannot highlight the wrong fields and the user fixes one
error per attempt.
**Fix:** return all field errors at once as a Problem Details extension.
```json
{
  "type": "https://api.example.com/problems/validation",
  "title": "Invalid request",
  "status": 422,
  "errors": [
    { "pointer": "/email", "detail": "must be a valid e-mail" },
    { "pointer": "/birthDate", "detail": "must be in the past" }
  ]
}
```
**Source:** RFC 9457 §3.2 (Extension Members); JSON Pointer RFC 6901.

---

## MET: Methods

### MET-001 · GET with side effects
**Severity:** 🔴 Critical when it deletes, pays or sends; 🟠 Major otherwise
**Check:** `GET` that changes state (`GET /orders/5/cancel`,
`GET /emails/send`).
**Why:** `GET` is defined as safe. Crawlers, link previews, prefetching,
caches and automatic retries call it freely and trigger the effect.
**Fix:** use `POST` (or `DELETE`/`PATCH` as appropriate).
```
❌ GET  /orders/5/cancel
✅ POST /orders/5/cancel
```
**Source:** RFC 9110 §9.2.1 (Safe Methods).

### MET-002 · PUT used for partial update
**Severity:** 🟡 Minor
**Check:** `PUT` that accepts a subset of fields and keeps the rest, or `PATCH`
with no documented semantics.
**Why:** `PUT` means "replace with this representation". Clients that follow
the standard send partial data and expect missing fields to be cleared, or the
reverse. Data gets erased or silently kept.
**Fix:** `PUT` replaces the whole resource. Use `PATCH` for partial updates and
document its format: JSON Merge Patch (`application/merge-patch+json`) or JSON
Patch (`application/json-patch+json`).
**Source:** RFC 9110 §9.3.4 (PUT); RFC 5789 (PATCH); RFC 7396 (JSON Merge Patch); RFC 6902 (JSON Patch).

### MET-003 · GET with a request body
**Severity:** 🟠 Major
**Check:** `GET` operations that expect a body (common for complex filters).
**Why:** a body in `GET` has no defined semantics. Many proxies, caches, CDNs
and client libraries drop it, so the filter silently disappears.
**Fix:** use query parameters, or `POST /resources/search` for complex or
sensitive filters (see SEC-004).
**Source:** RFC 9110 §9.3.1 (GET).

---

## PAG: Pagination

### PAG-001 · Unbounded collection
**Severity:** 🟠 Major (🔴 on large or growing tables)
**Check:** list operations without pagination, or with a client `limit` that
has no enforced maximum.
**Why:** works with 100 rows in development, times out or exhausts memory
with 2 million in production. `?limit=1000000` becomes a denial of service.
**Fix:** paginate every collection, set a default page size and enforce a
maximum (e.g. default 20, max 100).
**Source:** OWASP API4:2023 Unrestricted Resource Consumption; Google AIP-158 (Pagination).

### PAG-002 · Offset pagination on large or fast-changing data
**Severity:** 🟡 Minor (🟠 for feeds, logs, sync jobs)
**Check:** `?page=`/`?offset=` on large tables or data that changes while
clients paginate.
**Why:** inserts and deletes shift the offset, so items are skipped or repeated.
Deep offsets (`OFFSET 500000`) get slow because the database still scans the
skipped rows.
**Fix:** cursor (keyset) pagination: return an opaque `nextCursor` based on a
stable, unique sort key. Offset is acceptable for small, mostly static data
or when the UI needs "jump to page N".
```
✅ GET /events?limit=50
   → { "items": [...], "nextCursor": "eyJpZCI6MTIzNH0" }
   GET /events?limit=50&cursor=eyJpZCI6MTIzNH0
```
**Source:** Google AIP-158; Zalando RESTful API Guidelines (Pagination).

### PAG-003 · Inconsistent pagination contract
**Severity:** 🟡 Minor
**Check:** different parameter names or response shapes across collections
(`page/size` here, `offset/limit` there, `data` vs `items`).
**Why:** clients cannot write one pagination helper.
**Fix:** one contract for all collections. Optionally expose navigation with
the `Link` header (`rel="next"`).
**Source:** RFC 8288 (Web Linking).

---

## DAT: Data types

### DAT-001 · Ambiguous dates and times
**Severity:** 🟠 Major
**Check:** timestamps without a timezone offset (`2026-03-10 14:00`), local
formats (`10/03/2026`), or epoch numbers with no documented unit.
**Why:** clients in other timezones shift the time by hours. `10/03` means
March 10 or October 3 depending on the reader.
**Fix:** RFC 3339 timestamps with offset, preferably UTC
(`2026-03-10T17:00:00Z`). Use plain dates (`2026-03-10`) when the time is
irrelevant (birth dates).
**Source:** RFC 3339 (Date and Time on the Internet: Timestamps).

### DAT-002 · Money as floating point
**Severity:** 🟠 Major (🔴 in payment or billing APIs)
**Check:** amounts as JSON numbers with decimals (`"amount": 10.1`) typed as
`number`/`float`, or with no currency.
**Why:** binary floating point cannot represent most decimal values
exactly. Sums drift by cents and reconciliations fail. Without currency the
value is ambiguous.
**Fix:** either a decimal string (`"amount": "10.10"`) or an integer in minor
units (`"amountInCents": 1010`), always with an ISO 4217 currency code.
```json
✅ { "amount": "10.10", "currency": "BRL" }
```
**Source:** ISO 4217; RFC 8259 §6 (number interoperability).

### DAT-003 · Large integer ids as JSON numbers
**Severity:** 🟡 Minor (🟠 for 64-bit or snowflake ids)
**Check:** ids typed `integer` with `int64` values.
**Why:** JavaScript and many JSON parsers use IEEE 754 doubles. Integers above
2^53 lose precision, so the client silently references the wrong record.
**Fix:** serialize ids as strings. Treat ids as opaque.
**Source:** RFC 8259 §6.

### DAT-004 · Undocumented or numeric enums
**Severity:** 🟡 Minor
**Check:** status fields as magic numbers (`"status": 3`), or enums with no
documented values or evolution policy.
**Why:** `3` means nothing in a log. Clients break when a new value appears.
**Fix:** string enums with readable values (`"status": "cancelled"`),
documented with `enum` in OpenAPI. State that clients must tolerate unknown
values, so adding one is not a breaking change.
**Source:** Zalando RESTful API Guidelines (Enumerations); Google AIP-126.

---

## VER: Versioning

### VER-001 · No versioning or inconsistent versioning
**Severity:** 🟠 Major for public or partner APIs; 🟡 Minor for internal APIs with one consumer
**Check:** no version strategy, or a mix (`/v1/` in some paths, a header in
others, nothing elsewhere).
**Why:** the first breaking change forces every client to update at the same
moment.
**Fix:** choose one strategy (URL major version `/v1/` is the simplest and
most common) and apply it everywhere. Keep changes additive within a version.
**Source:** Microsoft REST API Guidelines (Versioning); Google AIP-180 (Backwards compatibility).

### VER-002 · Breaking change without deprecation
**Severity:** 🟠 Major
**Check:** when comparing versions or reviewing a change: removed or renamed
fields, new required inputs, changed types, removed enum values, changed
status codes or URLs, without a new version or a deprecation period.
**Why:** clients break in production with no warning.
**Fix:** ship the change in a new version, keep the old one running for a
published period, and signal it with `Deprecation` and `Sunset` headers (plus
a `Link` to migration docs).
```
✅ Deprecation: @1798761600
   Sunset: Fri, 30 Jul 2027 23:59:59 GMT
   Link: <https://api.example.com/docs/migrate-v2>; rel="deprecation"
```
**Source:** RFC 9745 (Deprecation header); RFC 8594 (Sunset header); Google AIP-180.

---

## NAM: Naming

### NAM-001 · Verbs in paths for CRUD operations
**Severity:** 🟡 Minor
**Check:** `/createUser`, `/getOrders`, `/deleteProduct/5`.
**Why:** the HTTP method already carries the action. Verb paths multiply
endpoints and hide the resource model.
**Fix:** resources as nouns, actions as methods. For real actions that are not
CRUD (cancel, approve, send), a sub-resource or custom action is fine:
`POST /orders/5/cancel`.
```
❌ POST /createUser        ✅ POST   /users
❌ GET  /getOrders         ✅ GET    /orders
❌ POST /deleteProduct/5   ✅ DELETE /products/5
```
**Source:** Google AIP-131 to AIP-136 (standard and custom methods); Microsoft REST API Guidelines.

### NAM-002 · Inconsistent naming and casing
**Severity:** 🟡 Minor
**Check:** mixed plural/singular collections (`/user` and `/orders`), mixed
casing in paths (`/order-items` and `/orderItems`) or fields (`created_at` and
`updatedAt`).
**Why:** clients guess instead of knowing. Every inconsistency becomes a bug
or a support question.
**Fix:** plural collections, one casing for paths (kebab-case is common) and
one for fields (camelCase or snake_case). Apply the API's existing majority
convention rather than introducing a new one.
**Source:** Zalando RESTful API Guidelines (Naming); Google AIP-122.

### NAM-003 · Deep nesting
**Severity:** 🟡 Minor
**Check:** paths deeper than `collection/{id}/sub-collection`, for example
`/clinics/1/doctors/2/patients/3/exams/4`.
**Why:** clients must know the whole parent chain to reach a resource that has
its own unique id, and every level is another place to forget an ownership
check.
**Fix:** nest one level for clear ownership. Give resources with unique ids
their own top-level path (`/exams/4`) and filter with query params
(`/exams?patientId=3`).
**Source:** Microsoft REST API Guidelines; Zalando RESTful API Guidelines.
