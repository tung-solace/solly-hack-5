# Storybook Generator - Detailed Examples

This document provides detailed examples of using the storybook-generator skill with real-world scenarios.

## Table of Contents

1. [Example 1: Simple Badge Component](#example-1-simple-badge-component)
2. [Example 2: Component with React Query](#example-2-component-with-react-query)
3. [Example 3: Component with Jotai State](#example-3-component-with-jotai-state)
4. [Example 4: Dialog Component](#example-4-dialog-component)
5. [Example 5: Page Component with Stepper](#example-5-page-component-with-stepper)
6. [Example 6: Batch Processing](#example-6-batch-processing)
7. [Example 7: Updating Existing Story](#example-7-updating-existing-story)

---

## Example 1: Simple Badge Component

### Component File
`src/components/common/StatusBadge.tsx`

```typescript
import React from "react";
import { SolaceBadge } from "@SolaceDev/maas-react-components";

export interface StatusBadgeProps {
	status: "running" | "stopped" | "error" | "pending";
	label?: string;
}

const StatusBadge: React.FC<StatusBadgeProps> = ({ status, label }) => {
	const colorMap = {
		running: "success",
		stopped: "neutral",
		error: "error",
		pending: "warning"
	};

	return (
		<SolaceBadge
			color={colorMap[status]}
			dataQa={`status-badge-${status}`}
		>
			{label || status}
		</SolaceBadge>
	);
};

export default StatusBadge;
```

### Command
```bash
/storybook-generator src/components/common/StatusBadge.tsx
```

### Generated Story
`src/stories/Common/StatusBadge.stories.tsx`

```typescript
import type { Meta, StoryObj } from "@storybook/react-webpack5";
import StatusBadge from "components/common/StatusBadge";
import React from "react";
import { expect, within } from "storybook/test";

const meta: Meta<typeof StatusBadge> = {
	title: "Common/StatusBadge",
	component: StatusBadge
};

export default meta;

type Story = StoryObj<typeof StatusBadge>;

export const Running: Story = {
	args: {
		status: "running"
	},
	play: async ({ canvasElement }) => {
		const canvas = within(canvasElement);
		expect(canvas.getByTestId("status-badge-running")).toBeInTheDocument();
		expect(canvas.getByText("running")).toBeInTheDocument();
	}
};

export const Stopped: Story = {
	args: {
		status: "stopped"
	},
	play: async ({ canvasElement }) => {
		const canvas = within(canvasElement);
		expect(canvas.getByTestId("status-badge-stopped")).toBeInTheDocument();
		expect(canvas.getByText("stopped")).toBeInTheDocument();
	}
};

export const Error: Story = {
	args: {
		status: "error"
	},
	play: async ({ canvasElement }) => {
		const canvas = within(canvasElement);
		expect(canvas.getByTestId("status-badge-error")).toBeInTheDocument();
		expect(canvas.getByText("error")).toBeInTheDocument();
	}
};

export const Pending: Story = {
	args: {
		status: "pending"
	},
	play: async ({ canvasElement }) => {
		const canvas = within(canvasElement);
		expect(canvas.getByTestId("status-badge-pending")).toBeInTheDocument();
		expect(canvas.getByText("pending")).toBeInTheDocument();
	}
};

export const WithCustomLabel: Story = {
	args: {
		status: "running",
		label: "Active"
	},
	play: async ({ canvasElement }) => {
		const canvas = within(canvasElement);
		expect(canvas.getByTestId("status-badge-running")).toBeInTheDocument();
		expect(canvas.getByText("Active")).toBeInTheDocument();
	}
};
```

### Output
```
✓ Analyzing component: src/components/common/StatusBadge.tsx
✓ Component type: Simple presentational component
✓ Props detected: status (union type), label (optional string)
✓ Generated story: src/stories/Common/StatusBadge.stories.tsx

Story includes:
- Running variant
- Stopped variant
- Error variant
- Pending variant
- WithCustomLabel variant

Next steps:
1. Run Storybook: npm run storybook --prefix micro-frontends/intg
2. View at http://localhost:6006/?path=/story/common-statusbadge
```

---

## Example 2: Component with React Query

### Component File
`src/components/agents/AgentMetrics.tsx`

```typescript
import React from "react";
import { useQuery } from "@tanstack/react-query";
import { getAgentMetrics } from "api/agents";
import { SolaceCard, SolaceLoader } from "@SolaceDev/maas-react-components";

export interface AgentMetricsProps {
	agentId: string;
}

const AgentMetrics: React.FC<AgentMetricsProps> = ({ agentId }) => {
	const { data, isLoading, error } = useQuery({
		queryKey: ["agentMetrics", agentId],
		queryFn: () => getAgentMetrics(agentId),
		enabled: !!agentId
	});

	if (isLoading) return <SolaceLoader />;
	if (error) return <div>Error loading metrics</div>;
	if (!data) return <div>No metrics available</div>;

	return (
		<SolaceCard dataQa="agent-metrics-card">
			<h3>Agent Metrics</h3>
			<div>Messages Processed: {data.messagesProcessed}</div>
			<div>Success Rate: {data.successRate}%</div>
		</SolaceCard>
	);
};

export default AgentMetrics;
```

### Command
```bash
/storybook-generator src/components/agents/AgentMetrics.tsx
```

### Generated Story
`src/stories/Agents/AgentMetrics.stories.tsx`

```typescript
import type { Meta, StoryObj } from "@storybook/react-webpack5";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import AgentMetrics from "components/agents/AgentMetrics";
import { http } from "msw";
import React from "react";
import { expect, waitFor, within } from "storybook/test";

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

// Mock data
const mockMetricsData = {
	messagesProcessed: 15420,
	successRate: 98.5
};

// MSW handler
const agentMetricsHandler = http.get("/api/v2/agents/:agentId/metrics", () => {
	return new Response(JSON.stringify({ data: mockMetricsData }));
});

const agentMetricsErrorHandler = http.get("/api/v2/agents/:agentId/metrics", () => {
	return new Response(null, { status: 500 });
});

const meta: Meta<typeof AgentMetrics> = {
	title: "Agents/AgentMetrics",
	component: AgentMetrics,
	decorators: [
		(Story) => (
			<QueryClientProvider client={queryClient}>
				<Story />
			</QueryClientProvider>
		)
	]
};

export default meta;

type Story = StoryObj<typeof AgentMetrics>;

export const Default: Story = {
	args: {
		agentId: "agent-123"
	},
	parameters: {
		msw: {
			handlers: [agentMetricsHandler]
		}
	},
	play: async ({ canvasElement }) => {
		const canvas = within(canvasElement);

		await waitFor(() => {
			expect(canvas.getByTestId("agent-metrics-card")).toBeInTheDocument();
		}, { timeout: 3000 });

		expect(canvas.getByText("Agent Metrics")).toBeInTheDocument();
		expect(canvas.getByText("Messages Processed: 15420")).toBeInTheDocument();
		expect(canvas.getByText("Success Rate: 98.5%")).toBeInTheDocument();
	}
};

export const Loading: Story = {
	args: {
		agentId: "agent-123"
	},
	parameters: {
		msw: {
			handlers: [
				http.get("/api/v2/agents/:agentId/metrics", async () => {
					await new Promise((resolve) => setTimeout(resolve, 10000));
					return new Response(JSON.stringify({ data: mockMetricsData }));
				})
			]
		}
	},
	play: async ({ canvasElement }) => {
		const canvas = within(canvasElement);
		// Should show loader while data is loading
		expect(canvas.getByRole("progressbar")).toBeInTheDocument();
	}
};

export const Error: Story = {
	args: {
		agentId: "agent-123"
	},
	parameters: {
		msw: {
			handlers: [agentMetricsErrorHandler]
		}
	},
	play: async ({ canvasElement }) => {
		const canvas = within(canvasElement);

		await waitFor(() => {
			expect(canvas.getByText("Error loading metrics")).toBeInTheDocument();
		}, { timeout: 3000 });
	}
};

export const NoData: Story = {
	args: {
		agentId: "agent-456"
	},
	parameters: {
		msw: {
			handlers: [
				http.get("/api/v2/agents/:agentId/metrics", () => {
					return new Response(JSON.stringify({ data: null }));
				})
			]
		}
	},
	play: async ({ canvasElement }) => {
		const canvas = within(canvasElement);

		await waitFor(() => {
			expect(canvas.getByText("No metrics available")).toBeInTheDocument();
		}, { timeout: 3000 });
	}
};
```

---

## Example 3: Component with Jotai State

### Component File
`src/components/filters/AgentStatusFilter.tsx`

```typescript
import React from "react";
import { useAtom } from "jotai";
import { agentStatusFilterAtom } from "atoms/filterAtoms";
import { SolaceSelect } from "@SolaceDev/maas-react-components";

const AgentStatusFilter: React.FC = () => {
	const [status, setStatus] = useAtom(agentStatusFilterAtom);

	const options = [
		{ value: "all", label: "All Statuses" },
		{ value: "running", label: "Running" },
		{ value: "stopped", label: "Stopped" },
		{ value: "error", label: "Error" }
	];

	return (
		<SolaceSelect
			label="Status Filter"
			value={status}
			onChange={(e) => setStatus(e.target.value)}
			options={options}
			dataQa="agent-status-filter"
		/>
	);
};

export default AgentStatusFilter;
```

### Command
```bash
/storybook-generator src/components/filters/AgentStatusFilter.tsx
```

### Generated Story
`src/stories/Filters/AgentStatusFilter.stories.tsx`

```typescript
import type { Meta, StoryObj } from "@storybook/react-webpack5";
import { agentStatusFilterAtom } from "atoms/filterAtoms";
import AgentStatusFilter from "components/filters/AgentStatusFilter";
import { Provider as JotaiProvider, useSetAtom } from "jotai";
import React, { useEffect } from "react";
import { expect, screen, userEvent, waitFor, within } from "storybook/test";

const meta: Meta<typeof AgentStatusFilter> = {
	title: "Filters/AgentStatusFilter",
	component: AgentStatusFilter,
	decorators: [
		(Story) => (
			<JotaiProvider>
				<Story />
			</JotaiProvider>
		)
	]
};

export default meta;

type Story = StoryObj<typeof AgentStatusFilter>;

// Component wrapper to reset atoms
const FilterWithReset = () => {
	const setStatus = useSetAtom(agentStatusFilterAtom);

	useEffect(() => {
		setStatus("all");
	}, [setStatus]);

	return <AgentStatusFilter />;
};

export const Default: Story = {
	render: () => <FilterWithReset />,
	play: async ({ canvasElement }) => {
		const canvas = within(canvasElement);

		expect(canvas.getByTestId("agent-status-filter")).toBeInTheDocument();
		expect(canvas.getByLabelText("Status Filter")).toBeInTheDocument();
	}
};

export const WithSelection: Story = {
	render: () => <FilterWithReset />,
	play: async ({ canvasElement }) => {
		const canvas = within(canvasElement);

		const select = canvas.getByTestId("agent-status-filter");
		expect(select).toBeInTheDocument();

		// Click to open dropdown
		await userEvent.click(select);

		// Wait for options to appear
		await waitFor(() => {
			expect(screen.getByRole("listbox")).toBeVisible();
		});

		// Verify all options are present
		const options = screen.getAllByRole("option");
		expect(options).toHaveLength(4);
		expect(screen.getByText("All Statuses")).toBeInTheDocument();
		expect(screen.getByText("Running")).toBeInTheDocument();
		expect(screen.getByText("Stopped")).toBeInTheDocument();
		expect(screen.getByText("Error")).toBeInTheDocument();

		// Select "Running"
		await userEvent.click(screen.getByText("Running"));

		// Verify selection
		await waitFor(() => {
			expect(canvas.getByDisplayValue("Running")).toBeInTheDocument();
		});
	}
};
```

---

## Example 4: Dialog Component

### Component File
`src/components/dialogs/DeleteAgentDialog.tsx`

```typescript
import React from "react";
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { deleteAgent } from "api/agents";
import { SolaceDialog, SolaceButton } from "@SolaceDev/maas-react-components";

export interface DeleteAgentDialogProps {
	agentId: string;
	agentName: string;
	onClose: () => void;
	onSuccess?: () => void;
}

const DeleteAgentDialog: React.FC<DeleteAgentDialogProps> = ({
	agentId,
	agentName,
	onClose,
	onSuccess
}) => {
	const queryClient = useQueryClient();

	const deleteMutation = useMutation({
		mutationFn: () => deleteAgent(agentId),
		onSuccess: () => {
			queryClient.invalidateQueries({ queryKey: ["agents"] });
			onSuccess?.();
			onClose();
		}
	});

	return (
		<SolaceDialog
			open
			onClose={onClose}
			title="Delete Agent"
			dataQa="delete-agent-dialog"
		>
			<p>Are you sure you want to delete "{agentName}"?</p>
			<p>This action cannot be undone.</p>

			<SolaceButton
				onClick={() => deleteMutation.mutate()}
				disabled={deleteMutation.isPending}
				dataQa="confirm-delete-button"
				color="error"
			>
				{deleteMutation.isPending ? "Deleting..." : "Delete"}
			</SolaceButton>

			<SolaceButton
				onClick={onClose}
				dataQa="cancel-delete-button"
			>
				Cancel
			</SolaceButton>
		</SolaceDialog>
	);
};

export default DeleteAgentDialog;
```

### Command
```bash
/storybook-generator src/components/dialogs/DeleteAgentDialog.tsx
```

### Generated Story (Abbreviated)
`src/stories/Dialogs/DeleteAgentDialog.stories.tsx`

```typescript
import { Meta, StoryObj } from "@storybook/react-webpack5";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import DeleteAgentDialog from "components/dialogs/DeleteAgentDialog";
import { http } from "msw";
import React from "react";
import { expect, userEvent, waitFor, within } from "storybook/test";

const queryClient = new QueryClient({
	defaultOptions: {
		queries: {
			retry: false,
			refetchOnWindowFocus: false,
			staleTime: Infinity
		}
	}
});

const deleteAgentHandler = http.delete("/api/v2/agents/:agentId", () => {
	return new Response(null, { status: 204 });
});

const deleteAgentErrorHandler = http.delete("/api/v2/agents/:agentId", () => {
	return new Response(null, { status: 500 });
});

const meta: Meta<typeof DeleteAgentDialog> = {
	title: "Dialogs/DeleteAgentDialog",
	component: DeleteAgentDialog,
	decorators: [
		(Story) => (
			<QueryClientProvider client={queryClient}>
				<Story />
			</QueryClientProvider>
		)
	]
};

export default meta;

type Story = StoryObj<typeof DeleteAgentDialog>;

export const Default: Story = {
	args: {
		agentId: "agent-123",
		agentName: "Test Agent",
		onClose: () => {},
		onSuccess: () => {}
	},
	parameters: {
		msw: {
			handlers: [deleteAgentHandler]
		}
	},
	play: async ({ canvasElement }) => {
		const canvas = within(canvasElement);

		expect(canvas.getByTestId("delete-agent-dialog")).toBeInTheDocument();
		expect(canvas.getByText("Delete Agent")).toBeInTheDocument();
		expect(canvas.getByText('Are you sure you want to delete "Test Agent"?')).toBeInTheDocument();
		expect(canvas.getByTestId("confirm-delete-button")).toBeEnabled();
		expect(canvas.getByTestId("cancel-delete-button")).toBeEnabled();
	}
};

export const WithDeleteAction: Story = {
	args: {
		agentId: "agent-123",
		agentName: "Test Agent",
		onClose: () => {},
		onSuccess: () => {}
	},
	parameters: {
		msw: {
			handlers: [deleteAgentHandler]
		}
	},
	play: async ({ canvasElement }) => {
		const canvas = within(canvasElement);

		const deleteButton = canvas.getByTestId("confirm-delete-button");
		expect(deleteButton).toBeEnabled();

		// Click delete
		await userEvent.click(deleteButton);

		// Button should show loading state
		await waitFor(() => {
			expect(canvas.getByText("Deleting...")).toBeInTheDocument();
		});

		// Wait for success (dialog should close, but we can't test that in isolation)
	}
};
```

---

## Example 6: Batch Processing

### Command
```bash
/storybook-generator src/components/agents/AgentCard.tsx src/components/agents/AgentList.tsx src/components/agents/AgentFilters.tsx
```

### Output
```
Processing 3 components...

✓ Analyzing component 1/3: src/components/agents/AgentCard.tsx
  Component type: Simple presentational component with props
  Generated story: src/stories/Agents/AgentCard.stories.tsx
  Variants: Default, WithActions, Selected, Disabled

✓ Analyzing component 2/3: src/components/agents/AgentList.tsx
  Component type: Complex component with React Query
  Found existing mocks: agentsMock, organizationsMock
  Generated story: src/stories/Agents/AgentList.stories.tsx
  Variants: Default, Loading, Error, Empty, WithPagination

✓ Analyzing component 3/3: src/components/agents/AgentFilters.tsx
  Component type: Component with Jotai state management
  Atoms detected: agentStatusFilterAtom, agentTypeFilterAtom
  Generated story: src/stories/Agents/AgentFilters.stories.tsx
  Variants: Default, WithFilterSelection, ResetFilters

Summary: 3 stories created successfully

Files created:
- src/stories/Agents/AgentCard.stories.tsx
- src/stories/Agents/AgentList.stories.tsx
- src/stories/Agents/AgentFilters.stories.tsx

Next steps:
1. Run Storybook: npm run storybook --prefix micro-frontends/intg
2. Review generated stories at http://localhost:6006/?path=/story/agents
3. Adjust tests and variants as needed
```

---

## Example 7: Updating Existing Story

### Scenario
The `AgentCard` component had new props added: `onDelete` callback and `isDeletable` boolean.

### Component (Updated)
```typescript
export interface AgentCardProps {
	agent: Agent;
	onSelect?: () => void;
	selected?: boolean;
	onDelete?: (agentId: string) => void;  // NEW
	isDeletable?: boolean;                   // NEW
}
```

### Command
```bash
/storybook-generator src/components/agents/AgentCard.tsx --update
```

### Output
```
Analyzing component: src/components/agents/AgentCard.tsx
Found existing story: src/stories/Agents/AgentCard.stories.tsx

Comparing with existing story...

Changes detected:
✓ New prop: 'onDelete' (function)
✓ New prop: 'isDeletable' (boolean)

Updates being applied:
✓ Adding new story variant: 'WithDeleteAction'
✓ Adding new story variant: 'NotDeletable'
✓ Updating 'Default' story with new optional props
✓ Preserving existing variants: Selected, Disabled

Story updated successfully!

Updated: src/stories/Agents/AgentCard.stories.tsx

New variants added:
- WithDeleteAction: Shows delete button and tests click interaction
- NotDeletable: Shows card without delete option

Existing variants preserved:
- Default (updated with new props)
- Selected
- Disabled

Next steps:
1. Review the updated story
2. Run Storybook to test new variants
3. Commit changes
```

---

## Example 8: Recursive Directory Processing

### Command
```bash
/storybook-generator src/components/multiFlow/ --recursive
```

### Output
```
Scanning directory: src/components/multiFlow/

Found components:
✓ DeleteMIDialog.tsx (no story)
✓ DeleteMIFlowDialog.tsx (no story)
✓ FlowEditDetailsTab.tsx (no story)
✓ FlowViewDetailsTab.tsx (no story)
✓ MicroIntegrationFlowList.tsx (story exists)
✓ MultiFlowCardsAndOverlayArrowWrapper.tsx (no story)
✓ NodeConnectionOverview.tsx (no story)
✓ ParentMIViewDetailsTab.tsx (no story)
✓ SummaryTabSection.tsx (no story)
✓ MultiflowArrowWithOverlay.tsx (no story)
✓ stepperComponents/MiParentDetailsStep.tsx (no story)
✓ stepperComponents/MiParentEnvironment.tsx (no story)
✓ stepperComponents/SolaceEventBrokerSelection.tsx (no story)
✓ utils/MultiflowValidationUtils.tsx (utility file, skipping)
✓ utils/RenderSolaceOrVendor.tsx (no story)

Processing 13 components...

[1/13] DeleteMIDialog.tsx
✓ Generated src/stories/Multiflow/DeleteMIDialog.stories.tsx

[2/13] DeleteMIFlowDialog.tsx
✓ Generated src/stories/Multiflow/DeleteMIFlowDialog.stories.tsx

[3/13] FlowEditDetailsTab.tsx
✓ Generated src/stories/Multiflow/FlowEditDetailsTab.stories.tsx

... (showing all 13)

Summary:
✓ 13 stories created
⊘ 1 existing story skipped (MicroIntegrationFlowList)
⊘ 1 utility file skipped (MultiflowValidationUtils)

Total processing time: 45 seconds

All stories created in: src/stories/Multiflow/

Next steps:
1. Run Storybook: npm run storybook --prefix micro-frontends/intg
2. Review stories at http://localhost:6006/?path=/story/multiflow
3. Enhance stories with additional test cases as needed
```

---

## Tips from Real Usage

### Tip 1: Start Small
Begin with a single simple component to verify the generated story meets your expectations, then move to batch processing.

### Tip 2: Review Mock Handlers
After generation, check if the MSW handlers are correctly configured. Sometimes you may need to adjust mock data to match your test scenarios.

### Tip 3: Enhance Generated Tests
While the skill generates comprehensive tests, consider adding edge cases specific to your component's business logic.

### Tip 4: Use --update Regularly
After component changes, use `--update` to keep stories in sync without losing custom test logic you've added.

### Tip 5: Combine with CI/CD
Consider adding a check in your CI/CD pipeline to ensure all components have corresponding stories.

---

For more information, see:
- [README.md](README.md) - Full documentation
- [QUICK_REFERENCE.md](QUICK_REFERENCE.md) - Quick command reference
- [SKILL.md](SKILL.md) - Implementation details
