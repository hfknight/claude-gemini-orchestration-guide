---
name: gemini-review
description: >
  Deep code review for bugs, security, performance, and React best practices.
  Triggers on: "review this", "code review", "audit code", "check for bugs",
  "is this code good", "review my PR", "review staged changes".
---

## Instructions

This task involves Gemini. Before proceeding, ASK the user:

"This is a code review task. How should I handle it?

1. **Sub-agent + Gemini** (recommended) — deep analysis in isolated context.
   Best for reviewing full files or PRs.
2. **Direct Gemini call** — faster for a quick check on a single function.
3. **I'll handle it myself** — no Gemini, I review everything directly."

### If user picks option 1 (sub-agent):
- Spawn the **code-reviewer** sub-agent
- Pass it the scope: staged changes, specific file, or recent commits
- When it returns, present the prioritized findings
- If user wants fixes applied, handle them yourself (you have the file context)

### If user picks option 2 (direct MCP):
- Gather the diff or file content yourself
- Call mcp__gemini__ask-gemini with the code and review prompt
- Synthesize findings and present to user

### If user picks option 3 (self):
- Review the code yourself without Gemini
- Follow the same categories: bugs, TypeScript, React patterns, security,
  performance
