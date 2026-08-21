# Requirement Compliance Review

**Ticket:** PIQ-445 - Improve logging visibility for payment requests
**Branch:** env/piq-more-visibility
**Changes:** 19 files changed, +5095 -2452 lines

## Overall Assessment

**NON-COMPLIANT**

The PR makes peripheral improvements — dependency upgrades, CI scripting fixes, a latency threshold config tweak, and a minor logging data-structure refactor — but does not implement any of the core observability requirements described in the ticket. No new request/response logging, correlation ID propagation, error stack-trace logging, or latency breakdown instrumentation is present in the diff.

## Requirements Coverage

| # | Requirement | Status | Notes |
|---|-------------|--------|-------|
| 1 | Log full request payload (method, path, headers, body) for every incoming HTTP request | ❌ Not Met | No incoming-request logging added |
| 2 | Log response status and latency for every request | ❌ Not Met | Latency threshold config changed but no logging code added |
| 3 | Include correlation ID / trace ID in all log entries | ❌ Not Met | No trace/correlation ID logic present |
| 4 | Log errors with full stack traces | ❌ Not Met | No error-logging changes found |
| 5 | Latency logs should include breakdown (db time, external call time) | ❌ Not Met | No breakdown instrumentation added |
| 6 | All log fields should use consistent snake_case naming | ⚠️ Partial | `data` field structure improved in `payment-iq.http.ts`, but no systematic snake_case enforcement across log fields |

## Issues

### 🔴 HIGH

**[H1] No incoming request payload logging**
**Requirement:** 1 — Log full request payload (method, path, headers, body) for every incoming HTTP request
**Problem:** The diff contains no middleware, interceptor, or handler that logs incoming HTTP request details. The change in `src/http/payment-iq.http.ts` renames an internal interface and adjusts the `data` field shape, but this function handles *outgoing* calls to PaymentIQ — not incoming requests to this service.
**Location:** `src/http/payment-iq.http.ts`
**Suggestion:** Add an HTTP middleware (e.g. Express/Fastify request logger or a Lambda event logger at the handler entry point) that captures and logs `method`, `path`, `headers`, and `body` for every inbound request.

---

**[H2] No response status or latency logging**
**Requirement:** 2 — Log response status and latency for every request
**Problem:** The terraform change lowers `EXTERNAL_HTTP_LATENCY_WARNING_THRESHOLD_MS` from 1000 ms to 800 ms, which implies a latency-warning mechanism exists somewhere, but no code that actually logs response status or records end-to-end latency is added or modified in this PR.
**Location:** `terraform/lambda_env_vars.tf`
**Suggestion:** Instrument the response path (e.g. after `makePaymentIQRequest` resolves) to log status code and elapsed time; wire the same mechanism to the inbound request handler.

---

**[H3] No correlation ID / trace ID propagation**
**Requirement:** 3 — Include correlation ID / trace ID in all log entries
**Problem:** There is no code in the diff that reads, generates, or forwards a correlation/trace ID, nor any middleware that injects it into a logging context.
**Location:** Not present
**Suggestion:** Extract a trace ID from inbound headers (e.g. `x-correlation-id`, `x-amzn-trace-id`) at the entry point and attach it to a request-scoped logger or pass it explicitly to all log calls.

---

**[H4] No error logging with stack traces**
**Requirement:** 4 — Log errors with full stack traces
**Problem:** No error-handling code is added or modified. There is no catch block or global error handler introduced that logs `error.stack`.
**Location:** Not present
**Suggestion:** Add try/catch around request processing and log `error.stack` (or the full `Error` object) via the logging library. Verify that the `@gamingfactory/logging` upgrade (1.0.1 → 1.1.3) exposes this capability, then call it in all error paths.

---

**[H5] No latency breakdown instrumentation**
**Requirement:** 5 — Latency logs should include breakdown (e.g. db time, external call time)
**Problem:** No timing instrumentation is added around database calls or external HTTP calls. The threshold config change in terraform is a configuration value only — not instrumentation.
**Location:** Not present
**Suggestion:** Record start/end timestamps around each distinct phase (DB queries, calls to external services via `makePaymentIQRequest`) and include the breakdown in the structured log entry.

---

### 🟡 MEDIUM

**[M1] snake_case naming not systematically enforced**
**Requirement:** 6 — All log fields should use consistent snake_case naming
**Problem:** The PR improves the shape of one field (`data`) in `payment-iq.http.ts`, but there is no evidence of a broader audit or enforcement of snake_case across all log fields, and the requirement implies this applies to every log entry in the service.
**Location:** `src/http/payment-iq.http.ts`
**Suggestion:** Once logging calls are added (per H1–H5), ensure every log field key uses snake_case. If the `@gamingfactory/logging` library supports a schema or serializer, configure it to enforce the naming convention uniformly.

---

### 🟢 LOW

**[L1] Dependency upgrades not verified against logging requirements**
**Requirement:** 1–6 (indirect)
**Problem:** `@gamingfactory/logging` is bumped from 1.0.1 to 1.1.3 and `@gamingfactory/external-communications` from 1.0.1 to 1.0.7. These may introduce capabilities needed by the ticket (e.g. structured log fields, trace ID support), but without changelogs or usage in the diff, it cannot be determined whether they contribute to satisfying any acceptance criteria.
**Location:** `package.json`
**Suggestion:** Document which new capabilities from these library upgrades are being relied upon for this ticket, and add code that actively uses them to meet the stated requirements.
