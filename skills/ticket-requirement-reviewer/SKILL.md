---
name: ticket-requirement-reviewer
description: Reviews code changes (git diff of current branch) against ticket requirements to verify implementation completeness. Use whenever the user wants to check if their PR/code changes satisfy a ticket's requirements, acceptance criteria, or user story — even if they just say "check if my code matches the ticket", "review against requirements", "does my PR cover the ticket?", "kiểm tra code có đáp ứng yêu cầu ticket không", or similar. Issues are categorized as HIGH, MEDIUM, or LOW severity.
---

# Requirement Verification

Verify whether the implementation satisfies a set of user-provided requirements.

This skill is **not a code review**. It does not evaluate code quality, coding style, architecture, design patterns, refactoring quality, performance, or best practices. Its sole responsibility is to determine whether the implementation matches the requirements supplied by the user.

## Inputs

### Requirements

A list of requirements provided by the user when invoking this skill. The provided requirements are the source of truth.

### Destination Branch

Default: `master`

Can be overridden by the user. If `master` doesn't exist, try `main`.

## Verification Process

### Step 1: Understand the Requirements

Read all provided requirements carefully. Break them down into individual verification items. Do not infer additional requirements unless they are necessary to satisfy an explicitly stated requirement.

### Step 2: Analyze the Implementation

Compare the current branch against the destination branch:

```bash
git diff master...HEAD
git diff master...HEAD --stat
git log master..HEAD --oneline
```

Review all relevant code changes introduced by the implementation. Focus on behavior and functionality rather than implementation details.

### Step 3: Verify Requirement Coverage

For each requirement, determine one of the following statuses:

- **Satisfied** — code clearly implements this requirement
- **Partially Satisfied** — code addresses it but incompletely (missing edge cases, incomplete logic)
- **Not Satisfied** — no evidence this requirement is handled
- **Cannot Verify** — the requirement is about runtime behavior, UI appearance, or external integrations that can't be verified from a diff alone

Every conclusion must be supported by evidence from the implementation. Do not assume a requirement is satisfied without evidence.

### Step 4: Identify Missing Functionality

Look for requirements that were not implemented, were only partially implemented, or cannot be verified from the changes. List each missing or incomplete requirement separately.

### Step 5: Identify Out-of-Scope Functionality

Look for functionality that appears unrelated to the provided requirements:

- Additional features not requested
- Behavioral changes not requested
- Unrelated business logic changes
- Scope expansion beyond the requirements

Do not flag implementation details that are necessary to support a required feature. Only report meaningful scope deviations.

### Step 6: Produce the Report

Use this exact output format:

---

### Requirements Assessment

| Requirement | Status | Evidence |
| ----------- | ------ | -------- |
| [requirement text] | ✅ Satisfied / ⚠️ Partially Satisfied / ❌ Not Satisfied / 🔍 Cannot Verify | [specific file, function, or code path] |

### Missing Requirements

* None

OR

* [Requirement X — what is missing and why]
* [Requirement Y — what is missing and why]

### Out-of-Scope Functionality

* None

OR

* [Description of additional functionality not in the requirements]

### Final Verdict

**PASS** — All requirements are satisfied. No meaningful missing or out-of-scope functionality detected.

**PASS WITH CONCERNS** — All requirements are satisfied. However, significant out-of-scope functionality was introduced.

**FAIL** — One or more requirements are not satisfied, or one or more requirements cannot be verified.

### Confidence

**High** / **Medium** / **Low**

[Brief explanation of why this confidence level was chosen — e.g., "Low: several requirements depend on runtime behavior that cannot be confirmed from the diff alone"]

---

## Critical Rules

- Do not perform a code review.
- Do not comment on code quality.
- Do not suggest refactoring.
- Do not judge architecture.
- Do not evaluate coding style.
- Do not recommend optimizations unless they directly affect requirement compliance.

Your job is not to determine whether the implementation is good. Your job is to determine whether the implementation matches the provided requirements.

Be skeptical. Require evidence for every conclusion. Never assume a requirement is satisfied without verification.
