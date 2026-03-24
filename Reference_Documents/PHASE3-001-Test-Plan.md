# 10: Phase 3 - Test Plan & Validation Framework

## Overview

Phase 3 generates comprehensive test cases and validation procedures for the SBA implementation, building on Phase 2's verified findings. This document serves as the master test planning guide.

**Scope**: Tests target 5 verified components + 1 documented gap (spread cap)
**Framework**: CLI dryrun mode + manual validation (no separate test suite)
**Success Criterion**: All unit tests passing, integration tests successful, edge cases handled gracefully

---

## Component Testing Overview

### Testing Architecture

```
Phase 2: Code Verified ✅
    ↓
Phase 3: Unit Tests (Component-isolated)
    ↓
Phase 3: Integration Tests (Multi-component workflows)
    ↓
Phase 3: Edge Case Tests (Extremes, stress, anomalies)
    ↓
Phase 3: Regression Suite (Automated, baseline capture)
    ↓
COMPLETION: All Pass → SBA Implementation Ready
```

### Test Execution Environment

**Tool**: Existing CLI dryrun mode
```bash
python alm/main.py run-server --dryrun True --years 1 --setup alm_setup.xlsm
```

**Features**:
- 1-year projection (fast: ~1 sec per goal-seek iteration)
- All components execute (D&D, goal-seek, rebalancing, spreads)
- Deterministic output (same inputs → same results)
- Full projection comparison (structure identical, just fewer periods)

**Configuration**: `settings_loader.py:236`
```python
def load_run_variables(row: dict, parameter_overrides: dict[str,Any], dryrun: bool):
    # dryrun = True → 1-year projection
    # dryrun = False → Full 101-year projection
```

---

## Testing Layers

### Layer 1: Unit Tests (Component Isolation)

Each component tested independently with:
- Known input data
- Hand-calculated expected output
- Tolerance thresholds
- Pass/fail criteria

**Test Flow**:
```
Input Data → Component Logic → Actual Output
                              ↓
                    Compare to Expected Output
                              ↓
                    Within Tolerance? → PASS/FAIL
```

### Layer 2: Integration Tests (Multi-Component)

Components interact through shared data structures:
```
D&D applied to assets
    ↓
Assets feed to goal-seek
    ↓
Goal-seek determines SBA proportion
    ↓
Rebalancing executes with new KRDs
    ↓
Z-spreads update per period
    ↓
Next period uses updated values
```

**Validation**: Trace data flow through full transaction (6-month to 1-year dryrun)

### Layer 3: Edge Case Tests

Extreme scenarios to verify graceful handling:
- All-bond portfolio (minimal rebalancing)
- All-equity portfolio (maximum rebalancing)
- Extreme market stress (+/-400bps rates)
- Data anomalies (missing spreads, callable bonds at call date)

**Success**: No crashes, reasonable outputs, documented fallbacks

### Layer 4: Regression Suite

Automated tests to detect implementation changes:
- Baseline results captured
- Future runs compared to baseline
- Pass threshold: Results within tolerance of baseline
- Failure investigation guide provided

---

## Tolerance Thresholds

All test validations use numerical tolerances to account for rounding and floating-point precision:

| Category | Tolerance | Rationale |
|----------|-----------|-----------|
| **Duration/DV01** | ±1bp (0.01%) | QuantLib precision |
| **Market Values** | ±0.01% | Rounding in NPV calculations |
| **Z-Spreads** | ±5bps | Calibration convergence |
| **Default Cumulative** | ±0.0001 | Formula precision |
| **Goal-Seek Proportion** | ±0.001 (0.1%) | Iteration convergence tolerance |
| **Test Pass Rate** | 100% (all unit tests) | No failures acceptable |

---

## Test Data Principles

### Data Generation Strategy

**Parametric Approach**: Vary key dimensions to cover behavior spectrum

**Dimensions**:
- Portfolio size: Small (5 assets) → Large (50+ assets)
- Asset mix: Bonds only → Equities only → Mixed
- Maturity ladder: Concentrated → Spread (0-30y)
- Default rates: 0% → 5% → 50% → 100%
- Z-spread curves: Flat → Steep → Inverse
- Liability profile: Short tenor → Long tenor → Laddered

**Reproducibility**: All test data includes seed/parameters for regeneration

### Synthetic Portfolio Templates

**Template 1: Simple Bond Portfolio**
- 3-5 corporate bonds
- Maturities: 5y, 10y, 20y
- Spreads: 100bps, 150bps, 200bps
- Default rates: 1%, 2%, 3%
- Use case: Basic D&D & KRD testing

**Template 2: Mixed Asset Portfolio**
- 60% bonds (5-10 bonds varied maturities)
- 30% equities (stocks, indices)
- 10% cash/derivatives
- Varying spreads and credit quality
- Use case: Full rebalancing workflows

**Template 3: Extreme Maturity Ladder**
- Asset distribution: 5y(10%), 10y(20%), 15y(30%), 20y(30%), 30y(10%)
- Or inverse: 30y(10%), 20y(30%), 15y(30%), 10y(20%), 5y(10%)
- Use case: Rebalancing iteration convergence

**Template 4: Stress Portfolio**
- Assets with high default risk (5%+)
- Long-duration liabilities
- Negative spreads in some assets
- Use case: Edge case & stress scenario testing

---

## Reference to Phase 2 Findings

### Component Status Summary

| Component | Verification | Config | Issues |
|-----------|--------------|--------|--------|
| D&D Formula | ✅ Code verified match | N/A | None |
| Goal-Seek Algorithm | ✅ Code verified match | N/A | None |
| **KRD Liability Method** | ✅ Code verified match | ✅ Risk_Free confirmed | None |
| Rebalancing Iterations | ✅ Code verified match | N/A | None |
| Z-Spread Calibration | ✅ Code verified match | N/A | None |
| **Spread Cap Logic** | 🔴 NOT implemented | 📋 Documented gap | Known limitation |

**Phase 3 Implication**: Tests 1-5 validate verified implementations; Test 6 documents known limitation.

---

## Success Metrics

**Phase 3 Completion Criteria**:

- [ ] All 5 component unit tests PASS
- [ ] Integration test (6-month dryrun) PASS
- [ ] Edge case tests complete without crashes
- [ ] Spread cap limitation formally documented
- [ ] Regression test framework operational
- [ ] Test execution guide provided
- [ ] Phase 3 report generated with pass/fail matrix
- [ ] Baseline results captured for regression detection

**Pass Threshold**: ≥95% tests passing (major gaps require investigation)

---

## Risk Mitigation

### Known Risks in Testing

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Full 101y projection slow | Lengthy test runs | Use dryrun (1y) mode |
| Multi-worker nondeterminism | Flaky tests | Dryrun uses single worker |
| Callable bond test gap | Incomplete validation | Document as spec caveat |
| Spread cap not enforced | Regulatory risk | Document limitation & flag |
| Floating-point precision | False failures | Use tolerances (±1bp, ±0.01%) |

### Mitigation Implementation

1. **Speed**: Dryrun mode 10x faster than full projection
2. **Determinism**: Single-worker dryrun ensures reproducible results
3. **Documentation**: Caveats recorded in test results
4. **Tolerances**: Numerical thresholds prevent false negatives
5. **Baseline**: Regression detection via comparison (not absolute values)

---

## Test Logging & Reporting

### Test Output Format

```
TEST: [Component] [Scenario Name]
INPUT: [Test data parameters]
EXPECTED: [Hand-calculated or spec-defined values]
ACTUAL: [Component output]
TOLERANCE: [±X bps, ±Y%]
RESULT: [PASS / FAIL / CAVEAT]
NOTES: [Any warnings or special conditions]
```

### Failure Investigation Guide

**If test FAILS**:
1. Verify input data correctness (data entry errors?)
2. Check expected value calculation (manual math correct?)
3. Review code change log (did code change recently?)
4. Run baseline comparison (regression vs first-time failure?)
5. Check tolerance appropriateness (should threshold be wider?)
6. Isolate component (is integration issue masking unit issue?)

**If test has CAVEAT**:
- Document known limitation (e.g., callable bonds, spread cap)
- Flag for future fix
- Note in production readiness assessment

---

## Phase 3 Deliverables

### Documentation Files

1. **10-Phase3-Test-Plan.md** (this file)
   - Testing architecture, layers, data strategy
   
2. **11-Component-Unit-Tests.md** (detailed test specs per component)
   - D&D, goal-seek, KRD, rebalancing, z-spreads
   - Test cases, data, expected results
   
3. **12-Integration-Test-Scenarios.md** (multi-component workflows)
   - 6-month dryrun test, full transaction traces
   - Expected flow, validation points, pass criteria
   
4. **13-Edge-Case-Stress-Tests.md** (extremes & anomalies)
   - Portfolio extremes, market stress, data anomalies
   - Expected behavior, graceful failure handling
   
5. **14-Regression-Framework.md** (automated testing)
   - Baseline capture procedures
   - Regression detection logic
   - CI/CD integration guide
   
6. **15-Known-Gaps-Limitations.md** (spread cap & other caveats)
   - Spread cap not implemented (documented gap)
   - Callable bond validation status
   - Future implementation tasks
   
7. **16-Test-Execution-Guide.md** (how-to for implementation team)
   - Step-by-step test running
   - Interpreting results
   - Troubleshooting common issues

### Test Data Package

- Excel template: test_scenarios.xlsx
  - Tabs: Template1 (bonds), Template2 (mixed), Template3 (maturity ladder), Template4 (stress)
- Python script: generate_test_portfolio.py
  - Parametric portfolio generation
  - Seed/reproducibility control
- Expected results spreadsheet: test_baseline_results.xlsx
  - Hand-calculated expected values per scenario
  - Tolerance thresholds
  - Pass/fail criteria

---

## Next Steps

1. ✅ **Document**: Complete 7 Phase 3 reference documents (10-16)
2. ✅ **Template**: Generate test data templates and portfolio specs
3. ✅ **Guide**: Create test execution runbook for implementation team
4. **Review**: User review of documentation (no code changes yet)
5. **Implementation**: Execute tests when documentation approved

---

## Document Map

```
Phase 1: Reference Architecture (00-09) ✅
├── 00: Architecture Overview
├── 01: SBA Goal-Seeking
├── 02: Z-Spread Calibration
├── 03: Data Conversion & SBA Eligibility
├── 04: Rebalancing & Allocation
├── 05: Projection & Valuation
├── 06: Configuration Parameters
├── 07: CLI Execution Framework
├── 08: Defaults & Downgrades
└── 09: Spread Cap & Outputs

Phase 2: Code Verification ✅
├── Code Review Checklist
├── Verification Findings
└── Completion Summary (with decisions)

Phase 3: Test & Validation (10-16) ⏳ IN PROGRESS
├── 10: Test Plan [THIS FILE]
├── 11: Component Unit Tests
├── 12: Integration Test Scenarios
├── 13: Edge Case & Stress Tests
├── 14: Regression Framework
├── 15: Known Gaps & Limitations
└── 16: Test Execution Guide
```
