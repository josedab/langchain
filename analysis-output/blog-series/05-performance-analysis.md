# Performance Analysis and Optimization

> **Part 5 of 6** | Analysis based on commit [`990e346`](https://github.com/langchain-ai/langchain/tree/990e346c46251246966b562c8d6839903cb205ad)

## What You'll Learn

- LangChain's performance characteristics
- Overhead sources and mitigation
- Batching and concurrency optimization
- Caching strategies
- Streaming best practices
- Scaling considerations

## Introduction

When building production LLM applications, performance matters. In this post, we'll analyze where LangChain's performance overhead comes from and how to optimize for different workloads.

## Performance Characteristics

### Where Time Goes

In a typical LangChain application:

```
Total Request Time
├── Network latency to LLM provider: 80-95%
├── LangChain overhead: 2-10%
│   ├── Config management
│   ├── Callback dispatch
│   ├── Type validation
│   └── Schema introspection
└── Application logic: 3-10%
```

📌 **Key Insight**: For LLM workloads, network latency dominates. LangChain's overhead is typically negligible.

### Overhead Sources

#### 1. Configuration Management

Every invocation:

```python
# From: libs/core/langchain_core/runnables/config.py:192-242
def ensure_config(config: RunnableConfig | None = None) -> RunnableConfig:
    empty = RunnableConfig(
        tags=[],
        metadata={},
        callbacks=None,
        recursion_limit=DEFAULT_RECURSION_LIMIT,
        configurable={},
    )
    # Merge context config
    # Merge explicit config
    return empty
```

**Cost**: ~0.1ms per invocation
**Mitigation**: Reuse config objects

#### 2. Callback Dispatch

Creating and calling callback managers:

```python
# From: libs/core/langchain_core/callbacks/manager.py
callback_manager = get_callback_manager_for_config(config)
run_manager = callback_manager.on_chain_start(...)
# ... execution ...
run_manager.on_chain_end(result)
```

**Cost**: 0.5-2ms per chain step
**Mitigation**: Minimize callback handlers in hot paths

#### 3. Type Validation

Pydantic model validation:

```python
# Every Runnable validates its config
class MyRunnable(RunnableSerializable):
    temperature: float = Field(ge=0, le=2)
```

**Cost**: ~0.1ms per model
**Mitigation**: Pydantic v2 is fast; usually not a concern

#### 4. Schema Introspection

First-call cost for type inference:

```python
# From: libs/core/langchain_core/runnables/base.py:300-332
@property
def InputType(self) -> type[Input]:
    # Check Pydantic metadata
    # Check __orig_bases__
    # Fall back to Any
```

**Cost**: 1-5ms first call, cached thereafter
**Mitigation**: Warm up schemas at startup

## Batching Optimization

### How Batching Works

```python
# From: libs/core/langchain_core/runnables/base.py:863-920
def batch(
    self,
    inputs: list[Input],
    config: RunnableConfig | list[RunnableConfig] | None = None,
    *,
    return_exceptions: bool = False,
    **kwargs
) -> list[Output]:
    if not inputs:
        return []

    configs = get_config_list(config, len(inputs))

    with get_executor_for_config(config) as executor:
        futures = [
            executor.submit(self.invoke, input_, config_)
            for input_, config_ in zip(inputs, configs)
        ]
        return [f.result() for f in futures]
```

### Batch vs Sequential

```python
# Sequential: N * latency
results = [chain.invoke(x) for x in inputs]  # Slow

# Batched: max(latencies) with parallelism
results = chain.batch(inputs)  # Fast
```

**Performance difference**: 5-10x for typical LLM calls

### Optimal Batch Sizes

```python
# Control concurrency
results = chain.batch(
    inputs,
    config={"max_concurrency": 5}  # Limit parallel calls
)
```

**Guidelines**:
- OpenAI: max_concurrency = 5-10 (rate limits)
- Local models: max_concurrency = CPU cores
- Vector DBs: max_concurrency = connection pool size

### RunnableSequence Batch Optimization

```python
# From: libs/core/langchain_core/runnables/base.py:2989-3050
def batch(self, inputs, config=None, **kwargs):
    # Batch through first step
    outputs = self.first.batch(inputs, configs)

    # Batch through middle steps
    for step in self.middle:
        outputs = step.batch(outputs, configs)

    # Batch through last step
    return self.last.batch(outputs, configs)
```

Each step batches independently, preserving parallelism.

## Concurrency Control

### Thread Pool Executor

```python
# From: libs/core/langchain_core/runnables/config.py:503-524
class ContextThreadPoolExecutor(ThreadPoolExecutor):
    """Preserves context variables across threads."""

    def submit(self, fn, *args, **kwargs):
        ctx = copy_context()
        return super().submit(ctx.run, fn, *args, **kwargs)
```

### Async Concurrency

```python
# From: libs/core/langchain_core/runnables/base.py
async def abatch(self, inputs, config=None, **kwargs):
    # Uses asyncio.gather with semaphore
    max_concurrency = config.get("max_concurrency")

    async def process(input_, config_):
        async with semaphore:
            return await self.ainvoke(input_, config_)

    return await asyncio.gather(*[
        process(i, c) for i, c in zip(inputs, configs)
    ])
```

### Best Practices

```python
# ✅ Good: Use async for I/O-bound work
async def process_all(inputs):
    return await chain.abatch(inputs, config={"max_concurrency": 10})

# ❌ Bad: Block async with sync calls
async def process_all(inputs):
    return chain.batch(inputs)  # Blocks event loop
```

## Caching Strategies

### LLM Response Caching

```python
from langchain_core.caches import InMemoryCache
from langchain_core.globals import set_llm_cache

# Set global cache
set_llm_cache(InMemoryCache())

# Now duplicate calls are cached
model = ChatOpenAI()
result1 = model.invoke("Hello")  # API call
result2 = model.invoke("Hello")  # Cached!
```

### Custom Cache Implementation

```python
from langchain_core.caches import BaseCache
import redis

class RedisCache(BaseCache):
    def __init__(self, redis_url: str):
        self.client = redis.from_url(redis_url)

    def lookup(self, prompt: str, llm_string: str) -> list | None:
        key = self._key(prompt, llm_string)
        cached = self.client.get(key)
        return json.loads(cached) if cached else None

    def update(self, prompt: str, llm_string: str, return_val: list) -> None:
        key = self._key(prompt, llm_string)
        self.client.setex(key, 3600, json.dumps(return_val))  # 1hr TTL

    def _key(self, prompt: str, llm_string: str) -> str:
        return f"llm:{hashlib.md5((prompt + llm_string).encode()).hexdigest()}"
```

### Embedding Caching

```python
from langchain_core.embeddings import CacheBackedEmbeddings
from langchain_community.storage import LocalFileStore

# Cache embeddings to disk
store = LocalFileStore("./embeddings_cache")
cached_embeddings = CacheBackedEmbeddings.from_bytes_store(
    underlying_embeddings=OpenAIEmbeddings(),
    document_embedding_cache=store,
    namespace="my_embeddings"
)
```

### When to Cache

| Scenario | Cache? | Why |
|----------|--------|-----|
| Same prompts repeated | ✅ Yes | Exact matches |
| User queries | ❌ Usually no | Unique queries |
| Embeddings | ✅ Yes | Expensive & deterministic |
| RAG context | ⚠️ Maybe | If context is static |

## Streaming Optimization

### Why Stream?

- **Time to first token**: User sees response faster
- **Memory efficiency**: Don't buffer entire response
- **User experience**: Progressive feedback

### Basic Streaming

```python
for chunk in chain.stream(input):
    print(chunk, end="", flush=True)
```

### Streaming with Events

```python
async for event in chain.astream_events(input, version="v2"):
    kind = event["event"]

    if kind == "on_chat_model_stream":
        content = event["data"]["chunk"].content
        print(content, end="", flush=True)

    elif kind == "on_tool_end":
        print(f"\nTool result: {event['data']['output']}")
```

### Streaming Pitfalls

```python
# ❌ Bad: Blocks streaming
chain = prompt | model | (lambda x: x.content.upper())
# Lambda processes entire output at once

# ✅ Good: Stream-aware transformation
from langchain_core.output_parsers import StrOutputParser

chain = prompt | model | StrOutputParser()
# StrOutputParser streams chunks through
```

### Custom Streaming Runnable

```python
from langchain_core.runnables import Runnable
from typing import Iterator

class StreamingTransformer(Runnable[str, str]):
    def stream(self, input: str, config=None, **kwargs) -> Iterator[str]:
        for chunk in input:  # Process chunk by chunk
            yield chunk.upper()

    def invoke(self, input: str, config=None, **kwargs) -> str:
        return "".join(self.stream(input, config, **kwargs))
```

## Memory Optimization

### Avoid Large Intermediate Results

```python
# ❌ Bad: Loads all documents into memory
docs = loader.load()  # Could be GBs
texts = [doc.page_content for doc in docs]

# ✅ Good: Stream documents
for doc in loader.lazy_load():
    process(doc)
```

### Generator-Based Processing

```python
from langchain_core.runnables import RunnableGenerator

def process_chunks(input):
    for chunk in input:
        yield transform(chunk)

streaming_chain = prompt | model | RunnableGenerator(process_chunks)
```

### Limit Context Size

```python
# Truncate context to avoid OOM
def limit_context(docs: list[Document], max_chars: int = 10000) -> str:
    context = ""
    for doc in docs:
        if len(context) + len(doc.page_content) > max_chars:
            break
        context += doc.page_content + "\n"
    return context
```

## Scaling Considerations

### Horizontal Scaling

```python
# Worker process
async def worker(queue):
    while True:
        input = await queue.get()
        result = await chain.ainvoke(input)
        await send_result(result)

# Scale workers based on load
workers = [worker(queue) for _ in range(num_workers)]
await asyncio.gather(*workers)
```

### Connection Pooling

```python
import httpx

# Reuse HTTP client
client = httpx.AsyncClient(
    limits=httpx.Limits(max_connections=100)
)

model = ChatOpenAI(http_async_client=client)
```

### Rate Limiting

```python
from langchain_core.rate_limiters import InMemoryRateLimiter

rate_limiter = InMemoryRateLimiter(
    requests_per_second=10,
    check_every_n_seconds=0.1
)

model = ChatOpenAI(rate_limiter=rate_limiter)
```

## Benchmarking

### Simple Timing

```python
import time

start = time.perf_counter()
result = chain.invoke(input)
elapsed = time.perf_counter() - start
print(f"Elapsed: {elapsed:.3f}s")
```

### Profiling

```python
import cProfile
import pstats

profiler = cProfile.Profile()
profiler.enable()

result = chain.invoke(input)

profiler.disable()
stats = pstats.Stats(profiler)
stats.sort_stats('cumulative')
stats.print_stats(20)  # Top 20 functions
```

### Load Testing

```python
import asyncio
import time

async def load_test(chain, inputs, concurrency=10):
    semaphore = asyncio.Semaphore(concurrency)

    async def process(input):
        async with semaphore:
            start = time.perf_counter()
            await chain.ainvoke(input)
            return time.perf_counter() - start

    times = await asyncio.gather(*[process(i) for i in inputs])

    print(f"Total: {sum(times):.2f}s")
    print(f"Avg: {sum(times)/len(times):.3f}s")
    print(f"P99: {sorted(times)[int(len(times)*0.99)]:.3f}s")
```

## Optimization Checklist

### Quick Wins
- [ ] Use `batch()` instead of loops
- [ ] Set appropriate `max_concurrency`
- [ ] Enable LLM caching for repeated prompts
- [ ] Use async for I/O-bound operations
- [ ] Stream responses to users

### Medium Effort
- [ ] Cache embeddings
- [ ] Use connection pooling
- [ ] Implement rate limiting
- [ ] Profile and identify hotspots
- [ ] Optimize context window usage

### Architectural
- [ ] Design for horizontal scaling
- [ ] Use message queues for async processing
- [ ] Implement circuit breakers
- [ ] Monitor and alert on latency

## Performance Anti-Patterns

### Anti-Pattern 1: Sequential API Calls

```python
# ❌ Bad: 10 * latency
results = []
for input in inputs:
    results.append(model.invoke(input))

# ✅ Good: max(latencies)
results = model.batch(inputs)
```

### Anti-Pattern 2: Unbounded Concurrency

```python
# ❌ Bad: May hit rate limits or OOM
results = await model.abatch(huge_list)

# ✅ Good: Controlled concurrency
results = await model.abatch(
    huge_list,
    config={"max_concurrency": 10}
)
```

### Anti-Pattern 3: No Caching

```python
# ❌ Bad: Re-embeds same documents
for query in queries:
    docs = vectorstore.similarity_search(query)

# ✅ Good: Cache embeddings
cached_embeddings = CacheBackedEmbeddings(...)
vectorstore = Chroma(embedding_function=cached_embeddings)
```

### Anti-Pattern 4: Blocking in Async

```python
# ❌ Bad: Blocks event loop
async def handler(request):
    return chain.invoke(request.data)  # Sync call!

# ✅ Good: Use async
async def handler(request):
    return await chain.ainvoke(request.data)
```

## Key Takeaways

1. **LLM latency dominates**: LangChain overhead is typically <10% of total time.

2. **Batch everything**: Use `batch()` for parallel processing.

3. **Control concurrency**: Set `max_concurrency` to avoid rate limits.

4. **Cache strategically**: LLM responses and embeddings benefit most.

5. **Stream for UX**: Users prefer progressive responses.

6. **Profile before optimizing**: Find actual bottlenecks first.

## What's Next

In [Part 6](./06-observability-tracing.md), we'll explore LangChain's observability system—callbacks, tracing, and monitoring for production applications.

---

**Code references**: All examples reference [`990e346`](https://github.com/langchain-ai/langchain/tree/990e346c46251246966b562c8d6839903cb205ad).
