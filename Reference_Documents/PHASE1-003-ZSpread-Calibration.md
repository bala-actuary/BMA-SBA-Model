# 02-ZSpread-Calibration.md

# Z-SPREAD CALIBRATION MODULE - COMPREHENSIVE DEEP DIVE

**File**: [site-packages/alm/model/projection/zspread_provider.py](../../Athora_BEL_Model_v0.1.8/site-packages/alm/model/projection/zspread_provider.py)

**Purpose**: Manages z-spread (credit spread) extraction, calibration, and projection. Controls how liability cashflows accumulate over time based on reinvestment asset spreads.

---

## Table of Contents
1. Class Structure
2. Calibration Algorithm
3. Mean Reversion Formula (Exponential Decay)
4. Weighted Average Z-Spread Calculation
5. Spread Source Priority Logic
6. Integration with Goal Seeker
7. Projected Z-Spread (Time Evolution)
8. Indexing & Lookup Tables
9. Edge Cases & Bounds
10. Verification Checklist

---

## 1. Class Structure

```python
@dataclass
class ZSpreadProvider:
    """
    Manages z-spread calibration and projection.
    
    Z-spread (zero-volatility spread) = credit spread vs risk-free curve.
    Used to: (1) calibrate liability values at t=0, 
             (2) set liability accumulation rate per period.
    """
    
    settings: Settings
        # Global configuration (spread sources, calibration method, etc.)
    
    asset_param_provider: AssetParameterProvider
        # Lookup tables for asset parameters (TTC spreads, rho, etc.)
    
    dic_starting_spreads: dict[object, tuple[str, float]]
        # Cache of extracted spreads: {allocation_id: (name, spread_value)}
        # Populated via calibrate() at t=0
        # Example: {GroupID_123: ("Corporate Bonds", 0.0150)}
        # (0.0150 = 1.5% = 150 basis points)
    
    idx_of_individual_allocator: int
        # Index into asset_parameter_provider's lookup tables
        # Identifies which classification granularity to use
        # E.g., idx=0 → Asset class grouping (Corporate/Sovereign/Cash)
        #        idx=1 → Geographic grouping (EUR/USD/GBP)
        #        idx=2 → Liquidity grouping (Liquid/Illiquid)
    
    @property
    def is_calibrated(self) -> bool:
        """True once calibrate() has been called"""
        return self.idx_of_individual_allocator is not None
```

---

## 2. Calibration Algorithm

### Three T=0 Calibration Methods (User-Configurable)

**SPECIFICATION REFERENCE**: Section 3.3.8.9, Table "by Model Type"

The model supports three different approaches for calculating t=0 z-spreads on individual assets. The user specifies which method to use for each Model Type in the Excel alm_setup.xlsx file, under the **"by Model Type"** tab:

#### Method 1: **CalibrateToMarketPrice** (Goal-Seek Spread)

**When Used**: For bonds with known market prices (Fixed Rate Bond, Floating Rate Bond, Callable Bond, etc.)

**Algorithm**:
- Goal-seek the z-spread S such that:
  $PV(Asset CFs, IR_{t=0} + S) = Market\_Value$
- Uses QuantLib CashFlows.zSpread() to solve
- Result: Each asset gets a unique z-spread (calibrated to its MV)

**Advantage**: Precise asset-by-asset calibration
**Disadvantage**: Assumes market prices are accurate/representative

**Excel Config**: ea_spread_goal_seek_to_mv = "CalibrateToMarketPrice"

---

#### Method 2: **UseRateInAssetFile** (Provided Z-Spread)

**When Used**: For assets where z-spread is pre-calculated/externally provided

**Algorithm**:
- Read z-spread directly from asset datafile column (e.g., "ZSpread" or "CreditSpread")
- Use provided value without modification
- Updated per-period using same evolution as calibrated spreads

**Advantage**: Avoids goal-seek computation (faster)
**Disadvantage**: Relies on external data quality

**Excel Config**: ea_spread_goal_seek_to_mv = "UseRateInAssetFile"

---

#### Method 3: **UseRateInThisTable** (User Assumption)

**When Used**: For assets where market price unreliable or no individual z-spread available (e.g., Cash, Derivatives)

**Algorithm**:
- Use z-spread from assumption table (by Model Type)
- Per-period projection: applicable to all assets of this type

**Advantage**: Simple, consistent across asset group
**Disadvantage**: Less precise (uniform assumption applied)

**Excel Config**: ea_spread_goal_seek_to_mv = "UseRateInThisTable"
**Table Column**: ea_t0_zspread = [value, e.g., -0.10% for cash]

---

### Example Configuration (from Spec Lines 3.2.1.3)

| Model Type | Method | Value (if UseTable) | Projection |
|------------|--------|-------------------|-----------|
| FixedRateBond | CalibrateToMarketPrice | — | ScaleInlineWithAssetClass |
| FloatingRateBond | CalibrateToMarketPrice | — | ScaleInlineWithAssetClass |
| CallableFixedRateBond | CalibrateToMarketPrice | — | ScaleInlineWithAssetClass |
| VanillaSwap | UseRateInThisTable | 0.00% | ScaleInlineWithAssetClass |
| Swaption | UseRateInThisTable | 0.00% | ScaleInlineWithAssetClass |
| CashAccount | UseRateInThisTable | -0.10% | ScaleInlineWithAssetClass |
| RMLs (Residential Mortgages) | CalibrateToMarketPrice | — | ScaleInlineWithAssetClass |

### Main Calibration Method

```
PURPOSE: Extract z-spreads from existing asset portfolio at t=0.

Called ONCE at start of projection before any projections.

FUNCTION: calibrate(
    bs: BalanceSheet,
    md: MarketData
) -> None:

┌──────────────────────────────────────────────────────────────┐
│ STEP 1: Identify Lookup Table Granularity                   │
└──────────────────────────────────────────────────────────────┘

table_lookup = asset_param_provider.table_lookup

// Find which table defines reinvestment spread grouping
IF table_lookup.reinvestment_spread_short_term is defined:
  idx_allocator = table_lookup.reinvestment_spread_short_term
ELSE:
  idx_allocator = 0  // Default to first table

STORE: self.idx_of_individual_allocator = idx_allocator


┌──────────────────────────────────────────────────────────────┐
│ STEP 2: Group Assets by Allocation Class                    │
└──────────────────────────────────────────────────────────────┘

Groups = {}

FOR each asset IN bs.non_cash_assets:
  
  // Skip eligibility check - use all assets
  
  // Get allocation class identifier for this asset
  allocation_id = asset.params.tbl_idx[idx_allocator]
  // E.g., allocation_id = "Corporate_EUR"
  
  IF allocation_id NOT IN Groups:
    Groups[allocation_id] = {
        'name': asset_param_provider.get_name(allocation_id),
        'assets': []
    }
  
  Groups[allocation_id]['assets'].append(asset)


┌──────────────────────────────────────────────────────────────┐
│ STEP 3: Calculate Weighted Average Spread per Group         │
└──────────────────────────────────────────────────────────────┘

FOR each allocation_id, group IN Groups.items():
  
  // Filter out negative-valued assets (not real investments)
  assets_positive_mv = [a for a in group['assets'] 
                        IF a.value.market_value >= 0]
  
  IF len(assets_positive_mv) == 0:
    // No assets in group, skip
    CONTINUE
  
  // Extract spreads and weights
  spreads = [a.value.spread_asset for a in assets_positive_mv]
  // E.g., [0.015, 0.018, 0.016] = [150, 180, 160 bps]
  
  weights = [a.value.market_value for a in assets_positive_mv]
  // E.g., [€5M, €3M, €7M]
  
  // Calculate weighted average
  total_weight = SUM(weights)
  
  IF total_weight == 0:
    // Handle zero weight edge case
    wa_spread = AVERAGE(spreads)  // Equal weight fallback
  ELSE:
    wa_spread = SUM(spreads[i] * weights[i]) / total_weight
    // Weighted average
  
  // Validate
  IF wa_spread is NaN:
    ERROR: "Z-spread contains NaN for group " + allocation_id
  
  // Store
  dic_starting_spreads[allocation_id] = (
      group['name'],
      wa_spread
  )
  
  LOG(f"Group {allocation_id}: wa_spread = {wa_spread:.4f} "
      f"(from {len(assets_positive_mv)} assets)")


┌──────────────────────────────────────────────────────────────┐
│ STEP 4: Store for Later Use                                 │
└──────────────────────────────────────────────────────────────┘

// dic_starting_spreads is now populated and can be used in
// get_zspread_for_asset_class() for projection periods

LOG(f"Z-spread calibration complete: {len(dic_starting_spreads)} groups")
```

### Example Walkthrough

```
INPUT PORTFOLIO (at t=0):

Corporate Bonds:
  ├─ ISIN1: MV=€4M, Spread=200bps (2.0%)
  ├─ ISIN2: MV=€3M, Spread=180bps (1.8%)
  └─ negative MV position: -€0.1M, Spread=150bps (excluded)

Sovereign Bonds:
  ├─ BUND: MV=€5M, Spread=150bps (1.5%)
  ├─ OAT: MV=€2M, Spread=140bps (1.4%)
  └─ ASSI: MV=€3M, Spread=160bps (1.6%)

Swaps & Cash: (excluded from reinvestment cal)

CALCULATION:

Corporate:
  weights = [4, 3]
  spreads = [0.020, 0.018]
  wa_spread = (0.020*4 + 0.018*3) / (4+3)
            = (0.080 + 0.054) / 7
            = 0.134 / 7
            = 0.01914
            = 191.4 bps

Sovereign:
  weights = [5, 2, 3]
  spreads = [0.015, 0.014, 0.016]
  wa_spread = (0.015*5 + 0.014*2 + 0.016*3) / (5+2+3)
            = (0.075 + 0.028 + 0.048) / 10
            = 0.151 / 10
            = 0.0151
            = 151 bps

OUTPUT:

dic_starting_spreads = {
    'Corporate': ('Corporate Bonds', 0.01914),
    'Sovereign': ('Sovereign Bonds', 0.0151)
}
```

---

## 3. Mean Reversion Formula (Exponential Decay)

### The Formula

```
CORE FORMULA:

  spread(t) = L + (S - L) × ρ^t
  
  Where:
  - S = short_term_spread (at t=0, from calibrate())
  - L = long_term_spread (target as t→∞)
  - ρ = rho = reversion speed parameter (typically 0.93-0.97)
  - t = time in years (0, 1, 2, ..., 100)

INTERPRETATION:

  At t=0:   spread(0) = L + (S-L)*1 = S ✓
            (equals starting spread)
  
  At t=1:   spread(1) = L + (S-L)*ρ
            (partially reverted, weighted toward long-term)
  
  At t→∞:   spread(∞) = L + (S-L)*0 = L
            (converges to long-term average)

EXAMPLE:

  Scenario: Corporate spreads currently tight (100 bps vs 140 bps normal)
            S = 0.0100 (100 bps, tight)
            L = 0.0140 (140 bps, long-term average)
            ρ = 0.95 (5% reversion per year)
  
  t=0:    spread = 0.0140 + (0.0100 - 0.0140) × 0.95^0 
               = 0.0140 + (-0.0040) × 1
               = 0.0100 (100 bps)
  
  t=1:    spread = 0.0140 + (-0.0040) × 0.95
               = 0.0140 + (-0.0038)
               = 0.0102 (102 bps)
  
  t=5:    spread = 0.0140 + (-0.0040) × 0.95^5
               = 0.0140 + (-0.0040) × 0.7738
               = 0.0140 + (-0.00309)
               = 0.01091 (109.1 bps)
  
  t=10:   spread = 0.0140 + (-0.0040) × 0.95^10
               = 0.0140 + (-0.0040) × 0.5987
               = 0.0140 + (-0.00239)
               = 0.01161 (116.1 bps)
  
  t=20:   spread = 0.0140 + (-0.0040) × 0.95^20
               = 0.0140 + (-0.0040) × 0.3585
               = 0.0140 + (-0.00143)
               = 0.01257 (125.7 bps)
  
  t=100:  spread ≈ 0.0140 (converged to long-term)
```

### Two Speed Parameters

```
IF spread(0) > spread(∞):  // Starting spread is WIDE
  ρ = ρ_over  (typically 0.93-0.95)
  // Faster reversion (higher speed)
  // Premium reduces quicker

ELSE:  // Starting spread is TIGHT
  ρ = ρ_under  (typically 0.95-0.97)
  // Slower reversion (lower speed)
  // Premium builds back gradually

RATIONALE:
- Wide spreads more likely to tighten (market normalizes)
- Tight spreads less likely to widen (markets stable)
- Asymmetry reflects market dynamics
```

---

## 4. Weighted Average Z-Spread Calculation

### Algorithm

```
INPUT:
  assets_in_group: List[Asset]
  spreads: [s0, s1, s2, ...]  (z-spread values)
  weights: [w0, w1, w2, ...]  (market values)

PROCESS:

Step 1: Filter negative-valued positions
  assets_clean = [a for a in assets_in_group 
                  IF a.market_value >= 0]
  
  // Reasoning: Only real investments, not short positions

Step 2: Extract arrays
  spreads_clean = [a.spread_asset for a in assets_clean]
  weights_clean = [a.market_value for a in assets_clean]

Step 3: Calculate total weight
  total_weight = SUM(weights_clean)

Step 4: Handle zero weight case
  IF total_weight ≈ 0:
    wa_spread = AVERAGE(spreads_clean)
    // All assets weighted equally
  ELSE:
    wa_spread = DOT_PRODUCT(spreads_clean, weights_clean) / total_weight
    // Standard weighted average

Step 5: Validate result
  IF wa_spread is NaN:
    ERROR("NaN detected in z-spread")
  
  IF wa_spread < -0.10:  // -1000 bps
    CLIP wa_spread to -0.10
  
  IF wa_spread > 0.10:   // +1000 bps
    CLIP wa_spread to 0.10

OUTPUT:
  wa_spread: Representative spread for asset group

FORMULA:
  wa_spread = Σ(spreads_i × weights_i) / Σ(weights_i)
```

---

## 5. Spread Source Priority Logic

### Available Spread Sources

```python
class SpreadSource(Enum):
    ExistingAssetSpreadAtTimeZero = 1
        # Use weighted average extracted at t=0 (from calibrate())
        # Most realistic: reflects actual portfolio
    
    PiT = 2
        # Point-in-Time: current market spread
        # Dynamic: changes with market conditions
        # Source: asset_param.pit_spread
    
    TTC = 3
        # Through-the-Cycle: long-term structural average
        # Stable: anchors to average across cycles
        # Source: asset_param.ttc_spread
```

### Selection Logic

```
Priority order (checked in sequence):

1. TRY: ExistingAssetSpreadAtTimeZero
   IF allocation_id found IN dic_starting_spreads:
     USE: dic_starting_spreads[allocation_id][1]
     // Most preferred: actual portfolio spreads
   ELSE IF PiT available:
     USE: asset_param.pit_spread
     // Fall back to market
   ELSE:
     USE: asset_param.ttc_spread
     // Final fallback: long-term average

2. USE: PiT
   IF pit_spread is defined AND < 0.10:
     USE: pit_spread
   ELSE:
     USE: ttc_spread

3. USE: TTC
   ALWAYS available (calibration default)
```

### Mean Reversion Rate Selection

```
ALGORITHM:

  spread_short = spread_source_at_t0
  spread_long = spread_source_longterm
  
  IF spread_short > spread_long:
    // Spread is WIDE (premium)
    rho = rho_over  (e.g., 0.93)
    // Fast reversion: expect tightening
  ELSE IF spread_short <= spread_long:
    // Spread is TIGHT or at long-term
    rho = rho_under  (e.g., 0.96)
    // Slow reversion: expect gradual widening
  
  projected_spread(t) = spread_long + (spread_short - spread_long) * rho^t
```

---

## 6. Integration with Goal Seeker

### When Calibrate() is Called

```
Timeline:
  
  run_client(single scenario):
    ├─ initialise_settings()
    ├─ initialise_model()
    ├─ calibrate_model()
    │   └─ balance_sheet_projector.calibrate_all_asset_values()
    │       // Asset valuations at t=0, NOT z-spread calibration
    │
    └─ project_liability()
        └─ SBAGoalSeeker.perform_alm()
            ├─ Iteration 0: project_balance_sheet ...
            │   └─ _project_single_time_period(t=0)
            │       └─ IF MD.t == 0:
            │           zspread_provider.calibrate(bs, md)
            │           // FIRST call to calibrate()
            │
            ├─ Iteration 1: project_balance_sheet ...
            │   └─ Uses dic_starting_spreads (already cached)
            │
            └─ ... (iterations continue using cached values)
```

### Integration: Liability Accumulation Rate

```
EACH PROJECTION PERIOD (t > 0):

Step 1: Get eligible reinvestment assets
  reinvestment_assets = [
    a for a in bs.all_assets
    IF a.params.consider_in_taa 
       AND a.value_prev.spread_asset_class is not None
  ]

Step 2: Extract spreads and weights
  spreads = [a.value_prev.spread_asset_class for a in reinvestment_assets]
  weights = [a.value_prev.market_value for a in reinvestment_assets]

Step 3: Calculate weighted average reinvestment spread
  avg_reinvest_spread = weighted_average(spreads, weights)
  // Uses weighted average formula from Section 4

Step 4: Get z-spread for this period
  projected_spread = get_zspread_for_asset_class(
      asset_class = reinvestment_assets[0].params,
      t = current_period,
      spread_source = settings.sba_zspread_calibration_method
  )
  // Applies mean reversion formula

Step 5: Set liability accumulation rate
  accumulation_rate = CLAMP(
      projected_spread,
      min = -0.10,  // -1000 bps
      max = +0.10   // +1000 bps
  )
  
  bs.liability_sba.set_cashflow_accumulation_rate(accumulation_rate)
  // Liability PV grows at (1 + rate) per period

Step 6: Project liability with new rate
  asset_projector.project(bs.liability_sba, ...)
  // Liability value updated: PV_new = PV_prev × (1 + rate)
```

---

## 7. Projected Z-Spread (Time Evolution)

### Method: get_projected_asset_zspread()

```
FUNCTION: get_projected_asset_zspread(
    asset: Asset,
    zSpread_asset_t0: float,      # Current z-spread
    t: float                       # Time period
) -> float:

Purpose: Determines z-spread evolution strategy for individual asset.

Strategies:

  1. HoldConstant:
     projected_spread(t) = zSpread_asset_t0
     // Spread never changes
     
  2. ScaleInlineWithAssetClass:
     asset_class_spread_t0 = get_zspread_for_asset_class(
         asset.params, t=0)
     asset_class_spread_t = get_zspread_for_asset_class(
         asset.params, t=t)
     
     scale = asset_class_spread_t / asset_class_spread_t0
     projected_spread(t) = zSpread_asset_t0 * scale
     // Scale by asset class average change
```

---

## 8. Indexing & Lookup Tables

### Asset Parameter Lookup Tables

```
asset_param_provider.table_lookup is a multi-level classification system.

Each asset has:
  asset.params.tbl_idx = [idx_0, idx_1, idx_2, ...]
  
Where:
  tbl_idx[0] = Asset class identifier (e.g., "Corporate", "Sovereign")
  tbl_idx[1] = Geographic identifier (e.g., "EUR", "USD")
  tbl_idx[2] = Liquidity identifier (e.g., "Liquid", "Illiquid")
  ...

USAGE in calibrate():

  idx_of_individual_allocator = 0  // Use asset class table
  
  FOR each asset:
    allocation_id = asset.params.tbl_idx[idx_of_individual_allocator]
    // E.g., allocation_id = "Corporate"
    
    dic_starting_spreads[allocation_id] = wa_spread
```

### Example Table Structure

```
LOOKUP TABLE 0 (Asset Class):
┌────────────────┬──────────┬──────────┬──────────┐
│ Asset Class    │ PiT Spr. │ TTC Spr. │ Rho Over │
├────────────────┼──────────┼──────────┼──────────┤
│ Corporate      │ 1.50%    │ 1.80%    │ 0.93     │
│ Sovereign      │ 0.80%    │ 1.20%    │ 0.94     │
│ Cash           │ 0.05%    │ 0.10%    │ 0.95     │
└────────────────┴──────────┴──────────┴──────────┘

At t=0:
  Corporates in portfolio: [€4M @ 2.0%, €3M @ 1.8%]
  wa = 1.914%
  dic_starting_spreads['Corporate'] = 0.01914

At t=5 (projection):
  get_zspread_for_asset_class('Corporate', t=5)
  = 0.0180 + (0.01914 - 0.0180) × 0.93^5
  = 0.0180 + 0.00114 × 0.657
  = 0.0180 + 0.00075
  = 0.01875 (187.5 bps)
```

---

## 9. Edge Cases & Bounds

### Edge Case 1: Zero Weight in Group

```
Scenario: No assets in asset class

Handling:
  IF total_weight == 0:
    wa_spread = AVERAGE(spreads)  // Equal-weight fallback
  
  If no assets at all:
    dic_starting_spreads[class_id] = None
    // Can't calibrate, use default
```

### Edge Case 2: All Negative Market Values

```
Scenario: Short positions / margin accounts

Handling:
  Filter out negative MV positions
  Use only long positions for averaging
  
  If ALL positions are negative:
    Can't calibrate, use default spread
```

### Edge Case 3: Spread Explosion

```
Scenario: Spread becomes 500+ bps (unreal)

Bounds:
  wa_spread = CLAMP(wa_spread, [-0.10, +0.10])
  // Hard bounds: -1000 to +1000 bps
  
  projected_spread(t) = CLAMP(projected_spread(t), [-0.10, +0.10])
  // Each period
```

### Edge Case 4: Mean Reversion to Negative Spread

```
Scenario: Long-term spread is -50 bps (negative, unusual)

Handling:
  mean_reversion_formula = L + (S-L) × rho^t
  
  If L = -0.005, S = 0.010, rho = 0.95:
    spread(t) = -0.005 + 0.015 × rho^t
    // Still valid, but unusual
  
  Bounds check clips to [-0.10, +0.10]
```

---

## 10. Verification Checklist

When reviewing z-spread implementation:

- [ ] **Calibration**
  - [ ] calibrate() extracts spreads from existing portfolio ✓
  - [ ] Weighted average formula correct: Σ(s×w) / Σ(w)
  - [ ] Negative-MV positions excluded
  - [ ] Zero-weight groups handled (equal-weight fallback)
  - [ ] Results stored in dic_starting_spreads

- [ ] **Mean Reversion**
  - [ ] Formula: L + (S - L) × ρ^t applies ✓
  - [ ] ρ_over used when S > L (fast reversion)
  - [ ] ρ_under used when S ≤ L (slow reversion)
  - [ ] At t=0: spread(0) = S ✓
  - [ ] At t→∞: spread(∞) → L ✓

- [ ] **Spread Sources**
  - [ ] Existing asset spreads preferred (if calibrated)
  - [ ] Falls back to PiT if not calibrated
  - [ ] Falls back to TTC if PiT unavailable
  - [ ] Sources in Excel parameters point to correct cells

- [ ] **Integration**
  - [ ] Calibrate() called only once per scenario (t=0)
  - [ ] Results cached in dic_starting_spreads
  - [ ] Liability accumulation rate updated each period
  - [ ] Projected spread affects liability PV growth

- [ ] **Bounds & Validation**
  - [ ] Spreads clipped to [-10%, +10%] bounds
  - [ ] No NaN values in output
  - [ ] Weighted averages sum correctly

- [ ] **Cross-Check**
  - [ ] Compare calibrated spreads vs User Guide assumptions
  - [ ] Verify mean reversion rates (ρ) vs market expectations
  - [ ] Check liability PV growth vs reinvestment rate

---

## References

**Excel Parameters**: Setup workbook → Parameters sheet (rows 60-80, roughly)
**User Guide**: Section 2.2, "Z-Spread Calibration"
**Code**: [site-packages/alm/model/projection/zspread_provider.py](../../Athora_BEL_Model_v0.1.8/site-packages/alm/model/projection/zspread_provider.py)
