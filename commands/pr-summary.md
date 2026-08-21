---
name: pr-summary
description: Generate a concise PR summary from the current diff
---

Analyze the current branch diff against the target branch.

Generate a concise GitHub PR summary that includes:
- Business goal
- High-level implementation summary
- Affected areas
- Risks or reviewer focus areas

Focus on WHY and IMPACT, not just WHAT changed.

Keep it short, clear, and reviewer-friendly.

Write the final output to:
tmp/pr-summary.md

Return only:
Generated tmp/pr-summary.md