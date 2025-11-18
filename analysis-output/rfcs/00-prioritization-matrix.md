# RFC Prioritization Matrix

> **Analysis based on commit:** [`990e346`](https://github.com/langchain-ai/langchain/tree/990e346c46251246966b562c8d6839903cb205ad)

## Overview

This document prioritizes the RFCs based on impact vs. effort, helping teams decide what to tackle first.

## Prioritization Framework

| Priority | Effort | Impact | Timeline |
|----------|--------|--------|----------|
| **Quick Wins** | <1 week | Immediate value | Do first |
| **Strategic** | 2-4 weeks | Significant impact | Plan for next quarter |
| **Long-term** | >1 month | Architectural | Roadmap for future |

## Impact vs. Effort Grid

```
High Impact │
            │  ┌─────────────┐   ┌─────────────┐
            │  │   RFC-0004  │   │   RFC-0007  │
            │  │  Split Base │   │ Async-First │
            │  └─────────────┘   └─────────────┘
            │
            │  ┌─────────────┐   ┌─────────────┐
            │  │   RFC-0005  │   │   RFC-0006  │
            │  │Property Test│   │Error Improve│
            │  └─────────────┘   └─────────────┘
            │
            │  ┌─────────────┐   ┌─────────────┐
            │  │   RFC-0001  │   │   RFC-0002  │
            │  │  McCabe CI  │   │ Benchmarks  │
            │  └─────────────┘   └─────────────┘
            │
 Low Impact │  ┌─────────────┐
            │  │   RFC-0003  │
            │  │  Doc Tenacity│
            │  └─────────────┘
            └────────────────────────────────────────
               Low Effort                High Effort
```

## RFC Summary Table

| RFC | Title | Category | Effort | Impact | Priority |
|-----|-------|----------|--------|--------|----------|
| 0001 | Enable McCabe Complexity Checking | Code Quality | 2 days | Medium | Quick Win |
| 0002 | Add Performance Benchmarks to CI | Performance | 4 days | Medium | Quick Win |
| 0003 | Document Dependency Exclusions | Documentation | 1 day | Low | Quick Win |
| 0004 | Split runnables/base.py | Architecture | 2 weeks | High | Strategic |
| 0005 | Add Property-Based Testing | Testing | 2 weeks | High | Strategic |
| 0006 | Standardize Error Messages | Developer Experience | 3 weeks | High | Strategic |
| 0007 | Async-First Core Redesign | Architecture | 6+ weeks | Very High | Long-term |

## Recommended Execution Order

### Phase 1: Quick Wins (Next Sprint)

1. **RFC-0001**: Enable McCabe complexity - immediate code quality improvement
2. **RFC-0003**: Document tenacity exclusion - low effort, removes tech debt
3. **RFC-0002**: Add benchmarks - establishes performance baseline

**Total effort**: ~7 dev-days
**Expected outcomes**: Better code quality metrics, performance visibility

### Phase 2: Strategic (Next Quarter)

4. **RFC-0006**: Error messages - improves debugging experience
5. **RFC-0005**: Property testing - catches edge cases
6. **RFC-0004**: Split base.py - improves maintainability

**Total effort**: ~7 weeks
**Expected outcomes**: Better developer experience, more maintainable code

### Phase 3: Long-term (Future Roadmap)

7. **RFC-0007**: Async-first redesign - fundamental performance improvement

**Total effort**: 6+ weeks
**Expected outcomes**: Better async performance, reduced overhead

## Dependencies Between RFCs

```
RFC-0001 (McCabe) ──────┐
                        │
RFC-0002 (Benchmarks) ──┼──► RFC-0004 (Split Base)
                        │           │
RFC-0003 (Doc) ─────────┘           │
                                    ▼
RFC-0005 (Property Test) ──────► RFC-0007 (Async-First)
                                    ▲
RFC-0006 (Errors) ──────────────────┘
```

## Resource Requirements

| RFC | Skills Needed | Stakeholders |
|-----|---------------|--------------|
| 0001 | Python, CI/CD | Core maintainers |
| 0002 | Python, Benchmarking | Performance team |
| 0003 | Technical writing | Documentation team |
| 0004 | Architecture, Python | Core maintainers |
| 0005 | Testing, Hypothesis | QA team |
| 0006 | Python, API design | Core maintainers |
| 0007 | Async Python, Architecture | Core maintainers, Performance team |

## Success Metrics

| RFC | Metric | Target |
|-----|--------|--------|
| 0001 | Files exceeding complexity 10 | 0 |
| 0002 | Benchmark variance | <10% |
| 0003 | Doc coverage for exclusions | 100% |
| 0004 | Max file size | <2000 LOC |
| 0005 | Edge case coverage | +30% |
| 0006 | Error message consistency | >95% |
| 0007 | Async overhead reduction | 50% |

## Risk Assessment

| RFC | Risk Level | Key Risks | Mitigation |
|-----|------------|-----------|------------|
| 0001 | Low | False positives | Tune thresholds |
| 0002 | Low | Flaky benchmarks | Statistical methods |
| 0003 | Very Low | None significant | - |
| 0004 | Medium | Breaking imports | Careful aliasing |
| 0005 | Low | Test maintenance | Clear guidelines |
| 0006 | Medium | Breaking changes | Gradual migration |
| 0007 | High | Performance regression | Extensive benchmarks |

## Next Steps

1. **Review RFCs**: Gather feedback from maintainers
2. **Prioritize**: Confirm prioritization with team
3. **Assign**: Allocate resources for Phase 1
4. **Track**: Set up project board for progress
5. **Iterate**: Adjust based on learnings
