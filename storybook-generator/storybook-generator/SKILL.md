---
name: storybook-generator
description: Generate or update Storybook stories for React components. Handles single components, multiple components, page-level stories with complex flows, and batch operations. Use when user wants to create or update .stories.tsx files.
---

# Storybook Story Generator

This skill generates and updates Storybook stories for React components in the MaaS-UI Integration micro-frontend, following established patterns and conventions.

## Core Responsibilities

1. **Analyze Components**: Extract props, dependencies, contexts, and state management requirements
2. **Generate Stories**: Create properly structured .stories.tsx files with appropriate decorators, MSW handlers, and tests
3. **Handle Multiple Components**: Process single components, multiple components in batch, or entire directories
4. **Page-Level Stories**: Generate comprehensive stories for complex page components with multi-step flows
5. **Update Existing Stories**: Intelligently update stories when components change

## Usage Patterns

### Single Component
```
/storybook-generator src/components/agents/AgentCard.tsx
```

### Multiple Components
```
/storybook-generator src/components/agents/AgentCard.tsx src/components/agents/AgentList.tsx src/components/agents/AgentFilter.tsx
```

### Page with Dependencies
```
/storybook-generator src/pages/multiFlow/CreateChildMI.tsx --include-children
```

### Directory (Recursive)
```
/storybook-generator src/components/multiFlow/ --recursive
```

### Update Existing Story
```
/storybook-generator src/components/connectors/ConnectorCard.tsx --update
```

## Component Analysis Process

For each component, analyze:

1. **File Structure**:
   - Read the component file
   - Identify component type (simple component, dialog, page, stepper, table, etc.)
   - Extract component name and props interface
   - Identify default export vs named export

2. **Dependencies**:
   - React Query hooks (`useQuery`, `useMutation`)
   - Jotai atoms (state management)
   - Context usage (`UserProvider`, `ApplicationProvider`, etc.)
   - Router usage (`MemoryRouter`, `Route`, navigation)
   - Form libraries (react-hook-form, RJSF)
   - i18n usage

3. **API Calls**:
   - Identify API endpoints being called
   - Find matching mock handlers in `mocks/` directory
   - Create new MSW handlers if needed

4. **Child Components**:
   - For page-level stories, identify all imported components
   - Determine which need stories of their own

## Understanding the Provider Hierarchy

### Global Decorators (Already Provided)

From `.storybook/preview.tsx`, **every story automatically receives these providers**:

```
ThemeProvider (Solace Design System)
  → EnvironmentProvider
    → UserProvider (default permissions)
      → SnackbarProvider (notistack)
        → ReactQueryProvider (QueryClient)
          → Suspense
            → Story
```

**IMPORTANT**: You do NOT need to add these providers in story-level decorators. Only add decorators when you need to:
- **Override** default values (e.g., custom permissions in UserProvider)
- **Add** additional context not in global stack (e.g., MemoryRouter, JotaiProvider, ApplicationProvider)

### When to Add Story-Level Decorators

Use this decision tree:

| Component Uses | Add Decorator |
|----------------|---------------|
| React Router hooks (`useHistory`, `useLocation`, `Link`) | MemoryRouter + Route |
| `useQuery`, `useMutation` with custom QueryClient config | QueryClientProvider (with strict config) |
| Jotai atoms | JotaiProvider |
| `useApplicationContext` | ApplicationProvider |
| Custom permissions beyond defaults | UserProvider (override) |
| Location state (stepper pages) | Route with `location` prop |

## Component Type Decision Matrix

Use this matrix to quickly determine story complexity and requirements:

| Component Type | Decorators | MSW Handlers | State Management | Example |
|----------------|-----------|--------------|------------------|---------|
| **Pure Presentational** | None | No | No | MatchedChip, Badge |
| **With Routing** | MemoryRouter | No | No | NavigationLink, Breadcrumb |
| **List/Table** | Router + Jotai | Yes | Jotai atoms | AgentTable, ConnectorList |
| **Card with Selection** | QueryClient + Jotai + Router | Yes | Jotai atoms | FlowDetailCard, AgentCard |
| **Dialog/Modal** | QueryClient + Jotai + Router | Yes | Jotai atoms | CreateDialog, ConfirmDialog |
| **Form Page** | Full Stack | Yes | Multiple atoms | CreateConnector, EditAgent |
| **Stepper** | Full Stack + Location State | Yes (8+ handlers) | Multiple atoms | CreateMiParentStepper |

### Component Analysis Checklist

Before writing story, analyze component to determine:

- [ ] Does it use React Router hooks? → Add MemoryRouter
- [ ] Does it call APIs? → Add QueryClientProvider + MSW handlers
- [ ] Does it use Jotai atoms? → Add JotaiProvider + wrapper for reset
- [ ] Is it a page-level component? → Add full context stack
- [ ] Does it use location.state? → Add Route with location prop
- [ ] Does it have complex multi-step flow? → Plan sequential play() function

## Story Generation Templates - The 4 Decorator Patterns

### Pattern 1: Simple Presentational Component (NO decorators)

**When to use**: Pure presentational components with no routing, no API calls, no complex state.

**Example components**: MatchedChip, StatusBadge, IconButton

```typescript
import type { Meta, StoryObj } from "@storybook/react-webpack5";
import ComponentName from "components/path/to/ComponentName";
import React from "react";
import { expect, waitFor, within } from "storybook/test";

const meta: Meta<typeof ComponentName> = {
	title: "Category/ComponentName",
	component: ComponentName
	// NO decorators - uses global stack only
};

export default meta;

type Story = StoryObj<typeof ComponentName>;

export const Default: Story = {
	args: {
		// Component props
	},
	play: async ({ canvasElement }) => {
		const canvas = within(canvasElement);

		await waitFor(() => {
			expect(canvas.getByText("Expected Text")).toBeInTheDocument();
		});
	}
};
```

### Pattern 2: Routing Only

**When to use**: Component uses `useHistory`, `useLocation`, `Link`, or other React Router hooks.

**Example components**: NavigationButton, BreadcrumbPath

```typescript
import type { Meta, StoryObj } from "@storybook/react-webpack5";
import ComponentName from "components/path/to/ComponentName";
import React from "react";
import { MemoryRouter, Route } from "react-router-dom";
import { expect, waitFor, within } from "storybook/test";

const meta: Meta<typeof ComponentName> = {
	title: "Category/ComponentName",
	component: ComponentName,
	decorators: [
		(Story) => (
			<MemoryRouter>
				<Route>
					<div style={{ height: "200vh" }}>
						<Story />
					</div>
				</Route>
			</MemoryRouter>
		)
	]
};

export default meta;

type Story = StoryObj<typeof ComponentName>;

export const Default: Story = {
	args: {},
	play: async ({ canvasElement }) => {
		const canvas = within(canvasElement);
		// Tests here
	}
};
```

### Pattern 3: React Query + Jotai (State Management)

**When to use**: Component uses `useQuery`, `useMutation`, or Jotai atoms.

**Example components**: FlowDetailCard, AgentCard, ConnectorList

**CRITICAL for React Query v5**: QueryClient configuration prevents aggressive re-renders during story interactions.

**CRITICAL for Jotai**: Wrapper component resets atoms between stories.

```typescript
import type { Meta, StoryObj } from "@storybook/react-webpack5";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { selectedConnectorRows } from "components/filters/filterAtoms";
import ComponentName from "components/path/to/ComponentName";
import { Provider as JotaiProvider, useSetAtom } from "jotai";
import { mockHandlers } from "mocks/mockFile";
import React, { useEffect } from "react";
import { MemoryRouter, Route } from "react-router-dom";
import { expect, waitFor, within } from "storybook/test";

const queryClient = new QueryClient({
	defaultOptions: {
		queries: {
			retry: false,
			refetchOnWindowFocus: false,
			refetchOnMount: false,
			refetchOnReconnect: false,
			gcTime: 0,
			staleTime: Infinity  // Prevents automatic refetching
		}
	}
});

const meta: Meta<typeof ComponentName> = {
	title: "Category/ComponentName",
	component: ComponentName,
	decorators: [
		(Story) => (
			<QueryClientProvider client={queryClient}>
				<JotaiProvider>
					<MemoryRouter>
						<Route>
							<div style={{ height: "100vh", padding: "16px" }}>
								<Story />
							</div>
						</Route>
					</MemoryRouter>
				</JotaiProvider>
			</QueryClientProvider>
		)
	]
};

export default meta;

type Story = StoryObj<typeof ComponentName>;

// Wrapper component to reset atom state between stories
const ComponentWrapper = ({ ...props }) => {
	const setSelectedRowIds = useSetAtom(selectedConnectorRows);

	useEffect(() => {
		setSelectedRowIds([]);  // Reset to empty on mount
	}, [setSelectedRowIds]);

	return <ComponentName {...props} />;
};

export const Default: Story = {
	parameters: {
		msw: {
			handlers: [
				...mockHandlers
			]
		}
	},
	render: (args) => <ComponentWrapper {...args} />,
	play: async ({ canvasElement }) => {
		const canvas = within(canvasElement);

		await waitFor(() => {
			expect(canvas.getByText("Loaded Content")).toBeInTheDocument();
		});
	}
};
```

### Pattern 4: Full Context Stack (Page-Level Components)

**When to use**: Complex page components, steppers, multi-step flows that need full application context.

**Example components**: CreateMiParentStepper, FormPages with navigation

```typescript
import type { Meta, StoryObj } from "@storybook/react-webpack5";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import ComponentName from "components/path/to/ComponentName";
import { UserProvider } from "contexts";
import { ApplicationProvider } from "contexts/ApplicationContext";
import { Provider as JotaiProvider } from "jotai";
import { mockHandlers } from "mocks/mockFile";
import React from "react";
import { MemoryRouter, Route } from "react-router-dom";
import { expect, waitFor, within } from "storybook/test";

// Mock ApplicationContext data
const mockApplicationContext = {
	isFreeTrial: false,
	isSAP: false,
	displayAuthErrorMessage: () => {},
	track: () => {},
	selectedEnvironment: {
		environmentId: "67vkrnseeuh",
		environmentName: "Test Environment",
		showAllResources: false,
		orgId: "test-org"
	},
	setSelectedEnvironment: () => {}
};

// Mock user context for permissions
const userContextValue = {
	user: {
		userId: "currentUser",
		email: "test@example.com"
	},
	permissions: ["micro_integrations:*:*"]
};

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

const meta: Meta<typeof ComponentName> = {
	title: "Pages/ComponentName",
	component: ComponentName,
	decorators: [
		(Story) => (
			<QueryClientProvider client={queryClient}>
				<UserProvider value={userContextValue}>
					<JotaiProvider>
						<ApplicationProvider value={mockApplicationContext}>
							<MemoryRouter initialEntries={["/path"]}>
								<Route
									path="/path"
									location={{
										state: { connectorTypeId: "awssqs-source" },
										pathname: "/path",
										search: "",
										hash: "",
										key: ""
									}}
								>
									<div style={{ height: "100vh" }}>
										<Story />
									</div>
								</Route>
							</MemoryRouter>
						</ApplicationProvider>
					</JotaiProvider>
				</UserProvider>
			</QueryClientProvider>
		)
	]
};

export default meta;

type Story = StoryObj<typeof ComponentName>;

export const Default: Story = {
	parameters: {
		msw: {
			handlers: [...mockHandlers]
		}
	}
};
```

## Story Variant Generation

For each component, generate multiple story variants based on component type:

### Basic Component Variants
- `Default`: Default state with minimal props
- `WithProps`: Various prop combinations
- `Loading`: Loading state (if applicable)
- `Error`: Error state (if applicable)
- `Empty`: Empty/no data state (if applicable)

### Interactive Component Variants
- `WithUserInteraction`: Tests user interactions (clicks, typing, etc.)
- `WithValidation`: Tests form validation
- `WithSelection`: Tests row/item selection

### Data Component Variants
- `WithData`: Fully populated with data
- `WithPagination`: Tests pagination functionality
- `WithSorting`: Tests sorting functionality
- `WithFiltering`: Tests filtering functionality

### State-Based Component Variants
For components with multiple states (e.g., status badges):
- Create a variant for each significant state
- Example: `NotDeployedState`, `RunningState`, `ErrorState`, `DeployingState`

## MSW Handler Management

### Handler Organization Strategies

Choose the right strategy based on your needs:

#### Strategy 1: Imported Handlers (Reusable)

**When to use**: Standard API endpoints used across multiple stories.

```typescript
// Import from mocks directory
import { agentsMock } from "mocks/agentsMock";
import { connectorEnvironments } from "mocks/connectorEnvironmentsMock";
import { organizationsMock } from "mocks/organizationsMock";

parameters: {
	msw: {
		handlers: [
			...agentsMock,                // Spread array of handlers
			...connectorEnvironments,
			...organizationsMock
		]
	}
}
```

**Common mock files**:
- `agentsMock.ts` - `/api/v2/integration/agents`
- `connectorEnvironmentsMock.ts` - `/api/v2/platform/environments`
- `connectorTypes.ts` - `/api/v2/integration/microIntegrationTypes` (list)
- `organizationsMock.ts` - `/api/v2/organizations`
- `users.ts` - `/api/v2/users`
- `serviceDetails.ts` - `/api/v2/missionControl/eventBrokerServices/{id}`

#### Strategy 2: Inline Handlers (Story-Specific)

**When to use**:
- Story-specific endpoint variations
- Testing different API responses
- Showcasing endpoint structure for documentation
- Data transformations require specific mock shapes

```typescript
// Define handlers directly in story file
const ibmmqConnectorType = http.get(
	"/api/v2/integration/microIntegrationTypes/ibmmq",
	() => {
		return new Response(JSON.stringify({
			data: {
				id: "ibmmq-source",
				name: "IBM MQ",
				microIntegrationType: "ibmmq",
				direction: "source"
			}
		}));
	}
);

parameters: {
	msw: {
		handlers: [
			ibmmqConnectorType,          // Inline handler
			awssnsConnectorType          // Inline handler
		]
	}
}
```

#### Strategy 3: Hybrid Approach (RECOMMENDED)

**Why hybrid**: Reuse common handlers + customize specific endpoints

```typescript
parameters: {
	msw: {
		handlers: [
			...connectorTypes,              // Imported: List endpoint
			...connectorEnvironments,       // Imported: Standard
			ibmmqConnectorType,             // Inline: By-ID endpoint
			servicebusConnectorType,        // Inline: By-ID endpoint
			kafkaConnectorType              // Inline: By-ID endpoint
		]
	}
}
```

### Critical Handler Patterns

#### List vs By-ID Endpoints (COMMON MISTAKE)

**Critical distinction**: List endpoints and by-ID endpoints are SEPARATE handlers!

```typescript
// List endpoint (returns array)
http.get("/api/v2/integration/microIntegrationTypes", () => {
	return new Response(JSON.stringify({
		data: [...]  // Array of types
	}));
});

// By-ID endpoint (returns single object) - SEPARATE HANDLER NEEDED
http.get("/api/v2/integration/microIntegrationTypes/:id", ({ params }) => {
	return new Response(JSON.stringify({
		data: {...}  // Single type
	}));
});
```

**Common mistake**: Assuming list handlers cover by-ID calls. They don't!

If component uses:
- `useGetConnectorTypes()` → needs list endpoint handler
- `useGetConnectorTypeById(id)` → needs by-ID endpoint handler
- **Both** → needs **BOTH handlers**

#### Data Transformation Handlers

When component transforms data before API call:

```typescript
// Component code:
const { data: connectorType } = useGetConnectorTypeById(
	splitMITypesString(flow.microIntegrationType + "-" + flow.direction)
);

// splitMITypesString("ibmmq-source") → "ibmmq" (removes direction)

// Handler MUST match transformed value:
const ibmmqConnectorType = http.get(
	"/api/v2/integration/microIntegrationTypes/ibmmq",  // NOT "ibmmq-source"
	() => {
		return new Response(JSON.stringify({
			data: {
				id: "ibmmq-source",  // Can return full ID in response
				name: "IBM MQ",
				microIntegrationType: "ibmmq"
			}
		}));
	}
);
```

**Process**:
1. Trace hook to find endpoint pattern
2. Identify transformation functions
3. Test transformation with sample data
4. Mock the ACTUAL endpoint called (after transformation)

#### Error Response Handlers

```typescript
http.post("/api/v2/integration/microIntegrationsV2", () => {
	return HttpResponse.json(
		{
			message: "Bad Request",
			errorId: "test-error-id",
			meta: {
				microIntegrationV2: [
					{
						name: ["must be unique."]
					}
				]
			}
		},
		{ status: 400 }
	);
});
```

**When to use**: Testing error states, validation failures, network errors.

## Advanced API Call Tracing (CRITICAL)

### Problem: Missing 404 Errors from Untraced API Calls

One of the most common failures in generated stories is **404 errors from API endpoints that weren't properly mocked**. This happens when the skill doesn't fully trace through:
- React Query hooks to their endpoint definitions
- Data transformation functions that modify IDs/paths before API calls
- Dynamic endpoint construction

### Required Deep Analysis Process

When analyzing a component with React Query hooks, follow this **mandatory deep-dive workflow**:

#### Step 1: Identify All Hooks
```typescript
// Component uses:
const { data: connectorType } = useGetConnectorTypeById(
  splitMITypesString(flow.microIntegrationType + "-" + flow.direction)
);
```

Extract:
- Hook name: `useGetConnectorTypeById`
- Arguments: `splitMITypesString(flow.microIntegrationType + "-" + flow.direction)`

#### Step 2: Find Hook Definition
Use Grep to locate the hook implementation:
```bash
grep -r "export.*useGetConnectorTypeById" src/api
```

Read the hook file to extract:
- API endpoint pattern
- Query key structure
- Any transformations applied to parameters

Example hook definition:
```typescript
export function useGetConnectorTypeById(connectorTypeId?: string) {
  return useQuery({
    queryKey: ["getConnectorTypeById", connectorTypeId],
    queryFn: () =>
      axios.get(`/api/v2/integration/microIntegrationTypes/${connectorTypeId}`)
  });
}
```

Endpoint pattern: `/api/v2/integration/microIntegrationTypes/${connectorTypeId}`

#### Step 3: Trace Data Transformations
If the hook receives a function call as an argument, **trace that function**:

```typescript
// Component calls: splitMITypesString(flow.microIntegrationType + "-" + flow.direction)
// Find this function:
```

Use Grep/Read to find `splitMITypesString`:
```typescript
export function splitMITypesString(miType: string) {
  return miType.split("-").slice(0, -1).join("-");
}
```

**Test the transformation**:
- Input: `"ibmmq-source"`
- Output: `"ibmmq"` (direction suffix removed!)

#### Step 4: Determine Actual API Call
Combine endpoint pattern + transformed value:
- Endpoint: `/api/v2/integration/microIntegrationTypes/${connectorTypeId}`
- After transformation: `connectorTypeId = "ibmmq"` (NOT `"ibmmq-source"`)
- **Actual API call**: `/api/v2/integration/microIntegrationTypes/ibmmq`

#### Step 5: Find or Create Matching Handler
Search for handlers matching the **actual endpoint**:

```typescript
// WRONG - Handler for untransformed value:
http.get("/api/v2/integration/microIntegrationTypes/ibmmq-source", ...)

// CORRECT - Handler for transformed value:
http.get("/api/v2/integration/microIntegrationTypes/ibmmq", ...)
```

### Common Transformation Patterns to Watch For

1. **String manipulation**:
   - `split()`, `slice()`, `join()`
   - `replace()`, `substring()`
   - `toLowerCase()`, `toUpperCase()`

2. **ID extraction**:
   - `id.split('/').pop()`
   - `path.split('.').slice(-1)[0]`

3. **Path construction**:
   - Template literals with dynamic parts
   - URL encoding/decoding

### Learning from Similar Components

Before creating new handlers, **ALWAYS** search for similar components:

1. **Find components using the same hook**:
```bash
grep -r "useGetConnectorTypeById" src/components
```

2. **Check their story files**:
```bash
# For each component found, check if it has a story:
find src/stories -name "ComponentName.stories.tsx"
```

3. **Analyze working stories**:
- Read the MSW handlers they use
- Check if they handle the same transformed endpoints
- **Reuse their exact handler patterns**

Example:
```typescript
// Found: ListMICard.stories.tsx successfully handles the same hook
// It uses: ibmmqConnectorTypev218 from mocks/connectorTypesAndCategories
// This handles: /api/v2/integration/microIntegrationTypes/ibmmq-source AND ibmmq-target
// BUT we need: /api/v2/integration/microIntegrationTypes/ibmmq (base type!)
```

### Handler Path Validation Checklist

Before finalizing MSW handlers, verify:

- [ ] Handler path **exactly matches** the API call after all transformations
- [ ] Handler covers all possible transformed values (e.g., if direction varies, handle all directions)
- [ ] Handler response structure matches what the hook expects
- [ ] Handler is included in the story's `parameters.msw.handlers` array

### Distinguishing List vs By-ID Endpoints

**Critical distinction**: List endpoints and by-ID endpoints are different!

```typescript
// List endpoint - returns array
http.get("/api/v2/integration/microIntegrationTypes", () => {
  return { data: [...] }  // Array of types
});

// By-ID endpoint - returns single object
http.get("/api/v2/integration/microIntegrationTypes/:id", ({ params }) => {
  return { data: {...} }  // Single type matching ID
});
```

**Common mistake**: Assuming list handlers cover by-ID calls. They don't!

If component uses:
- `useGetConnectorTypes()` → needs list endpoint
- `useGetConnectorTypeById(id)` → needs by-ID endpoint
- **Both** → needs **both handlers**

### When to Create Inline vs Shared Handlers

**Create inline handlers** in the story when:
- The endpoint is only used by this one component
- The transformation is unique to this use case
- Quick prototyping/testing

**Create shared mock file** when:
- Multiple components use the same endpoint
- The handler will be reused across stories
- Complex response structures

Example inline handler with transformation note:
```typescript
// MSW handler for base microIntegrationType (without direction suffix)
// Note: splitMITypesString removes the direction, so "ibmmq-source" becomes "ibmmq"
const ibmmqConnectorType = http.get("/api/v2/integration/microIntegrationTypes/ibmmq", () => {
  return new Response(
    JSON.stringify({
      data: {
        id: "ibmmq-source",
        name: "IBM MQ",
        microIntegrationType: "ibmmq",
        direction: "source",
        // ... rest of data
      }
    })
  );
});
```

### Documentation Requirements

Every generated story with API call tracing should include:

1. **Comment explaining transformations**:
```typescript
// NOTE: useGetConnectorTypeById receives splitMITypesString(flow.microIntegrationType + "-" + flow.direction)
// splitMITypesString("ibmmq-source") returns "ibmmq" (direction suffix removed)
// Therefore handlers must match base types like /ibmmq, not /ibmmq-source
```

2. **Reference to transformation function**:
```typescript
// See: src/components/multiFlow/utils/ListMIUtils.tsx - splitMITypesString()
```

3. **List of covered endpoints**:
```typescript
// This story mocks the following endpoints:
// - /api/v2/integration/microIntegrationTypes (list)
// - /api/v2/integration/microIntegrationTypes/ibmmq (by-ID, base type)
// - /api/v2/integration/microIntegrationTypes/awssns (by-ID, base type)
// - /api/v2/platform/environments
```

## Test Assertions

### Common Assertions

```typescript
// Element exists
expect(canvas.getByText("Text")).toBeInTheDocument();
expect(canvas.getByTestId("element-id")).toBeInTheDocument();

// Element visible
expect(canvas.getByText("Text")).toBeVisible();

// Element enabled/disabled
expect(button).toBeEnabled();
expect(button).toBeDisabled();

// Element has attribute
expect(link).toHaveAttribute("href", "/path");

// Element count
expect(canvas.getAllByText("Item")).toHaveLength(3);

// Wait for async operations
await waitFor(() => {
	expect(canvas.getByText("Loaded")).toBeInTheDocument();
}, { timeout: 5000 });

// Find element (async)
const element = await canvas.findByText("Text", {}, { timeout: 5000 });
```

### Testing User Interactions

```typescript
import { userEvent } from "storybook/test";

// Click
await userEvent.click(button);

// Type
await userEvent.type(input, "text to type");

// Select dropdown
const dropdown = canvas.getByLabelText("Label");
await userEvent.click(dropdown);
const option = screen.getByRole("option", { name: "Option 1" });
await userEvent.click(option);

// Hover
await userEvent.hover(element);
```

### React Query v5 Considerations

When testing components with React Query:

1. Configure QueryClient for testing:
```typescript
const queryClient = new QueryClient({
	defaultOptions: {
		queries: {
			retry: false,
			refetchOnWindowFocus: false,
			refetchOnMount: false,
			refetchOnReconnect: false,
			gcTime: 0,
			staleTime: Infinity  // Prevent automatic refetching
		}
	}
});
```

2. After mutations or dialogs, refresh DOM references:
```typescript
// After dialog closes
const refreshedPanel = await canvas.findByTestId("panel-id");
// Use refreshed reference for subsequent assertions
const tab = within(refreshedPanel).getByText("Tab Name");
```

## File Naming and Location

### Story File Location
- Stories should be in `src/stories/` directory
- Mirror component structure: `src/components/agents/AgentCard.tsx` → `src/stories/Agents/AgentCard.stories.tsx`
- Use PascalCase for directory names in stories: `agents` → `Agents`, `connectors` → `Connectors`

### Story Title Convention
- Use forward slash for categorization: `"Agents/AgentCard"`
- Match directory structure in stories folder
- Categories: `Agents`, `Connectors`, `Multiflow`, `Transformations`, `Filters`, etc.

## Multi-Component Handling

### Batch Processing (Multiple Components)

When given multiple component paths:

1. Process each component independently
2. Generate individual story files for each
3. Report summary of created stories

Example:
```
Processing 3 components:
✓ Created src/stories/Agents/AgentCard.stories.tsx
✓ Created src/stories/Agents/AgentList.stories.tsx
✓ Created src/stories/Agents/AgentFilter.stories.tsx
```

### Page with Dependencies (--include-children)

When analyzing a page component:

1. Read the page component file
2. Extract all imported components from `src/components/`
3. For each child component:
   - Check if a story already exists
   - If not, create a story
   - If exists, optionally update it
4. Generate the main page story with all dependencies mocked

### Directory Recursive (--recursive)

When given a directory with `--recursive` flag:

1. Find all `.tsx` component files (exclude `.stories.tsx`, `.test.tsx`, `.spec.tsx`)
2. For each component:
   - Check if story exists
   - Generate story if missing
   - Update story if `--update` flag is present
3. Report summary

## Updating Existing Stories

When the `--update` flag is provided:

1. **Read Existing Story**: Read the current `.stories.tsx` file
2. **Read Updated Component**: Read the component file
3. **Compare**:
   - Check if props have changed (added, removed, renamed)
   - Check if new dependencies were added (hooks, contexts)
4. **Validate Against Pitfalls** (CRITICAL - see "Common Pitfalls & Solutions" section):
   - ✓ Check for React import (`import React from "react"`)
   - ✓ Verify MSW handler paths match actual endpoints after transformations
   - ✓ Validate QueryClient config (strict settings for React Query v5)
   - ✓ Check for atom reset wrapper (if using Jotai atoms)
   - ✓ Verify proper decorator stack matches component dependencies
   - ✓ Check timeout values for async operations (extend if needed)
   - ✓ Validate list vs by-ID endpoint handlers (both needed if component uses both)
5. **Update Story**:
   - Add missing React import if needed
   - Add new story variants for new props
   - Update existing variants with new prop types
   - Add new decorators if new dependencies detected
   - Add new MSW handlers if new API calls detected
   - Fix any pitfall violations found in step 4
   - Preserve existing test logic unless props changed significantly
6. **Report Changes**: Summarize what was updated, including pitfall fixes

Example update process:
```
Analyzing component: src/components/agents/AgentCard.tsx
Found existing story: src/stories/Agents/AgentCard.stories.tsx

Changes detected:
- New prop: 'onDelete' (function)
- New prop: 'isSelectable' (boolean)
- New hook: useDeleteAgent (requires MSW handler)

Pitfall validation:
✓ React import present
✗ Missing QueryClient strict configuration (React Query v5)
✗ Missing atom reset wrapper for selectedAgentRows

Updates made:
✓ Added 'WithDeleteAction' story variant
✓ Added 'WithSelection' story variant
✓ Added deleteAgentHandler to MSW handlers
✓ Updated Default story with new props
✓ Fixed QueryClient configuration (added staleTime: Infinity)
✓ Added AgentCardWrapper to reset selectedAgentRows atom
```

## Important Testing Patterns

### Using data-qa vs data-testid

This project uses `data-qa` attributes instead of `data-testid`:

```typescript
// In component
<SolaceButton dataQa="createButton" onClick={handleCreate}>
	Create
</SolaceButton>

// In test
const button = await screen.findByTestId("createButton"); // Queries for data-qa="createButton"
```

### Handling Async Operations

For elements depending on async data:
```typescript
await screen.findByTestId("asyncElement", {}, { timeout: 5000 });
```

Use `waitFor` for assertions that depend on query completion:
```typescript
await waitFor(() => {
	expect(canvas.getByText("Data Loaded")).toBeInTheDocument();
}, { timeout: 5000 });
```

## Workflow

### For New Stories

1. **Analyze component**
   - Read component file
   - Identify type, props, dependencies
2. **Select template**
   - Choose appropriate template based on component complexity
3. **Find mocks**
   - Search for existing MSW handlers in `mocks/` directory
4. **Generate story**
   - Create story file with appropriate decorators and handlers
   - Generate multiple variants
   - Add comprehensive tests
5. **Report**
   - Confirm story creation
   - Show file location

### For Updating Stories

1. **Read existing story**
2. **Analyze component changes**
3. **Compare for differences** (props, hooks, dependencies)
4. **Validate against pitfalls** (CRITICAL STEP):
   - Run through complete pitfall checklist (see "Common Pitfalls & Solutions")
   - Check React import, MSW handlers, QueryClient config, atom wrappers, decorators, timeouts
5. **Update story file**
   - Preserve existing logic
   - Add new variants for new props/states
   - Update props/types
   - Fix all pitfall violations
6. **Report changes** (include both feature updates and pitfall fixes)

### For Multiple Components

1. **Parse input**
   - Extract all component paths or scan directory
2. **For each component**:
   - Follow "New Stories" or "Updating Stories" workflow
3. **Report summary**
   - List all created/updated stories
   - Note any errors or skipped files

## Error Handling

- If component file doesn't exist, report error and continue with next component
- If story already exists without `--update` flag, ask user whether to overwrite
- If mock handlers can't be found, create basic handlers or note in output
- If component is too complex, generate basic story and note that manual review is needed

## Best Practices

1. **Follow existing patterns**: Match the style and structure of existing stories in the codebase
2. **Reuse mocks**: Always search for existing MSW handlers before creating new ones
3. **Comprehensive testing**: Include tests for critical user paths and edge cases
4. **Use proper decorators**: Include all necessary providers based on component dependencies
5. **Clear variant names**: Use descriptive names for story variants that indicate what they test
6. **Timeout considerations**: Use appropriate timeouts for async operations (default 1000ms may be too short)
7. **Query key matching**: When testing React Query components, ensure invalidation keys match query keys exactly
8. **DOM reference refreshing**: After mutations or dialogs, get fresh DOM references before assertions

## Output Format

After generating or updating stories, provide:

1. **Summary**: Number of stories created/updated
2. **File paths**: List each story file created/updated
3. **Manual review notes**: Any aspects that need manual review
4. **Next steps**: Suggestions for running/viewing the stories

Example output:
```
✓ Generated 3 Storybook stories

Created:
- src/stories/Agents/AgentCard.stories.tsx
- src/stories/Agents/AgentList.stories.tsx
- src/stories/Agents/AgentFilter.stories.tsx

Notes:
- AgentList uses a custom hook that may need additional mock setup
- Consider adding more edge case variants for AgentCard

Next steps:
1. Run Storybook: npm run storybook --prefix micro-frontends/intg
2. View stories at http://localhost:6006
3. Review generated stories and adjust as needed
```

## References

- Storybook docs: https://storybook.js.org/docs/react/writing-stories/introduction
- MSW docs: https://mswjs.io/docs/
- React Testing Library: https://testing-library.com/docs/react-testing-library/intro/
- Project patterns: See existing stories in `src/stories/` directory

---

## Quick Reference: Story Pattern Selector

Use this decision tree to quickly select the right pattern:

### Decision Tree

```
Does component use routing hooks (useHistory, useLocation, Link)?
├─ NO → Does it call APIs (useQuery, useMutation)?
│   ├─ NO → Does it use Jotai atoms?
│   │   ├─ NO → Use Pattern 1: Simple Presentational
│   │   └─ YES → Use Pattern 3 with JotaiProvider only
│   └─ YES → Use Pattern 3: React Query + Jotai
└─ YES → Is it a page-level component with complex flows?
    ├─ NO → Use Pattern 2 or Pattern 3 (with MemoryRouter)
    └─ YES → Use Pattern 4: Full Context Stack
```

### Pattern Selection Table

| If Component Has... | Use Pattern |
|-------------------|-------------|
| Only props, no hooks | Pattern 1 |
| `useHistory` or `Link` only | Pattern 2 |
| `useQuery` + Jotai atoms | Pattern 3 |
| `useApplicationContext` | Pattern 4 |
| Location state for stepper | Pattern 4 |
| 3+ contexts needed | Pattern 4 |

### MSW Handler Selection

| If Component Needs... | Handler Strategy |
|---------------------|------------------|
| Standard list endpoint | Strategy 1: Imported |
| By-ID with transformation | Strategy 2: Inline |
| Both list and by-ID | Strategy 3: Hybrid |
| Unique response shape | Strategy 2: Inline |

---

## Execution Instructions

When this skill is invoked:

1. Parse the command arguments to determine:
   - Component path(s)
   - Flags: `--update`, `--recursive`, `--include-children`

2. For each component, follow this **mandatory analysis workflow**:

   **Phase 1: Component Analysis**
   - Use the Read tool to analyze the component
   - Extract all React Query hooks used
   - Identify all imported utility functions
   - Note any data transformations

   **Phase 2: API Call Deep Dive** (CRITICAL)
   - For each React Query hook:
     - Use Grep to find the hook definition in `src/api`
     - Read the hook file to extract endpoint pattern
     - If hook receives function calls as arguments:
       - Trace those functions using Grep/Read
       - Test transformations with example data
       - Determine the ACTUAL endpoint after transformations
   - Document all actual API endpoints that will be called

   **Phase 3: Similar Component Analysis**
   - Use Grep to find other components using the same hooks
   - Use Glob to check if those components have stories
   - Read successful stories to understand handler patterns
   - Note any reusable handlers or patterns

   **Phase 4: Mock Handler Discovery**
   - Use Glob to search `mocks/` for relevant mock files
   - For each potential mock file:
     - Read it to verify it covers the ACTUAL endpoints (after transformations)
     - Distinguish between list vs by-ID endpoints
     - Note any path mismatches
   - If existing handlers don't match actual endpoints, create new inline handlers

   **Phase 5: Handler Validation**
   - Cross-reference actual API endpoints with available handlers
   - Verify handler paths exactly match transformed endpoints
   - Create checklist of covered vs missing endpoints
   - Generate inline handlers for missing endpoints

   **Phase 6: Story Generation**
   - Use the Glob tool to find existing stories
   - Generate or update the story using Write or Edit tools
   - Include comprehensive comments explaining:
     - Data transformations and why they matter
     - Which handlers cover which endpoints
     - References to transformation functions
   - Generate multiple variants based on component type

   **CRITICAL FOR --update FLAG**: If updating an existing story, add **Phase 7**:

   **Phase 7: Pitfall Validation** (MANDATORY for --update)
   - Read the existing story file completely
   - Validate against the **complete "Pitfall Validation Checklist"** (see section above)
   - Check ALL 7 checklist categories:
     1. Imports & Dependencies (React import, correct paths)
     2. MSW Handlers (list vs by-ID, transformation matching)
     3. QueryClient Configuration (strict settings)
     4. Jotai Atoms (wrapper component for reset)
     5. Decorator Stack (matches dependencies)
     6. Test Assertions (timeouts, async handling)
     7. Story Variants (complete coverage)
   - Fix ANY violations found
   - Document pitfall fixes in the update report

3. Provide clear output summarizing actions taken, including:
   - Components analyzed
   - Hooks traced and their actual endpoints
   - Transformations discovered
   - Handlers created/reused
   - **Pitfall validation results** (for --update: which pitfalls were checked, which were fixed)
   - Any manual review needed

4. DO NOT ask for confirmation between components in batch mode - process all components as instructed

5. If user provides multiple components or uses --recursive, process them all and provide a summary at the end

## Common Pitfalls & Solutions

Based on real-world debugging, these are the most common failures and how to fix them:

### Pitfall 1: Missing React Import

**Error**: `React is not defined`

**Solution**:
```typescript
import React from "react";  // Always needed for JSX
```

Even if TypeScript shows "unused", JSX requires it in scope.

### Pitfall 2: 404 Errors from API Calls

**Error**: `Request failed with status code 404`

**Root Cause**: Handler doesn't match actual endpoint after data transformations.

**Solution Process**:
1. Find the hook definition (e.g., `useGetConnectorTypeById`)
2. Trace any transformation functions (e.g., `splitMITypesString`)
3. Test transformation: `"ibmmq-source"` → `"ibmmq"`
4. Mock the ACTUAL endpoint: `/microIntegrationTypes/ibmmq` (not `ibmmq-source`)

### Pitfall 3: List Handler Doesn't Cover By-ID

**Error**: 404 on by-ID endpoint even though list handler exists

**Solution**: List and by-ID are SEPARATE endpoints:
```typescript
// Both needed:
http.get("/api/v2/integration/microIntegrationTypes", ...)      // List
http.get("/api/v2/integration/microIntegrationTypes/:id", ...)  // By-ID
```

### Pitfall 4: React Query Aggressive Re-renders

**Error**: Test fails due to stale DOM references after mutations

**Solution**: Use strict QueryClient config:
```typescript
const queryClient = new QueryClient({
	defaultOptions: {
		queries: {
			retry: false,
			refetchOnWindowFocus: false,
			refetchOnMount: false,
			refetchOnReconnect: false,
			gcTime: 0,
			staleTime: Infinity  // Critical: prevents refetch during story
		}
	}
});
```

### Pitfall 5: State Persists Between Stories

**Error**: Second story shows selected items from first story

**Solution**: Create wrapper component that resets atoms:
```typescript
const ComponentWrapper = (props) => {
	const setAtomValue = useSetAtom(atomName);

	useEffect(() => {
		setAtomValue(initialValue);  // Reset on mount
	}, [setAtomValue]);

	return <Component {...props} />;
};
```

### Pitfall 6: Dropdown Not Visible During Test

**Error**: `Unable to find element by role "listbox"`

**Solution**: Wait for dropdown to become visible:
```typescript
await userEvent.click(canvas.getByRole("button", { name: "Open" }));
await waitFor(() =>
	expect(canvas.getByRole("listbox")).toBeVisible()
);
```

### Pitfall 7: Custom Timeout Too Short

**Error**: Test fails with "Unable to find element" but element eventually appears

**Solution**: Increase timeout for async operations:
```typescript
await waitFor(() => {
	expect(canvas.getByTestId("element")).toBeInTheDocument();
}, { timeout: 5000 });  // Extended from default 1000ms
```

---

## Pitfall Validation Checklist (For --update)

When updating an existing story, **ALWAYS** validate against this complete checklist:

### 1. Imports & Dependencies
- [ ] **React import present**: `import React from "react";`
- [ ] All necessary test imports: `expect`, `waitFor`, `within`, `userEvent` (if interactions)
- [ ] Component import path correct (check casing: `multiFlow` not `multiflow`)
- [ ] Mock imports match actual mock file names

### 2. MSW Handlers (If Component Uses React Query)
- [ ] **List endpoint handler** exists (if using list query)
- [ ] **By-ID endpoint handler** exists (if using by-ID query)
- [ ] Handler paths match **actual endpoints** (after any transformations)
- [ ] Transformation functions traced (e.g., `splitMITypesString`)
- [ ] All endpoints documented in comments

### 3. QueryClient Configuration (If Component Uses React Query)
- [ ] QueryClient has strict configuration:
  ```typescript
  retry: false,
  refetchOnWindowFocus: false,
  refetchOnMount: false,
  refetchOnReconnect: false,
  gcTime: 0,
  staleTime: Infinity
  ```

### 4. Jotai Atoms (If Component Uses State Management)
- [ ] **Wrapper component** present to reset atoms between stories
- [ ] Wrapper uses `useEffect` with `useSetAtom` to reset state
- [ ] All stories use wrapper via `render` prop

### 5. Decorator Stack
- [ ] Matches component dependencies (see Component Type Decision Matrix)
- [ ] Has MemoryRouter if component uses `useHistory`, `useLocation`, `Link`
- [ ] Has QueryClientProvider if component uses `useQuery`, `useMutation`
- [ ] Has JotaiProvider if component uses Jotai atoms
- [ ] Has ApplicationProvider if component uses `useApplicationContext`
- [ ] Has Route with `location` prop if component uses `location.state`

### 6. Test Assertions
- [ ] Async operations use `waitFor` or `findBy` with appropriate timeouts
- [ ] Timeout values >= 3000ms for slow operations
- [ ] Dropdown/modal assertions wait for visibility
- [ ] No stale DOM references after mutations

### 7. Story Variants
- [ ] Covers all significant component states
- [ ] Tests user interactions where applicable
- [ ] Tests error states
- [ ] Tests loading states
- [ ] Each variant has descriptive name

### Quick Validation Command

Use this mental checklist during `--update`:

```
✓ React imported?
✓ Handlers match actual endpoints (after transformations)?
✓ QueryClient strict config (if React Query)?
✓ Atom wrapper (if Jotai)?
✓ Decorators match dependencies?
✓ Timeouts adequate (3000ms+)?
✓ List + By-ID handlers (if both used)?
```

**If ANY checklist item fails**, fix it during the update process.
