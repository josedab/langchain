# Understanding LangChain: Architecture and Core Concepts

> **Part 1 of 6** | Analysis based on commit [`990e346`](https://github.com/langchain-ai/langchain/tree/990e346c46251246966b562c8d6839903cb205ad)

## What You'll Learn

- How LangChain's monorepo is organized
- The philosophy behind "minimal core, maximum extensibility"
- Key architectural decisions and their trade-offs
- Core abstractions and how they relate to each other

## Introduction

LangChain is one of the most popular frameworks for building applications with Large Language Models. But what makes it tick? In this series, we'll explore the internals of LangChain, understanding not just *what* it does but *why* it's designed the way it is.

Let's start by understanding the high-level architecture and the reasoning behind key design choices.

## The Monorepo Structure

LangChain is organized as a monorepo containing 18+ interconnected Python packages. This isn't accidental—it's a deliberate design choice that enables the "minimal core, maximum extensibility" philosophy.

```
langchain/
├── libs/
│   ├── core/               # Foundation (langchain-core)
│   ├── langchain/          # Legacy chains (langchain-classic)
│   ├── cli/                # CLI tooling
│   ├── text-splitters/     # Text utilities
│   ├── standard-tests/     # Compliance tests
│   └── partners/           # Provider integrations
│       ├── openai/
│       ├── anthropic/
│       ├── chroma/
│       └── ... (15+ more)
└── pyproject.toml
```

### Why a Monorepo?

1. **Independent versioning**: Each package has its own version. OpenAI integration can update without affecting Anthropic users.

2. **Selective installation**: Users install only what they need:
   ```bash
   pip install langchain-openai  # Just OpenAI
   pip install langchain-anthropic  # Just Anthropic
   ```

3. **Shared tooling**: All packages share linting, testing, and CI configuration.

4. **Easy cross-package development**: Contributors can modify core and partners together.

5. **Consistent quality**: Shared CI/CD pipelines ensure all packages meet the same quality bar for testing, linting, and type checking.

6. **Atomic releases**: Related changes across packages can be released together, ensuring compatibility.

This structure mirrors successful patterns seen in other large Python projects like the Scientific Python ecosystem (NumPy, SciPy, Pandas) and modern JavaScript monorepos using tools like Lerna or Turborepo.

## The Package Hierarchy

Let's understand how packages depend on each other:

```
                    ┌──────────────────┐
                    │   User App       │
                    └────────┬─────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│langchain-openai │ │langchain-anthro │ │ langchain-cli   │
│  (Provider)     │ │   (Provider)    │ │   (Tooling)     │
└────────┬────────┘ └────────┬────────┘ └────────┬────────┘
         │                   │                   │
         └─────────┬─────────┘                   │
                   │                             │
                   ▼                             │
        ┌─────────────────────┐                  │
        │  langchain-classic  │◄─────────────────┘
        │  (Legacy Chains)    │
        └─────────┬───────────┘
                  │
                  ▼
        ┌─────────────────────┐
        │   langchain-core    │
        │   (Foundation)      │
        └─────────────────────┘
```

### langchain-core: The Foundation

The core package contains zero provider-specific code. It defines:

- **Runnable protocol**: Universal interface for all executable components
- **Base classes**: `BaseChatModel`, `Embeddings`, `VectorStore`, `BaseRetriever`
- **Message types**: `HumanMessage`, `AIMessage`, `ToolMessage`
- **Callback system**: Event handling for observability
- **Serialization**: JSON-based persistence

📌 **Key Insight**: Core has only 7 direct dependencies. This is intentional—it ensures the foundation is stable and lightweight.

```toml
# From libs/core/pyproject.toml
dependencies = [
    "langsmith>=0.3.45,<1.0.0",
    "tenacity!=8.4.0,>=8.1.0,<10.0.0",
    "jsonpatch>=1.33.0,<2.0.0",
    "PyYAML>=5.3.0,<7.0.0",
    "pydantic>=2.7.4,<3.0.0",
    "typing-extensions>=4.7.0,<5.0.0",
    "packaging>=23.2.0,<26.0.0",
]
```

### Partner Packages: Provider Integrations

Each LLM provider gets its own package:

```python
# Each partner is independent
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic
from langchain_ollama import ChatOllama

# But they all share the same interface
model: BaseChatModel = ChatOpenAI()  # or ChatAnthropic()
result = model.invoke([HumanMessage("Hello")])
```

Partner packages follow a consistent structure:

```
partners/openai/
├── langchain_openai/
│   ├── chat_models.py      # Main implementation
│   ├── embeddings/
│   └── middleware/
├── pyproject.toml
└── tests/
```

## Core Abstractions

Let's examine the key abstractions that make LangChain work.

### 1. The Runnable Protocol

Every component in LangChain implements the `Runnable` protocol. This is the heart of the framework.

```python
# From: libs/core/langchain_core/runnables/base.py:124
class Runnable(ABC, Generic[Input, Output]):
    """A unit of work that can be invoked, batched, streamed, transformed."""

    def invoke(self, input: Input, config: RunnableConfig | None = None) -> Output:
        """Transform a single input into an output."""
        ...

    async def ainvoke(self, input: Input, config: RunnableConfig | None = None) -> Output:
        """Async version of invoke."""
        ...

    def batch(self, inputs: list[Input], ...) -> list[Output]:
        """Process multiple inputs."""
        ...

    def stream(self, input: Input, ...) -> Iterator[Output]:
        """Stream output chunks."""
        ...
```

**Why this matters**: Any Runnable can be:
- Called synchronously or asynchronously
- Batched for efficiency
- Streamed for real-time output
- Composed with other Runnables

### 2. Messages

Chat models communicate via structured messages:

```python
# From: libs/core/langchain_core/messages/
from langchain_core.messages import (
    HumanMessage,    # User input
    AIMessage,       # Model response
    SystemMessage,   # Instructions
    ToolMessage,     # Tool results
)

messages = [
    SystemMessage("You are helpful."),
    HumanMessage("What is 2+2?"),
    AIMessage("4"),
]
```

### 3. Tools

Tools let models interact with external systems:

```python
# From: libs/core/langchain_core/tools/
from langchain_core.tools import tool

@tool
def search(query: str) -> str:
    """Search the web for information."""
    return f"Results for: {query}"

# Tools become first-class Runnables
result = search.invoke({"query": "Python"})
```

### 4. Callbacks

The callback system enables observability:

```python
# From: libs/core/langchain_core/callbacks/base.py
class BaseCallbackHandler(ABC):
    def on_llm_start(self, serialized, prompts, **kwargs): ...
    def on_llm_end(self, response, **kwargs): ...
    def on_chain_start(self, serialized, inputs, **kwargs): ...
    def on_chain_end(self, outputs, **kwargs): ...
    def on_tool_start(self, serialized, input_str, **kwargs): ...
    def on_tool_end(self, output, **kwargs): ...
```

Every Runnable invocation can be traced, logged, and monitored. This is not an afterthought—observability was designed into the framework from the beginning. When you enable LangSmith tracing (via environment variables), every chain execution is automatically recorded with:

- Input/output data
- Timing information
- Token usage and costs
- Error traces
- Parent-child relationships for nested calls

This makes debugging production issues significantly easier compared to frameworks where observability is bolted on later.

## Development Workflow

Before diving into patterns, let's understand how to work with the codebase:

```bash
# Clone and set up
cd libs/core
uv sync

# Run tests (no network calls allowed)
make test

# Code quality
make lint format

# Type checking
uv run --group lint mypy .
```

The project uses modern Python tooling:
- **uv**: Fast package manager (replacing pip)
- **ruff**: Linting and formatting (replacing flake8, black, isort)
- **mypy**: Static type checking with strict mode
- **pytest**: Testing with socket isolation to prevent accidental network calls

This tooling ensures that contributions maintain the high quality bar expected across all packages.

## Architectural Pattern: Chain of Responsibility + Strategy

LangChain combines several classic design patterns:

### Chain of Responsibility

`RunnableSequence` implements this pattern—each Runnable processes input and passes output to the next:

```python
# From: libs/core/langchain_core/runnables/base.py:2789
class RunnableSequence(RunnableSerializable[Input, Output]):
    first: Runnable[Input, Any]
    middle: list[Runnable[Any, Any]]
    last: Runnable[Any, Output]

    def invoke(self, input, config):
        # Process through chain
        result = self.first.invoke(input, config)
        for step in self.middle:
            result = step.invoke(result, config)
        return self.last.invoke(result, config)
```

### Strategy Pattern

The Runnable protocol defines multiple execution strategies:

```python
# Same chain, different strategies
chain = prompt | model | parser

chain.invoke(input)           # Strategy: synchronous
await chain.ainvoke(input)    # Strategy: asynchronous
chain.batch([input1, input2]) # Strategy: parallel
chain.stream(input)           # Strategy: streaming
```

### Decorator Pattern

`RunnableBinding` wraps Runnables with additional behavior:

```python
# From: libs/core/langchain_core/runnables/base.py:5369
class RunnableBinding(RunnableSerializable[Input, Output]):
    bound: Runnable[Input, Output]
    kwargs: Mapping[str, Any]
    config: RunnableConfig

# Usage
model_with_temp = model.bind(temperature=0.7)
model_with_retry = model.with_retry(max_attempts=3)
```

## Key Design Decisions and Trade-offs

### Decision 1: Everything is a Runnable

**Trade-off**: Simplicity vs. Uniformity

✅ **Benefits**:
- Consistent interface everywhere
- Composition via `|` operator
- Automatic streaming/batching support

❌ **Costs**:
- Overhead for simple operations
- Learning curve for new users
- "Magic" behavior can be surprising

### Decision 2: Configuration Propagation

`RunnableConfig` flows through the entire chain automatically:

```python
result = chain.invoke(
    input,
    config={
        "tags": ["production"],
        "callbacks": [my_tracer],
        "max_concurrency": 5
    }
)
# Config automatically reaches every component
```

**Trade-off**: Convenience vs. Explicitness

✅ **Benefits**:
- Observability without boilerplate
- Per-invocation configuration
- Consistent tracing

❌ **Costs**:
- Implicit behavior
- Debugging complexity
- Memory overhead

### Decision 3: Separate Core from Providers

**Trade-off**: Bundle size vs. Integration ease

✅ **Benefits**:
- Lean installations
- Independent versioning
- Clear boundaries

❌ **Costs**:
- Multiple packages to install
- Version compatibility concerns
- Fragmented documentation

### Decision 4: Pydantic v2 Only

LangChain requires Pydantic v2.7.4+:

```toml
pydantic = ">=2.7.4,<3.0.0"
```

**Trade-off**: Modern features vs. Migration burden

✅ **Benefits**:
- Better performance
- Improved validation
- Better type inference

❌ **Costs**:
- Breaking change from v1
- Ecosystem compatibility issues

## When LangChain Shines (and When It Doesn't)

### Best For

- **LLM chains**: Where network latency dominates, the Runnable overhead is negligible
- **Complex workflows**: Branching, fallbacks, and parallel execution
- **Production monitoring**: Built-in observability with LangSmith
- **Provider flexibility**: Switch between OpenAI, Anthropic, local models easily

### Less Suited For

- **Simple scripts**: The abstraction overhead isn't worth it for one-off tasks
- **Tight loops**: Each invocation creates callbacks, validates config, and manages state
- **CPU-bound work**: Designed for I/O-bound LLM operations where network latency dominates
- **Minimal dependencies**: Core is lean, but full features need multiple packages installed

## Diagram: Component Relationships

```
┌─────────────────────────────────────────────────────────────┐
│                    LangChain Architecture                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐      ┌─────────────┐     ┌─────────────┐  │
│  │   Prompt    │  |   │    Model    │  |  │   Parser    │  │
│  │  Template   │──────▶│  (LLM/Chat) │─────▶│  (Output)   │  │
│  └─────────────┘      └──────┬──────┘     └─────────────┘  │
│         │                    │                    │        │
│         ▼                    ▼                    ▼        │
│  ┌───────────────────────────────────────────────────────┐ │
│  │              Runnable Protocol                        │ │
│  │  invoke() | ainvoke() | batch() | stream()           │ │
│  └───────────────────────────────────────────────────────┘ │
│                           │                                │
│         ┌─────────────────┼─────────────────┐              │
│         ▼                 ▼                 ▼              │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐       │
│  │  Callbacks  │   │   Config    │   │   Tracing   │       │
│  │  (Events)   │   │(Propagation)│   │ (LangSmith) │       │
│  └─────────────┘   └─────────────┘   └─────────────┘       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Key Takeaways

1. **Monorepo enables modularity**: Install only what you need, update independently, and avoid dependency conflicts.

2. **Runnable is the universal interface**: Everything speaks the same protocol.

3. **Core is deliberately minimal**: 7 dependencies, zero provider code.

4. **Trade-offs are intentional**: The design optimizes for LLM workloads where network latency dominates.

5. **Patterns combine strategically**: Chain of Responsibility + Strategy + Decorator patterns enable powerful, flexible composition.

## What's Next

In [Part 2](./02-deep-dive-runnables.md), we'll dive deep into the Runnable protocol—understanding how `RunnableSequence`, `RunnableParallel`, and `RunnableLambda` work under the hood, and mastering LCEL (LangChain Expression Language).

---

**Questions or feedback?** Open an issue at [langchain-ai/langchain](https://github.com/langchain-ai/langchain/issues).
