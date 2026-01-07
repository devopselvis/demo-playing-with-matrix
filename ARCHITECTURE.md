# Solution Architecture

## Overview Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    GitHub Actions Workflow                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────────────────────────────────────┐       │
│  │  Job 1: Generate Matrix                              │       │
│  │  ─────────────────────                               │       │
│  │  • Determine components to test                      │       │
│  │  • Split into batches of 256                         │       │
│  │  • Output JSON arrays: batch1, batch2, batch3...     │       │
│  └────────────┬─────────────────────────────────────────┘       │
│               │                                                  │
│               │ outputs: batch1, batch2, has_batch1, etc.       │
│               │                                                  │
│  ┌────────────▼──────────────────────────────────────┐          │
│  │  Job 2: Test Batch 1 (if has_batch1 == 'true')   │          │
│  │  ─────────────────────                            │          │
│  │  strategy:                                         │          │
│  │    matrix:                                         │          │
│  │      component: ${{ fromJSON(batch1) }}            │          │
│  │                                                    │          │
│  │  Runs up to 256 parallel jobs:                    │          │
│  │  ┌──────┐ ┌──────┐ ┌──────┐     ┌──────┐         │          │
│  │  │comp-1│ │comp-2│ │comp-3│ ... │256   │         │          │
│  │  └──────┘ └──────┘ └──────┘     └──────┘         │          │
│  └───────────────────────────────────────────────────┘          │
│                                                                  │
│  ┌────────────────────────────────────────────────┐             │
│  │  Job 3: Test Batch 2 (if has_batch2 == 'true')│             │
│  │  ─────────────────────                         │             │
│  │  strategy:                                      │             │
│  │    matrix:                                      │             │
│  │      component: ${{ fromJSON(batch2) }}         │             │
│  │                                                 │             │
│  │  Runs up to 256 parallel jobs:                 │             │
│  │  ┌──────┐ ┌──────┐ ┌──────┐     ┌──────┐      │             │
│  │  │257   │ │258   │ │259   │ ... │512   │      │             │
│  │  └──────┘ └──────┘ └──────┘     └──────┘      │             │
│  └────────────────────────────────────────────────┘             │
│                                                                  │
│  ┌────────────────────────────────────────────────┐             │
│  │  Job 4: Summary (always runs)                  │             │
│  │  ─────────────────                             │             │
│  │  • Collect results from all batches            │             │
│  │  • Report overall status                       │             │
│  │  • Fail if any batch failed                    │             │
│  └────────────────────────────────────────────────┘             │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

## Key Components

### 1. Matrix Generation Job

**Purpose**: Generate dynamic JSON arrays for each batch

**Inputs**:
- Test scope (single, all, custom)
- Component filters
- Changed files (for git-based detection)

**Outputs**:
```yaml
batch1: '["component-1", "component-2", ..., "component-256"]'
batch2: '["component-257", "component-258", ..., "component-512"]'
has_batch1: 'true'
has_batch2: 'true'
has_batch3: 'false'
```

### 2. Test Batch Jobs

**Purpose**: Execute tests for components in each batch

**Configuration**:
```yaml
strategy:
  fail-fast: false        # Continue even if some jobs fail
  max-parallel: 50        # Limit concurrent jobs
  matrix:
    component: ${{ fromJSON(needs.generate.outputs.batch1) }}
```

**Execution**: Each component runs as a separate matrix job

### 3. Summary Job

**Purpose**: Aggregate results and report overall status

**Conditions**: `if: always()` - runs even if previous jobs fail

**Responsibilities**:
- Check status of all batch jobs
- Generate comprehensive report
- Exit with failure if any batch failed

## Workflow Comparison

| Workflow | Components | Batches | Use Case |
|----------|-----------|---------|----------|
| `dynamic-matrix-batched.yml` | 300 | 2 | Basic demonstration |
| `dynamic-matrix-reusable.yml` | Configurable | Auto | Production-ready template |
| `ios-monorepo-example.yml` | 300 | 2 | iOS monorepo scenario |
| `advanced-file-based-matrix.yml` | 400 | 2-4 | Git diff detection |

## Scaling Examples

### 256 Components (1 Batch)
```
Batch 1: 256 jobs
Total:   256 jobs ✓
```

### 500 Components (2 Batches)
```
Batch 1: 256 jobs
Batch 2: 244 jobs
Total:   500 jobs ✓
```

### 1000 Components (4 Batches)
```
Batch 1: 256 jobs
Batch 2: 256 jobs
Batch 3: 256 jobs
Batch 4: 232 jobs
Total:   1000 jobs ✓
```

## Data Flow

```
User Input
    ↓
Generate Matrix Job
    ↓
Calculate Batches
    ↓
Create JSON Arrays
    ↓
Output to GitHub
    ↓
┌───────┬───────┬───────┐
│Batch 1│Batch 2│Batch 3│
└───┬───┴───┬───┴───┬───┘
    │       │       │
fromJSON()  │       │
    │   fromJSON()  │
    │       │   fromJSON()
    ↓       ↓       ↓
┌───────┬───────┬───────┐
│256    │256    │256    │
│jobs   │jobs   │jobs   │
└───┬───┴───┬───┴───┬───┘
    └───────┴───────┘
            ↓
      Summary Job
            ↓
    Overall Status
```

## Implementation Patterns

### Pattern 1: Simple Batching
- Fixed number of components (e.g., 300)
- Hardcoded batch splits
- Best for: Demos and simple use cases

### Pattern 2: Dynamic Batching
- Variable component count
- Calculated batch distribution
- Best for: Production workflows

### Pattern 3: Conditional Batching
- Only create batches with components
- Skip empty batches
- Best for: Optimizing execution time

### Pattern 4: Reusable Workflow Batching
- Call reusable workflow for each component
- Matrix expansion with workflow calls
- Best for: Standardized test patterns

## Performance Considerations

| Factor | Impact | Mitigation |
|--------|--------|------------|
| Runner availability | High | Use `max-parallel` |
| API rate limits | Medium | Batch size optimization |
| Network bandwidth | Low | Minimize data transfer |
| Job startup time | High | Optimize dependencies |
| Total execution time | Medium | Parallel execution |

## Example: 300 Component iOS Monorepo

```
Modified Component: component-5 (core component)
                ↓
        Trigger: ALL tests
                ↓
        Generate Matrix:
            - Batch 1: components 1-256
            - Batch 2: components 257-300
                ↓
        Execute:
            - Batch 1: 256 parallel jobs
            - Batch 2: 44 parallel jobs
                ↓
        Summary:
            - Batch 1: ✓ Success
            - Batch 2: ✓ Success
            - Overall: ✓ All tests passed
```

## Conclusion

This architecture enables:
- ✅ Exceeding 256 job limit through batching
- ✅ Dynamic matrix generation with `fromJSON()`
- ✅ Flexible test scope (single, all, custom)
- ✅ Scalable to thousands of components
- ✅ Maintainable with reusable workflows
- ✅ Production-ready patterns and best practices
