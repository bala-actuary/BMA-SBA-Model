# 13: Edge Case & Stress Tests

Comprehensive tests for extreme scenarios, data anomalies, and boundary conditions.

---

## Edge Case Testing Strategy

Edge cases verify that the SBA implementation handles unusual/extreme scenarios gracefully:
- No crashes or exceptions
- Numerical stability maintained
- Reasonable fallback behavior
- Clear error/warning messages

**Testing Principle**: "Chaos engineering" — break things intentionally to find where code fails gracefully

---

## Category 1: Portfolio Extremes

### Test E1.1: All-Bond Portfolio (No Rebalancing Needed)

**Scenario**:
- Portfolio: 100% bonds (5y, 10y, 20y blend)
- Equity allocation: 0%
- Liability: Matched duration to bond portfolio

**Expected Behavior**:
- Asset/liability durations already matched
- Goal-seek finds proportion near 1.0 (all assets, minimal liability sidecar)
- Rebalancing: Minimal swaps needed (already matched)
- No errors; projection runs smoothly

**Validation**:
- [ ] Goal-seek converges (likely in 2-3 iterations)
- [ ] SBA proportion ≈ 0.95-1.0 (high asset allocation acceptable)
- [ ] Rebalancing iterations = 0-1 per period (early exits)
- [ ] No crashes or exceptions

---

### Test E1.2: All-Equity Portfolio (Maximum Rebalancing)

**Scenario**:
- Portfolio: 100% equities or 100% cash
- No bonds for structural hedging
- Liability: 10-year duration bonds

**Expected Behavior**:
- Equity duration ≈ 0 (no interest rate sensitivity)
- Liability mismatch forces maximum hedging
- Goal-seek finds low asset percentage (e.g., 20% assets, 80% liability)
- Heavy swap rebalancing needed (synthetic bonds via swaps)

**Validation**:
- [ ] Goal-seek converges (might take more iterations; challenging optimization)
- [ ] SBA proportion ≤ 0.3 (low asset %, high liability %+swaps)
- [ ] Rebalancing iterations = 2-3 per period (working hard)
- [ ] Swap notionals large (synthetic duration matching)
- [ ] No crashes; numerical stability maintained

---

### Test E1.3: Single Asset Class (All-Bonds of One Maturity)

**Scenario**:
- Portfolio: $100M of 10-year corporate bonds only
- No maturity ladder diversity
- No equity diversification

**Expected Behavior**:
- Limited allocation flexibility
- Duration matching limited to single maturity
- Rebalancing: Only swaps can hedge duration mismatch
- May struggle to match complex liability cashflows

**Validation**:
- [ ] Goal-seek converges (or documented as "difficult optimization")
- [ ] Swaps added to hedge remaining mismatch
- [ ] Rebalancing iterations possibly 3 (hits limit)
- [ ] Final KRD match approximate (not perfect due to constraints)
- [ ] Warning logged: "Limited asset diversity" (if feature exists)

---

### Test E1.4: Extreme Maturity Ladder (Very Concentrated)

**Scenario**:
- Ladder A: All assets in 2-3y bucket only (bullet portfolio)
- Liability: Spread 1-30y

**Expected Behavior**:
- Bullet portfolio has concentrated duration
- Massive mismatch with liability ladder
- Rebalancing: Requires sophisticated swaps (steepeners, butterflies, etc.)
- May hit rebalancing limits

**Validation**:
- [ ] Goal-seek converges
- [ ] Swap rebalancing complex (multiple key rates active)
- [ ] Final KRD match within acceptable tolerance
- [ ] No numerical errors in complex swaps

---

## Category 2: Market Stress Scenarios

### Test E2.1: Parallel Rate Shock (+100bps)

**Scenario**:
- Base rates: 2.5% (all maturities)
- Stress: 3.5% (flat parallel shift)
- Shock realism: Large but plausible rate rise

**Expected Behavior**:
- Bond MVs sharply decrease (~7% for 10-year bonds)
- KRDs recalculated under new base rates
- Goal-seek re-converges (may find different proportions)
- Rebalancing targets change

**Validation**:
- [ ] Asset P&L: Large loss (~7% per bond)
- [ ] KRDs: Smaller magnitude (higher rates flatten duration)
- [ ] Rebalancing: Positions adjusted for new rates
- [ ] Goal-seek finding under stress rates valid
- [ ] No crashes, reasonable outputs

---

### Test E2.2: Negative Interest Rates

**Scenario**:
- Base rates: -0.50% (some maturities, like Eurozone scenario)
- Short rates negative; long rates positive (inverted from current norm)

**Expected Behavior**:
- QuantLib handles negative rates
- NPV calculations still work (no division by zero, etc.)
- KRD durations still meaningful
- Rebalancing logic unchanged

**Validation**:
- [ ] Projection runs without crashes
- [ ] NPV finite and appropriate (bonds worth more in negative rate environment)
- [ ] KRDs calculated correctly (no sign inversions, NaN)
- [ ] No special cases break (e.g., log functions)

---

### Test E2.3: Extreme Rates (+400bps Floor, -1% Ceiling)

**Scenario**:
- Base rates: 4.0% (very high, e.g., 1980s scenario)
- Upper: 5.0% (even higher stress)
- Lower: -1.0% (zero-lower-bound violation, theoretical)

**Expected Behavior**:
- QuantLib may have rate boundaries
- Numerical precision: Higher chance of overflow/underflow
- Duration calculations: Large changes with rate moves

**Validation**:
- [ ] No crashes at extreme rates
- [ ] QuantLib gracefully handles bounds (error message if exceeded)
- [ ] NPV conversions correct despite large rate magnitudes
- [ ] Output numerical stability maintained

---

### Test E2.4: Curve Steepening (Long-end rates up, short-end down)

**Scenario**:
- Curve initially: Flat 3.0%
- Stress: 1y = 2.5%, 10y = 3.5%, 30y = 4.0% (steep curve)

**Expected Behavior**:
- Long-duration assets affected more
- Short-duration liabilities less affected
- KRD mismatch by key rate (short ≠ long)
- Rebalancing targets by key rate change non-uniformly

**Validation**:
- [ ] KRD by key rate shows differential impact
- [ ] Swap rebalancing considers multiple key rates
- [ ] Convergence across key rates achieved
- [ ] No numerical issues from curve twist

---

### Test E2.5: Credit Spread Widening (+500bps)

**Scenario**:
- Base spreads: 100bps (normal credit environment)
- Stress spreads: 600bps (financial crisis scenario)
- Asset z-spreads recalibrated

**Expected Behavior**:
- Corporate bond values plummet
- Recalibration updates spreads
- Goal-seek may find higher liability allocation (SBA bonds more attractive)
- Rebalancing adjusts for new asset values

**Validation**:
- [ ] Asset MVs decrease ~20-40% (depending on duration/spread sensitivity)
- [ ] Z-spread recalibration reflects wider spreads
- [ ] KRDs updated (wider spreads can affect duration)
- [ ] Projection stable despite crisis conditions

---

## Category 3: Data Anomalies & Boundary Cases

### Test E3.1: Missing Z-Spread (Calibration Fails)

**Scenario**:
- Asset: Corporate bond, but z-spread calibration fails
  - Market price inconsistent with formula
  - Optimization doesn't converge
  - Missing data field

**Expected Behavior**:
- Fallback mechanism activates (if built-in)
- Use alternative source (asset file, assumption table)
- Or: Return default z-spread (0bps, warning)
- Projection continues (degraded, but not crashed)

**Validation**:
- [ ] Projection doesn't crash
- [ ] Fallback z-spread used (logged/audited)
- [ ] Asset still in portfolio (with default spread)
- [ ] Warning message generated

---

### Test E3.2: Zero Market Value Asset

**Scenario**:
- Asset: Corporate bond, $0 notional (position liquidated or erroneously entered)
- Portfolio still includes entry

**Expected Behavior**:
- Asset included but contributes nothing to KRD, P&L
- Rebalancing ignores (zero weight in optimization)
- No division-by-zero errors
- Projection stable

**Validation**:
- [ ] Market value: $0 (not negative, not NaN)
- [ ] KRD: Treated as immaterial (~$0 or excluded)
- [ ] P&L: Unaffected by zero asset
- [ ] Rebalancing doesn't break (linear algebra stable)

---

### Test E3.3: Callablbe Bond at Call Date (Or Past It)

**Scenario**:
- Callable bond: 10y maturity, callable from year 5, call strike $104
- Projection: At year 5 or year 6 (call date reached or passed)
- Market: Bond trading above call strike

**Expected Behavior**:
- Bond likely called (issuer exercise)
- Removed from portfolio
- Position ends
- Cash proceeds reinvested

**Validation**:
- [ ] Wave-off detected and executed
- [ ] Bond P&L: Capped at call strike (issuer upside captured)
- [ ] Portfolio updated (bond removed)
- [ ] No holdover errors (valuation of "dead" bond)

**Caveat**: Spec notes callable bond validation "not fully tested"

---

### Test E3.4: Liability Cashflows Mismatched (Projection Horizon)

**Scenario**:
- Liability: All cashflows in years 1-5
- Projection: 101 years
- Mismatch: Liability fully paid off after year 5

**Expected Behavior**:
- Liability continues in years 6-101 (balance = $0 or fully paid)
- Portfolio runs on inertia (no more liability rebalancing)
- Goal-seek may re-trigger if configuration says "annual" not "one-time"

**Validation**:
- [ ] Liability correctly reduces to zero after final cashflow
- [ ] Portfolio stable after liability runoff
- [ ] No crashes from zero-liability edge case
- [ ] Final state reasonable

---

### Test E3.5: Negative Spreads (Rare, But Possible)

**Scenario**:
- Asset: Government bond with z-spread = -50bps (trading below risk-free)
- Possible in very strong credit, negative rate environment

**Expected Behavior**:
- QuantLib handles negative spreads
- NPV calculations use: discount = RF + (-50bps)
- Bond prices higher due to negative spread
- KRD calculations still valid

**Validation**:
- [ ] No crashes on negative spreads
- [ ] NPV reasonable (likely higher than RF-only case)
- [ ] KRD calculations logical
- [ ] Projection stable

---

## Category 4: Numerical Extremes

### Test E4.1: Very Small Tolerance (Tight Goal-Seek)

**Scenario**:
- Configuration: goal_seek_tolerance_optimal = 0.0001% (very tight, 0.000001 in decimal)
- Portfolio: Normal

**Expected Behavior**:
- Goal-seek takes more iterations to satisfy (10-15 vs normal 8-12)
- May cascade to absolute tolerance if optimal unachievable
- Eventually converges or hits iteration limit

**Validation**:
- [ ] Converges within attempts limit (e.g., ≤20 iterations)
- [ ] Or cascades gracefully to absolute tolerance
- [ ] Final proportion high precision (matches tolerance)
- [ ] No slow runaway (iterations increase but bounded)

---

### Test E4.2: Very Large Portfolio (50+ Assets)

**Scenario**:
- Portfolio: 50 bonds, 10 equities, 5 derivatives
- Liability: Complex cashflow ladder

**Expected Behavior**:
- Memory usage increases but should remain manageable (<500MB)
- Linear algebra (portfolio optimization): Larger matrix (50x50+)
- Computation time increases (typically linear or quadratic)
- Rebalancing solver may take longer

**Validation**:
- [ ] Projection completes (no crash)
- [ ] Computation time reasonable (< 10 sec for 1-year dryrun)
- [ ] Memory usage acceptable
- [ ] Output quality same as smaller portfolio

---

### Test E4.3: Long Projection Horizon (101 Years)

**Scenario**:
- Full 101-year projection (tail behavior)
- All portfolio math for 101 periods

**Expected Behavior**:
- Cumulative numerical errors possible but manageable
- D&D: Assets largely eliminated by year 50+ (cumulative ~90%+)
- Projection stability: Portfolio converges to terminal state
- Memory: Storing 101 periods worth of data

**Validation**:
- [ ] Year 100-101 values reasonable (not oscillating nor exploding)
- [ ] No sudden jumps in metrics (smooth tail)
- [ ] Numerical precision acceptable (balance sheet sums correct)
- [ ] Final year results make sense (portfolio runoff complete)

---

### Test E4.4: Floating-Point Precision at Boundaries

**Scenario**:
- Cumulative default: Asset with 1% annual default reaching year 100
- Expected cumulative: 1 - (0.99)^100 ≈ 0.6339 (63.39%)
- Actual calculation: May have rounding

**Expected Behavior**:
- Cumulative default correctly calculated despite rounding
- Bounds checked: 0 ≤ cumulative ≤ 1
- No overflow/underflow (0.99^100 is well-behaved in float)

**Validation**:
- [ ] Cumulative value within ±0.0001 of expected
- [ ] Bounds assertions pass
- [ ] No NaN or Inf values
- [ ] Scalar applied correctly despite rounding

---

## Category 5: Configuration Extremes

### Test E5.1: Rebalancing Disabled (swap_rebalancer_enable = False)

**Scenario**:
- Configuration: All rebalancing features disabled
- Portfolio: Normal, but no hedging activity

**Expected Behavior**:
- Projection runs (skips rebalancing logic)
- Zero swaps added
- KRD mismatch persists (not hedged)
- Goal-seek still runs (finds proportion but can't match via swaps)

**Validation**:
- [ ] Projection completes
- [ ] No swap activity
- [ ] Asset/liability KRD mismatch documented (not hedged)
- [ ] Output still reasonable (though non-optimal)

---

### Test E5.2: Goal-Seek Disabled (goal_seek_attempts_limit = 0)

**Scenario**:
- Configuration: Goal-seek disabled (attempts_limit_optimal = 0, use initial guess)
- Portfolio: Normal

**Expected Behavior**:
- Use initial proportion guess (e.g., 50%)
- No goal-seeking optimization
- Projection runs with fixed proportion
- May be suboptimal (SBA spread doesn't match)

**Validation**:
- [ ] Projection completes
- [ ] Uses fixed proportion throughout
- [ ] Output reasonable even if non-optimal
- [ ] No crashes from disabled feature

---

### Test E5.3: All Features Disabled

**Scenario**:
- Rebalancing: OFF
- Goal-seek: OFF (or iterations=0)
- Z-spread updates: OFF (or static)
- D&D: Applied (mandatory)

**Expected Behavior**:
- Minimal investment model
- Just apply D&D, value positions, report
- No optimization
- Very fast execution

**Validation**:
- [ ] Projection completes quickly
- [ ] D&D still applied
- [ ] Output minimal but valid
- [ ] Useful for debugging/validation baseline

---

## Category 6: Graceful Failures & Error Handling

### Test E6.1: Singular Matrix in Rebalancing Solver

**Scenario**:
- Rebalancing optimization: Linear algebra solver encounters singular matrix
- Cause: Degenerate portfolio (all assets perfectly correlated, etc.)

**Expected Behavior**:
- Solver detects singularity
- Error message: "Singular matrix in portfolio optimization"
- Fallback: Use no-rebalancing output or iterative solver
- Projection continues (or stops gracefully)

**Validation**:
- [ ] No crash (handled exception)
- [ ] Clear error message logged
- [ ] Reasonable fallback behavior
- [ ] User can debug (know what went wrong)

---

### Test E6.2: Convergence Timeout (Goal-Seek Takes Too Long)

**Scenario**:
- Difficult optimization landscape
- Goal-seek close but doesn't hit tolerance before iteration 20 limit

**Expected Behavior**:
- Return best_attempt (closest found)
- Flag: success = False or success_absolute = True (if absolute tolerance met)
- Log warning: "Goal-seek max iterations reached; using best attempt"
- Continue projection with best proportion found

**Validation**:
- [ ] Doesn't hang indefinitely
- [ ] Iteration limit respected ≤20 attempts)
- [ ] Best attempt used (not zeros or invalid)
- [ ] Warning logged for audit trail

---

### Test E6.3: File I/O Error (Missing Data File)

**Scenario**:
- Projection references data file that doesn't exist
- E.g., asset data, z-spread table, liability cashflows

**Expected Behavior**:
- Error detected at load time (before projection)
- Clear message: "File not found: [filename]"
- Graceful exit (no crash mid-iteration)
- Debugging information provided

**Validation**:
- [ ] Error caught early
- [ ] User-friendly message
- [ ] No partial execution/corrupted state
- [ ] Can retry with correct file

---

## Edge Case Test Execution Plan

**Execution Order** (suggests dependencies):

1. **Portfolio Extremes** (E1.1-E1.4): Parallel, independent
2. **Market Stress** (E2.1-E2.5): Parallel, independent
3. **Data Anomalies** (E3.1-E3.5): Sequential (build on previous)
4. **Numerical Extremes** (E4.1-E4.4): Parallel
5. **Configuration Extremes** (E5.1-E5.3): Sequential
6. **Error Handling** (E6.1-E6.3): Sequential (simulate failures)

**Total Tests**: 20 edge case scenarios

**Time Budget**: ~2-3 minutes total (most run fast via dryrun)

---

## Pass Criteria for Edge Case Tests

✅ **PASS** if:
- All 20 tests complete
- No crashes or unhandled exceptions
- Graceful fallbacks work as designed
- Clear error messages on failures
- Output reasonable (even if non-optimal)
- Numerical stability maintained

❌ **FAIL** if:
- Any test causes crash/exception
- Infinite loop or timeout
- Silent data corruption (wrong results without warning)
- Numerical instability (NaN propagation, overflow)

---

## Test Results Template

```
TEST: [E1.1] All-Bond Portfolio
STATUS: ✅ PASS
SCENARIO: 100% bonds, no equity, matched duration
EXPECTED: Goal-seek converges in 2-3 iterations; minimal rebalancing
ACTUAL: Converged in 3 iterations; 0 rebalancing iterations
NOTES: Works as expected; early exit in rebalancing loop

---

TEST: [E2.3] Extreme Rates (+400bps / -1%)
STATUS: ✅ PASS (with caveat)
SCENARIO: Base = 4.0%, Stress high = 5.0%, Stress low = -1.0%
EXPECTED: QuantLib handles extreme rates; NPV conversions stable
ACTUAL: Projection runs; -1% rate hits QuantLib boundary
NOTES: Warning message: "Rates below QuantLib minimum; clamped to -0.5%"
        → Output still valid, documented limitation

---

TEST: [E3.3] Callable Bond at Call Date
STATUS: ⚠️ CAVEAT
SCENARIO: Callable bond reaches call date in projection
EXPECTED: Bond called; removed from portfolio
ACTUAL: Wave-off logic exists but "not fully tested" per spec
NOTES: Feature implemented but requires full regression testing
        → Flag for future validation phase
```

---

## Summary: Edge Case Coverage

| Category | Tests | Coverage | Notes |
|----------|-------|----------|-------|
| Portfolio Extremes | 4 | High | All-bond, all-equity, single-class, ladder |
| Market Stress | 5 | High | Rates +/-100bps, negative, extreme, curve twist, spreads |
| Data Anomalies | 5 | High | Missing spreads, zero MV, callable bonds, liability EOL, negative spreads |
| Numerical | 4 | High | Tight tolerance, large portfolio, 101-year, floating-point precision |
| Configuration | 3 | Medium | Rebalancing off, goal-seek off, all off |
| Error Handling | 3 | Medium | Singular matrix, convergence timeout, file I/O error |
| **TOTAL** | **20** | **High** | Comprehensive coverage of edge cases |

