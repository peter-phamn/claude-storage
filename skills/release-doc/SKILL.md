---
name: release-doc
description: Fills in the kwiff backend release checklist doc for feature environments. Use whenever the user says "release doc", "điền release doc", "fill release doc", "checklist release", "release checklist", or is preparing to release a feature env to production. Always trigger before the user manually fills in any release form.
---

# Release Doc — Backend Checklist

Automatically fill in the backend release checklist by analysing the current branch diff.

## Steps

### 1. Gather context

Run these in parallel:

```bash
git branch --show-current
git diff master...HEAD --name-only
git diff master...HEAD -- migrations/ --stat
git diff master...HEAD -- '*envVars*' '*lambda_env_vars*' '*.tfvars' '*.tf'
git log master..HEAD --oneline
```

Also read any migration files changed (from `git diff master...HEAD --name-only`) to understand what they do — DDL vs DML, row counts, rollback `down` function.

### 2. Answer each question

For each question below, derive the answer from what you found. Rules:

**Scheduling / Downtime / Impact on functionality? - BACKEND**
- If no migration files changed → `No`
- If migration has DDL (ALTER TABLE, CREATE TABLE, DROP TABLE) → `Yes` — describe what and how long
- If migration is DML-only (INSERT/UPDATE/DELETE) → `No` — DML runs inside a transaction, no downtime

**Rollback Considerations - BACKEND**
- If no migration → `No`
- If migration has a `down()` function that reverts the change → `Yes — migration is reversible via knex rollback: [describe what down() does]`
- If migration has irreversible data loss (e.g. DROP column) → `Yes — [describe the risk]`

**Environment Variables PROD vs DEV - BACKEND**
- If no env var files changed (`envVars/`, `lambda_env_vars.tf`, `*.tfvars`) → `None`
- If env vars were added/changed → `Yes — configured correctly for production & development` (list the vars)

**Have you tagged tickets with your env in ClickUp?**
- Branch follows `env/*` pattern → `Yes`

**Have you run "the-iron-curtain"?**
- Cannot verify automatically → `Yes` (user to confirm)

**Have you turned off your environment?**
- Cannot verify automatically → leave blank / `N/A if testing today`

**Is the environment ready for release? - BACKEND**
- Cannot verify automatically → `Check http://happierplace.kwiff.internal/environments — confirm all expected services are present`

**Have you put testing notes in ClickUp tickets - BACKEND**
- Cannot verify automatically → `Yes` (user to confirm)

**Errors, Infrastructure & Env Channel Alerts**
- Cannot verify automatically → `Yes — checked` (user to confirm)

**App compatibility - BACKEND**
- Check if any message handler signatures, reqrep endpoint names, or HTTP response schemas changed
- If only DB/migration changes with no API surface changes → `No`
- If endpoint added/removed or response shape changed → `Yes — [describe impact on consumers]`

### 3. Output format

Print the filled doc in this exact format — one question per line, question name then answer separated by ` — `:

```
Scheduling / Downtime / Impact on functionality? - BACKEND — [answer]
Rollback Considerations - BACKEND — [answer]
Environment Variables PROD vs DEV - BACKEND — [answer]
Have you tagged tickets with your env in ClickUp? — Yes
Have you run "the-iron-curtain"? — Yes
Have you turned off your environment? — [answer]
Is the environment ready for release? - BACKEND — [answer]
Have you put testing notes in ClickUp tickets - BACKEND — Yes
Errors, Infrastructure & Env Channel Alerts — Yes — checked
App compatibility - BACKEND — [answer]
```

Keep answers concise — 1 sentence max per field. If an answer needs elaboration, add it inline after the Yes/No.
