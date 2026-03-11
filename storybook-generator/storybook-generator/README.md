# Storybook Generator Skill

A comprehensive Claude Code skill for automatically generating and updating Storybook stories for React components in the MaaS-UI Integration micro-frontend.

## Overview

This skill analyzes React components and generates properly structured `.stories.tsx` files following your project's established patterns and conventions. It handles everything from simple presentational components to complex page-level components with multi-step flows.

## Features

- **Single Component Story Generation**: Generate a complete story file for one component
- **Batch Processing**: Generate stories for multiple components at once
- **Directory Scanning**: Recursively process entire directories
- **Page-Level Stories**: Create comprehensive stories for complex page components with all dependencies
- **Story Updates**: Intelligently update existing stories when components change
- **Smart Dependency Detection**: Automatically identifies and includes:
  - React Query hooks and MSW handlers
  - Jotai atoms and state management
  - Context providers (UserProvider, ApplicationProvider, etc.)
  - Router requirements
  - i18n usage
- **Pattern Matching**: Follows your existing story patterns for consistency

## Installation

The skill is already installed in your project at:
```
/Users/angelapinelo/maas-ui/.claude/skills/storybook-generator/
```

Claude Code will automatically detect and load it when you start a session in the project directory.

## Usage

### Basic Usage

Generate a story for a single component:
```bash
/storybook-generator src/components/agents/AgentCard.tsx
```

### Multiple Components

Generate stories for multiple components in one command:
```bash
/storybook-generator src/components/agents/AgentCard.tsx src/components/agents/AgentList.tsx src/components/agents/AgentFilter.tsx
```

### Directory (Recursive)

Generate stories for all components in a directory:
```bash
/storybook-generator src/components/multiFlow/ --recursive
```

This will:
- Find all `.tsx` files (excluding `.stories.tsx`, `.test.tsx`, `.spec.tsx`)
- Generate a story for each component that doesn't have one
- Report a summary of created stories

### Page with Dependencies

Generate a comprehensive story for a page component and all its child components:
```bash
/storybook-generator src/pages/multiFlow/CreateChildMI.tsx --include-children
```

This will:
- Analyze the page component
- Identify all child components imported from `src/components/`
- Generate stories for child components that don't have them
- Create a comprehensive page-level story with all dependencies mocked

### Update Existing Story

Update an existing story when the component has changed:
```bash
/storybook-generator src/components/connectors/ConnectorCard.tsx --update
```

This will:
- Read the existing story
- Compare with the current component
- Add new story variants for new props
- Update existing variants
- Add new decorators/handlers if new dependencies detected
- Preserve existing test logic

## Story Types Generated

The skill automatically detects component types and generates appropriate stories:

### 1. Simple Component Stories
For basic presentational components:
- Default variant
- Variants for different prop combinations
- Basic interaction tests

### 2. Component with State Management (Jotai)
For components using Jotai atoms:
- JotaiProvider decorator
- Atom reset logic
- State-dependent variants

### 3. Component with React Query
For components that fetch data:
- QueryClientProvider decorator
- Configured QueryClient with test-friendly defaults
- MSW handlers for API mocking
- Loading/Error state variants

### 4. Component with Full Context Stack
For complex components requiring multiple providers:
- UserProvider (with permissions)
- ApplicationProvider (with environment context)
- JotaiProvider
- QueryClientProvider
- MemoryRouter for routing
- All necessary MSW handlers

### 5. Dialog/Modal Stories
For dialog components:
- Dialog wrapper with proper styling
- User interaction tests
- Validation tests
- Multiple state variants

### 6. Page/Stepper Stories
For complex page components:
- Full provider stack
- Multi-step navigation tests
- Breadcrumb verification
- Form interaction tests

## Generated Story Features

Each generated story includes:

1. **Proper Imports**: All necessary dependencies and types
2. **Meta Configuration**: Correct title, component reference, and decorators
3. **Type Safety**: Proper TypeScript types for Story objects
4. **MSW Handlers**: Reuses existing mocks from `mocks/` directory
5. **Multiple Variants**: Different states and use cases
6. **Comprehensive Tests**: Using `play` functions with:
   - Element existence checks
   - Visibility assertions
   - User interaction simulations
   - Async operation handling
   - Proper timeouts for data loading

## Examples

### Example 1: Generate Story for New Component

```bash
/storybook-generator src/components/agents/NewAgentBadge.tsx
```

Output:
```
✓ Analyzing component: src/components/agents/NewAgentBadge.tsx
✓ Component type detected: Simple presentational component
✓ Found existing mocks: agentsMock, organizationsMock
✓ Generated story: src/stories/Agents/NewAgentBadge.stories.tsx

Story includes:
- Default variant
- WithStatus variant (for each status type)
- Basic interaction tests

Next steps:
1. Run Storybook: npm run storybook --prefix micro-frontends/intg
2. View story at http://localhost:6006
```

### Example 2: Batch Generate Stories

```bash
/storybook-generator src/components/multiFlow/FlowCard.tsx src/components/multiFlow/FlowList.tsx src/components/multiFlow/FlowFilters.tsx
```

Output:
```
Processing 3 components...

✓ Created src/stories/Multiflow/FlowCard.stories.tsx
✓ Created src/stories/Multiflow/FlowList.stories.tsx
✓ Created src/stories/Multiflow/FlowFilters.stories.tsx

Summary: 3 stories created successfully

Next steps:
1. Run Storybook: npm run storybook --prefix micro-frontends/intg
2. Review generated stories
```

### Example 3: Update Existing Story

```bash
/storybook-generator src/components/agents/AgentCard.tsx --update
```

Output:
```
Analyzing component: src/components/agents/AgentCard.tsx
Found existing story: src/stories/Agents/AgentCard.stories.tsx

Changes detected:
- New prop: 'onDelete' (function)
- New prop: 'isSelectable' (boolean)
- New hook: useDeleteAgent (requires MSW handler)

Updates made:
✓ Added 'WithDeleteAction' story variant
✓ Added 'WithSelection' story variant
✓ Added deleteAgentHandler to MSW handlers
✓ Updated Default story with new props

Story updated successfully!
```

### Example 4: Page with Child Components

```bash
/storybook-generator src/pages/agents/CreateAgent.tsx --include-children
```

Output:
```
Analyzing page: src/pages/agents/CreateAgent.tsx

Child components found:
- src/components/agents/AgentDetailsForm.tsx (story exists)
- src/components/agents/AgentTypeSelector.tsx (no story)
- src/components/agents/AgentValidation.tsx (no story)

Actions:
✓ Created src/stories/Agents/AgentTypeSelector.stories.tsx
✓ Created src/stories/Agents/AgentValidation.stories.tsx
✓ Created src/stories/Pages/CreateAgent.stories.tsx (page-level story)

Summary: 3 stories created

The page-level story includes:
- Full provider stack
- All MSW handlers for the entire flow
- Stepper navigation tests
- Form validation tests
```

### Example 5: Recursive Directory Processing

```bash
/storybook-generator src/components/filters/ --recursive
```

Output:
```
Scanning directory: src/components/filters/

Found 8 components:
- DefaultFilterMultiSelect.tsx (no story)
- EmptyFilterPanel.tsx (story exists, skipping)
- SearchPanelV2.tsx (no story)
- FilterUtils.tsx (utility file, skipping)
- filterChips/MultiSelectFilterChip.tsx (no story)
- filterChips/SingleSelectFilterChip.tsx (no story)

Processing 4 components...

✓ Created src/stories/Filters/DefaultFilterMultiSelect.stories.tsx
✓ Created src/stories/Filters/SearchPanelV2.stories.tsx
✓ Created src/stories/Filters/MultiSelectFilterChip.stories.tsx
✓ Created src/stories/Filters/SingleSelectFilterChip.stories.tsx

Summary: 4 stories created, 1 skipped (already exists), 3 utility files skipped
```

## Flags and Options

### `--update`
Update an existing story file instead of creating a new one. Intelligently merges changes.

**Example:**
```bash
/storybook-generator src/components/agents/AgentCard.tsx --update
```

### `--recursive`
Process all components in a directory and its subdirectories.

**Example:**
```bash
/storybook-generator src/components/agents/ --recursive
```

### `--include-children`
For page components, also generate stories for all child components that don't have them.

**Example:**
```bash
/storybook-generator src/pages/multiFlow/CreateChildMI.tsx --include-children
```

### Combining Flags

```bash
# Update all stories in a directory
/storybook-generator src/components/connectors/ --recursive --update

# Generate page story with all child dependencies
/storybook-generator src/pages/agents/CreateAgent.tsx --include-children
```

## How It Works

### 1. Component Analysis

The skill reads your component file and identifies:
- Component name and type (default/named export)
- Props interface and types
- React hooks used (useState, useEffect, useQuery, useMutation, etc.)
- Jotai atoms imported
- Context providers required
- Router dependencies
- i18n usage
- API calls and endpoints

### 2. Template Selection

Based on component complexity, it selects the appropriate template:
- Simple component → Basic template
- Uses Jotai → Add JotaiProvider
- Uses React Query → Add QueryClientProvider with test config
- Uses contexts → Add UserProvider, ApplicationProvider
- Uses router → Add MemoryRouter
- Dialog/Modal → Add dialog-specific wrapper
- Page/Stepper → Add full page template with navigation

### 3. Mock Handler Discovery

The skill searches the `mocks/` directory for existing MSW handlers:
- Matches by domain (agents, connectors, organizations, etc.)
- Reuses existing handlers when available
- Creates new handlers only when necessary
- Suggests creating shared mock files for reusable handlers

### 4. Story Variant Generation

Generates multiple story variants based on component type:
- **Basic variants**: Default, WithProps, Loading, Error, Empty
- **Interactive variants**: WithUserInteraction, WithValidation, WithSelection
- **State variants**: For each significant component state
- **Data variants**: WithData, WithPagination, WithSorting, WithFiltering

### 5. Test Generation

Creates comprehensive tests using Storybook's `play` function:
- Element existence and visibility checks
- User interaction simulations (clicks, typing, selections)
- Async operation handling with proper timeouts
- Form validation tests
- Navigation tests for multi-step flows
- Proper use of `data-qa` attributes (not `data-testid`)

## Project-Specific Patterns

The skill follows your project's conventions:

### Testing with data-qa
Your project uses `data-qa` instead of `data-testid`:

```typescript
// Component
<SolaceButton dataQa="submitButton" onClick={handleSubmit}>
  Submit
</SolaceButton>

// Generated test
const button = await screen.findByTestId("submitButton"); // Queries data-qa="submitButton"
```

### React Query v5
Generates test-friendly QueryClient configuration:

```typescript
const queryClient = new QueryClient({
	defaultOptions: {
		queries: {
			retry: false,
			refetchOnWindowFocus: false,
			refetchOnMount: false,
			refetchOnReconnect: false,
			gcTime: 0,
			staleTime: Infinity
		}
	}
});
```

### Mock Reuse
Always searches for and reuses existing mocks:

```typescript
import { agentsMock } from "mocks/agentsMock";
import { connectorEnvironments } from "mocks/connectorEnvironmentsMock";
import { organizationsMock } from "mocks/organizationsMock";
```

## Troubleshooting

### Story Already Exists

If a story already exists and you run the command without `--update`:

```
✗ Story already exists: src/stories/Agents/AgentCard.stories.tsx
  Use --update flag to update it, or delete it first.
```

**Solution:** Use `--update` flag or delete the existing story.

### Component Not Found

```
✗ Component not found: src/components/agents/MissingComponent.tsx
```

**Solution:** Check the file path and ensure it exists.

### Missing Mock Handlers

If the skill can't find appropriate mock handlers:

```
⚠ Warning: Could not find existing mock handlers for API endpoint: /api/v2/agents
  Generated basic handler. Consider creating a shared mock in mocks/ directory.
```

**Action:** Review the generated handler and consider moving it to a shared mock file if it will be reused.

### Complex Component Needs Manual Review

```
⚠ Component is highly complex with multiple dependencies.
  Generated basic story structure. Manual review and enhancement recommended.
```

**Action:** Review the generated story and add additional test cases as needed.

## Best Practices

1. **Run After Component Creation**: Generate stories right after creating or significantly modifying a component
2. **Review Generated Tests**: While tests are comprehensive, review them for component-specific edge cases
3. **Reuse Mocks**: If you create new mock handlers, consider moving them to the `mocks/` directory for reuse
4. **Update Regularly**: Use `--update` flag when component props or behavior changes
5. **Batch Process**: Use recursive mode for new component directories to ensure full coverage

## Integration with Your Workflow

### After Creating a New Component

```bash
# 1. Create component
# 2. Generate story
/storybook-generator src/components/newFeature/NewComponent.tsx

# 3. Run Storybook
npm run storybook --prefix micro-frontends/intg

# 4. View and test the story
# 5. Commit both component and story
```

### After Refactoring

```bash
# Update stories for all refactored components
/storybook-generator src/components/refactoredModule/ --recursive --update
```

### Before Creating a PR

```bash
# Ensure all components have stories
/storybook-generator src/components/ --recursive

# Review the summary to see which components got new stories
```

## Technical Details

### File Locations

- **Skill Location**: `.claude/skills/storybook-generator/SKILL.md`
- **Generated Stories**: `micro-frontends/intg/src/stories/`
- **Component Location**: `micro-frontends/intg/src/components/`
- **Mock Handlers**: `micro-frontends/intg/mocks/`

### Supported Component Types

- React functional components (default export)
- React functional components (named export)
- TypeScript components (.tsx)
- Components with hooks (useState, useEffect, useQuery, useMutation, etc.)
- Components with Jotai atoms
- Components with context consumers
- Page components with routing
- Dialog/Modal components
- Form components
- Table components
- Stepper/Wizard components

### Excluded Files

The skill automatically skips:
- Test files (`.test.tsx`, `.spec.tsx`)
- Story files (`.stories.tsx`)
- Type definition files (`.d.ts`)
- Utility files without React components
- Index files that only re-export

## Feedback and Improvements

If you encounter issues or have suggestions for improvements, you can:

1. Modify the skill at `.claude/skills/storybook-generator/SKILL.md`
2. Add new templates or patterns
3. Update mock handler discovery logic
4. Enhance test generation for specific component types

The skill is designed to be extensible and can be customized to match your evolving project patterns.

## Version History

### v1.0.0 (Current)
- Initial release
- Support for all component types
- Batch processing
- Recursive directory scanning
- Story updates
- Smart dependency detection
- Comprehensive test generation
- MSW handler management

---

**Happy Story Generating! 🎨📚**

For questions or issues, refer to the SKILL.md file for detailed implementation instructions.
