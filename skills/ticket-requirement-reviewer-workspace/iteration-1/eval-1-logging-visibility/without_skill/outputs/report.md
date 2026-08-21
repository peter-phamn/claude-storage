# PR Review: PIQ-445 - Improve logging visibility for payment requests

**Branch:** `env/piq-more-visibility`
**Diff stat:** 19 files changed, 5095 insertions(+), 2452 deletions(-)

---

## Summary

The PR does **not** adequately satisfy the ticket requirements. The changes are primarily dependency upgrades, infrastructure threshold adjustments, and bash script fixes — none of which directly implement the logging improvements described in the ticket.

---

## Acceptance Criteria Review

### 1. Log the full request payload (method, path, headers, body) for every incoming HTTP request
**Status: NOT MET**

No changes in the diff add request payload logging. The rename from `RequestPromiseOptions` to `RequestArgs` in `src/http/payment-iq.http.ts` is a refactor with a data normalization fix, not a logging addition. There is no evidence of middleware or interceptors that log method, path, headers, or body for incoming HTTP requests.

---

### 2. Log response status and latency for every request
**Status: NOT MET**

No code changes log response status codes or response latency. The only latency-related change is in Terraform (`EXTERNAL_HTTP_LATENCY_WARNING_THRESHOLD_MS` lowered from 1000ms to 800ms), which adjusts an existing threshold but does not add new latency logging.

---

### 3. Include correlation ID / trace ID in all log entries
**Status: NOT MET**

No changes introduce correlation ID or trace ID propagation or injection into log entries.

---

### 4. Log errors with full stack traces
**Status: NOT MET**

No error handling or logging changes are present in the diff that would add or improve stack trace logging.

---

### 5. Latency logs should include breakdown (e.g. db time, external call time)
**Status: NOT MET**

No latency breakdown logging is introduced. The Terraform threshold change only modifies a warning threshold value, not the logging implementation itself.

---

### 6. All log fields should use consistent snake_case naming
**Status: CANNOT VERIFY**

The logging library was bumped (`@gamingfactory/logging`: `1.0.1` → `1.1.3`) and the external-communications library was bumped (`@gamingfactory/external-communications`: `1.0.1` → `1.0.7`). It is possible these upstream upgrades contain the actual logging improvements, but since the library source code is not visible in the diff, compliance with this criterion cannot be confirmed.

---

## Other Changes (Not Related to Ticket Requirements)

| File | Change | Ticket Relevance |
|------|--------|-----------------|
| `bash/process.env.sh` | Fixed INCLUDE_DIRS to only grep existing directories | None |
| `bash/new-code-checks.sh` | Added `setTimeout` detection in test files | None |
| `src/http/payment-iq.http.ts` | Renamed interface, normalized data to object | None (minor safety fix) |
| `terraform/lambda_env_vars.tf` | Lowered latency warning threshold 1000ms → 800ms | Tangential |
| Various bash scripts | Version comment bumps | None |
| `tests/integration/setup/index.setup.ts` | Removed `automatedSafeNetConnect: true` | None |

---

## Overall Assessment

**Requirements met: 0 out of 6** (1 unverifiable due to opaque dependency updates)

The PR branch name and dependency bumps to `@gamingfactory/logging` and `@gamingfactory/external-communications` suggest that the actual logging improvements may be encapsulated in those library upgrades. However, based solely on the visible diff, none of the six acceptance criteria are demonstrably implemented in the application code of this repository.

**Recommended actions before merging:**

1. Inspect the changelogs/diffs of `@gamingfactory/logging@1.1.3` and `@gamingfactory/external-communications@1.0.7` to verify whether they implement criteria 1–6.
2. If the libraries are the primary delivery vehicle for these requirements, the PR description should document this explicitly and reference the library PRs/changelogs.
3. If the libraries do not cover all criteria, additional application-level code (middleware, interceptors, error handlers) must be added before this ticket can be considered complete.
