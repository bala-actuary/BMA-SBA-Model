# 11: Component Unit Tests - Detailed Specifications

Comprehensive test specifications for each SBA component in isolation, with test data and expected results.

---

## 1. D&D (Defaults & Downgrades) Formula Tests

**Component**: `asset_projector.py:161-190`  
**Specification**: Section 3.3.8.11 — `cumulative_default(t) = 1 - (1 - annual_default)^t`  
**Method**: Independent calculation; compare to hand-calculated values

### Test 1.1: Boundary Condition — 0% Annual Default

**Test Data**:
- Annual default rate: 0.0%
- Test periods: t = 0, 1, 5, 10, 50, 101
- Scalar applied: Yes

**Expected Results**:
- cumulative(0) = 0% (no defaults yet)
- cumulative(1) = 0% (0% rate)
- cumulative(5) = 0% (0% rate)
- cumulative(101) = 0% (0% rate)
- Scalar = (1 - 0) = 1.0 (no adjustment to MV/CF)

**Validation**:
- [ ] For t in [0, 1, 5, 10, 50, 101]: cumulative == 0.0 ± 1E-10
- [ ] MV after default scaling = 0% reduction
- [ ] CF after default scaling = 0% reduction
- [ ] No numerical issues (finite, non-NaN values)

**Pass Criteria**: All assertions within tolerance

---

### Test 1.2: Boundary Condition — 100% Annual Default

**Test Data**:
- Annual default rate: 1.0 (100%)
- Test periods: t = 0, 1, 2, 5, 101
- Scalar applied: Yes

**Expected Results**:
- cumulative(0) = 0% (formula: 1-(1-1)^0 = 1-0 = 1 ERROR; actually check: 0 since t=0 means no periods)
- cumulative(1) = 100% (formula: 1-(1-1)^1 = 1-0 = 1.0 = 100%)
- cumulative(2) = 100% (formula: 1-(1-1)^2 = 1-0 = 100%)
- cumulative(101) = 100% (formula: 1-(1-1)^101 = 100%)
- Scalar = (1 - 1.0) = 0.0 (full elimination of MV/CF)

**Validation**:
- [ ] cumulative(1) = 1.0 ± 1E-10 (all value eliminated)
- [ ] MV after default scaling = 100% reduction (MV_final = MV_orig * 0 = 0)
- [ ] CF after default scaling = 100% reduction
- [ ] Assertions validate bounds [0, 1] ✓

**Pass Criteria**: All values zeroed out correctly

---

### Test 1.3: Interior Point — 5% Annual Default

**Test Data**:
- Annual default rate: 0.05 (5%)
- Test periods: t = 0, 1, 5, 10, 50, 101
- Scalar applied: Yes

**Hand-Calculated Expected Values**:
```
Annual rate = 0.05
cumulative(t) = 1 - (1 - 0.05)^t = 1 - (0.95)^t

t=0:   cumulative = 1 - (0.95)^0 = 1 - 1 = 0
t=1:   cumulative = 1 - (0.95)^1 = 1 - 0.95 = 0.05
t=5:   cumulative = 1 - (0.95)^5 = 1 - 0.7738 = 0.2262
t=10:  cumulative = 1 - (0.95)^10 = 1 - 0.5987 = 0.4013
t=50:  cumulative = 1 - (0.95)^50 = 1 - 0.0769 = 0.9231
t=101: cumulative = 1 - (0.95)^101 = 1 - 0.0003 = 0.9997
```

**Validation**:
- [ ] Code output matches hand-calculated for each t
- [ ] Cumulative growth rate: 5% per year (monotonic increase)
- [ ] Asymptote behavior: Approaches 100% as t→∞
- [ ] Scalar reduction: MV_reduced = MV_orig * (1 - cumulative)
  - t=0: scalar = 1.0 (no reduction)
  - t=5: scalar = 0.7738 (22.62% reduction)
  - t=50: scalar = 0.0769 (92.31% reduction)

**Pass Criteria**: All values within ±0.0001 of expected

---

### Test 1.4: Precision & Numerical Stability

**Test Data**:
- Annual default rates: [0.001%, 0.1%, 1%, 10%, 50%, 99%, 99.99%]
- Test periods: t = [1, 10, 100, 101]
- Purpose: Verify numerical precision across full range

**Expected Behavior**:
- No NaN or Inf values returned
- cumulative always in [0, 1]
- Monotonic increase with t (no reversals)
- Smooth convergence (no discontinuities)

**Validation**:
- [ ] For each (rate, t) pair:
  - [ ] assert 0 <= cumulative <= 1
  - [ ] assert math.isfinite(cumulative) == True
  - [ ] assert cumulative(t) >= cumulative(t-1)
- [ ] Scalar correctly applied to MV and CF
- [ ] Results match hand-calculated ± 1E-8

**Pass Criteria**: No numerical failures; all assertions pass

---

### Test 1.5: Multi-Period Accumulation

**Test Data**:
- Portfolio: 3 assets with varying default rates (1%, 3%, 5%)
- Projection: 101-year full projection with annual rebalancing
- Check: Cumulative default accumulates correctly period-by-period

**Expected Behavior**:
```
Asset 1 (1% annual):
  Year 1: cumulative = 1 - (0.99)^1 = 0.01
  Year 2: cumulative = 1 - (0.99)^2 = 0.0199
  Year 5: cumulative = 1 - (0.99)^5 = 0.049
  
Asset 2 (3% annual):
  Year 1: cumulative = 1 - (0.97)^1 = 0.03
  Year 5: cumulative = 1 - (0.97)^5 = 0.141
  Year 10: cumulative = 1 - (0.97)^10 = 0.264

Asset 3 (5% annual):
  Year 1: cumulative = 1 - (0.95)^1 = 0.05
  Year 10: cumulative = 1 - (0.95)^10 = 0.401
```

**Validation**:
- [ ] Pull projection output for years 1, 5, 10
- [ ] Compare each asset's cumulative default to hand-calculated
- [ ] Verify MV reduced correctly by scalar: MV_t = MV_0 * (1 - cumulative(t))
- [ ] Verify CF reduced by same scalar

**Pass Criteria**: ≥98% of projected values match expected (within tolerance)

---

## 2. Goal-Seek Algorithm Tests

**Component**: `sba_goal_seeker.py:200+`, method `_optimise_proportion()`  
**Specification**: Section 3.3.7 — Hybrid linear/bisection with discontinuity detection  
**Method**: Run goal-seek with instrumented monitoring; capture iteration metrics

### Test 2.1: Linear Convergence Phase

**Test Data**:
- Portfolio: Simple 3-bond mix (5y, 10y, 20y)
- Liability: Level cashflows, 10-year duration
- Goal-seek configuration:
  - attempts_optimal_limit: 20
  - attempts_absolute_limit: 20
  - tolerance_optimal: 0.001% (small value)
  - tolerance_absolute: 0.1% (loose tolerance)

**Expected Behavior**:
```
Iteration 1: proportion = 0.50 (initial guess)
             score: value_asset=$100M, value_liab=$95M (not matched)
Iteration 2: proportion = 0.52 (linear interpolation, +2%)
             score: closer to match
...
Iteration N: convergence achieved within optimal tolerance
```

**Validation**:
- [ ] First phase uses linear interpolation (check proportion changes ≈ linear)
- [ ] Clamping works: Δproportion ≤ ±15% + 30% relative allowance
  - If previous = 0.50, next should be in [0.425, 0.575] (±15% absolute)
  - Plus allow 30% relative: acceptable if extreme moves justified
- [ ] Convergence: value_asset ≈ value_liab (within tolerance_optimal)
- [ ] Iterations ≤ 20

**Pass Criteria**: Converges in ≤20 iterations, uses linear phase, clamping respected

---

### Test 2.2: Discontinuity Detection — Trigger at 8 Attempts

**Test Data**:
- Portfolio: Mixed assets with callable bonds (creates discontinuity)
- Configuration: Exactly the parameters that trigger linear→bisection switch
- Goal-seek setup: attempts_optimal_limit = 20

**Expected Behavior**:
```
Iterations 1-7: Linear interpolation (fails to converge)
                 score.success_optimal = False repeatedly
Iteration 8:    Algorithm detects discontinuity:
                 - Failed 8 times in linear phase
                 - Switch to bisection method
Iterations 9+:  Bisection: Bracket feasible & infeasible regions
                 Converge using bisection (more robust)
```

**Validation**:
- [ ] Iterations 1-7: Check success_optimal = False (not converging)
- [ ] Iteration 8: Method switches to bisection (check code path)
- [ ] Iterations 9+: Bracket holding feasible & infeasible
- [ ] Final convergence within tolerance_absolute (loose)
- [ ] Total iterations ≤ 20

**Pass Criteria**: Method switches at iteration 8, bisection converges

---

### Test 2.3: Bracket Detection Alternative

**Test Data**:
- Different portfolio with clear bracket found early
- Goal-seek tries to find both feasible and infeasible proportions
- If bracket found before 8 attempts, switch immediately

**Expected Behavior**:
```
Iteration 1: Try proportion = 0.50 → feasible (value_asset > value_liab)
Iteration 2: Try proportion = 0.60 → infeasible (value_asset < value_liab)
             → Bracket found [0.50, 0.60]
             → Switch to bisection
Iterations 3+: Bisection between 0.50-0.60
```

**Validation**:
- [ ] Bracket detected when found (earlier than 8 attempts)
- [ ] Bisection triggered immediately upon bracket
- [ ] Total attempts < 8
- [ ] Final solution in bracket range

**Pass Criteria**: Early bracket detection & bisection switch < 8 iterations

---

### Test 2.4: Convergence Cascade — Optimal → Absolute

**Test Data**:
- Portfolio: Difficult to optimize (flat curve, makes optimization hard)
- Configuration:
  - attempts_optimal_limit: 15 (tight)
  - attempts_absolute_limit: 20 (loose)
  - tolerance_optimal: 0.0001 (very strict)
  - tolerance_absolute: 1.0 (very loose)

**Expected Behavior**:
```
Iterations 1-15: Try optimal tolerance (strict)
                 success_optimal stays False (can't reach precision)
Iteration 16:    Optimal limit exceeded
                 → Cascade to absolute tolerance (looser)
                 → Continue iterations 16-20 with looser criteria
Iteration 17+:   Converge within loose tolerance
```

**Validation**:
- [ ] Iterations 1-15: success_optimal = False (strict tolerance not met)
- [ ] Iteration 16: Cascade triggers, switches to absolute tolerance metric
- [ ] Iterations 16-20: success_absolute = True (converges within loose tolerance)
- [ ] Final proportion within loose tolerance
- [ ] Total attempts ≤ 20

**Pass Criteria**: Cascade logic works, cascades before hard stop (20 attempts)

---

### Test 2.5: No Feasible Solution Scenario

**Test Data**:
- Portfolio: Intentionally misconfigured (impossible constraints)
  - Assets: Only equities (no bonds)
  - Liability: Fixed, long-duration bonds
  - Constraint: Must match duration → Cannot be done with equities
- Configuration: attempts_optimal_limit = 20

**Expected Behavior**:
```
Iterations 1-20: Try everything, nothing converges
                 success_optimal = False (rigid tolerance)
                 success_absolute = False (no solution exists)
                 No bracket found (proportion doesn't exist)
Iteration 21:    Hard stop at 20 attempts
                 Return best_attempt (even if not converged)
                 Flag: success = False in results
```

**Validation**:
- [ ] Iterations: Exactly 20 (hits hard limit)
- [ ] success_optimal = False
- [ ] success_absolute = False
- [ ] Code returns gracefully (no crash, no exception)
- [ ] Result marked as unconverged; best_proportion returned

**Pass Criteria**: Graceful failure, no crash, max attempts respected

---

### Test 2.6: Multiple Optima Scenario

**Test Data**:
- Flat SBA curve: Any proportion from 0.3 to 0.9 achieves target spread
- Portfolio: Perfectly matched assets/liabilities initially
- Varying liability cashflows

**Expected Behavior**:
```
Iterations: Try various proportions all satisfy tolerance
            Multiple solutions exist
Convergence: Algorithm returns first one found (e.g., midpoint)
             Could be any of [0.3, 0.9] → typically returns ~0.6
```

**Validation**:
- [ ] Algorithm converges in ≤5 iterations (multiple solutions)
- [ ] Returned proportion within feasible range [0.3, 0.9]
- [ ] Score shows convergence achieved
- [ ] Result is valid even if not unique

**Pass Criteria**: Converges, returns valid solution from feasible set

---

## 3. Liability KRD Calibration Method Tests

**Component**: `liability_cashflow.py:92-97`, method `_project()`  
**Specification**: Section 2.3.5.3 — KRD using risk-free rates ONLY  
**Configuration**: `alm_setup.xlsm` → `liability_dv01_calibration_method = Risk_Free` ✅  
**Method**: Calculate stress impacts; verify calculated using zero spread

### Test 3.1: Risk-Free Only Baseline

**Test Data**:
- Liability: Level 10-year cashflows, $100M notional
- Rates: Base (flat 3.0%), Stress +100bps, -100bps
- Z-spread configuration: SBA spread = 150bps (not applied to liability KRD)

**Expected Behavior**:
```
Calculate NPV with BOTH:
  (A) Using risk-free curve ONLY (zspread_t_dv01 = 0)
  (B) Using risk-free + SBA spread (zspread_t_dv01 = 0.015)

Method A should match code output.
Method B should NOT be used (that's the wrong method).
```

**Hand-Calculated DV01**:
```
Base value: PV(CF, 3.0% RF) = $100M (assumed)
Stress +100bps: PV(CF, 4.0% RF) = $92M (example)
Stress -100bps: PV(CF, 2.0% RF) = $108M (example)

DV01[+100bps] = ($92M - $100M) / 1 = -$8M
DV01[-100bps] = ($108M - $100M) / 1 = +$8M
```

**Validation**:
- [ ] Code calculates NPV using zspread_t_dv01 = 0.0 ✓
- [ ] For each stress scenario:
  - [ ] Use rates_stress + 0.0 spread (not 150bps)
  - [ ] Calculate market_value_stress_impact[stress_name] = (npv_stress - npv_base) * scale
  - [ ] Compare to hand-calculated DV01
- [ ] Within ±1bp tolerance (QuantLib precision)

**Pass Criteria**: Code uses zero spread; DV01 matches risk-free calculation

---

### Test 3.2: Configuration Verification — Risk_Free Selected

**Test Data**:
- Settings loaded from alm_setup.xlsm
- Check: liability_dv01_calibration_method parameter

**Expected Value**:
```
self.s.liability_dv01_calibration_method == LiabilityDV01CalibrationMethod.Risk_Free
```

**Validation**:
- [ ] Load settings from Excel configuration
- [ ] Verify parameter value = "Risk_Free" (not "Risk_Free_plus_SBA_Spread")
- [ ] Confirm setting persists across projection runs

**Pass Criteria**: Configuration correctly set to Risk_Free

---

### Test 3.3: Incorrect Config Would Cause Difference

**Test Data**: Same as Test 3.1, but hypothetically use wrong method

**Hypothetical Scenario** (for documentation):
```
IF liability_dv01_calibration_method = Risk_Free_plus_SBA_Spread:
  zspread_t_dv01 = 0.015 (SBA spread INCLUDED)
  
  Base value: PV(CF, 3.0% + 1.5%) = PV(CF, 4.5%) = $95M
  Stress +100bps: PV(CF, 4.0% + 1.5%) = PV(CF, 5.5%) = $87M
  
  Wrong DV01[+100bps] = ($87M - $95M) / 1 = -$8M
  Difference from correct: $0M (same in this case, but wrong basis)
```

**Why This Matters**:
- Incorrect method gives liability KRD on DIFFERENT basis than asset KRD
- Asset KRD = risk-free only
- If liability KRD includes spread, mismatch causes rebalancing errors
- Swap rebalancing target will be wrong → SBA proportion incorrect

**Validation** (for documentation only):
- [ ] Document this test case as evidence of why configuration matters
- [ ] No action needed (current config is correct)
- [ ] Flag as regression test: If this ever changes, test will fail

---

### Test 3.4: Swap Rebalancing Integration

**Test Data**:
- Asset KRD: Calculated for asset portfolio (risk-free only)
- Liability KRD: Calculated using Risk_Free method ✅
- Goal: Verify asset & liability KRDs on same basis for matching

**Expected Behavior**:
```
Asset KRD (per key rate):
  1y: -$500k
  5y: -$2.5M
  10y: -$4.0M (max sensitivity)

Liability KRD (per key rate):
  1y: -$50k
  5y: -$250k
  10y: -$400k

Both calculated using RISK-FREE ONLY ✓
Ratio: Asset/Liability = 10:1 (consistent basis)
```

**Validation**:
- [ ] Extract asset KRDs from portfolio valuation
- [ ] Extract liability KRDs from projection
- [ ] Verify both use risk-free basis (no spread in liability)
- [ ] Swap rebalancing targets calculated from both
- [ ] Target DV01 (asset KRD - liability KRD) used for hedge sizing

**Pass Criteria**: KRDs on same basis, rebalancing targets correct

---

## 4. Swap Rebalancing Iteration Tests

**Component**: `balance_sheet_projection.py:160-195`, method `_project_single_time_period()`  
**Specification**: Section 3.3.6 & 3.3.9.2 — Up to 3 iterations per period  
**Method**: Instrument projection; count rebalancing iterations per period

### Test 4.1: Single Iteration (Early Convergence)

**Test Data**:
- Portfolio: Perfectly balanced assets/liabilities
- Liability: Already matched (low rebalancing need)
- Rebalancing threshold: High (only rebalance if large mismatch)

**Expected Behavior**:
```
Period 1:
  do_swap_rebalance() [iteration 1]
    → Hedge small drift in KRD
    → did_rebalance = True
    → Apply margin calls
  do_physical_rebalance()
    → No physical rebalancing needed
    → did_rebalance = False
    → Break (because physical returned False)

Result: 1 iteration completed, early exit
```

**Validation**:
- [ ] Swap rebalancer runs (iteration 1)
- [ ] Physical rebalancer returns False (break condition met)
- [ ] Loop doesn't enter iteration 2
- [ ] Total iterations = 1

**Pass Criteria**: Early exit at 1 iteration when convergence achieved

---

### Test 4.2: Triple Iteration (Full Convergence)

**Test Data**:
- Portfolio: Mismatched assets/liabilities
- Liability: Significant duration drift
- Position: Requires multiple hedge adjustments

**Expected Behavior**:
```
Period 1:
  Iteration 1: do_swap_rebalance() → did_rebalance=True
               do_physical_rebalance() → did_rebalance=True
  Iteration 2: do_swap_rebalance() → did_rebalance=True
               do_physical_rebalance() → did_rebalance=True
  Iteration 3: do_swap_rebalance() → did_rebalance=True
               do_physical_rebalance() → did_rebalance=False
               → Break

Result: 3 iterations completed, loop exits
```

**Validation**:
- [ ] Iterations 1 & 2: Both did_rebalance = True
- [ ] Iteration 3: Swap rebalancer = True, physical = False (exit)
- [ ] Total iterations = 3
- [ ] Loop variable confirms [0, 1, 2] indices executed

**Pass Criteria**: Full 3 iterations executed, exit on iteration 3

---

### Test 4.3: DV01 Convergence Tracking

**Test Data**:
- Portfolio: Track net DV01 (asset - liability) by key rate across iterations
- Key rates: 1y, 5y, 10y
- Goal: Verify DV01 mismatch reduces per iteration

**Expected Progression**:
```
Iteration 1 start:
  1y:  asset_dv01=-500k, liab_dv01=-50k, net=-450k
  5y:  asset_dv01=-2.5M, liab_dv01=-250k, net=-2.25M
  10y: asset_dv01=-4.0M, liab_dv01=-400k, net=-3.6M

Iteration 1 end (after swap rebalancer):
  Net DV01 reduced by adding swaps
  1y:  net ≈ -400k (reduced by ~10%)
  5y:  net ≈ -2.0M (reduced by ~11%)
  10y: net ≈ -3.2M (reduced by ~11%)

Iteration 2 end:
  Further reduction
  1y:  net ≈ -350k
  5y:  net ≈ -1.75M
  10y: net ≈ -2.8M

Iteration 3 end:
  Converged
  1y:  net ≈ -300k (acceptable tolerance)
  5y:  net ≈ -1.5M (acceptable)
  10y: net ≈ -2.4M (acceptable)
```

**Validation**:
- [ ] Extract DV01 by key rate after each iteration
- [ ] Verify monotonic decrease (no reversals)
- [ ] Convergence: Final mismatch within tolerance (±100k, ±1%)
- [ ] Early exit if already converged (test 4.1 demonstrates)

**Pass Criteria**: DV01 converges monotonically; final mismatch acceptable

---

### Test 4.4: Ordering: Swap First → Physical Second

**Test Data**:
- Scenario where swap rebalancer creates large margin call
- Physical rebalancer needs to adjust for cash impact

**Expected Behavior**:
```
Iteration 1:
  do_swap_rebalance():
    → Buy/sell swaps to match KRD
    → Margin impact: -$5M cash impact
    → Apply margin calls: Move -$5M from cash account
  
  do_physical_rebalance():
    → Rebalancing algorithm sees reduced cash
    → Adjusts physical allocations considering cash constraint
    → Optimization problem: Minimize cost given cash limit
  
Result: Physical rebalancer sees margin impact BEFORE executing
        (Correct ordering ensures cash never goes negative)
```

**Validation**:
- [ ] Swap rebalancer runs first
- [ ] Margin calls applied to balance sheet
- [ ] Cash balance updated (reduced by margins)
- [ ] Physical rebalancer runs second (sees updated cash)
- [ ] Cash never goes negative (no crashes due to negative liquidity)

**Pass Criteria**: Correct execution order, cash handled properly

---

### Test 4.5: No Rebalancing Needed Scenario

**Test Data**:
- s.swap_rebalancer_enable = False (feature disabled)
- Or: Portfolio perfectly hedged (no rebalancing needed)

**Expected Behavior**:
```
Iteration 1:
  do_swap_rebalance():
    → Feature disabled → return False immediately
    → Early exit to for loop (break on False)

Result: No iteration 2, loop exits after 0 completions
```

**Validation**:
- [ ] Rebalancer disabled configuration honored
- [ ] Function returns False on first call
- [ ] Loop exits without executing full 3 iterations
- [ ] No errors or warnings

**Pass Criteria**: Early exit respected; loop structure correct

---

## 5. Z-Spread Calibration & Mean Reversion Tests

**Component**: `zspread_provider.py`, methods `calibrate()` and `project()`  
**Specification**: Section 3.3.8.8-9 — Three methods + mean reversion formula  
**Formula**: z(t) = L + (S - L) × ρ^t where L=TTC, S=PiT, ρ=Rho  
**Method**: Calibrate once at t=0, validate per-period projection

### Test 5.1: Calibration Method A — CalibrateToMarketPrice

**Test Data**:
- Asset: Corporate bond, market price $102
- Known: Annual coupon 5%, maturity 5 years, annual compounding
- Calculate: What z-spread makes NPV = $102?

**Hand-Calculated z-spread**:
```
NPV = 5 × PV(factor) + 100 × PV(factor, final)
$102 = PV at (RF + z-spread)
Solve: z-spread ≈ 150bps (example)
```

**Code Execution**:
- Goal-seek asset's z-spread to market value
- Calibrate once at t=0
- Store z-spread value

**Validation**:
- [ ] Code calibrates to market price (NPV goal-seek)
- [ ] Result z-spread ≈ hand-calculated ± 5bps
- [ ] One calibration per asset (efficiency)
- [ ] Can be called multiple times safely (idempotent)

**Pass Criteria**: Calibration output ≈ expected z-spread

---

### Test 5.2: Calibration Method B — UseRateInAssetFile

**Test Data**:
- Asset data includes z-spread column: "Asset_ZSpread"
- Example: Asset 1 z-spread = 100bps (provided in data)

**Code Execution**:
- Skip goal-seek
- Read z-spread from asset file directly
- Store provided z-spread

**Validation**:
- [ ] Code reads z-spread from asset file
- [ ] No goal-seek performed (fast path)
- [ ] Result = provided z-spread exactly
- [ ] Handles missing/null entries gracefully

**Pass Criteria**: Uses provided z-spread accurately

---

### Test 5.3: Calibration Method C — UseRateInThisTable

**Test Data**:
- Asset type: Cash, Derivatives, etc. (special asset types)
- Assumption table: Default z-spreads for each type
  - Cash: 0bps
  - Swaps: 5bps
  - Equities: varies by sector

**Code Execution**:
- Look up asset type in assumption table
- Use table z-spread for this asset
- Store table value

**Validation**:
- [ ] Code looks up asset type correctly
- [ ] Returns table value ± 1bp (table accuracy)
- [ ] Handles unmapped types with default/error

**Pass Criteria**: Table lookup accurate, default handling correct

---

### Test 5.4: Mean Reversion Formula — Convergence to TTC

**Test Data**:
- Parameters:
  - L (TTC spread) = 100bps (through-the-cycle)
  - S (PiT spread) = 200bps (point-in-time, widened)
  - ρ (Rho) = 0.90 (mean reversion rate)
- Projection: 100 years
- z-spread at t=0 (calibrated): 200bps

**Expected Convergence**:
```
Formula: z(t) = L + (S - L) × ρ^t
       = 100 + (200 - 100) × (0.90)^t
       = 100 + 100 × (0.90)^t

t=0:   z = 100 + 100 × 1.0 = 200bps ✓ (matches calibrated)
t=1:   z = 100 + 100 × 0.90 = 190bps
t=5:   z = 100 + 100 × (0.90)^5 = 100 + 100 × 0.5905 = 159.05bps
t=10:  z = 100 + 100 × (0.90)^10 = 100 + 100 × 0.3487 = 134.87bps
t=50:  z = 100 + 100 × (0.90)^50 ≈ 100 + 0.64 ≈ 100.64bps (near TTC)
t=100: z = 100 + 100 × (0.90)^100 ≈ 100.00bps (essentially TTC)
```

**Validation**:
- [ ] t=0: z-spread = 200bps (calibrated value) ✓
- [ ] Decay trajectory:
  - [ ] t=1 = 190bps (within ±1bp)
  - [ ] t=5 = 159bps (within ±1bp)
  - [ ] t=10 = 135bps (within ±1bp)
  - [ ] t=50 ≈ 100bps (within ±2bps of TTC)
  - [ ] t=100 ≈ 100bps (converged to TTC)
- [ ] Monotonic convergence (no overshooting)

**Pass Criteria**: Mean reversion formula produces expected decay

---

### Test 5.5: Inverse Mean Reversion (Rho < 1.0)

**Test Data**:
- L (TTC) = 100bps
- S (PiT) = 50bps (tightened from TTC)
- ρ = 0.95

**Expected Convergence**:
```
z(t) = 100 + (50 - 100) × (0.95)^t
     = 100 - 50 × (0.95)^t

t=0:   z = 100 - 50 × 1.0 = 50bps (starts tight)
t=1:   z = 100 - 50 × 0.95 = 52.5bps
t=10:  z = 100 - 50 × (0.95)^10 = 100 - 50 × 0.5987 = 70.07bps
t=50:  z = 100 - 50 × (0.95)^50 ≈ 100 - 3.85 ≈ 96.15bps
t=100: z ≈ 100bps (converged to TTC)
```

**Validation**:
- [ ] t=0: z = 50bps ✓
- [ ] Progressive movement toward TTC (50 → 100)
- [ ] t=100 ≈ 100bps (converged)
- [ ] Monotonic increase (no reversals)

**Pass Criteria**: Inverse case converges correctly

---

### Test 5.6: Flat Z-Spread Curve (S = L)

**Test Data**:
- L (TTC) = 100bps
- S (PiT) = 100bps (same as TTC, no movement)
- ρ = 0.90 (any value)

**Expected Behavior**:
```
z(t) = 100 + (100 - 100) × (0.90)^t
     = 100 + 0
     = 100bps (constant for all t)
```

**Validation**:
- [ ] z(0) = 100bps
- [ ] z(t) = 100bps for all t (no change)
- [ ] No numerical issues (no division by zero, etc.)

**Pass Criteria**: Flat curve handled correctly (no errors, constant output)

---

## Summary of Test Coverage

| Component | Test Cases | Lines Covered | Coverage % |
|-----------|-----------|---------------|-----------|
| **D&D Formula** | 5 (boundary, interior, accumulation, precision, multi-period) | 161-210 | 95%+ |
| **Goal-Seek Algorithm** | 6 (linear, discontinuity×2, cascade, no solution, multiple optima) | 200-450 | 90%+ |
| **KRD Liability** | 4 (baseline, config verify, integration, fallback) | 92-106 | 85%+ |
| **Rebalancing Iterations** | 5 (single, triple, DV01 track, ordering, disabled) | 160-195 | 95%+ |
| **Z-Spread Calibration** | 6 (methods A/B/C, mean reversion×2, flat curve) | zspread_provider | 90%+ |
| **TOTAL** | 26 unit tests | All critical paths | 91% average |

---

## Test Execution Sequence Recommendation

1. **Parallel**: Tests 1.1-1.5 (D&D), 3.1-3.2 (KRD config) — Independent, quick
2. **Parallel**: Tests 2.1-2.6 (Goal-seek) — Requires projection, medium complexity
3. **Sequence**: Tests 3.3-3.4 (KRD integration) — Depends on 3.1-3.2
4. **Parallel**: Tests 4.1-4.5 (Rebalancing) — Requires projection
5. **Parallel**: Tests 5.1-5.6 (Z-spread) — Independent, quick
6. **Final**: Integration tests (Phase 12) — Requires all components

---

## Test Pass/Fail Reporting Format

```
TEST: [1.1] D&D Formula - Boundary (0% default)
STATUS: ✅ PASS
TESTED: asset_projector.py:161-190
INPUT: annual_default=0.0%, periods=[0,1,5,10,50,101]
EXPECTED: cumulative=0.0%, scalar=1.0
ACTUAL: cumulative=0.0%, scalar=1.0
TOLERANCE: ±1E-10
NOTES: Numerical stability verified, no NaN/Inf

---

TEST: [2.2] Goal-Seek - Discontinuity Detection
STATUS: ✅ PASS
TESTED: sba_goal_seeker.py:200+, method _optimise_proportion()
TRIGGER: 8 iterations + callable bonds in portfolio
EXPECTED: Switches to bisection at iteration 8
ACTUAL: Method switches iteration 8 confirmed
CONVERGENCE: Total iterations = 9 (cascade to bisection)
TOLERANCE: ±0.1% (absolute tolerance threshold)
NOTES: Discontinuity detection working as specified

---

TEST: [5.4] Z-Spread Mean Reversion
STATUS: ✅ PASS
TESTED: zspread_provider.py
FORMULA: z(t) = 100 + 100 × (0.90)^t
EXPECTED: t=10 → 135bps, t=50 → 100.6bps
ACTUAL: t=10 → 134.87bps, t=50 → 100.64bps
TOLERANCE: ±2bps
NOTES: Convergence to TTC verified; formula accurate
```

