# Code Changes Analysis

A two-skill system for Claude Code that detects, classifies, and reports on git changes in any repository. It produces structured, actionable summaries that can feed into downstream skills like story generators or test generators.

## Skills

### `/git-diff`

Detects all git changes (unstaged, staged, untracked) and produces a categorized markdown report.

**Trigger phrases:** "detect changes", "show what changed", "analyze diff", "find modified components", "what needs stories", "review my changes"

**Usage:**

```
/git-diff                              # all uncommitted changes vs HEAD
/git-diff main                         # changes since diverging from main
/git-diff HEAD~3                       # last 3 commits
/git-diff --staged                     # staged changes only
/git-diff main -- src/components/      # scoped to a subdirectory
```

**What it does:**

1. Gathers raw changes via `git diff` and `git ls-files`
2. Reads repo-specific config from `.claude/git-diff.config.md` (falls back to sensible defaults)
3. Classifies files into categories: Components, Stories, Tests, Mocks, Hooks/API, Styles, Config, Docs, Other
4. Analyzes changed components — extracts props, hooks, imports, and test IDs
5. Checks story/test coverage for modified components
6. Outputs a structured markdown report with suggested next actions

### `/codebase-analyzer`

Explores a repository's structure and generates a purpose-specific config file for `/git-diff`.

**Trigger phrases:** "analyze codebase", "generate config", "setup git-diff", "explore repo structure"

**Usage:**

```
/codebase-analyzer                     # analyze current directory
/codebase-analyzer /path/to/repo       # analyze a specific repo
```

**What it does:**

1. Validates the target is a git repository
2. Asks what you want to track:
   - **Story coverage** — map components to stories, detect MSW mocks
   - **Test coverage** — map source files to tests, detect mock patterns
   - **API change tracking** — track API files, query keys, hook changes
   - **Refactoring impact** — analyze import graphs and breaking-change blast radius
   - **Custom** — define your own focus
3. Auto-detects repo layout, frameworks, state management, test patterns, Storybook config, and more
4. Generates `.claude/git-diff.config.md` tailored to your chosen purpose

## Recommended Workflow

```
# 1. One-time setup: analyze your repo and generate config
/codebase-analyzer

# 2. Recurring use: analyze changes whenever you need a summary
/git-diff
```

## Installation

Copy the skill directories into your project's `.claude/skills/` folder:

```
.claude/skills/
├── git-diff/
│   └── SKILL.md
└── codebase-analyzer/
    ├── SKILL.md
    └── references/
        └── config-templates.md
```

## How It Works

Both skills are **read-only** — they never modify your source code. The codebase-analyzer writes a single config file (`.claude/git-diff.config.md`), and git-diff produces reports as markdown output.

The config file tells git-diff how to classify files, what metadata to extract, and which downstream skills to suggest. Purpose-specific templates are stored in `codebase-analyzer/references/config-templates.md`.

## Detected Technologies

The codebase-analyzer can identify:

| Category | Examples |
|---|---|
| Repo layout | Monorepo, single-project |
| Frameworks | React, Vue, Angular, Svelte, Solid |
| State management | Jotai, Zustand, Redux, MobX, Recoil |
| Data fetching | @tanstack/react-query, SWR, Apollo, tRPC |
| Testing | Jest, Vitest, naming conventions |
| Storybook | Config version, story file patterns |
| Mocks | MSW handlers, mock directories |
| Test IDs | data-testid, data-qa, and other attributes |
