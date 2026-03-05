# Skill Benchmark: clevertap-assist

**Model**: claude-opus-4-6
**Date**: 2026-03-04
**Evals**: 1 (react-integration), 2 (ios-debug), 3 (android-audit) — 1 run each per configuration

## Summary

| Metric | With Skill | Without Skill | Delta |
|--------|------------|---------------|-------|
| Pass Rate | 100% ± 0% | 83% ± 3% | +0.17 |
| Time | 127.6s ± 35.0s | 93.6s ± 23.7s | +34.0s |
| Tokens | 47,633 ± 3,503 | 32,926 ± 1,069 | +14,707 |

## Per-Eval Breakdown

| Eval | With Skill | Without Skill | Delta |
|------|------------|---------------|-------|
| 1. react-integration | 6/6 (100%) | 5/6 (83%) | +1 |
| 2. ios-debug | 5/5 (100%) | 4/5 (80%) | +1 |
| 3. android-audit | 7/7 (100%) | 6/7 (86%) | +1 |

## Failed Assertions (Without Skill)

All 3 baseline failures are the same category — **loading curated reference files**:

1. **react-integration**: "References or loads core/integration/web.md" — baseline has no access to curated reference
2. **ios-debug**: "Loads common debugging reference first" — baseline has no access to debugging checklist
3. **android-audit**: "Loads core/audit/features.md" — baseline has no access to audit reference

## Analysis

### What the skill adds
- **Structured mode detection**: Explicitly identifies Integration/Debug/Audit mode from the prompt
- **Reference file loading**: Loads curated, verified SDK knowledge from core/ files
- **Workflow discipline**: Follows prescribed steps (detect platform → load reference → execute → verify)
- **Audit templates**: Provides comprehensive feature-area coverage with exact regex patterns

### What the baseline already knows
- CleverTap SDK APIs (npm install, useEffect pattern, onUserLogin, event.push, etc.)
- iOS debugging (viewDidLoad vs AppDelegate init placement)
- Android audit patterns (grep commands, per-flow analysis, push checklist)
- The model has strong domain knowledge of CleverTap SDKs from training data

### Trade-offs
- **+17% pass rate** at the cost of **+45% tokens** and **+36% time**
- The token/time overhead comes from reading reference files during the skill workflow
- Baseline is surprisingly strong — model already knows CleverTap well
- Skill's main value: **consistency and completeness** rather than raw knowledge

## Recommendations for Iteration 2

1. **Add assertions that test completeness** — baseline may miss edge cases the reference files cover
2. **Test on less common platforms** (Flutter, React Native) where model knowledge may be weaker
3. **Run multiple iterations** per eval to measure variance
4. **Add harder eval cases** — unsupported platform detection, monorepo ambiguity
