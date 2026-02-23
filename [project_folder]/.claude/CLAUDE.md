## Project Context (for Gemini delegation prompts)

When delegating to Gemini (via sub-agents or direct MCP), include this
context in the mcp__gemini__ask-gemini prompt so Gemini produces
project-compatible output:

### Test Conventions
- Test runner: Jest with React Testing Library
- Test file location: `YOUR TEST DIR`
- Shared utilities: ``
- Mock patterns: ``
- Coverage command: `npx jest --coverage --collectCoverageFrom='YOUR_TEST_DIR/**/*.{ts,tsx}'`

### UI Conventions
- Component library: Ant Design v6
- State management: Redux Toolkit with typed hooks (useAppDispatch, useAppSelector)
- Form handling: Ant Design Form with custom validation rules
- Styling: Emotion css-in-js library and native css styles

### Project Structure
