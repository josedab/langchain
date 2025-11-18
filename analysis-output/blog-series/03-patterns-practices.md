# Patterns and Practices in LangChain

> **Part 3 of 6** | Analysis based on commit [`990e346`](https://github.com/langchain-ai/langchain/tree/990e346c46251246966b562c8d6839903cb205ad)

## What You'll Learn

- Design patterns employed in LangChain
- Error handling and resilience strategies
- Testing approaches and best practices
- Security patterns
- Code organization and style

## Introduction

LangChain is a large codebase (~135,000+ LOC) that needs to be maintainable, extensible, and reliable. In this post, we'll explore the patterns and practices that make this possible.

## Design Patterns

### Pattern 1: Chain of Responsibility

**Where it's used**: `RunnableSequence`

Each Runnable in a sequence is responsible for:
1. Processing its input
2. Passing output to the next handler
3. Optionally stopping the chain (via exception)

```python
# From: libs/core/langchain_core/runnables/base.py:2789
class RunnableSequence(RunnableSerializable[Input, Output]):
    first: Runnable[Input, Any]
    middle: list[Runnable[Any, Any]]
    last: Runnable[Any, Output]

    def invoke(self, input, config):
        result = self.first.invoke(input, config)
        for step in self.middle:
            result = step.invoke(result, config)  # Pass to next handler
        return self.last.invoke(result, config)
```

**Benefits**:
- Decoupled processing stages
- Easy to add/remove/reorder steps
- Each step has single responsibility

### Pattern 2: Decorator Pattern

**Where it's used**: `RunnableBinding`, `RunnableWithFallbacks`, `RunnableRetry`

Wraps existing Runnables to add behavior without modifying them:

```python
# From: libs/core/langchain_core/runnables/base.py:5369
class RunnableBinding(RunnableSerializable[Input, Output]):
    bound: Runnable[Input, Output]   # The wrapped Runnable
    kwargs: Mapping[str, Any]         # Extra arguments
    config: RunnableConfig            # Additional config

    def invoke(self, input, config=None, **kwargs):
        config = self._merge_configs(config)
        merged_kwargs = {**self.kwargs, **kwargs}
        return self.bound.invoke(input, config, **merged_kwargs)
```

**Usage**:
```python
# Add retry behavior
model_with_retry = model.with_retry(max_attempts=3)

# Add fallbacks
model_with_fallback = model.with_fallbacks([backup_model])

# Bind arguments
configured_model = model.bind(temperature=0.7)
```

**Benefits**:
- Open for extension, closed for modification
- Compose behaviors flexibly
- No subclassing needed

### Pattern 3: Strategy Pattern

**Where it's used**: Execution strategies in `Runnable`

The Runnable interface defines multiple execution strategies:

```python
# From: libs/core/langchain_core/runnables/base.py
class Runnable:
    def invoke(self, ...): ...     # Sync strategy
    async def ainvoke(self, ...): ... # Async strategy
    def batch(self, ...): ...      # Batch strategy
    def stream(self, ...): ...     # Streaming strategy
```

**Usage**:
```python
# Same chain, different strategies
result = chain.invoke(input)           # Sync
result = await chain.ainvoke(input)    # Async
results = chain.batch([input1, input2]) # Batch
for chunk in chain.stream(input):      # Stream
    print(chunk)
```

**Benefits**:
- Caller chooses execution mode
- Implementation details hidden
- Easy to add new strategies

### Pattern 4: Template Method

**Where it's used**: `BaseChatModel`, `BaseRetriever`

Base classes define the skeleton of an algorithm, with subclasses filling in specific steps:

```python
# From: libs/core/langchain_core/language_models/chat_models.py
class BaseChatModel(BaseLanguageModel):
    def invoke(self, input, config=None, **kwargs):
        # Template: common logic
        input = self._convert_input(input)
        config = ensure_config(config)

        # Abstract method: subclass implements
        return self._generate_with_cache(input, config, **kwargs)

    @abstractmethod
    def _generate(self, messages, **kwargs):
        """Subclasses implement this."""
        ...
```

**Benefits**:
- Consistent behavior across implementations
- DRY: common code in base class
- Easy to add new providers

### Pattern 5: Observer Pattern

**Where it's used**: Callback system

Callbacks observe and react to events during execution:

```python
# From: libs/core/langchain_core/callbacks/base.py
class BaseCallbackHandler:
    def on_llm_start(self, serialized, prompts, **kwargs): ...
    def on_llm_end(self, response, **kwargs): ...
    def on_chain_start(self, serialized, inputs, **kwargs): ...
    def on_chain_end(self, outputs, **kwargs): ...
```

**Usage**:
```python
class LoggingHandler(BaseCallbackHandler):
    def on_llm_start(self, serialized, prompts, **kwargs):
        print(f"LLM called with: {prompts}")

result = chain.invoke(input, config={"callbacks": [LoggingHandler()]})
```

**Benefits**:
- Decoupled observability
- Multiple observers per subject
- Easy to add new event types

### Pattern 6: Factory Pattern

**Where it's used**: `coerce_to_runnable()`

Creates appropriate Runnable type based on input:

```python
# From: libs/core/langchain_core/runnables/base.py:6015
def coerce_to_runnable(thing: RunnableLike) -> Runnable[Input, Output]:
    if isinstance(thing, Runnable):
        return thing
    if callable(thing):
        return RunnableLambda(thing)      # Factory creates RunnableLambda
    if isinstance(thing, dict):
        return RunnableParallel(thing)    # Factory creates RunnableParallel
    # ...
```

**Benefits**:
- Clean API for users
- Type selection logic centralized
- Easy to extend with new types

## Error Handling Patterns

### Custom Exception Hierarchy

```python
# From: libs/core/langchain_core/exceptions.py
class LangChainException(Exception):
    """Base exception for LangChain."""

class OutputParserException(LangChainException):
    """Error parsing output."""
    llm_output: str | None = None  # Include context

class TracerException(LangChainException):
    """Error in tracing."""
```

**Pattern**: Include context in exceptions for debugging.

### Error Messages with Context

```python
# From: libs/core/langchain_core/output_parsers/xml.py
try:
    return ElementTree.fromstring(text)
except ElementTree.ParseError as e:
    msg = f"Failed to parse XML: {text}. Got: {e}"
    raise OutputParserException(msg, llm_output=text) from e
```

**Best Practice**:
- Use `msg` variable for error messages
- Chain exceptions with `from e`
- Include relevant context (like `llm_output`)

### Retry with Tenacity

```python
# From: libs/core/langchain_core/runnables/retry.py
from tenacity import retry, stop_after_attempt, wait_exponential_jitter

class RunnableRetry(RunnableBinding):
    max_attempt_number: int = 3

    def invoke(self, input, config=None, **kwargs):
        @retry(
            stop=stop_after_attempt(self.max_attempt_number),
            wait=wait_exponential_jitter(),
        )
        def _invoke():
            return self.bound.invoke(input, config, **kwargs)

        return _invoke()
```

### Fallback Chains

```python
# From: libs/core/langchain_core/runnables/fallbacks.py:37
class RunnableWithFallbacks(RunnableSerializable):
    runnable: Runnable
    fallbacks: Sequence[Runnable]
    exceptions_to_handle: tuple[type[BaseException], ...] = (Exception,)

    def invoke(self, input, config=None, **kwargs):
        try:
            return self.runnable.invoke(input, config, **kwargs)
        except self.exceptions_to_handle:
            for fallback in self.fallbacks:
                try:
                    return fallback.invoke(input, config, **kwargs)
                except self.exceptions_to_handle:
                    continue
            raise
```

## Testing Practices

### Test Organization

```
libs/core/tests/
├── unit_tests/           # No network calls
│   ├── test_runnables.py
│   ├── test_messages.py
│   └── ...
└── integration_tests/    # With network
    └── ...
```

### Socket Isolation

```bash
# From Makefile
test:
    pytest tests/unit_tests --disable-socket
```

Uses `pytest-socket` to prevent accidental network calls in unit tests.

### Deterministic Testing

```python
# From: libs/core/tests/unit_tests/test_runnables.py
@pytest.fixture
def deterministic_uuids():
    """Replace UUIDs for snapshot testing."""
    counter = 0
    def make_uuid():
        nonlocal counter
        counter += 1
        return UUID(int=counter)

    with patch("uuid.uuid4", make_uuid):
        yield
```

### Snapshot Testing

```python
# Using syrupy
def test_chain_output(snapshot):
    result = chain.invoke({"input": "test"})
    assert result == snapshot  # Compare to stored snapshot
```

Snapshots stored in `.ambr` files (~381KB for runnables!).

### Async Testing

```python
# From pytest.ini
asyncio_mode = auto

# In tests
async def test_async_invoke():
    result = await chain.ainvoke({"input": "test"})
    assert result == expected
```

### Mocking LLM Calls

```python
from langchain_core.language_models import FakeChatModel
from langchain_core.messages import AIMessage

def test_with_fake_model():
    model = FakeChatModel(responses=[
        AIMessage(content="Hello!")
    ])
    result = model.invoke([HumanMessage("Hi")])
    assert result.content == "Hello!"
```

### Standard Tests

Partner packages use shared compliance tests:

```python
# From: libs/standard-tests/langchain_tests/unit_tests/chat_models.py
class ChatModelUnitTests:
    @property
    @abstractmethod
    def chat_model_class(self):
        ...

    def test_invoke(self):
        model = self.chat_model_class()
        result = model.invoke([HumanMessage("test")])
        assert isinstance(result, AIMessage)
```

## Security Patterns

### Secret Handling

```python
# From: libs/partners/openai/tests/unit_tests/test_secrets.py
from pydantic import SecretStr

class ChatOpenAI(BaseChatModel):
    openai_api_key: SecretStr | None = None

# Secrets are masked
model = ChatOpenAI(openai_api_key="sk-...")
print(model.openai_api_key)  # SecretStr('**********')

# Access actual value
key = model.openai_api_key.get_secret_value()
```

### XML Security

```python
# From: libs/core/langchain_core/output_parsers/xml.py
try:
    from defusedxml import ElementTree  # Safe by default
    _HAS_DEFUSEDXML = True
except ImportError:
    from xml.etree import ElementTree
    _HAS_DEFUSEDXML = False
```

Uses `defusedxml` to prevent XXE attacks.

### No eval() on User Input

```python
# ❌ Bad
result = eval(user_input)

# ✅ Good - Use ast.literal_eval for safe parsing
import ast
result = ast.literal_eval(user_input)
```

The codebase has minimal eval usage, mostly in test contexts.

### Environment Variable Safety

```python
# From: libs/core/langchain_core/utils/env.py
def get_from_env(key: str, env_key: str, default: str | None = None) -> str:
    if env_value := os.getenv(env_key):
        return env_value
    if default is not None:
        return default
    raise ValueError(f"Did not find {key}")  # Clear error
```

No `eval()` or `exec()` on environment variables.

## Code Style

### Type Hints Required

```python
# ❌ Bad
def process(data, options):
    return data

# ✅ Good
def process(data: dict[str, Any], options: ProcessOptions) -> Result:
    """Process data with given options.

    Args:
        data: Input data dictionary.
        options: Processing configuration.

    Returns:
        Processed result.
    """
    return Result(...)
```

### Google-Style Docstrings

```python
def search(query: str, limit: int = 10) -> list[Document]:
    """Search for documents matching query.

    Args:
        query: Search query string.
        limit: Maximum results to return.

    Returns:
        List of matching documents.

    Raises:
        SearchError: If search service unavailable.
    """
```

### Linting Configuration

```toml
# From pyproject.toml
[tool.ruff.lint]
select = ["ALL"]  # Enable all rules

ignore = [
    "COM812",   # Trailing comma
    "ISC001",   # Implicit string concat
    "C90",      # McCabe complexity (disabled)
    # ... more
]
```

Very strict by default, with documented exceptions.

### Mypy Strict Mode

```toml
[tool.mypy]
strict = true
warn_unreachable = true
```

Ensures complete type coverage.

## Code Organization

### Single Responsibility

Each file has a clear purpose:
- `base.py` - Base classes
- `config.py` - Configuration
- `utils.py` - Utilities

### Consistent Partner Structure

```
partners/{provider}/
├── langchain_{provider}/
│   ├── chat_models.py
│   ├── embeddings/
│   └── __init__.py
├── pyproject.toml
└── tests/
```

### Import Organization

```python
# Standard library
import os
from typing import Any

# Third-party
from pydantic import BaseModel

# Local
from langchain_core.runnables import Runnable
from .utils import helper
```

## Best Practices Summary

### Do

✅ Use type hints everywhere
✅ Write Google-style docstrings
✅ Handle errors with context
✅ Use `SecretStr` for secrets
✅ Isolate unit tests from network
✅ Use deterministic fixtures for snapshots
✅ Chain exceptions with `from e`

### Don't

❌ Use bare `except:`
❌ Use `eval()` on user input
❌ Skip type hints
❌ Make network calls in unit tests
❌ Store secrets in logs/traces

## Anti-Patterns to Avoid

### Anti-Pattern 1: God Functions

```python
# ❌ Bad: Too much in one function
def process_and_store_and_notify(data, db, email):
    validated = validate(data)
    db.save(validated)
    email.send(validated)
    return validated

# ✅ Good: Single responsibility
def process(data: dict) -> ValidatedData:
    return validate(data)

def store(data: ValidatedData, db: Database) -> None:
    db.save(data)
```

### Anti-Pattern 2: Magic Values

```python
# ❌ Bad
if status == 1:
    ...

# ✅ Good
from enum import Enum

class Status(Enum):
    SUCCESS = 1
    FAILURE = 2

if status == Status.SUCCESS:
    ...
```

### Anti-Pattern 3: Silent Failures

```python
# ❌ Bad
try:
    result = risky_operation()
except:
    pass  # Silent failure

# ✅ Good
try:
    result = risky_operation()
except SpecificError as e:
    logger.error(f"Operation failed: {e}")
    raise
```

## Key Takeaways

1. **Patterns enable flexibility**: Decorator, Strategy, and Chain of Responsibility work together.

2. **Testing is comprehensive**: Socket isolation, snapshots, and standard tests ensure quality.

3. **Security is explicit**: SecretStr, defusedxml, and careful eval usage.

4. **Consistency matters**: All packages follow same structure and style.

5. **Errors include context**: Exception chaining and informative messages.

## What's Next

In [Part 4](./04-extending-integrating.md), we'll learn how to extend LangChain—creating custom Runnables, building provider integrations, and developing tools.

---

**Code references**: All examples reference [`990e346`](https://github.com/langchain-ai/langchain/tree/990e346c46251246966b562c8d6839903cb205ad).
