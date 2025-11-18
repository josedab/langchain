# RFC-0005: Add Property-Based Testing with Hypothesis

**Status:** Draft
**Author:** Codebase Analysis
**Created:** 2025-11-18
**Effort:** 2 weeks
**Category:** Strategic

## Summary

Introduce property-based testing using Hypothesis to automatically discover edge cases and increase confidence in core abstractions like Runnables, messages, and serialization.

## Motivation

Current testing approach:
- Example-based tests (specific inputs/outputs)
- Good coverage but limited edge case discovery
- Manual test case creation

Property-based testing benefits:
- Automatically generates test cases
- Finds edge cases developers don't anticipate
- Tests invariants, not specific examples
- Shrinks failures to minimal reproducible cases

Areas that would benefit:
- Runnable composition (any input type should work)
- Message serialization (round-trip should preserve data)
- Config merging (should be associative, commutative for some fields)
- Type coercion (should handle all valid inputs)

## Detailed Design

### Add Hypothesis Dependency

```toml
# libs/core/pyproject.toml
[project.optional-dependencies]
test = [
    # ... existing
    "hypothesis>=6.100.0,<7.0.0",
]
```

### Property Tests for Runnables

```python
# tests/unit_tests/test_runnables_properties.py
from hypothesis import given, strategies as st, settings
from langchain_core.runnables import RunnableLambda, RunnableSequence

# Strategy for various input types
any_input = st.one_of(
    st.text(),
    st.integers(),
    st.floats(allow_nan=False),
    st.lists(st.integers()),
    st.dictionaries(st.text(), st.integers()),
)

@given(input_value=any_input)
def test_runnable_lambda_identity(input_value):
    """Identity function should return input unchanged."""
    identity = RunnableLambda(lambda x: x)
    assert identity.invoke(input_value) == input_value

@given(input_value=any_input)
def test_runnable_sequence_single_is_identity(input_value):
    """Single-element sequence should act as identity wrapper."""
    identity = RunnableLambda(lambda x: x)
    sequence = identity | RunnableLambda(lambda x: x)

    result = sequence.invoke(input_value)
    assert result == input_value

@given(a=st.integers(), b=st.integers(), c=st.integers())
def test_runnable_sequence_associativity(a, b, c):
    """Sequence composition should be associative."""
    f = RunnableLambda(lambda x: x + a)
    g = RunnableLambda(lambda x: x * b)
    h = RunnableLambda(lambda x: x - c)

    # (f | g) | h should equal f | (g | h)
    left = (f | g) | h
    right = f | (g | h)

    result_left = left.invoke(0)
    result_right = right.invoke(0)

    assert result_left == result_right
```

### Property Tests for Config

```python
# tests/unit_tests/test_config_properties.py
from hypothesis import given, strategies as st
from langchain_core.runnables.config import merge_configs

config_strategy = st.fixed_dictionaries({
    "tags": st.lists(st.text(min_size=1, max_size=10), max_size=5),
    "metadata": st.dictionaries(
        st.text(min_size=1, max_size=10),
        st.one_of(st.text(), st.integers(), st.booleans()),
        max_size=5
    ),
})

@given(config=config_strategy)
def test_merge_with_empty_is_identity(config):
    """Merging with empty config should return original."""
    result = merge_configs(config, {})
    assert result["tags"] == sorted(set(config["tags"]))
    for key, value in config.get("metadata", {}).items():
        assert result["metadata"][key] == value

@given(c1=config_strategy, c2=config_strategy)
def test_merge_tags_are_deduplicated(c1, c2):
    """Merged tags should have no duplicates."""
    result = merge_configs(c1, c2)
    tags = result.get("tags", [])
    assert len(tags) == len(set(tags))

@given(c1=config_strategy, c2=config_strategy, c3=config_strategy)
def test_merge_is_associative_for_metadata(c1, c2, c3):
    """Metadata merge should be associative."""
    left = merge_configs(merge_configs(c1, c2), c3)
    right = merge_configs(c1, merge_configs(c2, c3))

    assert left["metadata"] == right["metadata"]
```

### Property Tests for Serialization

```python
# tests/unit_tests/test_serialization_properties.py
from hypothesis import given, strategies as st
from langchain_core.messages import HumanMessage, AIMessage, SystemMessage
from langchain_core.load import dumps, loads

message_content = st.text(min_size=0, max_size=1000)

@given(content=message_content)
def test_human_message_roundtrip(content):
    """Serialization should preserve HumanMessage content."""
    msg = HumanMessage(content=content)
    serialized = dumps(msg)
    deserialized = loads(serialized)
    assert deserialized.content == content

@given(content=message_content)
def test_ai_message_roundtrip(content):
    """Serialization should preserve AIMessage content."""
    msg = AIMessage(content=content)
    serialized = dumps(msg)
    deserialized = loads(serialized)
    assert deserialized.content == content

@given(
    content=message_content,
    metadata=st.dictionaries(st.text(max_size=10), st.text(max_size=100), max_size=5)
)
def test_message_metadata_preserved(content, metadata):
    """Serialization should preserve message metadata."""
    msg = HumanMessage(content=content, additional_kwargs=metadata)
    serialized = dumps(msg)
    deserialized = loads(serialized)
    assert deserialized.additional_kwargs == metadata
```

### Property Tests for Type Coercion

```python
# tests/unit_tests/test_coercion_properties.py
from hypothesis import given, strategies as st
from langchain_core.runnables.base import coerce_to_runnable

@given(value=st.integers())
def test_callable_coercion_preserves_behavior(value):
    """Coerced callable should preserve function behavior."""
    def double(x):
        return x * 2

    runnable = coerce_to_runnable(double)
    assert runnable.invoke(value) == double(value)

@given(keys=st.lists(st.text(min_size=1), min_size=1, max_size=5, unique=True))
def test_dict_coercion_creates_parallel(keys):
    """Dict should coerce to RunnableParallel with same keys."""
    d = {k: (lambda x, k=k: f"{k}:{x}") for k in keys}
    runnable = coerce_to_runnable(d)

    result = runnable.invoke("test")
    assert set(result.keys()) == set(keys)
```

### Custom Strategies

```python
# tests/conftest.py
from hypothesis import strategies as st

# Custom strategy for LangChain messages
@st.composite
def message_strategy(draw):
    """Generate random valid message."""
    msg_type = draw(st.sampled_from([HumanMessage, AIMessage, SystemMessage]))
    content = draw(st.text(max_size=500))
    return msg_type(content=content)

# Custom strategy for tool calls
@st.composite
def tool_call_strategy(draw):
    """Generate random valid tool call."""
    return {
        "name": draw(st.text(min_size=1, max_size=50).filter(str.isidentifier)),
        "args": draw(st.dictionaries(st.text(min_size=1, max_size=20), st.text(), max_size=5)),
        "id": draw(st.text(min_size=1, max_size=50)),
    }
```

## Example Usage

### Running Property Tests

```bash
# Run all property tests
pytest tests/unit_tests/test_*_properties.py

# Run with more examples
pytest tests/unit_tests/test_*_properties.py --hypothesis-seed=0 -v

# Show statistics
pytest tests/unit_tests/test_*_properties.py --hypothesis-show-statistics
```

### Example Failure Output

```
FAILED tests/unit_tests/test_serialization_properties.py::test_roundtrip
Falsifying example: test_roundtrip(
    content='\x00'  # Null byte causes issue
)

Explanation: The serializer doesn't handle null bytes in strings.
```

## Implementation Plan

| Phase | Task | Duration |
|-------|------|----------|
| 1 | Add Hypothesis dependency | 1 hour |
| 2 | Create custom strategies | 2 days |
| 3 | Runnable property tests | 3 days |
| 4 | Config property tests | 2 days |
| 5 | Serialization property tests | 2 days |
| 6 | Type coercion tests | 1 day |
| 7 | CI integration | 1 day |
| 8 | Documentation | 1 day |

**Total:** 2 weeks

## Backwards Compatibility

No impact. Tests are additive.

## Alternatives Considered

### Alternative 1: Manual edge case tests
**Rejected:** Doesn't scale; developers miss cases.

### Alternative 2: Fuzzing with atheris
**Rejected:** Overkill; Hypothesis is better for API testing.

### Alternative 3: Only for serialization
**Rejected:** Many components benefit from property testing.

## Open Questions

1. **Settings**: Default max_examples (100) or higher?
2. **CI time**: How to balance thoroughness vs speed?
3. **Database**: Use hypothesis database for reproducibility?
4. **Profiles**: Separate settings for CI vs local?

## Success Criteria

- [ ] Property tests for all core Runnable types
- [ ] Config merging properties verified
- [ ] Serialization round-trip tested
- [ ] CI runs property tests
- [ ] <30 second test time increase
- [ ] At least 3 bugs found by property tests

## References

- [Hypothesis Documentation](https://hypothesis.readthedocs.io/)
- [Property-Based Testing with Python](https://semaphoreci.com/community/tutorials/testing-with-property-based-testing-in-python)
- [The Algebra of Algebraic Data Types](https://chris-taylor.github.io/blog/2013/02/10/the-algebra-of-algebraic-data-types/)
