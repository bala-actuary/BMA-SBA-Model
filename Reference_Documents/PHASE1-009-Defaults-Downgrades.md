# DEFAULTS & DOWNGRADES (D&D) METHODOLOGY

## Overview

The Pythora model applies Defaults and Downgrades (D&D) costs to asset valuations throughout the projection period, reducing both market values and cashflows to reflect expected credit losses.

**⚠️ CRITICAL VALIDATION NOTE**: This methodology is implemented in the code but **HAS NOT BEEN FULLY TESTED** as of model v1.0.0 approval (Oct 2024). Previous expected loss assumptions have been used for validation instead.

---

## D&D Formula & Application

### Time-Zero & Per-Period Calculation

For each asset in each projection period, D&D costs are applied using:

$$MV\_Adj(t) = MV(t) \times (1-X)^t$$

$$CF\_Adj(t) = CF(t) \times (1-X)^t$$

Where:
- $MV(t)$ = Unadjusted market value at period t
- $CF(t)$ = Unadjusted cashflow in period t
- $X$ = Cumulative D&D cost (from lookup table)
- $t$ = Years since inception for that asset cohort
- Both market value and cashflows reduced by the same factor

### D&D Lookup Table

Assumptions are specified by three dimensions:

| Dimension | Values | Source |
|-----------|--------|--------|
| **Asset Class** | IG Corporate, Sovereign, IG HY, Covered Bonds, ABS, RMLs, etc. | User assumptions |
| **Credit Rating** | AAA, AA, A, BBB, BB, B, CCC, CC/C, D, NR | User assumptions |
| **Years Since Inception** | 1y, 2y, 3y, ..., 10y+ | Cumulative duration |

Example lookup:
- IG Corporate, BBB rated, 3 years since issue: 0.17% D&D cost per year
- This means: each year for 3 years, apply 0.17% reduction

---

## Implementation Details

### Where D&D is Applied

- ✅ **Individual Assets** (Bonds, Swaps, RMLs, Covered Bonds, ABS, etc.)
- ✅ **Cashflows** (Coupons and redemptions reduced)
- ✅ **Market Values** (Notional reduced, valuation reduced)
- ❌ **Derivatives**: NOT applied (fully collateralized, exempt per spec)

### Per-Period Calculation Steps

1. **At end of period**: Calculate cumulative default at start of period + expected default during period
2. **Cumulative formula**: X(t) = X(t-1) + D&D(t)
3. **Apply to cashflows**: Reduce by (1-X(t))^t factor
4. **Apply to market value**: Reduce by (1-X(t))^t factor
5. **Result**: Credibly reduce asset value and accumulated defaults iteratively

### Code Location

- **File**: `asset_projector.py`
- **Method**: `project()` (called per asset per period)
- **Formula implementation**: Lines [TBD - to be verified in code review]

---

## D&D Table Structure (Excel Input)

The D&D assumptions are loaded from the `alm_setup.xlsx` file under the **"D&D"** tab.

### Table Columns

| Column | Meaning | Example |
|--------|---------|---------|
| cohort | existing or reinvestment | existing |
| bu | Business Unit (*, ANL, AB, ARE) | ANL |
| apollo_internal_classification | Asset class | IG Corporate |
| defaults_table | Identifier for defaults vs downgrades | Def_Total |
| rating_to_use_for_defaults | Credit rating | BBB |
| Term_1y to Term_10y+ | D&D cost for each maturity bucket | 0.17% |

### Example Data (from Spec Lines 3.2.1.4)

| Cohort | BU | Asset Class | Rating | 1y | 2y | 3y | 4y | 5y | 10y |
|--------|----|----|---------|------|------|------|------|------|------|
| existing | ANL | IG Corporate (Snr Unsecured) | BBB | 0.14% | 0.17% | 0.20% | 0.23% | 0.25% | 0.31% |
| existing | ANL | IG Corporate (Snr Unsecured) | A | 0.04% | 0.05% | 0.07% | 0.08% | 0.09% | 0.13% |
| existing | ANL | IG Corporate (Snr Unsecured) | AA | 0.02% | 0.02% | 0.03% | 0.04% | 0.05% | 0.06% |

---

## Validation Status & Limitations

### Current Status (v1.0.0 - Oct 2024)

🔴 **NOT FULLY TESTED**
- D&D methodology implemented in code
- Alternative "expected loss" methodology used for validation
- New BMA D&D methodology not yet validated against actual results

### Known Limitations

1. **No validation against historical data**: D&D assumptions not backtested
2. **No scenario variation**: D&D costs same across all interest rate scenarios
3. **Assumed constant per year**: No term structure of D&D costs within year
4. **No recovery assumptions**: Only reduction, no recovery modelled

### Recommended Validation (Pre-Production)

- [ ] Verify formula implementation matches spec exactly (line-by-line code review)
- [ ] Test against simple bond example with known D&D rates
- [ ] Compare expected loss method results vs D&D method results
- [ ] Validate impact on SBA BEL calculation
- [ ] Stress test with extreme D&D assumptions
- [ ] Back-test against historical defaults if data available

---

## Verification Checklist

- [ ] **Formula Check**: Confirm asset_projector.py implements $MV\_Adj(t) = MV(t) \times (1-X)^t$
- [ ] **Lookup Logic**: Confirm D&D table lookup by (asset_class, rating, years_since_inception)
- [ ] **Application**: Confirm D&D applied to ALL assets except derivatives
- [ ] **Exclusion**: Confirm derivatives NOT affected by D&D (spec section 3.3.8.11)
- [ ] **Cumulative**: Confirm cumulative default tracked correctly across periods
- [ ] **Reinvestment Assets**: Confirm D&D applied to reinvested assets (not just existing)
- [ ] **Output Reporting**: Confirm D&D impact visible in projection outputs

---

## Reference to Specification

- **Specification Reference**: SBA BEL Model Implementation Specification v1.0.0, Section 3.3.8.11
- **Methodology Reference**: SBA BEL Methodology Paper, Section 10
- **Data Source**: alm_setup.xlsx, "D&D" tab
- **BMA Source**: BMA prescribed Defaults & Downgrades assumptions (updated YE24)

---

## Next Steps

1. **Code Review** (Phase 2): Extract asset_projector.py and verify formula
2. **Validation** (Phase 3): Create test cases with known D&D rates
3. **Production Readiness**: Validate against expected loss baseline before go-live

---

## Questions for Implementation Verification

- Q1: Is D&D applied at end-of-period or beginning-of-period?
- Q2: Are D&D costs accumulated across years correctly?
- Q3: What happens if asset rating changes or downgrade during projection?
- Q4: Are reinvestment assets getting correct D&D assumptions?
- Q5: How is D&D impact reported in output files?

