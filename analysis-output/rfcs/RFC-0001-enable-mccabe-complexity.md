# RFC-0001: Enable McCabe Complexity Checking

**Status:** Draft
**Author:** Codebase Analysis
**Created:** 2025-11-18
**Effort:** 2 dev-days
**Category:** Quick Win

## Summary

Enable McCabe cyclomatic complexity checking in the ruff linter configuration to identify and refactor overly complex functions, improving code maintainability and reducing bug density.

## Motivation

Currently, LangChain's ruff configuration explicitly ignores McCabe complexity rules (`C90`):

```toml
# From: libs/core/pyproject.toml
[tool.ruff.lint]
select = ["ALL"]
ignore = [
    "C90",  # McCabe complexity ← Currently ignored
    # ...
]
```

This means:
- Complex functions go undetected
- Code maintainability degrades over time
- Bug density increases in complex code
- Onboarding new contributors is harder

The codebase has files with 6,000+ LOC (e.g., `runnables/base.py`), suggesting complexity hotspots exist.

## Detailed Design

### Step 1: Enable McCabe Rules

```toml
# Update libs/core/pyproject.toml
[tool.ruff.lint]
select = ["ALL"]
ignore = [
    # Remove "C90" from ignore list
    "COM812",
    "ISC001",
    # ... rest unchanged
]

# Add complexity threshold
[tool.ruff.lint.mccabe]
max-complexity = 10  # Industry standard
```

### Step 2: Run Analysis

```bash
cd libs/core
uv run --group lint ruff check --select C90
```

### Step 3: Triage Results

Categorize violations:
- **Refactor now**: Complexity > 15
- **Refactor soon**: Complexity 10-15
- **Monitor**: Complexity < 10 but flagged

### Step 4: Refactor High-Complexity Functions

For each high-complexity function:

```python
# Before: Complex function with many branches
def process(input, options):
    if options.type == "A":
        if options.subtype == "A1":
            # ... 20 lines
        elif options.subtype == "A2":
            # ... 20 lines
    elif options.type == "B":
        # ... more branches
    # Complexity: 15+

# After: Extracted into focused functions
def process(input, options):
    handlers = {
        "A": _process_type_a,
        "B": _process_type_b,
    }
    return handlers[options.type](input, options)

def _process_type_a(input, options):
    handlers = {
        "A1": _process_a1,
        "A2": _process_a2,
    }
    return handlers[options.subtype](input, options)
```

### Step 5: Add to CI

```yaml
# .github/workflows/lint.yml
- name: Check complexity
  run: ruff check --select C90 --exit-non-zero-on-fix
```

## Example Usage

### Before

```python
def invoke(self, input, config=None, **kwargs):
    # Complex method with 15+ branches
    if config is None:
        config = {}
    if "callbacks" in config:
        if isinstance(config["callbacks"], list):
            # ...
        elif isinstance(config["callbacks"], CallbackManager):
            # ...
    if "tags" in config:
        # ...
    # Many more conditions
```

### After

```python
def invoke(self, input, config=None, **kwargs):
    config = self._prepare_config(config)
    callbacks = self._setup_callbacks(config)
    return self._execute(input, config, callbacks, **kwargs)

def _prepare_config(self, config):
    return config if config else {}

def _setup_callbacks(self, config):
    callbacks = config.get("callbacks")
    if isinstance(callbacks, list):
        return CallbackManager(callbacks)
    return callbacks
```

## Implementation Plan

| Phase | Task | Duration |
|-------|------|----------|
| 1 | Enable rule in config | 1 hour |
| 2 | Run analysis, triage results | 2 hours |
| 3 | Refactor critical violations | 1 day |
| 4 | Add to CI pipeline | 1 hour |
| 5 | Document guidelines | 2 hours |

**Total:** 2 dev-days

## Backwards Compatibility

No impact on public APIs. This is purely internal code quality improvement.

## Alternatives Considered

### Alternative 1: Use pylint instead
**Rejected:** Already using ruff; no need for additional tool.

### Alternative 2: Higher threshold (15+)
**Rejected:** Industry standard is 10; being strict is better for maintainability.

### Alternative 3: Per-file exemptions
**Considered:** May be needed for some legacy code, but should be explicit.

## Open Questions

1. **Threshold value**: Should we start with 15 and reduce to 10?
2. **Exemptions**: How to handle necessary complex functions (e.g., parsers)?
3. **Migration period**: Allow temporary `# noqa: C901` comments?

## Success Criteria

- [ ] Zero functions with complexity > 15
- [ ] <10 functions with complexity 10-15 (documented exemptions)
- [ ] CI blocks new high-complexity code
- [ ] Guidelines documented in CLAUDE.md

## References

- [McCabe Complexity](https://en.wikipedia.org/wiki/Cyclomatic_complexity)
- [Ruff C90 Rules](https://docs.astral.sh/ruff/rules/#mccabe-c90)
- Research: "Cyclomatic complexity is a strong predictor of bug density" (Landman et al., 2016)
