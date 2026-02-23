# Claude Code + Gemini: Multi-AI Orchestration Setup Guide

A step-by-step guide to set up Claude Code as an orchestrator that delegates
token-heavy tasks (test writing, code review, UI generation) to Gemini via
MCP, using sub-agents to keep your main context window clean.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Layer 1: Install the Gemini MCP Server](#layer-1-install-the-gemini-mcp-server)
4. [Layer 2: Create Sub-Agents](#layer-2-create-sub-agents)
5. [Layer 3: Create Skills for Task Routing](#layer-3-create-skills-for-task-routing)
6. [Layer 4: Configure CLAUDE.md Routing Rules](#layer-4-configure-claudemd-routing-rules)
7. [Real-World Usage Examples](#real-world-usage-examples)
8. [Quick Reference](#quick-reference)
9. [Troubleshooting](#troubleshooting)
10. [Cost-Saving Tips](#cost-saving-tips)

---

## Architecture Overview

```
You (developer)
  │
  ▼
Claude Code (orchestrator — conserve tokens here)
  │
  ├─── Asks: "Sub-agent, direct MCP, or handle myself?"
  │
  ├─── Option 1: Sub-Agent + Gemini (recommended for large outputs)
  │     │
  │     ▼
  │     Sub-Agent (isolated context — disposable after task)
  │       │
  │       ├── Reads source files
  │       ├── Calls Gemini via mcp__gemini__ask-gemini ──► Gemini generates (free tokens)
  │       ├── Reviews Gemini's output
  │       ├── Writes files, runs tests
  │       └── Returns summary to main context ──► Main context stays clean
  │
  ├─── Option 2: Direct Gemini MCP call (for small/quick tasks)
  │     │
  │     └── Calls mcp__gemini__ask-gemini — response lands in main context
  │
  └─── Option 3: Claude handles it (for small edits, refactoring, git ops)
```

### Why sub-agents matter

| Approach | Main context cost | Best for |
|----------|------------------|----------|
| Claude does everything | 100% | Small edits, refactoring |
| Direct Gemini MCP call | ~50-60% savings | Quick questions, small outputs |
| Sub-agent + Gemini | ~80-90% savings | Test suites, reviews, new components |

Without sub-agents, Gemini's 200-line response dumps into your main context.
Claude still *reads* all of it (costing tokens) and it accelerates compaction.
Sub-agents do all the heavy lifting in an isolated context that gets discarded
when the task is done. Your main conversation only sees the summary.

---

## Prerequisites

- **Claude Code CLI** installed and authenticated
- **Node.js 18+** installed
- **Gemini API key** — free at [Google AI Studio](https://aistudio.google.com/apikey)

---

## Layer 1: Install the Gemini MCP Server

This gives Claude Code (and its sub-agents) the ability to call Gemini.

### Step 1.1 — Install Gemini CLI (optional fallback)

```bash
npm install -g @google/gemini-cli
gemini  # follow the auth prompts
```

### Step 1.2 — Add the Gemini MCP server to Claude Code

```bash
claude mcp add --env GEMINI_API_KEY=your_key_here --env GEMINI_MODEL=gemini-2.5-flash -s user gemini -- npx -y gemini-mcp-tool
```

> **Important**:
> - Run this as a **single line** — backslash line breaks can cause parsing errors.
> - The `--env` flags must come *before* the server name `gemini`.
> - The `--` separates Claude Code flags from the command that runs the server.
> - We use **gemini-2.5-flash** instead of the default Pro model because the
>   free tier quota for Pro is extremely low and you'll hit daily limits fast.
>   Flash is still very capable for test writing, code review, and UI generation.
>   If you have a paid API key, you can switch to `gemini-2.5-pro` for higher quality.

### Step 1.3 — Verify

```bash
claude
# Inside Claude Code:
> /mcp
# You should see "gemini · ✔ connected"
```

Quick test:

```
> Ask gemini: what is the best way to structure a React context provider?
```

If Claude calls `mcp__gemini__ask-gemini` and returns a response, Layer 1 is done.

### Available Gemini MCP tools

After setup you'll have these tools:

| Tool | Purpose |
|------|---------|
| `mcp__gemini__ask-gemini` | Main tool — send any prompt to Gemini (configured: gemini-2.5-flash). Supports @file syntax for including files, sandbox mode for safe code execution, and changeMode for structured edit suggestions. |
| `mcp__gemini__brainstorm` | Creative ideation with frameworks (SCAMPER, Design Thinking, etc.), domain context, feasibility analysis, and iterative refinement. |
| `mcp__gemini__fetch-chunk` | Retrieves paginated chunks from large changeMode responses that were split across multiple parts. |
| `mcp__gemini__ping` | Connectivity test. |
| `mcp__gemini__Help` | Returns help info about the server. |

For our workflow, you'll primarily use **`ask-gemini`** for test writing, code review, and UI generation, and **`brainstorm`** for exploring approaches.

---

## Layer 2: Create Sub-Agents

Sub-agents are isolated Claude instances with their own context window.
They do the heavy work and return only a summary to your main conversation.

### Step 2.1 — Open the agent creator

Inside Claude Code, run:

```
/agents
```

This opens an interactive UI for creating sub-agents.

### Step 2.2 — Create the Test Writer agent

```
Name:         test-writer
Description:  Writes unit tests by delegating generation to Gemini.
              Handles the full lifecycle: read source → Gemini generates
              tests → review → write file → run → fix failures.

System prompt:

  You are a test writing specialist working on a React TypeScript project.
  You have access to Gemini via the mcp__gemini__ask-gemini tool.

  ## Your workflow

  1. Read the source file provided to understand the component/function.
  2. Identify what needs testing:
     - Props and their variations
     - User interactions (clicks, inputs, form submissions)
     - Conditional rendering logic
     - Side effects (useEffect, API calls)
     - Error states and edge cases
  3. Call the mcp__gemini__ask-gemini tool with this prompt:

     "Write comprehensive unit tests for the following React TypeScript
     component.

     Tech stack:
     - Jest + React Testing Library
     - TypeScript strict mode
     - Ant Design v6 components (mock where needed)
     - Redux store (use renderWithProviders if available)

     Requirements:
     - describe/it blocks with clear test names
     - AAA pattern (Arrange, Act, Assert)
     - Mock external API calls with jest.fn()
     - Test all conditional rendering branches
     - Test user interactions with userEvent
     - Test error states and edge cases
     - Use accessible queries: screen.getByRole, screen.getByText
     - Type all mocks properly with TypeScript
     - Aim for 100% branch coverage

     Source code:
     [paste the full component source code here]"

  4. Review Gemini's output for correctness:
     - Verify import paths match the project structure
     - Check that component props are correctly typed
     - Ensure mock patterns match existing test utilities
     - Confirm no hallucinated APIs or methods
  5. Write the test file to __tests__/ComponentName.test.tsx
  6. Run: npx jest --coverage <test-file-path>
  7. Fix any failures yourself (do NOT re-delegate to Gemini for small fixes)
  8. Return a summary to the main conversation:
     - Number of test suites and individual tests
     - Coverage percentage (statements, branches, functions, lines)
     - Any issues found and how you fixed them

Allowed tools: Read, Write, Bash, Glob, Grep, mcp__gemini__ask-gemini
```

### Step 2.3 — Create the Code Reviewer agent

```
Name:         code-reviewer
Description:  Performs deep code review by delegating analysis to Gemini.
              Returns only a prioritized list of actionable findings.

System prompt:

  You are a senior code reviewer working on a React TypeScript project.
  You have access to Gemini via the mcp__gemini__ask-gemini tool.

  ## Your workflow

  1. Gather the code to review:
     - If reviewing staged changes: run `git diff --cached`
     - If given a file path: read the file
     - If reviewing recent work: run `git diff HEAD~1`
  2. Call the mcp__gemini__ask-gemini tool with this prompt:

     "Perform a thorough code review of this React TypeScript code.

     Review for these categories (rate each as ✅ Good, ⚠️ Warning, ❌ Issue):

     **Bugs & Logic Errors**
     - Off-by-one errors, null/undefined access, race conditions
     - Incorrect conditional logic, missing return statements

     **TypeScript**
     - Proper typing (no `any` unless justified)
     - Correct generic usage, discriminated unions where appropriate

     **React Patterns**
     - Missing or incorrect useEffect dependencies
     - Stale closures in callbacks
     - Unnecessary re-renders (missing useMemo/useCallback where it matters)
     - State that should be derived vs stored

     **Security**
     - XSS via dangerouslySetInnerHTML
     - Unsanitized user input
     - Exposed secrets or credentials

     **Performance**
     - Expensive computations in render path
     - Missing pagination or virtualization for large lists
     - Bundle size concerns (large imports)

     Code to review:
     [paste code here]"

  3. Synthesize Gemini's findings into a prioritized report.
  4. Return ONLY the actionable findings to the main conversation:
     - ❌ Critical issues (must fix)
     - ⚠️ Warnings (should fix)
     - 💡 Suggestions (nice to have)
     - Include file name, line number, and a one-line fix description for each

Allowed tools: Read, Bash(git *), Grep, Glob, mcp__gemini__ask-gemini
```

### Step 2.4 — Create the UI Builder agent

```
Name:         ui-builder
Description:  Builds new React UI components by delegating initial
              generation to Gemini, then adapting to project conventions.

System prompt:

  You are a UI component specialist working on a React TypeScript project
  with Ant Design v6. You have access to Gemini via the
  mcp__gemini__ask-gemini tool.

  ## Your workflow

  1. Read 1-2 existing components in the project to learn conventions:
     - File structure and naming patterns
     - How Redux/context is used
     - Styling approach (CSS Modules, styled-components, etc.)
     - Common patterns for loading/error states
  2. Call the mcp__gemini__ask-gemini tool with this prompt:

     "Create a React TypeScript component with these requirements:

     Component: [name and description from task]

     Tech stack:
     - React 18+ with TypeScript (strict)
     - Ant Design v6 for UI primitives (Button, Table, Form, Input, Modal)

     Requirements:
     - Export a properly typed functional component
     - Define a Props interface with JSDoc comments
     - Use Ant Design components, not raw HTML
     - Include loading and error states
     - Make it accessible (ARIA attributes, keyboard navigation)
     - Include responsive behavior if relevant

     [specific requirements from the task]"

  3. Adapt Gemini's output to match the project:
     - Fix import paths to match project structure
     - Wire up to Redux store if needed (useAppDispatch, useAppSelector)
     - Replace generic patterns with project-specific ones
     - Add to barrel exports (index.ts) if they exist
  4. Write the component file(s)
  5. Run `npx tsc --noEmit` to verify types compile
  6. Return a summary to the main conversation:
     - Files created (paths)
     - Component API (props interface)
     - Any decisions made during adaptation

Allowed tools: Read, Write, Bash, Glob, Grep, mcp__gemini__ask-gemini
```

---

## Layer 3: Create Skills for Task Routing

Skills tell Claude Code *when* to trigger the delegation workflow and *how*
to ask the user which execution mode to use.

### Step 3.1 — Create the directory structure

```bash
# Global skills (available in all projects)
mkdir -p ~/.claude/skills/gemini-tests
mkdir -p ~/.claude/skills/gemini-review
mkdir -p ~/.claude/skills/gemini-ui
mkdir -p ~/.claude/skills/gemini-brainstorm
```

### Step 3.2 — Test Writing Skill

Create `~/.claude/skills/gemini-tests/SKILL.md`:

```markdown
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
```

### Step 3.3 — Code Review Skill

Create `~/.claude/skills/gemini-review/SKILL.md`:

```markdown
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
```

### Step 3.4 — UI Component Building Skill

Create `~/.claude/skills/gemini-ui/SKILL.md`:

```markdown
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
```

### Step 3.5 — Brainstorming Skill (auto-direct, no prompt needed)

Create `~/.claude/skills/gemini-brainstorm/SKILL.md`:

```markdown
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
```

---

## Layer 4: Configure CLAUDE.md Routing Rules

This is the "brain" layer. It tells Claude Code how to think about every task.

### Step 4.1 — Global rules (`~/.claude/CLAUDE.md`)

Append this to your existing `~/.claude/CLAUDE.md` (don't overwrite what's there):

```markdown
## Multi-AI Delegation Strategy

I have a Gemini MCP server connected (gemini-mcp-tool) with these tools:
- mcp__gemini__ask-gemini: Main prompt tool (gemini-2.5-flash, supports @file, sandbox, changeMode)
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
```

### Step 4.2 — Project-specific rules (`.claude/CLAUDE.md`)

Add to your project's `.claude/CLAUDE.md`:

```markdown
## Project Context (for Gemini delegation prompts)

When delegating to Gemini (via sub-agents or direct MCP), include this
context in the mcp__gemini__ask-gemini prompt so Gemini produces
project-compatible output:

### Test Conventions
- Test runner: Jest with React Testing Library
- Test file location: `__tests__/` next to source files
- Shared utilities: `src/test-utils/renderWithProviders.tsx`
- Mock patterns: see `src/__mocks__/` for existing examples
- Coverage command: `npx jest --coverage --collectCoverageFrom='src/**/*.{ts,tsx}'`

### UI Conventions
- Component library: Ant Design v6
- State management: Redux Toolkit with typed hooks (useAppDispatch, useAppSelector)
- Form handling: Ant Design Form with custom validation rules
- Styling: CSS Modules with `.module.scss` files

### Project Structure
```
src/
  components/     # Shared/reusable components
  features/       # Feature-specific modules
    auth/
    audit/
    media/
  hooks/          # Custom React hooks
  store/          # Redux store, slices, selectors
  test-utils/     # Shared test utilities
  types/          # Shared TypeScript types
```
```

---

## Real-World Usage Examples

### Example 1: Writing tests (sub-agent mode)

**You type:**
```
> Write comprehensive tests for src/features/audit/LoginReportSearch.tsx
```

**Claude responds:**
```
This is a test writing task. How should I handle it?

1. **Sub-agent + Gemini** (recommended) — runs in isolated context
2. **Direct Gemini call** — faster, output stays in our conversation
3. **I'll handle it myself** — no Gemini
```

**You pick 1.**

**What happens behind the scenes:**

```
Main context:
  → Claude spawns test-writer sub-agent            (tiny cost)
  → ...waiting...
  → Sub-agent returns summary                      (tiny cost)

Sub-agent context (isolated, discarded after):
  → Reads LoginReportSearch.tsx                     (sub-agent tokens)
  → Calls mcp__gemini__ask-gemini with source       (sub-agent tokens)
  → Gemini returns 200-line test file               (FREE — Gemini tokens)
  → Reviews output, fixes 2 import paths            (sub-agent tokens)
  → Writes __tests__/LoginReportSearch.test.tsx      (sub-agent tokens)
  → Runs npx jest --coverage                        (sub-agent tokens)
  → Returns summary                                 (tiny cost back to main)
```

**Your main context only sees:**
```
✅ Tests written for LoginReportSearch.tsx

- 14 tests across 4 describe blocks
- Coverage: 96% statements, 92% branches, 100% functions
- All tests passing
- File: src/features/audit/__tests__/LoginReportSearch.test.tsx

Tests cover:
- Date range picker interactions
- Search form submission
- Loading and error states
- Table pagination
- CSV export button
- Empty results display

Want me to adjust anything?
```

**Token savings: ~85% vs Claude writing all tests in main context.**

---

### Example 2: Code review (sub-agent mode)

**You type:**
```
> Review my staged changes before I push
```

**Claude asks mode → you pick sub-agent.**

**Your main context only sees:**

```
✅ Code review complete for 4 changed files.

❌ Critical (1):
  - src/features/auth/useSession.ts:42
    Missing null check: user.profile?.email accessed without optional chaining
    Fix: change user.profile.email to user.profile?.email

⚠️ Warnings (2):
  - src/features/audit/SearchForm.tsx:67
    useEffect missing `searchTerm` in dependency array — will use stale value
  - src/features/audit/ResultsTable.tsx:23
    Ant Design Table pagination object recreated every render — wrap in useMemo

💡 Suggestions (1):
  - src/hooks/useDebounce.ts:15
    Consider adding a cleanup function to cancel pending timeout on unmount

Want me to fix the critical issue and warnings?
```

---

### Example 3: Quick question (direct MCP — no sub-agent needed)

**You type:**
```
> What's the cleanest way to handle AbortController cleanup in useEffect?
```

**Claude asks mode → you pick "Direct Gemini call".**

Claude calls `mcp__gemini__ask-gemini` with your question. Gemini's response
(a short code snippet with explanation) appears directly in your conversation.
Fast, and small enough that context impact is minimal.

---

### Example 4: Brainstorming (auto-direct, no prompt)

**You type:**
```
> What are my options for implementing real-time notifications in this app?
```

**Claude recognizes this as brainstorming and calls `mcp__gemini__brainstorm`
directly** — no execution mode prompt needed (per the CLAUDE.md exception rule).

Gemini returns structured ideas with pros/cons. Claude presents them.
You pick an approach, and Claude proceeds with implementation.

---

### Example 5: Building a new component (sub-agent mode)

**You type:**
```
> Create a reusable CSV export button that takes data and columns as props,
> shows a spinner during export, and supports cancellation with AbortController
```

**Claude asks mode → you pick sub-agent.**

**Your main context only sees:**

```
✅ Component created: CsvExportButton

Files:
  - src/components/CsvExportButton.tsx

Props interface:
  data: Record<string, unknown>[]
  columns: { key: string; header: string }[]
  filename?: string (default: "export.csv")
  onExportComplete?: () => void
  onExportError?: (error: Error) => void

Features:
  - Ant Design Button with download icon
  - Loading spinner during export
  - AbortController for cancellation (cancel button appears during export)
  - Error notification via Ant Design message
  - TypeScript strict, all props typed

TypeScript check: ✅ passing

Want me to add tests for this component?
```

---

### Example 6: Multi-file refactoring (Claude handles itself — no Gemini)

**You type:**
```
> Refactor src/features/auth to replace manual fetch calls with React Query
```

**Claude responds:**
```
This is a refactoring task across multiple files. I'll handle this myself
since I need full project context to coordinate the changes.
```

**Why no delegation?** Refactoring requires reading multiple interconnected
files, understanding how they depend on each other, and making coordinated
changes. Gemini can't do this — it has no filesystem access or project context.

---

### Example 7: Hybrid workflow — multi-step feature

**You type:**
```
> Add a debounced search input to the UsersTable that filters by name and email
```

**Claude identifies this as a multi-step task and asks:**
```
This involves both new code generation and integration work. I'd recommend:

- Sub-agent for generating the useDebounce hook + its tests
- I'll handle the integration into UsersTable myself (needs project context)

Sound good?
```

**You say yes. What happens:**

1. Claude spawns **ui-builder** → generates `useDebounce.ts` hook
2. Claude spawns **test-writer** → generates tests for the hook
3. Claude handles the integration **itself**:
   - Adds search input to UsersTable.tsx
   - Wires up debounced state to existing API calls
   - Updates the table filtering logic
4. Claude runs full test suite to verify nothing broke

**Pattern: Gemini generates isolated pieces via sub-agents, Claude integrates.**

---

## Quick Reference

### Setup checklist

| Step | Command / Action | Time |
|------|-----------------|------|
| Install Gemini MCP | `claude mcp add --env GEMINI_API_KEY=... --env GEMINI_MODEL=gemini-2.5-flash -s user gemini -- npx -y gemini-mcp-tool` | 2 min |
| Verify MCP | `/mcp` inside Claude Code | 1 min |
| Create sub-agents | `/agents` → create test-writer, code-reviewer, ui-builder | 10 min |
| Create skills | Write 4 SKILL.md files in `~/.claude/skills/` | 10 min |
| Update global CLAUDE.md | Add delegation rules to `~/.claude/CLAUDE.md` | 5 min |
| Update project CLAUDE.md | Add project conventions to `.claude/CLAUDE.md` | 5 min |

### Gemini MCP tools

| Tool | When to use |
|------|-------------|
| `mcp__gemini__ask-gemini` | All code generation, review, analysis, Q&A |
| `mcp__gemini__brainstorm` | Exploring approaches, ideation, comparing patterns |
| `mcp__gemini__fetch-chunk` | Retrieving parts of large paginated responses |

### Skills and slash commands

| Command | What it does |
|---------|-------------|
| `/gemini-tests` | Triggers test writing workflow (asks execution mode) |
| `/gemini-review` | Triggers code review workflow (asks execution mode) |
| `/gemini-ui` | Triggers UI building workflow (asks execution mode) |
| `/gemini-brainstorm` | Triggers brainstorming (auto-direct, no mode prompt) |
| `/agents` | Manage sub-agents |
| `/mcp` | Verify Gemini MCP connection |
| `/clear` | Clear context (use between tasks!) |

### Natural language triggers

You don't need to use slash commands. These phrases trigger the skills:

- "write tests for..." / "add test coverage" → gemini-tests
- "review this code" / "check for bugs" → gemini-review
- "create a component" / "build a page" → gemini-ui
- "brainstorm" / "what are my options" → gemini-brainstorm

### Execution mode cheat sheet

| Situation | Recommended mode |
|-----------|-----------------|
| Full test suite for a component | Sub-agent + Gemini |
| One quick unit test | Direct Gemini MCP |
| PR / staged changes review | Sub-agent + Gemini |
| "Is this line correct?" | Direct Gemini MCP or self |
| New component from scratch | Sub-agent + Gemini |
| Simple component skeleton | Direct Gemini MCP |
| Quick "how do I..." question | Direct Gemini MCP |
| Brainstorming approaches | Direct MCP (auto, no prompt) |
| Multi-file refactoring | Self (no Gemini) |
| Git operations | Self (no Gemini) |
| Fixing < 30 lines | Self (no Gemini) |
| Architecture decisions | Self (no Gemini) |

---

## Troubleshooting

### Gemini MCP not showing up

```bash
# Verify registration
claude mcp list

# If missing, re-add with correct syntax
claude mcp remove gemini
claude mcp add --env GEMINI_API_KEY=your_key --env GEMINI_MODEL=gemini-2.5-flash -s user gemini -- npx -y gemini-mcp-tool

# Restart Claude Code
exit
claude
```

### "env" command error or "unknown option -y" during MCP setup

If you see `Command: env` in the error, you used the wrong syntax.
If you see `unknown option '-y'`, backslash line breaks broke the parsing.
The `--env` flags must come before the server name, and **run as a single line**:

```bash
# ❌ Wrong (tries to run "env" as executable)
claude mcp add gemini -- env GEMINI_API_KEY=key npx -y gemini-mcp-tool

# ❌ Wrong (backslash line breaks cause parsing errors)
claude mcp add \
  --env GEMINI_API_KEY=key \
  -s user \
  gemini \
  -- npx -y gemini-mcp-tool

# ✅ Correct (single line, --env before server name)
claude mcp add --env GEMINI_API_KEY=key --env GEMINI_MODEL=gemini-2.5-flash -s user gemini -- npx -y gemini-mcp-tool
```

### Gemini daily quota exhausted (429 error)

If you see `TerminalQuotaError: You have exhausted your daily quota`:
- The **free tier Pro model** has very low daily limits (can hit 0 quickly)
- **Solution**: Use `gemini-2.5-flash` via `--env GEMINI_MODEL=gemini-2.5-flash`
  (this is already set in the install command above)
- If you still hit limits on Flash, consider enabling billing on your
  Google AI Studio account — Gemini is very cheap ($0.15/M input tokens for Flash)
- **Immediate workaround**: Tell Claude to use option 3 (handle itself) until
  quota resets (typically midnight Pacific time)
- Add this to your CLAUDE.md: "If mcp__gemini__ask-gemini returns a quota or
  rate limit error, inform the user and offer to handle the task myself.
  Do NOT retry Gemini repeatedly."

### Sub-agents can't access Gemini MCP tools

Sub-agents inherit MCP tools from the parent Claude Code session.
If the Gemini MCP is registered at `user` scope (`-s user`), sub-agents
should see it. Verify:

```
> /mcp
# Confirm gemini tools are listed

> /agents
# Check that your agent's allowed tools include mcp__gemini__ask-gemini
```

### Claude doesn't ask execution mode, just does everything itself

- Verify skills are loaded: `/skills` or `/context`
- Check that CLAUDE.md has the delegation rules: `cat ~/.claude/CLAUDE.md`
- Check for character budget warnings in `/context` (too many skills can
  exceed the budget)
- Be explicit while the habit forms: "Use Gemini to write tests for this"

### Gemini returns low-quality or incorrect code

- Make sure you're sending **actual source code** in the prompt, not file paths
- Include project conventions (TypeScript strict, Ant Design v6, etc.)
- gemini-mcp-tool is configured with gemini-2.5-flash for best free tier quota
- If quality is insufficient, switch to `gemini-2.5-pro` (requires paid API key
  or very light usage to stay within free tier)
- If response is truncated, use mcp__gemini__fetch-chunk to get remaining parts

### Skills not loading

```bash
# Check file locations
ls ~/.claude/skills/*/SKILL.md

# Verify inside Claude Code
> /skills

# Check for budget warnings
> /context
# If skills are excluded due to budget, set:
export SLASH_COMMAND_TOOL_CHAR_BUDGET=32000
```

---

## Cost-Saving Tips

1. **`/clear` between tasks** — don't let old context eat your token budget.
   Every time you start something new, clear the chat.

2. **Default to sub-agents for large outputs** — the whole point is keeping
   generated code out of your main context.

3. **Use Gemini for exploration** — "What approaches exist for X?" burns
   Gemini tokens (free), not Claude tokens.

4. **Batch your requests** — send multiple files to a sub-agent at once
   rather than spawning one per file.

5. **Fix small issues yourself** — if Gemini's output needs a 5-line fix,
   Claude should just do it. Don't re-delegate.

6. **Don't re-delegate** — if Gemini's first output is 80% right, fix
   the 20% yourself. One round trip is almost always enough.

7. **Direct MCP for small stuff** — spinning up a sub-agent for a
   one-line question is overkill.

8. **Keep CLAUDE.md lean** — don't overload it with context. Reference
   files instead of inlining everything.

9. **Use `mcp__gemini__brainstorm` for ideation** — it's specifically
   designed for structured brainstorming and costs zero Claude tokens.