# Storybook Generator - Quick Reference

## Common Commands

### Single Component
```bash
/storybook-generator src/components/agents/AgentCard.tsx
```

### Multiple Components
```bash
/storybook-generator src/components/agents/AgentCard.tsx src/components/agents/AgentList.tsx
```

### Entire Directory
```bash
/storybook-generator src/components/agents/ --recursive
```

### Update Existing Story
```bash
/storybook-generator src/components/agents/AgentCard.tsx --update
```

### Page with Child Dependencies
```bash
/storybook-generator src/pages/multiFlow/CreateChildMI.tsx --include-children
```

### Update All Stories in Directory
```bash
/storybook-generator src/components/connectors/ --recursive --update
```

## Flags

| Flag | Description |
|------|-------------|
| `--update` | Update existing story instead of creating new |
| `--recursive` | Process all components in directory |
| `--include-children` | Generate stories for child components too |

## File Locations

- **Skill**: `.claude/skills/storybook-generator/SKILL.md`
- **Stories**: `micro-frontends/intg/src/stories/`
- **Components**: `micro-frontends/intg/src/components/`
- **Mocks**: `micro-frontends/intg/mocks/`

## Story Types Generated

1. **Simple Component** - Basic presentational components
2. **With State (Jotai)** - Components using Jotai atoms
3. **With React Query** - Components fetching data
4. **Full Context Stack** - Complex components with providers
5. **Dialog/Modal** - Dialog components with interactions
6. **Page/Stepper** - Multi-step page flows

## Story Variants

Each story includes variants like:
- `Default` - Basic state
- `WithData` - Populated with data
- `Loading` - Loading state
- `Error` - Error state
- `Empty` - No data state
- `WithUserInteraction` - Interactive tests
- `WithValidation` - Validation tests

## Testing Patterns

### Element Assertions
```typescript
expect(canvas.getByText("Text")).toBeInTheDocument();
expect(canvas.getByTestId("element-id")).toBeVisible();
expect(button).toBeEnabled();
```

### User Interactions
```typescript
await userEvent.click(button);
await userEvent.type(input, "text");
await userEvent.hover(element);
```

### Async Operations
```typescript
await waitFor(() => {
	expect(canvas.getByText("Loaded")).toBeInTheDocument();
}, { timeout: 5000 });

const element = await canvas.findByTestId("async-element", {}, { timeout: 5000 });
```

## Project-Specific Notes

- Use `data-qa` attributes (NOT `data-testid`)
- React Query v5 patterns with object syntax
- MSW handlers from `mocks/` directory
- QueryClient configured with `staleTime: Infinity` for tests

## Running Storybook

```bash
# Start Storybook
npm run storybook --prefix micro-frontends/intg

# View at
http://localhost:6006
```

## Common Workflow

```bash
# 1. Create/modify component
# 2. Generate story
/storybook-generator src/components/path/Component.tsx

# 3. Run Storybook
npm run storybook --prefix micro-frontends/intg

# 4. Review and adjust
# 5. Commit both component and story
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Story already exists | Use `--update` flag |
| Component not found | Check file path |
| Missing mocks | Review generated handlers, move to `mocks/` |
| Complex component | Review generated story, add custom tests |

## Example Outputs

### Success
```
✓ Generated story: src/stories/Agents/AgentCard.stories.tsx
Story includes: Default, WithData, Loading, Error variants
```

### Batch Success
```
Processing 3 components...
✓ Created src/stories/Agents/AgentCard.stories.tsx
✓ Created src/stories/Agents/AgentList.stories.tsx
✓ Created src/stories/Agents/AgentFilter.stories.tsx
Summary: 3 stories created
```

### Update
```
Changes detected:
- New prop: 'onDelete'
- New hook: useDeleteAgent
Updates made:
✓ Added 'WithDeleteAction' variant
✓ Added deleteAgentHandler
```

## Need Help?

- Full docs: `README.md`
- Implementation details: `SKILL.md`
- Existing story examples: `micro-frontends/intg/src/stories/`
