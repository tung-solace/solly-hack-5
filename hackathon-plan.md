# Hackathon Plan: Storybook Test Generation Plugin for maas-ui

## Project Vision

Build a Claude Code plugin (skill) that **automatically generates Storybook stories with interactive tests** for components in the maas-ui repo. Combine this with the existing `pr-context-loader` skill to create a workflow where PR feedback about missing/broken stories can be addressed automatically.

---

## Part 1: Understanding What We're Building

### The Problem
- maas-ui has **~193 stories** across EP (110) and INTG (83) micro-frontends
- There are **200-300+ components/pages** that lack story coverage
- Writing stories is tedious: each requires MSW handlers, mock data, decorators, play functions, and knowledge of project-specific patterns (`data-qa`, React Query v5, theme providers, etc.)
- PR reviewers frequently request story additions/fixes, requiring manual effort

### The Solution
A two-skill system:
1. **`storybook-generator`** — A Claude Code skill that analyzes a component and generates a complete `.stories.tsx` file with MSW mocks, play functions, and interactive tests
2. **Integration with `pr-context-loader`** — When PR feedback mentions missing stories or story fixes, the developer can pipe that context directly into the generator

---

## Part 2: Plugin Architecture

### Directory Structure

```
maas-ui/.claude/skills/storybook-generator/
├── SKILL.md                          # Main skill definition & workflow
├── references/
│   ├── story-patterns.md             # Extracted patterns from existing stories
│   ├── msw-patterns.md               # MSW handler patterns & utilities reference
│   ├── component-analysis-guide.md   # How to analyze a component for story generation
│   └── checklist.md                  # Validation checklist for generated stories
└── scripts/
    └── find_component_context.sh     # Script to gather component deps & API usage
```

### Skill Definition (SKILL.md)

```yaml
---
name: storybook-generator
description: >
  Generate Storybook stories with interactive tests for React components in
  maas-ui. Use when the user asks to "generate a story", "create storybook
  tests", "add story coverage", or wants to create stories for a component.
  Also use when addressing PR feedback about missing or broken stories.
argument-hint: [component-path-or-name]
allowed-tools: Read, Grep, Glob, Bash(npm run test-storybook:*), Edit, Write
---
```

---

## Part 3: The Generation Workflow

### Phase 1: Component Analysis

When invoked with a component path (e.g., `src/components/SearchPanel/SearchPanel.tsx`):

1. **Read the component file** — Extract props interface, hooks used, context dependencies
2. **Trace API dependencies** — Find `useQuery`/`useMutation` calls, map to `queryKeys.ts` entries, find the API endpoints
3. **Find existing MSW mocks** — Search `tests/mocks/` for handlers that match the endpoints
4. **Identify context requirements** — Check what providers the component needs (QueryClient, Router, Theme, Jotai atoms, etc.)
5. **Check for `data-qa` attributes** — Extract all `data-qa` values for test selectors
6. **Find related existing stories** — Look for stories of parent/sibling components for pattern reference

The `scripts/find_component_context.sh` script automates steps 2-6:

```bash
#!/bin/bash
# Usage: find_component_context.sh <component-path> <mfe-name>
# Outputs structured analysis to stdout

COMPONENT_PATH=$1
MFE=$2  # "ep" or "intg"
MFE_ROOT="micro-frontends/$MFE/src"

echo "## Component Analysis: $COMPONENT_PATH"

# 1. Find imports and hooks
echo "### Dependencies"
grep -E "^import|useQuery|useMutation|useContext|useAtom" "$COMPONENT_PATH"

# 2. Find API endpoints used
echo "### API Endpoints"
grep -rn "queryKey" "$COMPONENT_PATH" | head -20

# 3. Find existing MSW mocks for those endpoints
echo "### Existing MSW Mocks"
# Extract query key names, search for matching handlers
grep -oP 'queryKey:\s*\[([^\]]+)\]' "$COMPONENT_PATH" | while read -r key; do
  grep -rl "$key" "$MFE_ROOT/tests/mocks/" 2>/dev/null
done

# 4. Find data-qa attributes
echo "### Test Selectors (data-qa)"
grep -oP 'data-qa="([^"]+)"' "$COMPONENT_PATH"
grep -oP 'dataQa="([^"]+)"' "$COMPONENT_PATH"

# 5. Find context providers needed
echo "### Context Dependencies"
grep -E "useContext|Provider|useAtom" "$COMPONENT_PATH"

# 6. Find related stories
COMPONENT_NAME=$(basename "$COMPONENT_PATH" .tsx)
echo "### Related Stories"
find "$MFE_ROOT/stories" -name "*.stories.tsx" | xargs grep -l "$COMPONENT_NAME" 2>/dev/null
```

### Phase 2: Story Generation

Using the analysis, generate a story file following these exact patterns:

#### Template Structure (EP micro-frontend)

```typescript
import { action } from "@storybook/addon-actions";
import { expect } from "@storybook/jest";
import { Meta, StoryObj } from "@storybook/react";
import { configure, userEvent, within, waitFor } from "@storybook/testing-library";
import React from "react";
// Component import
import { ComponentName } from "components/path/ComponentName";
// Mock data imports
import { mockHandler1, mockHandler2 } from "tests/mocks/api/domain";
// Additional mock data
import { mockData } from "./mockProps";

configure({ testIdAttribute: "data-qa" });

const meta: Meta<typeof ComponentName> = {
  title: "Domain/ComponentName",
  component: ComponentName,
  parameters: {
    design: {
      type: "figma",
      url: "<!-- Figma URL if available -->"
    }
  }
};
export default meta;

type Story = StoryObj<typeof ComponentName>;

// === Default Story ===
export const Default: Story = {
  parameters: {
    msw: {
      handlers: [mockHandler1, mockHandler2]
    }
  },
  render: () => (
    <ComponentName
      prop1="value"
      prop2={mockData}
      onAction={action("onAction")}
    />
  )
};

// === Interactive Test Story ===
export const WithInteraction: Story = {
  ...Default,
  play: async ({ canvasElement }) => {
    const canvas = within(canvasElement);

    // Wait for component to load
    const element = await canvas.findByTestId("componentDataQa", {}, { timeout: 5000 });
    expect(element).toBeInTheDocument();

    // Simulate user interaction
    await userEvent.click(element);

    // Assert result
    await waitFor(() => {
      expect(canvas.getByText("Expected Text")).toBeInTheDocument();
    });
  }
};

// === Error State Story ===
export const ErrorState: Story = {
  parameters: {
    msw: {
      handlers: [errorHandler]
    }
  },
  render: () => (
    <ComponentName prop1="value" />
  )
};

// === Empty State Story ===
export const EmptyState: Story = {
  parameters: {
    msw: {
      handlers: [emptyHandler]
    }
  },
  render: () => (
    <ComponentName prop1="value" />
  )
};
```

### Phase 3: Validation

After generating the story:

1. **Lint check** — Run ESLint on the generated file
2. **Type check** — Run `npm run types` to verify TypeScript compiles
3. **Storybook build** — Optionally run `npm run test-storybook` on the specific file
4. **Pattern compliance** — Check against `references/checklist.md`:
   - [ ] Uses `data-qa` not `data-testid`
   - [ ] Uses CSF3 format (`Meta`/`StoryObj`, not `ComponentMeta`/`ComponentStory`)
   - [ ] MSW handlers use `http` from `msw` (v2) not `rest` (v1) — verify which version the MFE uses
   - [ ] QueryClient configured with strict test options
   - [ ] DOM references refreshed after mutations (React Query v5)
   - [ ] Uses `isPending` not `isLoading` in any conditional rendering
   - [ ] Play functions use `await` for all async operations
   - [ ] `within()` used to scope queries

---

## Part 4: PR Context Integration

### Combined Workflow

When a developer gets PR feedback about stories:

```
Developer: /pr-context-loader https://github.com/SolaceDev/maas-ui/pull/1234

Claude: Loaded PR #1234: "Add event mesh filtering"
         8 files changed, 5 review comments

         Actionable (2):
           #1 [EventMeshFilter.tsx:42] reviewer — "Please add storybook story for this new component"
           #2 [EventMeshFilter.tsx:86] reviewer — "Add interaction tests for the filter dropdown"
         ...

Developer: Address #1 and #2

Claude: I'll generate a story for EventMeshFilter. Let me analyze the component first.
        [Runs storybook-generator workflow]
        [Generates EventMeshFilter.stories.tsx with MSW mocks + play functions]
        [Shows the generated story for confirmation]
```

### Implementation

The integration is natural — no code changes needed to `pr-context-loader`. The workflow:

1. Developer loads PR context with `/pr-context-loader`
2. Claude categorizes comments, identifies story-related ones
3. When addressing story comments, Claude recognizes it needs to generate a story
4. Claude invokes the `storybook-generator` logic (reads the SKILL.md references internally)
5. The story is generated using the component analysis workflow

To make this smoother, add a note in `storybook-generator/SKILL.md`:

```markdown
## Integration with PR Context

When invoked after pr-context-loader has loaded PR context:
1. Extract the component file path from the PR comment
2. Check the PR diff for the component's current state (it may be new/modified)
3. Generate the story targeting the component as it exists in the working tree
4. Reference the reviewer's specific feedback in the story design choices
```

---

## Part 5: Reference Files Content

### references/story-patterns.md

Document these patterns extracted from existing stories:

| Pattern | EP | INTG |
|---------|-----|------|
| Import style | `@storybook/react` | `@storybook/react-webpack5` |
| CSF version | CSF3 (Meta/StoryObj) | CSF3 (Meta/StoryObj) |
| MSW version | v1 (rest) in mocks, v2 (http) in newer stories | Mixed |
| Router | MemoryRouter when needed | MemoryRouter with Route |
| Test IDs | `data-qa` via configure() | `data-qa` via configure() |
| React Query | v5 (object config) | v3 (positional args) |
| Theme | withSolaceLayout decorator | ThemeProvider decorator |
| State | Jotai atoms | Jotai atoms |

### references/msw-patterns.md

Document the mock utility functions from `tests/mocks/utils.ts`:
- `getDto()`, `getDtos()`, `getMeta()` — Data builders
- `getEmpty()`, `getAllEmpty()` — Empty list handlers
- `getPost()`, `getPatch()`, `getPut()`, `getDelete()` — CRUD handlers
- `getError()` — Error response handlers
- Domain-specific handler factories (e.g., `getApplications()`, `getApplicationVersions()`)

### references/checklist.md

```markdown
# Story Generation Checklist

## Required
- [ ] Uses correct import style for the target MFE
- [ ] Uses CSF3 format (Meta<typeof Component> + StoryObj)
- [ ] Configures testIdAttribute to "data-qa"
- [ ] Has MSW handlers for all API calls the component makes
- [ ] Has at least: Default, Empty State, Error State stories
- [ ] Play functions use async/await consistently
- [ ] DOM queries scoped with within(canvasElement)

## Interactive Tests
- [ ] Tests cover primary user workflows
- [ ] userEvent used for all interactions (not direct DOM methods)
- [ ] waitFor used for async assertions
- [ ] DOM references refreshed after mutations (EP/React Query v5)
- [ ] Timeouts set appropriately for slow operations

## Mock Data
- [ ] Reuses existing mock utilities from tests/mocks/utils.ts
- [ ] Reuses existing domain handlers where available
- [ ] Creates minimal new mock data (only what's needed)
- [ ] Mock data is realistic (not lorem ipsum)

## Project Conventions
- [ ] data-qa attributes used (not data-testid)
- [ ] isPending used (not isLoading) for EP components
- [ ] Query keys from queryKeys.ts (not hardcoded)
- [ ] File placed in correct stories/ subdirectory
```

---

## Part 6: Hackathon Execution Plan

### Milestone 1: Core Skill Structure
**Goal:** Skill invocable via `/storybook-generator ComponentPath`

- Create `SKILL.md` with frontmatter and workflow instructions
- Create `references/story-patterns.md` from analysis above
- Create `references/msw-patterns.md` from analysis above
- Create `references/checklist.md`
- Create `scripts/find_component_context.sh`
- Test: invoke on a simple component (e.g., `PendingBadge`)

### Milestone 2: Component Analysis Pipeline
**Goal:** Skill correctly identifies all dependencies for a component

- Test on a medium-complexity component (e.g., `EmptyState`)
- Test on a complex component (e.g., `SearchPanel`)
- Refine the analysis script based on what Claude misses
- Verify MSW mock discovery works reliably

### Milestone 3: Story Generation Quality
**Goal:** Generated stories compile, lint, and pass interaction tests

- Generate a story for a component that already has one — compare quality
- Generate a story for a component without one — validate it works
- Iterate on the template and checklist
- Run `npm run test-storybook` on generated stories

### Milestone 4: PR Integration
**Goal:** Seamless flow from PR feedback to generated story

- Load a real PR with story-related feedback using `pr-context-loader`
- Address the feedback using the generator
- Verify the generated story addresses the reviewer's specific concerns
- Document the combined workflow

### Milestone 5: Demo & Polish
**Goal:** Impressive hackathon demo

- Prepare a demo PR with story feedback
- Record the end-to-end workflow
- Document the ROI: manual story writing time vs. generator time
- Add edge case handling (components with no API calls, pure UI components, etc.)

---

## Part 7: Key Technical Decisions

### Q: Should the skill use `context: fork` (isolated subagent)?
**A: No.** The skill needs access to the full conversation context, especially when used with `pr-context-loader`. Running in the main context allows it to reference PR comments and file diffs already loaded.

### Q: Should we generate mock data or reuse existing?
**A: Reuse first, generate only what's missing.** The codebase has rich mock utilities in `tests/mocks/`. The skill should search for existing handlers before creating new ones.

### Q: Which MSW version to target?
**A: Match the MFE.** EP uses a mix (older stories use v1 `rest`, newer use v2 `http`). The skill should detect which version the MFE's storybook config uses and match it. For new stories, prefer v2 (`http`/`HttpResponse`).

### Q: How to handle components with many context dependencies?
**A: Mirror the storybook preview.tsx decorators.** The storybook preview already wraps stories with QueryClient, ThemeProvider, EnvironmentProvider, etc. The skill only needs to add component-specific context (like MemoryRouter for routed components or specific Jotai atom values).

### Q: Should play functions test everything or just key interactions?
**A: Key interactions only.** Generate play functions for the primary user workflow (render → interact → assert). Don't try to achieve 100% interaction coverage — that's diminishing returns for a hackathon.

---

## Part 8: Success Criteria

1. **Invokable** — `/storybook-generator src/components/SearchPanel/SearchPanel.tsx` produces a valid story
2. **Compiles** — Generated story passes TypeScript type checking
3. **Runs** — Story renders in Storybook without errors
4. **Tests pass** — Play functions execute successfully in `test-storybook`
5. **Integrates** — Works seamlessly after `/pr-context-loader` loads story-related PR feedback
6. **Fast** — Full generation takes under 2 minutes per component
7. **Accurate** — Uses correct project conventions (data-qa, React Query v5, CSF3, MSW handlers)
