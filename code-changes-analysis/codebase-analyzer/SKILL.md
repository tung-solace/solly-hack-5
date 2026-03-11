---
name: codebase-analyzer
description: >
  Explore a repository's structure and generate a purpose-specific config for
  git-diff analysis. Use when the user asks to "analyze codebase",
  "generate config", "setup git-diff", "configure diff analysis",
  "explore repo structure", or "set up change tracking". Also triggered by
  git-diff when no config exists.
---

# Codebase Analyzer

Explore any git repository, ask the user what they care about, and generate a
purpose-specific `.claude/git-diff.config.md` so that `/git-diff` produces
tailored, actionable reports.

## Arguments

- `path` (optional) — path to the repository root. Defaults to the current
  working directory.

Examples:
```
/codebase-analyzer
/codebase-analyzer /path/to/repo
```

## Workflow

### 1. Determine Target Repo and Purpose

1. If `path` is provided, use it. Otherwise use the current working directory.
2. Validate the target is a git repo by checking for a `.git/` directory or
   running `git -C <path> rev-parse --git-dir`.
3. If not a git repo, tell the user and stop.
4. Present purpose options via an interactive prompt. Ask the user:

   > **What would you like to track with git-diff analysis?**
   >
   > 1. **Story coverage** — map components to stories, detect MSW mocks,
   >    feed into `/storybook-generator`
   > 2. **Test coverage** — map source files to tests, detect mock patterns,
   >    feed into `/test-generator`
   > 3. **API change tracking** — track API files, query keys, hook changes,
   >    and consumer impact
   > 4. **Refactoring impact** — analyze import graphs, shared utils, and
   >    breaking-change blast radius
   > 5. **Custom** — describe your own focus

5. Record the selected purpose for use in Step 4.

### 2. Structural Analysis

Run **read-only** commands to detect repo characteristics. Do not modify any
files. Collect results into a structured findings object.

#### 2a. Repo Layout

Detect monorepo indicators:

```bash
# Check for monorepo config files
ls lerna.json pnpm-workspace.yaml nx.json turbo.json 2>/dev/null

# Check for workspace directories
ls -d packages/ apps/ micro-frontends/ modules/ libs/ 2>/dev/null

# Count package.json files (monorepo signal if > 1)
find . -name "package.json" -not -path "*/node_modules/*" -maxdepth 4 | head -20
```

If monorepo indicators are found, identify each sub-project by locating
`package.json` files and their `name` fields.

#### 2b. Framework Detection

Read the root `package.json` (and sub-project `package.json` files in a
monorepo) to detect frameworks:

- **React**: `react`, `react-dom`, `next`, `gatsby`, `remix`
- **Vue**: `vue`, `nuxt`
- **Angular**: `@angular/core`
- **Svelte**: `svelte`, `@sveltejs/kit`
- **Solid**: `solid-js`

Record framework and version for each sub-project.

#### 2c. State Management

Search `package.json` dependencies for:

- `jotai`, `zustand`, `redux` / `@reduxjs/toolkit`, `mobx`, `recoil`,
  `pinia`, `valtio`, `xstate`, `@ngrx/store`

#### 2d. Data Fetching

Search `package.json` dependencies for:

- `@tanstack/react-query` or `react-query` (note version — v3 uses
  `isLoading`, v5 uses `isPending`)
- `swr`
- `@apollo/client` or `apollo-client`
- `@trpc/client`
- `axios`, `ky`, `got`

#### 2e. Test Patterns

```bash
# Detect test config
ls jest.config.* vitest.config.* cypress.config.* playwright.config.* 2>/dev/null

# Find test file naming convention
find . -not -path "*/node_modules/*" \( -name "*.test.*" -o -name "*.spec.*" \) -maxdepth 6 | head -20

# Detect test directories
find . -not -path "*/node_modules/*" -type d \( -name "__tests__" -o -name "tests" -o -name "test" \) -maxdepth 5 | head -20
```

Determine dominant pattern: `*.test.*` vs `*.spec.*`, co-located vs separate
directory.

#### 2f. Story Patterns

```bash
# Detect Storybook config
find . -not -path "*/node_modules/*" -type d -name ".storybook" -maxdepth 5

# Find story files
find . -not -path "*/node_modules/*" -name "*.stories.*" -maxdepth 6 | head -20
```

Determine: co-located with components vs separate `stories/` directory, naming
convention (`<Component>.stories.tsx` vs other).

#### 2g. Mock Patterns

```bash
# MSW detection
grep -r "setupWorker\|setupServer\|http\.get\|http\.post\|rest\.get\|rest\.post" \
  --include="*.ts" --include="*.tsx" --include="*.js" -l \
  -not -path "*/node_modules/*" | head -10

# MSW version from package.json
grep -h '"msw"' **/package.json package.json 2>/dev/null

# Mock directories
find . -not -path "*/node_modules/*" -type d \( -name "mocks" -o -name "__mocks__" -o -name "fixtures" \) -maxdepth 5 | head -20
```

#### 2h. API / Hook Patterns

```bash
# API and service directories
find . -not -path "*/node_modules/*" -type d \
  \( -name "api" -o -name "apis" -o -name "services" -o -name "hooks" -o -name "queries" \) \
  -maxdepth 5 | head -20

# Query key files
find . -not -path "*/node_modules/*" \
  \( -name "queryKeys*" -o -name "query-keys*" -o -name "keys.*" \) \
  -maxdepth 6 | head -10
```

#### 2i. Test ID Attribute

Sample up to 5 component files (`.tsx` / `.jsx`) and search for test ID
attributes:

```bash
grep -roh 'data-\(testid\|qa\|test\|cy\)' \
  --include="*.tsx" --include="*.jsx" \
  -not -path "*/node_modules/*" | sort | uniq -c | sort -rn | head -5
```

Use the most common attribute found.

#### 2j. Config Files

```bash
ls CLAUDE.md .claude/ tsconfig*.json .eslintrc* eslint.config.* \
  .prettierrc* prettier.config.* .editorconfig 2>/dev/null
```

Note any existing `.claude/git-diff.config.md` — it will be relevant in
Step 5.

### 3. Summarize Findings to User

Present a table of detected sub-projects (or the single project) with their
characteristics. Example:

```
## Detected Repository Structure

| Property | Value |
|----------|-------|
| Layout | Monorepo (pnpm workspaces) |
| Sub-projects | ep, intg, shared |
| Framework | React 18 |
| State | Jotai |
| Data fetching | @tanstack/react-query (ep: v5, intg: v3) |
| Tests | *.spec.tsx in co-located __tests__/ dirs |
| Stories | Separate stories/ dirs per MFE |
| Mocks | MSW v2 in tests/mocks/ |
| Test ID attr | data-qa |
| Storybook | .storybook/ in ep and intg |

**Selected purpose:** Story coverage
```

Ask the user to **confirm** before generating the config. If they want to
adjust the purpose or correct a detection, do so before proceeding.

### 4. Generate Purpose-Specific Config

Read [references/config-templates.md](references/config-templates.md) for the
template matching the selected purpose. Substitute detected values from Step 2
into the `{placeholder}` fields. Omit lines where a value wasn't detected.

### 5. Write the Config File

1. Determine the output path: `{repo-root}/.claude/git-diff.config.md`
2. If the `.claude/` directory does not exist, create it.
3. If a config file **already exists**, ask the user:
   > A config file already exists at `.claude/git-diff.config.md`.
   > Would you like to **overwrite** it or **merge** the new config into the
   > existing one?
   - **Overwrite**: replace the file entirely.
   - **Merge**: read the existing config, combine categories (avoiding
     duplicates), and merge sections. Prefer the newly detected values where
     there are conflicts, but preserve any custom additions the user made.
4. Write the final config content to the file.

### 6. Confirm and Suggest Next Steps

Tell the user the config was written and suggest next steps:

```
Config written to .claude/git-diff.config.md

Suggested next steps:
- Run `/git-diff` to analyze current uncommitted changes with the new config
- Run `/git-diff main` to see all changes since diverging from main
- Edit `.claude/git-diff.config.md` to fine-tune categories or patterns
```

## Edge Cases

- **Not a git repo**: Stop and inform the user.
- **Empty repo (no commits)**: Warn that diff analysis needs at least one
  commit; still generate the config based on file structure.
- **Very large monorepo**: Limit `find` depth and file counts to avoid slow
  scans. Use `head` to cap results.
- **No package.json**: Detect language from file extensions and adjust
  categories (e.g. Python, Go, Rust repos).
- **Existing config**: Always ask before overwriting. Offer merge option.
- **Permission errors**: If directories are unreadable, skip and note in the
  summary.
