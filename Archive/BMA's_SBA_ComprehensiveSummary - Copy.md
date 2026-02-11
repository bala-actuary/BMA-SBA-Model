# Comprehensive Overview of the Bermuda Monetary Authority's (BMA) Scenario-Based Approach (SBA)

## Executive Summary
The **Scenario-Based Approach** (SBA) is a sophisticated methodology introduced under the Insurance (Prudential Standards) (Insurance Group Solvency Requirement) Amendment Rules 2024 and detailed in the 2024 Year-End Insurance Group Instructions Handbook. It is designed for insurance groups, particularly those engaged in long-term business, to calculate Best Estimate Liabilities (BEL) and associated risk charges by projecting asset and liability cash flows under multiple predefined interest rate scenarios. This approach emphasizes asset-liability matching (ALM), risk management, and governance, allowing for reduced capital requirements in well-matched portfolios while ensuring solvency and policyholder protection. SBA integrates with the broader Bermuda Solvency Capital Requirement (BSCR) framework, replacing standard discounting methods with scenario-based projections.<br><br> It applies to eligible contracts with minimal policyholder options and requires BMA approval. Key elements include nine interest rate scenarios, detailed asset eligibility criteria, capital adjustments, and robust model governance. This document synthesizes essential rules and handbook guidance, cross-referenced for verification, to provide a complete understanding for implementation and regulatory compliance.

## 1. Introduction to SBA
The SBA is a principle-based framework for determining technical provisions and solvency requirements for insurance groups under BMA supervision. It focuses on projecting cash flows to assess the assets needed to meet liabilities under stress, promoting better ALM and reducing reliance on simplistic discounting. SBA is elective for groups with well-matched portfolios but can be mandated by the BMA based on factors like matching quality, optionality, business complexity, and supervisory judgment. It applies primarily to long-term insurance business but integrates with general business risks and other BSCR modules (e.g., fixed income, equity, credit). Proportionality is key: methods must align with the group's scale, nature, and complexity, with error assessments required.

- **Purpose and Scope**: To calculate BEL by considering interest rate risks through scenarios, ensuring liabilities are backed by sufficient, predictable assets. Excludes contracts with significant policyholder options unless demonstrated as negligible via modeling and stress testing. No contract splitting for eligibility.
  - Rules: Para. 28(1-6), pp. 159-162; Handbook: E1, D19, pp. 315, 221-229.
- **Integration with BSCR/ECR/TCL**: SBA feeds into the Economic Balance Sheet (EBS) valuations and BSCR calculations, including diversification benefits via correlation matrices.<br> *Enhanced Capital Requirement (ECR)* is the higher of *Minimum Solvency Margin (MSM)* or BSCR/approved model;<br> *Target Capital Limit (TCL)* is 120% ECR.
  - Rules: Para. 28(10), p. 160; Handbook: D19.2, D19.16-21, pp. 221-229.
- **Key Principles**: ALM emphasis, prudent reinvestment/disinvestment, inclusion of Events Not in Data Set (ENIDS) for future risks, and governance to mitigate model risks.
  - Rules: Para. 28(9), p. 160; Para. 34(11) (ENIDS), p. 178; Handbook: E3-E4, pp. 317-319.

## 2. Key Definitions
Understanding SBA requires familiarity with core terms, which guide calculations and eligibility.

| Term | Definition | Cross-Reference |
|------|------------|-----------------|
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

## 3. Eligibility, Election, and Application
- **Eligibility Criteria**: Applicable to long-term business with no/minimal policyholder options (e.g., annuities). Must demonstrate via ALM tests, stress testing, liquidity management, and GSSA. Hold LapC, maintain 100% ECR under lapse stresses, and 105% Liquidity Coverage Ratio (LCR). No eligibility for high-optionality products.
  - Rules: Para. 28(2-4), 29(1-2), pp. 159, 175; Handbook: E5, pp. 319-321.
- **Election and Mandates**: Groups elect SBA; BMA may mandate or revert to standard approach. Requires passing combined stress tests (e.g., credit spread widening + mass lapse).
  - Rules: Para. 29(1), p. 175; Handbook: E1, E5.6h, pp. 315, 320 (Stress Table: AAA +277 bps, etc.).
- **Application Package**: Submit detailed package for approval, including model overview, validation reports, data quality, governance policies, asset/liability models, well-matched assessments, and stress test results. Pre-engagement recommended; approval in 4-8 weeks.
  - Rules: Subpara. 30, p. 176; Handbook: E5, pp. 319-321 (includes ALM strategy, policies for model risk, data, changes, validation).
- **Asset Approvals**: Separate approvals for non-acceptable assets (e.g., structured securities, derivatives). Limits: ≤10% for limited-basis assets; diversification required. For long-term (>30 years), allow equities with capital adjustments.
  - Rules: Subparas. 13-19, pp. 161-163; Handbook: E6-E8, pp. 322-329 (derivatives for risk mitigation only; no dynamic hedging).

## 4. Scenarios and Projections
SBA uses nine interest rate scenarios to project cash flows annually.

- **Scenarios Overview**:
  1. Base: No change.
  2-3. Parallel shift: ±1.5% over 10 years.
  4-5. Hump shifts: ±1.5% over 5 years, then reverse.
  6-7. Decrease with positive/negative twist (e.g., Year 1: -1.5/-0.5%; Year 30: -0.5/-1.5%).
  8-9. Increase with positive/negative twist (symmetric to 6-7).
  - Rules: Para. 28(7-8), pp. 159-160; Handbook: E9, pp. 330-331 (risk-free curve based on AA sovereigns or market rates).
- **Projection Mechanics**: Use current portfolio; build future yield curves from spot rates. For shortfalls, sell assets at scenario yields; for excesses, reinvest per allocation (maintain rating/tenor). Include default/downgrade costs (historical +1SD margin), transaction costs (bid-ask, liquidity impact), and no management actions.
  - Rules: Subparas. 33-34, pp. 164-166; Handbook: E10-E11, pp. 331-335 (defaults: 40% recovery; downgrades phased 20-100% by 2028).
- **Additional Stresses**: Lapse (up/down/mass), liquidity (105% LCR), combined credit/mass lapse.
  - Rules: Subpara. 29(2), p. 175; Handbook: D37, E5.6h, pp. 310-320.

## 5. Asset Requirements and Management
- **Asset Categories**:
  - Acceptable: Investment-grade governments, cash.
  - Require Approval: Other fixed income, structured.
  - Limited: Below-investment grade (≤10%, annual review).
  - Rules: Subparas. 13-16, pp. 161-162; Handbook: E6-E7, pp. 322-326.
- **Capital Adjustment for Non-Acceptable Assets**: BEL differential (with/without), yields reduced by 1SD; positive only, BMA-limited.
  - Rules: Subpara. 18, pp. 162-163; Handbook: E7, p. 326.
- **Reinvestment/Disinvestment**: Maintain ALM; no borrowing; CIO attestation.
  - Rules: Subparas. 33-36, pp. 164-166; Handbook: E11, p. 335.
- **Fungibility and Assignment**: Assets identifiable, not encumbered; fungibility demonstrated via tests.
  - Rules: Subpara. 37, pp. 166-167; Handbook: E4.3, p. 319.

## 6. Calculations
- **BEL Calculation**: Highest assets across scenarios for fungible blocks; adjust for stresses.
  - Formula: Assets = Base Coverage + Max(Stress Adjustments).
  - Rules: Para. 28(10), p. 160; Handbook: D19.1, p. 221.
- **Lapse Cost (LapC)**: (Sigma / Shock) × Capital Req.
  - Rules: Subpara. 29(1)(i)D, p. 175; Handbook: D37, pp. 310-313.
- **Liquidity Coverage Ratio (LCR)**: Eligible Sources / Outflows ≥105%.
  - Rules: Subpara. 29(2)(iii), p. 175; Handbook: D22, p. 253.
- **Risk Margin**: CoC (6%) × Discounted Future Modified ECR.
  - Rules: Subpara. 36(4), p. 179; Handbook: D38.5, p. 314.
- **BSCR Integration**: Basic BSCR = √(Σ Corr × Risk Charges); includes fixed income (factors 0-25%), equity (shocks 0.6-45%), credit (δ × Debtor), etc.
  - Rules: Tables 1A-2B (implied); Handbook: D20-D28, pp. 232-289 (e.g., Fixed Income: Gov't 5%; Credit Spread Up: AAA +277 bps).
- **Transitional Provisions**: For pre-2016 business; linear phase-in over 16 years.
  - Rules: Subpara. (3), p. 181; Handbook: D38, p. 314.

## 7. Governance and Model Risk Management
- **Board and Committees**: Approve SBA use/changes; challenge assumptions; policies for risk management/data.
  - Rules: Subparas. 40-41, pp. 169-173; Handbook: E2-E3, pp. 315-317 (attestations, model change policy).
- **Model Risk**: Define materiality; independent validation every 3 years; quantify uncertainties; annual reviews.
  - Rules: Subpara. 41(a-k), pp. 170-174; Handbook: E3, p. 317.
- **Data Policy**: Complete, accurate, appropriate; justify external data.
  - Rules: Subpara. 39, pp. 168-169; Handbook: E5, p. 319.
- **Liquidity Risk Management**: Board-approved program; stress scenarios; contingency plans.
  - Rules: Subpara. 31, pp. 176-177; Handbook: D22, p. 253.

## 8. Reporting and Approvals
- **Annual Reporting**: Submit lapse/liquidity/SBA templates; affiliate assets require approval.
  - Rules: Subparas. 32-33, p. 177; Handbook: E2, p. 315.
- **Material Changes**: 30-day notification; BMA approval if impacting ECR.
  - Rules: Implied in Para. 29; Handbook: E3, p. 317.
- **General/Long-Term Specifics**: Include Bound But Not Incepted (BBNI); project per contract/group.
  - Rules: Subparas. 34-35, pp. 178-179; Handbook: D29-D37, pp. 289-313.

## 9. Integration with Broader Risks
SBA interacts with other BSCR modules:
- **Market Risks**: Fixed income/equity shocks applied to EBS values.
  - Handbook: D20-D22, pp. 232-253.
- **Insurance Risks**: Mortality/morbidity/longevity/lapse/expense stresses on BEL.
  - Handbook: D29-D37, pp. 289-313.
- **Catastrophe Risks**: Net Probable Maximum Loss (PML) adjustments.
  - Handbook: D28, pp. 279-289.
- **Diversification**: Via correlation matrices reducing aggregated charges.
  - Handbook: D19.2, p. 221.

This framework ensures SBA is robust, flexible, and aligned with international standards, providing a foundation for modeling innovations in Task 2.

(References drawn from primary documents for accuracy.)