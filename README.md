# api-design-reviewer

🇧🇷 [Leia em português](README.pt-BR.md)

A Claude skill that reviews REST/HTTP API designs like a senior API reviewer would: it finds the problems that actually hurt clients and operators, explains why each one matters, and shows the fix at the contract level, so it works for any language or framework.

Linters check syntax ("missing description", "duplicate operationId"). This skill catches the problems that need judgment:

- `POST /payments` with no idempotency key will **charge the customer twice** when the network fails and the client retries.
- `GET /invoices/{id}` that only checks login lets any user **read someone else's invoice** by changing the id.
- Offset pagination on a busy table **skips and repeats records** while users page through it.
- `"amount": 10.1` as a float makes **totals drift by cents** and breaks reconciliation.

## What it checks

32 rules in 9 categories, each backed by an RFC, the OWASP API Security Top 10 or a widely used API guideline.

| Category | What it looks for |
|---|---|
| **SEC** Security | Broken object level authorization (BOLA), mass assignment, missing rate limits, sensitive data in URLs, undeclared auth |
| **IDM** Idempotency | Payments, orders and messages that can run twice on retry |
| **CON** Concurrency | Lost updates when two users edit the same record (ETag / If-Match) |
| **ASY** Async operations | Long-running work that should return `202 Accepted` |
| **STS** Status codes | `200` with an error body, `201` without `Location`, `401` vs `403`, `400` vs `409` vs `422` |
| **ERR** Errors | Problem Details (RFC 9457), leaked stack traces, field-level validation errors |
| **MET** Methods | `GET` with side effects, `PUT` used as partial update, `GET` with a body |
| **PAG** Pagination | Unbounded lists, offset vs cursor, inconsistent contracts |
| **DAT** Data types | Money as float, ambiguous dates, large ids losing precision in JavaScript, magic-number enums |
| **VER** Versioning | Missing strategy, breaking changes without `Deprecation` / `Sunset` |
| **NAM** Naming | Verbs in paths, inconsistent casing, deep nesting |

The full catalog is in [references/rules.md](references/rules.md). It also works as a study guide on HTTP API design.

## Example finding

````markdown
### 🔴 IDM-001 · Payment creation without an idempotency key
**Where:** `POST /payments`
**Confidence:** Confirmed

**Problem:** if the client times out after the server processed the payment
and retries, the customer is charged twice.

**Before**
```
POST /payments
{ "amount": 150.0 }
```

**After**
```
POST /payments
Idempotency-Key: 8e03978e-40d5-43e8-bc93-6894a57f9324
{ "amount": "150.00", "currency": "BRL" }
```

**Breaking change:** No (the header can be optional at first)
**Source:** IETF draft-ietf-httpapi-idempotency-key-header; RFC 9110 §9.2.2
````

Every review starts with a summary by severity, highlights what the API already does well, lists what could not be reviewed, and ends with a suggested order of work.

## Install

**Claude Code**, with the [skills CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add pedroCifoni/api-design-reviewer
```

Or clone it into your personal skills folder:

```bash
git clone https://github.com/pedroCifoni/api-design-reviewer.git ~/.claude/skills/api-design-reviewer
```

**Claude.ai / Claude Desktop:** download this repository as a ZIP and upload it as a custom skill in Claude's settings.

## Usage

Give Claude any of these and ask for a review:

- an **OpenAPI / Swagger** file (YAML or JSON)
- **route or controller code** in any language (Express, ASP.NET Core, Spring, FastAPI, Laravel, Go...)
- a **written proposal** of an API you are still designing
- a **single question** about a decision

Example prompts:

```text
Review the API design in openapi.yaml.
```

```text
Review the endpoints in src/controllers before I open the PR.
```

```text
I'm designing an appointment booking API: POST /bookings, GET /bookings?userId=,
PUT /bookings/{id} to change the date. What problems do you see?
```

```text
Should this endpoint return 403 or 404 when the user doesn't own the order?
```

The report comes back in your language. Rule IDs, HTTP terms and status codes stay in English.

## How it reviews

1. **Builds a route inventory:** method, path, inputs, responses and auth for every operation. From code, it also notes whether each lookup is filtered by the current user.
2. **Reads the context:** the domain and consumers set the severity. A missing idempotency key is critical on `/payments` and minor on `/feedback`.
3. **Respects your conventions:** it flags inconsistencies, not choices that differ from its own preferences.
4. **Applies the rules**, security first.
5. **Calibrates the findings:** every finding points to evidence and is marked **Confirmed** or **Verify** (for example, an ownership check cannot be seen in a spec alone). Repeated issues are grouped, and fixes that would break existing clients come with a migration path.

It does not invent problems to fill a report. A short report on a good API is a good result.

## Sources

- [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110): HTTP Semantics
- [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457): Problem Details for HTTP APIs
- [OWASP API Security Top 10 (2023)](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)
- [IETF Idempotency-Key header draft](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/)
- RFC 3339, 5789, 6585, 6902, 7396, 8259, 8288, 8594, 9745
- [Google API Improvement Proposals](https://google.aip.dev/), [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines), [Zalando RESTful API Guidelines](https://opensource.zalando.com/restful-api-guidelines/)

RFCs define the protocol. The guidelines are conventions, and the skill presents them that way.

## Roadmap

- [ ] **Evals:** APIs with planted defects and an answer key, comparing Claude with and without the skill
- [ ] **Design mode:** generate a complete OpenAPI contract from requirements
- [ ] **Diff mode:** detect breaking changes between two versions of a spec

## Contributing

Issues and pull requests are welcome, especially:

- a rule you think is missing, with its source,
- a review where the skill got something wrong (paste the input and the finding),
- examples of real-world API mistakes.

## License

[MIT](LICENSE) © Pedro Cifoni Montezuma
