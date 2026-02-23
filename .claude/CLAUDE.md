## Multi-AI Delegation Strategy

I have a Gemini MCP server connected (gemini-mcp-tool) with these tools:
- mcp__gemini__ask-gemini: Main prompt tool (gemini-2.5-pro, supports @file, sandbox, changeMode)
- mcp__gemini__brainstorm: Creative ideation with frameworks
- mcp__gemini__fetch-chunk: Retrieve paginated chunks from large responses

I also have three sub-agents:
- test-writer: Generates tests via Gemini in isolated context
- code-reviewer: Performs code review via Gemini in isolated context
- ui-builder: Builds UI components via Gemini in isolated context

### CRITICAL RULE: Always confirm execution mode before calling Gemini.

When a task triggers one of my Gemini-related skills (gemini-tests,
gemini-review, gemini-ui), I MUST ask the user which execution mode to use
before doing any work:

1. **Sub-agent + Gemini** — recommended for tasks with large outputs.
   Keeps main context clean. Use test-writer, code-reviewer, or ui-builder.
2. **Direct Gemini MCP call** — for quick/small tasks where speed matters
   more than context cleanliness. Call mcp__gemini__ask-gemini directly.
3. **Handle myself** — no Gemini. For when the user wants full control or
   the task is too small to justify delegation.

NEVER auto-spawn a sub-agent or auto-call Gemini without asking first.

**Exception**: For brainstorming tasks, always use mcp__gemini__brainstorm
directly without asking (output is small and conversational).

### When to recommend each option:

**Recommend sub-agent (option 1) when:**
- Generating a full test suite (many tests, high output volume)
- Reviewing an entire file or multi-file PR
- Building a new component from scratch
- Analyzing a large codebase or file (300+ lines)

**Recommend direct MCP (option 2) when:**
- Quick question: "how do I do X in React?"
- Small generation: a single test, one regex, one utility function
- Brainstorming: "what approaches exist for X?"
- Getting a second opinion on a small code snippet

**Recommend self (option 3) when:**
- Small edits under ~30 lines
- Fixing a few failing tests
- Multi-file refactoring (needs full project context)
- Git operations, build commands, running scripts
- Architecture decisions requiring cross-file understanding
- Integrating or modifying sub-agent output after the fact

### HOW I DELEGATE:
1. Ask the user which mode to use
2. If sub-agent: spawn the appropriate agent, pass it the task, relay its summary
3. If direct MCP: read the code myself, call mcp__gemini__ask-gemini, review output, apply it
4. If self: do everything without Gemini
5. Always verify results (run tests, type check, etc.)

### IMPORTANT RULES:
- Never send Gemini file paths — it cannot read my filesystem. Always paste
  the actual source code into the mcp__gemini__ask-gemini prompt.
- Always verify Gemini's output against the real component API before writing.
- If Gemini's output needs small fixes (< 10 lines), fix them myself rather
  than re-delegating. One round trip is almost always enough.
- For large Gemini responses that come back in chunks, use mcp__gemini__fetch-chunk
  to retrieve all parts before reviewing.
- Use /clear between unrelated tasks to keep context lean.

### Gemini fallback rule:
If mcp__gemini__ask-gemini returns a quota or rate limit error, inform
the user and ask user if handle the task myself (option 3). Do NOT retry
Gemini repeatedly — it wastes time and context
