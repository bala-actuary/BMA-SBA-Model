# 16: Test Execution Guide

Step-by-step runbook for executing Phase 3 tests and interpreting results.

---

## Quick Start (5 minutes)

### Fastest Smoke Test

**Goal**: Quick validation that model runs without crashes

**Steps**:
```bash
# 1. Open terminal
cd c:\BMADev\Anthora_SBA\02. Model

# 2. Run 6-month dryrun (test integration scenario 1)
python alm/main.py run-server \
  --dryrun True \
  --years 1 \
  --setup 01. Setup/alm_setup.xlsm \
  --output test_smoke_check_output.xlsx

# 3. Check output exists
ls -la test_smoke_check_output.xlsx

# Expected: File created, no errors in console
```

**Result**: ✅ PASS if no exceptions; output file generated  
**Result**: ❌ FAIL if crash or output file missing

---

## Complete Test Execution (4-6 hours)

### Prerequisites

**Software Requirements**:
- Python 3.8+ (verify: `python --version`)
- QuantLib 1.25+ (verify: `pip list | grep QuantLib`)
- Pandas, NumPy (already in .pyz)
- Excel reader (openpyxl included)

**File Preparation**:
- [ ] Ensure alm_setup.xlsx exists: `01. Setup/alm_setup.xlsm`
- [ ] Ensure test data ready: Test templates (see section below)
- [ ] Ensure write permissions to `02. Model/` (for output files)
- [ ] Backup any existing `*_output.xlsx` files

**Environment**:
- [ ] Windows or Linux? (Test uses Python, should be OS-agnostic)
- [ ] Python virtual environment activated? (recommended)

---

## Test Data Preparation

### Step 1: Create Test Data Workbooks

**From Section 11**: Component Unit Tests require parametric test data

**Create**: `test_data_component_unit_tests.xlsx`

**Tabs needed**:
1. D&D_Tests (Test 1.1-1.5)
   - Column A: Test_ID (1.1, 1.2, 1.3, 1.4, 1.5)
   - Column B: Annual_Default_Rate (0%, 100%, 5%, varied, varied)
   - Column C: Periods (t=0, t=1, t=5, etc.)
   - Column D: Expected_Cumulative
   - Column E: Hand_Calculated (for verification)

2. GoalSeek_Tests (Test 2.1-2.6)
   - Column A: Test_ID
   - Column B: Scenario (linear convergence, discontinuity, etc.)
   - Column C: Portfolio_Type (bonds, mixed, callable, etc.)
   - Column D: Expected_Iterations
   - Column E: Expected_Converged

3. KRD_Tests (Test 3.1-3.4)
   
4. Rebalancing_Tests (Test 4.1-4.5)

5. ZSpread_Tests (Test 5.1-5.6)

**Templates Available** (See section below for portfolio templates)

### Step 2: Create Test Portfolio Excel Files

**Create**: `test_portfolios.xlsx` (contains all test scenarios)

**Tab 1: Portfolio_Simple_Bonds** (Test 1, 2, basic tests)
```
Asset_ID  Type              Amount($M)  Maturity  Coupon  Spread  Annual_Default
Bond_5y   CorporateBond     20          5y        3.5%    100bps  0.5%
Bond_10y  CorporateBond     30          10y       4.0%    150bps  1.0%
Bond_20y  CorporateBond     15          20y       4.5%    200bps  2.0%
```

**Tab 2: Portfolio_All_Bonds** (Test E1.1)

**Tab 3: Portfolio_All_Equity** (Test E1.2)
```
Equity_1  Equity            40          -          -       -       0.0%
Equity_2  Equity            60          -          -       -       0.0%
```

**Tab 4: Portfolio_Callable** (Test E3.3)
```
Bond_Call CorporateBond     25          10y        4.0%    150bps  1.0%  [callable params]
```

**Tab 5: Portfolio_Extreme_Ladder** (Test E1.4)
```
[Concentrated in specific maturity buckets]
```

**Tab 6: Liability_Standard**
```
Tenor     Cashflow           Duration_Years
1-10y     $10M per year      10.0y (average)
```

**Tab 7: Configuration_Settings**
```
Parameter                            Value
goal_seek_attempts_optimal           15
goal_seek_attempts_absolute          20
goal_seek_tolerance_optimal          0.001%
goal_seek_tolerance_absolute         0.1%
liability_dv01_calibration_method    Risk_Free
swap_rebalancer_enable               True
rebalance_enable                      True
```

### Step 3: Set Up Test Output Directory

```bash
mkdir test_results
mkdir test_results/unit_tests
mkdir test_results/integration_tests
mkdir test_results/edge_cases
```

---

## Executing Unit Tests

### Test Group 1: D&D Formula Tests (15 minutes)

**Files Needed**:
- Input: `test_portfolios.xlsx` (Bond portfolios from Step 2)
- Output: `test_results/unit_tests/test_1_dd_formula_results.xlsx`

**Execution**:

```bash
# Run each test individually (for detailed control and debugging)

# Test 1.1: 0% Default
python alm/main.py run-server \
  --dryrun True \
  --years 1 \
  --portfolio_file test_portfolios.xlsx \
  --scenario_name "Portfolio_Simple_Bonds" \
  --override_default_rates "0%" \
  --output test_results/unit_tests/test_1_1_output.xlsx

# Verify output:
# Expected: No D&D applied (MV unchanged)
# Check: Asset MV in period 1 same as period 0
```

**Repeat** for Tests 1.2-1.5 with different parameters

**Interpretation**:
- Check column: "Market_Value_After_Default"
- For 0% rate: Should equal Market_Value_Before_Default
- For 100% rate: Should equal 0 (or very small)
- For 5% rate: Should be 99.5% of previous (1/(1-0.05)^1)

### Test Group 2: Goal-Seek Tests (30 minutes)

**Execution**:

```bash
# Test 2.1: Linear Convergence
python alm/main.py run-server \
  --dryrun True \
  --years 1 \
  --portfolio_file test_portfolios.xlsx \
  --scenario_name "Portfolio_Simple_Bonds" \
  --goal_seek_attempts_optimal 20 \
  --goal_seek_tolerance_optimal 0.001% \
  --goal_seek_tolerance_absolute 0.1% \
  --output test_results/unit_tests/test_2_1_output.xlsx

# Extract metrics from output:
# - Column: "GoalSeek_Iterations_Total"
# - Column: "GoalSeek_Success_Optimal"
# - Column: "SBA_Proportion"
```

**Interpretation**:
- Chart: iterations vs period (should show stabilization)
- Check: success_optimal = True (or success_absolute if optimal not met)
- Check: Proportion between 0.0 and 1.0
- Check: Value difference (asset_value - liability_value) < tolerance

### Test Group 3-5: Other Components (45 minutes)

**Similar execution for**:
- Test Group 3: KRD Liability Method validation
- Test Group 4: Rebalancing Iterations tracking
- Test Group 5: Z-Spread Mean Reversion tracking

**For each**:
1. Run projection with specific scenario
2. Extract key metrics from output
3. Compare to expected values (from hand calculations)
4. Record PASS/FAIL

---

## Executing Integration Tests

### Integration Test 1: 6-Month Dryrun (30 minutes)

**Goal**: Quick validation of component integration

**Execution**:

```bash
python alm/main.py run-server \
  --dryrun True \
  --years 1 \
  --setup 01. Setup/alm_setup.xlsm \
  --portfolio_file test_portfolios.xlsx \
  --scenario_name "Portfolio_Simple_Bonds" \
  --output test_results/integration_tests/test_1_dryrun_6month.xlsx

# Monitor console output for:
# - "ALM Projection started" → "ALM Projection completed"
# - No exceptions or errors
# - Execution time < 1 minute
```

**Post-Execution Analysis**:

```bash
# Open in Python to extract metrics
python << 'EOF'
import pandas as pd

output = pd.read_excel('test_results/integration_tests/test_1_dryrun_6month.xlsx')

print("=== Integration Test 1 Results ===")
print(f"Total periods: {len(output)}")
print(f"Initial asset value: ${output['Asset_Value'].iloc[0]:.2f}M")
print(f"Final asset value: ${output['Asset_Value'].iloc[-1]:.2f}M")
print(f"Max rebalancing iterations: {output['Rebalancing_Iterations'].max()}")
print(f"Goal-seek convergence: {(output['GoalSeek_Success'] == True).sum()} of {len(output)} succeeded")

# Validation
assert len(output) == 6, "Should have 6 months"
assert output['Asset_Value'].iloc[0] > 0, "Initial AV should be positive"
assert output['Rebalancing_Iterations'].max() <= 3, "Max iterations should be ≤3"
print("✅ All assertions passed")
EOF
```

**Expected Results**:
- ✅ Projection completes (no crash)
- ✅ All components active (D&D applied, goals seek converges, rebalancing executes)
- ✅ Output reasonable (positive values, monotonic trends)

### Integration Test 2: 1-Year Dryrun (1 minute execution + 5 min analysis)

**Execution**: Same as Test 1, but `--years 1` (already set by default)

**Analysis**: Same Python script as above, verify 12 months instead of 6

### Integration Tests 3-8: Follow Same Pattern

- Test 3: Full 101-year projection (5-10 minutes execution)
- Test 4-8: Same pattern, different scenarios

---

## Executing Edge Case Tests

### Edge Test Group 1: Portfolio Extremes (15 minutes)

**Test E1.1: All-Bond Portfolio**

```bash
python alm/main.py run-server \
  --dryrun True \
  --years 1 \
  --portfolio_file test_portfolios.xlsx \
  --scenario_name "Portfolio_All_Bonds" \
  --output test_results/edge_cases/test_e1_1_all_bonds.xlsx

# Expected: No crash, minimal rebalancing
python << 'EOF'
output = pd.read_excel('test_results/edge_cases/test_e1_1_all_bonds.xlsx')
assert output['Rebalancing_Iterations'].sum() <= 6, "All-bond should need little rebalancing"
print("✅ Test E1.1 passed")
EOF
```

**Test E1.2: All-Equity Portfolio**

```bash
python alm/main.py run-server \
  --portfolio_file test_portfolios.xlsx \
  --scenario_name "Portfolio_All_Equity" \
  --output test_results/edge_cases/test_e1_2_all_equity.xlsx

# Expected: Heavy rebalancing
python << 'EOF'
output = pd.read_excel('test_results/edge_cases/test_e1_2_all_equity.xlsx')
assert output['Rebalancing_Iterations'].sum() > 12, "All-equity should require heavy rebalancing"
print("✅ Test E1.2 passed")
EOF
```

**Similarly for E1.3, E1.4**: Execute, compare output to expectations

### Edge Test Group 2: Market Stress (15 minutes)

**Test E2.1: Rates +100bps**

```bash
python alm/main.py run-server \
  --dryrun True \
  --stress_scenario "RatesUp100bps" \
  --output test_results/edge_cases/test_e2_1_rates_up.xlsx

# Expected: Asset values down ~7%, KRDs recalculated
python << 'EOF'
output = pd.read_excel('test_results/edge_cases/test_e2_1_rates_up.xlsx')
mv_change = (output['Asset_Value'].iloc[-1] / output['Asset_Value'].iloc[0]) - 1
assert mv_change < -0.05, f"Rates up should cause MV decline; got {mv_change:.2%}"
print("✅ Test E2.1 passed")
EOF
```

**Similarly for E2.2-E2.5**: Execute with different stress scenarios

### Edge Test Groups 3-6: Continue Same Pattern

- E3.1-E3.5: Data anomalies
- E4.1-E4.4: Numerical extremes
- E5.1-E5.3: Configuration extremes
- E6.1-E6.3: Error handling (simulate failures)

---

## Result Compilation & Reporting

### Step 1: Aggregate All Test Results

```bash
# Create summary spreadsheet
python << 'EOF'
import pandas as pd
import glob

results = []

# Collect unit test results
for file in glob.glob('test_results/unit_tests/*.xlsx'):
    test_id = file.split('_')[-2]  # Extract test ID format
    # Read file and extract metrics
    # Add to results list

# Collect integration test results
for file in glob.glob('test_results/integration_tests/*.xlsx'):
    # Similar collection logic

# Compile into master report
master_results = pd.DataFrame(results)
master_results.to_excel('test_results/MASTER_RESULTS.xlsx', index=False)
print(f"Master results file created: {len(master_results)} tests")
EOF
```

### Step 2: Generate Phase 3 Test Report

**Format**: Markdown document with embedded tables

```markdown
# Phase 3 SBA Validation Test Report
Date: [Today's Date]
Environment: Python 3.9+, QuantLib 1.25, OS: [Windows/Linux]

## Summary
- Unit Tests: 26 total
  - Passed: 26 ✅
  - Failed: 0 ❌
  - Caveats: 0 ⚠️

- Integration Tests: 8 total
  - Passed: 8 ✅
  - Failed: 0 ❌

- Edge Case Tests: 20 total
  - Passed: 19 ✅
  - Failed: 0 ❌
  - Caveats: 1 ⚠️ (Callable bonds - validation incomplete per spec)

## Overall Status
**🟢 PHASE 3 VALIDATION COMPLETE**

All critical components verified. Known limitation (spread cap) documented.
Model ready for [pilot deployment / regulatory audit / production use].

## Findings

### All Passed ✅
[Table with 26+8+19 = 53 passing tests]

### Caveats ⚠️
[Table with 1 caveat: Callable bonds incomplete validation per spec v1.0.0]

### Deferred Issues
[Spread cap not implemented - future enhancement]

## Recommendations
1. ✅ Approve model for [intended use]
2. ⏳ Plan spread cap implementation (Phase 4)
3. 📋 Monitor callable bond performance in production
```

### Step 3: Archive Results

```bash
# Create timestamped archive
mkdir test_results/archive
cp -r test_results/ test_results/archive/test_results_[YYYYMMDD]_[HHMM]

# Save master results
cp test_results/MASTER_RESULTS.xlsx test_results/archive/
```

---

## Interpreting Results

### Test Status Meanings

| Status | Meaning | Action |
|--------|---------|--------|
| ✅ PASS | Output within tolerance of expected | Document result, proceed |
| ❌ FAIL | Output outside tolerance | Investigate root cause, fix code, re-test |
| ⚠️ CAVEAT | Feature partially implemented | Document caveat, plan future work |
| ❓ INCONCLUSIVE | Output reasonable but not validated | Flag for manual review |

### Common Failure Scenarios & Troubleshooting

#### Failure: Goal-Seek Unconverged (success = False)

**Symptoms**:
```
Test 2.1 result: success_optimal = False, success_absolute = False
Iterations = 20 (hit limit)
```

**Likely Causes**:
1. Portfolio is degenerate (all assets perfectly correlated)
2. Tolerance too tight (impossible to satisfy)
3. No feasible solution exists (infeasible constraints)

**Investigation Steps**:
1. Check portfolio data for duplicates/correlations
2. Relax tolerance: Try tolerance_absolute = 1.0% (very loose)
3. Try different portfolio (simple_bonds instead of extreme_ladder)

**Next Steps**:
- If simple portfolio works: Issue is portfolio-specific, document
- If still fails: Bug in goal-seek algorithm, escalate to developer

#### Failure: NaN or Inf Values

**Symptoms**:
```
Output contains NaN or Infinity values in columns: Market_Value, KRD, DV01
```

**Likely Causes**:
1. Numerical instability (very large/small numbers)
2. Division by zero (zero duration asset, zero MV)
3. Invalid input data (negative prices, invalid spreads)

**Investigation Steps**:
1. Check input data for anomalies (negatives, zeros, outliers)
2. Reduce portfolio complexity (fewer assets, shorter duration)
3. Run on smaller time horizon (fewer periods)

**Next Steps**:
- Fix input data and re-test
- If still fails: Numerical robustness issue, escalate

#### Failure: Execution Timeout or Crash

**Symptoms**:
```
python process hangs or terminates unexpectedly
No output file generated
```

**Likely Causes**:
1. Infinite loop in algorithm
2. Memory exhaustion (very large portfolio or long projection)
3. Unhandled exception raised

**Investigation Steps**:
1. Run with smaller portfolio (5 vs 50 assets)
2. Run with dryrun mode (1 year vs 101 years)
3. Check console output for error messages

**Next Steps**:
- If works on smaller case: Scalability issue, may need optimization
- If still fails: Bug in core algorithm, escalate

---

## Post-Test Checklist

Before declaring Phase 3 complete:

- [ ] All 26 unit tests executed and PASS
- [ ] All 8 integration tests executed and PASS
- [ ] All 20 edge case tests executed (result: PASS or documented caveat)
- [ ] Master results spreadsheet created
- [ ] Phase 3 test report generated (markdown or PDF)
- [ ] Baseline results captured for regression testing
- [ ] Results reviewed by [QA/Risk Manager/Technical Lead]
- [ ] Sign-off obtained: "Phase 3 validation approved"
- [ ] Results archived with timestamp
- [ ] Deferred items documented (e.g., spread cap implementation)

---

## Sign-Off Template

```
PHASE 3 SBA VALIDATION TEST EXECUTION
================================================

Test Execution Date: [Date]
Executed By: [Name]
Environment: Python [version], QuantLib [version], OS [OS]

RESULTS SUMMARY:
- Unit Tests (26): ✅ 26 PASS
- Integration Tests (8): ✅ 8 PASS
- Edge Case Tests (20): ✅ 19 PASS, 1 ⚠️ CAVEAT

OVERALL ASSESSMENT: 
🟢 PHASE 3 VALIDATION COMPLETE - APPROVED FOR DELIVERY

Known Limitations:
1. Spread cap not implemented (future enhancement)
2. Callable bond validation incomplete per spec (production monitoring)

Recommendations:
- ✅ Model approved for [intended use]
- 📋 Plan/Q2 2024: Spread cap implementation
- 📋 Q1 2024: Callable bond production validation

Sign-Off:
Tester: _________________________ Date: _______

Reviewer: _______________________ Date: _______

Approved by: ____________________ Date: _______
```

---

## Quick Reference: Command Cheat Sheet

```bash
# Smoke test (fastest validation)
python alm/main.py run-server --dryrun True --years 1 --output test_output.xlsx

# Unit test template (component-specific)
python alm/main.py run-server --portfolio_file test_portfolios.xlsx --scenario_name "Portfolio_Simple_Bonds" --output test_unit_output.xlsx

# Integration test (full pipeline)
python alm/main.py run-server --dryrun True --years 1 --setup alm_setup.xlsm --output test_integration_output.xlsx

# Full validation (all years, stress scenarios)
python alm/main.py run-server --years 101 --stress_enable True --output test_full_validation.xlsx

# Extract metrics from output
python << 'EOF'
import pandas as pd
df = pd.read_excel('test_output.xlsx')
print(df[['Period', 'Asset_Value', 'Goal_Seek_Iterations', 'Rebalancing_Iterations']].head(10))
EOF
```

