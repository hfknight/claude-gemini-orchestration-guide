---
name: gemini-brainstorm
description: >
  Explore approaches, compare patterns, or generate ideas for architecture
  and design decisions. Triggers on: "brainstorm", "what approaches",
  "how could I", "what are my options", "explore ideas for".
---

## Instructions

This task benefits from Gemini's brainstorming capabilities.
Use the mcp__gemini__brainstorm tool directly (no sub-agent needed —
brainstorm outputs are typically small enough for the main context).

Do NOT ask the user which mode to use for brainstorming.
Always use direct MCP call since the output is conversational, not code.

1. Call mcp__gemini__brainstorm with the topic and any relevant context
2. Present the ideas to the user
3. If the user picks an approach, proceed with implementation yourself
   or delegate to the appropriate sub-agent
