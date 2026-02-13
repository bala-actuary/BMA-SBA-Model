# Consolidated Guide to the Bermuda Monetary Authority's (BMA) Scenario-Based Approach (SBA)

## Executive Summary
The **Scenario-Based Approach** (SBA) is a sophisticated methodology under the Insurance (Prudential Standards) (Insurance Group Solvency Requirement) Amendment Rules 2024 and detailed in the 2024 Year-End Insurance Group Instructions Handbook. It is designed for insurance groups, particularly those engaged in long-term business, to calculate Best Estimate Liabilities (BEL) and associated risk charges by projecting asset and liability cash flows under multiple predefined interest rate scenarios.<br><br> SBA emphasizes asset-liability matching (ALM), risk management, and governance, allowing for reduced capital requirements in well-matched portfolios while ensuring solvency and policyholder protection. It integrates with the broader Bermuda Solvency Capital Requirement (BSCR) framework, replacing standard discounting methods with scenario-based projections. This approach applies to eligible contracts with minimal policyholder options and requires BMA approval. Key elements include nine interest rate scenarios, detailed asset eligibility criteria, capital adjustments, and robust model governance.<br><br> This document synthesizes essential rules and handbook guidance to provide a complete understanding for implementation and regulatory compliance.

## 1. Introduction and Scope
The SBA is a principle-based framework for determining technical provisions and solvency requirements for insurance groups under BMA supervision. It focuses on projecting cash flows to assess the assets needed to meet liabilities under stress, promoting better ALM and reducing reliance on simplistic discounting.

*   **Purpose and Scope**: To calculate BEL by considering interest rate risks through scenarios, ensuring liabilities are backed by sufficient, predictable assets. It applies primarily to long-term insurance business but integrates with general business risks and other BSCR modules. Proportionality is key: methods must align with the group's scale, nature, and complexity.
    *   *Ref:* Rules Para. 28(1-6), pp. 159-162; Handbook E1, D19, pp. 315, 221-229.
*   **Eligibility**: SBA is elective for groups with well-matched portfolios but can be mandated by the BMA. Excludes contracts with significant policyholder options unless demonstrated as negligible via modeling and stress testing. No contract splitting for eligibility.
    *   *Ref:* Rules Para. 28(2-4); Handbook E1, E4.1.
*   **Integration with BSCR/ECR/TCL**: SBA feeds into the Economic Balance Sheet (EBS) valuations and BSCR calculations, including diversification benefits. Enhanced Capital Requirement (ECR) is the higher of Minimum Solvency Margin (MSM) or BSCR/approved model; Target Capital Limit (TCL) is 120% ECR.
    *   *Ref:* Rules Para. 28(10), p. 160; Handbook D19.2, D19.16-21, pp. 221-229.

## 2. Key Definitions
Understanding SBA requires familiarity with core terms, which guide calculations and eligibility.

| Term | Definition | Cross-Reference |
|:-----|:-----------|:----------------|
| **Scenario-Based Approach (SBA)** | Projection of asset/liability cash flows under nine interest rate scenarios to determine BEL and risk charges, focusing on ALM. | Rules: Para. 28(1), p. 159; Handbook: E1, p. 315. |
| **Base Scenario** | Projections with no interest rate adjustments, used as a benchmark. | Rules: Para. 28(7)(a), p. 159; Handbook: E9, pp. 330-331. |
| **Stress Scenarios** | Eight predefined adjustments to yield curves (increases/decreases, twists). | Rules: Para. 28(7)(b-i), p. 159; Handbook: E5.6h, p. 320. |
| **Best Estimate Liabilities (BEL)** | Assets required to cover projected liability cash flows under the biting scenario (highest requirement). | Rules: Para. 28(10)(a-c), p. 160; Handbook: D19.1, p. 221. |
| **Net Cash-Flows** | Asset inflows minus liability outflows in each projection period. | Rules: Para. 28(9)(b), p. 160; Handbook: E4.3, p. 319. |
| **Cash-Flow Shortfall/Excess** | Negative/positive net cash flows triggering asset sales/purchases. | Rules: Para. 28(9)(c-d), p. 160; Handbook: E11, p. 335. |
| **Biting Scenario** | The scenario requiring the most assets for a liability block. | Rules: Subpara. 11, p. 160; Handbook: E4, p. 318. |
| **Fungibility** | Ability to treat liabilities/blocks as interchangeable; requires demonstration and BMA approval. | Rules: Subpara. 37(f), p. 166; Handbook: E5, p. 319. |
| **Lapse Cost (LapC)** | Adjustment for lapse risk: (Lapse Rate Sigma / BSCR Shock) × Lapse Capital Requirement. | Rules: Subpara. 29(1)(i)D, p. 175; Handbook: D37, pp. 310-313. |
| **Unsellable Assets** | Non-traded or encumbered assets; require BMA approval with haircuts. | Rules: Subpara. 34(g), p. 166; Handbook: E6-E7, pp. 322-326. |
| **Well-Matched Portfolio** | Assets aligned with liabilities in timing, currency, and predictability, minimizing basis risk. | Rules: Para. 28(5), p. 159; Handbook: E4, pp. 318-319. |
| **Risk Margin (RM)** | Cost-of-capital addition to BEL: CoC × ∑ (Modified ECR_t / (1 + r_{t+1})^{t+1}). | Rules: Subpara. 36, pp. 179-180; Handbook: D38.5, p. 314. |

## 3. Eligibility and Governance
To utilize the SBA, an insurer must demonstrate that its asset and liability portfolios are well-matched and subject to robust governance.

### 3.1. Eligibility Criteria
*   **Well-Matched Portfolios**: The insurer must demonstrate that asset and liability portfolios are "well-matched" through formal assessment, including cost of mismatch, dispersion of results across scenarios, and ALM positioning tolerances.
    *   *Ref:* Rules Sched XXV Para 28(5); Handbook E4.
*   **No Splitting of Contracts**: Insurers cannot split underlying policyholder contracts to achieve SBA eligibility.
    *   *Ref:* Rules Sched XXV Para 28(4).
*   **Currency Matching**: Assets backing liabilities must be in the same currency. Mismatches must be hedged and effectively managed.
    *   *Ref:* Rules Sched XXV Para 28(6).
*   **Lapse Risk Eligibility**: If policyholder options exist, the insurer must demonstrate residual risk is insignificant. This includes holding a "Lapse Cost" (LapC) within the BEL and passing specific lapse stress tests (100% ECR under Lapse Up/Down).
    *   *Ref:* Rules Sched XXV Para 29(2); Handbook D37, pp. 310-313.
*   **Liquidity Coverage Ratio (LCR)**: Must maintain 105% LCR.
    *   *Ref:* Rules Sched XXV Para 29(2)(iii), p. 175; Handbook D22, p. 253.
*   **Combined Stress Tests**: Requires passing combined stress tests (e.g., credit spread widening + mass lapse).
    *   *Ref:* Rules Para. 29(1), p. 175; Handbook E1, E5.6h, pp. 315, 320.

### 3.2. Governance & Attestations
*   **Board Approval**: The Board must approve the initial use of SBA and any "major" model changes.
    *   *Ref:* Rules Sched XXV Para 40(a).
*   **Attestations**: Annual attestations are required from the Chief Risk Officer (CRO), Chief Investment Officer (CIO), Chief Actuary, and Chief Executive Officer (CEO) regarding model adequacy, reinvestment strategies, and default/downgrade assumptions.
    *   *Ref:* Handbook E2; Rules Sched XXV Para 36, 41(c).
*   **Model Change Policy**: A policy must be in place defining "major" vs. "minor" changes. Major changes require BMA approval.
    *   *Ref:* Handbook E3; Rules Sched XXV Para 40(e).

## 4. Scenarios and Projections
SBA uses nine interest rate scenarios to project cash flows annually. The scenario requiring the highest amount of assets to cover liabilities is the "Biting Scenario," which determines the BEL.

### 4.1. The 9 Interest Rate Scenarios
*   *Ref:* Rules Sched XXV Para 28(7); Handbook E9, pp. 330-331.
1.  **Base Scenario:** No adjustment to current rates.
2.  **Decrease:** Rates decrease annually to -1.5% in year 10, flat thereafter.
3.  **Increase:** Rates increase annually to +1.5% in year 10, flat thereafter.
4.  **Down-Up Scenario:** Rates decrease to -1.5% in year 5, then return to base by year 10.
5.  **Up-Down Scenario:** Rates increase to +1.5% in year 5, then return to base by year 10.
6.  **Decrease with Positive Twist:** Short rates down (-1.5%), Long rates down less (-0.5%).
7.  **Decrease with Negative Twist:** Short rates down (-0.5%), Long rates down more (-1.5%).
8.  **Increase with Positive Twist:** Short rates up (+0.5%), Long rates up more (+1.5%).
9.  **Increase with Negative Twist:** Short rates up (+1.5%), Long rates up less (+0.5%).

### 4.2. Projection Mechanics
*   **Cash Flow Matching**: At each time step (at least annually), liability cash flows are compared to asset cash flows.
    *   *Shortfalls:* Must be met by selling assets at the scenario's prevailing market yields.
    *   *Excesses:* Must be reinvested according to the defined reinvestment strategy.
    *   *Ref:* Rules Sched XXV Para 28(9).
*   **Fungibility**: Fungibility of assets between blocks of business is generally **not allowed** unless explicitly demonstrated and legally/contractually permissible.
    *   *Ref:* Rules Sched XXV Para 37(f).
*   **No Management Actions**: Projections should generally exclude discretionary management actions unless explicitly approved.
    *   *Ref:* Rules Subpara. 33-34, pp. 164-166.

## 5. Asset Requirements and Management
Assets are categorized by their admissibility and governance requirements within the SBA.

### 5.1. Asset Categories
*   **Acceptable Assets (Unrestricted)**: Can be used without specific prior approval.
    *   Government Bonds, Municipal Bonds, Public Corporate Bonds (Investment Grade), Cash.
    *   *Ref:* Rules Sched XXV Para 28(13).
*   **Assets Requiring Prior Approval**: Require BMA approval.
    *   Private Assets, Structured Securities (MBS, ABS, CLO), Residential/Commercial Mortgage Loans, Investment Grade Preferred Stock.
    *   *Ref:* Rules Sched XXV Para 28(14); Handbook E7.
*   **Assets Acceptable on a Limited Basis (10% Limit)**: Capped at **10%** of the total SBA portfolio value.
    *   Below Investment Grade assets, Commercial Real Estate, Credit Funds.
    *   *Constraint:* These cannot be sold to meet cash flow shortfalls in the model.
    *   *Ref:* Rules Sched XXV Para 28(15)-(16).
*   **Long-Term Investment Credit (LTIC)**: For liabilities with duration >30 years, insurers may apply to use "not-acceptable" assets (e.g., equities) to back the tail liabilities.
    *   *Ref:* Rules Sched XXV Para 28(17)-(18).

### 5.2. Reinvestment/Disinvestment
*   Maintain ALM; no borrowing; CIO attestation. Reinvestment of excess cash flow should be in line with the current asset allocation and consistent with the group's ALM and investment policies.
    *   *Ref:* Rules Subparas. 33-36, pp. 164-166; Handbook E11, p. 335.
*   **Derivatives**: Can be used for risk mitigation (hedging) only, not speculation. Modelling must capture collateral/margin requirements, basis risk, and residual risks. Specific application required. Dynamic hedging is not allowed.
    *   *Ref:* Handbook E8; Rules Sched XXV Para 30(1).

## 6. Assumptions and Adjustments
To ensure prudence, the BMA mandates specific adjustments to the raw cash flows.

### 6.1. Default and Downgrade Costs
Projected asset cash flows must be **reduced** to account for defaults and rating migration.
*   **Methodology**: Use realized average default losses (baseline) + uncertainty margin (downgrade). Assumptions cannot be lower than BMA published tables.
*   **Transitional Arrangement**: Downgrade cost component is phased in over 5 years for business in force as of Dec 31, 2023 (20% in 2024, 100% by 2028).
    *   *Ref:* Rules Sched XXV Para 28(22)-(26); Handbook E10.

### 6.2. Transaction Costs (Bid-Ask Spreads)
The model must reflect the full expected price impact of buying/selling assets.
*   **Guidance**: If current spreads are tighter than long-term averages, a grading-in to the long-term average must be applied. Illiquid assets must assume higher costs than public equivalents.
    *   *Ref:* Rules Sched XXV Para 28(30)-(31); Handbook E11.

### 6.3. Risk-Free Curve
Insurers must use either the BMA published curve or a relevant market curve (e.g., Swap/Govt) without adjustments. Spreads must be consistent with the chosen curve.
*   *Ref:* Handbook E9.

## 7. Application Stress Tests
As part of the SBA application (and ongoing validation), insurers must run specific severe stress tests.

*   **Combined Credit Spread and Mass Lapse Stress**: Instantaneous mass lapse (higher of 20% or BSCR shock) combined with instantaneous widening of credit spreads (e.g., AAA: +277bps).
    *   *Ref:* Handbook E5.6h(i).
*   **One-Notch Downgrade Stress**: Apply a one-notch rating downgrade to **all** assets.
    *   *Ref:* Handbook E5.6h(ii).
*   **No Reinvestment Stress**: Model a scenario where the insurer cannot reinvest into "Assets acceptable on a limited basis."
    *   *Ref:* Handbook E5.6h(iii).

## 8. Calculations
*   **BEL Calculation**: Highest assets across scenarios for fungible blocks; adjust for stresses. Formula: Assets = Base Coverage + Max(Stress Adjustments).
    *   *Ref:* Rules Para. 28(10), p. 160; Handbook D19.1, p. 221.
*   **Lapse Cost (LapC)**: (Sigma / Shock) × Capital Req.
    *   *Ref:* Rules Subpara. 29(1)(i)D, p. 175; Handbook D37, pp. 310-313.
*   **Liquidity Coverage Ratio (LCR)**: Eligible Sources / Outflows ≥105%.
    *   *Ref:* Rules Subpara. 29(2)(iii), p. 175; Handbook D22, p. 253.
*   **Risk Margin**: CoC (6%) × Discounted Future Modified ECR.
    *   *Ref:* Rules Subpara. 36(4), p. 179; Handbook D38.5, p. 314.
*   **BSCR Integration**: Basic BSCR = √(Σ Corr × Risk Charges); includes fixed income (factors 0-25%), equity (shocks 0.6-45%), credit (δ × Debtor), etc.
    *   *Ref:* Rules Tables 1A-2B; Handbook D20-D28, pp. 232-289.
*   **Transitional Provisions**: For pre-2016 business; linear phase-in over 16 years.
    *   *Ref:* Rules Subpara. (3), p. 181; Handbook D38, p. 314.

## 9. Governance and Risk Management
*   **Board and Senior Management Oversight**: The Board is responsible for approving the use of SBA and any major changes. Clear governance structure required.
    *   *Ref:* Rules: Para. 28(40)(a); Handbook E2-E3.
*   **Model Risk**: Define materiality; independent validation every 3 years; quantify uncertainties; annual reviews.
    *   *Ref:* Rules Subpara. 41(a-k), pp. 170-174; Handbook E3, p. 317.
*   **Data Policy**: Complete, accurate, appropriate; justify external data.
    *   *Ref:* Rules Subpara. 39, pp. 168-169; Handbook E5, p. 319.
*   **Liquidity Risk Management**: Board-approved program; stress scenarios; contingency plans. Must pass a minimum 105% LCR. A pool of highly liquid assets must be set aside.
    *   *Ref:* Rules Subpara. 31, pp. 176-177; Handbook D22, p. 253.

## 10. Application and Approval Process

### 10.1. Application Package
To use the SBA, an insurance group must submit a comprehensive application package to the BMA, including:
*   Evidence that eligibility requirements are met.
*   Board sign-off.
*   Completed Lapse, Liquidity, and SBA reporting templates.
*   Full SBA model calculations.
*   Assessment of the asset/liability portfolio being well-matched.
*   Detailed SBA methodology documentation.
*   Validation reports.
*   Summary and analysis of stress testing results.
*   Policies on model risk, data quality, model change, and ALM.
    *   *Ref:* Handbook E5.1-E5.11.

### 10.2. BMA Review and Approval
The BMA will review the application package. The review timeframe is expected to be 4-8 weeks for complete and high-quality submissions. Approval is subject to the BMA's discretion.
*   *Ref:* Handbook E5.10, E5.11.

## 11. Integration with Broader Risks
SBA interacts with other BSCR modules:
*   **Market Risks**: Fixed income/equity shocks applied to EBS values.
    *   *Ref:* Handbook D20-D22, pp. 232-253.
*   **Insurance Risks**: Mortality/morbidity/longevity/lapse/expense stresses on BEL.
    *   *Ref:* Handbook D29-D37, pp. 289-313.
*   **Catastrophe Risks**: Net Probable Maximum Loss (PML) adjustments.
    *   *Ref:* Handbook D28, pp. 279-289.
*   **Diversification**: Via correlation matrices reducing aggregated charges.
    *   *Ref:* Handbook D19.2, p. 221.

## 12. Summary Checklist for Modeling
Based on this guide, an SBA model should capably handle:
1.  **Multi-Scenario Projection**: Running 9 prescribed interest rate paths simultaneously.
2.  **Dynamic Rebalancing**: Logic to sell assets (at haircut prices) or reinvest (at scenario yields) based on cash flow timing.
3.  **Granular Asset Modeling**: Distinguishing between asset classes (Gov vs. Private vs. Limited) to apply specific caps (10% limit) and stresses (downgrade costs).
4.  **Stress Testing Flexibility**: Ability to overlay "Mass Lapse" or "Spread Widening" shocks on top of the base SBA runs.
5.  **Audit Trail**: Tracking the "Biting Scenario" selection and the resulting BEL.