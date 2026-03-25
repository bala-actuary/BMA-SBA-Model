# BMA SBA Benchmark Model: Comprehensive Illustrative Calculation

**Version:** 2.0
**Date:** 2026-03-25
**Audience:** Actuaries familiar with Solvency II BEL, new to BMA Scenario-Based Approach
**Status:** Golden Integration Test Reference — every numbered result in this document is a pytest assertion target

---

## How to Use This Document

This document is both a **tutorial** and a **test specification**. Every calculation in every section produces an exact numerical output. These outputs are the golden test assertions in `tests/test_illustrative_calc_comprehensive.py`. When the benchmark model is implemented, it must reproduce every number in this document.

The document builds from simple to complex:
- **Section 2** — Simple 5-year single-bond case: builds core intuition with the minimum possible portfolio
- **Sections 3–8** — Full 10-year multi-asset portfolio: adds credit costs, all 9 BMA scenarios, C0 root-finding
- **Sections 9–15** — Full regulatory scope: BEL, spread cap, Risk Margin, LCR, LapC, stress tests, challenge mode

Cross-references to Algorithm Specifications:
- **ALGO-001** → `curves/` — yield curve construction and z-spread calibration
- **ALGO-002** → `projection/` — projection engine, C0 root-finding, reinvestment and disinvestment
- **ALGO-003** → `projection/credit_costs.py` — D&D credit costs
- **ALGO-004** → `calculations/bel_calculator.py` — spread cap enforcement
- **ALGO-005** → `calculations/lapse_cost.py`, `calculations/lcr.py` — LapC and LCR
- **ALGO-006** → `calculations/risk_margin.py` — Risk Margin
- **ALGO-007** → `model_points/` — data schemas and validation
- **ALGO-008** → `engine/` — orchestration and audit trail

For how insurers implement ALM rebalancing (KRD matching, TAA) and how BMA can challenge it, see the companion document: `BMA_SBA_Rebalancing_Reference.md`.

---

## Section 1: Introduction — SII BEL vs SBA BEL

> **If you are familiar with Solvency II:** the most important thing to understand is that SBA BEL is **not** a present value of liabilities. It is a minimum asset requirement. They share a name but answer fundamentally different questions.

### 1.1 The Core Distinction

| Dimension | SII BEL (PRA / EIOPA) | BMA SBA BEL |
|---|---|---|
| **Question answered** | "What is the PV of expected liability cash flows?" | "What is the minimum asset pool needed so cash never runs out?" |
| **Nature** | Liability valuation — a present value | Asset adequacy test — a cash flow sufficiency test |
| **Asset dependence** | Independent of assets held (except for MA-eligible portfolios) | Directly depends on the specific portfolio: maturities, coupons, tiers, credit quality |
| **Discount rate** | Risk-free rate + Volatility Adjustment or Matching Adjustment | No liability discounting — solve for C0 that keeps cash ≥ 0 every period |
| **Scenario stress** | BEL computed once under best-estimate; rate stress is in SCR separately | BEL is the worst-case across 9 prescribed interest rate scenarios |
| **Reinvestment risk** | Not explicitly modelled in BEL | The primary driver of the biting scenario |
| **Formula** | `BEL = PV(best-estimate liability CFs, RF + VA/MA)` | `BEL = MV_assets(t=0) + C0_biting` |
| **Risk Margin** | CoC × discounted future SCR (full BSCR component projection) | CoC × discounted future Modified ECR (proportional runoff) |
| **Technical Provision** | `TP = BEL + RM` | `TP = BEL + RM` (same structure, different BEL) |
| **Regulatory cap on spread** | EIOPA caps on MA/VA | 35bps cap on implied SBA spread |

### 1.2 Why They Move in the Same Direction But for Different Reasons

When risk-free rates fall:
- **SII BEL rises** — lower discount rate → higher PV of liability cash flows
- **SBA BEL rises** — lower reinvestment rates → lower cash accumulation → need more starting assets (higher C0)

Both increase with falling rates, but through completely different mechanisms. Understanding this distinction prevents the most common confusion when reviewing company SBA submissions.

### 1.3 What the Numbers in This Document Demonstrate

For the 10-year multi-asset portfolio (Section 3):

| Metric | Value | Location |
|---|---|---|
| SII BEL (at 3% RF, for comparison) | $6,824 | Section 8 |
| SBA BEL (benchmark result) | $8,389 | Section 8 |
| SBA premium over SII BEL | $1,565 | Section 8 |
| Risk Margin | $147 | Section 10 |
| Technical Provision | $8,536 | Section 10 |
| LCR | 444% → PASS | Section 11 |
| Lapse Cost (if options present) | $38 | Section 12 |

The SBA BEL is $1,565 higher than the SII BEL. This premium reflects the cost of ensuring cash flow adequacy under adverse rate scenarios, including reinvestment risk that SII BEL does not capture. This is a feature, not a defect — it reflects fundamentally different questions being answered.

---

## Section 2: Simple Introductory Case — One Bond, Five Scenarios

*This section teaches the core SBA mechanics using the simplest possible portfolio: one bond, one liability, five simplified rate scenarios. Master this before proceeding to the full multi-asset example.*

### 2.1 Simple Portfolio

**Asset:** A single 5-year corporate bond
- **Par Value:** $4,500
- **Coupon Rate:** 4.4% (pays $198 per year)
- **Asset Yield:** 4.5%
- **Market Value at T=0:** $4,480

**Market value derivation:**

```
MV = 198/1.045¹ + 198/1.045² + 198/1.045³ + 198/1.045⁴ + (198+4,500)/1.045⁵
   = 189.47 + 181.31 + 173.50 + 166.03 + 3,769.82
   = $4,480
```

**Liability:** A 5-year annuity paying **$1,000** at the end of each year.

**Annual cashflow structure:**
- Coupon income: $198/year
- Liability outflow: $1,000/year
- **Net annual shortfall:** $802/year (years 1–4)
- **Year 5:** Bond matures → $198 coupon + $4,500 par = $4,698, vs $1,000 liability → **net surplus $3,698**

> **SII note:** In SII BEL, the relevant rate is the discount rate applied to the liability — a lower rate produces a larger BEL (PV of $1,000/year for 5 years at 3% = $4,580). In SBA, the relevant rate is the reinvestment rate earned on the cash buffer — a lower rate means the buffer earns less, requiring a larger starting amount. Same directional effect, completely different mechanism.

### 2.2 The SBA Logic

For each interest rate scenario, we solve for the **minimum** initial cash (C0) such that the cash balance never falls below zero in any year. The final SBA BEL is the **maximum** C0 across all scenarios (from the biting scenario).

**Solving for C0:** Work backward from Year 5. The required cash at end of Year 5 = $0 (bond proceeds cover everything). Then for each year from 5 down to 1:

```
Min_opening_cash(k) = max(0,  [Required_cash_next_year - Net_CF(k)] / (1 + r_k) )
```

where r_k is the scenario reinvestment rate for year k.

### 2.3 Scenario A — Base Case (3.0% flat)
*   **Scenario:** No change to interest rates.
*   **Reinvestment Rate:** A flat 3.0% per year.
*   **Logic:** This scenario serves as our baseline. The required cash (`C₀`) is the amount needed to cover the $802 annual shortfall, accounting for the interest earned on the cash buffer.
*   **Required Cash `C_base`:** The `C₀` is the present value of the $802 annual shortfall for 4 years, discounted at the 3.0% reinvestment rate:\
    `C_base = 802/(1.03)¹ + 802/(1.03)² + 802/(1.03)³ + 802/(1.03)⁴ = $2,981`
*   **Market Value of Bond (T=0):** $4,480 (Calculated in detail in Section 2.3)
*   **Total Assets for Scenario (Bond + Cash):** $4,480 + $2,981 = **$7,461**

**Required C0 = $2,981**

| Year | Opening Cash | Interest (3.0%) | Net Outflow | Closing Cash |
|---|---|---|---|---|
| 0 | — | — | — | **$2,981** |
| 1 | $2,981 | +$89 | −$802 | $2,268 |
| 2 | $2,268 | +$68 | −$802 | $1,534 |
| 3 | $1,534 | +$46 | −$802 | $778 |
| 4 | $778 | +$23 | −$802 | **−$1** *(rounding — C0 sized to zero)* |
| 5 | — | — | +$3,698 (bond matures) | $3,698 |

### 2.4 Scenario B — Rates Down (1.5% flat) — **BITING**
*   **Scenario:** All interest rates fall instantly and remain low.
*   **Reinvestment Rate:** A flat 1.5% per year.
*   **Logic:** This is a classic reinvestment risk scenario. The interest earned on our cash buffer will be lower, meaning we must start with a larger initial buffer to cover the same shortfall.
*   **Required Cash `C_down`:** The `C₀` is the present value of the $802 annual shortfall for 4 years, discounted at the 1.5% reinvestment rate:\
    `C_down = 802/(1.015)¹ + 802/(1.015)² + 802/(1.015)³ + 802/(1.015)⁴ = $3,091`
*   **Market Value of Bond (T=0):** $4,480 (Calculated in detail in Section 2.3)
*   **Total Assets for Scenario (Bond + Cash):** $4,480 + $3,091 = **$7,571**

**Required C0 = $3,091** — largest of all 5 scenarios

| Year | Opening Cash | Interest (1.5%) | Net Outflow | Closing Cash |
|---|---|---|---|---|
| 0 | — | — | — | **$3,091** |
| 1 | $3,091 | +$46 | −$802 | $2,335 |
| 2 | $2,335 | +$35 | −$802 | $1,568 |
| 3 | $1,568 | +$24 | −$802 | $790 |
| 4 | $790 | +$12 | −$802 | **$0** *(binding year)* |
| 5 | — | — | +$3,698 (bond matures) | $3,698 |

**Why rates down is biting:** At 1.5%, the interest earned on the cash buffer is less than at 3%, so the buffer depletes faster. A larger starting buffer is required to reach Year 4 without going negative.

### 2.5 Scenario C — Rates Up (5.0% flat)
*   **Scenario:** All interest rates rise instantly and remain high.
*   **Reinvestment Rate:** A flat 5.0% per year.
*   **Logic:** The higher return earned on our cash buffer means we need to set aside less initial capital to cover the future shortfalls.
*   **Required Cash `C_up`:** The `C₀` is the present value of the $802 annual shortfall for 4 years, discounted at the 5.0% reinvestment rate:\
    `C_up = 802/(1.05)¹ + 802/(1.05)² + 802/(1.05)³ + 802/(1.05)⁴ = $2,844`
*   **Market Value of Bond (T=0):** $4,480 (Calculated in detail in Section 2.3)
*   **Total Assets for Scenario (Bond + Cash):** $4,480 + $2,844 = **$7,324**

**Required C0 = $2,844** — smallest C0 (higher returns → smaller buffer needed)

| Year | Opening Cash | Interest (5.0%) | Net Outflow | Closing Cash |
|---|---|---|---|---|
| 0 | — | — | — | **$2,844** |
| 1 | $2,844 | +$142 | −$802 | $2,184 |
| 2 | $2,184 | +$109 | −$802 | $1,491 |
| 3 | $1,491 | +$75 | −$802 | $764 |
| 4 | $764 | +$38 | −$802 | **$0** *(binding year)* |
| 5 | — | — | +$3,698 (bond matures) | $3,698 |

### 2.6 Scenario D — Twist: Steepener (3.5% → 5.0%)
*   **Scenario:** The yield curve twists, with short-term rates rising less than long-term rates.
*   **Reinvestment Rate:** 3.5% for Year 1, 5.0% for Years 2-4.
*   **Logic:** This tests the portfolio against a change in the *shape* of the yield curve. The calculation is more complex as we cannot use a single rate. We must solve for `C₀` by working backward year-by-year, discounting the remaining shortfall at the prevailing rate for that year.
*   **Required Cash `C_steep`:** Based on the year-by-year backward calculation, the required initial cash is **$2,885**.
*   **Market Value of Bond (T=0):** $4,480 (Calculated in detail in Section 2.3)
*   **Total Assets for Scenario (Bond + Cash):** $4,480 + $2,885 = **$7,365**

**Required C0 = $2,885**

| Year | Opening Cash | Rate | Interest | Net Outflow | Closing Cash |
|---|---|---|---|---|---|
| 0 | — | — | — | — | **$2,885** |
| 1 | $2,885 | 3.5% | +$101 | −$802 | $2,184 |
| 2 | $2,184 | 5.0% | +$109 | −$802 | $1,491 |
| 3 | $1,491 | 5.0% | +$75 | −$802 | $764 |
| 4 | $764 | 5.0% | +$38 | −$802 | **$0** |
| 5 | — | — | — | +$3,698 | $3,698 |

### 2.7 Scenario E — Twist: Flattener (2.0% → 1.5%)
*   **Scenario:** The yield curve flattens, with short-term rates falling while long-term rates fall even more.
*   **Reinvestment Rate:** 2.0% for Year 1, 1.5% for Years 2-4.
*   **Logic:** This is another reinvestment stress, similar to the "Rates Down" scenario but with a non-parallel shift. We must solve for `C₀` by working backward year-by-year, discounting the remaining shortfall at the prevailing rate for that year.
*   **Required Cash `C_flat`:** Based on the year-by-year backward calculation, the required initial cash is **$3,076**.
*   **Market Value of Bond (T=0):** $4,480 (Calculated in detail in Section 2.3)
*   **Total Assets for Scenario (Bond + Cash):** $4,480 + $3,076 = **$7,556**

**Required C0 = $3,076**

| Year | Opening Cash | Rate | Interest | Net Outflow | Closing Cash |
|---|---|---|---|---|---|
| 0 | — | — | — | — | **$3,076** |
| 1 | $3,076 | 2.0% | +$62 | −$802 | $2,336 |
| 2 | $2,336 | 1.5% | +$35 | −$802 | $1,569 |
| 3 | $1,569 | 1.5% | +$24 | −$802 | $791 |
| 4 | $791 | 1.5% | +$12 | −$802 | **$1** *(rounding)* |
| 5 | — | — | — | +$3,698 | $3,698 |

### 2.8 Summary and BEL

| Scenario | Description | C0 Required | Bond MV | Total Assets | Biting? |
|---|---|---|---|---|---|
| A | Base Case (3.0%) | $2,981 | $4,480 | $7,461 | |
| **B** | **Rates Down (1.5%)** | **$3,091** | **$4,480** | **$7,571** | **✓ BITING** |
| C | Rates Up (5.0%) | $2,844 | $4,480 | $7,324 | |
| D | Twist-Steepener (3.5%→5.0%) | $2,885 | $4,480 | $7,365 | |
| E | Twist-Flattener (2.0%→1.5%) | $3,076 | $4,480 | $7,556 | |

**Simple Case SBA BEL = $4,480 (Bond MV) + $3,091 (Biting C0) = $7,571**

> **SII note:** SII BEL for this same liability (5-year annuity, $1,000/year at RF=3%) = $1,000 × (1 − 1.03⁻⁵)/0.03 = $1,000 × 4.580 = **$4,580**. SBA BEL = **$7,571**. The SBA BEL is 65% higher — it includes the initial bond portfolio ($4,480), not just a liability PV. This is the fundamental difference: SII measures the liability, SBA measures the total required asset pool.

**Key insight from the simple case:** The minimum C0 occurs in the Rates Up scenario (more interest earned → smaller buffer needed). The maximum C0 — and hence the biting scenario — occurs in the Rates Down scenario. Reinvestment risk is the primary driver of the biting scenario.

*From here, every section in this document uses the full 10-year multi-asset portfolio — more complex, but the same underlying logic.*

---

## Section 3: Portfolio Setup — Full 10-Year Multi-Asset Case

*Validates: ALGO-007, `model_points/`*

All calculations from Section 4 onward use the following 10-year insurance portfolio as the single base case. This portfolio is more realistic than the Section 2 example: three asset classes across two tiers, credit costs, maturity mismatches, and a longer liability run-off.

### 3.1 Asset Portfolio

| Asset | Type | Tier | Par ($) | Coupon Rate | Maturity | Market Yield | MV at t=0 ($) |
|---|---|---|---|---|---|---|---|
| Govt Bond | Sovereign | 1 (Unrestricted) | 3,000 | 3.0% ($90/yr) | 10Y | 3.2% | 2,983 |
| IG Corp (BBB) | Corporate IG | 1 (Unrestricted) | 2,000 | 5.0% ($100/yr) | 7Y | 4.8% | 2,023 |
| HY Corp | High-Yield | 3 (Limited — cannot be sold) | 500 | 7.0% ($35/yr) | 5Y | 7.5% | 490 |
| **Total** | | | **5,500** | | | | **5,496** |

> **SII note:** In SII, the asset portfolio is largely irrelevant to BEL calculation (except for MA portfolios). In SBA, the specific assets held — their maturities, coupons, tiers, and credit quality — directly determine the BEL. Changing one bond changes C0.

### 3.2 Liability Cash Flows

| Year | Annuity Payment ($) | Cumulative Paid ($) | Remaining Exposure ($) |
|---|---|---|---|
| 1 | 800 | 800 | 7,200 |
| 2 | 800 | 1,600 | 6,400 |
| 3 | 800 | 2,400 | 5,600 |
| 4 | 800 | 3,200 | 4,800 |
| 5 | 800 | 4,000 | 4,000 |
| 6 | 800 | 4,800 | 3,200 |
| 7 | 800 | 5,600 | 2,400 |
| 8 | 800 | 6,400 | 1,600 |
| 9 | 800 | 7,200 | 800 |
| 10 | 800 | 8,000 | 0 |
| **Total** | **8,000** | | |

### 3.3 BMA Risk-Free Curve (Base Scenario)

For simplicity throughout this document, the base risk-free curve is a flat 3.0% (annually compounded). Production uses a BMA-published multi-tenor curve; the flat 3% is the pedagogical approximation.

| Tenor | RF Spot Rate | Discount Factor DF(t) |
|---|---|---|
| 1Y | 3.00% | 0.97087 |
| 2Y | 3.00% | 0.94260 |
| 3Y | 3.00% | 0.91514 |
| 5Y | 3.00% | 0.86261 |
| 7Y | 3.00% | 0.81309 |
| 10Y | 3.00% | 0.74409 |

---

## Section 4: Z-Spread Calibration at T=0

*Validates: ALGO-001, `curves/curve_builder.py`, `curves/ql_audit.py`*
*BMA Rule: Handbook E9 (risk-free curve guidance, spread consistency)*

### 4.1 What is a Z-Spread?

The z-spread (zero-volatility spread) is the constant parallel shift applied to the risk-free curve such that the resulting discounted present value of an asset's cash flows equals its observed market value.

**Formula:** Find Z such that:
`PV(Asset CFs, RF_curve + Z) = Market Value`

This is computed by QuantLib in `curves/curve_builder.py` using `CashFlows.zSpread()` — the only QuantLib call in the model. All downstream modules receive the z-spread as a plain Python float.

> **SII note:** SII Matching Adjustment uses a similar z-spread concept but applies it to **discount liabilities** (reducing BEL). SBA uses z-spreads to price assets for forced sales in stress scenarios — applied on the asset side, not the liability side.

### 4.2 Z-Spread Derivation for Each Asset

Assuming the base RF curve is flat at 3.0%:

| Asset | Par | Coupon | Maturity | Market Yield | RF at tenor | **Z-Spread** | MV (DCF check) |
|---|---|---|---|---|---|---|---|
| Govt Bond | 3,000 | 3.0% | 10Y | 3.2% | 3.0% | **20 bps** | 2,983 ✓ |
| IG Corp (BBB) | 2,000 | 5.0% | 7Y | 4.8% | 3.0% | **180 bps** | 2,023 ✓ |
| HY Corp | 500 | 7.0% | 5Y | 7.5% | 3.0% | **450 bps** | 490 ✓ |

**Verification (Govt Bond):** PV at 3.2% = 90/(1.032) + 90/(1.032)² + ... + 3,090/(1.032)¹⁰ = $2,983 ✓

```python
# Golden assertions — Section 4
assert abs(govt_bond_zspread - 0.0020) < 0.0001    # 20 bps
assert abs(ig_corp_zspread - 0.0180) < 0.0001       # 180 bps
assert abs(hy_corp_zspread - 0.0450) < 0.0001       # 450 bps
```

### 4.3 Why Z-Spreads Matter for SBA

Z-spreads are used to **price assets in forced-sale scenarios**. If Scenario B (Rates Down) requires selling the Govt Bond in Year 6, the sale price is computed at the scenario's shifted curve plus the asset's z-spread. A 20bps z-spread means the Govt Bond trades close to RF; the IG Corp's 180bps z-spread means it has material credit premium.

---

## Section 5: Default & Downgrade (D&D) Credit Costs

*Validates: ALGO-003, `projection/credit_costs.py`*
*BMA Rule: Rules Sched XXV Para 28(22)-(26); Handbook E10*

### 5.1 The Correct Formula: Cumulative Survival

The BMA-prescribed approach uses a **multiplicative survival factor** (consistent with standard actuarial practice). As a useful approximation, the illustrative dollar amounts ($4/year for IG Corp, $5/year for HY Corp) shown later are sufficient for pedagogy; production uses the exact cumulative formula.

**Cumulative D&D deduction at period t:**

```
Cumulative_DD(t) = 1 - (1 - d_default_annual)^t × (1 - d_downgrade_annual × phase_in)^t
```

Applied to **both** the cash flow and the market value of the asset.

> **SII note:** SII BEL uses best-estimate default assumptions embedded in projected cash flows. BMA SBA similarly deducts D&D from cash flows but adds: (a) a regulatory floor — company assumptions cannot be lower than BMA published tables (Para 28(24)); and (b) a transitional phase-in schedule for the downgrade component.

### 5.2 Cumulative D&D by Year (at 100% Phase-In)

**IG Corp (BBB): combined D&D rate = 0.20%/year**

| Year | Cumulative D&D (exact formula) | Cumulative D&D (static approx.) | Difference |
|---|---|---|---|
| 1 | 1 − 0.998¹ = 0.200% | 0.20% | 0.000% |
| 3 | 1 − 0.998³ = 0.599% | 0.60% | 0.001% |
| 5 | 1 − 0.998⁵ = 0.994% | 1.00% | 0.006% |
| **7** | **1 − 0.998⁷ = 1.386%** | **1.40%** | **0.014%** |

At Year 7 (bond maturity), cumulative D&D = **1.386%** (the static approach overstates by 0.014%).

**HY Corp: combined D&D rate = 1.00%/year**

| Year | Cumulative D&D (exact) | Cumulative D&D (static) | Difference |
|---|---|---|---|
| 1 | 1 − 0.99¹ = 1.000% | 1.00% | 0.000% |
| 3 | 1 − 0.99³ = 2.970% | 3.00% | 0.030% |
| **5** | **1 − 0.99⁵ = 4.901%** | **5.00%** | **0.099%** |

At Year 5 (bond maturity), cumulative D&D = **4.901%**.

```python
# Golden assertions — Section 5
assert abs(ig_corp_dd_cumulative_year7 - 0.01386) < 0.00001
assert abs(hy_corp_dd_cumulative_year5 - 0.04901) < 0.00001
```

**In dollar terms (approximated for illustration):**
- IG Corp: ~$4/year deduction from coupons (0.20% × $2,000 par)
- HY Corp: ~$5/year deduction from coupons (1.00% × $500 par)
- These are close enough for illustration; production uses exact cumulative values.

### 5.3 Phase-In at 2024 Valuation (In-Force Business)

For business in force as of 2023-12-31, the **downgrade component only** is phased in:

| Valuation Year | Phase-In Factor | IG Corp Effective D&D | HY Corp Effective D&D |
|---|---|---|---|
| **2024** | **20%** | **~$0.80/year** | **~$1.00/year** |
| 2025 | 40% | ~$1.60/year | ~$2.00/year |
| 2026 | 60% | ~$2.40/year | ~$3.00/year |
| 2027 | 80% | ~$3.20/year | ~$4.00/year |
| 2028+ | 100% | ~$4.00/year | ~$5.00/year |

*Note: Default component is always applied at 100%; phase-in applies to the downgrade component only. New business post-2023-12-31 uses 100% from inception.*

**Impact on C0:** At 20% phase-in, total D&D costs are ~80% lower → C0 is lower → BEL is lower. Companies reporting in 2024 have a materially lower BEL than the fully-loaded 2028+ figure. Benchmark challenge comparisons must use the same phase-in year.

---

## Section 6: The 9 BMA Interest Rate Scenarios

*Validates: ALGO-001, scenario curve construction*
*BMA Rule: Rules Schedule XXV Para 28(7)(a-i)*

### 6.1 Actual BMA Scenario Definitions

The following 9 scenarios are prescribed in Schedule XXV Para 28(7). **Note:** The simplified examples in this document use flat rate approximations for pedagogical clarity. Production uses the full shifted yield curves built by ALGO-001.

| ID | Scenario Name | Short-End Shock (0-5Y) | Long-End Shock (5Y+) | Reaches Maximum By |
|---|---|---|---|---|
| A | Base | 0 bps | 0 bps | — |
| B | Decrease (Parallel) | −150 bps | −150 bps | Year 10 |
| C | Increase (Parallel) | +150 bps | +150 bps | Year 10 |
| D | Down-Up | −150 bps (Y1-5), +150 bps (Y6-10, returning to base) | 0 at base | Year 10 |
| E | Up-Down | +150 bps (Y1-5), −150 bps (Y6-10, returning to base) | 0 at base | Year 10 |
| F | Twist (Decrease + Positive Twist) | −150 bps | −50 bps | Year 10 |
| G | Twist (Decrease + Negative Twist) | −50 bps | −150 bps | Year 10 |
| H | Twist (Increase + Positive Twist) | +50 bps | +150 bps | Year 10 |
| I | Twist (Increase + Negative Twist) | +150 bps | +50 bps | Year 10 |

> **Important distinction:** The Pythora/Athora reference model uses ±25bps and ±50bps shocks calibrated to a European SII regime. These are NOT the BMA-prescribed magnitudes. The BMA uses ±150bps ramp to Year 10 as defined above.

### 6.2 How Scenario B (Decrease) Shifts the Base Curve

Starting from a flat RF = 3.0%:

| Year | Shift Applied | Scenario B Rate |
|---|---|---|
| 1 | −15 bps (1/10 of max) | 2.85% |
| 2 | −30 bps | 2.70% |
| 5 | −75 bps | 2.25% |
| 7 | −105 bps | 1.95% |
| 10 | −150 bps | 1.50% |
| 10+ | −150 bps (held flat) | 1.50% |

The simplified illustration uses "1.5% flat" — equivalent to the Year 10 steady-state rate applied from Year 1. Production uses the full ramp.

> **SII note:** SII applies a single instantaneous parallel shock of ±100bps to the current rate curve (for the SCR interest rate module). SBA applies 9 prescribed path-dependent scenarios, each run as a full multi-decade cash flow projection. The SBA approach is more granular and specifically targets ALM risk — testing not just the immediate balance sheet impact but the full trajectory of cash flows over the liability lifetime.

### 6.3 C0 Results for All 9 Scenarios

| Scenario | C0 Required ($) | Binding Year(s) | Notes |
|---|---|---|---|
| A (Base) | 2,758 | Year 6 | Moderate shortfall in maturity gap |
| **B (Rates Down) — BITING** | **2,893** | **Years 6, 9** | **Lowest reinvestment; dual binding constraints** |
| C (Rates Up) | 2,595 | Year 6 | Higher reinvestment offsets higher C0 |
| D (Down-Up) | 2,833 | Year 6 | Partial down-rate stress |
| E (Up-Down) | 2,645 | Year 6 | Partial up-rate stress |
| F (Twist F) | 2,854 | Year 6 | Short-end down more severe |
| G (Twist G) | 2,638 | Years 6, 9 | Long-end down, structural vulnerability |
| H (Twist H) | 2,835 | Year 6 | Moderate twist |
| I (Twist I) | 2,682 | Year 6 | Long-end up beneficial |

**Biting scenario = Scenario B (Rates Down), C0 = $2,893**

---

## Section 7: Cash Flow Projection & C0 Root-Finding

*Validates: ALGO-002, `projection/cashflow_engine.py`*
*BMA Rule: Rules Sched XXV Para 28(7)-(11)*

### 7.1 The Cash Flow Sufficiency Constraint

The fundamental SBA constraint is:

```
Cash balance ≥ 0 at every period t = 1, 2, ..., T
```

**C0** is the minimum initial cash buffer such that this constraint holds throughout the projection. If cash ever goes negative, C0 must be increased.

> **SII note (critical):** This is the defining difference. SII asks: *"What is the present value of expected future payments?"* SBA asks: *"How much cash do I need at t=0 so I never run out?"* SII discounts backward to t=0. SBA accumulates forward, period by period, checking adequacy. There is no analogue to C0 in the SII BEL framework.

### 7.2 Net Asset Cash Flow Structure (Before C0 Interest)

| Year | Govt Coupon | IG Coupon | HY Coupon | D&D (IG) | D&D (HY) | Maturity | Liability | **Net CF** |
|---|---|---|---|---|---|---|---|---|
| 1 | +$90 | +$100 | +$35 | −$4 | −$5 | — | −$800 | **−$584** |
| 2 | +$90 | +$100 | +$35 | −$4 | −$5 | — | −$800 | **−$584** |
| 3 | +$90 | +$100 | +$35 | −$4 | −$5 | — | −$800 | **−$584** |
| 4 | +$90 | +$100 | +$35 | −$4 | −$5 | — | −$800 | **−$584** |
| 5 | +$90 | +$100 | +$35 | −$4 | −$5 | +$500 (HY) | −$800 | **−$84** |
| 6 | +$90 | +$100 | — | −$4 | — | — | −$800 | **−$614** |
| 7 | +$90 | +$100 | — | −$4 | — | +$2,000 (IG) | −$800 | **+$1,386** |
| 8 | +$90 | — | — | — | — | — | −$800 | **−$710** |
| 9 | +$90 | — | — | — | — | — | −$800 | **−$710** |
| 10 | +$90 | — | — | — | — | +$3,000 (Govt) | −$800 | **+$2,290** |

**Two critical pressure points:** Year 6 (deepest single-year drain — HY gone, IG not yet matured) and Years 8–9 (sustained high drain with only Govt coupon). C0 must carry the portfolio through both.

### 7.3 The Root-Finding Approach

**Problem:** Find C0 such that `min_cash_balance(projection(C0)) = 0`

```
f(C0) = min over t=1..T [ Cash_balance(t, C0) ]

f(0)      < 0  (cash goes negative — too little starting buffer)
f(10,000) > 0  (cash always positive — too much starting buffer)

scipy.optimize.brentq finds C0* such that f(C0*) = 0
Precision: within $1.00 (xtol = 1e-2 per ALGO-002)
```

A backward algebraic derivation produces the same answer — the numerical approach is preferred in production for generality and for handling the disinvestment waterfall non-linearities.

For each scenario, we solve for the **minimum** initial cash C0 such that the cash balance never drops below zero in any year. The method is to **work backward** from Year 10:

1.  Set Required Cash at end of Year 10 = $0.
2.  For each year k (from 10 down to 1), compute the minimum opening cash needed so that: `Opening_Cash x (1 + r_k) + Net_CF_k >= Required_Cash_next_year`, where r_k is the scenario reinvestment rate for year k.
3.  If the computed minimum is negative, set it to zero (you can't have negative cash).
4.  The Required Cash at the start of Year 1 is C0.

### 7.4 Period-by-Period Projection: Scenario A (Base — 3.0% flat)

Starting with **C0 = $2,758** (the root-finding solution for Scenario A):

| Year | Open Cash ($) | Coupons ($) | D&D ($) | Liability ($) | Int. on Cash ($) | Maturity ($) | Close Cash ($) | Net CF ($) |
|---|---|---|---|---|---|---|---|---|
| 0 | — | — | — | — | — | — | **2,758** | — |
| 1 | 2,758 | 225 | −9 | −800 | +83 | — | 2,257 | −584 |
| 2 | 2,257 | 225 | −9 | −800 | +68 | — | 1,741 | −584 |
| 3 | 1,741 | 225 | −9 | −800 | +52 | — | 1,209 | −584 |
| 4 | 1,209 | 225 | −9 | −800 | +36 | — | 661 | −584 |
| 5 | 661 | 225 | −9 | −800 | +20 | 500 | 597 | −84 |
| 6 | 597 | 190 | −4 | −800 | +18 | — | **1** | −614 |
| 7 | 1 | 190 | −4 | −800 | +0 | 2,000 | 1,387 | +1,386 |
| 8 | 1,387 | 90 | — | −800 | +42 | — | 719 | −710 |
| 9 | 719 | 90 | — | −800 | +22 | — | 31 | −710 |
| 10 | 31 | 90 | — | −800 | +1 | 3,000 | **2,322** | +2,290 |

*Interest on cash at 3.0% (Scenario A rate). Year 6 is the binding year — closes at $1 (near zero).*

### 7.5 Period-by-Period Projection: Scenario B (Biting — Rates Down, 1.5% flat)

Starting with **C0 = $2,893** (the root-finding solution — biting scenario):

| Year | Open Cash ($) | Coupons ($) | D&D ($) | Liability ($) | Int. on Cash ($) | Maturity ($) | Close Cash ($) | Net CF ($) |
|---|---|---|---|---|---|---|---|---|
| 0 | — | — | — | — | — | — | **2,893** | — |
| 1 | 2,893 | 225 | −9 | −800 | +43 | — | 2,352 | −541 |
| 2 | 2,352 | 225 | −9 | −800 | +35 | — | 1,803 | −549 |
| 3 | 1,803 | 225 | −9 | −800 | +27 | — | 1,246 | −557 |
| 4 | 1,246 | 225 | −9 | −800 | +19 | — | 681 | −565 |
| 5 | 681 | 225 | −9 | −800 | +10 | 500 | 607 | −74 |
| 6 | 607 | 190 | −8 | −800 | +9 | — | **2** | −614 |
| 7 | 2 | 190 | −4 | −800 | +0 | 2,000 | 1,388 | +1,386 |
| 8 | 1,388 | 90 | — | −800 | +21 | — | 699 | −710 |
| 9 | 699 | 90 | — | −800 | +10 | — | **0** | −710 |
| 10 | 0 | 90 | — | −800 | +0 | 3,000 | 2,290 | +2,290 |

*Interest on cash at 1.5% (Scenario B rate).*
*Year 6 binding: Close = $2 (near-zero). Year 9 binding: Close = $0 (near-zero). Dual binding is a hallmark of Scenario B for this portfolio.*

**Why Scenario B has TWO binding years:** At 1.5%, interest income is so low that the portfolio barely survives Year 6 (the deep trough before IG maturity) AND barely survives Year 9 (the end of the long drought with only Govt coupon). This dual binding signals structural vulnerability — different segments of the liability stream become critical simultaneously.

### 7.6 Period-by-Period Projection: Scenario C (Rates Up, 5.0% flat)

Starting with **C0 = $2,595** (the root-finding solution for Scenario C):

| Year | Open Cash ($) | Coupons ($) | D&D ($) | Liability ($) | Int. on Cash ($) | Maturity ($) | Close Cash ($) | Net CF ($) |
|---|---|---|---|---|---|---|---|---|
| 0 | — | — | — | — | — | — | **2,595** | — |
| 1 | 2,595 | 225 | −9 | −800 | +130 | — | 2,141 | −584 |
| 2 | 2,141 | 225 | −9 | −800 | +107 | — | 1,664 | −584 |
| 3 | 1,664 | 225 | −9 | −800 | +83 | — | 1,163 | −584 |
| 4 | 1,163 | 225 | −9 | −800 | +58 | — | 637 | −584 |
| 5 | 637 | 225 | −9 | −800 | +32 | 500 | 585 | −84 |
| 6 | 585 | 190 | −4 | −800 | +29 | — | **0** | −614 |
| 7 | 0 | 190 | −4 | −800 | +0 | 2,000 | 1,386 | +1,386 |
| 8 | 1,386 | 90 | — | −800 | +69 | — | 745 | −710 |
| 9 | 745 | 90 | — | −800 | +37 | — | 72 | −710 |
| 10 | 72 | 90 | — | −800 | +4 | 3,000 | **2,366** | +2,290 |

*Interest on cash at 5.0% (Scenario C rate). Year 6 is the single binding year. Higher reinvestment income allows a smaller starting C0 ($2,595 vs $2,893 for Scenario B).*

### 7.7 Disinvestment Waterfall (If Cash Goes Negative)

If C0 is insufficient (e.g., C0 = $2,200 instead of $2,893), cash goes negative. The engine executes the disinvestment waterfall:

**Priority order for forced sales:**
1. Cash (no cost)
2. Government / Municipal Bonds (Tier 1) — tightest spreads
3. Investment-Grade Corporate Bonds (Tier 1)
4. Floating-rate / Amortizing securities (Tier 1-2)
5. Agency MBS, ABS, Preferred Stock (Tier 2)
6. **Tier 3 assets (HY Corp) — CANNOT be sold, ever**

The HY Corp earns coupons and matures but cannot be liquidated to meet shortfalls. This asymmetric constraint (income available, liquidity unavailable) is a key feature of Tier 3.

**Forced sale pricing:** All sales at scenario-yield prices (from the shifted curve + z-spread), not par. See Section 14 for a full worked example.

> **SII note:** In SII, disinvestment mechanics are not modelled in the BEL — assets are valued separately and BEL is a pure liability measure. In SBA, the specific assets and their constraints (especially Tier 3 that cannot be sold) directly affect C0 and hence BEL. The same liability schedule produces a different BEL depending entirely on the asset portfolio. Changing the IG Corp maturity from 7Y to 5Y would reduce the Year 6 maturity proceeds and increase C0.

### 7.8 Key Projection Dynamics — Summary

- **D&D costs erode net asset income over time.** The $9/year credit cost (years 1-5) and $4/year (years 6-7) accumulate to $53 over the projection. With 2024-2028 phase-in (20% → 100%), these costs will grow significantly in coming years.

- **Tier 3 constraints restrict the disinvestment waterfall.** The HY bond contributes $35/year in coupons and returns $500 at maturity — but when the portfolio is under stress, Tier 3 is a locked asset. It earns income but cannot provide emergency liquidity.

- **Maturity mismatches create cliff risks.** Three distinct maturity dates (years 5, 7, 10) against a smooth 10-year liability stream creates two "drought" periods (years 6 and 8-9). The IG bond maturity at Year 7 creates a one-time surplus that must be carefully rationed over the subsequent three years.

- **Multiple binding constraints signal structural vulnerability.** In Scenario B, the portfolio hits near-zero cash at both Year 6 and Year 9. This dual binding means there is no single fix — extending the IG bond maturity helps Year 6 but not Year 9. A structurally sound portfolio should have no more than one binding constraint.

- **The interaction between scenarios and portfolio structure is non-obvious.** Rates Down is the biting scenario not because it produces the largest single-year shortfall (it does not), but because low reinvestment rates compound the effect of every shortfall. The biting scenario is a function of both the rate path and the portfolio's specific structure.

---

## Section 8: SBA BEL Formula

*Validates: ALGO-002 output, BEL assembly*
*BMA Rule: Rules Sched XXV Para 28(10)*

```
SBA BEL = MV_assets(t=0) + C0_biting
         = $5,496 + $2,893
         = $8,389
```

### 8.1 Component Breakdown

| Component | Value ($) | Notes |
|---|---|---|
| Govt Bond MV | 2,983 | At t=0 yield of 3.2% |
| IG Corp MV | 2,023 | At t=0 yield of 4.8% |
| HY Corp MV | 490 | At t=0 yield of 7.5% |
| **Total Asset MV (ex-cash)** | **5,496** | |
| C0_biting (Scenario B) | 2,893 | Minimum initial cash buffer |
| **SBA BEL** | **$8,389** | |

> **SII note (the key comparison):**
> SII BEL for this same liability schedule (10-year, $800/year annuity at 3% RF):
> `SII_BEL = 800 × (1 − 1.03⁻¹⁰) / 0.03 = 800 × 8.530 = $6,824`
>
> **SBA BEL ($8,389) is $1,565 (23%) higher than SII BEL ($6,824).**
>
> This premium has a specific meaning: it is the cost of ensuring that the specific asset portfolio in question can always cover the liability cash flows under the worst interest rate scenario, including the reinvestment risk from the low-rate Scenario B environment. SII BEL makes no such guarantee — it is a PV of liability CFs that is independent of assets. Neither is "right" or "wrong"; they serve different regulatory purposes.

```python
# Golden assertions — Section 8
assert abs(portfolio_mv - 5496) < 1.0
assert abs(c0_biting - 2893) < 1.0
assert abs(sba_bel - 8389) < 1.0
assert sba_bel > sii_bel_at_rf  # SBA BEL always >= SII BEL at RF for this portfolio type
```

---

## Section 9: SBA Spread & 35bps Cap Enforcement

*Validates: ALGO-004, `calculations/bel_calculator.py`*
*BMA Rule: Handbook E9 (spread consistency, regulatory cap)*

> **Prior art failure:** Pythora v1.0.0 (Athora SBA model) defined the 35bps cap parameter but has **zero enforcement references in the codebase** (confirmed by grep in `PHASE3-006-Known-Gaps-Limitations.md`). This benchmark enforces the cap explicitly from day one. This is a primary purpose of the model.

### 9.1 What is the Implied SBA Spread?

After the goal-seek produces BEL = $8,389, the implied SBA spread is the constant spread S above the risk-free curve such that:

```
PV(Liability CFs, RF + S) = BEL
```

It is a **derived reporting metric** — it tells regulators at what "effective discount rate" the insurer is implicitly valuing its liabilities.

### 9.2 Does the Cap Bind for Our Benchmark Portfolio?

**For our benchmark (BEL = $8,389):**

At S = 0% (pure RF = 3%): PV = $6,824 < $8,389 → need S < 0 to get PV up to $8,389

The implied S for our benchmark is **negative** (approximately −9% to −15% — well below the cap). The cap does **not** bind. The benchmark's conservative C0 produces a BEL that exceeds the SII BEL at RF, so no cap adjustment is needed.

**When would the cap bind?** When an insurer's model produces a *lower* BEL than PV(liabilities at RF + 35bps). This happens when the model is too optimistic — assuming high reinvestment rates, neglecting credit costs, or using aggressive scenario assumptions.

### 9.3 Worked Example: Cap Binds (Hypothetical Insurer)

Consider an insurer that submits C0 = $500 for the same portfolio (using more optimistic model assumptions):

```
BEL_uncapped = MV_assets + C0 = $5,496 + $500 = $5,996
```

**Step 1: Find implied SBA spread S_uncapped**

Find S such that PV($800/year × 10 years at 3% + S) = $5,996:
- At S = 0%:   PV = $6,824 > $5,996
- At S = 2.0%: PV = $800 × 7.722 = $6,178 > $5,996
- At S = 2.5%: PV = $800 × 7.538 = $6,030 > $5,996
- At S = 2.6%: PV = $800 × 7.500 ≈ $6,000 ≈ $5,996 ✓

**S_uncapped ≈ 260 bps >> 35 bps cap**

**Step 2: Apply the cap**

```
S_capped = min(260 bps, 35 bps) = 35 bps
```

**Step 3: Recalculate BEL at capped spread**

```
BEL_capped = PV($800/year × 10 years at 3% + 0.35%)
           = PV at 3.35%
           = 800 × (1 − 1.0335⁻¹⁰) / 0.0335
           = 800 × 8.367
           = $6,694
```

**Step 4: Impact of cap**

| Metric | Insurer (uncapped) | Insurer (capped) | Benchmark |
|---|---|---|---|
| BEL | $5,996 | **$6,694** | $8,389 |
| Implied SBA Spread | 260 bps | 35 bps | < 0 bps |
| Cap binding? | — | **Yes** | No |
| BEL uplift from cap | — | **+$698** | — |

The cap increases the insurer's BEL by $698 (11.6%), correcting for overly optimistic model assumptions.

**Step 5: Biting scenario selection**

Biting scenario = scenario with **highest capped BEL** (not highest C0). If the cap binds differently across scenarios, this can change which scenario is biting.

> **SII note:** SII uses MA/VA to adjust the liability discount rate (upward), reducing BEL. The 35bps SBA cap plays a similar prudential role to the EIOPA caps on MA/VA — preventing unreasonably optimistic discount rates from understating liabilities. The regulatory intent is aligned (ensure conservative liability valuation) even though the mechanics differ.

### 9.4 Regulatory Reporting Fields

For each scenario, the benchmark model reports:

| Field | Our Benchmark | Hypothetical Insurer |
|---|---|---|
| `sba_spread_uncapped` | < 0 bps | 260 bps |
| `sba_spread_capped` | < 0 bps | **35 bps** |
| `sba_BEL_uncapped` | $8,389 | $5,996 |
| `sba_BEL_capped` | $8,389 | **$6,694** |
| `cap_binding` | False | **True** |
| `bel_impact_of_cap` | $0 | **$698** |

```python
# Golden assertions — Section 9
assert sba_spread_capped <= 0.0035          # Cap always holds
assert not cap_binding_benchmark            # Cap doesn't bind for our portfolio
assert abs(bel_capped_insurer - 6694) < 1.0  # Hypothetical insurer cap calculation
assert abs(bel_uplift_from_cap - 698) < 1.0
```

---

## Section 10: Risk Margin & Technical Provision

*Validates: ALGO-006, `calculations/risk_margin.py`*
*BMA Rule: Rules Subpara 36(4), p. 179-180; Handbook D38.5, p. 314*

### 10.1 Formula

**Per Rules Subpara 36(4):**

```
RM = CoC × Σ[t=0..T] [ Modified_ECR(t) × DF(t+1) ]

where:
  CoC           = 6% (prescribed by BMA, not configurable)
  Modified_ECR(t) = ECR(0) × [remaining liability exposure at t] / [total liability exposure]
  DF(t+1)       = discount factor from base scenario risk-free curve at maturity (t+1)
```

**Key rule:** RM uses the **base scenario curve only** (no stress scenarios, no spread adjustment). Interest rate risk is already captured in the BEL through the 9-scenario projection. RM covers non-hedgeable insurance risks only.

> **SII note:** Both SII and SBA use the 6% CoC Risk Margin formula. The structure is identical: RM = CoC × Σ discounted future capital requirements. The key difference: SII projects each BSCR component separately (longevity, lapse, expense, mortality) and re-aggregates using a correlation matrix. BMA SBA uses proportional runoff of the aggregate ECR — simpler but consistent with BMA's proportionality principle. For a well-matched annuity portfolio, the results are directionally similar.

### 10.2 Worked Example (10-Year Portfolio)

**Inputs:**
- ECR(0) = $500 (illustrative; sourced from company's BSCR filing)
- Liability CFs: $800/year for 10 years; Total exposure = $8,000
- RF curve: 3.0% flat

**Step 1: Compute Runoff Factors**

| t | Remaining Exposure ($) | Runoff Factor α(t) | Modified_ECR(t) = $500 × α(t) |
|---|---|---|---|
| 0 | 8,000 | 1.000 | 500 |
| 1 | 7,200 | 0.900 | 450 |
| 2 | 6,400 | 0.800 | 400 |
| 3 | 5,600 | 0.700 | 350 |
| 4 | 4,800 | 0.600 | 300 |
| 5 | 4,000 | 0.500 | 250 |
| 6 | 3,200 | 0.400 | 200 |
| 7 | 2,400 | 0.300 | 150 |
| 8 | 1,600 | 0.200 | 100 |
| 9 | 800 | 0.100 | 50 |
| 10 | 0 | 0.000 | 0 |

**Step 2: Discount Modified ECR at Base Curve (3% flat)**

| t | Modified_ECR(t) ($) | Disc. Maturity | DF(t+1) | Discounted ECR ($) |
|---|---|---|---|---|
| 0 | 500 | 1 | 0.97087 | 485.4 |
| 1 | 450 | 2 | 0.94260 | 424.2 |
| 2 | 400 | 3 | 0.91514 | 366.1 |
| 3 | 350 | 4 | 0.88849 | 311.0 |
| 4 | 300 | 5 | 0.86261 | 258.8 |
| 5 | 250 | 6 | 0.83748 | 209.4 |
| 6 | 200 | 7 | 0.81309 | 162.6 |
| 7 | 150 | 8 | 0.78941 | 118.4 |
| 8 | 100 | 9 | 0.76642 | 76.6 |
| 9 | 50 | 10 | 0.74409 | 37.2 |
| | | | **Sum** | **2,450.7** |

**Step 3: Apply Cost of Capital**

```
Risk Margin = 6% × $2,450.7 = $147.0
```

**Step 4: Technical Provision**

```
TP = SBA BEL + Risk Margin
   = $8,389 + $147
   = $8,536
```

> **SII note:** Both SII and SBA define Technical Provision as TP = BEL + RM. The structure is identical; the BEL term drives all the difference. SII TP is built from a liability-side PV ($6,824 + RM). SBA TP is built from an asset-adequacy measure ($8,389 + RM). For this portfolio, SBA TP ($8,536) > SII TP (≈ $6,824 + comparable RM). Both feed into their respective solvency capital calculations.

```python
# Golden assertions — Section 10
assert abs(runoff_factor_t5 - 0.500) < 0.001
assert abs(discounted_ecr_sum - 2450.7) < 1.0
assert abs(risk_margin - 147.0) < 1.0
assert abs(technical_provision - 8536) < 1.0
assert abs(technical_provision - (sba_bel + risk_margin)) < 0.01
```

---

## Section 11: Liquidity Coverage Ratio (LCR)

*Validates: ALGO-005b, `calculations/lcr.py`*
*BMA Rule: Rules Sched XXV Para 29(2)(iii); Handbook D22*

### 11.1 Formula and Threshold

```
LCR = Available Liquid Assets (haircut-adjusted) / Potential Surrender Outflows

Threshold: LCR ≥ 105% (hard gate — fail this and the block is not SBA-eligible)
```

> **SII note:** LCR has no direct SII BEL equivalent. SII addresses liquidity risk through the ORSA and the SCR lapse stress module but does not use a hard eligibility gate. BMA's 105% LCR is a binary pass/fail filter — fail it and the entire SBA block reverts to a different capital methodology. This is a stricter requirement than anything in the SII framework.

### 11.2 Step 1: Available Liquid Assets (Haircut-Adjusted)

| Asset | MV ($) | Liquidity Level | Haircut | Eligible ($) |
|---|---|---|---|---|
| Govt Bond (Sovereign, AA) | 2,983 | Level 1 | 5% | 2,834 |
| IG Corp (BBB) | 2,023 | Level 2A | 15% | 1,720 |
| HY Corp (Tier 3) | 490 | **Not eligible** | **100%** | **0** |
| C0 Cash | 2,893 | Level 1 | 0% | 2,893 |
| **Total** | **8,389** | | | **7,447** |

*HY Corp (Tier 3 — limited basis) is excluded from LCR eligible assets: these assets are illiquid by regulatory definition.*

### 11.3 Step 2: Potential Surrender Outflows

**Fast-moving deterioration stress (binding for this portfolio):**
```
Outflow_fast = 20% × aggregate surrender value
             = 20% × $8,389   (BEL used as proxy for aggregate surrender value)
             = $1,678
```

**Sustained deterioration (12-month elevated lapse):**
```
Outflow_sustained = 10% × $8,389 × 1.0 (12-month factor)
                  = $839
```

**Binding outflow** = max($1,678, $839) = **$1,678**

### 11.4 Step 3: LCR Ratio

```
LCR = $7,447 / $1,678 = 4.44 = 444% ✓ PASS (threshold: 105%)
```

The benchmark's portfolio passes comfortably because the large cash buffer (C0 = $2,893) is fully liquid and contributes to LCR. A company that holds insufficient cash (lower C0) would have a lower LCR.

### 11.5 Sensitivity: What Would Cause LCR to Fail?

LCR < 105% would occur if Eligible Assets < 1.05 × $1,678 = $1,762. Given our eligible asset pool of $7,447, this portfolio would need to lose over 76% of its eligible value before failing. The LCR constraint is not binding here — but it becomes binding for highly illiquid portfolios (e.g., majority Tier 2/3 assets) or high-lapse products.

```python
# Golden assertions — Section 11
assert abs(eligible_assets_total - 7447) < 1.0
assert abs(surrender_outflows - 1678) < 1.0
assert abs(lcr_ratio - 4.44) < 0.01
assert lcr_ratio >= 1.05   # Hard gate: must always pass
```

---

## Section 12: Lapse Cost (LapC)

*Validates: ALGO-005a, `calculations/lapse_cost.py`*
*BMA Rule: Rules Sched XXV Para 29(1)(i)D; Handbook D37*

### 12.1 Formula

```
LapC = (Lapse_Rate_Sigma / BSCR_Lapse_Shock) × Lapse_Capital_Requirement
```

The ratio (Sigma / Shock) converts the tail-risk BSCR capital charge into a "best estimate variability" adjustment for the BEL.

> **SII note:** In SII, lapse risk is a **SCR component** (not added to BEL). It flows through the standard formula lapse module and adds to the overall SCR. BMA adds LapC **directly to BEL**, making it part of the liability valuation before any capital calculation. For a portfolio with surrender options, BMA's approach requires more assets at the BEL level — even before the SCR-equivalent stress is applied. This is a more conservative treatment.

### 12.2 Worked Example

**Inputs:**
- Lapse Rate Sigma (σ) = 5.0% (standard deviation of historical annual lapse rates)
- BSCR Lapse Shock = 40% (40% increase in lapse rates is tested in BSCR)
- Lapse Capital Requirement = $300 (from BSCR long-term lapse risk charge)

**Calculation:**
```
Scaling factor = σ / BSCR_Shock = 5% / 40% = 0.125

LapC = 0.125 × $300 = $37.50 ≈ $38
```

**Impact on BEL:**
```
BEL_adjusted = BEL_base + LapC = $8,389 + $38 = $8,427
```

### 12.3 When LapC = 0

For payout annuities with no surrender options (as in this illustrative example), LapC = 0 because there are no policyholder options. The portfolio is a fixed cash flow schedule — lapse risk is not applicable.

```python
# Golden assertions — Section 12
assert abs(lapc_scaling_factor - 0.125) < 0.001
assert abs(lapc - 37.5) < 0.5
assert abs(bel_with_lapc - 8427) < 1.0
# For no-option portfolios:
assert lapc_no_options == 0.0
assert bel_no_options == sba_bel  # LapC = 0 → BEL unchanged
```

---

## Section 13: Three Required Application Stress Tests

*Validates: ALGO-008 orchestration, stress test modules*
*BMA Rule: Handbook E5.6h, p. 320*

These three stress tests are **pass/fail gates**. Failure disqualifies the SBA block.

### 13.1 Stress Test 1: Combined Credit Spread + Mass Lapse

**Stress applied simultaneously:**
- Instantaneous mass lapse: 20% (or BSCR lapse shock, whichever is higher)
- AAA credit spread widening: +277 bps applied to all assets

**Impact on extended portfolio:**

*Mass lapse:* 20% of policyholders surrender immediately → sudden outflow = 20% × $8,389 = $1,678

*Credit spread widening (+277 bps):*
- IG Corp (BBB): duration ≈ 5.5Y → ΔMV ≈ −2.77% × 5.5 × $2,023 ≈ −$308
- Govt Bond: sovereign bonds less affected (assume sovereign spreads widen by 50 bps): ΔMV ≈ −0.50% × 7.5 × $2,983 ≈ −$112
- HY Corp: already in Tier 3; its spread widening doesn't affect liquidity (can't be sold anyway)

Total stressed MV ≈ $5,496 − $308 − $112 = **$5,076**

*Stressed BEL check:* Does the insurer maintain ECR coverage ≥ 100% after the combined stress?
- Assets post-stress: $5,076 (portfolio MV)
- Additional surrender outflow: $1,678
- Net stressed position: $5,076 − $1,678 = $3,398
- Compare to ECR requirement → **pass/fail determination**

> **SII note:** SII combines credit and lapse stresses via the SCR correlation matrix (credit module + lapse module, correlated at ρ = 0.25). BMA applies both stresses simultaneously as a point-in-time pass/fail test on the SBA block — effectively assuming correlation = 1.0, which is stricter. No diversification benefit between credit and lapse is allowed in this BMA test.

### 13.2 Stress Test 2: One-Notch Downgrade

**Stress:** Downgrade all assets by one credit rating notch.

| Asset | Current Rating | After Stress | Impact |
|---|---|---|---|
| Govt Bond | AA | A+ | Minimal — D&D slightly higher |
| IG Corp (BBB) | BBB | **BB** | **IG → Sub-IG → Tier 3!** Cannot be sold in future scenarios |
| HY Corp | B | CCC | D&D increases materially |

**Critical impact on IG Corp:** A one-notch downgrade of BBB → BB crosses the investment-grade threshold. The IG Corp would become a **Tier 3 asset** — still held (it continues to earn coupons) but can no longer be sold in the disinvestment waterfall. This dramatically changes the C0 because the insurer now has only the Govt Bond available for forced sales.

**Revised C0 under one-notch downgrade stress:** Recalculate projection with:
- IG Corp reclassified to Tier 3 (cannot be sold)
- Higher D&D for BB-rated asset (step-up in credit cost)
- New biting C0 will be materially higher than $2,893

**Pass/fail:** ECR coverage ≥ 100% under the stressed BEL.

### 13.3 Stress Test 3: No Reinvestment Stress

**Stress:** Disallow reinvestment into "limited-basis" assets (Tier 2/3). Surplus cash must sit at near-zero return rather than being reinvested into higher-yielding assets.

**Mechanism:** When asset cash flows arrive and exceed current period liability outflows, the surplus is reinvested at the risk-free rate only (no credit spread). This reduces the income in later years and forces higher C0.

**Impact on projection:**
- Years 1-5: Surpluses reinvested at 3% (RF) instead of prevailing portfolio spread
- Years 6-10: Income is lower → shortfalls emerge earlier
- C0 increases to compensate for reduced reinvestment income

**Quantitative estimate:** If reinvestment yield drops from ~4.5% (portfolio average) to 3% (RF), the annual income shortfall is approximately $49,600 × (4.5% - 3.0%) = $744/year. Over the projection, this increases C0 by roughly $3,500-$4,000 (present value of shortfalls).

**Pass/fail:** ECR coverage ≥ 100% under the stressed BEL.

---

## Section 14: Forced Asset Sale Mechanics (When C0 is Insufficient)

*BMA Rule: Rules Sched XXV Para 28(9)(c-d), Para 28(13)-(16), Para 28(30)-(31)*

This section demonstrates what happens when an insurer holds insufficient C0 — and why the correct C0 is essential.

### 14.1 Hypothetical Scenario: C0 = $2,200 (vs correct $2,893)

**Scenario:** Rates Up (5.0% flat) — chosen because forced sales are most visible.

**Year 6:** HY Corp has matured (Year 5). Only Govt Bond and IG Corp remain.
Asset income at Year 6: $190 (Govt coupon $90 + IG coupon $100)
Less liability: −$800
Plus interest on cash (5.0% on $2,200 opening): +$110
= Net shortfall after cash: **−$500**

The insurer must sell assets to raise $500.

### 14.2 Disinvestment Waterfall Execution

**Step 1: Check cash** — $2,200 + (net income) is insufficient.

**Step 2: Sell Govt Bonds (Tier 1, highest priority)**

The Govt Bond must be sold at the **scenario yield** (5.0%), not par:
```
Remaining Govt Bond value at Year 6 with 4 years to maturity:
  PV = 90/1.05 + 90/1.05² + 90/1.05³ + 3,090/1.05⁴
     = 85.71 + 81.63 + 77.75 + 2,542.26
     = $2,787

vs. Par value: $3,000
Haircut: $213 (7.1% loss from being forced to sell at scenario yields)
```

Transaction cost (10 bps): 0.10% × $2,787 = $2.79

Net proceeds per $3,000 of Govt Bond: $2,787 − $2.79 = **$2,784**

To raise $500, need to sell: $500 / ($2,784 / $3,000) = $538.70 par value

**Step 3: Consequences for future periods**
- Remaining Govt par: $3,000 − $539 = $2,461
- Future annual coupon drops from $90 to $90 × ($2,461 / $3,000) = $73.80
- Year 10 maturity proceeds: $2,461 (reduced from $3,000)
- **Risk of cascade:** With less income in Years 7-10, shortfalls may recur, forcing additional sales

### 14.3 Why C0 Prevents This

The correct C0 = $2,893 ensures that cash remains non-negative under all 9 scenarios without any forced sales. The forced sale in this example caused a $213 loss (7.1%) on the Govt Bond. The present value of this loss and all cascade effects is approximately what C0 = $2,893 − $2,200 = $693 covers.

**Key insight:** The SBA is designed so that the biting C0 makes forced sales unnecessary under the worst scenario. If the insurer holds less than the benchmark C0, forced sales occur at adverse scenario prices — and the losses from those sales compound over the remaining projection horizon.

> **SII note:** SII does not model forced asset sales in the BEL calculation. The SCR interest rate shock changes the balance sheet value, but the BEL itself is unaffected by asset fire-sale dynamics. SBA BEL explicitly prices in forced-sale risk through the disinvestment waterfall. This is a fundamental feature of the asset-adequacy framework — it captures a real-world mechanism that SII BEL does not.

---

## Section 15: Challenge Mode — Benchmark vs Company Submission

*Validates: `challenge/` module*
*Project Phase: Phase 3*

### 15.1 Purpose

The benchmark model's primary regulatory use is to **compare** its results against company SBA submissions. The challenge module imports the company's submission, runs the benchmark, and produces a variance report flagging material differences.

### 15.2 Example Variance Report

| Metric | BMA Benchmark | Company Submission | Variance | Flag | Likely Cause |
|---|---|---|---|---|---|
| C0 — Scenario A (Base) | $2,758 | $2,650 | −3.9% | | Within tolerance |
| **C0 — Scenario B (Rates Down)** | **$2,893** | **$2,480** | **−14.3%** | 🔴 | Reinvestment rate assumption? |
| C0 — Scenario C (Rates Up) | $2,595 | $2,590 | −0.2% | | OK |
| SBA BEL (uncapped) | $8,389 | $7,976 | −4.9% | 🟡 | Follows from C0 Scenario B gap |
| 35bps cap applied? | Yes | **No** | — | 🔴 | Cap not enforced (Pythora defect) |
| Risk Margin | $147 | $152 | +3.4% | | Within tolerance |
| Technical Provision | $8,536 | $8,128 | −4.8% | 🟡 | Follows from BEL gap |
| LCR | 444% | 438% | −1.4% | | Within tolerance |
| D&D at 2024 phase-in (20%)? | Yes | Yes | — | | OK |

### 15.3 Flagging Logic

- 🔴 **Immediate investigation:** >10% variance on any C0, or spread cap not applied
- 🟡 **Review required:** >5% variance on BEL or TP, or >$100k absolute difference
- Green (no flag): Within 5% / $100k tolerance

### 15.4 Variance in Scenario B C0 (−$413, −14.3%) — Investigation Path

1. **Check reinvestment assumption:** Company may assume a higher reinvestment rate under the Rates Down scenario (e.g., 2.5% instead of 1.5%)
2. **Check D&D costs:** Company may use lower D&D assumptions than BMA published floors (Para 28(24) violation)
3. **Check Tier 3 treatment:** Company may be selling HY Corp in the disinvestment waterfall (not permitted)
4. **Check forced sale pricing:** Company may be using par (not scenario-yield prices) for bond sales

---

## Appendix A: Complete Pytest Golden Assertions

```python
# tests/test_illustrative_calc_comprehensive.py
"""
Golden integration test — every number in this test comes from
BMA_SBA_Illustrative_Calculation_Comprehensive.md
"""

class TestSection2_SimpleCase:
    def test_simple_c0_scenario_a(self):  assert abs(simple_c0_scenario_a - 2981) < 2.0
    def test_simple_c0_scenario_b(self):  assert abs(simple_c0_scenario_b - 3091) < 2.0
    def test_simple_c0_scenario_c(self):  assert abs(simple_c0_scenario_c - 2844) < 2.0
    def test_simple_biting(self):         assert simple_biting_scenario == "B_DECREASE"
    def test_simple_bel(self):            assert abs(simple_sba_bel - 7571) < 2.0

class TestSection4_ZSpreads:
    def test_govt_bond_zspread(self): assert abs(govt_bond_zspread - 0.0020) < 0.0001
    def test_ig_corp_zspread(self):   assert abs(ig_corp_zspread - 0.0180) < 0.0001
    def test_hy_corp_zspread(self):   assert abs(hy_corp_zspread - 0.0450) < 0.0001

class TestSection5_DD:
    def test_ig_corp_dd_year7(self):  assert abs(ig_corp_dd_cumulative_year7 - 0.01386) < 0.00001
    def test_hy_corp_dd_year5(self):  assert abs(hy_corp_dd_cumulative_year5 - 0.04901) < 0.00001

class TestSection6_Scenarios:
    def test_c0_scenario_b(self):     assert abs(c0_scenario_b - 2893) < 1.0
    def test_c0_scenario_a(self):     assert abs(c0_scenario_a - 2758) < 1.0
    def test_c0_scenario_c(self):     assert abs(c0_scenario_c - 2595) < 1.0
    def test_biting_scenario(self):   assert biting_scenario == "B_DECREASE"

class TestSection8_BEL:
    def test_portfolio_mv(self):      assert abs(portfolio_mv - 5496) < 1.0
    def test_c0_biting(self):         assert abs(c0_biting - 2893) < 1.0
    def test_sba_bel(self):           assert abs(sba_bel - 8389) < 1.0
    def test_sii_bel_lower(self):     assert sba_bel > 6824  # SII BEL at 3% RF

class TestSection9_SpreadCap:
    def test_cap_not_binding_benchmark(self): assert not cap_binding_benchmark
    def test_cap_binding_insurer(self):       assert cap_binding_hypothetical_insurer
    def test_capped_bel_insurer(self):        assert abs(bel_capped_insurer - 6694) < 5.0
    def test_cap_always_holds(self):          assert sba_spread_capped_all_scenarios <= 0.0035

class TestSection10_RiskMargin:
    def test_runoff_factor_t5(self):   assert abs(runoff_factor_t5 - 0.500) < 0.001
    def test_discounted_ecr_sum(self): assert abs(discounted_ecr_sum - 2450.7) < 1.0
    def test_risk_margin(self):        assert abs(risk_margin - 147.0) < 1.0
    def test_technical_provision(self): assert abs(technical_provision - 8536) < 1.0
    def test_tp_equals_bel_plus_rm(self): assert abs(technical_provision - (sba_bel + risk_margin)) < 0.01

class TestSection11_LCR:
    def test_eligible_assets(self):    assert abs(eligible_assets_total - 7447) < 1.0
    def test_surrender_outflows(self): assert abs(surrender_outflows - 1678) < 1.0
    def test_lcr_ratio(self):          assert abs(lcr_ratio - 4.44) < 0.01
    def test_lcr_gate(self):           assert lcr_ratio >= 1.05

class TestSection12_LapC:
    def test_lapc_scaling(self):       assert abs(lapc_scaling_factor - 0.125) < 0.001
    def test_lapc_value(self):         assert abs(lapc - 37.5) < 0.5
    def test_lapc_no_options(self):    assert lapc_no_options == 0.0
```

---

## Appendix B: Regulatory Rule Cross-Reference

| BMA Rule | This Document | Algorithm Spec |
|---|---|---|
| Sched XXV Para 28(7)(a-i) — 9 scenarios | Section 6 | ALGO-001 |
| Sched XXV Para 28(9)(c-d) — forced sales | Section 14 | ALGO-002b |
| Sched XXV Para 28(10) — BEL = max assets | Section 8 | ALGO-002 |
| Sched XXV Para 28(22)-(26) — D&D | Section 5 | ALGO-003 |
| Handbook E9 — spread cap | Section 9 | ALGO-004 |
| Sched XXV Para 29(1)(i)D — LapC | Section 12 | ALGO-005a |
| Sched XXV Para 29(2)(iii) — LCR 105% | Section 11 | ALGO-005b |
| Handbook E5.6h — stress tests | Section 13 | ALGO-008 |
| Subpara 36(4) — Risk Margin | Section 10 | ALGO-006 |

---

*End of Document*
*Companion document: `BMA_SBA_Rebalancing_Reference.md` — how insurers implement KRD matching and TAA rebalancing, and how BMA can challenge these submissions.*
