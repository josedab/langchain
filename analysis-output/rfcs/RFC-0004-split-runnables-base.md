# RFC-0004: Split runnables/base.py into Smaller Modules

**Status:** Draft
**Author:** Codebase Analysis
**Created:** 2025-11-18
**Effort:** 2 weeks
**Category:** Strategic

## Summary

Split the 6,100-line `runnables/base.py` file into focused modules, improving maintainability, reducing cognitive load, and enabling better code organization.

## Motivation

`libs/core/langchain_core/runnables/base.py` is extremely large:
- **6,100+ lines of code**
- Multiple distinct concerns
- Difficult to navigate
- Hard to review changes
- Merge conflicts common

The file contains:
- Core `Runnable` class
- `RunnableSequence`
- `RunnableParallel`
- `RunnableLambda`
- `RunnableGenerator`
- `RunnableBinding`
- `RunnableEach`
- Helper functions
- Type definitions

## Detailed Design

### Proposed Module Structure

```
runnables/
├── __init__.py           # Public exports (unchanged API)
├── base.py               # Core Runnable ABC (500 lines)
├── sequence.py           # RunnableSequence (800 lines)
├── parallel.py           # RunnableParallel (600 lines)
├── lambda_.py            # RunnableLambda, RunnableGenerator (800 lines)
├── binding.py            # RunnableBinding, RunnableBindingBase (500 lines)
├── each.py               # RunnableEach, RunnableEachBase (300 lines)
├── protocols.py          # Type protocols (200 lines)
├── utils.py              # coerce_to_runnable, helpers (300 lines)
├── config.py             # (existing, unchanged)
├── fallbacks.py          # (existing, unchanged)
├── branch.py             # (existing, unchanged)
└── ...
```

### Migration Strategy

#### Phase 1: Create New Modules

```python
# runnables/sequence.py
from __future__ import annotations

from typing import TYPE_CHECKING

from langchain_core.runnables.base import Runnable, RunnableSerializable

if TYPE_CHECKING:
    from langchain_core.runnables.config import RunnableConfig

class RunnableSequence(RunnableSerializable[Input, Output]):
    """Compose Runnables in sequence."""
    # ... implementation moved from base.py
```

#### Phase 2: Update Imports in base.py

```python
# runnables/base.py (after splitting)
from langchain_core.runnables.sequence import RunnableSequence
from langchain_core.runnables.parallel import RunnableParallel
from langchain_core.runnables.lambda_ import RunnableLambda, RunnableGenerator

# Re-export for backwards compatibility
__all__ = [
    "Runnable",
    "RunnableSequence",
    "RunnableParallel",
    "RunnableLambda",
    "RunnableGenerator",
    # ...
]
```

#### Phase 3: Update __init__.py

```python
# runnables/__init__.py
from langchain_core.runnables.base import (
    Runnable,
    RunnableSerializable,
)
from langchain_core.runnables.sequence import RunnableSequence
from langchain_core.runnables.parallel import RunnableParallel
from langchain_core.runnables.lambda_ import RunnableLambda, RunnableGenerator
from langchain_core.runnables.binding import RunnableBinding
from langchain_core.runnables.each import RunnableEach
from langchain_core.runnables.utils import coerce_to_runnable
# ... all public exports

__all__ = [
    "Runnable",
    "RunnableSequence",
    "RunnableParallel",
    # ... complete list
]
```

### Class Distribution

| Module | Classes | LOC |
|--------|---------|-----|
| `base.py` | Runnable, RunnableSerializable | ~800 |
| `sequence.py` | RunnableSequence | ~600 |
| `parallel.py` | RunnableParallel | ~500 |
| `lambda_.py` | RunnableLambda, RunnableGenerator | ~800 |
| `binding.py` | RunnableBinding, RunnableBindingBase | ~500 |
| `each.py` | RunnableEach, RunnableEachBase | ~300 |
| `protocols.py` | Type protocols | ~200 |
| `utils.py` | Helpers | ~300 |

### Handling Circular Imports

Use `TYPE_CHECKING` pattern:

```python
from __future__ import annotations
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from langchain_core.runnables.sequence import RunnableSequence

class Runnable:
    def __or__(self, other) -> "RunnableSequence":
        from langchain_core.runnables.sequence import RunnableSequence
        return RunnableSequence(self, other)
```

## Example Usage

### Before

```python
# Everything from one file
from langchain_core.runnables.base import (
    Runnable,
    RunnableSequence,
    RunnableParallel,
    RunnableLambda,
    coerce_to_runnable,
)
```

### After

```python
# Same import still works (backwards compatible)
from langchain_core.runnables.base import (
    Runnable,
    RunnableSequence,
    RunnableParallel,
    RunnableLambda,
    coerce_to_runnable,
)

# Or more specific imports
from langchain_core.runnables.sequence import RunnableSequence
from langchain_core.runnables.parallel import RunnableParallel
```

## Implementation Plan

| Phase | Task | Duration |
|-------|------|----------|
| 1 | Design module structure | 2 days |
| 2 | Create protocol/utils modules | 1 day |
| 3 | Extract RunnableSequence | 2 days |
| 4 | Extract RunnableParallel | 1 day |
| 5 | Extract RunnableLambda | 2 days |
| 6 | Extract RunnableBinding/Each | 1 day |
| 7 | Update imports and exports | 1 day |
| 8 | Test and fix issues | 2 days |
| 9 | Update documentation | 1 day |

**Total:** 2 weeks

## Backwards Compatibility

**Full backwards compatibility maintained.**

All existing imports continue to work:
```python
from langchain_core.runnables import RunnableSequence  # Works
from langchain_core.runnables.base import RunnableSequence  # Works
```

Internal structure changes are transparent to users.

## Alternatives Considered

### Alternative 1: Leave as-is
**Rejected:** File is too large; maintainability suffers.

### Alternative 2: One class per file
**Rejected:** Too granular; would create 15+ tiny files.

### Alternative 3: Split by functionality (invoke/batch/stream)
**Rejected:** Would separate related code; classes should stay together.

## Open Questions

1. **Naming**: Use `lambda_.py` or `functions.py`?
2. **Protocols**: Separate file or keep in base?
3. **Private modules**: Use `_internal/` directory?
4. **Migration period**: How long to keep old import paths?

## Success Criteria

- [ ] No file exceeds 1,000 LOC
- [ ] All existing imports work
- [ ] Tests pass without modification
- [ ] Documentation updated
- [ ] Type checking passes
- [ ] Import cycle-free

## Rollback Strategy

If issues arise:
1. Revert module split
2. Re-export everything from base.py
3. No user impact (imports unchanged)

## References

- [Python Import System](https://docs.python.org/3/reference/import.html)
- [Circular Imports in Python](https://stackabuse.com/python-circular-imports/)
- Similar refactoring in [pydantic v2](https://github.com/pydantic/pydantic)
