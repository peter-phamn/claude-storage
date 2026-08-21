---
name: pr-respond
description: Respond to PR review comments — fetch unresolved threads, triage into fixes vs replies, apply code fixes, and draft casual human replies for pushback/clarification. Use when the user says "respond to PR comments", "handle review feedback", "address PR reviews", "trả lời comment PR", "xử lý review", or pastes a PR link and wants to address reviewer feedback.
---

# PR Respond — Triage and Address Review Comments

Fetch unresolved PR review comments, decide which ones need code fixes vs. replies, fix the code, and draft replies — all in one pass.

## Input

Accepts a PR link or number:
- `https://github.com/owner/repo/pull/123`
- `#123` (uses current repo context)
- Just a number: `123`

## Process

### Step 1: Parse PR and fetch unresolved comments

Extract `owner`, `repo`, `pullNumber` from the input. If only a number is given, detect owner/repo from git remote.

```bash
git remote get-url origin
```

Fetch all review comment threads using `pull_request_read` with method `get_review_comments`. Filter to threads where `isResolved` is `false` and `isOutdated` is `false`.

### Step 2: Read the code and CLAUDE.md

For each comment thread, read the file and lines referenced. Also read the project's CLAUDE.md to understand conventions — some reviewer suggestions might conflict with project rules.

### Step 3: Triage each comment

For each unresolved thread, categorize it:

**FIX** — the reviewer is right, code needs changing:
- Genuine bug or logic error
- Valid style/convention issue backed by CLAUDE.md
- Missing edge case or error handling
- Legitimate simplification opportunity

**REPLY** — no code change needed, draft a response instead:
- Reviewer lacks context (explain the constraint)
- Suggestion would break something (explain what)
- Already handled elsewhere (point to where)
- Intentional decision (explain why)
- Nitpick or preference with no clear winner (acknowledge, explain choice)
- Suggestion conflicts with CLAUDE.md rules

**UNCLEAR** — can't determine without more info:
- Ambiguous suggestion
- Requires domain knowledge you don't have

### Step 4: Present the triage

Before doing anything, present the full triage to the user for approval:

```markdown
## PR #123 — Review Comment Triage

### Will Fix (X comments)

1. **file.ts:42** — @reviewer: "This should use early return"
   → Agree, restructuring to early return pattern.

2. **utils.ts:15** — @reviewer: "Missing null check"
   → Valid, adding guard clause.

### Will Reply (Y comments)

3. **file.ts:85** — @reviewer: "Why not use a simple if/else?"
   → Draft reply: "The switch handles future tag types without nested conditions — cleaner to extend."

4. **file.ts:120** — @reviewer: "This cast looks unsafe"
   → Draft reply: "`request` is still an object here — `serialiseNoteForDB` stringifies it later. The cast is safe at this point in the flow."

### Unclear (Z comments)

5. **file.ts:200** — @reviewer: "Consider refactoring this"
   → Need clarification — what specifically should change?
```

Wait for user confirmation before proceeding.

### Step 5: Apply fixes

For each FIX item:
1. Make the code change.
2. Post a short reply on the thread confirming what changed (see reply style guide) using `add_reply_to_pull_request_comment`.
3. Resolve the thread immediately using `pull_request_review_write` with method `resolve_thread` — no extra confirmation needed here, since the user already approved this item as FIX in the Step 4 triage.

After all fixes are applied, show a summary of what changed.

### Step 6: Post replies

For each REPLY item, post the reply using `add_reply_to_pull_request_comment` to the specific comment thread. Leave these threads **unresolved** by default — a pushback/clarification reply might get more discussion from the reviewer, so don't resolve unless the user explicitly says to.

For UNCLEAR items, post a clarifying question. Leave unresolved.

## Reply style guide

Replies represent the PR author talking to their reviewer. The tone should be like texting a coworker — friendly, direct, no fluff. The reviewer should read it and feel like a person casually typed this in ten seconds, not that a system generated a report.

**Principles:**
- Answer the actual question, don't dodge
- If pushing back, give the reason in one sentence, not a paragraph
- No performative gratitude ("Great catch!", "Thanks for pointing this out!")
- No corporate-speak ("I appreciate your thorough review")
- If they're right and you're fixing it, just say what you did — don't grovel
- If they're wrong, say so politely but clearly — don't hedge with "I think maybe perhaps"
- **Keep explanations as short as the reviewer needs to understand, nothing more.** If you did research to answer the comment (read a migration, traced a call path, tested a theory), give the conclusion and the one fact that backs it up — not a walkthrough of every step you took to get there. A reply that reads like an investigation writeup is a tell that it wasn't written by a person; a coworker replying in Slack states the answer, not their research process.
- One idea per reply. If there's a fix + a proof point (e.g. "fixed it" + "here's why it works"), that's fine in one or two sentences — but don't stack three separate justifications for the same decision.

**Good replies:**

Reviewer: "This cast looks unsafe"
> `request` is still an object here — `serialiseNoteForDB` stringifies it later. Safe at this point in the flow.

Reviewer: "Why not use a simple map instead of switch?"
> Switch is easier to extend when we add more note types — each case is self-contained. Map would work too but the switch reads better here imo.

Reviewer: "Missing error handling"
> Fixed — added a guard clause with `log.warn` for the malformed data case.

Reviewer: "This violates our no-casting rule"
> Yeah, forced by the type def — `Activity.Note.request` is typed as `string` but it's actually an object at this point. Would need a `just-my-type` update to fix properly. Cast is safe here though.

Reviewer: "Can you add a comment explaining this?"
> The code is fairly self-documenting with the named constant and method name. Prefer not to add one here — the why's already in the PR description.

Reviewer: "Do we have a test case for this no-op branch? Is that difficult to test?"
> Checked — `access_block.user_id` has a hard FK to `users.id`, so you can't force a real 0-affected-rows update there without orphaning the user first, which MySQL blocks. Stubbed `setUserStatus` to return `false` instead to hit the branch directly. Added the test, confirmed via coverage it's hit now.

**Bad replies:**

> Thanks for catching this! You're absolutely right, great eye! *(performative)*

> I respectfully disagree with this suggestion because upon careful consideration of the architectural implications... *(corporate-speak)*

> Done. *(too terse — what did you do?)*

> I think you might be right but I'm not sure, let me check... maybe we should... *(indecisive — either check first then reply, or state your position)*

> Checked — `access_block.user_id` has a hard FK to `users.id` (still there as of the Feb 2026 migration, which couldn't even drop an index because the FK still needed it), so there's no way to get a genuine 0-affected-rows update on `users` while the block still references that user — MySQL blocks deleting/orphaning the row first. So I stubbed `userModel.setUserStatus` to resolve `false` for one test — forces the exact branch without touching the DB, same idea as how this suite already stubs `NotificationSQS.sendEmail` for side effects. Added `'Should reject and leave the block untouched...'` — asserts `success:false` and that the block's `expires_at` is unchanged (proves the transaction actually rolled back, not just that it errored). Verified against the coverage report — lines 650/651/653/658 are now hit. *(technically all true, but this is a report, not a reply — say the conclusion and one supporting fact, trim the rest)*

## Edge cases

- **Bot comments** (e.g., from CI, Copilot): skip these, only process human reviewer comments
- **Resolved threads**: skip — only process unresolved
- **Outdated threads** (code has changed since comment): flag to user, suggest resolving without action
- **Multiple pages of comments**: paginate through all pages using the `after` cursor
