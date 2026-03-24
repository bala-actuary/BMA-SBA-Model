# 12: Integration Test Scenarios - Multi-Component Workflows

Comprehensive integration tests validating multi-component workflows and data flow across SBA calculation pipeline.

---

## Integration Test Architecture

Integration tests verify that components work together correctly through complete calculation workflows:

```
External Input (Excel data)
    ↓
[Phase 1: Asset & Liability Loading]
    ↓
[Phase 2: Z-Spread Calibration] ← Component Test 5 verified ✓
    ↓
[Phase 3: SBA Goal-Seeking] ← Component Test 2 verified ✓
    |
    ├→ Portfolio projection loop
    |   ├→ Apply D&D defaults ← Component Test 1 verified ✓
    |   ├→ Calculate KRD liability ← Component Test 3 verified ✓
    |   └→ Rebalancing iterations ← Component Test 4 verified ✓
    |       └→ Update asset/liability positions
    |
    └→ Output: Final SBA proportion, hedges, projections
```

**Validation Points**: Data flows correctly through pipeline; components interact as expected

---

## Integration Test 1: 6-Month Dryrun Projection (Fastest)

**Scope**: Minimal projection (6 months) with all components active  
**Duration**: ~30 seconds total execution  
**Purpose**: Quick validation that components integrate; catch integration bugs early

### Scenario Setup

**Portfolio**:
- 3 corporate bonds: 5y, 10y, 20y maturities, 100-150bps spreads
- 2 equities: Fixed allocations
- 1 liability: 10-year level cashflows, $100M notional
- Configuration: All features enabled (KRD matching, rebalancing, z-spread updates)

### Execution Flow

```
T=0 (Initial):
  1. Load Excel data → Assets, liability, parameters
  2. Apply Z-Spread Calibration [Component 5]
     → Calibrate spreads for all bonds
  3. SBA Goal-Seeking [Component 2]
     → Find optimal asset/liability split (e.g., 60% assets, 40% liability)
  4. Store initial position (for later comparison)

T=1 (Month 1):
  1. Apply D&D defaults [Component 1]
     → Asset market values reduced by cumulative default
  2. Revalue all assets [Component 3 data input]
     → New market values, updated KRDs
  3. Update liability KRD [Component 3]
     → Revalue liability with new rates
  4. Calculate swap rebalancing targets
     → Target DV01 = updated asset KRD - liability KRD
  5. Execute rebalancing [Component 4]
     → Add/sell swaps up to 3 iterations per month (integration point)
  6. Update z-spreads [Component 5]
     → Mean reversion per month
  
T=2-6 (Months 2-6):
  Repeat T=1 for each remaining month

Final Output:
  - Asset & liability positions over 6 months
  - Hedging activity (swaps bought/sold)
  - Performance metrics (P&L, KRD movement)
  - Convergence metrics (rebalancing iterations, goal-seek attempts)
```

### Validation Checkpoints

| Checkpoint | What to Verify | Expected Result |
|-----------|----------------|-----------------|
| **After Z-Spread Calibration** | Spreads calibrated for each asset | z-spreads in 50-250bps range |
| **After Goal-Seek** | SBA proportion found | asset_value ≈ liability_value (converged) |
| **Month 1 D&D Applied** | Market values reduced | MV_after < MV_before (cumulative = 0.83%, so ~0.83% smaller) |
| **Month 1 KRD Recalc** | Liability KRD updated | New KRD reflects 1-month less duration |
| **Month 1 Rebalancing** | Swaps added if needed | Either 0-3 iterations executed, did_rebalance = True/False |
| **Month 1 Z-Spread Update** | Spreads moved toward TTC | z-spread changed per mean reversion formula |
| **All Months** | No crashes/exceptions | Projection completes successfully |

### Test Data Template

**Bonds**:
```
Name           Type             Maturity  Coupon  Spread  Annual_Default  Amount($M)
Bond_5y        CorporateBond    5y        3.5%    100bps  0.5%           20
Bond_10y       CorporateBond    10y       4.0%    150bps  1.0%           30
Bond_20y       CorporateBond    20y       4.5%    200bps  2.0%           15
```

**Liability**:
```
Tenor    Cashflow($M)  z-spread
1y       5.0           0bps (liability, no spread)
2y       5.0           0bps
3-10y    5.0           0bps
```

**Configuration**:
```
liability_dv01_calibration_method = Risk_Free ✓
swap_rebalancer_enable = True
rebalance_enable = True
goal_seek_attempts_optimal = 15
goal_seek_attempts_absolute = 20
goal_seek_tolerance_optimal = 0.001%
goal_seek_tolerance_absolute = 0.1%
```

### Expected Results

**Goal-Seek**: 
- Converges in 8-12 iterations
- SBA proportion ≈ 0.60 (60% assets, 40% liability)
- success_optimal = True or success_absolute = True

**D&D Impact**:
- Month 1: Market values reduced by ~0.83% (cumulative default from 5 bonds × 1 month)
- Month 6: Cumulative reduction ≈ 2.5%

**Rebalancing**:
- Iterations per month: 1-3 (most likely 1-2)
- Swap notional added: $5-15M depending on KRD drift

**Z-Spread Movement**:
- Spreads move toward TTC by ~10% per month (ρ=0.90)
- After 6 months: ~47% of the way from PiT to TTC

### Pass/Fail Criteria

✅ **PASS** if:
- Projection completes without crashes
- All components executed (D&D, KRD, rebalancing, z-spreads)
- Goal-seek converged (success = True)
- Output numbers are reasonable (positive MV, finite durations, etc.)
- Convergence smooth (no sudden jumps in metrics)

❌ **FAIL** if:
- Projection crashes (exception thrown)
- NaN or Inf values in output
- Goal-seek unconverged (success = False) after 20 attempts
- Rebalancing iterations > 3 in any month
- DV01 diverges instead of converges

---

## Integration Test 2: 1-Year Dryrun Projection (Standard)

**Scope**: Full 1-year projection (12 periods) with all components  
**Duration**: ~1 minute total execution  
**Purpose**: Standard validation; detect cycle-to-cycle integration issues

### Extended Scenario

Build on Test 1 with additional complexity:
- Larger portfolio: 8 bonds, 3 equities, 1 currency forward
- More complex liability: Nonlevel cashflows (increasing over time)
- Stress scenarios: Small rate/spread shocks mid-year

### Validation Checkpoints (Monthly)

Month 1-12:
- [ ] D&D defaults apply correctly (cumulative grows)
- [ ] KRD updates per period (duration decreases)
- [ ] Rebalancing executes (0-3 iterations)
- [ ] Z-spread converges (moves toward TTC each month)
- [ ] No reversals in convergence (monotonic improvement)
- [ ] Cash balance stays positive (no negative liquidity)

### Year-End Validation

- [ ] Total cumulative defaults: Calculated per formula (should be 1-(1-X_annual)^12)
- [ ] Total hedging activity: Track swap notional changes
- [ ] P&L: Asset/liability gains/losses realistic
- [ ] Portfolio composition: Rebalancing maintained target allocations (roughly)

### Pass/Fail Criteria

Same as Test 1, applied to 12 months:
- Projection completes
- No numerical errors
- Goal-seek conversions each period (if recalculated)
- All components active and integrated

---

## Integration Test 3: Full 101-Year Projection (Validation Only)

**Scope**: Complete projection (101 periods) — longest scenario  
**Duration**: ~5-10 minutes (or use dryrun mode for ~1 min)  
**Purpose**: Full lifecycle validation; verify long-term convergence

### Key Milestones

| Time Horizon | Validation Point | Expected Behavior |
|--------------|-----------------|-------------------|
| **Year 1-5** | Early convergence | D&D accumulation 5%, z-spreads 50% → TTC, stable rebalancing |
| **Year 10** | Mid-term trends | D&D ≈ 49%, most assets mature (5-10y bonds), z-spreads near TTC |
| **Year 30** | Long-term behavior | D&D ≈ 74%, portfolio mostly equity/cash, minimal rebalancing |
| **Year 50+** | Tail behavior | D&D → 92%+, assets nearly eliminated, liability runoff complete |
| **Year 101** | Terminal state | Portfolio minimal; projection stabilizes |

### Validation Steps (Sampled)

- [ ] Year 1: Compare to full 1-year test (should be identical)
- [ ] Year 10: D&D cumulative ≈ 1-(1-0.01)^10 ≈ 9.5% (for 1% annual default assets)
- [ ] Year 30: Asset allocation shifted (rebalancing accumulated)
- [ ] Year 50: Convergence complete (further periods minimal change)
- [ ] Year 101: Projection stable (no errors in tail)

### Long-Horizon Risks to Check

- [ ] Numerical stability: No precision loss over 101 years
- [ ] Memory usage: Projection doesn't explode (typical: 100MB-500MB)
- [ ] Matrix operations: Swap rebalancing LP-solver remains stable

### Pass Criteria

✅ **PASS** if:
- Full 101-year projection completes
- Intermediate years (1, 10, 30, 50, 100) reasonable
- No numerical degradation in tail years
- Final state makes sense (low MV, runoff complete, etc.)

❌ **FAIL** if:
- Crash before year 101
- NaN/Inf values appear in later years
- Matrix solver fails (singular matrix, etc.)
- Memory exhaustion

---

## Integration Test 4: Component Interaction — D&D → Goal-Seek → Rebalancing

**Scope**: Trace data flow through three-component sequence  
**Purpose**: Verify specific interactions between D&D, goal-seek, and rebalancing

### Scenario: Callable Bond Portfolio

**Setup**:
- Portfolio includes callable bonds (added complexity for D&D + rebalancing)
- Default rates: 2% annual
- Callable bonds: May be called early if rates fall

### Execution Trace

```
Month 1: Initial Setup
  Asset 1: Callable Bond, MV=$100M, Duration=7y, Default Rate=2%
  Asset 2: Regular Bond, MV=$50M, Duration=5y, Default Rate=1%
  Liability: Duration=6.5y
  Goal: Find asset/liability split for DV01 matching

Step 1: D&D Applies
  Asset 1 cumulative = 1 - (1-0.02)^1 = 0.02 = 2%
  Asset 1 MV after = $100M × (1-0.02) = $98M ✓
  Asset 2 cumulative = 1 - (1-0.01)^1 = 0.01 = 1%
  Asset 2 MV after = $50M × (1-0.01) = $49.5M ✓

Step 2: Asset KRD Recalculated
  Callable bond DV01 reduced (2% less MV)
  Regular bond DV01 reduced (1% less MV)

Step 3: Goal-Seek Searches for New Proportion
  Previous proportion: 70% assets, 30% liability
  New calculation needed due to MV changes
  Try: 68% assets, 32% liability
  → Check: KRD(0.68 × assets) ≈ KRD(liability)?
  → If yes, converged; if no, iterate

Step 4: Rebalancing Adjusts Hedge
  New proportion might require different hedges
  Swap rebalancer calculates new DV01 targets
  Physical rebalancer adjusts allocations (if needed)
```

### Validation Checkpoints

| Point | What Changes | How to Verify |
|-------|-------------|---|
| After D&D | Asset MVs reduced | MV_month1 < MV_month0 |
| After KRD Recalc | Liability duration shorter by 1/12 | Duration_month1 = Duration_month0 - 1/12 |
| After Goal-Seek | Proportion may change | Check convergence: value_asset ≈ value_liability |
| After Rebalancing | Swap positions updated | Swap notional changed by N units |

### Pass Criteria

✅ **PASS** if:
- D&D reduces MV by expected amount
- Liability KRD decreases monotonically
- Goal-seek converges with new proportion
- Rebalancing adjusts proportionally
- No data corruption between steps

---

## Integration Test 5: Cross-Period Consistency Check

**Scope**: Verify data consistency across period boundaries  
**Purpose**: Catch end-of-period/start-of-period state errors

### Scenario: 3-Month Projection

**Validation**:

```
Period 1 Output:
  end_-of_period_assets_mv = $148.5M
  end_of_period_swaps = Long 2Y swap, notional $10M
  end_of_period_zspread = 125bps (moved from 200bps initial)

Period 2 Input:
  beginning_of_period_assets_mv = ?
  Should be: $148.5M (to Period 2 opening = Period 1 closing)
  
  Validate: Period2_start_MV == Period1_end_MV ✓

  beginning_of_period_swaps = ?
  Should be: Long 2Y swap, notional $10M (carried forward)
  
  Validate: Period2_swaps_inherited == Period1_swaps ✓

  beginning_of_period_zspread = ?
  Should be: 125bps (continued from Period 1 end)
  
  Validate: Period2_start_zspread == Period1_end_zspread ✓
```

### Pass Criteria

✅ **PASS** if:
- All state variables carry over correctly (MV, swaps, z-spreads)
- No state resets between periods
- Portfolio identity maintained (same assets, updated values)

❌ **FAIL** if:
- State mismatch (Period N+1 opening ≠ Period N closing)
- Missing/duplicated transactions
- Asset IDs don't match period-to-period

---

## Integration Test 6: Stress Scenario Propagation

**Scope**: Verify stress scenarios impact all components correctly  
**Purpose**: Validate that stress rates/spreads propagate through pipeline

### Scenario: Interest Rate Shock

**Setup**:
- Base scenario: Flat 3% rates
- Stress scenario: +100bps rates for all maturities
- 8 stress scenarios total

### Propagation Trace

```
Stress Scenario | Rates Parallel Shift
  ↓
Asset Revaluation [Component 1 input]
  → Lower MV (rates up, bond values down)
  ↓
KRD Calculation [Component 3]
  → Calculate DV01 under stress rates
  → stress_impact["+100bps"] = (npv_stressed - npv_base) × scale
  ↓
Swap Rebalancing [Component 4]
  → Recalculate hedge sizes under stress
  → DV01 targets adjusted per stress scenario
  ↓
Goal-Seek [Component 2]
  → Re-optimize for each stress scenario
  → May find different optimal proportions
  ↓
Output Reports
  → Present most onerous scenario (max capital)
```

### Validation Checkpoints

| Step | Validation |
|------|-----------|
| Asset P&L | Rates +100bps → Bond MV down ~7% (example), Equity MV stable |
| KRD Updates | Stress impact = (lower NPV) - (base NPV) < 0 ✓ |
| Rebalancing | Hedge sizes may increase (higher sensitivity to rates) |
| Goal-Seek | Can find proportion for each stress scenario |
| Report | Most onerous scenario selected |

### Pass Criteria

✅ **PASS** if:
- Stress rates correctly propagated
- All components respond (P&L, KRDs, rebalancing)
- Most onerous scenario identified correctly
- No crashes under stress

---

## Integration Test 7: Z-Spread Mean Reversion During Projection

**Scope**: Verify z-spreads evolve correctly over projections  
**Purpose**: Ensure mean reversion formula applies correctly period-by-period

### Scenario

**Setup**:
- Initial z-spread: 200bps (calibrated from market)
- TTC (long-term): 100bps
- Rho: 0.90 (90% of current spread retained each month)
- Projection: 12 months

### Expected Mean Reversion Path

```
Formula: z(t) = 100 + (200 - 100) × (0.90)^t

Starting:  z(0) = 200bps
Month 1:   z(1) = 100 + 100 × 0.90 = 190bps (10bps tighter)
Month 2:   z(2) = 100 + 100 × 0.81 = 181bps (9bps tighter)
Month 3:   z(3) = 100 + 100 × 0.73 = 173bps (8bps tighter)
...
Month 12:  z(12) = 100 + 100 × 0.282 = 128.2bps (converging)
```

### Validation

- [ ] Extract z-spread for each asset at months 0, 1, 2, 3, 12
- [ ] Compare to hand-calculated values (above)
- [ ] All within ±2bps tolerance
- [ ] Monotonic convergence (no reversals)

### Impact on Other Components

**Effect on Asset Valuations**:
- Spreads tighten → Asset MVs increase slightly
- Affects D&D scaling (less aggressive on increasing MV)

**Effect on KRDs**:
- Different spreads → Different duration impacts
- KRDs may shift as spreads normalize

**Validation**:
- [ ] KRD movement consistent with z-spread movement
- [ ] Rebalancing targets adjust for KRD changes
- [ ] Goal-seek may re-converge with different spreads

### Pass Criteria

✅ **PASS** if:
- Z-spreads follow mean reversion formula
- Convergence smooth and correct
- Other components respond appropriately
- Asset MVs, KRDs updated consistently

---

## Integration Test 8: Callable Bond Wave-Off (If Applicable)

**Scope**: Verify callable bond option exercise during projection  
**Purpose**: Validate callable bond component interacts with other features

### Scenario

**Setup**:
- Callable bond: 10y maturity, callable from year 5, strike 104
- Market rates: 3% initially, fall to 2% (making option in-the-money)
- Portfolio: Mix with regular bonds

### Projection Path

```
Years 1-4:
  - Bond is not callable (before call date)
  - Treated as regular bond
  - D&D applies normally
  - KRD calculated with full duration

Year 5+:
  - Bond becomes callable
  - If rates have fallen, option exercised (issuer calls bond)
  - Bond removed from portfolio
  - Cash proceeds reinvested
  - D&D no longer applies to this bond
```

### Validation Checkpoints

- [ ] Years 1-4: Bond in portfolio, duration ≥ 9y
- [ ] Year 5: Bond callable; check if in-the-money
  - If yes: Bond called, MV = 104, removed
  - If no: Bond stays in portfolio
- [ ] Post-call: Portfolio rebalanced, goal-seek recalculates
- [ ] No crashes from missing bond

### Pass Criteria

✅ **PASS** if:
- Callable bond wave-off handled correctly
- Portfolio updated after call
- Rebalancing adjusts for missing bond
- No numerical errors

**Note**: This test applies IF callable bonds are in test portfolio; spec notes validation incomplete

---

## Integration Test Summary

| Test # | Scenario | Duration | Components | Purpose |
|--------|----------|----------|-----------|---------|
| 1 | 6-month dryrun | 30 sec | All 5 | Quick smoke test |
| 2 | 1-year dryrun | 1 min | All 5 | Standard validation |
| 3 | 101-year full | ~5-10 min | All 5 | Full lifecycle, long-term behavior |
| 4 | D&D→Goal-Seek→Rebal | Variable | 1,2,4 | Component interaction |
| 5 | Cross-period consistency | Variable | All | State management |
| 6 | Stress scenario propagation | Variable | All | Stress impact flow |
| 7 | Z-spread mean reversion | 12 months | 5 + impact | Spread dynamics |
| 8 | Callable bond wave-off | Variable | 1,4 + callable | Option exercise handling |

---

## Success Metrics for Integration Testing

**Phase 3 Completion Criteria** (Integration Tests):

- [ ] Test 1 (6-month): ✅ PASS
- [ ] Test 2 (1-year): ✅ PASS
- [ ] Test 3 (101-year): ✅ PASS
- [ ] Test 4-8: ✅ PASS (or documented caveat if callable bonds not fully tested)
- [ ] All data flows correctly between components
- [ ] No numerical errors or crashes
- [ ] Stress scenarios work as expected

**Pass Threshold**: All 8 tests passing → Integration validation complete → Proceed to Phase 3 test 13 (edge cases)

