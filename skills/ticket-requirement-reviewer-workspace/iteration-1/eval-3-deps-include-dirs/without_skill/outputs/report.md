# Ticket Review: PIQ-490 - Update dependencies and fix INCLUDE_DIRS initialization

**Branch:** env/piq-more-visibility
**Overall verdict:** PARTIALLY SATISFIES — 3 of 4 acceptance criteria met, with 1 critical gap and 1 version discrepancy.

---

## Acceptance Criteria Review

### AC1: INCLUDE_DIRS should be properly initialized even if some directories are missing
**PASS**

`bash/process.env.sh` now builds an `EXISTING_DIRS` array by checking each directory with `[ -d "$dir" ]` before including it. The grep only runs on directories that actually exist. This correctly addresses the fragile initialization issue.

`bash/side-effect-imports.sh` also adds `2>/dev/null` to suppress errors when directories don't exist, providing additional robustness.

---

### AC2: Update external-communications to version 1.0.2
**PARTIAL / VERSION MISMATCH (MEDIUM severity)**

`package.json` shows `@gamingfactory/external-communications` updated from `1.0.1` to `1.0.7`, not `1.0.2` as the ticket specifies. The package was updated, but to a higher version. This may be intentional (taking the latest stable release), but it deviates from the explicit ticket requirement and should be confirmed with the ticket author.

---

### AC3: Any changes to dependency versions must be reflected in package.json and package-lock.json
**PASS**

Both `package.json` and `package-lock.json` were updated. The `package.json` reflects all new dependency versions (`external-communications` 1.0.7, `logging` 1.1.3, `just-my-type` 2.1.47) and the lock file was regenerated to match.

---

### AC4: Initialization errors should be logged rather than silently ignored
**FAIL (HIGH severity)**

The implementation silently skips missing directories — it does not log any warning or error when a directory in `INCLUDE_DIRS` is found to be absent. The ticket explicitly requires that errors are "logged rather than silently ignored."

There is no `echo`, log output, or any other notification added when a directory is missing. The current behavior is: silently skip. The required behavior is: log the skip/error.

A minimal fix would be:
```sh
if [ -d "$dir" ]; then
  EXISTING_DIRS+=("$dir")
else
  echo "Warning: directory '$dir' not found in INCLUDE_DIRS, skipping." >&2
fi
```

This criterion is not met.

---

## Additional Observations

- `src/http/payment-iq.http.ts` includes interface renames and response data normalization that are out of scope for this ticket.
- `terraform/lambda_env_vars.tf` has latency threshold changes, also unrelated to this ticket.
- `@gamingfactory/logging` was bumped `1.0.1` → `1.1.3` — not mentioned in the ticket but may be a related improvement.

---

## Issues Summary

| Criterion | Severity | Finding |
|-----------|----------|---------|
| AC4: Log initialization errors | HIGH | Missing directories are silently skipped with no logging. Ticket explicitly requires logging. |
| AC2: Version 1.0.2 | MEDIUM | `external-communications` updated to `1.0.7` instead of ticket-specified `1.0.2`. Needs confirmation. |

---

**Conclusion:** AC1 (robust initialization) and AC3 (package.json + lock file in sync) are correctly implemented. AC4 (logging initialization errors) is missing entirely — directories are skipped without any log output, which directly contradicts the ticket requirement. AC2 has a version mismatch that needs sign-off. The PR should not be merged without resolving AC4 at minimum.
