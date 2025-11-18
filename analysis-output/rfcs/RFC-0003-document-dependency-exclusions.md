# RFC-0003: Document Dependency Exclusions

**Status:** Draft
**Author:** Codebase Analysis
**Created:** 2025-11-18
**Effort:** 1 dev-day
**Category:** Quick Win

## Summary

Add inline documentation explaining why specific dependency versions are excluded (e.g., `tenacity!=8.4.0`), reducing confusion and maintaining security awareness.

## Motivation

The core package excludes specific dependency versions without explanation:

```toml
# From: libs/core/pyproject.toml
dependencies = [
    "tenacity!=8.4.0,>=8.1.0,<10.0.0",  # Why 8.4.0 excluded?
]
```

This causes:
- Confusion for contributors
- Lost institutional knowledge
- Security blind spots
- Potential accidental removal during updates

## Detailed Design

### Add Inline Comments

```toml
# libs/core/pyproject.toml
dependencies = [
    # LangSmith observability integration
    "langsmith>=0.3.45,<1.0.0",

    # Retry logic for API resilience
    # SECURITY: 8.4.0 has infinite loop bug (https://github.com/jd/tenacity/issues/471)
    "tenacity!=8.4.0,>=8.1.0,<10.0.0",

    # JSON patching for serialization diffs
    "jsonpatch>=1.33.0,<2.0.0",

    # YAML configuration parsing
    "PyYAML>=5.3.0,<7.0.0",

    # Data validation and settings management
    "pydantic>=2.7.4,<3.0.0",

    # Extended type hints for Python <3.11
    "typing-extensions>=4.7.0,<5.0.0",

    # Version parsing utilities
    "packaging>=23.2.0,<26.0.0",
]
```

### Create Dependency Documentation

```markdown
<!-- docs/dependencies.md -->
# Dependency Documentation

## Version Exclusions

| Package | Excluded | Reason | Reference |
|---------|----------|--------|-----------|
| tenacity | 8.4.0 | Infinite loop in retry logic | [Issue #471](https://github.com/jd/tenacity/issues/471) |

## Version Constraints

| Package | Constraint | Reason |
|---------|------------|--------|
| pydantic | >=2.7.4 | Required for model_fields API |
| langsmith | <1.0.0 | Pre-1.0 API may change |
```

### Add CI Check

```yaml
# .github/workflows/deps.yml
- name: Check dependency documentation
  run: |
    # Verify all exclusions are documented
    python scripts/check_dep_docs.py
```

## Example Usage

### Before
```toml
"tenacity!=8.4.0,>=8.1.0,<10.0.0",
```

**Problem**: Why is 8.4.0 excluded? Is it still needed?

### After
```toml
# Retry logic for resilient API calls
# SECURITY: 8.4.0 excluded due to infinite loop bug
# Reference: https://github.com/jd/tenacity/issues/471
# Last reviewed: 2025-01-15
"tenacity!=8.4.0,>=8.1.0,<10.0.0",
```

**Clear**: Anyone can understand and verify the exclusion.

## Implementation Plan

| Phase | Task | Duration |
|-------|------|----------|
| 1 | Research all exclusions | 2 hours |
| 2 | Add inline comments | 1 hour |
| 3 | Create documentation | 2 hours |
| 4 | Add CI check | 1 hour |

**Total:** 1 dev-day

## Backwards Compatibility

No impact. Documentation only.

## Alternatives Considered

### Alternative 1: External security database
**Rejected:** Over-engineered for current needs.

### Alternative 2: No documentation
**Rejected:** Creates maintenance burden and security risk.

## Open Questions

1. **Review schedule**: How often to review exclusions?
2. **Format**: TOML comments or separate file?
3. **Automation**: Should we auto-check if exclusions are still needed?

## Success Criteria

- [ ] All version exclusions documented with reason
- [ ] References to original issues included
- [ ] Review dates tracked
- [ ] CI validates documentation exists

## References

- [tenacity #471](https://github.com/jd/tenacity/issues/471)
- [PEP 508 - Dependency Specification](https://peps.python.org/pep-0508/)
