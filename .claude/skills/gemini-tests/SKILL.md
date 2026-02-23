---
name: gemini-tests
description: >
  Write unit tests for React TypeScript components. Triggers on: "write tests",
  "add tests", "test coverage", "create spec", "need tests for".
---

## Instructions

This task involves Gemini. Before proceeding, ASK the user:

"This is a test writing task. How should I handle it?

1. **Sub-agent + Gemini** (recommended) — runs in isolated context, keeps
   our conversation clean. Best for full test suites.
2. **Direct Gemini call** — faster, but the generated tests land in our
   conversation. Best for a single small test.
3. **I'll handle it myself** — no Gemini, I write everything."

### If user picks option 1 (sub-agent):
- Spawn the **test-writer** sub-agent
- Pass it the file path(s) to test
- Let it handle the full lifecycle:
  read → call mcp__gemini__ask-gemini → review → write → run → fix
- When it returns, relay the summary to the user
- Only re-engage yourself if the user wants further changes

### If user picks option 2 (direct MCP):
- Read the source file yourself
- Call mcp__gemini__ask-gemini with the source code and test generation prompt
- Review the output for correctness
- Write the test file and run it
- Fix any failures

### If user picks option 3 (self):
- Write the tests yourself without Gemini
- Follow the same conventions: Jest + RTL + TypeScript + AAA pattern
