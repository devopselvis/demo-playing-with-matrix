# Quick Reference: Exceeding GitHub Actions 256 Matrix Job Limit

## The Problem
GitHub Actions limits matrix jobs to 256 combinations. This is a hard limit per job.

## The Solution
Use **fromJSON** + **batching** to split work across multiple jobs, each with ≤256 matrix combinations.

## Quick Start

### Basic Pattern

```yaml
jobs:
  # Step 1: Generate matrix as JSON
  generate:
    outputs:
      batch1: ${{ steps.create.outputs.batch1 }}
      batch2: ${{ steps.create.outputs.batch2 }}
    steps:
      - id: create
        run: |
          # First 256 items
          batch1='["item-1", "item-2", ..., "item-256"]'
          echo "batch1=$batch1" >> $GITHUB_OUTPUT
          
          # Next 256 items
          batch2='["item-257", "item-258", ..., "item-512"]'
          echo "batch2=$batch2" >> $GITHUB_OUTPUT

  # Step 2: Use fromJSON in matrix
  test-batch-1:
    needs: generate
    strategy:
      matrix:
        item: ${{ fromJSON(needs.generate.outputs.batch1) }}
    steps:
      - run: echo "Testing ${{ matrix.item }}"

  test-batch-2:
    needs: generate
    strategy:
      matrix:
        item: ${{ fromJSON(needs.generate.outputs.batch2) }}
    steps:
      - run: echo "Testing ${{ matrix.item }}"
```

## Workflows in This Repository

| Workflow | Purpose | Max Components |
|----------|---------|----------------|
| `reusable-test.yml` | Template for component tests | N/A (reusable) |
| `dynamic-matrix-batched.yml` | Basic fromJSON + batching | 768 (3 batches) |
| `dynamic-matrix-reusable.yml` | Reusable workflow + batching | 768 (3 batches) |
| `ios-monorepo-example.yml` | Realistic iOS monorepo scenario | 300 (2 batches) |
| `advanced-file-based-matrix.yml` | Git diff + file mapping | 1024 (4 batches) |

## Common Patterns

### 1. Conditional Batches

Only run batches that have components:

```yaml
jobs:
  generate:
    outputs:
      has_batch2: ${{ steps.create.outputs.has_batch2 }}
  
  test-batch-2:
    if: needs.generate.outputs.has_batch2 == 'true'
    # ...
```

### 2. Dynamic Batch Count

Calculate batches based on actual count:

```bash
total=400
batch_size=256
num_batches=$(( (total + batch_size - 1) / batch_size ))
```

### 3. Max Parallel Control

Limit concurrent jobs to avoid overwhelming runners:

```yaml
strategy:
  fail-fast: false
  max-parallel: 50
  matrix:
    component: ${{ fromJSON(...) }}
```

### 4. Reusable Workflow with Matrix

```yaml
call-reusable:
  strategy:
    matrix:
      component: ${{ fromJSON(needs.generate.outputs.batch1) }}
  uses: ./.github/workflows/reusable-test.yml
  with:
    component: ${{ matrix.component }}
```

## Testing Strategies

### Test Only Changed Components
```bash
gh workflow run advanced-file-based-matrix.yml \
  -f strategy=changed_files
```

### Test All Components
```bash
gh workflow run dynamic-matrix-batched.yml \
  -f test_scope=all
```

### Test Specific Components
```bash
gh workflow run dynamic-matrix-batched.yml \
  -f test_scope=custom \
  -f component_filter="component-1,component-5,component-10"
```

## Shell Script Tips

### Generate JSON Array in Bash

```bash
# Method 1: Loop with counter
components="["
for i in {1..300}; do
  if [ $i -gt 1 ]; then components+=","; fi
  components+="\"component-$i\""
done
components+="]"

# Method 2: Using printf and paste
printf '"%s"\n' component-{1..300} | \
  paste -sd, - | \
  sed 's/^/[/;s/$/]/' > components.json
```

### Split into Batches

```bash
total=400
batch_size=256

for batch in {1..2}; do
  start=$(( (batch - 1) * batch_size + 1 ))
  end=$(( batch * batch_size ))
  
  json="["
  for i in $(seq $start $end); do
    if [ $i -gt $start ]; then json+=","; fi
    json+="\"component-$i\""
  done
  json+="]"
  
  echo "batch$batch=$json" >> $GITHUB_OUTPUT
done
```

## Limitations to Consider

| Aspect | Limit | Workaround |
|--------|-------|------------|
| Matrix combinations per job | 256 | Use multiple jobs (batching) |
| Output size | ~1 MB | Compress JSON or use artifacts |
| Jobs per workflow | 500 | Split into multiple workflows |
| Concurrent jobs | Varies | Use `max-parallel` |
| Workflow run time | 6 hours | Optimize test execution |

## Best Practices

1. ✅ Use `fail-fast: false` to see all failures
2. ✅ Set appropriate `max-parallel` limits
3. ✅ Add summary jobs for aggregated reporting
4. ✅ Use conditional batch execution
5. ✅ Cache dependencies to speed up jobs
6. ✅ Monitor runner availability and costs
7. ✅ Consider self-hosted runners for large-scale testing

## Common Pitfalls

❌ **Don't** exceed GitHub output size limits (~1 MB)
✅ **Do** compress large JSON or use artifacts

❌ **Don't** generate unnecessary batches
✅ **Do** use conditional execution for empty batches

❌ **Don't** run unlimited concurrent jobs
✅ **Do** set `max-parallel` to protect infrastructure

❌ **Don't** forget to aggregate results
✅ **Do** add summary jobs that check all batches

## Example: 500 Components Across 2 Batches

```yaml
jobs:
  generate:
    steps:
      - run: |
          # Batch 1: 1-256
          b1="["; for i in {1..256}; do
            [ $i -gt 1 ] && b1+=","
            b1+="\"c-$i\""
          done; b1+="]"
          echo "batch1=$b1" >> $GITHUB_OUTPUT
          
          # Batch 2: 257-500
          b2="["; for i in {257..500}; do
            [ $i -gt 257 ] && b2+=","
            b2+="\"c-$i\""
          done; b2+="]"
          echo "batch2=$b2" >> $GITHUB_OUTPUT

  test-1:
    strategy:
      matrix:
        c: ${{ fromJSON(needs.generate.outputs.batch1) }}
    steps:
      - run: test ${{ matrix.c }}

  test-2:
    strategy:
      matrix:
        c: ${{ fromJSON(needs.generate.outputs.batch2) }}
    steps:
      - run: test ${{ matrix.c }}
```

## Resources

- [GitHub Actions: fromJSON](https://docs.github.com/en/actions/learn-github-actions/expressions#fromjson)
- [GitHub Actions: Matrix Strategy](https://docs.github.com/en/actions/using-jobs/using-a-matrix-for-your-jobs)
- [GitHub Actions: Reusable Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [GitHub Actions: Usage Limits](https://docs.github.com/en/actions/learn-github-actions/usage-limits-billing-and-administration)
