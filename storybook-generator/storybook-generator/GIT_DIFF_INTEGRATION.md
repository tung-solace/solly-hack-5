# Git-Diff + Storybook Generator Integration

## Quick Reference Guide

### ✨ NEW: Fully Automatic Mode (Recommended!)

**Just run `/storybook-generator` with no arguments!**

```bash
# Fully automatic workflow:
/storybook-generator

# The skill will automatically:
# 1. Invoke /git-diff to analyze recent changes
# 2. Identify all components missing stories
# 3. Show you the list of components
# 4. Generate stories for all of them
# 5. Use git-diff context to avoid 404 errors
```

**Alternative: Specify components manually:**

```bash
# Manual workflow - specify component:
/storybook-generator src/components/multiFlow/FlowDetailCard.tsx

# The skill will automatically:
# 1. Check if component is in recent changes
# 2. Invoke /git-diff if needed
# 3. Use git-diff context to generate stories
# 4. Avoid 404 errors with correct handler paths
```

### Manual Workflow (Still Supported)

```bash
# Step 1: Analyze your recent changes
/git-diff

# Step 2: Generate stories with context
/storybook-generator src/components/multiFlow/FlowDetailCard.tsx
```

### How It Works

1. **Git-Diff Analysis** provides:
   - ✅ All React Query hooks used by the component
   - ✅ Actual API endpoints (after transformations)
   - ✅ Data transformation functions documented
   - ✅ Component dependencies (Jotai, Router, etc.)

2. **Storybook Generator** uses this context to:
   - ✅ Skip redundant API endpoint tracing
   - ✅ Generate correct MSW handler paths
   - ✅ Document transformations in comments
   - ✅ Avoid 404 errors from incorrect handler URLs

### When Git-Diff Integration Activates

✅ **AUTOMATICALLY ACTIVATES when:**
- Component was recently added or modified (in git status)
- Component has complex API calls with transformations
- Component uses multiple React Query hooks
- Running `/storybook-generator` on changed files

⏭️ **AUTOMATICALLY SKIPPED when:**
- Component is very simple (no API calls detected)
- Component hasn't changed recently
- Git-diff was already run manually in this session
- Generating stories for stable, unchanged components

### Example: FlowDetailCard

**Old workflow (manual git-diff):**
```bash
/git-diff
/storybook-generator src/components/multiFlow/FlowDetailCard.tsx

# ✅ Git-diff identifies: splitMITypesString("ibmmq-source") → "ibmmq"
# ✅ Generates correct handler: /microIntegrationTypes/ibmmq
# ✅ Documents transformation in comments
```

**✨ NEW workflow (automatic git-diff):**
```bash
/storybook-generator src/components/multiFlow/FlowDetailCard.tsx

# ✅ Automatically detects component in recent changes
# ✅ Automatically invokes /git-diff
# ✅ Git-diff identifies: splitMITypesString("ibmmq-source") → "ibmmq"
# ✅ Generates correct handler: /microIntegrationTypes/ibmmq
# ✅ Documents transformation in comments
```

### What Gets Enhanced

| Phase | Enhancement |
|-------|-------------|
| **Phase 0** | Loads git-diff context if available |
| **Phase 1** | Uses git-diff dependency analysis |
| **Phase 2** | Skips redundant API tracing, uses git-diff endpoints |
| **Phase 4** | Prioritizes handlers identified by git-diff |
| **Phase 6** | Better story variants based on props changes |

### Output Example

When git-diff context is used, you'll see:

```
✓ Generated story for FlowDetailCard component

Git-Diff Context:
✓ Used git-diff pre-analysis
✓ API endpoints identified: useGetConnectorTypeById, useEnvironments
✓ Transformation documented: splitMITypesString("ibmmq-source") → "ibmmq"

Component Analysis:
- Type: Card with Selection (Medium Complexity)
...

Handler Strategy:
- All handlers match actual endpoints (verified via git-diff)
...
```

### Troubleshooting

**Problem: Still getting 404 errors**
- ✅ Git-diff should invoke automatically for changed components
- Check that transformations are documented in generated story
- Verify component is in `git status` output
- Try running `/git-diff` manually first if needed

**Problem: Git-diff not activating automatically**
- Ensure component is in recent git changes (run `git status`)
- Check if git-diff was already run manually (automatic invocation skipped)
- Verify git repository detected

**Problem: Want to skip automatic git-diff integration**
- Component must not be in recent changes
- Or run `/git-diff` manually first (automatic invocation will be skipped)
- The skill will fall back to manual analysis

### Best Practices

1. **Trust automatic detection** - the skill will invoke git-diff when needed
2. **Review transformation comments** in generated stories
3. **Verify handler paths** match actual API endpoints
4. **Test stories** in Storybook to catch any issues
5. **For batch processing**, just list all components - git-diff runs once automatically

### Key Benefits

- 🚀 **Faster story generation** (skips redundant analysis)
- ✅ **Correct handler paths** (avoids 404 errors)
- 📝 **Better documentation** (transformation functions explained)
- 🎯 **More accurate stories** (leverages change context)

---

## Advanced Usage

### Batch Generation with Automatic Context

```bash
# ✨ NEW: Automatic git-diff for batch processing
/storybook-generator src/components/A.tsx src/components/B.tsx

# Git-diff automatically invoked ONCE for all components
# All components share the same git-diff context
```

### Recursive Generation with Automatic Context

```bash
# ✨ NEW: Automatic git-diff for recursive generation
/storybook-generator src/components/multiFlow/ --recursive

# Git-diff automatically invoked if components in recent changes
# Analyzes all components in directory
```

### Update Existing Stories with Automatic Context

```bash
# ✨ NEW: Automatic git-diff when updating
/storybook-generator src/components/Foo.tsx --update

# Git-diff automatically invoked if component changed recently
# Updates story with new context and transformations
```

### Manual Control (Optional)

```bash
# If you prefer manual control, run git-diff first:
/git-diff

# Then generate - automatic invocation will be skipped:
/storybook-generator src/components/Foo.tsx
```

---

## Technical Details

### Phase 0: Contextual Pre-Analysis

When `/git-diff` has been run previously in the session:

1. Skill checks if component is in git changes
2. Extracts API endpoints from git-diff analysis
3. Loads transformation functions documented
4. Uses this context in Phases 2, 4, and 6

### Phase 2: Smart API Tracing

If git-diff context exists:
- Uses pre-identified endpoints
- Skips manual hook tracing
- Verifies transformations are documented

If NO context:
- Falls back to manual tracing
- Searches for hook definitions
- Traces transformations step-by-step

### Phase 4: Handler Discovery

With git-diff:
- Prioritizes endpoints from git-diff
- Creates handlers for actual endpoints
- Documents transformations inline

Without git-diff:
- Searches existing mock files
- May miss transformation nuances
- Higher risk of incorrect paths

---

## Summary

✨ **✨ NEW: Fully Automatic Mode!** Just run `/storybook-generator` with no arguments!

✅ **Simplest Workflow:** `/storybook-generator`
- Automatically runs git-diff
- Identifies all components missing stories
- Shows you the list
- Generates stories for all of them

✅ **Manual Component Selection:** `/storybook-generator src/components/Foo.tsx`
- Automatically detects if component is in recent changes
- Invokes git-diff if needed
- Uses git-diff context for accurate generation

✅ **Batch Workflow:** `/storybook-generator src/components/A.tsx src/components/B.tsx`
- Git-diff runs once for all components
- All components share the same context

✅ **Automatic Benefits:**
- No manual git-diff invocation needed
- Avoids 404 errors from incorrect handler paths
- Provides better documentation and faster generation
- Works seamlessly for single and multiple files

✅ **Manual Workflow Still Supported:** `/git-diff` → `/storybook-generator`
