# RFC-0006: Standardize Error Messages

**Status:** Draft
**Author:** Codebase Analysis
**Created:** 2025-11-18
**Effort:** 3 weeks
**Category:** Strategic

## Summary

Standardize error message format across LangChain to improve debugging experience, enable better error handling, and provide actionable guidance to users.

## Motivation

Current error messages are inconsistent:

```python
# Inconsistent formats
raise ValueError("Invalid input")  # Too vague
raise ValueError(f"Failed to parse: {text}")  # No guidance
raise OutputParserException(f"Could not parse: {e}")  # Missing context
```

Problems:
- Vague messages don't help debugging
- Inconsistent format across modules
- Missing actionable guidance
- No error codes for programmatic handling
- Context lost in exception chains

## Detailed Design

### Error Message Standard

```python
# Standard format
"{Error Type}: {What happened}. {Why it matters}. {How to fix}."

# Example
"ValidationError: Input must be a dict, got str.
The chain expects structured input with keys ['question', 'context'].
Wrap your input: chain.invoke({'question': your_input})"
```

### Error Categories

```python
# libs/core/langchain_core/errors.py

class LangChainError(Exception):
    """Base error with standard format."""

    error_code: str = "LC000"

    def __init__(
        self,
        message: str,
        *,
        details: dict | None = None,
        suggestion: str | None = None,
        docs_url: str | None = None,
    ):
        self.message = message
        self.details = details or {}
        self.suggestion = suggestion
        self.docs_url = docs_url
        super().__init__(self._format_message())

    def _format_message(self) -> str:
        parts = [f"[{self.error_code}] {self.message}"]

        if self.suggestion:
            parts.append(f"\nSuggestion: {self.suggestion}")

        if self.docs_url:
            parts.append(f"\nDocs: {self.docs_url}")

        return "".join(parts)

class InputValidationError(LangChainError):
    """Invalid input to a Runnable."""
    error_code = "LC001"

class OutputParsingError(LangChainError):
    """Failed to parse model output."""
    error_code = "LC002"

class ConfigurationError(LangChainError):
    """Invalid configuration."""
    error_code = "LC003"

class SerializationError(LangChainError):
    """Failed to serialize/deserialize."""
    error_code = "LC004"

class ProviderError(LangChainError):
    """Error from LLM provider."""
    error_code = "LC005"
```

### Usage Examples

#### Input Validation

```python
# Before
if not isinstance(input, dict):
    raise ValueError("Input must be a dict")

# After
if not isinstance(input, dict):
    raise InputValidationError(
        f"Expected dict input, got {type(input).__name__}",
        details={
            "expected_type": "dict",
            "actual_type": type(input).__name__,
            "expected_keys": ["question", "context"],
        },
        suggestion="Wrap your input: chain.invoke({'question': your_text})",
        docs_url="https://python.langchain.com/docs/concepts/runnables"
    )
```

#### Output Parsing

```python
# Before
try:
    result = json.loads(text)
except json.JSONDecodeError as e:
    raise OutputParserException(f"Failed to parse JSON: {e}")

# After
try:
    result = json.loads(text)
except json.JSONDecodeError as e:
    raise OutputParsingError(
        "Model output is not valid JSON",
        details={
            "output": text[:200],  # Truncate for readability
            "parse_error": str(e),
            "error_position": e.pos,
        },
        suggestion=(
            "Ensure your prompt asks for JSON output explicitly. "
            "Example: 'Respond with valid JSON only.'"
        ),
        docs_url="https://python.langchain.com/docs/concepts/output_parsers"
    ) from e
```

#### Configuration

```python
# Before
if recursion_limit < 1:
    raise ValueError("recursion_limit must be at least 1")

# After
if recursion_limit < 1:
    raise ConfigurationError(
        f"recursion_limit must be >= 1, got {recursion_limit}",
        details={"parameter": "recursion_limit", "value": recursion_limit},
        suggestion="Set a positive recursion_limit in your config",
    )
```

### Error Code Registry

```python
# libs/core/langchain_core/errors.py

ERROR_CODES = {
    "LC000": "Generic LangChain error",
    "LC001": "Input validation failed",
    "LC002": "Output parsing failed",
    "LC003": "Configuration error",
    "LC004": "Serialization error",
    "LC005": "Provider API error",
    "LC006": "Timeout exceeded",
    "LC007": "Rate limit exceeded",
    "LC008": "Authentication failed",
    "LC009": "Resource not found",
    "LC010": "Callback error",
}
```

### Logging Integration

```python
import logging

logger = logging.getLogger(__name__)

class LangChainError(Exception):
    def __init__(self, ...):
        ...
        # Auto-log errors at appropriate level
        if self.error_code.startswith("LC00"):
            logger.error(self._format_message(), extra={"error_code": self.error_code})
```

### Error Handling Best Practices

```python
# Good: Catch specific errors
try:
    result = chain.invoke(input)
except InputValidationError as e:
    # Handle validation errors (user's fault)
    return {"error": str(e), "code": e.error_code}
except ProviderError as e:
    # Handle provider errors (retry might help)
    return {"error": "Service unavailable", "retry_after": 60}
except LangChainError as e:
    # Handle other LangChain errors
    logger.error(f"Unexpected error: {e}")
    raise
```

## Example Usage

### User Experience

```python
>>> from langchain_core.runnables import RunnableLambda
>>> chain = RunnableLambda(lambda x: x["key"])
>>> chain.invoke("not a dict")

InputValidationError: [LC001] Expected dict input, got str
Details: {'expected_type': 'dict', 'actual_type': 'str'}
Suggestion: Wrap your input: chain.invoke({'key': your_value})
Docs: https://python.langchain.com/docs/concepts/runnables
```

### Programmatic Handling

```python
try:
    result = chain.invoke(input)
except LangChainError as e:
    if e.error_code == "LC001":
        # Input validation - fix input
        result = chain.invoke({"question": input})
    elif e.error_code == "LC005":
        # Provider error - retry with backoff
        time.sleep(e.details.get("retry_after", 60))
        result = chain.invoke(input)
    else:
        raise
```

## Implementation Plan

| Phase | Task | Duration |
|-------|------|----------|
| 1 | Design error hierarchy | 2 days |
| 2 | Implement base classes | 2 days |
| 3 | Migrate core errors | 5 days |
| 4 | Migrate partner errors | 3 days |
| 5 | Add error codes registry | 1 day |
| 6 | Update documentation | 2 days |
| 7 | Add tests | 2 days |

**Total:** 3 weeks

## Backwards Compatibility

### Breaking Changes

Old exception types will be deprecated but not removed immediately:

```python
# Deprecation period
class OutputParserException(OutputParsingError):
    """Deprecated: Use OutputParsingError instead."""

    def __init__(self, message, **kwargs):
        import warnings
        warnings.warn(
            "OutputParserException is deprecated, use OutputParsingError",
            DeprecationWarning
        )
        super().__init__(message, **kwargs)
```

### Migration Path

1. **v1.1**: New error classes added alongside old
2. **v1.2**: Old classes emit deprecation warnings
3. **v2.0**: Old classes removed

## Alternatives Considered

### Alternative 1: Use Python's built-in exceptions only
**Rejected:** Doesn't provide structure or guidance.

### Alternative 2: Use exception notes (Python 3.11+)
**Rejected:** Requires Python 3.11+; LangChain supports 3.10.

### Alternative 3: Return error objects instead of raising
**Rejected:** Breaks Python conventions; complicates code.

## Open Questions

1. **Error codes**: Start at LC000 or LC001?
2. **Localization**: Support for non-English messages?
3. **Telemetry**: Send anonymous error codes for improvement?
4. **Severity levels**: Add severity to error classes?

## Success Criteria

- [ ] All public exceptions follow standard format
- [ ] Error codes documented
- [ ] Suggestions provided for common errors
- [ ] Docs URLs point to relevant documentation
- [ ] Migration guide published
- [ ] Partner packages updated

## References

- [Python Exception Best Practices](https://docs.python.org/3/tutorial/errors.html)
- [Stripe Error Codes](https://stripe.com/docs/error-codes)
- [AWS Error Responses](https://docs.aws.amazon.com/AmazonS3/latest/API/ErrorResponses.html)
