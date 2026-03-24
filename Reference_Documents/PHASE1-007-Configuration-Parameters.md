# 06-Configuration-Parameters.md

# CONFIGURATION & PARAMETERS MODULE

**File**: [settings_loader.py](../../Athora_BEL_Model_v0.1.8/site-packages/alm/model/loaders/settings_loader.py)

**Purpose**: Load all model parameters from Excel setup file and validate for consistency.

---

## Settings Dataclass (Master Config)

### Valuation & Projection
```
valuation_date: ql.Date           # t=0 date (e.g., 2023-12-31)
projection_term_years: int        # Horizon (e.g., 100)
reporting_currency: str           # EUR, USD, etc.
```

### SBA Goal Seeking
```
goal_seek_method: GoalSeekMethod  # Enum
  ├─ ReduceAllAssetsEqually
  └─ ReduceCashToSAAFirst

goal_seek_perform: bool           # Run optimization? (usually True)
goal_seek_tolerance_optimal: float    # €500k typical
goal_seek_tolerance_absolute: float   # €5M typical
goal_seek_attempts_min: int           # ≥5
goal_seek_attempts_limit_optimal: int # ≈10
goal_seek_attempts_limit_absolute: int # ≈20
enable_cache: bool                # Store intermediate results?
```

### SBA Z-Spread & Calibration
```
sba_zspread_calibration_method: ZSpreadCalibrationMethod
  ├─ CalibrateToAssetValues  (match liability to asset values)
  └─ UseRateProvided  (use fixed spread)

sba_zspread: float                # If UseRateProvided (e.g., 0.015 = 1.5%)
sba_spread_cap: float             # Max spread (e.g., 0.05 = 5%)
sba_minimise_liability_proportion_across_scenarios: bool
```

### Rebalancing
```
rebalance_enable: bool            # Run rebalancing? (usually False for SBA)
rebalance_taa_source: TargetAssetAllocationSource
  ├─ UseParameterInConfig  (SAA)
  ├─ ImplyFromExistingAssets  (CAA)
  └─ Most_Onerous  (min spread)

rebalancing_allowance_to_meet_taa: float  # Tolerance (e.g., 0.02 = 2%)
reinvestment_assets_max_duration_relative_to_liabilities: float  # e.g., 1.30
swap_rebalancer_enable: bool
swap_rebalancer_key_rates: list[int]  # [1, 3, 5, 10, 20, 30]
```

### Market Data Sheet Names
```
rates_sheet: str                  # "Rates"
sba_scenarios_sheet: str          # "SBA_Scenarios"
tickers_sheet: str                # "Tickers"
ticker_mapping_sheet: str         # "Ticker_Mapping"
swaption_vols_sheet: str          # "Swaption_Vols"
fx_rate_movement_sheet: str       # "FX_Movements"
```

### Liability Configuration
```
liability_sheet: str              # "Liability"
liability_column_sba_eligible: str   # Column with t=0..T CFs (SBA)
liability_column_std: str         # Column for standard valuation
liability_shock_sheet: str        # Optional shock multipliers
```

---

## Important Enums

```python
class GoalSeekMethod(Enum):
    ReduceAllAssetsEqually = auto()
    ReduceCashToSAAFirst = auto()

class ZSpreadCalibrationMethod(Enum):
    CalibrateToAssetValues = auto()
    UseRateProvided = auto()

class SpreadSource(Enum):
    ExistingAssetSpreadAtTimeZero = auto()  # Preferred
    PiT = auto()                           # Point-in-Time
    TTC = auto()                           # Through-the-Cycle

class TargetAssetAllocationSource(Enum):
    UseParameterInConfig = auto()      # SAA
    ImplyFromExistingAssets = auto()   # CAA
    Most_Onerous = auto()              # min(SAA, CAA) by spread

class BELValuationApproach(Enum):
    SBA = auto()                       # Use SIABS valuation
    Standard = auto()                  # Use corporate curve

class LiabilityDV01CalibrationMethod(Enum):
    DontCalibrateCloseToZero = auto()
    CalbrateCloseToDuration = auto()
```

---

## Loading Flow

```
STEP 1: Load from Excel Parameters sheet
  ├─ Key-value pairs: Parameter | Run_Name
  ├─ Convert types (int, float, bool, enum)
  └─ Populate Settings dataclass

STEP 2: Load Business Unit Parameters
  ├─ Which asset file to use
  ├─ SBA max liability proportion
  ├─ Valuation approach (SBA vs Standard)
  └─ Specific BU configuration

STEP 3: Load Market Data (if needed)
  ├─ Rates (base, stressed, corporate)
  ├─ Swaption vols
  ├─ FX movements
  └─ Liability cashflows

STEP 4: Apply Overrides (CLI --override)
  ├─ Override parameter=value from command line
  └─ Useful for sensitivity analysis

STEP 5: Validate
  ├─ Tolerances in hierarchy (optimal < absolute)
  ├─ Attempts in sequence (min < optimal < absolute)
  ├─ Asset file exists
  ├─ Rates defined for all currencies
  └─ Liability CFs not empty (for SBA)
```

---

## Typical Parameter Values

| Parameter | Typical Value | Range | Notes |
|-----------|---------------|-------|-------|
| projection_term_years | 100 | 1-150 | Longer for annuity portfolios |
| goal_seek_tolerance_optimal | €500,000 | €100k-€1M | Tighter = slower |
| goal_seek_tolerance_absolute | €5,000,000 | €2M-€10M | 10× wider than optimal |
| goal_seek_attempts_min | 5 | 3-10 | Before optimal accepted |
| goal_seek_attempts_limit_optimal | 10 | 8-15 | Before absolute fallback |
| goal_seek_attempts_limit_absolute | 20 | 15-30 | Hard stop |
| sba_spread_cap | 5% (0.05) | 2%-10% | Group-level max spread |
| reinvest_max_duration_relative | 1.30 | 1.1-1.5 | 30% buffer typical |
| rho_over | 0.93 | 0.90-0.95 | Fast reversion (spreads wide) |
| rho_under | 0.96 | 0.95-0.98 | Slow reversion (spreads tight) |

---

## Parameter Validation

```
VALIDATION CHECKS:

1. Goal Seek Cascade
   assert goal_seek_tolerance_absolute > goal_seek_tolerance_optimal
   assert goal_seek_attempts_min <= goal_seek_attempts_limit_optimal
   assert goal_seek_attempts_limit_optimal <= goal_seek_attempts_limit_absolute

2. Allocations
   assert SAA.sum() ≈ 1.00 (allow ±0.5%)
   
3. Asset File
   IF bel_valuation_approach == SBA:
     assert fp_assets is not None
     assert file_exists(fp_assets)

4. Rates
   assert all_currencies_in_asset_file have rate curves

5. Liability
   IF bel_valuation_approach == SBA:
     assert liability_cashflows is not None
     assert len(liability_cashflows) > 0
```

---

## Excel Sheet Structure (Example)

```
PARAMETERS SHEET:
┌──────────────────────────────────────────┬──────────────┐
│ Parameter Name                           │ Run_00_Base  │
├──────────────────────────────────────────┼──────────────┤
│ Valuation_Date                           │ 31/12/2023   │
│ Projection_Term_Years                    │ 100          │
│ Reporting_Currency                       │ EUR          │
│ SBA_GoalSeek_Perform                     │ TRUE         │
│ SBA_GoalSeek_Method                      │ ReduceAll    │
│ SBA_GoalSeek_Tolerance_Optimal           │ 500000       │
│ SBA_GoalSeek_Tolerance_Absolute          │ 5000000      │
│ SBA_GoalSeek_Attempts_Min                │ 5            │
│ SBA_GoalSeek_Attempts_Limit_Optimal      │ 10           │
│ SBA_GoalSeek_Attempts_Limit_Absolute     │ 20           │
│ SBA_ZSpread_Calibration_Method           │ CalibrateAss │
│ SBA_ZSpread_Cap                          │ 0.05         │
│ Rebalance_Enable                         │ FALSE        │
│ Rebalance_TAA_Source                     │ SAA          │
│ Reinvest_Max_Duration_Relative           │ 1.30         │
│ Output_BalanceSheet_Runoff               │ TRUE         │
│ Output_Portfolio_Runoff                  │ TRUE         │
│ Output_Trades                            │ FALSE        │
│ ... (50+ more rows)                      │ ...          │
└──────────────────────────────────────────┴──────────────┘

BU_PARAMETERS SHEET:
┌────────┬──────────────┬──────────────┬──────────────┐
│ BU     │ ParamSet     │ Valuation    │ AssetFile    │
├────────┼──────────────┼──────────────┼──────────────┤
│ ANL    │ 00_Base      │ SBA          │ ANL_Final... │
│ ABN    │ 00_Base      │ SBA          │ ABN_Final... │
│ AEL    │ 00_Base   │ Standard     │ AEL_Final... │
└────────┴──────────────┴──────────────┴──────────────┘
```

---

## CLI Override Example

```bash
# Override goal seek parameters for sensitivity test
python -m alm run-server --runs runs.txt \
  --override goal_seek_tolerance_optimal=1000000 \
             goal_seek_attempts_min=3

# This makes tolerance WIDER (€1M vs €500k)
# and allows fewer iterations before acceptance
# → Goal seek exits faster, less accurate
```

---

## Verification Checklist

- [ ] All goal seek parameters loaded from Parameters sheet
- [ ] Tolerances form correct cascade (optimal < absolute)
- [ ] Attempt limits in sequence (min < optimal < absolute)
- [ ] Asset file path resolved and file exists
- [ ] Market data (rates, vols, FX) loaded for all currencies
- [ ] Liability cashflows loaded (if SBA mode)
- [ ] Business unit parameters matched to run
- [ ] No conflicting flags (e.g., rebalance + goal seek)
- [ ] Output flags sensible (don't exceed memory)

---

## References

**Excel**: Setup workbook → Parameters sheet (rows 1-200)
**Code**: [settings_loader.py](../../Athora_BEL_Model_v0.1.8/site-packages/alm/model/loaders/settings_loader.py)
**User Guide**: Section 2.1 ("Setup Configuration")
