---
name: gemini-ui
description: >
  Build new React UI components, pages, or layouts from scratch.
  Triggers on: "create a component", "build a page", "design a form",
  "new UI for", "make a dashboard", "build a modal", "create a table".
---

## Instructions

This task involves Gemini. Before proceeding, ASK the user:

"This is a UI component task. How should I handle it?

1. **Sub-agent + Gemini** (recommended) — generates and integrates the
   component in isolated context. Best for full components.
2. **Direct Gemini call** — faster for a simple component skeleton.
3. **I'll handle it myself** — no Gemini, full control."

### If user picks option 1 (sub-agent):
- Spawn the **ui-builder** sub-agent
- Pass it the requirements and any reference component paths
- When it returns, present the summary (files created, component API)
- If user wants modifications, handle them yourself

### If user picks option 2 (direct MCP):
- Check existing components for project patterns
- Call mcp__gemini__ask-gemini with requirements
- Adapt output to project conventions
- Write files and verify with TypeScript compiler

### If user picks option 3 (self):
- Build the component yourself following project conventions
