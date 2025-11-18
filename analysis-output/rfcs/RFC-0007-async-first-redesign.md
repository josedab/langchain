# RFC-0007: Async-First Core Redesign

**Status:** Draft
**Author:** Codebase Analysis
**Created:** 2025-11-18
**Effort:** 6+ weeks
**Category:** Long-term

## Summary

Redesign the Runnable core to be async-first, reducing overhead for async workloads and providing better performance for production deployments.

## Motivation

Current architecture is sync-first with async wrappers:

```python
# Current: sync is primary, async calls sync
def invoke(self, input, config=None):
    # Primary implementation
    ...

async def ainvoke(self, input, config=None):
    # Wraps sync in thread pool
    return await run_in_executor(None, self.invoke, input, config)
```

Problems:
1. **Thread pool overhead**: Every async call spawns a thread
2. **Context switching**: Thread pool adds latency
3. **Memory overhead**: Thread stacks consume memory
4. **Blocking operations**: Some sync operations block the event loop
5. **Inconsistent behavior**: sync and async may behave differently

Most production deployments are async (FastAPI, aiohttp, etc.), so async should be first-class.

## Detailed Design

### New Architecture

```python
# Proposed: async is primary, sync wraps async
class Runnable(ABC, Generic[Input, Output]):
    @abstractmethod
    async def ainvoke(
        self,
        input: Input,
        config: RunnableConfig | None = None
    ) -> Output:
        """Primary implementation - async."""
        ...

    def invoke(
        self,
        input: Input,
        config: RunnableConfig | None = None
    ) -> Output:
        """Sync wrapper around async."""
        return asyncio.run(self.ainvoke(input, config))
```

### Migration Strategy

#### Phase 1: Add Async Primary Flag

```python
class Runnable(ABC):
    _async_primary: ClassVar[bool] = False  # Default to current behavior

    @abstractmethod
    def _invoke(self, input, config):
        """Override this for sync-first."""
        ...

    @abstractmethod
    async def _ainvoke(self, input, config):
        """Override this for async-first."""
        ...

    def invoke(self, input, config=None):
        if self._async_primary:
            return asyncio.run(self._ainvoke(input, config))
        return self._invoke(input, config)

    async def ainvoke(self, input, config=None):
        if self._async_primary:
            return await self._ainvoke(input, config)
        return await run_in_executor(None, self._invoke, input, config)
```

#### Phase 2: Migrate Core Classes

```python
class RunnableSequence(RunnableSerializable[Input, Output]):
    _async_primary = True  # Enable async-first

    async def _ainvoke(self, input, config=None):
        config = ensure_config(config)
        callback_manager = get_async_callback_manager_for_config(config)

        run_manager = await callback_manager.on_chain_start(
            dumpd(self), input, name=self.get_name()
        )

        try:
            for i, step in enumerate(self.steps):
                input = await step.ainvoke(input, config)

            await run_manager.on_chain_end(input)
            return input
        except Exception as e:
            await run_manager.on_chain_error(e)
            raise
```

#### Phase 3: Async Callback Manager

```python
class AsyncCallbackManager:
    """Native async callback manager."""

    handlers: list[BaseCallbackHandler]

    async def on_chain_start(self, serialized, inputs, **kwargs):
        await asyncio.gather(*[
            self._call_handler(h.on_chain_start, serialized, inputs, **kwargs)
            for h in self.handlers
        ])

    async def _call_handler(self, method, *args, **kwargs):
        if asyncio.iscoroutinefunction(method):
            await method(*args, **kwargs)
        else:
            await run_in_executor(None, method, *args, **kwargs)
```

### Performance Benefits

| Operation | Current | Async-First | Improvement |
|-----------|---------|-------------|-------------|
| Simple chain (3 steps) | 450μs | 150μs | 3x |
| Parallel (3 branches) | 620μs | 200μs | 3x |
| With callbacks (5) | 1200μs | 400μs | 3x |
| Batch (10 items) | 800μs | 300μs | 2.7x |

*Estimated improvements based on removing thread pool overhead.*

### Compatibility Layer

```python
class SyncRunnable(Runnable[Input, Output]):
    """Mixin for legacy sync-first runnables."""

    _async_primary = False

    def _invoke(self, input, config=None):
        # Subclasses implement this
        raise NotImplementedError

    async def _ainvoke(self, input, config=None):
        return await run_in_executor(None, self._invoke, input, config)
```

### New Async Utilities

```python
# libs/core/langchain_core/utils/async_utils.py

async def gather_with_concurrency(
    n: int | None,
    *coros: Coroutine
) -> list:
    """Gather coroutines with concurrency limit."""
    if n is None:
        return await asyncio.gather(*coros)

    semaphore = asyncio.Semaphore(n)

    async def limited(coro):
        async with semaphore:
            return await coro

    return await asyncio.gather(*[limited(c) for c in coros])

async def as_completed_with_concurrency(
    n: int | None,
    coros: list[Coroutine]
) -> AsyncIterator:
    """Yield results as they complete with concurrency limit."""
    ...
```

## Example Usage

### Before (Current)

```python
# Async calls thread pool for each operation
async def process(items):
    results = []
    for item in items:
        # Each ainvoke spawns a thread
        result = await chain.ainvoke(item)
        results.append(result)
    return results
```

### After (Async-First)

```python
# Native async, no thread pool
async def process(items):
    # Fully async, no threads
    return await chain.abatch(items)

# Or with streaming
async for event in chain.astream_events(input):
    # Native async events
    yield event
```

### Custom Async Runnable

```python
class AsyncDatabaseLookup(Runnable[str, dict]):
    _async_primary = True

    async def _ainvoke(self, query: str, config=None) -> dict:
        # Native async database call
        async with aiopg.connect(self.dsn) as conn:
            async with conn.cursor() as cur:
                await cur.execute("SELECT * FROM data WHERE id = %s", (query,))
                return await cur.fetchone()
```

## Implementation Plan

| Phase | Task | Duration |
|-------|------|----------|
| 1 | Design async-first architecture | 1 week |
| 2 | Add _async_primary flag | 3 days |
| 3 | Migrate RunnableSequence | 1 week |
| 4 | Migrate RunnableParallel | 3 days |
| 5 | Migrate RunnableLambda | 3 days |
| 6 | Async callback manager | 1 week |
| 7 | Update all core runnables | 1 week |
| 8 | Performance testing | 3 days |
| 9 | Documentation | 3 days |

**Total:** 6+ weeks

## Backwards Compatibility

### Breaking Changes

1. **Subclasses must implement async**: Custom Runnables need to implement `_ainvoke`
2. **Callback timing**: Async callbacks may fire in different order
3. **Thread-local state**: May not work as expected

### Migration Path

1. **v1.2**: Add `_async_primary` flag (opt-in)
2. **v1.3**: Migrate core classes to async-first
3. **v2.0**: Make async-first default; deprecate sync-first

### Compatibility Wrapper

```python
# For legacy sync implementations
class LegacyRunnable(SyncRunnable):
    def _invoke(self, input, config=None):
        # Old sync code works unchanged
        return self.legacy_method(input)
```

## Alternatives Considered

### Alternative 1: Keep current architecture
**Rejected:** Thread pool overhead is significant for high-throughput async apps.

### Alternative 2: Use anyio for sync/async bridge
**Rejected:** Adds dependency; doesn't solve fundamental issue.

### Alternative 3: Separate sync and async classes
**Rejected:** Code duplication; harder to maintain.

## Open Questions

1. **Event loop policy**: How to handle `asyncio.run()` in sync wrapper?
2. **Nested event loops**: Support for Jupyter notebooks?
3. **Thread safety**: How to handle sync calls from multiple threads?
4. **Cancellation**: How to propagate async cancellation?

## Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Performance regression | High | Extensive benchmarking |
| Breaking user code | High | Long deprecation period |
| Callback compatibility | Medium | Compatibility layer |
| Event loop issues | Medium | Clear documentation |

## Success Criteria

- [ ] 3x reduction in async operation overhead
- [ ] All core classes migrated
- [ ] Benchmarks show no regressions
- [ ] Migration guide published
- [ ] Partner packages updated
- [ ] No breaking changes without deprecation

## References

- [Python asyncio Best Practices](https://docs.python.org/3/library/asyncio-dev.html)
- [Designing for Performance](https://docs.aiohttp.org/en/stable/performance.html)
- [anyio - Async library compatibility](https://anyio.readthedocs.io/)
- [Trio - Async I/O design](https://trio.readthedocs.io/)
