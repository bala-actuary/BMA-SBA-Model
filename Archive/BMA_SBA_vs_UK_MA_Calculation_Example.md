# Illustrative Calculation: BMA SBA vs. UK MA

## 1. Introduction
This document provides a simplified, illustrative example of how the BMA's Scenario-Based Approach (SBA) and the UK's Matching Adjustment (MA) work in practice. The goal is to demonstrate the core calculation mechanics of each framework using a small, hypothetical portfolio of assets and liabilities.

**Disclaimer:** This is a highly simplified illustration. Real-world calculations involve complex, granular data, sophisticated modeling platforms, and a much wider range of assumptions (e.g., rating migrations, detailed transaction costs, stochastic modeling).

## 2. Hypothetical Portfolio
Let's assume an insurer has the following single liability block and has purchased assets to back it.

### 2.1. Liabilities
A 5-year annuity-style liability that pays out **$1,000** at the end of each year for 5 years.

*   **Total Liability Payout:** $5,000

### 2.2. Assets
To match these liabilities, the insurer purchases a single, high-quality corporate bond:

*   **Asset Type:** 5-Year Corporate Bond
*   **Par Value:** $4,500
*   **Coupon:** 4.4% (pays **$198** each year)
*   **Credit Spread:** 1.5% (150 bps) over the risk-free rate.
*   **Market Value:** We will determine the required market value under each approach.

### 2.3. Base Economic Assumptions
*   **Risk-Free Rate:** A flat **3.0%** for all maturities.
*   **Asset Yield:** Risk-Free Rate + Credit Spread = 3.0% + 1.5% = **4.5%**.
*   **Asset Sale Haircut (for SBA):** A 1.0% transaction cost is applied if we are forced to sell the bond.
*   **Reinvestment Rate (for SBA):** Surplus cash is reinvested at the prevailing risk-free rate in each scenario.

---

## 3. BMA Scenario-Based Approach (SBA) Calculation
The BMA SBA is a dynamic cash flow simulation. Its methodology can be confusing, so let's clarify the objective.

**Objective: Minimum vs. Maximum Assets**
The process has two main steps:
1.  For **each individual scenario** (Rates Up, Rates Down, etc.), we solve for the **minimum** amount of initial assets required to ensure all liability payments are met without the cash balance ever dropping below zero.
2.  The final SBA Best Estimate Liability (BEL) is then the **maximum** of these required minimums from across all scenarios.

In short, you find the minimum needed to survive each battle, and your total required army is sized for the toughest battle you might face (the "Biting Scenario").

First, let's calculate the market value of our bond at T=0. The value is the present value of its cash flows discounted at its own yield (4.5%).
*   **Bond Market Value (T=0):** PV($198 annual coupon, $4,500 principal @ 4.5% for 5 years) = **$4,480**

Our BEL will be this bond value plus a required initial cash amount (`C₀`) determined by the biting scenario.

### Scenario A: Rates Down (-1.5% instantly)
*   **Reinvestment Rate:** 1.5%
*   **Logic:** A lower rate on our cash holdings means we need to start with a larger cash pile to cover future shortfalls.
*   **Required Cash `C_down`:** PV($802 shortfall for 4 years @ 1.5%) = **$3,091**

The table below proves that if we start with $3,091 in cash, we survive the scenario.

| Year | (A) Opening Cash | (B) Interest Earned (A * 1.5%) | (C) Asset Inflow (Coupon) | (D) Liability Outflow | (E) Closing Cash (A+B+C-D) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 0 | | | | | **$3,091** |
| 1 | $3,091 | +$46 | +$198 | -$1,000 | $2,335 |
| 2 | $2,335 | +$35 | +$198 | -$1,000 | $1,568 |
| 3 | $1,568 | +$24 | +$198 | -$1,000 | $790 |
| 4 | $790 | +$12 | +$198 | -$1,000 | **$0** |
| 5 | $0 | +$0 | +$4,698 (*Principal+Cpn*) | -$1,000 | $3,698 (Surplus) |

### Scenario B: Base Case (No Rate Change)
*   **Reinvestment Rate:** 3.0%
*   **Logic:** The reinvestment rate is higher than the "Rates Down" scenario, so we should need less initial cash.
*   **Required Cash `C_base`:** PV($802 shortfall for 4 years @ 3.0%) = **$2,981**

The simulation below confirms this required cash amount.

| Year | (A) Opening Cash | (B) Interest Earned (A * 3.0%) | (C) Asset Inflow (Coupon) | (D) Liability Outflow | (E) Closing Cash (A+B+C-D) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 0 | | | | | **$2,981** |
| 1 | $2,981 | +$89 | +$198 | -$1,000 | $2,268 |
| 2 | $2,268 | +$68 | +$198 | -$1,000 | $1,534 |
| 3 | $1,534 | +$46 | +$198 | -$1,000 | $778 |
| 4 | $778 | +$23 | +$198 | -$1,000 | **-$1** (rounding diff) |
| 5 | $0 | +$0 | +$4,698 (*Principal+Cpn*) | -$1,000 | $3,698 (Surplus) |

*(The -$1 is due to rounding in the PV calculation; the principle holds.)*

### Scenario C: Rates Up (+2.0% instantly)
*   **Reinvestment Rate:** 5.0%
*   **Logic:** With a high 5% return on cash, we need the least amount of initial cash.
*   **Required Cash `C_up`:** PV($802 shortfall for 4 years @ 5.0%) = **$2,844**

The simulation confirms this is the minimum cash needed.

| Year | (A) Opening Cash | (B) Interest Earned (A * 5.0%) | (C) Asset Inflow (Coupon) | (D) Liability Outflow | (E) Closing Cash (A+B+C-D) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 0 | | | | | **$2,844** |
| 1 | $2,844 | +$142 | +$198 | -$1,000 | $2,184 |
| 2 | $2,184 | +$109 | +$198 | -$1,000 | $1,491 |
| 3 | $1,491 | +$75 | +$198 | -$1,000 | $764 |
| 4 | $764 | +$38 | +$198 | -$1,000 | **$0** |
| 5 | $0 | +$0 | +$4,698 (*Principal+Cpn*) | -$1,000 | $3,698 (Surplus) |

### Key Takeaways from the Simulation
*   **Why is Initial Cash Required?** The portfolio has a fundamental cash flow mismatch. The asset pays a small coupon ($198) each year, while the liability requires a large payment ($1,000). The SBA framework forces the insurer to hold a liquid cash buffer from day one to fund this predictable shortfall.
*   **What is the Final Year Surplus?** The large surplus in Year 5 is the remaining money after the bond matures (paying back its $4,500 principal and final $198 coupon) and the final $1,000 liability is paid. It represents the profit left in this specific portfolio after all obligations have been met.

### Determining the Biting Scenario and BEL
To find the SBA BEL, we compare the required initial assets for each scenario and select the highest value.

*   **Assets for Rates Down:** $4,480 (Bond) + $3,091 (Cash) = **$7,571**
*   **Assets for Base Case:** $4,480 (Bond) + $2,981 (Cash) = **$7,461**
*   **Assets for Rates Up:** $4,480 (Bond) + $2,844 (Cash) = **$7,324**

The **Rates Down** scenario is the **Biting Scenario** because it requires the most initial assets. The lower investment return on the cash buffer makes this the most stressful scenario for this particular portfolio.

**SBA Best Estimate Liability (BEL) = $7,571**

---

## 4. UK Matching Adjustment (MA) Calculation
The MA is a **valuation adjustment**. It does not model cash flows dynamically. Instead, it adjusts the discount rate used to value the liabilities.

**Objective:** Calculate the MA benefit and use it to find the Present Value of the liabilities.

### Step 1: Define the Components
*   **Risk-Free Rate:** 3.0%
*   **Asset Spread:** 1.5% (the extra yield on our corporate bond)

### Step 2: Calculate the Fundamental Spread (FS)
The FS represents the portion of the spread that compensates for credit risk (expected loss, etc.). This is normally calculated based on regulator-prescribed tables and methods. For this illustration, we will make a simplifying assumption.

*   **Assumption:** Let's assume the **Fundamental Spread for our bond is 0.4%**.

### Step 3: Calculate the Matching Adjustment (MA)
The MA is the part of the spread that is *not* related to credit risk—the "illiquidity premium."

*   **MA = Asset Spread - Fundamental Spread**
*   MA = 1.5% - 0.4% = **1.1%**

This 1.1% is the capital benefit. It's the extra yield the insurer is allowed to recognize.

### Step 4: Calculate the MA-Adjusted Discount Rate
This new discount rate is used to value the liabilities.

*   **MA-Adjusted Rate = Risk-Free Rate + MA**
*   MA-Adjusted Rate = 3.0% + 1.1% = **4.1%**

### Step 5: Calculate the Best Estimate Liability (BEL)
The BEL is the present value of the liability cash flows, discounted at the MA-adjusted rate.

| Year | Liability Cash Flow | PV Factor (at 4.1%) | Present Value |
| :--- | :--- | :--- | :--- |
| 1 | $1,000 | 0.9606 | $960.60 |
| 2 | $1,000 | 0.9228 | $922.80 |
| 3 | $1,000 | 0.8864 | $886.40 |
| 4 | $1,000 | 0.8515 | $851.50 |
| 5 | $1,000 | 0.8180 | $818.00 |
| **Total**| **$5,000** | | **$4,439.30** |

**UK MA Best Estimate Liability (BEL) = $4,439**

---

## 5. Comparison and Conclusion

| Metric | BMA SBA (Illustrative) | UK MA (Illustrative) |
| :--- | :--- | :--- |
| **Methodology** | Dynamic Cash Flow Simulation | Static Valuation Adjustment |
| **Result (BEL)** | **$7,571** | **$4,439** |
| **Source of Benefit** | Proving portfolio resilience across interest rate scenarios by ensuring full cash flow adequacy. | Adding a calculated "illiquidity premium" (the MA) to the discount rate to lower the liability valuation. |

This revised example reveals the core difference in philosophy much more starkly:

*   The **BMA SBA** is a stringent, operational solvency test. It asks: *"Do you have sufficient cash and liquid assets on day one to withstand prescribed stresses and meet every single payment on time?"* The calculation correctly identifies the severe cash flow mismatch in the hypothetical portfolio and forces the BEL to cover the entire funding gap from the start.

*   The **UK MA** is an economic valuation principle. It asks: *"What is the 'true' economic value of your liabilities, assuming you hold matching assets to maturity?"* It provides a direct, formulaic reduction to the liability value. It does not explicitly test the year-to-year cash flow adequacy and implicitly assumes the portfolio is better matched than it is in this example.

The significant difference in results ($7,571 vs. $4,439) demonstrates that the SBA is not just a valuation tool but a rigorous test of a firm's asset-liability management and liquidity planning. It penalizes cash flow mismatches heavily, which is central to its design.
