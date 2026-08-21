---
name: pr-review-teammate
description: Review a teammate's PR as an external reviewer — pulls the ClickUp ticket linked in the PR description for requirements/comments, runs mattpocock-skills:code-review against it (target branch defaults to master), and outputs a numbered report of exact file:line + a casual, human-sounding reply comment to leave. Re-run on the same PR later to triage the author's replies to a prior review and produce a follow-up report in the same format. Use when the user pastes a PR link and wants it reviewed, wants review comments drafted for a colleague's branch, or wants to check how the author responded to a previous review pass.
---

# Review a teammate's PR

Produces a numbered list of `file:line` + reply, worded so the author can't tell it came from an AI. Two modes, auto-detected from the PR's existing comment threads.

## Step 1: Resolve the PR and its repo

Parse the PR link/number into `owner`, `repo`, `pullNumber`. Find the local clone at `/Users/thanhpham/Projects/kwiff/<repo>` — if it's not there, ask the user for the path rather than guessing or cloning.

Fetch the PR via `pull_request_read` (method `get`) — you need its description, base branch, and head branch.

Fetch every review comment thread via `pull_request_read` (method `get_review_comments`).

**Done when:** you have the PR body, head branch name, and the full thread list (each thread's comments with author + timestamps).

## Step 2: Pick the mode

- **No thread in the list has a comment authored by you (the reviewer)** → Initial Review mode (Step 3).
- **At least one thread has a comment authored by you** → Follow-up mode (Step 6), for every one of those threads — whether or not the author left a text reply. A thread counts as needing follow-up if either is true: the author posted a newer comment, or the code at that file/line has changed since your comment's commit (silent fix, no reply). Threads where neither happened (no reply, code untouched) are skipped — nothing to report yet.

If both kinds of threads exist, do both: run Step 6 for the reviewer threads, and Step 3 for any part of the diff never commented on yet. Merge into one numbered report at the end.

## Step 3: Get requirements and standards (Initial Review mode)

Search the PR body for a ClickUp link (`app.clickup.com/t/...` or `clickup.com/t/...`). Extract the task ID and pull it with `clickup_get_task`, plus `clickup_get_task_comments` for discussion context. This is the spec — treat it as source of truth over anything you infer from the diff.

No ClickUp link in the description → tell the user and ask for the ticket, don't guess the requirement.

Read the target repo's **own** `CLAUDE.md` (`/Users/thanhpham/Projects/kwiff/<repo>/CLAUDE.md`), not the kwiff-root one already in your context — sub-repos (zendesk, zendesk-copilot, lambda-template, etc.) carry their own conventions the root file doesn't auto-load.

**Done when:** you can state the ticket's requirement in your own words, and you know which standards file(s) apply to this repo.

## Step 4: Run the review

`cd` into the repo, fetch and check out the PR head branch (`gh pr checkout <pullNumber>`, or `git fetch origin pull/<pullNumber>/head:pr-<pullNumber>` then checkout). Target branch is `master` unless the user named a different one — if `master` doesn't exist, try `main`.

Invoke `Skill` with `mattpocock-skills:code-review`, args: the fixed point (target branch), and the ClickUp requirement text pasted in directly as the spec source (so it skips its own spec-hunting step). Let it run its own Standards + Spec sub-agents.

**Done when:** the code-review skill has returned both its Standards and Spec sections.

## Step 5: Turn findings into reply rows

For every finding you're going to act on (skip anything trivial-nitpick the standards sources says is tooling-enforced already, or Spec findings marked "cannot verify"):

1. Open the file at the reported location and confirm the **exact line number** against the current diff — don't reuse a line number code-review guessed, re-derive it from the file/hunk yourself.
2. Write the reply per the style guide below.

Go to Step 7 for output.

## Step 6: Triage the author's replies (Follow-up mode)

For every thread flagged in Step 2, first check what actually happened in the code — the reply text is a claim, the diff is the evidence:

`git diff <commit-the-comment-was-made-on>..HEAD -- <file>` (or `git log -p` scoped to that file/line range) to see whether the flagged lines changed, and if so, read the current version of that code in full — not just the diff hunk — to judge whether it actually resolves the original concern, the same way you'd verify any other finding.

Then classify:

- **Code changed and it actually fixes the concern** → confirm it, quoting or referencing what changed. Applies whether or not they left a reply — a silent fix still gets a one-line confirmation, since resolving the thread without saying why looks like you didn't check.
- **Code changed but doesn't fully fix it** (wrong spot, half the cases, introduces a new issue) → say specifically what's still wrong, pointing at the current code, not the old comment.
- **Code untouched, they pushed back with a reason** → decide if the reason holds up against the actual code. If it holds, concede in one line. If it doesn't, restate the concern with the specific thing their reason missed.
- **Code untouched, they asked a clarifying question** → answer it directly, citing the file/line that backs the answer.
- **Code untouched, they claimed "done" / "fixed"** → say it plainly, don't accept the claim.

**Done when:** every flagged thread has a next-turn reply drafted, and every claim of "fixed" (spoken or silent) has been checked against the actual current code, not assumed from the comment text or the fact that *something* in the file changed.

## Step 7: Output the report

Plain numbered list, one entry per comment, no table:

```
1. lib/foo.service.ts:42
convert those strings to enum

2. lib/bar.controller.ts:88
magic string here too

3. lib/baz.ts:120
as asked before:
is the lambda behind API GW?
if yes: we had some concerns about it in the past
if no: TO won't apply
```

Don't post anything to GitHub. Show the report and ask if the user wants these posted as review comments — only then use `pull_request_review_write` (`create` → `add_comment_to_pending_review` per row → `submit_pending`).

## Reply style guide

Every reply must read like a coworker typed it in ten seconds on the PR, not like a report.

- **No capital letter at the start of the sentence.** "convert those strings to enum", not "Convert those strings to enum".
- **No em dash (—).** It's the single biggest AI tell — use a comma, "and", or a line break instead.
- Short. One idea per reply. A fix-suggestion needs no justification paragraph.
- Questions read as questions a person would actually ask, not a directive: "does this differ by env, or can it just be a constant? i get you want to release with it off, but this could just be a constant flag in /constants, not an env-var (which comes with more overhead)".
- Multi-part context (e.g. referencing an earlier conversation) is fine across multiple lines, still no capitals, still no em dash:
  ```
  as asked before:
  is the lambda behind API GW?
  if yes: we had some concerns about it in the past
  if no: TO won't apply
  ```
- Rhetorical/pointed questions are fine when they land: "isn't this just !awpDetectedOverridableMismatch?"
- No performative gratitude, no corporate-speak, no hedging ("I think maybe perhaps").
- Cite the concrete thing (a constant name, a variable, an existing helper) instead of speaking abstractly.
