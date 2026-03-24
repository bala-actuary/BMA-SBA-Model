# Plan: Phase 3 - SBA Validation & Test Case Generation

## Goal
Generate comprehensive test cases and validation procedures to verify SBA implementation correctness against Specification v1.0.0, building on completed Phase 1 & 2.

## Executive Summary
**Phase 1 & 2 Status**: ✅ COMPLETE
- 9 reference documents created (00-09)
- 5 of 6 components verified spec-compliant
- 1 known gap (spread cap) documented
- 1 critical config confirmed (KRD = Risk_Free)

**Phase 3 Objectives**:
1. Generate unit test cases for each component (D&D, goal-seek, KRD, spreads, rebalancing)
2. Create edge case scenarios and validation checks
3. Document regression test procedures
4. Generate test data for each scenario
5. Create verification checklist for implementation team

## Phase 3: Steps

### Step 1: Test Infrastructure Setup
- Identify test framework: pytest or custom Excel/CLI validation
- **Tool Available**: Dryrun mode enables 1-year minimal projection for fast validation
- **Output**: Test infrastructure documented

### Step 2: Component Unit Tests (Sequential)
#### 2.1 D&D Formula Regression Tests 
- **Spec**: Section 3.3.8.11, Formula: cumulative = 1 - (1 - X_annual)^t
- **Test Cases**:
  - Boundary: 0% and 100% annual default rates
  - Accumulation: Verify cumulative growth over 101-year horizon (expected: approach 100%)
  - Scalar application: Confirm MV and CF both reduced by same scalar
  - Precision: Verify numerical stability (no NaN/Inf)
- **Test Data**: Generate parametric test suite for default rates [0%, 5%, 25%, 50%, 100%]
- **Validation**: Compare code output to hand-calculated expected values
- **Reference Doc**: [08-Defaults-Downgrades.md](/memories/session/08-Defaults-Downgrades.md)

#### 2.2 Goal-Seek Algorithm Tests
- **Spec**: Section 3.3.7, Hybrid bisection/linear with discontinuity detection
- **Test Cases**:
  - Linear phase: Verify ±15% clamping works with 30% relative allowance
  - Discontinuity trigger: Confirm switches to bisection after 8 attempts or bracket found
  - Convergence cascade: Verify optimal → absolute → hard stop progression
  - No solution: Test with impossible constraints (verify max attempts reached)
  - Multiple optima: Test flat SBA curve (any proportion works)
- **Test Data**: Create 5 test scenarios spanning easy→hard goal-seek curves
- **Validation**: Confirm convergence metrics (success_optimal vs success_absolute) correct
- **Reference Doc**: [01-SBA-Goal-Seeking.md](/memories/session/01-SBA-Goal-Seeking.md)

#### 2.3 Liability KRD Calculation Tests
- **Spec**: Section 2.3.5.3, Risk-free rates ONLY (zspread_t_dv01 = 0)
- **Test Cases**:
  - Rate shock detection: Verify stress impacts calculated per key rate
  - Risk-free vs +spread: Compare KRDs from both methods (should differ by spread amortization amount)
  - Swap targeting: Verify asset KRDs match liability KRDs in magnitude (both risk-free basis)
  - Edge case: Test negative rates environment (QuantLib behavior)
- **Test Data**: Generate scenarios with varying liability cashflow profiles
- **Validation**: Hand-calculate DV01 using QuantLib standalone; compare to model output
- **Reference Doc**: [04-Rebalancing-Allocation.md](/memories/session/04-Rebalancing-Allocation.md#L114)

#### 2.4 Swap Rebalancing Iteration Tests
- **Spec**: Section 3.3.6/3.3.9.2, Up to 3 iterations per period
- **Test Cases**:
  - Single iteration: Verify swap rebalancer runs, exits if no further hedge needed
  - Triple iteration: Confirm all 3 iterations execute when needed
  - Convergence: Verify DV01 mismatch reduces with each iteration
  - Ordering: Confirm swap rebalancer runs first, physical second (margin call impact)
  - Early exit: Test scenarios where fewer than 3 iterations needed
- **Test Data**: Create portfolio requiring 0, 1, 2, 3 iterations
- **Validation**: Trace iteration count and DV01 progression through projection
- **Reference Doc**: [09-Spread-Cap-and-Outputs.md](/memories/session/09-Spread-Cap-and-Outputs.md)

#### 2.5 Z-Spread Calibration & Mean Reversion Tests
- **Spec**: Section 3.3.8.8-9, Three methods + mean reversion formula: L + (S-L)×ρ^t
- **Test Cases**: 
  - Calibration methods: Test all 3 methods produce reasonable z-spreads
  - Mean reversion: Verify spreads converge toward TTC at t=∞
  - Shock response: Confirm PiT/TTC changes propagate correctly
  - Boundary: Test with flat z-spread curves (no term structure)
- **Test Data**: Generate scenarios with varying z-spread term structures
- **Validation**: Compare mean reversion decay rate to parameter ρ manually
- **Reference Doc**: [02-ZSpread-Calibration.md](/memories/session/02-ZSpread-Calibration.md)

### Step 3: Integration Tests (6-month to full projection)
- **Test cross-component interactions**:
  - D&D formula applied to assets → Goal-seek finds proportion → Rebalancing executes
  - Z-spread changes → KRD updates → Swap rebalancer adjusts hedge
  - Margin calls → Physical rebalancer adjusts → Cash updated
- **Dryrun Mode**: Run 1-year projection (fast) instead of full 101-year to validate logic
- **Comparison**: Compare dryrun vs full projection structure (should be identical, just fewer periods)

### Step 4: Edge Case & Stress Scenario Tests
- **Portfolio extremes**:
  - All-bond portfolio (no rebalancing needed)
  - All-equity portfolio (maximum rebalancing)
  - Single-asset-class mix
  - Extreme maturity ladder (very short → very long)
- **Market stress**:
  - Parallel rate shift (+/- 100bps)
  - Steepening/flattening curve moves
  - Credit spread widening (+500bps)
  - Extreme rates (negative rates, +400bps)
- **Data anomalies**:
  - Missing z-spreads (use fallback method)
  - Callable bond at call date or past call date
  - Assets with 0 market value
  - Liabilities with 0 remaining duration

### Step 5: Known Limitation Validation (Spread Cap Gap)
- **Current State**: Spec requires 35bps cap enforcement; code parameter exists but NOT used
- **Test Case**:
  - Scenario with uncapped spread > 35bps
  - Verify: Current behavior allows uncapped spread (documents limitation)
  - Expected: Spread cap enforcement NOT applied (known gap)
  - Documentation: Record finding in validation report
- **Future**: If cap implemented later, add test for cap application and BEL recalculation

### Step 6: Regression Test Suite & Documentation
- **Create test framework document**:
  - How to run unit tests (pytest or CLI validation)
  - Test data generation templates
  - Expected results thresholds (tolerance levels)
  - Failure investigation guide
- **Automated validation checklist**:
  - Pre-run: Verify configuration, data integrity
  - During: Monitor iteration counts, convergence metrics
  - Post-run: Check output consistency, numerical stability
  - Report: Generate pass/fail summary
- **Continuous integration readiness**:
  - Version control test cases
  - Document baseline results (regression detection)
  - Specify environment (Python version, QuantLib version, OS)

## Relevant Files

### Test Infrastructure
- `settings_loader.py:236` - Dryrun mode parameter
- `main_server.py` - CLI execution, parameter passing
- `balance_sheet_projection.py` - Main projection loop (test target)

### Test Reference Data
- `alm_setup.xlsm` - Parameter templates for test scenarios
- Reference documents 00-09 - Component specifications

### Code Locations for Test Hooks
- `asset_projector.py:161-190` - D&D formula
- `sba_goal_seeker.py:200+` - Goal-seek algorithm
- `liability_cashflow.py:92-97` - KRD calculation
- `balance_sheet_projection.py:160-195` - Rebalancing iterations
- `zspread_provider.py` - Z-spread projection
- `settings_loader.py:76` - Spread cap parameter (unused)

## Verification

**Unit tests** (Step 2):
- Each component isolated with hand-calculated expected values
- Test data provided in code comments for reproducibility
- Pass/fail criteria explicitly stated

**Integration tests** (Step 3):
- Multi-component workflows validated end-to-end
- Dryrun mode enables fast validation
- Full projection periodically validated for regression

**Edge case tests** (Step 4):
- Extreme portfolios, market stresses, data anomalies
- Verify graceful handling (no crashes, reasonable outputs)
- Document any warnings or fallback behaviors

**Regression suite** (Step 6):
- Automated test runner with pass/fail reporting
- Baseline results captured (future change detection)
- Integration with CI/CD pipeline

## Decisions

- **Test Framework**: Use existing CLI dryrun mode + manual verification (no separate test suite needed immediately)
- **Test Scope**: Focus on 5 verified components + 1 documented gap
- **Test Data**: Generate synthetic portfolios covering spectrum of behaviors
- **Success Criteria**: All unit/integration tests passing, edge cases handled gracefully, gap documented
- **Spread Cap**: Known gap documented in test results, future implementation tracked separately

## Known Limitations & Caveats

1. **Spread Cap NOT Implemented**: Parameter exists but unused; regulatory test documents this limitation
2. **Callable Bonds**: Spec notes "dynamic call option not fully validated" — test data includes but with caveat
3. **Multi-processing**: 8 workers used; tests should be deterministic (same scenario → same results)
4. **Performance**: Full 101-year projection takes ~5-10 sec per goal-seek iteration; dryrun mode 1sec/iteration

## Success Criteria for Phase 3 Completion

✅ All 5 verified components have unit tests with passing results
✅ Integration tests on 6-month projection pass
✅ Edge case tests complete without crashes
✅ Regression test framework documented
✅ Spread cap limitation formally documented with caveat
✅ Test execution guide provided to implementation team
✅ Phase 3 report generated with test results matrix
