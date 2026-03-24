# 05-Projection-Valuation.md

# MAIN PROJECTION & VALUATION LOOP

**Files**:
- [balance_sheet_projection.py](../../Athora_BEL_Model_v0.1.8/site-packages/alm/model/projection/balance_sheet_projection.py)
- [market_projection.py](../../Athora_BEL_Model_v0.1.8/site-packages/alm/model/projection/market_projection.py)

**Purpose**: Run full 100-year projection with annual market evolution, asset revaluation, liability growth, and rebalancing.

---

## Architecture

```
for t in 0..100:
  ├─ Evolve market data (rates, FX, vols)
  ├─ Revalue assets at new market conditions
  ├─ Accrue liability cashflows
  ├─ Rebalance portfolio (if triggered)
  ├─ Apply margin calls (derivatives)
  └─ Record snapshot
```

---

## Per-Period Projection Steps (Detailed)

**SPEC REFERENCE**: Section 3.3.6, method `_project_single_time_period()`

```
STEP 1: Move to Beginning of Period
  ├─ Clear trades from previous period
  ├─ Remove matured assets (bonds reaching maturity)
  └─ Reset period-level accumulators

STEP 2: Project Assets Forward
  FOR each asset in portfolio:
    ├─ Revalue at end-of-period market rates
    ├─ Calculate defaults & downgrades reduction
    ├─ Calculate asset PnL (MV change + income)
    ├─ Split cashflows: liquid vs illiquid
    └─ Apply scalars (defaults, FX, SBA proportion)

STEP 3: Update Cash Account
  ├─ Add asset coupons and redemptions
  ├─ Add reinvestment asset cashflows
  ├─ Subtract liability outflows (net of premiums)
  ├─ Subtract investment management expenses (IMEs)
  └─ Net result: Cash change for period

STEP 4: Update Liability Rate (Per-Period)
  ├─ Calculate weighted average reinvestment spread (by asset class)
  ├─ Set liability accrue rate = RF + spread
  └─ Project liability CFs with rate applied

STEP 5: Rebalancing Loops (Iterate Up to 3 Times)
  ├─ Iteration Loop:
  │   ├─ PHASE A: Physical Asset Rebalancing
  │   │   ├─ Identify over/under-weighted classes
  │   │   ├─ SELL from over-weighted (liquid assets)
  │   │   ├─ BUY for under-weighted (per allocation profile)
  │   │   ├─ Update cash for trades and transaction costs
  │   │   └─ Enforce cash ≥ 0 constraint
  │   │
  │   ├─ PHASE B: Derivative Rebalancing
  │   │   ├─ Calculate target KRDs per liability (risk-free only)
  │   │   ├─ Calculate current asset KRDs
  │   │   ├─ Determine swaps needed to close gaps
  │   │   ├─ Purchase payer/receiver IR swaps
  │   │   ├─ Capitalize bid-offer as t0 cash outflow
  │   │   └─ Apply margin calls (swap collateral)
  │   │
  │   ├─ Check: Converged? (targets met) → EXIT loop
  │   └─ Else: NEXT iteration (max 3 total)
  │
  └─ Final: Enforce cash ≥ 0 (sell if needed)

STEP 6: Record Period Snapshot
  ├─ Balance sheet values (assets, liabilities, net)
  ├─ Cashflow detail (by type)
  ├─ Rebalancing trades (if any)
  ├─ Margin calls (if any)
  └─ Duration/spread metrics
```

---

## Per-Period Projection (7 Steps)

```
STEP 1: Initialize QuantLib
  Set evaluation date = valuation_date_t

STEP 2: Beginning of Period
  Record starting market values

STEP 3: Project Assets
  FOR each asset:
    ├─ Accrue coupons/interest
    ├─ Realize matured cashflows
    ├─ Update MV to t+1 market rates
    ├─ Calculate DV01 stress impacts
    └─ Remove if matured

STEP 4: t=0 Calibration (only at start)
  ├─ zspread_provider.calibrate()
  ├─ liability_sba.calibrate_zspread()
  └─ Clear t=0 liability value

STEP 5: Update Liability Rate
  ├─ Calculate weighted avg reinvestment spread
  ├─ Set liability accumulation rate
  └─ Project liability with rate

STEP 6: Collect Cashflows
  ├─ Asset coupons → cash
  ├─ Liability payments ← cash
  └─ Investment management expenses

STEP 7: Rebalancing Loops (up to 3 iterations)
  ├─ Swap rebalancing (if enabled)
  ├─ Physical asset rebalancing (if enabled)
  └─ Repeat until convergence

STEP 8: Margin Calls
  Move derivative margin to cash/collateral
```

---

## Market Projection

```
INPUT: Setup rates (provided in Excel)
  ├─ rates_provided_mc_base: Market-consistent rates
  ├─ rates_provided_std: Corporate curve
  └─ rates_provided_mc_stress: Shocked rates for SBA scenario

OUTPUT: MarketData for each period t

PROCESS:
  1. Construct rate curves (enable extrapolation for long end)
  2. Build swaption vol surfaces (interpolate matrix)
  3. Apply FX movements (scalars per currency)
  4. Calibrate Hull-White model (for swaption pricing)
  5. Return complete market snapshot
```

---

## Calibration (One-Time at t=0)

```
calibrate_all_asset_values(balance_sheet, md_initial):
  FOR each asset:
    asset_projector.calibrate(asset, md_initial)
    ├─ Compute PV using QuantLib engine
    ├─ Duration, convexity
    ├─ DV01 at key rates
    └─ Store in asset.value

apply_t0_stresses(balance_sheet):
  FOR each asset:
    ├─ Mark for key-rate DV01 calculation
    └─ Pre-compute stress impacts
```

---

## Projection Results

```
ProjectionResultsLiability:
  ├─ asset_proportion: float (e.g., 0.974)
  ├─ sba_liability_proportion: float (e.g., 0.753)
  ├─ balance_sheet_runoff: DataFrame
  │   ├─ Period | Asset_Value | Liability_PV | Net | Cash_Min
  │   └─ 101 rows (t=0..100 years)
  ├─ portfolio_runoff: DataFrame
  │   ├─ Period | ISIN | Asset_Type | MV_Start | MV_End | DV01
  │   └─ ~101,000 rows (101 periods × 1000 assets)
  └─ sba_convergence_metrics:
      ├─ Converged: bool
      ├─ Attempts: int
      ├─ Value_Net: float
      └─ Best_Tolerance_Met: ['Optimal', 'Absolute', 'None']
```

---

## Cashflow Flows

```
Asset Cashflows → Balance Sheet:
  ├─ Coupon accrual: +€2,500 (bond coupon paid)
  ├─ Principal matured: +€1,000,000 (bullet payment)
  └─ Market value change: Reflected in BS value

Liability Cashflows → Balance Sheet:
  ├─ Policy claims: -€500,000 (payment to policyholders)
  ├─ Accumulation: +€50,000 (reinvestment of proceeds)
  └─ Liability PV: Grows at (1 + accumulation_rate)

Cash Account:
  cash_new = cash_old + ΔAsset_CF - ΔLiab_CF - Exp
```

---

## Stress Application (Key-Rate DV01)

```
FOR each key rate term (1, 3, 5, 10, 20, 30 years):
  ├─ Create shock: +1bp at that term
  ├─ Project market with shock
  ├─ Compare asset values
  └─ Store DV01_i = (MV_shock - MV_base) / 1bp

Used for: Swap rebalancing risk calculation
```

---

## Edge Cases

```
Asset Explosion:
  IF asset_count > prev * 3 OR growth > 2000:
    TERMINATE early (use best results so far)

Negative Cash:
  IF cash < -1€:
    REJECT attempt (goal seeker tries different proportion)

NaN Detection:
  IF market_value or PV is NaN:
    ERROR (numerical issue)

Negative Market Value:
  IF bond MV becomes negative after stress:
    CLIP to 0 (protective floor)
```

---

## Performance

| Metric | Value |
|--------|-------|
| Time per projection | 5-10 seconds |
| Periods (t) | 101 (0-100 years) |
| Assets per portfolio | 1000 typical |
| Memory per result | 50-100 MB |
| Goal seek iterations | 8 typical (goal seek × projections) |
| Total run time | 40 seconds nominal (1-5 minutes worst case) |

---

## Verification Checklist

- [ ] Projection runs for 101 periods (t=0..100)
- [ ] Market data evolves (rates move each period)
- [ ] Assets mature and are removed
- [ ] Liability balloons over time
- [ ] Rebalancing triggered at configured periods
- [ ] Cash remains positive until near end-of-life
- [ ] No NaN or negative market values
- [ ] DV01 calculations make sense (duration-like)
- [ ] Results exportable to Excel without errors

---

## References

**Code**: [balance_sheet_projection.py](../../Athora_BEL_Model_v0.1.8/site-packages/alm/model/projection/balance_sheet_projection.py)
**User Guide**: Section 3 ("Valuation Engine")
