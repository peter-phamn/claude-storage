---
name: testing-note
description: Generates a testing note for a PR or ticket — focused only on technical edge cases and non-obvious scenarios that QA would miss without reading the code. Use this whenever the user asks to write a testing note, QA note, "hướng dẫn test", "viết testing note", "test note cho ticket/PR", wants ClickUp testing notes filled in, or is preparing a PR/ticket for QA handover — even if they don't say "testing note" explicitly.
---

# Testing Note Generator

A testing note exists to give QA **information they couldn't get by reading the ticket alone**. Standard happy-path and obvious negative cases are QA's job — don't list them. Only write what a tester would miss without the developer's knowledge of the implementation.

## What belongs in a testing note

Write a note only when at least one of the following is true:

- **Non-obvious trigger** — the behavior fires on a backend cron, a webhook, a specific payload variant, or a race condition — not a simple button press
- **Hidden edge case** — a specific input value, account state, or timing that the ticket doesn't describe but the code explicitly handles (e.g. empty string vs null, a fallback chain with two different sources)
- **Setup that requires dev knowledge** — specific DB state, feature flag, env var, or test-town trick needed before the case can be exercised
- **The negative case looks like the positive case** — something QA might mark as passing when it's actually wrong (e.g. a block that should stay but visually disappears for a second)
- **A fix that's invisible unless you know where to look** — a silent error path, a log-only outcome, a case that used to silently fall through to the wrong branch

If the change is purely a happy-path feature with no edge cases worth flagging, write: "No technical notes — standard happy-path testing sufficient."

## What does NOT belong

- Steps QA already knows: "log in as the user", "submit the form", "check it works"
- Restating the ticket's acceptance criteria
- A full scenario matrix — that's a test plan, not a note
- Anything obvious from the ticket description alone

## Process

### 1. Read the implementation

```bash
git diff <destination-branch>...HEAD
git log <destination-branch>..HEAD --oneline
```

Look specifically for:
- Branching logic with multiple conditions (what makes each branch fire?)
- Fallback chains (`??`, `||`, try/catch with a default)
- Guards on optional/nullable fields
- Cases the tests cover that the ticket doesn't mention

### 2. Filter ruthlessly

For each potential note, ask: **would an experienced QA tester think to test this without reading the diff?**

- Yes → leave it out
- No → include it

### 3. Write the note

Short, direct, developer-to-tester tone. No headers needed unless there are 3+ separate points. No numbered steps unless order matters.

Each point should be one of:
- A **specific thing to try** that isn't in the ticket ("try with txName in lowercase — it should still route as a reversal")
- A **specific thing to check** that's easy to miss ("the block should still appear — it's only the re-request email that stops, not the block itself")
- A **setup requirement** QA can't infer from the ticket ("needs `attributes.originTxId` to be absent at the top level but present inside `attributes` — the Truelayer reversal payload variant sends it that way")

Keep it to the point. If a tester needs to do something special to hit the case, say what. If they just need to know where to look, say that.

## Output format

Print directly in the response, ready to paste into the ticket's testing-notes field. No file needed.

If there's nothing technical to flag, say so in one line.

## Example

Ticket: *"Truelayer reversal should not check deposit limits."* The diff changes the reversal detection from `transactionType === DEPOSIT && txName === 'Reversal'` to `txName.toLowerCase() === 'reversal'` checked before the deposit/withdrawal branch.

What QA would naturally test: reversal moves the withdrawal to Approval Failed. ✓ They'll test that.

What they'd miss:

```
Two edge cases worth specifically testing:

1. **Reversal with a negative txAmount** — Truelayer sometimes sends the reversal with a negative amount (money going back). The old code treated this as a withdrawal and settled it as COMPLETED instead of FAILED. If you can trigger a reversal with a negative amount on the env, confirm it still lands in Approval Failed.

2. **Customer with an active deposit limit** — the original bug. Find a customer with a deposit limit set and trigger the reversal flow for them. Before this fix they'd hit the limit check and the reversal would fail. Should now go through to Approval Failed without touching the limit.
```
