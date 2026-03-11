# Config Templates

Purpose-specific templates for `.claude/git-diff.config.md`. Pick the template
matching the user's selected purpose, substitute detected values from the
structural analysis, then write the result.

## Table of Contents

- [Story Coverage](#story-coverage)
- [Test Coverage](#test-coverage)
- [API Change Tracking](#api-change-tracking)
- [Refactoring Impact](#refactoring-impact)
- [Custom](#custom)
- [Substitution Guide](#substitution-guide)

---

## Story Coverage

```markdown
# Git Diff Config — Story Coverage

Track component-to-story mapping so `/git-diff` highlights components missing
stories and suggests `/storybook-generator` for gaps.

## Categories
<!-- For each detected sub-project, add these two lines: -->
- {Name} Components: `{componentsGlob}`
- {Name} Stories: `{storiesGlob}`
<!-- Then add these shared categories: -->
- MSW Mocks: `{mockGlob}`
- Hooks/API: `{hooksGlob}`
- Styles: `**/*.css`, `**/*.scss`, `**/*.styled.*`
- Config: `package.json`, `tsconfig*`, `*config*`, `.storybook/**`

## Story Detection
- Story location: {storyLocation}
- Story naming: `<ComponentName>.stories.tsx`
- Search paths: {storySearchPaths}

## Extract Patterns
- Test IDs: `{testIdAttr}`
- Query keys: `{queryKeysPath}`
- Hooks: `useQuery`, `useMutation`
<!-- If a state management library was detected, add its primary hook: -->
- State: `{stateHook}`

## Downstream Skills
- Missing stories: `/storybook-generator`
- Missing mocks: suggest adding MSW handlers

## Notes
<!-- For each sub-project, note data-fetching lib and version: -->
- {Name} uses {dataFetchLib} {dataFetchVersion} ({versionNote})
<!-- If state management was detected: -->
- State management: {stateLib}
```

---

## Test Coverage

```markdown
# Git Diff Config — Test Coverage

Track source-to-test mapping so `/git-diff` highlights files missing test
coverage and suggests `/test-generator` for gaps.

## Categories
<!-- For each detected sub-project, add these two lines: -->
- {Name} Source: `{sourceGlob}`
- {Name} Tests: `{testGlob}`
<!-- Then add shared categories: -->
- Mocks: `{mockGlob}`
- Config: `package.json`, `tsconfig*`, `jest.config.*`, `vitest.config.*`

## Test Detection
- Test location: {testLocation}
- Test naming: `{testNamingPattern}`
- Test runner: {testRunner}
- Search paths: {testSearchPaths}

## Extract Patterns
- Test IDs: `{testIdAttr}`
- Assertions: common matchers from {testRunner}
- Mock patterns: {mockDescription}

## Downstream Skills
- Missing tests: `/test-generator`
- Missing mocks: suggest adding test mocks/fixtures

## Notes
<!-- For each sub-project, note its test runner config path: -->
- {Name}: {testRunner} config at `{testConfigPath}`
```

---

## API Change Tracking

```markdown
# Git Diff Config — API Change Tracking

Track API files, query keys, hooks, and consumer impact so `/git-diff`
highlights breaking API changes and affected consumers.

## Categories
<!-- For each detected sub-project, add these three lines: -->
- {Name} API/Services: `{apiGlob}`
- {Name} Hooks: `{hooksGlob}`
- {Name} Consumers: `{consumersGlob}`
<!-- Then add shared categories: -->
- Query Keys: `{queryKeysGlob}`
- Types/Interfaces: `{typesGlob}`
- Config: `package.json`, `tsconfig*`

## Impact Detection
- When a file in API/Services changes, find all importers
- When query keys change, find all `useQuery`/`useMutation` consumers
- When hook signatures change, find all call sites

## Extract Patterns
- Query keys: `{queryKeysPath}`
- API base URLs: look for base URL config
- Request/response types: `{typesGlob}`
- Hooks: `useQuery`, `useMutation`, `useSuspenseQuery`

## Downstream Skills
- API docs: suggest updating API documentation
- Consumer updates: flag components that need query key updates

## Notes
<!-- For each sub-project, note data-fetching lib and version: -->
- {Name} uses {dataFetchLib} {dataFetchVersion}
```

---

## Refactoring Impact

```markdown
# Git Diff Config — Refactoring Impact

Analyze import graphs, shared utilities, and breaking-change blast radius so
`/git-diff` highlights ripple effects of refactoring.

## Categories
<!-- For each detected sub-project, add: -->
- {Name} Source: `{sourceGlob}`
<!-- Then add shared categories: -->
- Shared Utils: `{sharedUtilsGlob}`
- Types/Interfaces: `{typesGlob}`
- Config: `package.json`, `tsconfig*`
- Tests: `{testGlob}`
- Stories: `{storiesGlob}`

## Impact Detection
- For each changed file, find all direct importers (depth 1)
- For shared utils, find all transitive importers (depth 2+)
- Flag exports that were removed or had signatures changed
- Flag type changes that affect multiple consumers

## Extract Patterns
- Exports: named exports, default exports
- Import paths: relative, alias (@/), package
- Shared patterns: files imported by 3+ other files

## Downstream Skills
- Broken imports: suggest updating consumers
- Removed exports: flag files still importing them

## Notes
<!-- For each sub-project, note its tsconfig for path aliases: -->
- {Name}: `{tsconfigPath}` (check path aliases)
```

---

## Custom

For the **Custom** purpose, ask the user to describe their focus. Then generate
a config that combines relevant sections from the templates above, tailored to
the user's description. Include only the categories, detection rules, and
extract patterns that match their focus.

---

## Substitution Guide

Replace `{placeholders}` with values detected during structural analysis.
Use sensible defaults when a value isn't detected, or omit the line entirely.

Key placeholders:

| Placeholder | Example value |
|-------------|---------------|
| `{Name}` | `EP`, `INTG`, `shared` |
| `{componentsGlob}` | `micro-frontends/ep/src/components/**/*.tsx` |
| `{storiesGlob}` | `micro-frontends/ep/src/stories/**/*.stories.tsx` |
| `{testGlob}` | `micro-frontends/ep/src/**/*.spec.tsx` |
| `{sourceGlob}` | `micro-frontends/ep/src/**/*.{ts,tsx}` |
| `{mockGlob}` | `**/tests/mocks/**` |
| `{hooksGlob}` | `**/hooks/**`, `**/api/**` |
| `{testIdAttr}` | `data-qa` |
| `{testNamingPattern}` | `*.spec.tsx` |
| `{testRunner}` | `Jest`, `Vitest` |
| `{dataFetchLib}` | `@tanstack/react-query` |
| `{dataFetchVersion}` | `v5` |
| `{versionNote}` | `isPending, not isLoading` |
| `{stateLib}` | `Jotai` |
| `{stateHook}` | `useAtom` |
