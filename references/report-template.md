# Report Template

Use this structure for full reviews. Translate headings and prose into the
user's language; keep rule IDs, HTTP terms and status codes in English.

````markdown
# API Design Review: <API name>

**Scope:** <N operations reviewed> · <input type: OpenAPI 3.x / source code / proposal>
**Context:** <inferred domain and consumers, e.g. "health scheduling API, used by a public mobile app">

## Summary

| 🔴 Critical | 🟠 Major | 🟡 Minor |
|---|---|---|
| <n> | <n> | <n> |

<2 to 3 sentences: overall state of the API and the single most important
thing to fix first.>

**What is already good:** <conventions or decisions worth keeping, e.g.
"consistent cursor pagination", "Problem Details used everywhere">

## Findings

### 🔴 <RULE-ID> · <short title>
**Where:** `<METHOD> <path>` (+ <n> more: `<path>`, `<path>`)
**Confidence:** Confirmed | Verify (<what to check in the code>)

**Problem:** <what happens, in one or two sentences, written as the
consequence for a client or the business.>

**Before**
```
<request/response or OpenAPI snippet as it is today>
```

**After**
```
<the corrected request/response or OpenAPI snippet>
```

**Breaking change:** No | Yes: <migration path>
**Source:** <from the rules catalog>

---

<repeat for each finding, sorted by severity, then by number of affected operations>

## Not reviewed

- <parts that could not be assessed and why, e.g. "auth: no security schemes
  in the spec", "ownership checks: only the spec was provided">

## Suggested order of work

1. <highest impact per effort first>
2. ...
````

## Short answers

For a single question ("404 or 403 here?"), skip the template. Answer in a few
sentences, cite the rule ID and source, and show a before/after only if it
helps.
