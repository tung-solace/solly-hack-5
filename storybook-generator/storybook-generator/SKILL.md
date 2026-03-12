# Storybook Story Generator Skill

A reusable skill for automatically generating and updating Storybook stories for React components.

## Core Concept

This skill is split into two parts:
1. **SKILL.md** (this file): Generic execution engine - reusable across all MFEs
2. **GUIDELINE.md**: Project-specific patterns, templates, and conventions

The skill reads GUIDELINE.md during execution to learn project-specific patterns.

---

## Usage Patterns

### Automatic Mode (Recommended)
```
/storybook-generator
```
**Automatically:**
- Runs `/git-diff` to analyze recent changes
- Identifies components missing stories
- Shows you the list of components
- Generates stories for all identified components

### Single Component
```
/storybook-generator src/components/agents/AgentCard.tsx
```

### Multiple Components
```
/storybook-generator src/components/agents/AgentCard.tsx src/components/agents/AgentList.tsx
```

### Directory (Recursive)
```
/storybook-generator src/components/multiFlow/ --recursive
```

### Page with Dependencies
```
/storybook-generator src/pages/multiFlow/CreateChildMI.tsx --include-children
```

### Update Existing Story
```
/storybook-generator src/components/connectors/ConnectorCard.tsx --update
```

---

## Execution Workflow

When this skill is invoked:

### Step 1: Parse Arguments

Parse the command to determine mode:

**A. Automatic Mode (No Arguments)**
- If NO component paths provided:
  - Proceed to Step 2 to run git-diff
  - Extract components needing stories from git-diff output
  - Show list to user
  - Generate stories for all identified components

**B. Manual Mode (With Arguments)**
- Component path(s) provided
- Flags: `--update`, `--recursive`, `--include-children`
- Proceed with specified components

### Step 2: Git Diff Integration

**AUTOMATIC BEHAVIOR**: This step runs automatically based on the mode.

**For Automatic Mode (no arguments):**

1. **Always invoke git-diff**:
   - Automatically invoke `/git-diff` skill
   - Wait for analysis to complete
   - Git-diff will identify components missing stories (based on Story coverage config)

2. **Extract components needing stories**:
   - Parse git-diff output for "Missing Stories" section
   - Get list of component paths that need stories generated

3. **Show list to user**:
   ```
   ✓ Git-diff analysis complete
   ✓ Found 3 components needing stories:
     - src/components/multiFlow/FlowDetailCard.tsx
     - src/components/multiFlow/MiFlowMetrics.tsx
     - src/components/agents/AgentCard.tsx

   Generating stories for all components...
   ```

4. **Proceed to Phase 1** for each component

**For Manual Mode (with component paths):**

1. **Check if any component is in recent git changes**:
   ```bash
   git status --short
   ```
   - Look for component file(s) in output (modified, added, or untracked)
   - Works for both single and multiple files

2. **Check if git-diff was already run**:
   - Look back in conversation history for recent `/git-diff` invocation
   - If found in last ~5 messages, skip automatic invocation

3. **Automatically invoke git-diff if needed**:
   - If ANY component is in recent changes AND git-diff wasn't run yet:
     - Automatically invoke `/git-diff` skill (once, even for multiple files)
     - Wait for analysis to complete
     - Store output for use in Phase 0

4. **Skip automatic invocation if**:
   - Component(s) haven't changed recently
   - Git-diff was already run manually in this session
   - No git repository detected

**Benefits:**
- ✅ Automatic 404 error prevention
- ✅ Faster generation (skip redundant API tracing)
- ✅ Better documentation (transformations explained)
- ✅ Works seamlessly for batch processing

**Example output:**
```
✓ Detected FlowDetailCard.tsx in recent changes
✓ Automatically invoking /git-diff for context...
✓ Git-diff analysis complete
✓ Found 2 API endpoints with transformations
✓ Proceeding to Phase 0 with git-diff context...
```

### Step 3: Process Each Component

For each component, follow the **7-Phase Mandatory Workflow**:

**Note:** If git-diff context was gathered in Step 2, use it to inform Phase 2 (API endpoints) and Phase 4 (handler discovery).

---

## Phase 0: Contextual Pre-Analysis (Automatic when available)

**Goal**: Leverage git-diff analysis if component was recently changed

**When this phase runs:**
- Step 2 detected component in recent changes
- Git-diff was automatically invoked (or run manually earlier)
- Git-diff analysis output is available

**Process:**

1. **Check if git-diff context exists** from Step 2
   - If YES: Extract API endpoints, hooks, and transformations from git-diff output
   - If NO: Skip to Phase 1 (use manual tracing in Phase 2)

2. **Extract from git-diff analysis:**
   - List of React Query hooks used
   - Actual API endpoints (after transformations)
   - Data transformation functions documented
   - Component dependencies

3. **Use this context to enhance:**
   - Phase 2: Skip redundant API tracing (already done by git-diff)
   - Phase 4: Prioritize handlers identified by git-diff
   - Phase 6: Better story variant ideas based on props changes

**Output**: Contextual information to speed up and improve story generation

**Example:**
```
Git-diff analysis found:
- useGetConnectorTypeById → /microIntegrationTypes/ibmmq (after splitMITypesString)
- useEnvironments → /api/v2/platform/environments
- Jotai atom: selectedConnectorRows
- Router: useHistory

Using this context to generate story...
```

---

## Phase 1: Component Analysis

**Goal**: Understand what the component needs

1. **Read the component file** using the Read tool
2. **Extract component information**:
   - Component name
   - Props interface
   - Default export vs named export
3. **Identify dependencies**:
   - React Query hooks: `useQuery`, `useMutation`, `useSuspenseQuery`
   - Jotai atoms: `useAtom`, `useSetAtom`, `useAtomValue`
   - Router hooks: `useHistory`, `useLocation`, `useParams`, `Link`
   - Context hooks: `useApplicationContext`, custom contexts
   - Form libraries: `react-hook-form`, RJSF
4. **Document findings** for next phases

**Output**: Component dependency profile

---

## Phase 2: API Call Deep Dive

**Goal**: Find ALL actual API endpoints the component calls

**Check Phase 0 context first:**
- If git-diff analysis was performed in Phase 0 and identified API endpoints:
  - Use those endpoints directly
  - Verify transformations are documented
  - Skip to documenting endpoints for Phase 4
- If NO git-diff context OR endpoints unclear:
  - Proceed with manual tracing below

### If Component Has React Query Hooks:

1. **Extract hook names** from component (e.g., `useGetConnectorTypeById`)

2. **Find hook definitions**:
   ```
   Use Grep to search: "export.*useGetConnectorTypeById" in src/api
   ```

3. **Read hook file** to extract:
   - Endpoint pattern (e.g., `/api/v2/integration/microIntegrationTypes/${id}`)
   - Query key structure
   - Any parameters

4. **Trace data transformations**:
   - If hook receives function calls as arguments (e.g., `splitMITypesString(...)`):
     - Use Grep to find the transformation function
     - Read the function to understand what it does
     - **Test the transformation** with example data
     - Example: `splitMITypesString("ibmmq-source")` → `"ibmmq"`

5. **Determine ACTUAL endpoints called**:
   - Combine endpoint pattern + transformed values
   - Example: `/microIntegrationTypes/ibmmq` (NOT `/microIntegrationTypes/ibmmq-source`)

6. **Document all endpoints** for Phase 4

**Output**: List of actual API endpoints (after transformations)

### If Component Has No React Query Hooks:

Skip to Phase 3.

---

## Phase 3: Learn Project Patterns

**Goal**: Understand how similar components are tested

1. **Read GUIDELINE.md** to learn:
   - Use the Read tool to read the entire GUIDELINE.md file
   - Learn the 4 decorator patterns
   - Learn the component type decision matrix
   - Learn the MSW handler strategies
   - Learn the pitfall checklist

2. **Find similar components** (optional but helpful):
   - Use Grep to find components using the same hooks
   - Use Glob to check if those components have stories
   - Read successful stories to see patterns in action

**Output**: Understanding of project patterns and conventions

---

## Phase 4: Mock Handler Discovery

**Goal**: Find or create MSW handlers for API endpoints

### If Component Calls APIs (from Phase 2):

1. **Read GUIDELINE.md section "Common Mock Files"**:
   - Use Read tool to read this specific section
   - Get list of common mock files and their locations

2. **Search for existing handlers**:
   - Use Glob to find mock files: `src/mocks/*.ts`
   - For each endpoint from Phase 2, search mock files
   - Check if handler paths match ACTUAL endpoints (after transformations)

3. **Distinguish list vs by-ID endpoints**:
   - List: `/api/v2/resource` → returns array
   - By-ID: `/api/v2/resource/:id` → returns single object
   - **Critical**: These are SEPARATE handlers!

4. **Read GUIDELINE.md section "MSW Handler Strategies"**:
   - Learn when to use imported handlers
   - Learn when to create inline handlers
   - Learn the hybrid approach

5. **Plan handler strategy**:
   - Reuse existing handlers where possible
   - Create inline handlers for missing endpoints
   - Document which strategy to use

**Output**: Handler strategy with list of handlers to use/create

### If Component Has No API Calls:

Skip this phase.

---

## Phase 5: Select Decorator Pattern

**Goal**: Choose the right decorator stack for the component

1. **Read GUIDELINE.md section "Component Type Decision Matrix"**:
   - Use the Read tool to read this section
   - Match component dependencies (from Phase 1) to pattern

2. **Determine pattern** based on dependencies:
   - No dependencies → Pattern 1 (Simple Presentational)
   - Uses routing only → Pattern 2 (Routing Only)
   - Uses React Query + Jotai → Pattern 3 (React Query + Jotai)
   - Page-level with full context → Pattern 4 (Full Context Stack)

3. **Read GUIDELINE.md pattern template**:
   - Use Read tool to read the specific pattern section
   - Get the decorator template
   - Get the wrapper component pattern (if Jotai atoms used)

**Output**: Selected pattern with decorator template

---

## Phase 6: Story Generation

**Goal**: Create or update the story file

### For New Stories:

1. **Check if story exists**:
   - Use Glob to find: `src/stories/**/ComponentName.stories.tsx`
   - If exists without `--update` flag, report to user

2. **Prepare story content**:
   - Start with basic template from selected pattern
   - Add imports (always include `import React from "react"`)
   - Add decorators from pattern template
   - Add MSW handlers from Phase 4
   - Add wrapper component if Jotai atoms detected

3. **Generate story variants**:
   - Read GUIDELINE.md to learn variant types
   - Generate variants based on component type:
     - State-based (for status components)
     - Props-based (different prop combinations)
     - Interaction-based (user flows)
     - Loading/Error states (if applicable)

4. **Add play functions**:
   - Read GUIDELINE.md section "Testing Patterns & Assertions"
   - Add appropriate test assertions
   - Use `data-qa` attributes (NOT `data-testid`)
   - Use appropriate timeouts for async operations

5. **Add comments**:
   - Document data transformations if any
   - Reference transformation functions
   - List covered endpoints
   - Example:
     ```typescript
     // NOTE: useGetConnectorTypeById receives splitMITypesString(...)
     // splitMITypesString("ibmmq-source") returns "ibmmq" (direction removed)
     // Therefore handlers match base types: /ibmmq, not /ibmmq-source
     ```

6. **Write story file**:
   - Use Write tool to create the story file
   - Location: `src/stories/CategoryName/ComponentName.stories.tsx`
   - Use PascalCase for directory names

**Output**: New story file created

### For Updating Stories (--update flag):

Continue to Phase 7.

---

## Phase 7: Pitfall Validation (MANDATORY for --update)

**Goal**: Validate and fix common issues

1. **Read existing story** using Read tool

2. **Read GUIDELINE.md section "Pitfall Validation Checklist"**:
   - Get the complete 7-category checklist

3. **Validate against each category**:

   **Category 1: Imports & Dependencies**
   - [ ] React import present: `import React from "react"`
   - [ ] Test imports correct: `expect`, `waitFor`, `within`, `userEvent`
   - [ ] Component import path correct (check casing)
   - [ ] Mock imports match actual files

   **Category 2: MSW Handlers** (if component uses React Query)
   - [ ] List endpoint handler exists (if using list query)
   - [ ] By-ID endpoint handler exists (if using by-ID query)
   - [ ] Handler paths match ACTUAL endpoints (after transformations)
   - [ ] Transformation functions documented in comments

   **Category 3: QueryClient Configuration** (if component uses React Query)
   - [ ] QueryClient has strict settings:
     - `retry: false`
     - `refetchOnWindowFocus: false`
     - `refetchOnMount: false`
     - `refetchOnReconnect: false`
     - `gcTime: 0`
     - `staleTime: Infinity`

   **Category 4: Jotai Atoms** (if component uses atoms)
   - [ ] Wrapper component present to reset atoms
   - [ ] Wrapper uses `useEffect` with `useSetAtom`
   - [ ] Stories use wrapper via `render` prop

   **Category 5: Decorator Stack**
   - [ ] Decorators match component dependencies
   - [ ] Has MemoryRouter if uses routing hooks
   - [ ] Has QueryClientProvider if uses React Query
   - [ ] Has JotaiProvider if uses Jotai
   - [ ] Has ApplicationProvider if uses context

   **Category 6: Test Assertions**
   - [ ] Async operations use `waitFor` or `findBy`
   - [ ] Timeout values adequate (>= 3000ms for slow ops)
   - [ ] Dropdown assertions wait for visibility
   - [ ] No stale DOM references

   **Category 7: Story Variants**
   - [ ] Covers significant component states
   - [ ] Tests user interactions where applicable
   - [ ] Tests error/loading states
   - [ ] Descriptive variant names

4. **Fix violations**:
   - Use Edit tool to fix any issues found
   - Add missing imports
   - Fix handler paths
   - Add/fix QueryClient config
   - Add wrapper component if missing
   - Adjust timeouts
   - Add missing handlers

5. **Document fixes** for reporting

**Output**: Validated and fixed story file

---

## Step 3: Report Results

Provide comprehensive output including:

0. **Git-Diff Context** (if used):
   - Note if git-diff pre-analysis was performed
   - API endpoints identified from git-diff
   - Transformations documented from git-diff

1. **Components Analyzed**:
   - List of components processed
   - Component types identified

2. **Dependencies Found**:
   - React Query hooks traced
   - Jotai atoms identified
   - Router hooks found

3. **API Endpoints** (if applicable):
   - Hooks traced and their actual endpoints
   - Transformations discovered
   - Example: `useGetConnectorTypeById` → `splitMITypesString` → `/microIntegrationTypes/ibmmq`

4. **Handler Strategy**:
   - Handlers reused from mocks/
   - Inline handlers created
   - List of all handlers included

5. **Pattern Selected**:
   - Which pattern was chosen (1-4)
   - Why it was selected

6. **Pitfall Validation** (for --update):
   - Checklist categories validated
   - Issues found
   - Issues fixed

7. **Files Created/Updated**:
   - Story file paths
   - Any manual review needed

8. **Next Steps**:
   ```
   1. Run Storybook: npm run storybook --prefix micro-frontends/intg
   2. View at http://localhost:6006
   3. Review generated stories
   ```

**Example Output Format (with git-diff context):**
```
✓ Generated story for FlowDetailCard component

Git-Diff Context:
✓ Used git-diff pre-analysis
✓ API endpoints identified: useGetConnectorTypeById, useEnvironments
✓ Transformation documented: splitMITypesString("ibmmq-source") → "ibmmq"

Component Analysis:
- Type: Card with Selection (Medium Complexity)
- Dependencies: React Query, Jotai, Router
- Pattern: Pattern 3 (React Query + Jotai + Router)

API Endpoints:
- useGetConnectorTypeById → /microIntegrationTypes/ibmmq (after transformation)
- useEnvironments → /api/v2/platform/environments
- Transformation: splitMITypesString removes direction suffix

Handler Strategy:
- Imported: connectorEnvironments
- Inline: ibmmqConnectorType, servicebusConnectorType, awssqsConnectorType
- All handlers match actual endpoints (verified via git-diff)

Story Details:
- Created: src/stories/Multiflow/FlowDetailCard.stories.tsx
- Variants: 8 (RunningState, DeployingState, ErrorState, NotDeployedState, CompactMode, WithMetrics, WithActionMenu, NoDescription)
- Decorator: QueryClientProvider + JotaiProvider + MemoryRouter

Pitfall Validation:
✓ React import present
✓ Correct decorator stack
✓ MSW handlers match git-diff identified endpoints
✓ Appropriate timeouts
✓ All test patterns valid

Next Steps:
1. Run Storybook: npm run storybook --prefix micro-frontends/intg
2. View at http://localhost:6006
3. Navigate to Multiflow/FlowDetailCard
```

**Example Output Format (without git-diff):**
```
✓ Generated story for MiFlowMetrics component

Component Analysis:
- Type: Simple presentational with routing
- Dependencies: useHistory, useTheme
- Pattern: Pattern 2 (Routing Only)

API Endpoints:
- None (component makes no API calls)

Handler Strategy:
- No MSW handlers needed

Story Details:
- Created: src/stories/Multiflow/MiFlowMetrics.stories.tsx
- Variants: 8 (RunningState, ErrorState, DownState, DeployingState, NotDeployedState, LoadingState, WithCustomErrorLogCallback, WithContactSupportInteraction)
- Decorator: MemoryRouter + Route

Pitfall Validation:
✓ React import present
✓ Correct decorator stack
✓ Appropriate timeouts
✓ All test patterns valid

Next Steps:
1. Run Storybook: npm run storybook --prefix micro-frontends/intg
2. View at http://localhost:6006
3. Navigate to Multiflow/MiFlowMetrics
```

---

## Batch Processing

### Multiple Components (comma-separated paths):

1. Process each component independently using Phase 1-7
2. DO NOT ask for confirmation between components
3. Report summary at the end:
   ```
   Processing 3 components...
   ✓ Created src/stories/Agents/AgentCard.stories.tsx
   ✓ Created src/stories/Agents/AgentList.stories.tsx
   ✓ Created src/stories/Agents/AgentFilter.stories.tsx

   Summary: 3 stories created
   ```

### Directory Recursive (--recursive flag):

1. Use Glob to find all `.tsx` files in directory
2. Exclude: `*.stories.tsx`, `*.test.tsx`, `*.spec.tsx`
3. For each component:
   - Check if story exists
   - Generate if missing (or update if `--update` flag)
4. Report summary

### Page with Dependencies (--include-children flag):

1. Read the page component
2. Extract all imports from `src/components/`
3. For each child component:
   - Check if story exists
   - Generate if missing
4. Generate main page story with all dependencies mocked
5. Report summary

---

## Error Handling

- **Component file doesn't exist**: Report error, continue with next component
- **Story exists without --update**: Ask user whether to overwrite
- **Mock handlers can't be found**: Create inline handlers, note in output
- **Component too complex**: Generate basic story, note manual review needed
- **GUIDELINE.md not found**: Error and stop (cannot proceed without project patterns)

---

## Important Notes

### Git-Diff Integration (Fully Automatic)

**AUTOMATIC BEHAVIOR**: This skill now automatically invokes git-diff!

**Fully Automatic Mode (Recommended):**
```bash
# Just run with no arguments:
/storybook-generator

# Automatically:
# 1. Invokes /git-diff to analyze all changes
# 2. Identifies components missing stories
# 3. Shows you the list
# 4. Generates stories for all of them
```

**Manual Component Selection:**
```bash
# Specify component(s):
/storybook-generator src/components/Foo.tsx

# Automatically:
# 1. Checks if component is in recent changes
# 2. Invokes /git-diff if needed
# 3. Uses git-diff context to generate story
```

**Benefits:**
- ✅ Automatically traces API endpoints and transformations
- ✅ Identifies all React Query hooks and their actual endpoints
- ✅ Documents data transformation functions in generated stories
- ✅ Prevents 404 errors by ensuring correct MSW handler paths
- ✅ Speeds up story generation by skipping redundant analysis
- ✅ Works seamlessly for both single and multiple files
- ✅ No manual git-diff invocation needed

**When automatic git-diff is skipped (Manual Mode only):**
- Component(s) not in recent git changes
- Git-diff already run in this session
- No git repository detected

### Always Include

- `import React from "react"` in every story (even if TypeScript says unused)
- Comments explaining data transformations
- Appropriate timeouts for async operations (>= 3000ms)
- Both list and by-ID handlers when component uses both

### Never Do

- Use `data-testid` (use `data-qa` instead - configured in project)
- Skip pitfall validation for `--update`
- Assume list handlers cover by-ID endpoints (they don't!)
- Create stories without reading GUIDELINE.md
- Use placeholder values in tool calls - always wait for dependencies

### Reading GUIDELINE.md

The skill **MUST read GUIDELINE.md** at these critical points:
1. Phase 3: Learn all patterns
2. Phase 4: Learn MSW handler strategies and mock file locations
3. Phase 5: Learn component type decision matrix and pattern templates
4. Phase 6: Learn testing patterns and variant types
5. Phase 7: Learn complete pitfall validation checklist

**How to read efficiently**:
- Read entire file once in Phase 3 to understand overall structure
- Re-read specific sections as needed in later phases
- Use the table of contents in GUIDELINE.md to navigate

---

## Project-Specific Conventions

All project-specific information is in **GUIDELINE.md**, including:
- 4 Decorator Patterns (Pattern 1-4)
- Component Type Decision Matrix
- MSW Handler Strategies (Imported, Inline, Hybrid)
- Common Mock File Locations
- Testing Patterns & Assertions
- Pitfall Validation Checklist (7 categories)
- Template Examples
- Step-by-Step Process

**Do NOT hardcode these patterns in this file**. Always read from GUIDELINE.md during execution.

---

## Reusability

This SKILL.md is designed to be **reusable across all MFEs**:

1. Copy this file to any MFE's `.claude/skills/storybook-generator/`
2. Create/update GUIDELINE.md with MFE-specific patterns
3. The skill will adapt to each MFE's patterns automatically

**Example**:
```
micro-frontends/
├── intg/
│   └── .claude/skills/storybook-generator/
│       ├── SKILL.md (this file)
│       └── GUIDELINE.md (intg patterns)
├── ep/
│   └── .claude/skills/storybook-generator/
│       ├── SKILL.md (same file)
│       └── GUIDELINE.md (ep patterns - different!)
├── mc/
│   └── .claude/skills/storybook-generator/
│       ├── SKILL.md (same file)
│       └── GUIDELINE.md (mc patterns - different!)
```

---

## References

- **GUIDELINE.md**: Project-specific patterns and conventions (REQUIRED)
- Storybook docs: https://storybook.js.org/docs/react/writing-stories/introduction
- MSW docs: https://mswjs.io/docs/
- React Testing Library: https://testing-library.com/docs/react-testing-library/intro/
