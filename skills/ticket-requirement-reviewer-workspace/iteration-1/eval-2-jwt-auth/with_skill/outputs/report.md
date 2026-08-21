# Requirement Compliance Review

**Ticket:** PIQ-512 - Add JWT authentication to payment-iq API
**Branch:** env/piq-more-visibility
**Changes:** 19 files changed, +5095 -2452 lines

## Overall Assessment

**NON-COMPLIANT**

The changes on this branch are entirely focused on observability improvements (enhanced logging data structures, HTTP latency threshold tuning) and dependency version bumps — none of the six JWT authentication requirements have been implemented. There is no JWT validation middleware, no Authorization header parsing, no `JWT_SECRET` environment variable, no 401 error handling, and no public endpoint whitelist anywhere in the diff.

## Requirements Coverage

| # | Requirement | Status | Notes |
|---|-------------|--------|-------|
| 1 | All API endpoints must require a valid JWT token in the Authorization header | ❌ Not Met | No auth middleware or header parsing found anywhere in the diff |
| 2 | JWT tokens should be validated against a shared secret from env var `JWT_SECRET` | ❌ Not Met | `JWT_SECRET` is absent from `terraform/lambda_env_vars.tf` and no JWT library was added |
| 3 | Requests with missing or invalid tokens must return 401 Unauthorized | ❌ Not Met | No 401 response logic present |
| 4 | The decoded JWT payload should be available to request handlers | ❌ Not Met | No payload-injection or request-context propagation found |
| 5 | A whitelist of public endpoints (e.g. `/health`) should bypass authentication | ❌ Not Met | No route whitelist or bypass logic present |
| 6 | Token expiry must be enforced | ❌ Not Met | Depends on requirement 2; neither is implemented |

## Issues

### 🔴 HIGH

**[H1] No JWT authentication middleware implemented**
**Requirement:** 1 - All API endpoints must require a valid JWT token
**Problem:** The diff contains zero authentication code. No middleware, interceptor, or guard that validates the `Authorization` header exists in any changed file.
**Location:** `src/http/payment-iq.http.ts` (the only HTTP layer file touched) - interface renamed and data normalised for logging; no auth logic added.
**Suggestion:** Add an authentication middleware (e.g. using `jsonwebtoken` or `jose`) that extracts the `Bearer` token from the `Authorization` header and validates it before passing control to any route handler.

---

**[H2] `JWT_SECRET` environment variable not provisioned**
**Requirement:** 2 - JWT tokens should be validated against `JWT_SECRET` from environment
**Problem:** `terraform/lambda_env_vars.tf` was modified (latency threshold updated) but `JWT_SECRET` was not added. Without the secret in the Lambda environment, token validation cannot function even if the code were present.
**Location:** `terraform/lambda_env_vars.tf`
**Suggestion:** Add `JWT_SECRET = var.jwt_secret` (or equivalent) to the Lambda environment variable block, and define the corresponding Terraform variable.

---

**[H3] No 401 Unauthorized response for missing/invalid tokens**
**Requirement:** 3 - Requests with missing or invalid tokens must return 401
**Problem:** No error-handling path for authentication failures exists in the diff.
**Location:** N/A - absent from the codebase changes entirely
**Suggestion:** In the auth middleware, catch missing headers and JWT verification errors and respond with HTTP 401 and a descriptive error body.

---

**[H4] Decoded JWT payload not propagated to request handlers**
**Requirement:** 4 - Decoded JWT payload should be available to request handlers
**Problem:** No mechanism for attaching the decoded payload to the request context (e.g. `req.user`, a context object, or a DI token) is present.
**Location:** N/A - absent from the codebase changes entirely
**Suggestion:** After successful verification, attach the decoded payload to the request object or equivalent context so downstream handlers can access it.

---

**[H5] No public-endpoint whitelist implemented**
**Requirement:** 5 - A whitelist of public endpoints (e.g. `/health`) should bypass auth
**Problem:** No route-level bypass or whitelist configuration exists in the diff.
**Location:** N/A - absent from the codebase changes entirely
**Suggestion:** Define a configurable list of public paths and check the incoming request path against it before running JWT validation.

---

**[H6] Token expiry enforcement absent**
**Requirement:** 6 - Token expiry must be enforced
**Problem:** This depends on requirements 2 and the JWT library; neither is present. Standard JWT libraries (e.g. `jsonwebtoken`) enforce `exp` by default, but nothing has been installed or configured.
**Location:** `package.json` - no JWT library (`jsonwebtoken`, `jose`, etc.) added
**Suggestion:** Install a JWT library and ensure expiry checking is enabled (it is on by default in most libraries). Do not disable `ignoreExpiration`.

---

*No MEDIUM or LOW issues - all gaps are against explicit, core requirements.*

---

**Summary:** All 6 requirements are completely unimplemented. The branch contains only observability and dependency changes that are unrelated to the PIQ-512 ticket. This PR should not be merged as a delivery for this ticket.
