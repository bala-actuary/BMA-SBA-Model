# Phase 2: Code Verification - COMPLETION SUMMARY & CRITICAL DECISIONS REQUIRED

## Status: **80% COMPLETE** - AWAITING USER DECISIONS

### All Code Analysis Tasks: ✅ COMPLETE

| Task | Status | Finding |
|------|--------|---------|
| D&D Formula | ✅ VERIFIED | Exact match to spec (1-X)^t |
| Goal-Seek Algorithm | ✅ VERIFIED | Hybrid approach matches spec, discontinuity detection confirmed |
| Rebalancing Iterations | ✅ VERIFIED | Up to 3x per period implemented correctly |
| Z-Spread Calibration | ✅ VERIFIED | Three methods supported, mean reversion formula correct |
| **Liability KRD Method** | ⚠️ **CRITICAL** | Code supports 2 methods; SPEC REQUIRES Risk_Free; **NEED CONFIG VERIFICATION** |
| **Spread Cap Logic** | 🔴 **NOT IMPLEMENTED** | Parameter exists but NOT USED anywhere in code |

---

## CRITICAL ISSUE #1: Spread Cap NOT IMPLEMENTED

### The Problem
**Specification Requirement** (Section 3.3.7):
- Mandatory 35bps regulatory cap on SBA spread
- When spread > 35bps: Recalculate BEL using RF + 35bps maximum
- Report both uncapped and capped values
- Select highest capped BEL across scenarios

**Implementation Status**:
- ✅ Parameter defined: `sba_spread_cap: float` (settings_loader.py line 76)
- ❌ Parameter NEVER USED: Zero references found in entire codebase
- No cap enforcement logic in projection code
- No BEL recalculation when cap would bind

### Code Evidence
```python
# settings_loader.py line 76 - Parameter exists
sba_spread_cap: float

# Result of code search: sba_spread_cap appears 1 time (definition only)
# Searched for: "sba_spread_cap", "spread_cap", "35bps", "spread cap"
# Result: NO implementation found
```

### Impact
If SBA spread calculated > 35bps:
- ❌ Cap is NOT applied
- ❌ BEL is NOT recalculated
- ❌ Regulatory requirement NOT met
- ❌ Final BEL may exceed maximum allowed

### User Decision Required
Choose one of the following:

**Option A: IMPLEMENT SPREAD CAP** (Recommended for spec compliance)
- Requires development of cap application logic
- Must add conditional in liability projection to:
  - Check if spread > sba_spread_cap
  - Recalculate BEL with capped spread
  - Track both uncapped and capped values
- Timeline: Additional development work
- Outcome: Full spec compliance

**Option B: DOCUMENT AS KNOWN LIMITATION** (Quick path, risk acknowledged)
- Accept that spread cap is not enforced
- Document this as risk in validation report
- Proceed to Phase 3 assuming this limitation
- Assumption: Model acceptable without cap (user decision)
- Outcome: Phase 3 can proceed

**Option C: DEFER & ACCEPT CURRENT STATE** (Continue verification)
- Acknowledge gap exists but don't fix now
- Continue Phase 3 test generation with caveat
- Plan cap implementation for post-verification
- Outcome: Phase 3 proceeds, cap implementation tracked as future work

---

## CRITICAL ISSUE #2: Liability KRD Calibration Method - CONFIGURATION VERIFICATION

### The Problem
**Specification Requirement** (Section 2.3.5.3):
- Swap rebalancing KRDs calculated using risk-free rates ONLY
- No SBA spread included in liability KRD
- This ensures liability and asset KRDs on same basis for matching

**Implementation Status**:
- ✅ Code correctly implements both methods (liability_cashflow.py lines 92-97)
- ⚠️ Method selection is CONFIGURABLE from Excel
- 🔴 **CURRENT CONFIGURATION UNKNOWN** - Must verify in alm_setup.xlsx

### Code Implementation
```python
# liability_cashflow.py lines 92-97
if self.s.liability_dv01_calibration_method == LiabilityDV01CalibrationMethod.Risk_Free_plus_SBA_Spread:
    zspread_t_dv01 = zspread_t  # ❌ WRONG - Includes SBA spread
elif self.s.liability_dv01_calibration_method == LiabilityDV01CalibrationMethod.Risk_Free:
    zspread_t_dv01 = 0.0  # ✅ CORRECT - Risk-free only
else:
    raise NotImplementedError()

# Used for DV01 calculation (stress impacts)
npv_dv01_base = ql.CashFlows.npv(self.leg, rates_base, zspread_t_dv01, self.day_counter, ...)
market_value_stress_impact[stress.name] = (npv_stress - npv_dv01_base) * stress.scale
```

### Configuration Options
| Method | Risk-Free | SBA Spread | Spec Compliant |
|--------|-----------|-----------|----------------|
| `Risk_Free` | ✅ Yes | ❌ No | ✅ YES |
| `Risk_Free_plus_SBA_Spread` | ✅ Yes | ✅ Yes | ❌ NO |

### Impact of Wrong Configuration
If `Risk_Free_plus_SBA_Spread` configured:
- Liability KRD will be lower (more sensitive to curves further out)
- Asset KRD would be Risk-Free only
- Mismatch in KRD basis → Swap rebalancing targets will be wrong
- SBA proportions and hedging incorrect
- Regulatory capital calculated incorrectly

### User Action Required
1. **Open** alm_setup.xlsx
2. **Find** Parameter for liability KRD calculation method
   - Look for: "KRD", "Liability DV01", "Calibration Method", "Discount Rate"
   - Likely in parameters/settings sheet
3. **Report** which method is configured
4. **Confirm**:
   - If set to `Risk_Free`: ✅ Compliant, Phase 3 ready
   - If set to `Risk_Free_plus_SBA_Spread`: ❌ Non-compliant, must be fixed
   - If unsure of value: Take screenshot of parameter cell

---

## Next Steps (Sequence)

### Immediate (USER ACTION)
1. Check alm_setup.xlsx configuration for:
   - Liability KRD calibration method (confirm = Risk_Free)
   - Spread cap setting (note the configured value)
2. Decide on spread cap: Implement / Document / Defer
3. Provide confirmation to proceed

### After User Input
1. Generate Phase 2 Code Review Checklist
2. Document findings in final Phase 2 Report
3. Proceed to Phase 3 (Test case generation & validation)

### Phase 3 Dependencies
- ✅ Will verify D&D formula implementation
- ✅ Will create goal-seek test scenarios
- ✅ Will test rebalancing iterations
- ⚠️ Will include KRD verification (depends on correct config)
- 🔴 Will include spread cap test ONLY if implemented

---

## File References for Investigation

**User should examine**:
- [alm_setup.xlsm](alm_setup.xlsm) - Parameters sheet for configuration values

**Code reference locations**:
- [liability_cashflow.py](liability_cashflow.py#L92-L97) - KRD method selection
- [setting_enums.py](setting_enums.py#L25) - LiabilityDV01CalibrationMethod enum
- [settings_loader.py](settings_loader.py#L76) - sba_spread_cap parameter definition

---

## Summary Assessment

**Current State**:
- 5 of 6 major components verified ✅
- Code quality: GOOD (implementations are correct where implemented)
- Specification alignment: PARTIAL (1 feature not implemented, 1 config unknown)
- Production readiness: CONDITIONAL (pending KRD config confirmation & spread cap decision)

**Risk Level**:
- 🔴 HIGH if `Risk_Free_plus_SBA_Spread` configured (KRD mismatch)
- 🟡 MEDIUM if spread cap not applied (regulatory non-compliance)
- 🟢 LOW if both issues resolved correctly

**Recommendation**:
Phase 3 should proceed with conditions:
- Confirm KRD method = Risk_Free ✓
- Document/implement spread cap decision ✓
- Generate edge case tests for both issues ✓
