# Deep Dive: The Runnable Protocol

> **Part 2 of 6** | Analysis based on commit [`990e346`](https://github.com/langchain-ai/langchain/tree/990e346c46251246966b562c8d6839903cb205ad)

## What You'll Learn

- How the Runnable protocol works internally
- LCEL (LangChain Expression Language) composition
- RunnableSequence, RunnableParallel, and RunnableLambda
- Configuration and state propagation
- Type system and generics

## Introduction

The Runnable protocol is the heart of LangChain. Every component—prompts, models, parsers, tools—implements this protocol. Understanding it unlocks the full power of the framework.

## The Runnable Base Class

Let's start with the definition:

```python
# From: libs/core/langchain_core/runnables/base.py:124
class Runnable(ABC, Generic[Input, Output]):
    """A unit of work that can be invoked, batched, streamed, and transformed."""
```

A Runnable takes an `Input` type and produces an `Output` type. This generic typing is crucial for chain composition.

### Core Methods

```python
# Synchronous execution
def invoke(self, input: Input, config: RunnableConfig | None = None) -> Output:
    ...

# Asynchronous execution
async def ainvoke(self, input: Input, config: RunnableConfig | None = None) -> Output:
    ...

# Batch processing
def batch(
    self,
    inputs: list[Input],
    config: RunnableConfig | list[RunnableConfig] | None = None,
    *,
    return_exceptions: bool = False,
    **kwargs
) -> list[Output]:
    ...

# Streaming output
def stream(
    self,
    input: Input,
    config: RunnableConfig | None = None,
    **kwargs
) -> Iterator[Output]:
    ...
```

📌 **Key Insight**: Every Runnable automatically gets async, batch, and streaming support. You implement `invoke`, and the framework provides the rest.

## LCEL: The Pipe Operator

LCEL is how we compose Runnables using the `|` operator:

```python
chain = prompt | model | parser
```

This is implemented via `__or__` and `__ror__`:

```python
# From: libs/core/langchain_core/runnables/base.py:616-656
def __or__(
    self,
    other: Runnable[Any, Other] | Callable[[Any], Other] | ...
) -> RunnableSerializable[Input, Other]:
    """Compose with another Runnable: self | other"""
    return RunnableSequence(self, coerce_to_runnable(other))

def __ror__(
    self,
    other: Runnable[Other, Any] | Callable[[Other], Any] | ...
) -> RunnableSerializable[Other, Output]:
    """Compose with another Runnable: other | self"""
    return RunnableSequence(coerce_to_runnable(other), self)
```

### Automatic Coercion

The magic of LCEL is `coerce_to_runnable()`:

```python
# From: libs/core/langchain_core/runnables/base.py:6015-6039
def coerce_to_runnable(thing: RunnableLike) -> Runnable[Input, Output]:
    """Convert various types to Runnable."""
    if isinstance(thing, Runnable):
        return thing
    if is_async_generator(thing) or inspect.isgeneratorfunction(thing):
        return RunnableGenerator(thing)
    if callable(thing):
        return RunnableLambda(cast("Callable[[Input], Output]", thing))
    if isinstance(thing, dict):
        return cast("Runnable[Input, Output]", RunnableParallel(thing))
    raise TypeError(f"Cannot coerce {thing} to Runnable")
```

This means:
- **Functions** become `RunnableLambda`
- **Dicts** become `RunnableParallel`
- **Generators** become `RunnableGenerator`

```python
# All of these work:
chain = prompt | model | (lambda x: x.content)  # Function → RunnableLambda
chain = prompt | {"a": model_a, "b": model_b}   # Dict → RunnableParallel
```

## RunnableSequence: Sequential Composition

When you use `|`, you create a `RunnableSequence`:

```python
# From: libs/core/langchain_core/runnables/base.py:2789
class RunnableSequence(RunnableSerializable[Input, Output]):
    first: Runnable[Input, Any]
    middle: list[Runnable[Any, Any]] = Field(default_factory=list)
    last: Runnable[Any, Output]
```

### Why first/middle/last?

This structure preserves type information:
- `first` has `Input` type
- `last` has `Output` type
- `middle` is `Any → Any`

### Invocation Flow

```python
# Simplified from libs/core/langchain_core/runnables/base.py:2900-2950
def invoke(self, input: Input, config: RunnableConfig | None = None) -> Output:
    # Setup
    config = ensure_config(config)
    callback_manager = get_callback_manager_for_config(config)

    # Start tracking
    run_manager = callback_manager.on_chain_start(
        dumpd(self), input, name=self.get_name()
    )

    try:
        # Execute sequence
        input_ = input
        for i, step in enumerate(self.steps):
            # Patch config with step-specific callbacks
            config = patch_config(
                config,
                callbacks=run_manager.get_child(f"seq:step:{i+1}")
            )
            # Invoke step
            input_ = step.invoke(input_, config)

        # Success
        run_manager.on_chain_end(input_)
        return input_
    except Exception as e:
        run_manager.on_chain_error(e)
        raise
```

### Example: Data Flow

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser

prompt = ChatPromptTemplate.from_template("Explain {topic}")
model = ChatOpenAI()
parser = StrOutputParser()

chain = prompt | model | parser

# Data flow:
# {"topic": "AI"} → prompt → "Explain AI" → model → AIMessage(...) → parser → "AI is..."
result = chain.invoke({"topic": "AI"})
```

## RunnableParallel: Concurrent Composition

`RunnableParallel` runs multiple Runnables with the same input:

```python
# From: libs/core/langchain_core/runnables/base.py:3537
class RunnableParallel(RunnableSerializable[Input, dict[str, Any]]):
    steps__: Mapping[str, Runnable[Input, Any]]
```

### Usage

```python
# Using dict literal (auto-coerced)
parallel = prompt | {
    "summary": summary_chain,
    "keywords": keyword_chain,
}

# Explicit construction
from langchain_core.runnables import RunnableParallel
parallel = RunnableParallel(
    summary=summary_chain,
    keywords=keyword_chain,
)

result = parallel.invoke({"text": "..."})
# {"summary": "...", "keywords": [...]}
```

### Async Execution

```python
# From: libs/core/langchain_core/runnables/base.py:3765-3850
async def ainvoke(self, input: Input, config: RunnableConfig | None = None) -> dict[str, Any]:
    # Run all branches concurrently
    coros = {
        key: self._arun_one(key, input, config)
        for key in self.steps__
    }

    # Gather with concurrency control
    results = await gather_with_concurrency(
        config.get("max_concurrency") if config else None,
        *coros.values(),
    )

    return dict(zip(self.steps__.keys(), results))
```

📌 **Key Insight**: `RunnableParallel` respects `max_concurrency` in config, preventing resource exhaustion.

## RunnableLambda: Function Wrapping

`RunnableLambda` converts any function to a Runnable:

```python
# From: libs/core/langchain_core/runnables/base.py:4370
class RunnableLambda(Runnable[Input, Output]):
    func: Callable[[Input], Output] | None
    afunc: Callable[[Input], Awaitable[Output]] | None
```

### Supports Multiple Function Types

```python
from langchain_core.runnables import RunnableLambda

# Sync function
sync_run = RunnableLambda(lambda x: x.upper())

# Async function
async def async_fn(x):
    await asyncio.sleep(0.1)
    return x.upper()
async_run = RunnableLambda(async_fn)

# Generator (for streaming)
def gen_fn(x):
    for char in x:
        yield char
stream_run = RunnableLambda(gen_fn)
```

### Type Inference

`RunnableLambda` inspects function signatures:

```python
# From: libs/core/langchain_core/runnables/base.py:4520-4537
@property
def InputType(self) -> Any:
    func = getattr(self, "func", None) or self.afunc
    try:
        params = inspect.signature(func).parameters
        first_param = next(iter(params.values()), None)
        if first_param and first_param.annotation != inspect.Parameter.empty:
            return first_param.annotation
    except ValueError:
        pass
    return Any
```

## RunnablePassthrough and RunnableAssign

### RunnablePassthrough: Identity + Side Effects

```python
# From: libs/core/langchain_core/runnables/passthrough.py:74
from langchain_core.runnables import RunnablePassthrough

# Simply pass input through
passthrough = RunnablePassthrough()
passthrough.invoke({"key": "value"})  # {"key": "value"}

# With side effect
passthrough = RunnablePassthrough(
    func=lambda x: print(f"Processing: {x}")
)
```

### RunnableAssign: Add Keys to Dict

```python
# From: libs/core/langchain_core/runnables/passthrough.py:352
from langchain_core.runnables import RunnablePassthrough

chain = RunnablePassthrough.assign(
    length=lambda x: len(x["text"]),
    words=lambda x: x["text"].split()
)

result = chain.invoke({"text": "hello world"})
# {"text": "hello world", "length": 11, "words": ["hello", "world"]}
```

This is incredibly useful for building up context:

```python
chain = (
    RunnablePassthrough.assign(
        context=retriever
    )
    | prompt
    | model
)
```

## Configuration Propagation

`RunnableConfig` flows through the entire chain:

```python
# From: libs/core/langchain_core/runnables/config.py:49-99
class RunnableConfig(TypedDict, total=False):
    tags: list[str]              # Tracing tags
    metadata: dict[str, Any]     # Custom metadata
    callbacks: Callbacks          # Event handlers
    run_name: str                # Custom run name
    max_concurrency: int | None  # Thread pool size
    recursion_limit: int         # Max depth (default: 25)
    configurable: dict[str, Any] # Runtime config
    run_id: uuid.UUID | None     # Unique ID
```

### Config Merging

```python
# From: libs/core/langchain_core/runnables/config.py:333-397
def merge_configs(*configs: RunnableConfig | None) -> RunnableConfig:
    base: RunnableConfig = {}
    for config in (ensure_config(c) for c in configs if c is not None):
        for key in config:
            if key == "metadata":
                # Merge dicts
                base["metadata"] = {**base.get("metadata", {}), **config["metadata"]}
            elif key == "tags":
                # Deduplicate
                base["tags"] = sorted(set(base.get("tags", []) + config["tags"]))
            # ... more merging logic
```

### Context Variables

Config can be set at context level:

```python
# From: libs/core/langchain_core/runnables/config.py:122-190
from contextvars import ContextVar

var_child_runnable_config: ContextVar[RunnableConfig | None] = ContextVar(
    "child_runnable_config", default=None
)
```

## Binding and Decorating

### bind(): Preset Arguments

```python
model = ChatOpenAI()
bound = model.bind(temperature=0.5, max_tokens=100)

# Every invocation uses these settings
result = bound.invoke(messages)
```

### with_config(): Preset Config

```python
chain = prompt | model | parser
configured = chain.with_config(
    tags=["production"],
    metadata={"version": "1.0"}
)
```

### with_retry(): Add Retry Logic

```python
# From: libs/core/langchain_core/runnables/retry.py
model_with_retry = model.with_retry(
    stop_after_attempt=3,
    wait_exponential_jitter=True
)
```

### with_fallbacks(): Add Fallbacks

```python
# From: libs/core/langchain_core/runnables/fallbacks.py
model_with_fallback = ChatOpenAI().with_fallbacks([
    ChatAnthropic(),
    ChatOllama(),
])
```

## Streaming and Events

### Basic Streaming

```python
for chunk in chain.stream(input):
    print(chunk, end="", flush=True)
```

### Event Streaming

```python
async for event in chain.astream_events(input, version="v2"):
    if event["event"] == "on_chat_model_stream":
        print(event["data"]["chunk"].content, end="")
    elif event["event"] == "on_chain_end":
        print("\nDone!")
```

Events include:
- `on_chain_start`, `on_chain_end`
- `on_chat_model_start`, `on_chat_model_stream`, `on_chat_model_end`
- `on_tool_start`, `on_tool_end`
- And more...

## Advanced: Configurable Runnables

Make Runnables configurable at runtime:

```python
# From: libs/core/langchain_core/runnables/base.py:2592-2702
from langchain_core.runnables import ConfigurableField

model = ChatOpenAI(max_tokens=20).configurable_fields(
    max_tokens=ConfigurableField(
        id="output_tokens",
        name="Max Tokens",
        description="Maximum output tokens"
    )
)

# Override at runtime
result = model.with_config(
    configurable={"output_tokens": 200}
).invoke("Hello")
```

### Alternative Strategies

```python
model = ChatOpenAI().configurable_alternatives(
    ConfigurableField(id="llm"),
    default_key="openai",
    anthropic=ChatAnthropic(),
    ollama=ChatOllama(),
)

# Switch at runtime
result = model.with_config(
    configurable={"llm": "anthropic"}
).invoke("Hello")
```

## Type System

### Input/Output Schema

Runnables can generate Pydantic schemas:

```python
# From: libs/core/langchain_core/runnables/base.py:366-486
chain = prompt | model | parser

# Get schemas
input_schema = chain.get_input_schema()
output_schema = chain.get_output_schema()

# Use for validation
validated = input_schema(**user_input)
```

### Schema Composition

For `RunnableSequence`:
- `InputType` = first runnable's `InputType`
- `OutputType` = last runnable's `OutputType`

For `RunnableParallel`:
- `InputType` = merged input requirements
- `OutputType` = `dict[str, Any]`

## Performance Considerations

### Batching Optimization

```python
# From: libs/core/langchain_core/runnables/base.py:863-920
def batch(self, inputs, config=None, *, return_exceptions=False):
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

### Concurrency Control

```python
# Limit parallel executions
result = chain.invoke(
    input,
    config={"max_concurrency": 5}
)
```

### Default Executor

Uses `ContextThreadPoolExecutor` to preserve context variables across threads:

```python
# From: libs/core/langchain_core/runnables/config.py:503-524
class ContextThreadPoolExecutor(ThreadPoolExecutor):
    def submit(self, fn, *args, **kwargs):
        ctx = copy_context()
        return super().submit(ctx.run, fn, *args, **kwargs)
```

## Example: Complete Chain

Let's build a real RAG chain:

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough, RunnableParallel
from langchain_core.output_parsers import StrOutputParser
from langchain_openai import ChatOpenAI
from operator import itemgetter

# Components
retriever = vectorstore.as_retriever()
prompt = ChatPromptTemplate.from_template("""
Answer the question based on the context:

Context: {context}

Question: {question}

Answer:""")
model = ChatOpenAI()
parser = StrOutputParser()

# Build chain
chain = (
    RunnableParallel(
        context=itemgetter("question") | retriever,
        question=itemgetter("question")
    )
    | prompt
    | model
    | parser
)

# Use
result = chain.invoke({"question": "What is LangChain?"})

# With streaming
for chunk in chain.stream({"question": "What is LangChain?"}):
    print(chunk, end="", flush=True)

# With tracing
result = chain.invoke(
    {"question": "What is LangChain?"},
    config={
        "tags": ["rag", "production"],
        "metadata": {"user": "123"}
    }
)
```

## Key Takeaways

1. **`|` creates RunnableSequence**: Data flows left to right.

2. **Dicts become RunnableParallel**: Same input, multiple branches.

3. **Functions become RunnableLambda**: Auto-coercion via `coerce_to_runnable()`.

4. **Config propagates automatically**: Tags, callbacks, and metadata flow through.

5. **Type safety is preserved**: Generic types enable IDE support and validation.

6. **Every Runnable is fully featured**: `invoke`, `ainvoke`, `batch`, `stream` all work.

## What's Next

In [Part 3](./03-patterns-practices.md), we'll examine the design patterns LangChain uses—Chain of Responsibility, Decorator, Strategy—and how they enable the framework's flexibility and maintainability.

---

**Code references**: All examples reference [`990e346`](https://github.com/langchain-ai/langchain/tree/990e346c46251246966b562c8d6839903cb205ad).
