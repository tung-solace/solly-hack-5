# Comprehensive Storybook Test Creation Guide for MaaS-UI Integration MFE

## Executive Summary

Based on analysis of 86 Storybook stories in `micro-frontends/intg/src/stories`, this guide documents the core patterns, steps, and requirements for creating Storybook tests.

## Table of Contents

1. Core Story Structure
2. Provider Hierarchy & Decorators
3. MSW Mock Handlers
4. Testing Patterns & Assertions
5. Component Type Decision Matrix
6. Common Pitfalls & Solutions

---

## 1. CORE STORY STRUCTURE

### Essential Imports

```typescript
// Required for all stories
import type { Meta, StoryObj } from "@storybook/react-webpack5";
import React from "react";  // Always needed for JSX
import { expect, waitFor, within } from "storybook/test";

// Optional based on component needs
import { userEvent } from "storybook/test";  // For user interactions
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";  // For React Query
import { Provider as JotaiProvider, useSetAtom } from "jotai";  // For Jotai atoms
import { MemoryRouter, Route } from "react-router-dom";  // For routing
import { http } from "msw";  // For inline MSW handlers
```

### Basic Story File Template

```typescript
import type { Meta, StoryObj } from "@storybook/react-webpack5";
import ComponentName from "components/path/to/ComponentName";
import React from "react";
import { expect, waitFor, within } from "storybook/test";

const meta: Meta<typeof ComponentName> = {
  title: "Category/ComponentName",
  component: ComponentName,
  // decorators: [...] if needed
};

export default meta;

type Story = StoryObj<typeof ComponentName>;

export const Default: Story = {
  args: {
    // Component props here
  },
  // parameters: { msw: { handlers: [...] } } if API calls
  play: async ({ canvasElement }) => {
    const canvas = within(canvasElement);

    await waitFor(() => {
      expect(canvas.getByText("Expected Text")).toBeInTheDocument();
    });
  }
};
```

---

## 2. PROVIDER HIERARCHY & DECORATORS

### Global Decorators (Applied to ALL stories)

From `.storybook/preview.tsx` - automatically wraps every story:

```
ThemeProvider (Solace Design System)
  → EnvironmentProvider
    → UserProvider (default permissions)
      → SnackbarProvider (notistack)
        → ReactQueryProvider (QueryClient)
          → Suspense
            → Story
```

**Key Point**: Every story already has these providers. Only add story-level decorators when you need to OVERRIDE or ADD additional context.

### Story-Level Decorator Patterns

#### Pattern 1: Simple Presentational Component (NO decorators needed)

```typescript
// Example: MatchedChip.stories.tsx
const meta: Meta<typeof MatchedChip> = {
  title: "Multiflow/MatchedChip",
  component: MatchedChip,
  // NO decorators - uses global stack only
};
```

**When to use**: Pure presentational components with no routing, no API calls, no complex state.

#### Pattern 2: Routing Only

```typescript
// Example: EmptyState.stories.tsx
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
```

**When to use**: Component uses `useHistory`, `useLocation`, `Link`, or other React Router hooks.

#### Pattern 3: React Query + Jotai (State Management)

```typescript
// Example: FlowDetailCard.stories.tsx
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
```

**When to use**: Component uses `useQuery`, `useMutation`, or Jotai atoms.

**Critical**: QueryClient configuration prevents React Query v5's aggressive re-renders during story interactions.

#### Pattern 4: Full Context Stack (Page-Level Components)

```typescript
// Example: CreateMiParentStepper.stories.tsx
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

const userContextValue = {
  user: {
    userId: "currentUser",
    email: "test@example.com"
  },
  permissions: ["micro_integrations:*:*"]
};

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
```

**When to use**: Complex page components, steppers, multi-step flows that need full application context.

### Wrapper Component Pattern (State Reset)

For components using Jotai atoms that persist between stories:

```typescript
// Pattern: Wrapper component resets atoms on mount
const FlowCardWrapper = ({ flow, ...props }) => {
  const setSelectedRowIds = useSetAtom(selectedConnectorRows);

  useEffect(() => {
    setSelectedRowIds([]);  // Reset to empty on mount
  }, [setSelectedRowIds]);

  return <FlowDetailCard flow={flow} {...props} />;
};

// Usage in story
export const Default: Story = {
  render: () => <FlowCardWrapper flow={mockData[0]} />
};
```

**When to use**: Component uses atoms like `selectedConnectorRows`, `agentSortAtom`, or any `atomWithStorage`.

---

## 3. MSW MOCK HANDLERS

### Handler Organization Strategies

#### Strategy 1: Imported Handlers (Reusable)

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

**When to use**: Standard API endpoints used across multiple stories.

#### Strategy 2: Inline Handlers (Story-Specific)

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
      ...connectorTypes,           // Imported
      ibmmqConnectorType,          // Inline
      awssnsConnectorType          // Inline
    ]
  }
}
```

**When to use**:
- Story-specific endpoint variations
- Testing different API responses
- Showcasing endpoint structure for documentation
- Data transformations require specific mock shapes

#### Strategy 3: Hybrid Approach (RECOMMENDED)

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

**Why hybrid**:
- Reuse common handlers
- Customize specific endpoints
- Show transformation-specific mocks inline

### Critical Handler Patterns

#### List vs By-ID Endpoints (COMMON MISTAKE)

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

**Critical**: Components using both `useGetConnectorTypes()` AND `useGetConnectorTypeById()` need BOTH handlers.

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

### Common Mock Files

| File | Endpoints | Use Case |
|------|-----------|----------|
| `agentsMock.ts` | `/api/v2/integration/agents` | Agent list, types |
| `connectorEnvironmentsMock.ts` | `/api/v2/platform/environments` | Environments |
| `connectorTypes.ts` | `/api/v2/integration/microIntegrationTypes` | List of connector types |
| `connectorTypesAndCategories.ts` | `/api/v2/integration/microIntegrationTypes/{id}` | By-ID endpoints |
| `organizationsMock.ts` | `/api/v2/organizations` | Org data, feature flags |
| `users.ts` | `/api/v2/users` | User data |
| `serviceDetails.ts` | `/api/v2/missionControl/eventBrokerServices/{id}` | Service details |

---

## 4. TESTING PATTERNS & ASSERTIONS

### Query Methods

```typescript
// Scoped queries (RECOMMENDED)
const canvas = within(canvasElement);
canvas.getByText("Text")              // Sync - throws if not found
canvas.queryByText("Text")            // Sync - returns null if not found
await canvas.findByText("Text")       // Async - waits up to 1000ms

// Test ID queries (for data-qa attributes)
canvas.getByTestId("field-name")
await canvas.findByTestId("field-name")

// Role-based queries (for accessibility)
canvas.getByRole("button", { name: "Create" })
canvas.getAllByRole("option")

// Parent element access (for dialogs/modals)
const root = within(canvasElement.parentElement);
```

### Async Handling

```typescript
// Pattern 1: waitFor with assertion
await waitFor(() => {
  expect(canvas.getByText("Loaded")).toBeInTheDocument();
}, { timeout: 5000 });

// Pattern 2: findBy (implicit wait)
expect(await canvas.findByText("Loaded")).toBeInTheDocument();

// Pattern 3: Custom timeout
await canvas.findByTestId("element", {}, { timeout: 5000 });
```

### User Interactions

```typescript
import { userEvent } from "storybook/test";

// Click
await userEvent.click(button);

// Type
await userEvent.type(input, "text to type");

// Dropdown selection
await userEvent.click(canvas.getByRole("button", { name: "Open" }));
await waitFor(() => expect(canvas.getByRole("listbox")).toBeVisible());
const options = canvas.getAllByRole("option");
await userEvent.click(options[0]);

// Tab navigation
await userEvent.click(canvas.getByRole("tab", { name: "Summary" }));
```

### Common Assertions

```typescript
// Presence
expect(element).toBeInTheDocument();
expect(element).toBeVisible();

// State
expect(button).toBeEnabled();
expect(button).toBeDisabled();

// Content
expect(element).toHaveTextContent("Expected Text");
expect(element).not.toHaveTextContent("Unexpected Text");
expect(link).toHaveAttribute("href", "/path");

// Count
expect(canvas.getAllByText("Item")).toHaveLength(3);

// Absence
expect(canvas.queryByText("Not Present")).not.toBeInTheDocument();
```

### Multi-Step Flow Testing

```typescript
play: async ({ canvasElement }) => {
  const canvas = within(canvasElement);

  // Step 1: Initial state
  await waitFor(() => {
    expect(canvas.getByText("Step 1")).toBeInTheDocument();
  });

  // Step 2: Fill form
  await userEvent.type(
    canvas.getByTestId("name_field"),
    "Test Name"
  );

  // Step 3: Navigate
  await userEvent.click(canvas.getByText("Next: Step 2"));

  // Step 4: Verify navigation
  await waitFor(() => {
    expect(canvas.getByText("Step 2")).toBeInTheDocument();
  });

  // Step 5: Select from dropdown
  await userEvent.click(canvas.getByRole("button", { name: "Open" }));
  const options = canvas.getAllByRole("option");
  await userEvent.click(options[0]);

  // Step 6: Verify selected value persists
  await userEvent.click(canvas.getByText("Summary"));
  await waitFor(() => {
    expect(canvas.getByTestId("summary_name"))
      .toHaveTextContent("Test Name");
  });
}
```

---

## 5. COMPONENT TYPE DECISION MATRIX

| Component Type | Decorators | MSW Handlers | State Management | Complexity |
|----------------|-----------|--------------|------------------|-----------|
| **Pure Presentational** | None | No | No | LOW |
| **With Routing** | MemoryRouter | No | No | LOW |
| **List/Table** | Router + Jotai | Yes | Jotai atoms | MEDIUM |
| **Card with Selection** | QueryClient + Jotai + Router | Yes | Jotai atoms | MEDIUM |
| **Dialog/Modal** | QueryClient + Jotai + Router | Yes | Jotai atoms | MEDIUM |
| **Form Page** | Full Stack | Yes | Multiple atoms | HIGH |
| **Stepper** | Full Stack + Location State | Yes (8+ handlers) | Multiple atoms | HIGH |

### Component Analysis Checklist

Before writing story, analyze component to determine:

- [ ] Does it use React Router hooks? → Add MemoryRouter
- [ ] Does it call APIs? → Add QueryClientProvider + MSW handlers
- [ ] Does it use Jotai atoms? → Add JotaiProvider + wrapper for reset
- [ ] Is it a page-level component? → Add full context stack
- [ ] Does it use location.state? → Add Route with location prop
- [ ] Does it have complex multi-step flow? → Plan sequential play() function

---

## 6. COMMON PITFALLS & SOLUTIONS

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

## Step-by-Step Story Creation Process

### Phase 1: Component Analysis

1. **Read the component file**
2. **Identify dependencies**:
   - React Query hooks? → Note which ones
   - Jotai atoms? → Note which ones
   - Router hooks? → Note if routing needed
   - Utility functions? → Trace transformations
3. **List API calls**:
   - Extract endpoint patterns from hooks
   - Trace data transformations
   - Determine ACTUAL endpoints called

### Phase 2: Find Similar Stories

1. **Search for components using same hooks**:
   ```bash
   grep -r "useGetConnectorTypeById" src/components
   ```
2. **Check if they have stories**:
   ```bash
   find src/stories -name "ComponentName.stories.tsx"
   ```
3. **Read successful stories** to understand handler patterns
4. **Reuse their decorator and mock patterns**

### Phase 3: Prepare Mocks

1. **Search for existing handlers**:
   ```bash
   grep -r "microIntegrationTypes" src/mocks
   ```
2. **Verify handler paths match actual endpoints** (after transformations)
3. **Distinguish list vs by-ID handlers**
4. **Create inline handlers** for missing endpoints

### Phase 4: Build Decorator Stack

Based on component analysis:
- Presentational only? → No decorators
- Uses routing? → Add MemoryRouter
- Uses React Query? → Add QueryClientProvider
- Uses Jotai? → Add JotaiProvider + wrapper
- Page-level? → Add full stack

### Phase 5: Write Story Variants

Generate variants based on component type:
- **State-based**: Different states (running, error, deploying)
- **Props-based**: Different prop combinations
- **Feature-based**: With/without optional features
- **Interaction-based**: User flow testing

### Phase 6: Write Play Functions

1. **Wait for initial render**:
   ```typescript
   await waitFor(() => {
     expect(canvas.getByTestId("element")).toBeInTheDocument();
   });
   ```
2. **Test user interactions** (if applicable)
3. **Verify state updates**
4. **Assert final state**

### Phase 7: Document Transformations

Add comments explaining:
```typescript
// NOTE: useGetConnectorTypeById receives splitMITypesString(...)
// splitMITypesString("ibmmq-source") returns "ibmmq" (direction removed)
// Therefore handlers match base types: /ibmmq, not /ibmmq-source
```

---

## Quick Reference: Story Template Selector

### Template 1: Simple Presentational
```typescript
// NO decorators, NO handlers, args pattern
export const Default: Story = {
  args: { prop1: "value" },
  play: async ({ canvasElement }) => {
    const canvas = within(canvasElement);
    expect(canvas.getByText("Text")).toBeInTheDocument();
  }
};
```

### Template 2: With API Calls
```typescript
// QueryClient + MSW handlers
const queryClient = new QueryClient({...});

decorators: [(Story) => (
  <QueryClientProvider client={queryClient}>
    <Story />
  </QueryClientProvider>
)]

parameters: {
  msw: {
    handlers: [...]
  }
}
```

### Template 3: With State Management
```typescript
// QueryClient + Jotai + Wrapper
const Wrapper = (props) => {
  const setAtom = useSetAtom(atomName);
  useEffect(() => { setAtom([]); }, []);
  return <Component {...props} />;
};

export const Default: Story = {
  render: () => <Wrapper {...props} />
};
```

### Template 4: Page-Level Stepper
```typescript
// Full stack + Router with location state
decorators: [(Story) => (
  <QueryClientProvider client={queryClient}>
    <UserProvider value={...}>
      <JotaiProvider>
        <ApplicationProvider value={...}>
          <MemoryRouter initialEntries={["/path"]}>
            <Route
              path="/path"
              location={{
                state: { connectorTypeId: "..." },
                pathname: "/path",
                search: "",
                hash: "",
                key: ""
              }}
            >
              <Story />
            </Route>
          </MemoryRouter>
        </ApplicationProvider>
      </JotaiProvider>
    </UserProvider>
  </QueryClientProvider>
)]

parameters: {
  msw: {
    handlers: [
      // 8+ handlers for stepper endpoints
    ]
  }
}
```

---

## Files Referenced

**Analysis Sources:**
- 86 story files in `micro-frontends/intg/src/stories/`
- Global decorators: `.storybook/preview.tsx`
- Mock files: `micro-frontends/intg/src/mocks/`
- Component examples analyzed in detail

**Key Examples:**
- Simple: `stories/Multiflow/MatchedChip.stories.tsx`
- Medium: `stories/Multiflow/FlowDetailCard.stories.tsx`
- Complex: `stories/Multiflow/CreateMiParentStepper.stories.tsx`

---

## Next Steps for Implementation

This guide should be used as a reference when:
1. Creating new Storybook stories
2. Debugging failing stories
3. Understanding existing story patterns
4. Training new team members on story creation
