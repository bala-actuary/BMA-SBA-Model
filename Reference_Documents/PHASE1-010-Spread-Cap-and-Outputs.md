# SBA SPREAD CAP & OUTPUT REPORTING

## Overview

The Pythora model applies a regulatory **35 basis point (bps) cap** on the derived SBA spread, with specific recalculation logic when the uncapped spread exceeds the cap.

---

## Spread Cap Mechanism

### The 35bps Regulatory Cap

**Regulatory Requirement**: BMA-mandated spread cap of 35bps (see Spec Section 2.3.6.1)

**Purpose**: Ensures SBA spreads do not become unreasonably high, limiting potential BEL volatility across scenarios

**Current Setting**: 35bps (user-configurable via alm_setup.xlsx, Parameters tab: `sba_spread_cap`)

---

## Spread Calculation Process (Per Scenario)

### Step 1: Goal-Seek Uncapped Spread

For each SBA scenario:

1. **Run goal-seek**: Adjust asset/liability proportions until converged
2. **Calculate uncapped spread**: Goal-seek the spread S such that:

$$PV(LiabilityCFs, r_{RF} + S) = MV(Assets_{scaled})$$

Where:
- $r_{RF}$ = Risk-free rate (from projection)
- $S$ = Uncapped SBA spread
- Expected result: S varies by scenario (base, stress1, stress2, ..., stress8)

**Output at this stage**:
- Uncapped SBA Spread (by scenario)
- Uncapped SBA BEL = $MV(Assets_{scaled})$ (by scenario)

### Step 2: Check if Cap Applies

**Decision Logic**:
```
IF uncapped_spread > 35bps THEN
  cap_applies = TRUE
  capped_spread = 35bps
ELSE
  cap_applies = FALSE  
  capped_spread = uncapped_spread
END
```

### Step 3: Recalculate BEL with Capped Spread

If cap applies (uncapped > 35bps):

**Recalculate BEL**:
$$BEL_{capped} = PV(LiabilityCFs, r_{RF} + 35bps)$$

Where:
- Discount rate = Risk-free rate + 35bps (the cap)
- Result: Higher PV due to lower discount rate → higher BEL

**Economics**: Capping spread at 35bps means liabilities discounted less aggressively, resulting in larger liability values (more conservative)

---

## Output Reporting

### Results Per Scenario

For each Business Unit and SBA Scenario (0-8), report:

| Metric | Description | Calculation | Notes |
|--------|-------------|-------------|-------|
| **sba_spread_uncapped** | Uncapped SBA spread | Goal-seek result | Can be > 35bps |
| **sba_spread_capped** | Capped SBA spread | min(uncapped, 35bps) | Regulatory constraint |
| **sba_BEL_uncapped** | BEL with uncapped spread | $PV(CF, r + S_{uncapped})$ | Pre-cap value |
| **sba_BEL_uncapped_est_overstatement** | Overstatement due to tolerance | Residual asset surplus | BEL overstatement estimate |
| **sba_BEL_capped** | BEL with capped spread | $PV(CF, r + 35bps)$ | Post-cap value for reporting |
| **sba_avg_dis_rate_uncapped** | Average discount rate (uncapped) | Avg(r + S_uncapped) | Used for reporting |
| **sba_avg_dis_rate_capped** | Average discount rate (capped) | Avg(r + 35bps) | Used for reporting |

### Multi-Scenario Selection

The model runs all 9 SBA scenarios (1 base + 8 stressed):

```
For each scenario i in [0, 1, 2, ..., 8]:
  - Calculate uncapped_spread[i]
  - Calculate uncapped_BEL[i]  
  - Apply cap if needed
  - Calculate capped_BEL[i]
```

**Most Onerous**: The **highest capped BEL** across all 9 scenarios is selected as the regulatory SBA BEL.

### Fallback Logic (If Multiple Scenarios Binding the Cap)

If cap applies in multiple scenarios:
- Report which scenarios have cap binding
- Use highest capped BEL (already conservative)
- Document spread at cap (35bps) in output

### EPC Adjustment (ANL Only)

For Athora Netherlands, Employee Pension Contribution (EPC) is:
- **Not SBA-eligible**
- But included in overall liability for asset matching
- Post-cap, apply capped spread to EPC-excluding liabilities:

$$SBA\_BEL_{ex\_EPC} = PV(LiabilityCFs_{ex\_EPC}, r + min(S, 35bps))$$

---

## Code Implementation References

### Location in Pythora

- **Main orchestration**: `main_client.py` → `project_liability()`
- **Spread calculation**: `alm_projection.py` → `calculate_spread_and_yield()`
- **Cap application**: [TBD - to be verified in code review]
- **Output reporting**: `writer_excel.py` → `write_results()` Summary tab

### Expected Code Flow

```python
# Pseudocode from spec section 3.3.7

uncapped_spread = goal_seek_spread(liability_cfs, asset_mv, risk_free_curve)

IF uncapped_spread > 35bps:
    capped_spread = 35bps
    capped_bel = pv(liability_cfs, risk_free_curve + 0.0035)
ELSE:
    capped_spread = uncapped_spread
    capped_bel = asset_mv  # Already solved in goal-seek

report_results(uncapped_spread, uncapped_bel, capped_spread, capped_bel)
```

---

## Output Excel Reporting

### Excel Tab: "Summary"

Main output tab includes:

- Total uncapped BEL (sum across scenarios)
- Total capped BEL (sum across scenarios)
- Spread uncapped (scenario-specific)
- Spread capped (scenario-specific)
- Target asset allocation used (CAA vs SAA)
- Convergence status (success/not converged)

### Excel Tab: "BalanceSheet"

Period-by-period runoff showing:
- Asset market values
- Liability present values
- Cash flows by type
- Can verify that cap was applied by checking implied discount rate

---

## Per-Scenario Output Detail

Each output includes projection results broken down by:

- **By Business Unit**: ANL, AB, ARE
- **By Scenario**: SBA_0_Base, SBA_1_Up, SBA_2_Down, ..., SBA_8
- **By Asset Class**: Within each scenario, breakdown by asset type (if requested)

### Scenario Naming

```
SBA_0_Base    : Base case rates, no shock
SBA_1_Up      : +1% parallel yield shock
SBA_2_Down    : -1% parallel yield shock
SBA_3_Steep   : Steepening curve
SBA_4_Flat    : Flattening curve
SBA_5_Vol+    : Swaption vol +; rates +
SBA_6_Vol-    : Swaption vol -; rates -
SBA_7_Combo1  : Combination shock 1
SBA_8_Combo2  : Combination shock 2
```

---

## Verification Checklist

- [ ] **Cap Logic**: Confirm IF uncapped_spread > 35bps is checked per scenario
- [ ] **Recalculation**: Confirm BEL recalculated when cap applies (not just capped linearly)
- [ ] **Output Fields**: Confirm all output fields present (uncapped, capped, spread, BEL)
- [ ] **Most Onerous**: Confirm highest BEL selected across scenarios
- [ ] **EPC Adjustment**: For ANL, confirm spread applied to non-EPC liabilities post-cap
- [ ] **Scenario Loop**: Confirm cap checked for each of 9 scenarios
- [ ] **Overstatement Estimate**: Confirm BEL overstatement reported when convergence not perfect

---

## Spread Cap Sensitivity

### High-Level Impact Analysis

| Scenario Type | Conditions | Cap Binds? | Impact |
|---------------|------------|-----------|--------|
| Normal rates | Tight spreads, good asset quality | No | Uses true derived spread |
| Stressed rates | Wide spreads, downgrades | Often | BEL floor at 35bps rate |
| Poor asset quality | Widespread defaults | Likely | 35bps acts as conservative floor |

### Example Calculation

**Scenario**: Base case ANL
- Derived uncapped spread: 28bps → **No cap** (≤ 35bps)
  - Report: spread_uncapped=28bps, spread_capped=28bps, BEL uses 28bps
  
**Scenario**: Stress case 1 ANL
- Derived uncapped spread: 42bps → **Cap applies** (> 35bps)
  - Report: spread_uncapped=42bps, spread_capped=35bps
  - Recalculate: BEL = PV(CFs, RF + 35bps) = higher BEL (more conservative)

---

## Configuration & Parameters

### User-Editable in alm_setup.xlsx

- **Parameters tab**, column SBA_params:
  - `sba_spread_cap`: Default 35 (in basis points)
  - `sba_do_spread_cap`: TRUE/FALSE flag (enable/disable cap logic)

### Derived During Run

- Base scenario spread
- Each stressed scenario spread
- Minimum spread across scenarios (if requirement)
- Maximum spread across scenarios (most onerous)

---

## Related Output Metrics

### Bonus Fields in Summary Output

| Field | Purpose |
|-------|---------|
| sba_valuation_successful | TRUE if goal-seek converged |
| sba_asset_proportion | % of assets used (< 100% if surplus) |
| sba_liability_proportion | % of liabilities used (< 100% if insufficient assets) |
| sba_have_sufficient_liquid_assets | TRUE if no borrowing required |
| bel_uncapped | Total uncapped BEL (SBA + Standard) |
| bel_capped | Total capped BEL (SBA + Standard) - **THIS IS PRIMARY REPORTING** |

---

## References

- **Specification**: SBA BEL Model Implementation Specification v1.0.0
  - Section 2.3.6: SBA Specifics
  - Section 3.3.7: Goal-Seek SBA BEL
  - Section 3.4: Output Reports
- **Methodology**: SBA BEL Methodology Paper, Section 15 (Spread cap)
- **Regulation**: BMA Rules 2024

---

## Outstanding Questions for Code Review

- Q1: Where exactly is the 35bps cap applied? (Which method/file?)
- Q2: Is cap applied per scenario or across all scenarios?
- Q3: How is overstatement estimate calculated? (From residual asset surplus?)
- Q4: Is cap logic tested with scenarios that produce > 50bps spread?
- Q5: Does ANL EPC adjustment correctly exclude EPC from the valuation?

