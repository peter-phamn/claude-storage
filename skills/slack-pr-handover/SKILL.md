---
name: slack-pr-handover
description: Generate a filled Slack PR handover report from the current git branch and diff. Trigger this skill whenever the user asks to generate a PR handover, deploy report, release note for Slack, or handover template. Also trigger when user says "tạo handover", "điền report slack", "generate handover", "PR report cho slack", or anything about filling out a handover template for a PR.
---

# Slack PR Handover Report

Generate a fully filled handover report for the current branch, ready to paste into Slack.

## Steps

### 1. Gather git context

Run these commands in parallel:

```bash
git branch --show-current
git log master..HEAD --oneline
git diff master...HEAD --stat
git diff master...HEAD --name-only
```

### 2. Analyse the changes

From the data above, determine:

**Branch / Env**
- The branch name is the `env/*` part of the Env line. Format it as-is.

**Description**
- Summarise the purpose of the PR in 2-4 sentences based on commit messages and file names.
- Focus on the *why* and *business impact*, not the list of files.

**Areas affected**
- Infer from changed file paths:
  - Files under `src/cancel`, `src/deposit`, `src/withdraw`, `src/transfer`, `src/manual-withdraw`, `src/approve-withdrawal`, `src/deny-withdrawal`, `src/payment`, `src/http/payment` → **Payments**
  - Files under `src/user`, `src/verifyuser`, `src/user-apms` → **User/Compliance**
  - Files under `src/sport`, `sportsbook` → **Sportsbook**
  - Files under `src/casino` → **Casino**
  - Keep only the areas that have matching files. If uncertain, include the area and note it.

**Scheduling / Downtime**
- Check for migration files (e.g. paths containing `migration`, `migrate`, `db/`, `.sql`).
- Check for Terraform changes (`terraform/`, `*.tf`) — these may require infra deploys.
- Check for changes to environment variables (`lambda_env_vars`, `.env`) — may need coordinated deploy.
- If any of these are present, summarise the risk. Otherwise: "No migration or downtime expected."

**Database concerns** (answer Yes/No for each)
- >10,000 rows affected: only Yes if there's a migration touching a large table (flag if unsure)
- Triggers check: Yes if a new column is added to an existing table
- Altering existing column: Yes if a column is renamed or its type changed
- Removing existing column: Yes if a column is dropped

**Complexity score (1–5)**
- 1 — trivial: config/text change, <50 lines
- 2 — small: <200 lines, 1–5 files, no migrations
- 3 — medium: <500 lines, 5–15 files, or 1 migration
- 4 — large: <1000 lines, 15–25 files, multiple services, or complex migration
- 5 — major: 1000+ lines, 25+ files, multiple migrations, cross-service impact

Use the stat output (`insertions + deletions` total, file count) to calibrate.

### 3. Write the report to a file

Write the filled report to `tmp/handover.txt` in the current working directory (create `tmp/` if it doesn't exist).

Use `[TBD]` for any field you cannot determine from the git context.

**Formatting rules — Slack style:**
- Use Slack emoji codes (`:ice_cube:`, etc.) as-is
- Use `*bold*` for section labels
- Use bullet points (`•`) for lists; indent nested bullets with two spaces
- Use backticks for inline code/values (e.g. `terraform/lambda_env_vars.tf`)
- Keep each section short — 1–3 lines max; no long paragraphs
- No markdown headers (`#`) — Slack does not render them

The file content should follow this template exactly:

```
*Env:* :ice_cube:
{branch-name}

*Description:*
{2-4 short sentences or a tight bullet list}

*Release doc (backend only):*
[TBD]

*Test run:*
BE only

*BE > FE Implementation Notes:*
N/A

*Areas affected:*
• {area 1}
• {area 2 if applicable}

*Scheduling/Downtime:*
• {one bullet per risk item, or "No migration or downtime expected."}

*Data team:*
• Do the data team need to be aware? {Yes/No}
• Will the migration insert, update or delete more than 10,000 rows? {Yes/No}
• Have you checked for triggers on tables with new columns? {Yes/No/N/A}
• Are we altering an existing column? {Yes/No}
• Are we removing an existing column? {Yes/No}

*Complexity Score:* {N}/5
• {one sentence justification}
```

### 4. Tell the user

After writing the file, tell the user:
- The path to `tmp/handover.txt`
- A list of any `[TBD]` fields they still need to fill in
