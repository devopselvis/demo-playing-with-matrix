# Demo: Playing with GitHub Actions Matrix

This repository demonstrates how to work around GitHub Actions' 256 matrix job limit using `fromJSON` and batching strategies, with practical examples for iOS monorepo testing scenarios.

## The Problem

GitHub Actions limits matrix jobs to a maximum of 256 combinations per workflow. For large monorepos with hundreds of components, this becomes a significant constraint when you need to run tests across all components.

## The Solution

This repository demonstrates **three complementary strategies** to exceed the 256 job limit:

### 1. **Dynamic Matrix with fromJSON**
Use `fromJSON()` to dynamically generate matrix values from job outputs, enabling runtime-based matrix configuration.

### 2. **Batching Strategy**
Split large test suites into multiple batches (each ≤256 jobs) that run in parallel as separate matrix jobs.

### 3. **Reusable Workflows**
Combine reusable workflows with dynamic matrices to create maintainable, scalable test infrastructure.

## Workflows in This Repository

### Original Examples
- **`matrix-256.yml`** - A workflow with exactly 256 matrix combinations (16×16)
- **`matrix-260.yml`** - A workflow exceeding the limit (13×20 = 260) - **will fail**

### Solution Workflows

#### `reusable-test.yml`
A reusable workflow template for running component tests. This demonstrates:
- How to create a reusable workflow that can be called with different inputs
- Proper input validation and parameter passing
- Standardized test execution pattern

**Usage:**
```yaml
uses: ./.github/workflows/reusable-test.yml
with:
  component: 'component-1'
  batch: '1'
```

#### `dynamic-matrix-batched.yml`
Demonstrates the batching strategy using `fromJSON` with inline test execution.

**Features:**
- Dynamic matrix generation based on test scope (single, all, custom)
- Automatic splitting into batches of 256 jobs each
- Support for 300+ components (demonstrated with 3 batches = 768 potential jobs)
- Conditional batch execution (batches only run if needed)
- Aggregated summary reporting

**How to run:**
```bash
# Test a single component
gh workflow run dynamic-matrix-batched.yml -f test_scope=single

# Test all components (300+, exceeds 256 limit)
gh workflow run dynamic-matrix-batched.yml -f test_scope=all

# Test custom component list
gh workflow run dynamic-matrix-batched.yml -f test_scope=custom -f component_filter="component-1,component-5,component-10"
```

#### `dynamic-matrix-reusable.yml`
Combines reusable workflows with dynamic matrices and batching.

**Features:**
- Calls the reusable workflow with matrix strategy
- Configurable component count (default: 300)
- Automatic batch calculation and distribution
- Clean separation of concerns (generation vs. execution)

**How to run:**
```bash
# Test 300 components (2 batches)
gh workflow run dynamic-matrix-reusable.yml -f num_components=300

# Test 100 components (1 batch)
gh workflow run dynamic-matrix-reusable.yml -f num_components=100

# Test 600 components (3 batches)
gh workflow run dynamic-matrix-reusable.yml -f num_components=600
```

#### `ios-monorepo-example.yml`
A realistic iOS monorepo testing scenario demonstrating the complete solution.

**Features:**
- Smart test scope determination based on which component was modified
- Core component detection (modifying core components triggers all tests)
- Automatic batching for 300 components
- Simulated iOS test execution (XCTest pattern)
- Comprehensive test reporting

**Business logic:**
- **Single component modified**: Run tests only for that component
- **Core component modified**: Run tests for ALL 300 components
- **Manual trigger**: Choose to test all or specific components

**How to run:**
```bash
# Test only the modified component
gh workflow run ios-monorepo-example.yml \
  -f trigger_type=modified_single_component \
  -f modified_component=component-42

# Simulate core component change (tests all 300)
gh workflow run ios-monorepo-example.yml \
  -f trigger_type=modified_core_component

# Manually test all components
gh workflow run ios-monorepo-example.yml \
  -f trigger_type=manual_all_components
```

#### `advanced-file-based-matrix.yml`
An advanced example showing git-based change detection and file-based component mapping.

**Features:**
- Git diff analysis to determine changed files (simulated)
- File-to-component mapping logic
- Support for 400+ components across 4 batches
- Three different strategies:
  - `changed_files`: Test only components affected by changes
  - `all_components`: Test all 400 components (2 batches)
  - `specific_batch`: Test a specific batch by number
- Demonstrates real-world monorepo change detection patterns

**How to run:**
```bash
# Test only changed components (based on git diff)
gh workflow run advanced-file-based-matrix.yml \
  -f strategy=changed_files

# Test all 400 components
gh workflow run advanced-file-based-matrix.yml \
  -f strategy=all_components

# Test specific batch (e.g., batch 2 = components 257-512)
gh workflow run advanced-file-based-matrix.yml \
  -f strategy=specific_batch \
  -f batch_number=2
```

## Key Techniques Demonstrated

### 1. Using `fromJSON()` for Dynamic Matrices

```yaml
jobs:
  generate-matrix:
    outputs:
      matrix: ${{ steps.create.outputs.components }}
    steps:
      - id: create
        run: |
          # Generate JSON array dynamically
          components='["component-1","component-2","component-3"]'
          echo "components=$components" >> $GITHUB_OUTPUT

  test:
    needs: generate-matrix
    strategy:
      matrix:
        component: ${{ fromJSON(needs.generate-matrix.outputs.matrix) }}
    steps:
      - run: echo "Testing ${{ matrix.component }}"
```

### 2. Batching to Exceed 256 Limit

```yaml
jobs:
  # Generate multiple batch outputs
  generate:
    outputs:
      batch1: ${{ steps.create.outputs.batch1 }}  # Components 1-256
      batch2: ${{ steps.create.outputs.batch2 }}  # Components 257-512
      has_batch2: ${{ steps.create.outputs.has_batch2 }}

  # Each batch runs separately
  test-batch-1:
    strategy:
      matrix:
        component: ${{ fromJSON(needs.generate.outputs.batch1) }}
    # ... max 256 jobs

  test-batch-2:
    if: needs.generate.outputs.has_batch2 == 'true'
    strategy:
      matrix:
        component: ${{ fromJSON(needs.generate.outputs.batch2) }}
    # ... another max 256 jobs
```

### 3. Conditional Batch Execution

```yaml
jobs:
  test-batch-2:
    needs: generate-matrix
    # Only run if this batch has components
    if: needs.generate-matrix.outputs.has_batch2 == 'true'
    strategy:
      matrix:
        component: ${{ fromJSON(needs.generate-matrix.outputs.batch2) }}
```

### 4. Combining with Reusable Workflows

```yaml
jobs:
  call-reusable:
    strategy:
      matrix:
        component: ${{ fromJSON(needs.generate.outputs.batch1) }}
    uses: ./.github/workflows/reusable-test.yml
    with:
      component: ${{ matrix.component }}
```

## Advantages of This Approach

1. **Scalability**: Can handle unlimited components by adding more batches
2. **Flexibility**: Dynamic matrix generation based on runtime conditions
3. **Efficiency**: Only run tests for affected components when possible
4. **Maintainability**: Reusable workflows reduce duplication
5. **Visibility**: Clear batch organization and comprehensive reporting
6. **Parallel Execution**: All batches run concurrently for maximum speed

## Limitations and Considerations

1. **Runner Availability**: Ensure sufficient GitHub-hosted or self-hosted runners
2. **Rate Limits**: Be aware of API rate limits with large job counts
3. **Execution Time**: More jobs = longer overall workflow time (mitigated by parallelization)
4. **Cost**: GitHub-hosted runners incur costs based on minutes used
5. **Max Parallel**: Use `max-parallel` to limit concurrent jobs and avoid overwhelming infrastructure

## Best Practices

1. **Use `fail-fast: false`** in matrix strategy to see all failures
2. **Implement `max-parallel`** to control concurrency
3. **Add comprehensive summary jobs** to aggregate results
4. **Use conditional execution** to skip empty batches
5. **Cache dependencies** to speed up individual jobs
6. **Monitor costs** when running hundreds of jobs

## Real-World Application

For the iOS monorepo scenario:
1. Detect which component(s) were modified
2. Determine if modified component is "core" (affects everything)
3. Generate appropriate test matrix:
   - Single component: 1 job
   - Core component: 300 jobs (split into 2 batches)
4. Execute tests in parallel across batches
5. Aggregate and report results

This approach provides the flexibility to run targeted tests for efficiency while maintaining the ability to run comprehensive test suites when needed.

## References

- [GitHub Actions: fromJSON](https://docs.github.com/en/actions/reference/workflows-and-actions/expressions#fromjson)
- [GitHub Actions: Matrix Strategy](https://docs.github.com/en/actions/using-jobs/using-a-matrix-for-your-jobs)
- [GitHub Actions: Reusable Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)