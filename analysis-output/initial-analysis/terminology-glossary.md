# LangChain Terminology Glossary

> **Analysis based on commit:** [`990e346`](https://github.com/langchain-ai/langchain/tree/990e346c46251246966b562c8d6839903cb205ad)

## Core Concepts

### Runnable

**Definition:** The fundamental protocol for all executable components in LangChain. Any object that can process input and produce output.

**Key Methods:**
- `invoke()` - Synchronous execution
- `ainvoke()` - Asynchronous execution
- `batch()` - Process multiple inputs
- `stream()` - Yield output chunks
- `astream()` - Async streaming

**Example:**
```python
from langchain_core.runnables import RunnableLambda

runnable = RunnableLambda(lambda x: x.upper())
runnable.invoke("hello")  # "HELLO"
```

**Location:** [`libs/core/langchain_core/runnables/base.py:124`](https://github.com/langchain-ai/langchain/blob/990e346c46251246966b562c8d6839903cb205ad/libs/core/langchain_core/runnables/base.py#L124)

---

### LCEL (LangChain Expression Language)

**Definition:** The declarative syntax for composing Runnables using the `|` (pipe) operator.

**Purpose:** Enables building complex chains with automatic support for streaming, batching, and tracing.

**Example:**
```python
chain = prompt | model | parser  # Creates RunnableSequence
result = chain.invoke({"question": "What is AI?"})
```

**Why it matters:** Replaces verbose imperative code with declarative composition.

---

### RunnableConfig

**Definition:** A TypedDict containing execution configuration that propagates through chains.

**Key Fields:**
- `tags` - Tracing labels
- `metadata` - Custom data for tracing
- `callbacks` - Event handlers
- `max_concurrency` - Thread pool size
- `recursion_limit` - Max nesting depth (default: 25)
- `configurable` - Runtime configuration values

**Example:**
```python
result = chain.invoke(
    input,
    config={
        "tags": ["production"],
        "metadata": {"user_id": "123"},
        "max_concurrency": 5
    }
)
```

**Location:** [`libs/core/langchain_core/runnables/config.py:49`](https://github.com/langchain-ai/langchain/blob/990e346c46251246966b562c8d6839903cb205ad/libs/core/langchain_core/runnables/config.py#L49)

---

### Callback Handler

**Definition:** An object that receives events during chain execution for tracing, logging, or monitoring.

**Event Types:**
- `on_llm_start/end/error`
- `on_chain_start/end/error`
- `on_tool_start/end/error`
- `on_retriever_start/end/error`

**Example:**
```python
from langchain_core.callbacks import BaseCallbackHandler

class MyHandler(BaseCallbackHandler):
    def on_llm_start(self, serialized, prompts, **kwargs):
        print(f"LLM called with: {prompts}")
```

**Location:** [`libs/core/langchain_core/callbacks/base.py`](https://github.com/langchain-ai/langchain/blob/990e346c46251246966b562c8d6839903cb205ad/libs/core/langchain_core/callbacks/base.py)

---

## Runnable Types

### RunnableSequence

**Definition:** Executes Runnables in sequence, passing output of each to the next.

**Created by:** Using `|` operator or `Runnable.pipe()`

**Structure:**
```python
class RunnableSequence:
    first: Runnable[Input, Any]
    middle: list[Runnable[Any, Any]]
    last: Runnable[Any, Output]
```

**Location:** [`libs/core/langchain_core/runnables/base.py:2789`](https://github.com/langchain-ai/langchain/blob/990e346c46251246966b562c8d6839903cb205ad/libs/core/langchain_core/runnables/base.py#L2789)

---

### RunnableParallel

**Definition:** Executes multiple Runnables concurrently with the same input, returns dict of results.

**Created by:** Using dict literal in chain

**Example:**
```python
parallel = {
    "summary": summary_runnable,
    "keywords": keyword_runnable
}
# Both run with same input, output: {"summary": ..., "keywords": ...}
```

**Location:** [`libs/core/langchain_core/runnables/base.py:3537`](https://github.com/langchain-ai/langchain/blob/990e346c46251246966b562c8d6839903cb205ad/libs/core/langchain_core/runnables/base.py#L3537)

---

### RunnableLambda

**Definition:** Wraps a Python function to make it a Runnable.

**Supports:**
- Sync functions
- Async functions
- Generators
- Async generators

**Example:**
```python
from langchain_core.runnables import RunnableLambda

@RunnableLambda
def process(x):
    return x.upper()

# or
runnable = RunnableLambda(lambda x: x * 2)
```

**Location:** [`libs/core/langchain_core/runnables/base.py:4370`](https://github.com/langchain-ai/langchain/blob/990e346c46251246966b562c8d6839903cb205ad/libs/core/langchain_core/runnables/base.py#L4370)

---

### RunnablePassthrough

**Definition:** Passes input through unchanged, optionally with side effects.

**Common uses:**
- Preserve input for later steps
- Add fields to dict via `.assign()`

**Example:**
```python
from langchain_core.runnables import RunnablePassthrough

chain = RunnablePassthrough.assign(
    length=lambda x: len(x["text"])
)
# Input: {"text": "hello"} → Output: {"text": "hello", "length": 5}
```

**Location:** [`libs/core/langchain_core/runnables/passthrough.py:74`](https://github.com/langchain-ai/langchain/blob/990e346c46251246966b562c8d6839903cb205ad/libs/core/langchain_core/runnables/passthrough.py#L74)

---

### RunnableBinding

**Definition:** Wraps a Runnable with additional configuration (callbacks, kwargs, etc.).

**Created by:** Methods like `.with_config()`, `.bind()`, `.with_retry()`

**Example:**
```python
bound = model.bind(temperature=0.5)
# Equivalent to model.invoke(..., temperature=0.5)
```

**Location:** [`libs/core/langchain_core/runnables/base.py:5369`](https://github.com/langchain-ai/langchain/blob/990e346c46251246966b562c8d6839903cb205ad/libs/core/langchain_core/runnables/base.py#L5369)

---

## Message Types

### BaseMessage

**Definition:** Abstract base class for all chat messages.

**Subclasses:**
- `HumanMessage` - User input
- `AIMessage` - Model response
- `SystemMessage` - System instructions
- `ToolMessage` - Tool execution result
- `FunctionMessage` - Legacy function result

**Location:** [`libs/core/langchain_core/messages/base.py`](https://github.com/langchain-ai/langchain/blob/990e346c46251246966b562c8d6839903cb205ad/libs/core/langchain_core/messages/base.py)

---

### AIMessage

**Definition:** Represents a response from a language model.

**Key Fields:**
- `content` - Text or multimodal content
- `tool_calls` - List of requested tool invocations
- `invalid_tool_calls` - Malformed tool calls
- `usage_metadata` - Token counts

**Example:**
```python
AIMessage(
    content="The answer is 42",
    tool_calls=[
        {"name": "calculator", "args": {"expr": "6*7"}, "id": "call_123"}
    ]
)
```

**Location:** [`libs/core/langchain_core/messages/ai.py`](https://github.com/langchain-ai/langchain/blob/990e346c46251246966b562c8d6839903cb205ad/libs/core/langchain_core/messages/ai.py)

---

### ToolCall

**Definition:** Represents a model's request to invoke a tool.

**Structure:**
```python
class ToolCall(TypedDict):
    name: str        # Tool name
    args: dict       # Tool arguments
    id: str | None   # Unique ID for correlation
```

**Location:** [`libs/core/langchain_core/messages/tool.py`](https://github.com/langchain-ai/langchain/blob/990e346c46251246966b562c8d6839903cb205ad/libs/core/langchain_core/messages/tool.py)

---

### ToolMessage

**Definition:** Contains the result of a tool execution.

**Structure:**
```python
class ToolMessage(BaseMessage):
    content: str | list
    tool_call_id: str    # Links to original ToolCall
    status: str          # "success" or "error"
    artifact: Any        # Full result if content is summary
```

---

## Tool Concepts

### BaseTool

**Definition:** Base class for all tools that can be invoked by models.

**Key Properties:**
- `name` - Tool identifier
- `description` - For model's understanding
- `args_schema` - Pydantic model for validation

**Location:** [`libs/core/langchain_core/tools/base.py:390`](https://github.com/langchain-ai/langchain/blob/990e346c46251246966b562c8d6839903cb205ad/libs/core/langchain_core/tools/base.py#L390)

---

### @tool decorator

**Definition:** Converts a function into a Tool.

**Example:**
```python
from langchain_core.tools import tool

@tool
def search(query: str) -> str:
    """Search the web for information."""
    return f"Results for: {query}"
```

**Location:** [`libs/core/langchain_core/tools/convert.py`](https://github.com/langchain-ai/langchain/blob/990e346c46251246966b562c8d6839903cb205ad/libs/core/langchain_core/tools/convert.py)

---

## Provider Concepts

### BaseChatModel

**Definition:** Abstract base class for chat-style language models.

**Key Methods:**
- `invoke()` - Single message list
- `generate()` - Multiple message lists
- `bind_tools()` - Attach tool definitions

**Location:** [`libs/core/langchain_core/language_models/chat_models.py`](https://github.com/langchain-ai/langchain/blob/990e346c46251246966b562c8d6839903cb205ad/libs/core/langchain_core/language_models/chat_models.py)

---

### Embeddings

**Definition:** Abstract base class for text embedding models.

**Key Methods:**
- `embed_documents()` - Embed multiple texts
- `embed_query()` - Embed single query

**Location:** [`libs/core/langchain_core/embeddings/__init__.py`](https://github.com/langchain-ai/langchain/blob/990e346c46251246966b562c8d6839903cb205ad/libs/core/langchain_core/embeddings/__init__.py)

---

### VectorStore

**Definition:** Abstract base class for vector databases.

**Key Methods:**
- `add_documents()` - Store documents with embeddings
- `similarity_search()` - Find similar documents
- `delete()` - Remove documents

**Location:** [`libs/core/langchain_core/vectorstores/__init__.py`](https://github.com/langchain-ai/langchain/blob/990e346c46251246966b562c8d6839903cb205ad/libs/core/langchain_core/vectorstores/__init__.py)

---

## Observability Concepts

### LangSmith

**Definition:** LangChain's observability platform for tracing, monitoring, and evaluation.

**Integration:** Automatic via `LANGCHAIN_TRACING_V2=true` environment variable.

---

### Run

**Definition:** A single execution of a Runnable, tracked for observability.

**Key Fields:**
- `run_id` - Unique identifier
- `parent_run_id` - For nested execution
- `start_time`, `end_time` - Timing
- `inputs`, `outputs` - Data
- `error` - If execution failed

---

### Tracer

**Definition:** Callback handler that records execution traces.

**Types:**
- `LangChainTracer` - Sends to LangSmith
- `ConsoleCallbackHandler` - Prints to console

**Location:** [`libs/core/langchain_core/tracers/`](https://github.com/langchain-ai/langchain/blob/990e346c46251246966b562c8d6839903cb205ad/libs/core/langchain_core/tracers/)

---

## Serialization Concepts

### Serializable

**Definition:** Base class for objects that can be serialized to JSON.

**Key Methods:**
- `to_json()` - Serialize to dict
- `is_lc_serializable()` - Whether serialization is supported
- `lc_secrets` - Map of secret field names

**Location:** [`libs/core/langchain_core/load/serializable.py`](https://github.com/langchain-ai/langchain/blob/990e346c46251246966b562c8d6839903cb205ad/libs/core/langchain_core/load/serializable.py)

---

## Common Patterns

### coerce_to_runnable()

**Definition:** Converts various types to Runnable automatically.

**Conversions:**
- `dict` → `RunnableParallel`
- `callable` → `RunnableLambda`
- `generator` → `RunnableGenerator`

**Location:** [`libs/core/langchain_core/runnables/base.py:6015`](https://github.com/langchain-ai/langchain/blob/990e346c46251246966b562c8d6839903cb205ad/libs/core/langchain_core/runnables/base.py#L6015)

---

### itemgetter

**Definition:** From `operator` module, used to extract keys from dict outputs.

**Example:**
```python
from operator import itemgetter

chain = ... | itemgetter("answer", "sources")
# Extracts only those keys from output dict
```

---

## Abbreviations

| Abbreviation | Full Form |
|--------------|-----------|
| LLM | Large Language Model |
| LCEL | LangChain Expression Language |
| RAG | Retrieval-Augmented Generation |
| API | Application Programming Interface |
| SDK | Software Development Kit |
| LOC | Lines of Code |
