# Observability and Tracing in LangChain

> **Part 6 of 6** | Analysis based on commit [`990e346`](https://github.com/langchain-ai/langchain/tree/990e346c46251246966b562c8d6839903cb205ad)

## What You'll Learn

- Callback system architecture
- LangSmith integration
- Custom tracing implementations
- Usage tracking and cost monitoring
- Error monitoring and debugging

## Introduction

In production LLM applications, observability is critical. You need to understand what's happening inside your chains, track costs, debug failures, and monitor performance. LangChain's callback system makes this possible.

## Callback System Architecture

### The Handler Hierarchy

```python
# From: libs/core/langchain_core/callbacks/base.py
class BaseCallbackHandler:
    """Base class for callback handlers."""

    # LLM events
    def on_llm_start(self, serialized, prompts, **kwargs): ...
    def on_llm_new_token(self, token, **kwargs): ...
    def on_llm_end(self, response, **kwargs): ...
    def on_llm_error(self, error, **kwargs): ...

    # Chain events
    def on_chain_start(self, serialized, inputs, **kwargs): ...
    def on_chain_end(self, outputs, **kwargs): ...
    def on_chain_error(self, error, **kwargs): ...

    # Tool events
    def on_tool_start(self, serialized, input_str, **kwargs): ...
    def on_tool_end(self, output, **kwargs): ...
    def on_tool_error(self, error, **kwargs): ...

    # Retriever events
    def on_retriever_start(self, serialized, query, **kwargs): ...
    def on_retriever_end(self, documents, **kwargs): ...
    def on_retriever_error(self, error, **kwargs): ...
```

### Mixin-Based Design

LangChain uses mixins for modular handler capabilities:

```python
# From: libs/core/langchain_core/callbacks/base.py
class LLMManagerMixin:
    def on_llm_new_token(self, token, **kwargs): ...

class ChainManagerMixin:
    def on_chain_end(self, outputs, **kwargs): ...

class ToolManagerMixin:
    def on_tool_end(self, output, **kwargs): ...

class RunManagerMixin(
    LLMManagerMixin,
    ChainManagerMixin,
    ToolManagerMixin
):
    """Combined capabilities."""
```

### Callback Manager

The manager coordinates callbacks and propagates them:

```python
# From: libs/core/langchain_core/callbacks/manager.py
class CallbackManager:
    handlers: list[BaseCallbackHandler]
    inheritable_handlers: list[BaseCallbackHandler]
    parent_run_id: UUID | None
    tags: list[str]
    metadata: dict[str, Any]

    def on_chain_start(
        self,
        serialized: dict,
        inputs: dict,
        **kwargs
    ) -> CallbackManagerForChainRun:
        # Call each handler
        for handler in self.handlers:
            handler.on_chain_start(serialized, inputs, **kwargs)

        # Return run manager for this execution
        return CallbackManagerForChainRun(
            handlers=self.handlers,
            parent_run_id=self.run_id,
            ...
        )
```

### Run Manager Types

12 specialized run managers for different contexts:

```python
# From: libs/core/langchain_core/callbacks/manager.py
CallbackManagerForLLMRun        # LLM execution
CallbackManagerForChainRun      # Chain execution
CallbackManagerForToolRun       # Tool execution
CallbackManagerForRetrieverRun  # Retriever execution
AsyncCallbackManagerFor...      # Async versions
```

## Using Callbacks

### Simple Logging Handler

```python
from langchain_core.callbacks import BaseCallbackHandler
from langchain_openai import ChatOpenAI

class LoggingHandler(BaseCallbackHandler):
    def on_llm_start(self, serialized, prompts, **kwargs):
        print(f"LLM Start: {prompts}")

    def on_llm_end(self, response, **kwargs):
        print(f"LLM End: {response.generations[0][0].text[:50]}...")

    def on_chain_start(self, serialized, inputs, **kwargs):
        print(f"Chain Start: {list(inputs.keys())}")

    def on_chain_end(self, outputs, **kwargs):
        print(f"Chain End: {list(outputs.keys())}")

# Use the handler
model = ChatOpenAI()
result = model.invoke(
    "Hello",
    config={"callbacks": [LoggingHandler()]}
)
```

### Streaming Token Handler

```python
class StreamingHandler(BaseCallbackHandler):
    def on_llm_new_token(self, token: str, **kwargs):
        print(token, end="", flush=True)

# Stream tokens
model = ChatOpenAI(streaming=True)
result = model.invoke(
    "Write a poem",
    config={"callbacks": [StreamingHandler()]}
)
```

### Cost Tracking Handler

```python
from langchain_core.callbacks import BaseCallbackHandler

class CostTracker(BaseCallbackHandler):
    total_tokens: int = 0
    prompt_tokens: int = 0
    completion_tokens: int = 0

    # Cost per 1K tokens (GPT-4)
    INPUT_COST = 0.03
    OUTPUT_COST = 0.06

    def on_llm_end(self, response, **kwargs):
        usage = response.llm_output.get("token_usage", {})
        self.prompt_tokens += usage.get("prompt_tokens", 0)
        self.completion_tokens += usage.get("completion_tokens", 0)
        self.total_tokens += usage.get("total_tokens", 0)

    @property
    def total_cost(self):
        return (
            (self.prompt_tokens / 1000) * self.INPUT_COST +
            (self.completion_tokens / 1000) * self.OUTPUT_COST
        )

# Usage
tracker = CostTracker()
result = model.invoke(input, config={"callbacks": [tracker]})
print(f"Total cost: ${tracker.total_cost:.4f}")
```

## LangSmith Integration

### Automatic Tracing

Enable with environment variables:

```bash
export LANGCHAIN_TRACING_V2=true
export LANGCHAIN_API_KEY=your-api-key
export LANGCHAIN_PROJECT=my-project
```

All chains are automatically traced to LangSmith.

### Manual Tracing

```python
from langsmith import traceable

@traceable
def my_function(input: str) -> str:
    # Your logic
    result = chain.invoke(input)
    return result

# Traced with full visibility
output = my_function("Hello")
```

### Run Metadata

```python
result = chain.invoke(
    input,
    config={
        "run_name": "user-query",
        "tags": ["production", "v2"],
        "metadata": {
            "user_id": "123",
            "session_id": "abc",
        }
    }
)
```

### The Run Schema

```python
# From: libs/core/langchain_core/tracers/core.py
class Run:
    id: UUID
    name: str
    start_time: datetime
    end_time: datetime | None
    inputs: dict
    outputs: dict | None
    error: str | None
    parent_run_id: UUID | None
    child_runs: list[Run]
    tags: list[str]
    metadata: dict[str, Any]
    # ... 20+ fields
```

## Custom Tracer Implementation

### Basic Tracer

```python
from langchain_core.tracers.base import BaseTracer
from langchain_core.tracers.schemas import Run

class MyTracer(BaseTracer):
    """Custom tracer that sends to my backend."""

    def __init__(self, api_endpoint: str):
        super().__init__()
        self.api_endpoint = api_endpoint

    def _persist_run(self, run: Run) -> None:
        """Called when a run completes."""
        import requests
        requests.post(
            self.api_endpoint,
            json={
                "run_id": str(run.id),
                "name": run.name,
                "start_time": run.start_time.isoformat(),
                "end_time": run.end_time.isoformat() if run.end_time else None,
                "inputs": run.inputs,
                "outputs": run.outputs,
                "error": run.error,
                "latency_ms": (run.end_time - run.start_time).total_seconds() * 1000
            }
        )

# Usage
tracer = MyTracer(api_endpoint="https://my-backend/traces")
result = chain.invoke(input, config={"callbacks": [tracer]})
```

### Tracer with Metrics

```python
from langchain_core.tracers.base import BaseTracer
import prometheus_client

class PrometheusTracer(BaseTracer):
    latency_histogram = prometheus_client.Histogram(
        'langchain_latency_seconds',
        'Latency of LangChain operations',
        ['operation_type']
    )
    error_counter = prometheus_client.Counter(
        'langchain_errors_total',
        'Total errors in LangChain',
        ['operation_type']
    )

    def _persist_run(self, run: Run) -> None:
        op_type = run.run_type  # "llm", "chain", "tool"
        latency = (run.end_time - run.start_time).total_seconds()

        self.latency_histogram.labels(operation_type=op_type).observe(latency)

        if run.error:
            self.error_counter.labels(operation_type=op_type).inc()
```

## Usage Tracking

### Token Usage

```python
# From: libs/core/langchain_core/callbacks/usage.py
def get_openai_token_cost_for_model(
    model_name: str,
    num_tokens: int,
    is_completion: bool = False
) -> float:
    """Calculate cost based on model and token count."""
    # Model-specific pricing
    ...
```

### Usage Metadata in Messages

```python
result = model.invoke(messages)

# Access usage info
if hasattr(result, 'usage_metadata'):
    usage = result.usage_metadata
    print(f"Input tokens: {usage.input_tokens}")
    print(f"Output tokens: {usage.output_tokens}")
    print(f"Total tokens: {usage.total_tokens}")
```

### Aggregate Usage Across Chains

```python
class UsageAggregator(BaseCallbackHandler):
    def __init__(self):
        self.usage_by_model = {}

    def on_llm_end(self, response, **kwargs):
        model = kwargs.get("invocation_params", {}).get("model_name", "unknown")
        usage = response.llm_output.get("token_usage", {})

        if model not in self.usage_by_model:
            self.usage_by_model[model] = {"prompt": 0, "completion": 0}

        self.usage_by_model[model]["prompt"] += usage.get("prompt_tokens", 0)
        self.usage_by_model[model]["completion"] += usage.get("completion_tokens", 0)

    def report(self):
        for model, usage in self.usage_by_model.items():
            print(f"{model}: {usage['prompt']} prompt, {usage['completion']} completion")
```

## Error Monitoring

### Error Handler

```python
class ErrorHandler(BaseCallbackHandler):
    def __init__(self):
        self.errors = []

    def on_llm_error(self, error: BaseException, **kwargs):
        self.errors.append({
            "type": "llm",
            "error": str(error),
            "kwargs": kwargs
        })

    def on_chain_error(self, error: BaseException, **kwargs):
        self.errors.append({
            "type": "chain",
            "error": str(error),
            "kwargs": kwargs
        })

    def on_tool_error(self, error: BaseException, **kwargs):
        self.errors.append({
            "type": "tool",
            "error": str(error),
            "kwargs": kwargs
        })
```

### Error with Context

```python
class ContextualErrorHandler(BaseCallbackHandler):
    def __init__(self):
        self.current_chain_inputs = {}

    def on_chain_start(self, serialized, inputs, **kwargs):
        run_id = kwargs.get("run_id")
        self.current_chain_inputs[run_id] = inputs

    def on_chain_error(self, error, **kwargs):
        run_id = kwargs.get("run_id")
        inputs = self.current_chain_inputs.get(run_id, {})

        # Log with full context
        logger.error(
            f"Chain failed: {error}",
            extra={
                "run_id": str(run_id),
                "inputs": inputs,
                "error_type": type(error).__name__
            }
        )
```

### Retry Event Tracking

```python
class RetryTracker(BaseCallbackHandler):
    def on_retry(self, retry_state, **kwargs):
        print(f"Retry attempt {retry_state.attempt_number}")
        print(f"Exception: {retry_state.outcome.exception()}")
```

## Advanced Patterns

### Nested Run Tracking

```python
class HierarchyTracker(BaseCallbackHandler):
    def __init__(self):
        self.runs = {}

    def on_chain_start(self, serialized, inputs, **kwargs):
        run_id = kwargs.get("run_id")
        parent_id = kwargs.get("parent_run_id")

        self.runs[run_id] = {
            "name": kwargs.get("name"),
            "parent": parent_id,
            "children": []
        }

        if parent_id and parent_id in self.runs:
            self.runs[parent_id]["children"].append(run_id)

    def print_tree(self, run_id=None, indent=0):
        for rid, run in self.runs.items():
            if run["parent"] == run_id:
                print("  " * indent + run["name"])
                self.print_tree(rid, indent + 1)
```

### Conditional Tracing

```python
class ConditionalTracer(BaseTracer):
    def __init__(self, sample_rate: float = 0.1):
        super().__init__()
        self.sample_rate = sample_rate

    def _persist_run(self, run: Run) -> None:
        import random
        if random.random() < self.sample_rate:
            # Only trace some percentage
            self._send_to_backend(run)
```

### Context Variable Integration

```python
from contextvars import ContextVar

user_context: ContextVar[dict] = ContextVar("user_context", default={})

class ContextAwareTracer(BaseTracer):
    def _persist_run(self, run: Run) -> None:
        ctx = user_context.get()

        # Add context to run metadata
        run.metadata.update(ctx)

        self._send_to_backend(run)

# Usage
user_context.set({"user_id": "123", "tenant": "acme"})
result = chain.invoke(input, config={"callbacks": [ContextAwareTracer()]})
```

## Environment Configuration

### Key Environment Variables

```bash
# LangSmith tracing
export LANGCHAIN_TRACING_V2=true
export LANGCHAIN_API_KEY=ls-...
export LANGCHAIN_PROJECT=my-project
export LANGCHAIN_ENDPOINT=https://api.smith.langchain.com

# Verbosity
export LANGCHAIN_VERBOSE=true

# Caching
export LANGCHAIN_CACHE=true
```

### Programmatic Configuration

```python
from langchain_core.globals import set_verbose, set_debug

set_verbose(True)  # Print run info
set_debug(True)    # Full debug output
```

## Debugging Techniques

### Verbose Mode

```python
from langchain_core.globals import set_verbose
set_verbose(True)

# Now see all chain inputs/outputs
result = chain.invoke(input)
```

### Debug Mode

```python
from langchain_core.globals import set_debug
set_debug(True)

# Full debug output including prompts
result = chain.invoke(input)
```

### Step-by-Step Inspection

```python
# Use astream_events for detailed inspection
async for event in chain.astream_events(input, version="v2"):
    print(f"Event: {event['event']}")
    print(f"Name: {event['name']}")
    if event['event'] == "on_chat_model_stream":
        print(f"Content: {event['data']['chunk'].content}")
    print("---")
```

## Production Checklist

### Observability Setup
- [ ] Enable LangSmith tracing
- [ ] Configure appropriate project names
- [ ] Set up error alerting
- [ ] Implement cost tracking

### Monitoring
- [ ] Track latency per operation type
- [ ] Monitor error rates
- [ ] Alert on cost anomalies
- [ ] Track token usage trends

### Debugging
- [ ] Use unique run names for filtering
- [ ] Add relevant metadata (user_id, session_id)
- [ ] Tag runs for categorization
- [ ] Log with full context on errors

## Key Takeaways

1. **Callbacks are extensible**: Easy to add custom handlers.

2. **LangSmith is comprehensive**: Full tracing with minimal setup.

3. **Track costs carefully**: Token usage adds up quickly.

4. **Context matters**: Always include relevant metadata.

5. **Hierarchical tracing**: Parent-child relationships help debugging.

6. **Use both sync and async**: Match your application's patterns.

## Series Conclusion

Over this 6-part series, we've explored LangChain from architecture to observability:

1. **Architecture**: Monorepo structure and core philosophy
2. **Runnables**: The universal protocol and LCEL
3. **Patterns**: Design patterns and best practices
4. **Extension**: Building integrations and tools
5. **Performance**: Optimization strategies
6. **Observability**: Tracing and monitoring

Armed with this knowledge, you can build, extend, optimize, and monitor production LLM applications with LangChain.

---

**Code references**: All examples reference [`990e346`](https://github.com/langchain-ai/langchain/tree/990e346c46251246966b562c8d6839903cb205ad).

**Questions or feedback?** Open an issue at [langchain-ai/langchain](https://github.com/langchain-ai/langchain/issues).
