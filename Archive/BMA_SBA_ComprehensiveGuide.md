Here is a comprehensive guide to the **Bermuda Monetary Authority’s (BMA) Scenario-Based Approach (SBA)**. This document breaks down the regulation into its essential elements, referencing the specific Rules and Handbook sections as requested.

***

# Comprehensive Guide to the BMA Scenario-Based Approach (SBA)

## 1. Introduction and Scope
The Scenario-Based Approach (SBA) is a dynamic modelling method used to determine the Best Estimate Liability (BEL) for long-term insurance business. Unlike the Standard Approach, which uses prescribed discount curves, the SBA determines the BEL based on the quantity of assets required to support liability cash flows under a range of interest rate scenarios and rigorous stress tests.

*   **Primary Regulation:** *Insurance (Prudential Standards) (Insurance Group Solvency Requirement) Rules 2011* (as amended 2024).
*   **Guidance:** *2024 Year-End Insurance Group Instructions Handbook*.

## 2. Eligibility and Governance
To utilize the SBA, an insurer must demonstrate that its asset and liability portfolios are well-matched and subject to robust governance.

### A. Eligibility Criteria
*   **Well-Matched Portfolios:** The insurer must demonstrate that the asset and liability portfolios are "well-matched." This requires a formal assessment including the cost of mismatch, dispersion of results across scenarios, and ALM positioning tolerances.
    *   *Ref:* Rules Sched XXV Para 28(5); Handbook E4.
*   **No Splitting of Contracts:** Insurers cannot split underlying policyholder contracts to achieve SBA eligibility.
    *   *Ref:* Rules Sched XXV Para 28(4).
*   **Currency Matching:** Assets backing liabilities must be in the same currency. Mismatches must be hedged and effectively managed.
    *   *Ref:* Rules Sched XXV Para 28(6).
*   **Lapse Risk Eligibility:** If policyholder options exist (e.g., surrender), the insurer must demonstrate residual risk is insignificant. This includes holding a "Lapse Cost" (LapC) within the BEL and passing specific lapse stress tests (100% ECR under Lapse Up/Down).
    *   *Ref:* Rules Sched XXV Para 29(2).

### B. Governance & Attestations
*   **Board Approval:** The Board must approve the initial use of SBA and any "major" model changes.
    *   *Ref:* Rules Sched XXV Para 40(a).
*   **Attestations:** Annual attestations are required from the Chief Risk Officer (CRO), Chief Investment Officer (CIO), Chief Actuary, and Chief Executive Officer (CEO) regarding model adequacy, reinvestment strategies, and default/downgrade assumptions.
    *   *Ref:* Handbook E2; Rules Sched XXV Para 36, 41(c).
*   **Model Change Policy:** A policy must be in place defining "major" vs. "minor" changes. Major changes require BMA approval.
    *   *Ref:* Handbook E3; Rules Sched XXV Para 40(e).

## 3. Methodology: The Core Mechanics
The SBA calculates the BEL by projecting asset and liability cash flows under specific scenarios.

### A. The 9 Interest Rate Scenarios
The insurer must project cash flows across nine prescribed interest rate scenarios. The scenario requiring the highest amount of assets to cover liabilities is the "Biting Scenario," which determines the BEL.
*   *Ref:* Rules Sched XXV Para 28(7).

1.  **Base Scenario:** No adjustment to current rates.
2.  **Decrease:** Rates decrease annually to -1.5% in year 10, flat thereafter.
3.  **Increase:** Rates increase annually to +1.5% in year 10, flat thereafter.
4.  **Pop-Up / Pop-Down:** Rates decrease to -1.5% in year 5, then return to base by year 10.
5.  **Pop-Down / Pop-Up:** Rates increase to +1.5% in year 5, then return to base by year 10.
6.  **Decrease with Positive Twist:** Short rates down (-1.5%), Long rates down less (-0.5%).
7.  **Decrease with Negative Twist:** Short rates down (-0.5%), Long rates down more (-1.5%).
8.  **Increase with Positive Twist:** Short rates up (+0.5%), Long rates up more (+1.5%).
9.  **Increase with Negative Twist:** Short rates up (+1.5%), Long rates up less (+0.5%).

### B. Projection Mechanics
*   **Cash Flow Matching:** At each time step (at least annually), liability cash flows are compared to asset cash flows.
    *   *Shortfalls:* Must be met by selling assets at the scenario's prevailing market yields.
    *   *Excesses:* Must be reinvested according to the defined reinvestment strategy.
    *   *Ref:* Rules Sched XXV Para 28(9).
*   **Fungibility:** Fungibility of assets between blocks of business is generally **not allowed** unless explicitly demonstrated and legally/contractually permissible.
    *   *Ref:* Rules Sched XXV Para 37(f).

## 4. Asset Classifications and Constraints
Assets are categorized by their admissibility and governance requirements within the SBA.

### A. Acceptable Assets (Unrestricted)
These can be used without specific prior approval (though standard reporting applies).
*   Government Bonds, Municipal Bonds, Public Corporate Bonds (Investment Grade), Cash.
    *   *Ref:* Rules Sched XXV Para 28(13).

### B. Assets Requiring Prior Approval
These require BMA approval to be included in the SBA.
*   Private Assets, Structured Securities (MBS, ABS, CLO), Residential/Commercial Mortgage Loans, Investment Grade Preferred Stock.
    *   *Ref:* Rules Sched XXV Para 28(14); Handbook E7.

### C. Assets Acceptable on a Limited Basis (10% Limit)
Certain assets can be used but are capped at **10%** of the total SBA portfolio value.
*   Below Investment Grade assets, Commercial Real Estate, Credit Funds.
*   *Constraint:* These cannot be sold to meet cash flow shortfalls in the model.
    *   *Ref:* Rules Sched XXV Para 28(15)-(16).

### D. Long-Term Investment Credit (LTIC)
For liabilities with duration >30 years, insurers may apply to use "not-acceptable" assets (e.g., equities) to back the tail liabilities.
*   *Method:* The model assumes these assets are converted into acceptable assets as the liability horizon shortens to 30 years.
    *   *Ref:* Rules Sched XXV Para 28(17)-(18).

## 5. Assumptions and Adjustments
To ensure prudence, the BMA mandates specific adjustments to the raw cash flows.

### A. Default and Downgrade Costs
Projected asset cash flows must be **reduced** to account for defaults and rating migration.
*   **Methodology:** Use realized average default losses (baseline) + uncertainty margin (downgrade).
*   **Floors:** Assumptions cannot be lower than BMA published tables for comparable assets.
*   **Transitional Arrangement:** For business in force as of Dec 31, 2023, the downgrade cost component is phased in over 5 years (20% in 2024, 100% by 2028).
    *   *Ref:* Rules Sched XXV Para 28(22)-(26); Handbook E10.

### B. Transaction Costs (Bid-Ask Spreads)
The model must reflect the full expected price impact of buying/selling assets.
*   **Guidance:** If current spreads are tighter than long-term averages, a grading-in to the long-term average must be applied. Illiquid assets must assume higher costs than public equivalents.
    *   *Ref:* Rules Sched XXV Para 28(30)-(31); Handbook E11.

### C. Reinvestment Strategy
*   Assumptions must vary by rating and tenor.
*   Cannot assume "overperformance" vs. historical market averages persists indefinitely.
*   Reinvestment into "Limited Basis" assets (e.g., below investment grade) is generally restricted in stress tests.
    *   *Ref:* Rules Sched XXV Para 28(33).

### D. Risk-Free Curve
Insurers must use either the BMA published curve or a relevant market curve (e.g., Swap/Govt) without adjustments. Spreads must be consistent with the chosen curve.
*   *Ref:* Handbook E9.

## 6. Application Stress Tests
As part of the SBA application (and ongoing validation), insurers must run specific severe stress tests to demonstrate robustness.

### A. Combined Credit Spread and Mass Lapse Stress
*   **Mass Lapse:** Instantaneous lapse of the higher of **20%** or the BSCR product-specific shock.
*   **Credit Spreads:** Instantaneous widening in Year 1 (e.g., AAA: +277bps, BBB: +498bps).
    *   *Ref:* Handbook E5.6h(i).

### B. One-Notch Downgrade Stress
*   Apply a one-notch rating downgrade to **all** assets.
*   If the downgrade pushes assets into "Limited Basis" or ineligible categories, the model must account for the resulting caps or force the use of the Standard Approach for that portion.
    *   *Ref:* Handbook E5.6h(ii).

### C. No Reinvestment Stress
*   Model a scenario where the insurer cannot reinvest into "Assets acceptable on a limited basis."
    *   *Ref:* Handbook E5.6h(iii).

## 7. Liquidity Risk Management
Because the SBA relies on asset selling/holding strategies, liquidity is a distinct pillar of the regulation.

*   **Liquidity Coverage Ratio (LCR):** Must pass a minimum **105% LCR**.
*   **Liquidity Buffer:** A pool of highly liquid assets must be set aside to meet deficiencies.
*   **Contingency Plan:** A formal "playbook" for meeting liquidity deficits, tested via dry-runs.
    *   *Ref:* Rules Sched XXV Para 31; Para 29(2)(iii).

## 8. Derivatives
Derivatives (e.g., Interest Rate Swaps, FX Forwards) can be used in the SBA but are subject to strict scrutiny.

*   **Risk Mitigation Only:** Must be for risk mitigation (hedging), not speculation.
*   **Modelling:** Must capture collateral/margin requirements, basis risk, and residual risks (risks not perfectly hedged).
*   **Approvals:** Specific application required detailing the hedging strategy and governance.
    *   *Ref:* Handbook E8; Rules Sched XXV Para 30(1).

***

### Summary Checklist for Task 2 Modeling
Based on the above, the model you design for Task 2 must capably handle:
1.  **Multi-Scenario Projection:** Running 9 prescribed interest rate paths simultaneously.
2.  **Dynamic Rebalancing:** Logic to sell assets (at haircut prices) or reinvest (at scenario yields) based on cash flow timing.
3.  **Granular Asset Modeling:** Distinguishing between asset classes (Gov vs. Private vs. Limited) to apply specific caps (10% limit) and stresses (downgrade costs).
4.  **Stress Testing Flexibility:** Ability to overlay "Mass Lapse" or "Spread Widening" shocks on top of the base SBA runs.
5.  **Audit Trail:** Tracking the "Biting Scenario" selection and the resulting BEL.