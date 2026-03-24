# BMA SBA: A Detailed Illustrative Calculation

## 1. Introduction
This document provides a detailed, step-by-step technical guide to the core calculation mechanics of the Bermuda Monetary Authority's (BMA) Scenario-Based Approach (SBA). Its purpose is to serve as a powerful and effective educational tool for practitioners seeking to understand how the SBA framework operates in practice.

Unlike a high-level comparison, this guide focuses exclusively on the SBA methodology. We will use a simplified, hypothetical portfolio to run multiple interest rate scenarios, identify the "Biting Scenario," and determine the final Best Estimate Liability (BEL). Through this process, we will uncover the key risks the SBA is designed to highlight, such as cash flow mismatch and reinvestment risk.

**Disclaimer:** This is a simplified illustration. A real-world application would involve all nine prescribed BMA scenarios, more complex portfolios, stochastic modeling for risks like credit defaults, and formal data governance.

---

## 2. Hypothetical Portfolio & Assumptions

To illustrate the calculation, we will use a simple portfolio with a significant, built-in cash flow mismatch.

### 2.1. Liabilities
A 5-year annuity-style liability that pays out **$1,000** at the end of each year for 5 years.

### 2.2. Assets
To back this liability, the insurer has purchased a single, high-quality corporate bond.
*   **Asset Type:** 5-Year Corporate Bond
*   **Par Value:** $4,500
*   **Coupon:** 4.4% (pays **$198** at the end of each year)
*   **Asset Yield:** 4.5%

### 2.3. Key Assumptions
*   **Base Risk-Free Rate (RFR) and Credit Spread:** For the purpose of establishing the initial market value of the bond at T=0, we assume a base Risk-Free Rate (RFR) of 3.0% and a credit spread of 1.5%. This results in the initial asset yield of 4.5%.
*   **A Single, Fixed Initial Market Value (T=0):** This is a critical concept in the SBA. The starting point for the assessment is the insurer's existing asset portfolio, which has a single, observable market value at time T=0. The five interest rate scenarios are *hypothetical future pathways* that branch out from this single T=0 reality. Therefore, we calculate the bond's market value **only once** based on the T=0 asset yield. This value ($4,480) is the fixed starting point for the assets in **every** scenario. The scenarios will affect the *reinvestment rate* for cash and the *market value* if assets need to be sold mid-projection, but they do not change the initial T=0 value.
*   **Initial Bond Value Calculation:** The market value of the bond at the start (T=0) is the present value of its future cash flows, discounted at its own yield of 4.5%. This is calculated using the standard Discounted Cash Flow (DCF) method:
    
    `MV = C/(1+y)¹ + C/(1+y)² + C/(1+y)³ + C/(1+y)⁴ + (C+P)/(1+y)⁵`
    
    Where:
    *   **MV:** Market Value
    *   **C:** Annual Coupon Payment ($198)
    *   **P:** Par Value ($4,500)
    *   **y:** Yield (4.5% or 0.045)
    
    Plugging in the values:
    
    `MV = 198/1.045 + 198/1.045² + 198/1.045³ + 198/1.045⁴ + (198+4500)/1.045⁵`
    `MV = 189.47 + 181.31 + 173.50 + 166.03 + 3769.82 = $4,480.13`
    
    This value is rounded to **$4,480** for the illustration.

*   **Initial Cash (`C₀`):** The core of the SBA calculation is to determine the amount of initial cash (`C₀`) that must be held alongside the bond to ensure solvency in all scenarios.
*   **Annual Shortfall:** The portfolio has a clear cash flow mismatch. The asset provides $198 each year, while the liability payout is $1,000. This creates a predictable **$802 shortfall** for the first four years that must be funded by the initial cash buffer.
*   **Reinvestment:** Any cash held in the buffer is reinvested at the risk-free interest rate prevailing in each specific scenario.

---

## 3. The SBA Calculation Framework: Minimum vs. Maximum

The SBA's core logic can be a source of confusion. The process involves two main steps:
1.  **For each individual scenario**, we solve for the **minimum** amount of initial cash (`C₀`) required to ensure all liability payments are met without the cash balance ever dropping below zero *within that scenario*.
2.  The final SBA Best Estimate Liability (BEL) is the **maximum** of these required minimums from across all scenarios.

In essence, you determine the minimum needed to survive each possible future, and your final required capital is sized for the most demanding of those futures (the "Biting Scenario").

**SBA BEL = Market Value of Bond + `C₀` from the Biting Scenario**

---

## 4. Scenario Projections

We will now project the portfolio's cash flows under five different interest rate scenarios to find the required initial cash (`C₀`) for each.

### Scenario A: Base Case
*   **Scenario:** No change to interest rates.
*   **Reinvestment Rate:** A flat 3.0% per year.
*   **Logic:** This scenario serves as our baseline. The required cash (`C₀`) is the amount needed to cover the $802 annual shortfall, accounting for the interest earned on the cash buffer.
*   **Required Cash `C_base`:** The `C₀` is the present value of the $802 annual shortfall for 4 years, discounted at the 3.0% reinvestment rate:
    `C_base = 802/(1.03)¹ + 802/(1.03)² + 802/(1.03)³ + 802/(1.03)⁴ = $2,981`
*   **Market Value of Bond (T=0):** $4,480 (Calculated in detail in Section 2.3)
*   **Total Assets for Scenario (Bond + Cash):** $4,480 + $2,981 = **$7,461**

The simulation table below validates this result.

| Year | (A) Opening Cash | (B) Interest Earned (A * 3.0%) | (C) Net Outflow (Liability - Coupon) | (D) Closing Cash (A+B-C) |
|:---|:---|:---|:---|:---|
| 0 | | | | **$2,981** |
| 1 | $2,981 | +$89 | -$802 | $2,268 |
| 2 | $2,268 | +$68 | -$802 | $1,534 |
| 3 | $1,534 | +$46 | -$802 | $778 |
| 4 | $778 | +$23 | -$802 | **-$1** *(due to rounding)* |
| 5 | $0 | +$0 | +$3,698 *(Asset Matures)* | $3,698 *(Surplus)* |

### Scenario B: Rates Down
*   **Scenario:** All interest rates fall instantly and remain low.
*   **Reinvestment Rate:** A flat 1.5% per year.
*   **Logic:** This is a classic reinvestment risk scenario. The interest earned on our cash buffer will be lower, meaning we must start with a larger initial buffer to cover the same shortfall.
*   **Required Cash `C_down`:** The `C₀` is the present value of the $802 annual shortfall for 4 years, discounted at the 1.5% reinvestment rate:
    `C_down = 802/(1.015)¹ + 802/(1.015)² + 802/(1.015)³ + 802/(1.015)⁴ = $3,091`
*   **Market Value of Bond (T=0):** $4,480 (Calculated in detail in Section 2.3)
*   **Total Assets for Scenario (Bond + Cash):** $4,480 + $3,091 = **$7,571**

| Year | (A) Opening Cash | (B) Interest Earned (A * 1.5%) | (C) Net Outflow | (D) Closing Cash (A+B-C) |
|:---|:---|:---|:---|:---|
| 0 | | | | **$3,091** |
| 1 | $3,091 | +$46 | -$802 | $2,335 |
| 2 | $2,335 | +$35 | -$802 | $1,568 |
| 3 | $1,568 | +$24 | -$802 | $790 |
| 4 | $790 | +$12 | -$802 | **$0** |
| 5 | $0 | +$0 | +$3,698 | $3,698 |

### Scenario C: Rates Up
*   **Scenario:** All interest rates rise instantly and remain high.
*   **Reinvestment Rate:** A flat 5.0% per year.
*   **Logic:** The higher return earned on our cash buffer means we need to set aside less initial capital to cover the future shortfalls.
*   **Required Cash `C_up`:** The `C₀` is the present value of the $802 annual shortfall for 4 years, discounted at the 5.0% reinvestment rate:
    `C_up = 802/(1.05)¹ + 802/(1.05)² + 802/(1.05)³ + 802/(1.05)⁴ = $2,844`
*   **Market Value of Bond (T=0):** $4,480 (Calculated in detail in Section 2.3)
*   **Total Assets for Scenario (Bond + Cash):** $4,480 + $2,844 = **$7,324**

| Year | (A) Opening Cash | (B) Interest Earned (A * 5.0%) | (C) Net Outflow | (D) Closing Cash (A+B-C) |
|:---|:---|:---|:---|:---|
| 0 | | | | **$2,844** |
| 1 | $2,844 | +$142 | -$802 | $2,184 |
| 2 | $2,184 | +$109 | -$802 | $1,491 |
| 3 | $1,491 | +$75 | -$802 | $764 |
| 4 | $764 | +$38 | -$802 | **$0** |
| 5 | $0 | +$0 | +$3,698 | $3,698 |

### Scenario D: Twist - Steepener
*   **Scenario:** The yield curve twists, with short-term rates rising less than long-term rates.
*   **Reinvestment Rate:** 3.5% for Year 1, 5.0% for Years 2-4.
*   **Logic:** This tests the portfolio against a change in the *shape* of the yield curve. The calculation is more complex as we cannot use a single rate. We must solve for `C₀` by working backward year-by-year, discounting the remaining shortfall at the prevailing rate for that year.
*   **Required Cash `C_steep`:** Based on the year-by-year backward calculation, the required initial cash is **$2,885**.
*   **Market Value of Bond (T=0):** $4,480 (Calculated in detail in Section 2.3)
*   **Total Assets for Scenario (Bond + Cash):** $4,480 + $2,885 = **$7,365**

| Year | (A) Opening Cash | (B) Interest Rate | (C) Interest Earned | (D) Net Outflow | (E) Closing Cash |
|:---|:---|:---|:---|:---|:---|
| 0 | | | | | **$2,885** |
| 1 | $2,885 | 3.5% | +$101 | -$802 | $2,184 |
| 2 | $2,184 | 5.0% | +$109 | -$802 | $1,491 |
| 3 | $1,491 | 5.0% | +$75 | -$802 | $764 |
| 4 | $764 | 5.0% | +$38 | -$802 | **$0** |
| 5 | $0 | 5.0% | +$0 | +$3,698 | $3,698 |

### Scenario E: Twist - Flattener
*   **Scenario:** The yield curve flattens, with short-term rates falling while long-term rates fall even more.
*   **Reinvestment Rate:** 2.0% for Year 1, 1.5% for Years 2-4.
*   **Logic:** This is another reinvestment stress, similar to the "Rates Down" scenario but with a non-parallel shift. We must solve for `C₀` by working backward year-by-year, discounting the remaining shortfall at the prevailing rate for that year.
*   **Required Cash `C_flat`:** Based on the year-by-year backward calculation, the required initial cash is **$3,076**.
*   **Market Value of Bond (T=0):** $4,480 (Calculated in detail in Section 2.3)
*   **Total Assets for Scenario (Bond + Cash):** $4,480 + $3,076 = **$7,556**

| Year | (A) Opening Cash | (B) Interest Rate | (C) Interest Earned | (D) Net Outflow | (E) Closing Cash |
|:---|:---|:---|:---|:---|:---|
| 0 | | | | | **$3,076** |
| 1 | $3,076 | 2.0% | +$62 | -$802 | $2,336 |
| 2 | $2,336 | 1.5% | +$35 | -$802 | $1,569 |
| 3 | $1,569 | 1.5% | +$24 | -$802 | $791 |
| 4 | $791 | 1.5% | +$12 | -$802 | **$1** *(rounding diff)* |
| 5 | $0 | 1.5% | +$0 | +$3,698 | $3,698 |

---

## 5. Analysis and Final BEL Calculation

### 5.1. Summary of Results
The minimum initial cash (`C₀`) required to survive each scenario is summarized below. The initial bond market value of $4,480 remains constant across all scenarios at time T=0.

| Scenario | Description | Required Initial Cash (`C₀`) | Bond Market Value (T=0) | Total Initial Assets (Bond + Cash) |
|:---|:---|:---|:---|:---|
| A | Base Case (Rates @ 3.0%) | $2,981 | $4,480 | $7,461 |
| C | Rates Up (Rates @ 5.0%) | $2,844 | $4,480 | $7,324 |
| D | Twist - Steepener (Rates 3.5% -> 5.0%) | $2,885 | $4,480 | $7,365 |
| E | Twist - Flattener (Rates 2.0% -> 1.5%) | $3,076 | $4,480 | $7,556 |
| **B** | **Rates Down (Rates @ 1.5%)** | **$3,091** | **$4,480** | **$7,571** |

### 5.2. Determining the Biting Scenario
The **Biting Scenario** is the one that requires the highest amount of initial assets to ensure solvency. Based on our analysis, the biting scenario is **Scenario B: Rates Down**.

The logic is clear: with the lowest overall reinvestment rate (1.5%), the portfolio's cash buffer earns the least income, thus requiring the largest starting amount to cover the fixed $802 annual shortfalls.

### 5.3. Final BEL Calculation
The final SBA Best Estimate Liability is the total initial assets required under the biting scenario.

**SBA BEL = $4,480 (Bond Market Value) + $3,091 (Required Cash for Rates Down) = $7,571**

---

## 6. Key Insights and Advanced Concepts

This detailed calculation provides several key insights into the SBA framework:

*   **SBA: A Solvency Test, Not a Liability Valuation:** This is the most crucial insight. It is natural to think that interest rate scenarios are used to re-calculate the value of liabilities by changing the discount rate. That is how a traditional valuation works, but it is **not** how the BMA SBA operates.
    *   **Traditional Valuation:** Aims to find the *Present Value* of liabilities. If an interest rate scenario lowers the discount rate, the present value of the liabilities goes up.
    *   **BMA SBA:** Aims to prove the existing asset portfolio can generate enough cash to meet all obligations as they fall due. It is a **cash flow adequacy test**. The liability payments are treated as fixed hurdles that must be cleared. The interest rate scenarios are used to stress the *asset side* of the balance sheet to make it harder to clear those hurdles. The scenarios impact:
        1.  **Reinvestment Risk:** The interest earned on any cash buffer. As seen in our example, a "Rates Down" scenario reduces this income, requiring a larger starting buffer (`C₀`).
        2.  **Market Risk:** The price you would get if forced to sell an asset before it matures. In a "Rates Up" scenario, bond prices fall, and a forced sale would result in a capital loss.
    
    In short, the BMA scenarios test the resilience of the asset portfolio against future uncertainty; they are not used to discount the liabilities themselves. The final BEL is the amount of initial assets needed to pass the toughest test, not a restated "value" of the liabilities.

*   **Cash Flow Matching is Paramount:** The large BEL is a direct penalty for the portfolio's significant asset-liability cash flow mismatch. A portfolio with perfectly matched cash flows would require a much smaller (or zero) initial cash buffer.

*   **Reinvestment Risk is a Primary Driver:** The fact that the "Rates Down" scenario is the most stressful demonstrates how sensitive the SBA is to reinvestment risk, especially for portfolios with large, pre-funded cash buffers.

*   **Interaction with the BSCR Calculation:** After calculating the BEL, the next step in the solvency assessment is to calculate the Bermuda Solvency Capital Requirement (BSCR). The BEL is a primary input *into* the BSCR calculation, not the other way around. When a BSCR risk (like mass lapse) is calculated, the impact is determined in one of two ways:
    *   **1. The "Pure" Internal Model (ICM) Approach:** This is the most accurate method. You would apply the shock (e.g., higher lapse rates), which creates a new set of liability cash flows. You would then **re-run the entire SBA simulation** with your original asset portfolio against these new stressed cash flows. The extra capital needed to make the portfolio pass the test again is the capital charge for that risk. This method captures all second-order interaction effects.
    *   **2. The Simplified "Standard Formula" (SF) Approach:** Because the "pure" approach is computationally intensive, regulators often provide a simpler "shock-and-subtract" method. Here, you would calculate a new liability value under the shocked assumptions (often without a full SBA re-run) and subtract the original BEL. The difference is the capital charge. This is simpler but may miss some interaction effects.

*   **Other Risks Captured:** While not modeled in this simple example, a full SBA implementation would also capture other critical risks, such as losses from forced sales of assets in a duration-mismatched portfolio and the impact of credit defaults and downgrades on asset cash flows.

This framework ensures that an insurer's capital and liquidity are sufficient not just on a static valuation basis, but on a dynamic, go-forward basis under a range of plausible stresses.

---

## 7. Extended Illustrative Calculation — Multi-Asset Portfolio with Credit Costs

This section extends the simple one-bond example from Sections 2-6 into a more realistic setting. It introduces multiple assets across different tiers, credit default and downgrade costs (D&D), Tier 3 constraints, maturity cliffs, forced asset sales, and all nine BMA scenarios. The goal is to bridge the gap between the simple pedagogical example and a production-grade model, while keeping every number hand-calculable.

### 7.1. Portfolio Setup

#### 7.1.1. Assets (at T=0)

| # | Asset | Tier | Par | Coupon Rate | Annual Coupon | Yield | Maturity | Market Value (T=0) |
|:--|:------|:-----|:----|:------------|:--------------|:------|:---------|:-------------------|
| 1 | 10-Year Government Bond | Tier 1 | $3,000 | 3.0% | $90 | 3.2% | 10 years | $2,983 |
| 2 | 7-Year IG Corporate Bond (BBB) | Tier 1 | $2,000 | 5.0% | $100 | 4.8% | 7 years | $2,023 |
| 3 | 5-Year HY Corporate Bond | Tier 3 | $500 | 7.0% | $35 | 7.5% | 5 years | $490 |

*   **Initial Cash:** C0 (to be solved)
*   **Total Portfolio Market Value (excluding cash):** $2,983 + $2,023 + $490 = **$5,496**

**Market Value Calculations:**

*   **Asset 1 (Govt Bond):** MV = sum of $90 coupon discounted at 3.2% for years 1-10, plus $3,000 par at year 10. MV = $90 x a(10, 3.2%) + $3,000 x v(10, 3.2%) = $762 + $2,221 = **$2,983**.
*   **Asset 2 (IG Corp):** MV = sum of $100 coupon discounted at 4.8% for years 1-7, plus $2,000 par at year 7. MV = $100 x a(7, 4.8%) + $2,000 x v(7, 4.8%) = $586 + $1,437 = **$2,023**.
*   **Asset 3 (HY Corp):** MV = sum of $35 coupon discounted at 7.5% for years 1-5, plus $500 par at year 5. MV = $35 x a(5, 7.5%) + $500 x v(5, 7.5%) = $142 + $348 = **$490**.

#### 7.1.2. Liabilities

A **10-year annuity** paying **$800** at the end of each year for 10 years.

#### 7.1.3. Key Assumptions

*   **Credit Costs (Default & Downgrade, "D&D"):** These are annual deductions from the portfolio's cash flow, representing expected losses from credit deterioration. They apply only to corporate bonds (not government bonds):
    *   **IG Corporate (BBB):** 0.20% of par per year = $2,000 x 0.20% = **$4/year** (years 1-7)
    *   **HY Corporate:** 1.00% of par per year = $500 x 1.00% = **$5/year** (years 1-5)
*   **Tier 3 Constraint:** The HY bond (Tier 3) earns coupons and matures normally, but it **cannot be sold** before maturity under any circumstances. This restricts the disinvestment waterfall.
*   **Transaction Costs:** 10 basis points (0.10%) on the proceeds of any forced asset sale.
*   **Reinvestment:** All cash earns interest at the scenario-specific risk-free rate.
*   **No reinvestment into new bonds:** When bonds mature, proceeds are held as cash earning the scenario rate. This simplification isolates the effects we want to study.

#### 7.1.4. Cash Flow Structure (Before C0 Interest)

The portfolio's net annual cash flows (excluding interest on the cash buffer) are:

| Year | Govt Coupon | IG Coupon | HY Coupon | D&D (IG) | D&D (HY) | Maturity Proceeds | Liability | **Net Cash Flow** |
|:-----|:-----------|:----------|:----------|:---------|:---------|:------------------|:----------|:------------------|
| 1 | +$90 | +$100 | +$35 | -$4 | -$5 | — | -$800 | **-$584** |
| 2 | +$90 | +$100 | +$35 | -$4 | -$5 | — | -$800 | **-$584** |
| 3 | +$90 | +$100 | +$35 | -$4 | -$5 | — | -$800 | **-$584** |
| 4 | +$90 | +$100 | +$35 | -$4 | -$5 | — | -$800 | **-$584** |
| 5 | +$90 | +$100 | +$35 | -$4 | -$5 | +$500 (HY matures) | -$800 | **-$84** |
| 6 | +$90 | +$100 | — | -$4 | — | — | -$800 | **-$614** |
| 7 | +$90 | +$100 | — | -$4 | — | +$2,000 (IG matures) | -$800 | **+$1,386** |
| 8 | +$90 | — | — | — | — | — | -$800 | **-$710** |
| 9 | +$90 | — | — | — | — | — | -$800 | **-$710** |
| 10 | +$90 | — | — | — | — | +$3,000 (Govt matures) | -$800 | **+$2,290** |

**Key observations from the cash flow structure:**

*   **Years 1-4:** Steady drain of $584/year. Total coupons ($225) less D&D ($9) fall far short of the $800 liability.
*   **Year 5:** HY bond matures, providing $500 of relief. Net drain drops to only $84.
*   **Year 6:** The worst single year — HY bond is gone, IG bond hasn't matured yet. Net drain is $614.
*   **Year 7:** The "maturity cliff" resolves as the IG bond matures, producing a $1,386 surplus.
*   **Years 8-9:** Only the government bond coupon ($90) remains against $800 liabilities. Net drain is $710/year.
*   **Year 10:** Government bond matures with a $2,290 surplus.

The portfolio has two critical pressure points: **Year 6** (deepest single-year drain) and **Years 8-9** (sustained high drain after IG maturity is consumed). The C0 must be large enough to carry the portfolio through both.

### 7.2. Solving for C0 — Methodology

For each scenario, we solve for the **minimum** initial cash C0 such that the cash balance never drops below zero in any year. The method is to **work backward** from Year 10:

1.  Set Required Cash at end of Year 10 = $0.
2.  For each year k (from 10 down to 1), compute the minimum opening cash needed so that: `Opening_Cash x (1 + r_k) + Net_CF_k >= Required_Cash_next_year`, where r_k is the scenario reinvestment rate for year k.
3.  If the computed minimum is negative, set it to zero (you can't have negative cash).
4.  The Required Cash at the start of Year 1 is C0.

### 7.3. Scenario A — Base Case (Flat 3.0%)

*   **Reinvestment Rate:** 3.0% per year, all years.
*   **Required C0:** Solving backward yields **C0 = $2,758**.
*   **Binding Constraint:** Year 6, where cash drops to near zero just before the IG bond maturity at Year 7 provides relief.

| Year | Opening Cash | Bond Coupons | D&D Cost | Liability | Maturity Proceeds | Net CF | Interest on Cash (3.0%) | Closing Cash | Notes |
|:-----|:-------------|:-------------|:---------|:----------|:------------------|:-------|:------------------------|:-------------|:------|
| 0 | — | — | — | — | — | — | — | **$2,758** | Initial cash buffer |
| 1 | $2,758 | +$225 | -$9 | -$800 | — | -$584 | +$83 | $2,257 | |
| 2 | $2,257 | +$225 | -$9 | -$800 | — | -$584 | +$68 | $1,741 | |
| 3 | $1,741 | +$225 | -$9 | -$800 | — | -$584 | +$52 | $1,209 | |
| 4 | $1,209 | +$225 | -$9 | -$800 | — | -$584 | +$36 | $661 | |
| 5 | $661 | +$225 | -$9 | -$800 | +$500 | -$84 | +$20 | $597 | HY bond matures |
| 6 | $597 | +$190 | -$4 | -$800 | — | -$614 | +$18 | **$1** | **Near-zero** (binding) |
| 7 | $1 | +$190 | -$4 | -$800 | +$2,000 | +$1,386 | +$0 | $1,387 | IG bond matures |
| 8 | $1,387 | +$90 | — | -$800 | — | -$710 | +$42 | $719 | |
| 9 | $719 | +$90 | — | -$800 | — | -$710 | +$22 | $31 | |
| 10 | $31 | +$90 | — | -$800 | +$3,000 | +$2,290 | +$1 | $2,322 | Govt bond matures; terminal surplus |

### 7.4. Scenario B — Rates Down (Flat 1.5%)

*   **Reinvestment Rate:** 1.5% per year, all years.
*   **Required C0:** Solving backward yields **C0 = $2,893**.
*   **Binding Constraints:** Year 6 (cash ~$3) and Year 9 (cash ~$0). The lower reinvestment rate means the cash buffer earns less, requiring a larger starting amount.

| Year | Opening Cash | Bond Coupons | D&D Cost | Liability | Maturity Proceeds | Net CF | Interest on Cash (1.5%) | Closing Cash | Notes |
|:-----|:-------------|:-------------|:---------|:----------|:------------------|:-------|:------------------------|:-------------|:------|
| 0 | — | — | — | — | — | — | — | **$2,893** | Initial cash buffer |
| 1 | $2,893 | +$225 | -$9 | -$800 | — | -$584 | +$43 | $2,352 | |
| 2 | $2,352 | +$225 | -$9 | -$800 | — | -$584 | +$35 | $1,803 | |
| 3 | $1,803 | +$225 | -$9 | -$800 | — | -$584 | +$27 | $1,246 | |
| 4 | $1,246 | +$225 | -$9 | -$800 | — | -$584 | +$19 | $681 | |
| 5 | $681 | +$225 | -$9 | -$800 | +$500 | -$84 | +$10 | $607 | HY bond matures |
| 6 | $607 | +$190 | -$4 | -$800 | — | -$614 | +$9 | **$2** | **Near-zero** (binding) |
| 7 | $2 | +$190 | -$4 | -$800 | +$2,000 | +$1,386 | +$0 | $1,388 | IG bond matures |
| 8 | $1,388 | +$90 | — | -$800 | — | -$710 | +$21 | $699 | |
| 9 | $699 | +$90 | — | -$800 | — | -$710 | +$10 | **$0** | **Near-zero** (binding) |
| 10 | $0 | +$90 | — | -$800 | +$3,000 | +$2,290 | +$0 | $2,290 | Govt bond matures; terminal surplus |

**Why Rates Down has TWO binding years:** At 1.5%, interest income is so low that the portfolio barely survives Year 6 (the deep trough before IG maturity) AND barely survives Year 9 (the end of the long drought with only government coupons). This dual binding is a hallmark of portfolios with maturity mismatches — different segments of the liability stream become critical under different rate environments.

### 7.5. Scenario C — Rates Up (Flat 5.0%)

*   **Reinvestment Rate:** 5.0% per year, all years.
*   **Required C0:** Solving backward yields **C0 = $2,595**.
*   **Binding Constraint:** Year 6. Higher rates generate more interest on the cash buffer, so the starting amount can be smaller.

| Year | Opening Cash | Bond Coupons | D&D Cost | Liability | Maturity Proceeds | Net CF | Interest on Cash (5.0%) | Closing Cash | Notes |
|:-----|:-------------|:-------------|:---------|:----------|:------------------|:-------|:------------------------|:-------------|:------|
| 0 | — | — | — | — | — | — | — | **$2,595** | Initial cash buffer |
| 1 | $2,595 | +$225 | -$9 | -$800 | — | -$584 | +$130 | $2,141 | |
| 2 | $2,141 | +$225 | -$9 | -$800 | — | -$584 | +$107 | $1,664 | |
| 3 | $1,664 | +$225 | -$9 | -$800 | — | -$584 | +$83 | $1,163 | |
| 4 | $1,163 | +$225 | -$9 | -$800 | — | -$584 | +$58 | $637 | |
| 5 | $637 | +$225 | -$9 | -$800 | +$500 | -$84 | +$32 | $585 | HY bond matures |
| 6 | $585 | +$190 | -$4 | -$800 | — | -$614 | +$29 | **$0** | **Near-zero** (binding) |
| 7 | $0 | +$190 | -$4 | -$800 | +$2,000 | +$1,386 | +$0 | $1,386 | IG bond matures |
| 8 | $1,386 | +$90 | — | -$800 | — | -$710 | +$69 | $745 | |
| 9 | $745 | +$90 | — | -$800 | — | -$710 | +$37 | $72 | |
| 10 | $72 | +$90 | — | -$800 | +$3,000 | +$2,290 | +$4 | $2,366 | Govt bond matures; terminal surplus |

### 7.6. Summary of All Nine Scenarios

The BMA prescribes nine interest rate scenarios. Below, simplified reinvestment rate structures are used for each. The detailed tables for Scenarios A, B, and C are shown above; the remaining six are computed using the same backward-solving methodology.

| Scenario | Description | Rate Structure | C0 Required | Binding Year(s) |
|:---------|:-----------|:---------------|:------------|:-----------------|
| A | Base Case | 3.0% flat | $2,758 | Year 6 |
| **B** | **Rates Down** | **1.5% flat** | **$2,893** | **Years 6, 9** |
| C | Rates Up | 5.0% flat | $2,595 | Year 6 |
| D | Down-Up | 1.5% yrs 1-3, 4.0% yrs 4-10 | $2,833 | Year 6 |
| E | Up-Down | 5.0% yrs 1-3, 2.5% yrs 4-10 | $2,645 | Year 6 |
| F | Twist 1 (Short Down, Long Up) | 1.5% yrs 1-3, 3.0% yrs 4-7, 5.0% yrs 8-10 | $2,854 | Year 6 |
| G | Twist 2 (Short Up, Long Down) | 5.0% yrs 1-3, 3.0% yrs 4-7, 1.5% yrs 8-10 | $2,638 | Years 6, 9 |
| H | Twist 3 (Parallel Down then Up) | 2.0% yrs 1-5, 4.0% yrs 6-10 | $2,835 | Year 6 |
| I | Twist 4 (Parallel Up then Down) | 4.0% yrs 1-5, 2.0% yrs 6-10 | $2,682 | Year 6 |

### 7.7. Identifying the Biting Scenario

The **Biting Scenario** is Scenario B (Rates Down) with C0 = **$2,893**. This requires the highest initial cash buffer because:

1.  **Lowest reinvestment income:** At 1.5%, the cash buffer earns the least interest, compounding the drain from the annual shortfalls.
2.  **Dual binding constraint:** The portfolio barely survives both Year 6 (the deep trough before the IG bond maturity) and Year 9 (the sustained drain period with only government coupons). No other scenario produces two simultaneous binding constraints.
3.  **D&D costs amplify the stress:** The $9/year credit cost in years 1-5 and $4/year in years 6-7, while modest, further erode the thin cash margins.

### 7.8. Final BEL Calculation

**SBA BEL = Total Portfolio Market Value + C0 from the Biting Scenario**

**SBA BEL = $5,496 + $2,893 = $8,389**

### 7.9. Forced Asset Sale Illustration

The C0 values above are sized so that forced sales are never needed. But what happens if an insurer holds **insufficient** initial cash? This subsection illustrates the mechanics of a forced sale under the Rates Up scenario (Scenario C), where rising rates depress bond prices.

**Setup:** Suppose the insurer holds only **C0 = $2,200** (about $395 less than the required $2,595) under the Rates Up scenario (5.0%).

Tracing the projection forward:

| Year | Opening Cash | Net CF | Interest (5.0%) | Closing Cash | Notes |
|:-----|:-------------|:-------|:-----------------|:-------------|:------|
| 1 | $2,200 | -$584 | +$110 | $1,726 | |
| 2 | $1,726 | -$584 | +$86 | $1,228 | |
| 3 | $1,228 | -$584 | +$61 | $705 | |
| 4 | $705 | -$584 | +$35 | $156 | |
| 5 | $156 | -$84 | +$8 | $80 | HY bond matures |
| 6 | $80 | -$614 | +$4 | **-$530** | **Cash shortfall!** |

At Year 6, the portfolio is **$530 short**. It must sell assets to raise cash.

**Disinvestment waterfall (BMA rules):**

1.  **Cash** — already exhausted.
2.  **Government bonds (Tier 1)** — eligible for sale.
3.  **IG Corporate bonds (Tier 1)** — eligible for sale (but already matured at year 7 in the future, not available to sell early... actually it has not matured yet at year 6, so it IS available).
4.  **HY bond (Tier 3)** — **CANNOT be sold.** This is the Tier 3 constraint in action.

**Selling the Government Bond at Scenario Prices:**

Under the Rates Up scenario (5.0%), the government bond's market value at Year 6 (with 4 years remaining, 3.0% coupon, valued at the 5.0% scenario rate) is:

`MV_yr6 = $90/1.05 + $90/1.05² + $90/1.05³ + $3,090/1.05⁴`
`MV_yr6 = $85.71 + $81.63 + $77.74 + $2,541.82 = $2,786.90`

The bond's par value is $3,000, so selling at $2,787 represents a **$213 loss (7.1% below par)**. This is the mark-to-market penalty from rising rates.

**Executing the forced sale:**

*   Amount needed: $530
*   Proceeds per dollar of par sold: $2,787 / $3,000 = $0.929 per dollar of par
*   Transaction cost: 0.10% of proceeds
*   Par amount to sell: $530 / ($0.929 x 0.999) = **$571 of par** (out of $3,000)
*   Remaining government bond par after sale: $3,000 - $571 = **$2,429**
*   Future annual coupon reduced from $90 to: $2,429 x 3.0% = **$73/year**

**Consequences of the forced sale:**

*   **Immediate:** The $213 loss on the sold portion is crystallized. The insurer receives $530 in cash but gives up $571 of par value.
*   **Ongoing:** Annual coupon income drops by $17/year for the remaining 4 years, further weakening the portfolio's ability to meet future liabilities.
*   **At maturity (Year 10):** The government bond returns only $2,429 instead of $3,000, reducing the terminal surplus by $571.
*   **Cascading risk:** The reduced coupon and par make Years 8-9 even tighter, potentially triggering additional forced sales.

This illustrates why the BMA sizes C0 to **avoid** forced sales — once they begin, the portfolio enters a downward spiral of reduced income and further shortfalls.

**Note on Tier 3:** The HY bond ($490 market value) could theoretically cover the $530 shortfall if sold. But BMA rules classify it as Tier 3, making it ineligible for sale. This is a deliberate regulatory constraint: HY bonds are assumed to be illiquid or to carry unacceptable fire-sale discounts. The Tier 3 classification forces the insurer to sell the higher-quality government bond instead, which — while liquid — generates a mark-to-market loss in a rising-rate environment.

### 7.10. Key Learning Points

This extended example reveals several dynamics that the simple one-bond example could not:

*   **D&D costs erode net asset income over time.** The $9/year credit cost (years 1-5) and $4/year (years 6-7) may seem small, but they accumulate to $53 over the projection. In a portfolio with tighter margins or lower-rated credits, D&D costs can be the difference between solvency and failure. The 2024-2028 BMA phase-in (20% to 100%) means these costs will grow significantly in coming years.

*   **Tier 3 constraints restrict the disinvestment waterfall.** The HY bond contributes $35/year in coupons and returns $500 at maturity — valuable cash flows. But when the portfolio is under stress, the Tier 3 bond is a locked asset. It earns income but cannot provide emergency liquidity. This asymmetry means Tier 3 assets contribute to income but not to resilience.

*   **Maturity mismatches create cliff risks.** The portfolio has three distinct maturity dates (years 5, 7, and 10) but a smooth 10-year liability stream. The result is two "drought" periods (years 6 and 8-9) where outflows far exceed inflows. The IG bond maturity at Year 7 creates a false sense of security — it generates a one-time surplus that must be carefully rationed over the subsequent three years.

*   **Forced sales at scenario prices generate real losses.** In the Rates Up illustration, selling the government bond at Year 6 crystallized a 7.1% loss versus par. Worse, the reduced coupon and par weakened the portfolio for the remaining 4 years. This is the "death spiral" risk that the SBA is specifically designed to detect: under-capitalized portfolios forced to sell assets at depressed prices, further reducing their ability to meet future obligations.

*   **Multiple binding constraints signal structural vulnerability.** In the Rates Down scenario (the biting scenario), the portfolio hits near-zero cash at both Year 6 and Year 9. This dual binding means there is no single "fix" — extending the IG bond maturity would help Year 6 but not Year 9, while adding more short-term assets would help Year 9 but not Year 6. A structurally sound portfolio should have no more than one binding constraint across all years.

*   **The interaction between scenarios and portfolio structure is non-obvious.** Rates Down is the biting scenario not because it produces the largest single-year shortfall (it does not), but because low reinvestment rates compound the effect of every shortfall. A portfolio with better cash flow matching would be less sensitive to rate levels and more sensitive to rate timing (twists). The biting scenario is a function of both the rate path and the portfolio's specific structure.
