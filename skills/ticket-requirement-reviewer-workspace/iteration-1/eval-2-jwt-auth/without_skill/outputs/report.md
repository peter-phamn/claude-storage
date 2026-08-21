# Code Review: PIQ-512 - Add JWT Authentication to payment-iq API

**Branch:** env/piq-more-visibility
**Diff:** 19 files changed, 5095 insertions(+), 2452 deletions(-)

---

## Summary

The code changes on this branch do **not implement any** of the JWT authentication requirements from PIQ-512. All changes are unrelated to authentication.

---

## Requirement-by-Requirement Assessment

### Req 1 - All API endpoints must require a valid JWT token in the Authorization header
**Severity: HIGH - NOT IMPLEMENTED**

No middleware or route-level guards were added. `src/http/payment-iq.http.ts` was modified only to rename an interface (`RequestPromiseOptions` → `RequestArgs`) and fix data normalization. No Authorization header validation was introduced.

---

### Req 2 - JWT tokens should be validated against our shared secret (from env var JWT_SECRET)
**Severity: HIGH - NOT IMPLEMENTED**

- `JWT_SECRET` was not added to `terraform/lambda_env_vars.tf` (only a latency threshold was changed: 1000ms → 800ms).
- No JWT secret lookup or verification logic exists anywhere in the diff.

---

### Req 3 - Requests with missing or invalid tokens must return 401 Unauthorized
**Severity: HIGH - NOT IMPLEMENTED**

No 401 error handling was added anywhere. No HTTP error response logic for auth failures exists.

---

### Req 4 - The decoded JWT payload should be available to request handlers
**Severity: HIGH - NOT IMPLEMENTED**

No mechanism to attach a decoded JWT payload to the request context (e.g., `req.user`) was added.

---

### Req 5 - A whitelist of public endpoints (e.g. /health) should bypass authentication
**Severity: MEDIUM - NOT IMPLEMENTED**

No public endpoint whitelist or bypass logic exists. This is also moot since no auth middleware was added at all.

---

### Req 6 - Token expiry must be enforced
**Severity: HIGH - NOT IMPLEMENTED**

No JWT expiry validation logic was added. Critically, no JWT library (e.g., `jsonwebtoken`, `jose`) was added to `package.json` — only `external-communications` (1.0.1→1.0.7) and `logging` (1.0.1→1.1.3) were updated.

---

## What Was Actually Changed

| File | Change |
|------|--------|
| `bash/process.env.sh` | `INCLUDE_DIRS` checks if directories exist before scanning |
| `src/http/payment-iq.http.ts` | Interface rename + data normalization fix |
| `terraform/lambda_env_vars.tf` | Latency threshold changed 1000ms → 800ms |
| `package.json` | Dependency version bumps only (no JWT lib) |
| `tests/integration/setup/index.setup.ts` | Removed `automatedSafeNetConnect: true` |
| Various bash scripts | Version comment bumps only |

---

## Conclusion

**0 of 6 requirements are implemented.** This branch contains no JWT authentication code. All ticket requirements are completely unaddressed and this should not be merged as a delivery for PIQ-512.

**Recommended next steps:**
1. Add a JWT library (e.g., `jsonwebtoken`) to `package.json`
2. Add `JWT_SECRET` to Lambda environment variables in `terraform/lambda_env_vars.tf`
3. Create an auth middleware that validates `Authorization: Bearer <token>`
4. Enforce token expiry during validation
5. Attach the decoded payload to the request context
6. Implement a public endpoint whitelist (at minimum `/health`)
7. Return `401 Unauthorized` for missing or invalid tokens
8. Add unit and integration tests covering auth scenarios
