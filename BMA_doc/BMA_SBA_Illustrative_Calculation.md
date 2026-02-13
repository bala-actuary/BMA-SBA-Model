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
*   **Initial Bond Value:** The market value of the bond at the start (T=0) is the present value of its future cash flows, discounted at its own yield of 4.5%. This is calculated using the standard Discounted Cash Flow (DCF) method:
    
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
