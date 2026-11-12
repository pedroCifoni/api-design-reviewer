---
name: api-design-reviewer
description: Reviews REST/HTTP API designs against established standards (RFC 9110, RFC 9457, OWASP API Security Top 10) and returns a prioritized report with concrete before/after fixes. Use when the user shares an OpenAPI/Swagger spec, API route or controller code in any language, or a written API proposal and asks to review, audit, validate or improve it, or asks whether an endpoint, status code, error format, pagination, versioning or idempotency approach is correct. Framework-agnostic.
---

# API Design Reviewer

Review an HTTP API the way a senior API reviewer would: find the problems that
actually hurt clients and operators (duplicate charges, data leaks, lost
updates, broken integrations), explain *why* each one matters, and show the fix
at the contract level so it applies to any language or framework.

## Input you may receive

| Input | What to do first |
|---|---|
| OpenAPI / Swagger (YAML or JSON) | Read it directly. |
| Source code (controllers, routes, handlers, in any language) | Build a **route inventory** first (see Step 1). |
| A written proposal or description | Turn it into a route inventory; state every assumption you made. |
| A single endpoint or question ("should this return 404 or 403?") | Answer directly using the relevant rules; skip the full report. |

If the input is too partial to review (for example, only paths with no
responses), review what is there and list what is missing under
"Not reviewed". Do not stop to ask questions unless the domain is impossible to
infer.

## Workflow

### Step 1: Build the route inventory

For every operation, record: method, path, purpose, request body or params,
documented responses and status codes, and the auth requirement. When reading
code, also note how each handler loads data (for example: by id only, or by id
filtered by the current user). Ownership checks are invisible in a spec, so
this matters for SEC-001.

For more than ~15 operations, show the inventory as a compact table before the
findings. Otherwise keep it internal.

### Step 2: Understand the context

Infer the domain (payments, health, e-commerce, internal tooling...) and the
consumers (public, partners, internal frontend). Context changes severity: a
missing idempotency key on `POST /payments` is critical; on
`POST /feedback` it is minor.

Detect the API's **existing conventions** (casing, pagination style, error
shape, versioning). Consistency beats personal preference: do not flag a
consistent convention just because another one is more common. Flag
*inconsistencies* instead.

### Step 3: Apply the rules

Read [references/rules.md](references/rules.md) and check each category
against the inventory:

1. **SEC**: security (always check first)
2. **IDM / CON / ASY**: idempotency, concurrency, long-running operations
3. **STS / ERR**: status codes and error format
4. **MET**: method semantics
5. **PAG**: pagination
6. **DAT**: data types and formats
7. **VER**: versioning and evolution
8. **NAM**: naming and resource design

### Step 4: Filter and calibrate

Before reporting a finding, check that:

- **There is evidence.** Point to the exact operation, field or line.
- **Confidence is honest.** Mark a finding **Confirmed** when the evidence is in
  the input, and **Verify** when it depends on code you cannot see (for example,
  BOLA risk from a spec alone).
- **Repeats are grouped.** The same rule broken on 12 endpoints is one finding
  with 12 locations, not 12 findings.
- **Breaking changes are flagged.** If the API is already published, say when
  a fix would break clients and suggest a migration path (new version,
  additive change, deprecation period).

### Step 5: Write the report

Follow [references/report-template.md](references/report-template.md).
Sort by severity, then by how many operations each finding affects.

Every finding must include:
- the rule ID and a one-line title,
- where it happens,
- the concrete consequence for a client or the business,
- a **before/after** at the HTTP or OpenAPI level,
- the source (RFC, OWASP or guideline) from the rules catalog.

If the user mentions their stack, add a short implementation hint for that
framework after the contract-level fix. Do not write framework code
otherwise.

## Severity

| Level | Meaning | Examples |
|---|---|---|
| 🔴 Critical | Security hole, data loss/duplication, or incorrect behavior in production | BOLA, double charge on retry, stack traces in responses |
| 🟠 Major | Breaks or confuses clients, blocks evolution, hurts interoperability | 200 with error body, no pagination, float for money |
| 🟡 Minor | Consistency and ergonomics | Mixed casing, verbs in paths |

## Language

Reply in the user's language. Keep rule IDs, HTTP terms, header names and
status codes in their original English form.

## What not to do

- Do not invent problems to fill the report. A short report on a good API is a
  good result. Say what the API does well.
- Do not recommend rewriting the whole API. Prioritize the fixes with the
  highest impact per effort.
- Do not cite sources that are not in the rules catalog unless you are certain
  of them.
- Do not treat a guideline (Google AIP, Microsoft, Zalando) as an absolute
  rule. RFCs define the protocol; guidelines are conventions.
