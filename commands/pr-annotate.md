---
name: pr-annotate
description: Analyze the current branch diff and post inline PR comments explaining non-obvious code decisions to reviewers. Use when the user says "annotate PR", "add review comments", "explain changes for reviewer", "pr-annotate", "comment trên PR", "giải thích code cho reviewer", or wants to add explanatory comments to a GitHub PR before requesting review.
---

# PR Annotate — Reviewer-Friendly Inline Comments

Generate and post GitHub PR inline comments that explain non-obvious code decisions to reviewers. The goal is to save reviewer time by preemptively answering "why did you do it this way?" questions.

## What to annotate

Not every line needs a comment. Only annotate lines where a reviewer would reasonably pause and wonder why. Specifically:

### Must annotate
- **Type casts / `as` assertions** — explain why the cast is necessary and why it's safe at runtime
- **Intentional CLAUDE.md / style guide violations** — explain the constraint that forced the violation
- **Defensive guards that look unnecessary** — explain what runtime shape the guard protects against
- **Business logic that isn't self-evident** — explain the domain rule driving the code
- **Workarounds for library/framework limitations** — explain what's broken and why this is the fix
- **Architecture decisions** — when there's an obvious simpler alternative, explain why you chose this path
- **Re-implementation of existing utility/library behavior** — if custom code does something a dependency already handles (retry logic, error handling, auth, caching), explain why the built-in behavior wasn't sufficient. Without a "why", reviewers assume the author didn't know the dependency existed.
- **Test placement decisions** — if test fixtures or helpers are inline instead of extracted to SDK helpers or a `helpers.ts`, explain why (new endpoint with no existing SDK, one-off test that doesn't need reuse, etc.). Reviewers should not have to ask "why not use the SDK?" or "should this be shared?".
- **Non-obvious placement of any code** — if a function, helper, or block of logic lives somewhere a reviewer might not expect, explain why it's there rather than elsewhere.

### Do not annotate
- Trivial changes (imports, formatting, renaming)
- Code that is self-explanatory from naming and context
- Changes that are obvious from the PR description alone
- Test scaffolding that follows existing SDK/helper patterns exactly — but if it *deviates* from those patterns, annotate the deviation

## Process

### Step 1: Determine diff scope

```bash
# Default: diff against master
git diff master...HEAD

# If user specifies a target branch:
git diff <target-branch>...HEAD
```

Also check if a PR already exists:
```bash
gh pr view --json number,url 2>/dev/null
```

If no PR exists, output the annotations as a markdown list instead (cannot post comments without a PR).

### Step 2: Read CLAUDE.md

Read the project's CLAUDE.md (and any parent CLAUDE.md files) to understand the coding conventions. This is critical — you need to know the rules to identify intentional violations.

### Step 3: Analyze each changed file

For each file in the diff:
1. Read the full file (not just the diff) to understand context
2. Identify lines in the diff that match the "must annotate" criteria
3. For each annotation, determine:
   - **file**: relative path from repo root
   - **line**: the line number in the new version of the file
   - **comment**: a concise explanation (1-3 sentences) written in English

### Step 4: Post as GitHub PR review

Use the GitHub MCP tools to post annotations as a single PR review:

1. **Create a pending review** using `pull_request_review_write` with method `create` (no `event` — keeps it pending)
2. **Add each annotation** using `add_comment_to_pending_review` with:
   - `path`: relative file path
   - `line`: line number in the new file
   - `side`: "RIGHT" (commenting on the new code)
   - `subjectType`: "LINE"
   - `body`: the comment text
3. **Submit the review** using `pull_request_review_write` with method `submit_pending` and event `COMMENT`

### Step 5: Summary

After posting, output a brief summary:
```
Posted X annotations on PR #N:
- file1.ts: Y annotations
- file2.ts: Z annotations
```

## Comment style guide

Write like a teammate leaving a quick note — casual, concise, but complete. The reader should walk away understanding the decision without needing to ask a follow-up. One or two sentences is the sweet spot. Don't over-explain what the code does (they can read it), but make sure the reasoning is clear enough that someone unfamiliar with the context won't be confused.

**Tone:** conversational, like a PR self-review comment. Not a docstring, not a one-word hint either.

**Good:**
> Cast needed — `request` is still an object here, `serialiseNoteForDB` stringifies it later.

> Warns when `request` exists but `tags` is missing/malformed, so we don't silently skip highlighting.

> `bulkCreate` is the only path that runs `highlightNotes()` — `create` skips it, so tests must use this entry point.

**Bad:**
> `note.request` is typed as `string` in `Activity.Note`, but at this point in the `bulkCreate` flow it hasn't been serialized yet — `serialiseNoteForDB()` calls `JSON.stringify` later. The cast is safe here. *(too long — reads like documentation, not a comment)*

> This line casts the type. *(too vague — doesn't explain WHY)*

> Changed this to use enum. *(obvious from the diff — adds no value)*

## Fallback: No PR exists

If no GitHub PR exists, output annotations as a markdown list:

```markdown
## PR Annotations

### `lib/messages/Notes.ts`

**Line 317** — Cast needed — `request` is still an object here, `serialiseNoteForDB` stringifies it later.

**Line 320-329** — Warns when `request` exists but `tags` is missing/malformed, so we don't silently skip highlighting.
```

## Arguments

The skill accepts an optional target branch argument:
- No argument: diff against `master`
- With argument: diff against the specified branch (e.g., `develop`, `main`)
