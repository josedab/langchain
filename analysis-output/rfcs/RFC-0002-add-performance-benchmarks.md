# RFC-0002: Add Performance Benchmarks to CI

**Status:** Draft
**Author:** Codebase Analysis
**Created:** 2025-11-18
**Effort:** 4 dev-days
**Category:** Quick Win

## Summary

Add a performance benchmarking suite to CI that tracks key operation latencies, preventing performance regressions and establishing baselines for optimization work.

## Motivation

LangChain currently lacks:
- Performance regression detection
- Baseline metrics for optimization
- Visibility into overhead costs
- Comparison between versions

Without benchmarks:
- Performance regressions go unnoticed
- Optimization claims can't be verified
- Users can't make informed decisions
- Contributors lack performance feedback

## Detailed Design

### Benchmark Suite Structure

```
libs/core/benchmarks/
├── conftest.py           # Shared fixtures
├── bench_runnables.py    # Runnable operations
├── bench_callbacks.py    # Callback overhead
├── bench_config.py       # Config management
├── bench_serialization.py # Serialization
└── bench_messages.py     # Message creation
```

### Key Benchmarks

#### 1. Runnable Invocation Overhead

```python
# benchmarks/bench_runnables.py
import pytest

def bench_runnable_lambda_invoke(benchmark):
    """Measure RunnableLambda.invoke overhead."""
    from langchain_core.runnables import RunnableLambda

    runnable = RunnableLambda(lambda x: x)

    benchmark(runnable.invoke, "test")

def bench_runnable_sequence_invoke(benchmark):
    """Measure RunnableSequence overhead for simple chain."""
    from langchain_core.runnables import RunnableLambda

    chain = (
        RunnableLambda(lambda x: x)
        | RunnableLambda(lambda x: x)
        | RunnableLambda(lambda x: x)
    )

    benchmark(chain.invoke, "test")

def bench_runnable_parallel_invoke(benchmark):
    """Measure RunnableParallel overhead."""
    from langchain_core.runnables import RunnableParallel, RunnableLambda

    parallel = RunnableParallel(
        a=RunnableLambda(lambda x: x),
        b=RunnableLambda(lambda x: x),
        c=RunnableLambda(lambda x: x),
    )

    benchmark(parallel.invoke, "test")
```

#### 2. Callback System Overhead

```python
# benchmarks/bench_callbacks.py
def bench_callbacks_none(benchmark):
    """Baseline: invoke without callbacks."""
    runnable = RunnableLambda(lambda x: x)
    benchmark(runnable.invoke, "test")

def bench_callbacks_single(benchmark):
    """Invoke with single callback handler."""
    from langchain_core.callbacks import BaseCallbackHandler

    class NoOpHandler(BaseCallbackHandler):
        pass

    runnable = RunnableLambda(lambda x: x)

    benchmark(
        runnable.invoke,
        "test",
        config={"callbacks": [NoOpHandler()]}
    )

def bench_callbacks_multiple(benchmark):
    """Invoke with multiple callback handlers."""
    handlers = [NoOpHandler() for _ in range(5)]
    runnable = RunnableLambda(lambda x: x)

    benchmark(
        runnable.invoke,
        "test",
        config={"callbacks": handlers}
    )
```

#### 3. Config Management

```python
# benchmarks/bench_config.py
def bench_ensure_config_empty(benchmark):
    """Measure ensure_config with no config."""
    from langchain_core.runnables.config import ensure_config

    benchmark(ensure_config, None)

def bench_ensure_config_full(benchmark):
    """Measure ensure_config with full config."""
    from langchain_core.runnables.config import ensure_config

    config = {
        "tags": ["a", "b", "c"],
        "metadata": {"key": "value"},
        "callbacks": [],
        "max_concurrency": 5,
    }

    benchmark(ensure_config, config)

def bench_merge_configs(benchmark):
    """Measure config merging."""
    from langchain_core.runnables.config import merge_configs

    config1 = {"tags": ["a"], "metadata": {"x": 1}}
    config2 = {"tags": ["b"], "metadata": {"y": 2}}

    benchmark(merge_configs, config1, config2)
```

#### 4. Serialization

```python
# benchmarks/bench_serialization.py
def bench_serialize_chain(benchmark):
    """Measure chain serialization."""
    chain = prompt | model | parser  # Pre-built chain

    benchmark(chain.to_json)

def bench_deserialize_chain(benchmark):
    """Measure chain deserialization."""
    from langchain_core.load import loads

    json_str = '...'  # Pre-serialized chain

    benchmark(loads, json_str)
```

### CI Integration

```yaml
# .github/workflows/benchmark.yml
name: Performance Benchmarks

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  benchmark:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install uv
          cd libs/core && uv sync --group benchmark

      - name: Run benchmarks
        run: |
          cd libs/core
          pytest benchmarks/ \
            --benchmark-json=benchmark-results.json \
            --benchmark-compare \
            --benchmark-compare-fail=mean:10%

      - name: Store results
        uses: actions/upload-artifact@v3
        with:
          name: benchmark-results
          path: libs/core/benchmark-results.json

      - name: Comment on PR
        if: github.event_name == 'pull_request'
        uses: benchmark-action/github-action-benchmark@v1
        with:
          tool: 'pytest'
          output-file-path: libs/core/benchmark-results.json
          comment-on-alert: true
          alert-threshold: '110%'
```

### Reporting

```python
# Generate markdown report
def generate_report(results):
    report = "## Performance Benchmark Results\n\n"
    report += "| Benchmark | Mean | Std Dev | Ops/sec |\n"
    report += "|-----------|------|---------|--------|\n"

    for bench in results:
        report += f"| {bench.name} | {bench.mean:.3f}ms | {bench.stddev:.3f}ms | {bench.ops:.0f} |\n"

    return report
```

## Example Usage

### Running Locally

```bash
cd libs/core

# Install benchmark dependencies
uv sync --group benchmark

# Run all benchmarks
pytest benchmarks/ --benchmark-only

# Run specific benchmark
pytest benchmarks/bench_runnables.py -k "lambda"

# Compare with baseline
pytest benchmarks/ --benchmark-compare=baseline.json
```

### Viewing Results

```
--------------------- benchmark: 5 tests ---------------------
Name                          Mean      StdDev    Ops/sec
-------------------------------------------------------------
bench_runnable_lambda       0.15ms     0.02ms     6,667
bench_runnable_sequence     0.45ms     0.05ms     2,222
bench_runnable_parallel     0.62ms     0.08ms     1,613
bench_callbacks_none        0.12ms     0.01ms     8,333
bench_callbacks_single      0.25ms     0.03ms     4,000
-------------------------------------------------------------
```

## Implementation Plan

| Phase | Task | Duration |
|-------|------|----------|
| 1 | Create benchmark structure | 2 hours |
| 2 | Write core benchmarks | 4 hours |
| 3 | Set up pytest-benchmark | 2 hours |
| 4 | CI integration | 4 hours |
| 5 | Reporting & documentation | 4 hours |
| 6 | Establish baselines | 2 hours |

**Total:** 4 dev-days

## Backwards Compatibility

No impact on existing code. Benchmarks are additive.

## Alternatives Considered

### Alternative 1: Use asv (Airspeed Velocity)
**Rejected:** More complex setup; pytest-benchmark is simpler and sufficient.

### Alternative 2: Manual benchmarking
**Rejected:** Not reproducible; lacks CI integration.

### Alternative 3: Production profiling only
**Rejected:** Too late to catch regressions.

## Open Questions

1. **Storage**: Where to store historical results? (GitHub artifacts vs. external)
2. **Frequency**: Every PR or just main branch?
3. **Thresholds**: What regression percentage triggers failure?
4. **Hardware**: Should we use dedicated runners for consistency?

## Success Criteria

- [ ] Benchmarks run on every PR
- [ ] Results visible in PR comments
- [ ] <10% variance between runs
- [ ] Regressions >10% block merge
- [ ] Historical data accessible

## References

- [pytest-benchmark](https://pytest-benchmark.readthedocs.io/)
- [GitHub Action Benchmark](https://github.com/benchmark-action/github-action-benchmark)
- [Continuous Benchmarking Best Practices](https://bencher.dev/learn/benchmarking/)
