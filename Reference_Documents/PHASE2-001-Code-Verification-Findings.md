# Phase 2: Code Verification Findings

## CRITICAL FINDING: Liability KRD Calibration Method

**Status**: ⚠️ REQUIRES CONFIGURATION VERIFICATION

### Code Location
- **File**: [liability_cashflow.py](liability_cashflow.py) lines 92-97
- **Config Enum**: LiabilityDV01CalibrationMethod (setting_enums.py line 25)

### Implementation Details
The code supports TWO liability KRD calculation methods:

```python
if self.s.liability_dv01_calibration_method == LiabilityDV01CalibrationMethod.Risk_Free_plus_SBA_Spread:
    zspread_t_dv01 = zspread_t  # INCLUDES SBA spread
elif self.s.liability_dv01_calibration_method == LiabilityDV01CalibrationMethod.Risk_Free:
    zspread_t_dv01 = 0.0  # RISK-FREE ONLY ✓
else:
    raise NotImplementedError()

# Then used for DV01 calculation:
npv_dv01_base = ql.CashFlows.npv(self.leg, rates_base, zspread_t_dv01, self.day_counter, ...)
# For stress scenarios:
npv_stress = ql.CashFlows.npv(self.leg, rates_stress, zspread_t_dv01, self.day_counter, ...)
market_value_stress_impact[stress.name] = (npv_stress - npv_dv01_base) * stress.scale
```

### Specification Requirement
**Spec Section 2.3.5.3**: "The Key Rate Duration (KRD) for liabilities is calculated using risk-free rates only"
- This means: `LiabilityDV01CalibrationMethod.Risk_Free` must be configured

### Integration Path
Flow: swap_rebalancer.py line 240
```python
liability_dv01 = 0 if liability_sba is None else liability_sba.value.market_value_stress_impact[name]
```
The liability_sba.value.market_value_stress_impact is populated by liability_cashflow.py _project() method using the configured KRD method.

### Rebalancing Dependency
The liability KRD value flows directly into swap rebalancing at:
- **swap_rebalancer.py** line 170+: `get_target_asset_dv01()` method
  - Uses liability KRD to calculate target asset DV01 for swap matching
  - Three DV01 projection methods available (LiabilityDV01_ByKeyRate, LiabilityDV01_Total, LiabilityDV01_ByKeyRateAndTotal)
  - Each method scales the net DV01 target based on liability KRD changes

### Verification Status
**NEXT STEP**: Must verify which method is configured in alm_setup.xlsx
- ✅ Code correctly implements both methods
- ✅ Risk_Free method correctly zeroes spread
- ⚠️ CRITICAL: Excel configuration must use Risk_Free method for spec compliance

---

## Verified Findings

### 1. D&D Formula Implementation ✅
**Status**: MATCHES SPECIFICATION

**File**: asset_projector.py lines 161-190
**Formula**: `closing_cumulative_default = 1 - (1 - asset.params.annual_default)^term_since_inception`

**Application**:
- Calculated once per asset per period
- Applied as scalar multiplier: `scalar_default = (1 - cumulative_default)`
- Multiplied into: `au.multiply(scalar_defaults = scalar_default)`
- Applied to BOTH market value AND cashflows

**Assertions** (lines 182-186):
```python
assert 0 <= closing_cumulative_default <= 1
assert math.isfinite(market_value_after_default)
```

**Verification Result**: ✅ Exact match to spec formula (1-X)^t, properly applied

**Spec Reference**: Section 3.3.8.11

---

### 2. Goal-Seek Algorithm ✅
**Status**: MATCHES SPECIFICATION

**File**: sba_goal_seeker.py, method `_optimise_proportion()`
**Algorithm**: Hybrid linear interpolation OR bisection search

**Discontinuity Detection** (Lines 200+):
- **Trigger 1**: After 8+ failed iterations → switch to bisection
- **Trigger 2**: Found both feasible AND infeasible bracket positions → switch to bisection

**Linear Interpolation Clamping**:
- Base change: Uses target proportion formula (unconstrained)
- Applied constraints: ±15% absolute + 30% relative allowance

**Convergence Cascade**:
1. Optimal tolerance (strict)
2. Absolute tolerance (loose) - FALLBACK
3. Hard stop (max attempts = 20)

**Verification Result**: ✅ Matches spec section 3.3.7 hybrid approach

---

### 3. Rebalancing Iterations ✅
**Status**: MATCHES SPECIFICATION

**File**: balance_sheet_projection.py lines 160-195

**Iteration Strategy**:
```python
for step in [0, 1, 2]:  # Up to 3 iterations
    if not do_swap_rebalance():
        break
    if not do_physical_rebalance():
        break
```

**Execution Order per Period**:
1. Swap rebalancer (derivative KRD matching)
2. Physical rebalancer (asset allocation optimization)
3. REPEAT up to 3 times per period

**Verification Result**: ✅ Up to 3 iterations confirmed in code

**Spec Reference**: Section 3.3.6 (implicit), section 3.3.9.2 (explicit)

---

### 4. Z-Spread Calibration Structure ✅
**Status**: MATCHES SPECIFICATION

**File**: zspread_provider.py

**Three Calibration Methods** (user-configurable):
1. **CalibrateToMarketPrice**: Goal-seek individual asset z-spread to market value
2. **UseRateInAssetFile**: Use z-spread from asset data file
3. **UseRateInThisTable**: Use assumption table value (Cash, Derivatives, etc.)

**Per-Period Projection**:
- Formula: `asset_zspread(t) = t=0_zspread + (change_in_class_zspread)`
- Class z-spread changes per-period via mean reversion: `L + (S-L)×ρ^t`
- Where L = TTC, S = PiT, ρ = RhoOver or RhoUnder

**Verification Result**: ✅ Structure matches spec section 3.3.8.9

---

### 5. Spread Cap Logic  🔴 NOT IMPLEMENTED
**Status**: CONFIGURATION PARAMETER EXISTS, BUT NOT USED IN CODE

**Configuration**: settings_loader.py line 76
```python
sba_spread_cap: float  # Loaded from Excel alm_setup.xlsm
```

**Code Search Results**:
- ✅ Parameter defined in Settings dataclass (line 76)
- ❌ Parameter NOT used anywhere in the codebase
- No references found to: `sba_spread_cap`, `spread_cap`, `spread cap` in any projection logic
- No conditional logic found for: cap application, cap enforcement, BEL recalculation

**Spec Requirement** (Section 3.3.7):
- IF uncapped spread > 35bps → apply cap
- Recalculate BEL = PV(liability_CF, RF + 35bps)
- Report both uncapped and capped values
- Per-scenario: Select highest capped BEL across 9 scenarios

**Verification Status**: 🔴 **CRITICAL GAP** - Feature specified but NOT implemented

### Key Finding
The spread cap parameter exists in the configuration but is completely unused in the projection code. This is a **critical specification gap**:
- Spec requires: Mandatory 35bps regulatory cap with BEL recalculation
- Code provides: Configuration parameter but NO enforcement logic
- Impact: BEL calculations may exceed regulatory 35bps limitation

---

## Summary Table: Code vs Specification Alignment

| Component | Specification | Implementation | Status | File Location |
|-----------|---------------|-----------------|--------|----------------|
| **D&D Formula** | (1-X)^t | 1-(1-X_annual)^t | ✅ Match | asset_projector.py:161-190 |
| **D&D Application** | Both MV + CF | Scalar multiplier | ✅ Match | asset_projector.py:156-210 |
| **D&D Assertions** | Bounds [0,1] | Lines 182-186 | ✅ Match | asset_projector.py:182-186 |
| **Goal-Seek Method** | Hybrid linear/bisection | _optimise_proportion | ✅ Match | sba_goal_seeker.py:200+ |
| **Discontinuity Trigger** | Not explicit | 8 attempts ∨ bracket found | ✅ Good | sba_goal_seeker.py:250+ |
| **Linear Clamping** | ±15% | ±15% + 30% rel | ✅ Match | sba_goal_seeker.py:280+ |
| **Convergence Cascade** | Optimal → Absolute | Implemented | ✅ Match | sba_goal_seeker.py:300+ |
| **Rebalancing Iterations** | Not explicit | 3× per period | ✅ Implemented | balance_sheet_projection.py:160-195 |
| **Z-Spread Calibration** | Method selection | 3 methods | ✅ Match | zspread_provider.py |
| **Z-Spread Projection** | L+(S-L)×ρ^t | Formula match | ✅ Match | zspread_provider.py |
| **Liability KRD Discount** | **Risk-free ONLY** | **Configurable (2 methods)** | **⚠️ CRITICAL** | liability_cashflow.py:92-97 |
| **Spread Cap** | 35bps + recalc | Parameter defined, NOT used | 🔴 NOT IMPL | settings_loader.py:76, NOT USED |

---

## Critical Path for Phase 2 Completion

### Completed
1. ✅ D&D formula verified exact match
2. ✅ Goal-seek algorithm structure verified
3. ✅ Rebalancing iterations verified  
4. ✅ Liability KRD calibration method identified (REQUIRES CONFIG VERIFICATION)

### Remaining  
1. ✅ Spread cap investigation COMPLETE - Feature NOT IMPLEMENTED (param exists, never used)
2. ⏳ Verify which KRD method is configured in Excel (alm_setup.xlsx) - USER ACTION
3. ⏳ Generate detailed Code Review Checklist with all findings

### Phase 3 Readiness
Once Phase 2 completes, Phase 3 will require:
- Test case generation for KRD verification (Risk-Free vs Risk-Free+SBA scenarios)
- Callable bond validation test (dynamic call option behavior)
- Spread cap edge case validation (test when spread > 35bps)
- D&D regression test (verify formula implementation against test data)
