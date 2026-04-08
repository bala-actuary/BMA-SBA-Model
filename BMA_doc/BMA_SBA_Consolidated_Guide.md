# Consolidated Guide to the BMA Scenario-Based Approach (SBA) — v2.0

> **Version history:**
> - v1.0 (2025): Initial extraction via Gemini from Rules and Handbook PDFs.
> - v2.0 (2026-04-08): Comprehensive rewrite verified against source PDFs. Corrected inaccuracies (twist scenarios, BEL formula, transitional provisions), added missing Paras 1–27 EBS framework, expanded reinvestment/disinvestment/governance sections, added Handbook E6–E11 detail.
>
> **Source documents:**
> - Rules: *Insurance (Prudential Standards) (Insurance Group Solvency Requirement) Amendment Rules 2024* — Schedule XXV, Paras 1–37 + Instructions.
> - Handbook: *2024 Year-End Insurance Group Instructions Handbook* — Sections D19, D22, D35, E1–E11.

---

## Executive Summary

The **Scenario-Based Approach (SBA)** is the BMA's alternative to the Standard Approach for calculating Best Estimate Liabilities (BEL) for long-term insurance business. It projects asset and liability cash flows under **nine prescribed interest rate scenarios**, determines the highest asset requirement (the "biting scenario"), and sets BEL equal to that amount. SBA emphasises asset-liability matching, governance, and transparency, and integrates with the broader BSCR framework.

This guide consolidates the Rules (Schedule XXV) and Handbook (Sections D/E) into a single implementation reference.

---

## 1. EBS Valuation Framework (Schedule XXV, Part 1: Paras 1–7)

These foundational principles underpin all SBA calculations.

- **GAAP fair-value basis**: EBS produced on consolidated GAAP basis. Where GAAP permits both fair-value and non-economic models, fair value must be used. *(Para 1–2)*
- **Valuation hierarchy** (when GAAP doesn't require economic valuation): (a) quoted prices in active markets for same/similar; (b) adjusted quotes for similar; (c) mark-to-model using maximum observable inputs. *(Para 3)*
- **No own-credit adjustment**: Liability valuation shall not reflect the insurer's own credit standing. *(Para 4)*
- **Off-balance-sheet**: All contractual/contingent liabilities from off-balance-sheet arrangements recognised on EBS. *(Para 7)*

---

## 2. Technical Provisions — Application Principles (Part 2: Paras 8–9)

- **Proportionality** (Para 8): Methods must be proportionate to nature, scale and complexity. Simplifications allowed (deterministic vs stochastic, aggregate vs policy-by-policy, scaling/mapping). Error from simplification must not misstate TP or underestimate risk.
- **Expert judgement** (Para 9): May be used but shall not replace data analysis. Must be documented, justified, and performed by qualified persons. Financial market projection models must be risk-neutral, arbitrage-free, and consistent with risk-free term structure.
- **Assumptions** (Para 9(5)): Must be clear, justified, consistent with portfolio characteristics, applied consistently over time, and reflective of cash flow uncertainty.

---

## 3. General Calculation Principles (Part 3: Paras 10–14)

- **TP = BEL + Risk Margin**, calculated separately (unless TP-as-a-whole under Para 37). *(Para 10(2))*
- **Transfer value**: TP shall correspond to current arm's-length transfer cost. *(Para 10(4))*
- **Segmentation**: Obligations segmented into homogeneous risk groups. *(Para 11)*
- **Contract boundaries** (Para 12): Recognise at earlier of contract date or cover start. Boundary ends when insurer can reassess risk and fully reprice, or is no longer required to provide coverage.
- **Data quality** (Para 13): Internal processes to ensure completeness, accuracy, appropriateness.
- **Back-testing** (Para 14): BEL calculations compared against experience; systematic deviations require adjustment.

---

## 4. Best Estimate — General Provisions (Part 4: Paras 15–26)

### 4.1 Overview (Para 15)
BEL = probability-weighted average of future cash flows, discounted using relevant risk-free term structure with illiquidity adjustment. Must allow for uncertainty, use up-to-date assumptions, and be calculated gross and net of reinsurance separately.

### 4.2 Cash Flows (Para 16)
All future in/outflows within contract boundaries, including: (a) premiums, (b) benefits, (c) benefits-in-kind, (d) expenses, (e) investment costs, (f) intermediary payments, (g) salvage/subrogation, (h) taxation, (i) reinsurance (with counterparty default allowance), (j) any other policyholder charges.

Uncertainty types: timing/frequency/severity, claim amounts/inflation, expenses, future developments, policyholder behaviour, dependencies, path-dependency.

### 4.3 Expenses (Para 17)
Admin, claims management, acquisition, investment, overhead. Closed-book expense considerations required. For SBA, investment expenses based on actual portfolio (not hypothetical). Projected forward with appropriate inflation.

### 4.4 Multi-Currency (Para 18)
Discount by currency using relevant risk-free term structure per currency. Convert to reporting currency at valuation-date FX rates.

### 4.5 Reinsurance Recoveries (Para 19)
Outwards reinsurance BEL calculated consistently with gross BEL principles. Include reinstatement premiums and admin expenses.

### 4.6 Counterparty Default (Para 20)
Adjust reinsurance recoveries for expected losses due to counterparty default (insolvency, dispute). Calculated as expected PV of change in cash flows from default at each point in time.

### 4.7 Management Actions (Para 22)
May be reflected if: documented, CEO-approved, consistent with past practice and policyholder representations, realistic, timely, legal, and mutually consistent. **Critical: Para 22(4) — Management actions do NOT apply to reinvestment/disinvestment assumptions in SBA.** SBA reinvestment/disinvestment governed by Para 28(33)–(34) instead.

### 4.8 Policyholder Behaviour (Para 23)
Analysis of past behaviour and prospective assessment considering: data, financial incentive, operating environment, management actions, economic conditions, other influences, recapture possibility.

### 4.9 Guarantees & Options (Para 25)
All material financial/non-financial guarantees and contractual options must be identified, valued, and modelled.

### 4.10 BEL Calculation Method (Para 26)
Must be reviewable by competent person, reflect risks, use relevant data, group policies into homogeneous risk groups, and analyse dependency of PV on expected vs. deviated outcomes.

### 4.11 Standard Approach (Para 27)
Standard discounting uses BMA-prescribed risk-free curves with illiquidity adjustment. SBA is the alternative to this.

---

## 5. Introduction and Scope of SBA (Para 28(1)–(6))

- **Elective or mandated**: Insurance groups may elect SBA for some or all long-term business, subject to eligibility. BMA may mandate SBA or Standard Approach at its discretion. *(Para 28(1)–(2))*
- **BMA discretion criteria**: degree of ALM, optionality in liabilities, nature/scale/complexity, supervisory review results. *(Para 28(3))*
- **BMA discretion — Handbook context (E1.1–E1.3)**: Exercise will be "principle-based, is expected to be rare," after "due diligence and engagement with relevant stakeholders." If regulatory requirements are being met, "there is little need for the Authority to exercise this discretion."
- **No contract splitting**: Cannot split policyholder contracts to achieve SBA eligibility. *(Para 28(4))*
- **Well-matched portfolios required**. *(Para 28(5))*
- **Currency matching**: Assets must be in same currency as liabilities; mismatches must be hedged. *(Para 28(6))*

---

## 6. Key Definitions

| Term | Definition | Reference |
|:-----|:-----------|:----------|
| **SBA** | Projection of asset/liability cash flows under 9 interest rate scenarios to determine BEL. | Para 28(1); Handbook E1 |
| **Base Scenario** | No adjustment to current rates; used as benchmark. | Para 28(7)(a) |
| **Stress Scenarios** | 8 predefined yield curve adjustments (increases, decreases, twists). | Para 28(7)(b)–(i) |
| **BEL** | Highest asset requirement across all 9 scenarios. | Para 28(10)(c) |
| **Net Cash Flows** | Asset inflows minus liability outflows in each projection period. | Para 28(9)(b) |
| **Biting Scenario** | The scenario requiring the most assets for a (fungible) liability block. | Para 28(11) |
| **Fungibility** | Ability to aggregate liability blocks; requires demonstration + BMA written approval. | Para 28(11), 28(37)(f) |
| **LapC (Lapse Cost)** | (Lapse Rate Sigma ÷ BSCR lapse shock) × Lapse up/down capital requirement. | Para 29(1)(i)D |
| **LCR** | Eligible Liquidity Sources ÷ Liability Outflows ≥ 105%. | Para 29(2)(iii) |
| **Risk Margin** | CoC × Σ [ModECR_t / (1 + r_{t+1})^{t+1}], CoC = 6%. | Para 36(4) |
| **Well-Matched Portfolio** | Assets aligned with liabilities in timing, currency, and predictability. | Para 28(5); Handbook E4 |
| **Approved Actuary** | Independent review and opinion role, separate from Chief Actuary attestation. | Handbook E2.5, E6.7 |

---

## 7. The 9 Interest Rate Scenarios

*Ref: Rules Para 28(7)(a)–(i); Handbook E9*

### 7.1 Parallel Shift Scenarios

| # | Scenario | Description |
|---|----------|-------------|
| (a) | **Base** | No adjustment to rates. |
| (b) | **Decrease** | All rates decrease annually to total -1.5% by Year 10; flat thereafter. |
| (c) | **Increase** | All rates increase annually to total +1.5% by Year 10; flat thereafter. |
| (d) | **Down-Up** | All rates decrease to -1.5% by Year 5, then return to base by Year 10. |
| (e) | **Up-Down** | All rates increase to +1.5% by Year 5, then return to base by Year 10. |

### 7.2 Twist Scenarios (3-point interpolation: Year 1, Year 10, Year 30)

| # | Scenario | Year 1 Spot | Year 10 Spot | Year 30 Spot |
|---|----------|:-----------:|:------------:|:------------:|
| (f) | **Decrease with Positive Twist** | -1.5% | -1.0% | -0.5% |
| (g) | **Decrease with Negative Twist** | -0.5% | -1.0% | -1.5% |
| (h) | **Increase with Positive Twist** | +0.5% | +1.0% | +1.5% |
| (i) | **Increase with Negative Twist** | +1.5% | +1.0% | +0.5% |

All twist adjustments are the **net change after ten years**, with intermediate durations interpolated.

---

## 8. Yield Curve Construction (Para 28(8))

**This paragraph is critical for implementation.**

Insurance groups shall determine future yield curves under each scenario as follows:

1. **Convert initial spot rates to forward rates.**
2. **Build future spot rate curves** (at years 1, 2, 3, ...) using the forward rates from step 1.
3. **Apply scenario adjustments** from Para 28(7) to the spot rate curves from step 2, to determine the spot rate curve at each future year along each scenario.

---

## 9. Risk-Free Curve (Handbook E9.1–E9.5)

- **Choice**: Either (a) BMA-published risk-free curve, or (b) relevant market curve (govt bond or swap rates per currency convention) with no adjustments. *(E9.1)*
- **No extrapolation**: Market curve kept flat beyond last traded tenor point. *(E9.2)*
- **Consistency**: Documented, used consistently; change requires BMA approval. *(E9.3)*
- **Spreads**: Must be consistent with chosen risk-free curve. *(E9.4)*
- **Idiosyncratic spread adjustment** *(E9.5 — critical for implementation)*: The choice of risk-free curve shall not distort initial asset market values. For each asset, a spread adjustment is determined such that:

  > PV(projected asset cash flows, using chosen risk-free rates + applicable spreads + spread adjustment) = actual market value of the asset.

  This adjustment must be carried through all SBA projections for that specific asset. Required whenever the market benchmark curve underlying the asset's actual market value differs from the chosen SBA risk-free curve.

---

## 10. Projection Mechanics (Para 28(9)–(10))

### 10.1 Within-Projection Requirements (Para 28(9))
- **(a)** Spot rate curves from Para 28(8)(c) applied together with **assumed spreads for each modelled asset class** to calculate yields and prices at purchase/sale.
- **(b)** At least annually, liability cash flows compared to asset cash flows → net cash flows.
- **(c) Shortfalls** (negative NCF): sell assets at scenario yields to cover.
- **(d) Excesses** (positive NCF): purchase assets at scenario yields per investment/reinvestment guidelines.

### 10.2 BEL Calculation (Para 28(10))
1. Using the **current asset portfolio** and reinvestment requirements per Para 28(33), determine assets required to cover liability cash flows under the **base scenario**.
2. Determine under **each of the 8 stress scenarios** the revised asset requirement.
3. **BEL = the highest asset requirement across ALL scenarios** (base + 8 stresses).

### 10.3 Biting Scenario & Fungibility (Para 28(11))
- For **fungible** liability sets: may apply the scenario producing the highest aggregate asset requirement.
- **Different blocks cannot be assumed fungible** unless demonstrated and **prior written approval** obtained from BMA.
- Where fungibility is restricted, blocks must NOT be aggregated for biting scenario determination.

### 10.4 No Management Actions (Para 22(4))
Projections shall not include discretionary management actions for reinvestment/disinvestment. SBA has its own prescribed approach under Paras 28(33)–(34).

---

## 11. Asset Categories & Requirements

### 11.1 Asset Tiers (Para 28(12)–(16))

| Tier | Category | Examples | Approval | Ref |
|------|----------|----------|----------|-----|
| **1 — Acceptable** | Investment-grade fixed income | Govt bonds, munis, public corporates, cash | None required | Para 28(13) |
| **2 — Prior Approval** | Other IG fixed income | Private assets, structured securities (MBS/ABS/CLO), mortgage loans, IG preferred stock | BMA pre-approval | Para 28(14) |
| **3 — Limited Basis** | Below-IG and alternatives | Sub-IG bonds, commercial real estate, credit funds | BMA approval + caps | Para 28(15)–(16) |
| **4 — LTIC** | Long-term investment credit | Equities and other non-acceptable assets for >30yr liabilities | BMA approval | Para 28(17)–(19) |

### 11.2 Limited Basis Assets — Constraints (Para 28(16))
- **(a)** Aggregate cap: **10%** of SBA portfolio value at calculation time and at each projection time step.
- **(b)** BMA may impose a **lower limit** if deemed appropriate.
- **(c)** Annual review by **approved actuary** required to demonstrate continued appropriateness.
- **(d)** Single-asset concentration cap: **0.5%** of total portfolio.
- **(e)** Subject to other conditions as BMA determines.
- **(f)** **Cannot be sold** to meet cash flow shortfalls (exception: no-maturity assets if BMA approves alternative treatment with haircuts).

### 11.3 Approval of Assets — Handbook Detail (E6.1–E6.8)
- E6.1: Applications for different asset types can be combined into one submission using BMA template.
- **E6.2: Delinquent, non-performing, troubled/challenged assets are NOT eligible** for SBA. If status is uncertain, exclude by default.
- E6.3: Amended/extended/restructured assets must be separately identified with justification.
- E6.4: IG assets with yields above limited-basis caps require assessment; BMA may apply same caps.
- E6.5: Assets must align with Board-approved risk appetite and prudent person principle.
- **E6.6: 10% limit is a conservative cap, not a target**; headroom expected to absorb downgrades.
- **E6.7: Approved Actuary** must assess and form independent opinion on asset application.
- E6.8: LTIC assets should have no contractual maturity or maturity commensurate with liability tenor.

### 11.4 Structured Assets (E7.1–E7.4i)
Applications for other IG assets (per Para 28(14)) must include: descriptions of liabilities, investment/ALM strategy, portfolio summary, investment manager description, quantitative/qualitative analysis of key features and risks (8 sub-items including valuation uncertainty, cashflow predictability, liquidity, credit assessment, diversification, spread analysis, stress testing), D&D cost compliance, asset details on BMA template, affiliated asset assessment.

### 11.5 Asset Optionality (Para 28(20)–(21))
Assets must provide predictable and stable cash flows with no or limited optionality. All optionality and rate-sensitive assumptions must be modelled under all 9 scenarios. Where insufficient data exists, prudent assumptions required.

### 11.6 Long-Term Investment Credit (Para 28(17)–(19))
- Capital adjustment = BEL(without not-acceptable assets) − BEL(with not-acceptable assets beyond 30yr).
- Annual cohort conversion: each year, not-acceptable assets converted to acceptable assets to cover liability cash flows 30 years beyond that year.
- Yields on not-acceptable assets reduced by one standard deviation of cumulative return (excluding interest rate/default risk).
- Non-IG structured securities excluded.
- Application must include: liability overview, projected cash flows, compliance demonstration, asset portfolio detail with rotation uncertainty, variability analysis, stability/predictability of liability cash flows, yield robustness analysis.

---

## 12. Reinvestment Strategy (Para 28(33))

When reinvesting excess net cash flows, insurance groups shall meet these 9 requirements:

1. **(a)** Asset purchases from classes in line with **current asset allocation**, consistent with ALM policy, investment policy, and Board-approved targets.
2. **(b)** No simplification to bucket different alternative assets into one — unless demonstrated quantitatively and qualitatively that this is **more prudent**.
3. **(c)** Reinvestment assumptions must **vary by rating and tenor** within each asset class, at appropriate granularity.
4. **(d)** Tenor may be simplified into **not less than 3 buckets** (short/medium/long-term) defined by the group's cashflow profile.
5. **(e)** Asset purchase assumptions must satisfy five conditions:
   - (i) Prices in line with projected market values per scenario/timestep/rating/tenor.
   - (ii) No material departure from current asset allocation.
   - (iii) Long-term historical market averages applied prudently in context of existing portfolio performance.
   - (iv) **Grade-in period asymmetry**: longer when short-term spreads < long-term; shorter when short-term > long-term. Departures must be demonstrated immaterial.
   - (v) **No overperformance assumption**: current portfolio outperformance vs. market shall not continue over projection period. When projected long-term spreads > current, suitability and materiality of reinvestment risk must be assessed.
6. **(f)** May only purchase from **already-approved** asset categories.
7. **(g)** Must demonstrate (via attestations and approved actuary review) that reinvestment strategy is **more prudent** than using existing asset allocation.
8. **(h)** Maintain **high degree of ALM matching**; where compliance cannot be clearly demonstrated, more prudent approach required.
9. **(i)** Material changes to reinvestment strategies require **BMA written approval**.

---

## 13. Disinvestment Strategy (Para 28(34))

Clearly defined disinvestment strategy aligned with investment and ALM policies:

1. **(a)** Assets sold **only** to meet excess liability cash flows not covered by maturities/coupons.
2. **(b)** Sell to **maintain existing asset allocation** within duration limits over time.
3. **(c)** **Cumulative default and downgrade costs** reflected in sale proceeds within projections.
4. **(d)** **Negative net cash flows cannot be rolled forward** in model projections.
5. **(e)** No borrowing.
6. **(f)** Proportionate to risk profile; aligned with practices; avoid inappropriate simplifications; comply with fungibility constraints.
7. **(g)** **Unsellable assets** cannot be sold. These include:
   - (i) All assets **not publicly traded** (unless BMA-approved as sellable with haircuts).
   - (ii) All assets requiring BMA approval under Para 28(14) and (15) (unless BMA-approved as sellable with haircuts).
   - (iii) All **encumbered assets** (unless sold for their encumbrance purpose with proper consent).

**Simplifications** to disinvestment permitted if assessed quantitatively and qualitatively to be prudent by CIO and approved actuary. *(Para 28(35))*

**CIO attestation** required on appropriateness and prudence of reinvestment/disinvestment strategies, investment expense and spread assumptions, alignment with practices and policies. *(Para 28(36))*

---

## 14. Asset Assignment to Liability Blocks (Para 28(37))

12 requirements across sub-paragraphs (a)–(f):

- **(a)** Assets separately identifiable and documented; not pledged for other purposes.
- **(b)** Adequate controls ensuring assets only exposed to assigned liabilities.
- **(c)** Assigned assets not used for losses from other group activities.
- **(d)** Different assignment approaches permitted; must align with ALM programme.
- **(e)** All approaches must ensure high degree of matching while reflecting legal, regulatory, and operational constraints. Approved actuary assesses constraint treatment.
- **(f)** **Fungibility of asset cash flows between blocks is not allowed** except where: transparent, practical, legally/contractually permitted, documented, tested, governed, reviewed, and limited to legal-entity level. Must also test fungibility under unexpected/severe scenarios (counterparty default, market dislocation). Separate collateral accounts → no fungibility without BMA written approval. **No fungibility between legal entities.**

---

## 15. Default and Downgrade Costs

### 15.1 Rules Requirements (Para 28(22)–(29))
- **(22)** Projected asset cash flows **reduced** by default and downgrade costs. BMA prescribes costs for some publicly-traded assets; methodology for others per Paras (24)–(25).
- **(23)** Applied by **reducing projected asset cash flows** (not initial market values).
- **(24)** For non-prescribed assets: (a) realised average default losses as baseline; (b) uncertainty margin as downgrade cost.
- **(25)** BMA **written approval** required before using non-prescribed assets.
- **(26)** Non-prescribed asset D&D costs must be: (a) no less than BMA-published for comparable credit quality; (b) no less prudent than approach in (24); (c) benchmarked where applicable; (d) for limited-basis assets: designated insurer applies to BMA, uncertainty adjustment ≥ 1 standard deviation of default costs, BMA may vary.
- **(27)** May be required to demonstrate net investment spread can be earned over asset tenor; assess illiquidity/complexity premia earnability.
- **(28)** BMA may vary criteria where internal model or IRB approach approved.
- **(29)** **Chief Actuary and CIO** must confirm D&D assumptions and regulatory compliance.

### 15.2 Handbook Detail (E10.1–E10.19b)

**Application mechanics:**
- **E10.2**: Simplified spreadsheet example on BMA website illustrates core mechanics.
- **E10.3**: Annualised costs converted to **cumulative loss rates** (and marginal loss rates) for application to asset cash flows. Same loss rate for all cash flows within a given period.
- **E10.4**: Full cumulative impact reflected both **at maturity and on sale**.
- **E10.5**: D&D costs do **NOT affect initial t=0 market values** or reinvestment purchase prices. Only actual projected cash flows impacted (including sale proceeds).
- **E10.6**: **Marginal loss rate must be non-negative** (cumulative non-decreasing). If otherwise, floor marginal at zero and adjust all subsequent cumulative rates accordingly.
- **E10.7**: If implied spread turns negative, D&D costs may be adjusted so implied spread = lower of zero and actual market spread. Post-adjustment D&D costs cannot be < 0.

**5-year transitional for downgrade costs (E10.8):**
For business in force as at 31 Dec 2023, the **downgrade cost** component phases in:

| Valuation Year | Phase-in % |
|:--------------:|:----------:|
| 2024 | 20% |
| 2025 | 40% |
| 2026 | 60% |
| 2027 | 80% |
| 2028+ | 100% |

- **E10.9**: After phase-in, round to nearest whole basis point per asset type/rating/tenor combination.
- **E10.10**: Early adoption allowed but **irreversible** without BMA approval.
- **E10.11**: Phase-in does **NOT** apply to default cost (expected loss) — applies in full immediately. Business written after 31 Dec 2023: full costs immediately (no phase-in).

**Issuer vs. issue ratings (E10.12–E10.14):**
- Costs calibrated on issuer defaults and issue-level LGDs.
- **Issuer ratings should be used.** Issue-level ratings permitted only if demonstrated no less conservative or differences are immaterial.

**Simplifications (E10.15):** Allowed if prudent and capture cumulative impact at all points including on sale.

**Floors (E10.16):** For assets without BMA-published D&D costs, floors = **corporate bond (senior unsecured)** D&D costs for corresponding rating. Structured assets/securitisations: floors apply at **tranche level**.

**Beyond published tenors (E10.17–E10.18):** Costs kept constant at last published tenor values (implies increasing cumulative losses). Alternative approaches allowed if demonstrated no less conservative.

**Government debt (E10.19):**
- **AA- or better**: No D&D costs for government debt in **local currency**. *(E10.19a)*
- **A- or better** with: local currency denomination, global reserve currency status (fully convertible), full independent fiscal/monetary policy → No D&D costs. *(E10.19b)*
- All other government debt treated as unsecured corporate of same rating.

---

## 16. Transaction Costs

### 16.1 Rules Requirements (Para 28(30)–(32))
Five requirements:
- **(a)** Best estimate transaction costs on all assets bought/sold.
- **(b)** Where historical costs not representative of expected future costs, adjust upward.
- **(c)** Where insufficient data or uncertainty, use **prudent assumptions**.
- **(d)** Include all applicable **fees, commissions, and expenses**.
- **(e)** Transaction cost assumptions **independently reviewed by approved actuary**.

Full expected price impact of selling/buying reflected. *(Para 28(31))*

Regular **back-testing** of bid-ask spreads and liquidity impacts against historical data and own experience. BMA may request test results. *(Para 28(32))*

### 16.2 Handbook Detail (E11.1–E11.7b)
- **E11.2**: Full expected price impact including implicit and explicit fees/commissions/expenses.
- **E11.3 (Grade-in)**: If current bid-ask < long-term average → grade in from current to long-term. If current > long-term → grade-in period must be **more prudent** (shorter). Applies to existing assets and reinvestments.
- **E11.4 (Effective bid-ask)**: Must reflect position size and volumes vs. market depth/liquidity. **Marginal bid-ask (incremental unit) not acceptable.** May use quantity-varying bid-ask based on observed data.
- **E11.5 (Liquid publicly traded)**: Minimum = bid-ask spreads, if demonstrated to capture full price impact.
- **E11.6 (Illiquid/other assets)**: Bid-ask may not fully reflect price impact; assumptions must reflect degree of illiquidity. Impact expected higher than advertised/broker quotes.
- **E11.7a**: Transaction costs must be **no lower than** implied bid-ask/discounts from **past actual trades**.
- **E11.7b**: Illiquid asset price impacts **no less than** similar liquid assets of equivalent credit quality.

---

## 17. Eligibility and Lapse Risk (Para 29)

An insurance group must satisfy **one of two conditions**:

### Condition 1 (Para 29(1))
Contracts include **no policyholder options** and cash flows are **well-matched** with assets.

### Condition 2 (Para 29(2))
Where policyholder options exist, residual risk demonstrated **insignificant** through modelling, ALM, stress testing, and liquidity risk management. Must meet ALL of:

- **(i) Lapse Cost (LapC)** held within SBA BEL:
  - A. Calculate difference between actual and expected lapse rates as % of expected.
  - B. Calculate 1 standard deviation of differences → lapse rate sigma (rounded up to nearest 1%).
  - C. Calculate lapse up/down capital requirement using BSCR shock per Para 44A of Schedule I.
  - **D. LapC = (Lapse Rate Sigma ÷ BSCR lapse shock) × Lapse up/down capital requirement.**
- **(ii)** Pass **100% ECR** under BSCR lapse up and lapse down stresses (permanent change).
- **(iii)** Pass liquidity stress test with **minimum 105% LCR** = Eligible Liquidity Sources ÷ Liability Outflows.
- **(iv)** Eligible sources and shocks prescribed by BMA.
- **(v)** Demonstrate via GSSA: robust ALM/capital/liquidity management, diligent lapse risk management, insurer-specific stress testing.
- **(vi)** Provide lapse/liquidity/SBA Return reporting per Para 32.

### Lapse Risk — SBA-Specific Provisions (Handbook D35)
- **D35.8–D35.16**: SBA users may elect **modified discounting** for mass lapse stress (subject to BMA written approval):
  - **Option A** (D35.11): Implied SBA yield = yield equating PV of worst-scenario liability cash flows to SBA BEL. Post-stress BEL = PV of stressed cash flows at this yield.
  - **Option B** (D35.13): Use implied book yield of assets required in worst scenario; apply to stressed cash flows.
  - Must be applied consistently; no switching without BMA approval.
- **D35.23, D35.26, D35.33**: SBA users may apply **lapse up, lapse down, and mass lapse stresses under SBA base scenario** (documented and disclosed).

---

## 18. Application & Approval Process

### 18.1 Rules Requirements (Para 30)
- **(i)** Groups registered on or after 1 Jan 2024 proposing SBA → require BMA approval.
- **(ii)** Groups registered before 1 Jan 2024 not currently using SBA → require approval.
- **(iii)** Groups already using SBA → require prior approval for **major** model changes.

### 18.2 Application Package (Handbook E5.1–E5.11)
Must include:
- **E5.1**: Evidence eligibility requirements met per sub-portfolio (fungible sub-portfolios assessed as one).
- **E5.2**: Board/committee sign-off.
- **E5.3**: Completed Lapse, Liquidity and SBA reporting template.
- **E5.4**: Full SBA model calculations (asset/liability models and/or cash flows).
- **E5.5**: Well-matched portfolio assessment.
- **E5.6**: SBA methodology documentation:
  - (a) Detailed methodology description, liability/asset valuation, cashflow projections.
  - (b) Data description and quality compliance.
  - (c) Key assumptions and expert judgements.
  - (d) Limitations and weaknesses.
  - (e) Governance structure for reserving and SBA.
  - (f) Fungibility assessment.
  - (g) Validation reports.
  - **(h)** Stress testing results (see Section 19 below).
  - **(i)** Policies: Model Risk Management, Data Quality, Model Change, Model Validation, ALM, Investment Policy & Guidelines, Liquidity Risk Management.
  - (j) Derivatives application requirements.
  - (k) Assets-requiring-approval application requirements.
  - **(l) Systems, infrastructure, and people resources** overview (may be graphical).
- **E5.7**: External dependency assessment (vendors/consultants) and outsourcing compliance.
- **E5.8**: Model Risk Management Framework logs.
- **E5.9**: Any other relevant information.
- **E5.10–E5.11**: Pre-application engagement encouraged. Decision expected **4–8 weeks** for complete, high-quality submissions.

---

## 19. Application Stress Tests (Handbook E5.6h)

### 19.1 Combined Credit Spread + Mass Lapse Stress (E5.6h(i))
- **Mass lapse shock**: instantaneous, higher of 20% (all products) or product-specific BSCR mass lapse shock.
- **Credit spread widening** (instantaneous, Year 1 only, revert to base after Year 1):

| Rating | AAA | AA | A | BBB | BB | B | CCC/C |
|--------|:---:|:--:|:-:|:---:|:--:|:-:|:-----:|
| Δ bps  | 277 | 328| 444| 498 | 842|1346| 2346 |

- **All assets** stressed (TP-backing + capital), including structured/ABS/MBS, rated and unrated (unrated assumed below CCC/C).

### 19.2 One-Notch Credit Downgrade Stress (E5.6h(ii))
- Apply one-notch downgrade to **all assets** (e.g., BBB → BBB-).
- Use post-shock assets for TP calculation; spreads and D&D costs respond accordingly.
- **No change to spreads themselves** for any rating — but downgraded assets re-priced on lower credit curve (lower MV).
- If insufficient SBA-eligible assets post-downgrade → use Standard Approach for remainder (no splitting at contract/block-product level; if not achievable → Standard Approach for all).
- Investment-grade per BMA ratings definition using lowest rating.
- Identify portion of liabilities using Standard Approach.

### 19.3 No Reinvestment into Limited-Basis Assets Stress (E5.6h(iii))
- Assume insurance group cannot continue reinvesting in assets acceptable on a limited basis.
- Update modelled reinvestment strategy accordingly.

---

## 20. Derivatives in SBA (Handbook E8)

Derivatives must qualify as **risk-mitigating** per Handbook Section B5. **Dynamic hedging (daily/intra-day) is not allowed in SBA.** *(E8 intro)*

Application must cover 9 areas (E8.1a–i):
1. **Investment strategy summary** — why derivatives needed.
2. **Hedging strategy & risk management** — program description, buy-and-hold vs. trading, ISDA/CSA details, effectiveness metrics/history, basis risk measurement, operational risk incidents (3yr).
3. **Derivatives list** — exhaustive, by type and currency. Currency-specific approval.
4. **Modelling & assumptions** — how modelled in SBA, impact analysis (BEL with/without derivatives, base + 8 scenarios), key assumption justification, hedging effectiveness/frictional costs in projections.
5. **Residual risks** — quantified to **1 standard deviation**. Attention to asymmetric risks. Must describe how captured by prudence in BEL. Common omission: credit spread evolution risk when using interest rate swaps.
6. **Liquidity & collateral** — management approach, sufficient in all SBA scenarios including residual risk scenarios.
7. **Stress & scenario testing** — methodology, frequency, liquidity stress, volatile rate environment impacts.
8. **Governance & oversight** — risk framework, external manager agreements, modelling governance, internal approvals, second-line reviews, management reports.
9. **Worked example** — preferably Excel, representative of all use cases.

---

## 21. Well-Matched Portfolio Assessment (Handbook E4.1–E4.3i)

Formal assessment documented, including definition of "well-matched" and thresholds/triggers for when the group would consider it is NOT well-matched. *(E4.1)*

### BMA's 9 Assessment Criteria (E4.3):
| Criterion | Description |
|-----------|-------------|
| E4.3a | **Cost of mismatch**: difference between base and biting scenario results. |
| E4.3b | **Dispersion** in results from the 8 stress scenarios. |
| E4.3c | Capital required for each of currency, interest rate, lapse, mortality, morbidity, longevity as **% of BEL**. |
| E4.3d | Currency denomination of backing assets; hedging extent/nature/effectiveness; residual risks. |
| E4.3e | **Extent of asset sales** to meet shortfalls, as % of BEL. |
| E4.3f | **Reinvestment analysis**: graphical representation of annual liability CFs, existing asset CFs, reinvestment asset CFs; sensitivity tests on reinvestment assumption impact. |
| E4.3g | **Highest accumulated cashflow shortfall** across all projection years, as % of BEL. |
| E4.3h | **ALM position vs. internal tolerances** at different key rate points. |
| E4.3i | Extent to which assets are **fungible or encumbered**. |

---

## 22. Governance and Internal Controls (Para 28(38)–(40))

### 22.1 Model Documentation (Para 28(38))
12 requirements including: (a) allow knowledgeable third party to understand; (b) materiality of assumptions confirmed; (c) limitations identified; (d) regulatory compliance confirmed; (e) detailed description of structure, scope, theory, data, assumptions, expert judgment, parameterisation, results, validation, changes, governance, policies; (f) all key software/external models/data and reasons; (g) documentation standard with roles, sign-off, review processes; (h) inventory of all model documents; (i) limitations/simplifications/weaknesses and conditions of inadequacy; (j) all model risk management activities documented; (k) upstream/downstream model interactions; (l) documentation detail proportionate to materiality.

### 22.2 Data Policy (Para 28(39))
6 sections: (a) internal processes for completeness, accuracy, appropriateness; (b) completeness = sufficient history + available per risk group; (c) accuracy = free from material errors, consistent over time, timely, modifications documented; (d) appropriateness = consistent with purpose, no material estimation error, consistent with assumptions; (e) external data: demonstrate suitability vs. internal, confirm origin/assumptions, identify trends, demonstrate portfolio relevance; (f) document data controls and justify adequacy.

### 22.3 Governance (Para 28(40))
10 requirements:
- **(a)** Board approves initial SBA use and major changes; model change policy defines "major."
- **(b)** Board responsible for ongoing appropriateness of SBA model.
- **(c)** Appropriate committees for challenge, approval, reporting; validation reports discussed.
- **(d)** Model risk management guidance within overall risk framework; minimum: MRM policy, model change policy, data quality policy.
- **(e)** Classification of changes as minor/major. Scope expansion classified as minor still requires BMA **no objection**. Major changes require BMA **prior written approval**.
- **(f)** Control function roles clearly defined.
- **(g)** Conflicts of interest identified and addressed; proportionality considered; remaining gaps assessed by approved actuary.
- **(h)** Adequate systems, infrastructure, and resources.
- **(i)** Adequate and effective controls for model operation/maintenance.
- **(j)** Third-party software allowed; **outsourcing of running/maintaining/managing the SBA model is not allowed**. Existing outsourcing requires oversight demonstration and is subject to BMA onsite review.

---

## 23. Model Risk Management (Para 28(41))

Comprehensive policy required, including:

- **(a) Materiality definition** specific to SBA, developed with control functions. Determines whether changes/findings are material → classified minor/major.
- **(b)** First/second-line roles clearly defined with independence for validation.
- **(c)** CRO and CEO attestation on adequacy of MRM practices. Comprehensive model inventory maintained.
- **(d) Development, testing, implementation**: structured and aligned with regulation; rigorous quality/change control for software/code; lifecycle testing for accuracy, stability, robustness; documented test plans; user feedback collection processes.
- **(e) Limitations and uncertainties**: quantified where possible (ranges, not false precision); qualitative assessment where quantification not possible; reported in model risk reporting with determination on BEL adjustments.
- **(f) Pre/post-model adjustments**: circumstances defined; review, approval, continued use, removal, back-testing processes; monitoring to address underlying issues; all documented and reported with/without adjustments.
- **(g) Model validation**: independent validation (external or internal, separate from developers/users); validated before regulatory reporting use and at **minimum 3-year intervals** thereafter; comprehensive validation process covering all components, including vendor models; feeder and downstream models included; validation techniques include: conceptual soundness, sensitivity/stress testing, back-testing, movement attribution, independent replications, process verification, benchmarking; detailed validation reports produced.
- **(h) Model review and monitoring**: periodic review (no less than annually); model performance monitored with key metrics; first-line actuarial work qualifies as review if covering technical aspects. Model review log maintained (approved actuary annual review).
- **(i) Model risk reporting**: captured promptly, reported regularly to management committee; reports must include reliability/adequacy analysis with sensitivity analysis; tolerance levels set and reviewed, with ~10 reporting items (high-risk models, exemptions, issues status, performance metrics, risk events, uncertainty quantification, development efforts, resource indicators, key outputs).
- **(j) Third-party/vendor models**: included in MRM framework; must obtain developmental evidence, data information, testing results, documentation of limitations/assumptions, ongoing performance monitoring. Validate vendor products; benchmark where source code unavailable; contingency plans for vendor discontinuation.
- **(k) Internal audit**: reviews all SBA MRM; assesses challenge effectiveness; assures validation/review adequacy and frequency; forms independent opinion on first/second-line MRM activities.

---

## 24. Attestations (Handbook E2)

### Required Officers
Chief Risk Officer, Chief Investment Officer, Chief Actuary, Chief Executive Officer. Substitution allowed with BMA written no-objection. *(E2.4)*

### Key Principles (E2.6a–E2.6m)
13 principles including: clear/concise language, evidence-based, demonstrate work/challenge as natural consequence of business processes, identify specific regulatory requirements, clearly outline roles/responsibilities, specify accountable individuals, highlight third-party assessments, internal controls effectiveness, avoid duplication (reference other submissions), provide supporting documentation list, identify attesting officer(s) by name/title, signed and filed annually.

### Additional Requirements
- BMA retains right to request additional information. *(E2.2)*
- Log of information used should be maintained. *(E2.3)*
- **Attestation does not replace the Approved Actuary's own review and opinion.** *(E2.5)*

---

## 25. Model Change Policy (Handbook E3)

- **E3.1**: Describe clear universe of changes, models covered, governance for approving major changes and borderline cases.
- **E3.2**: Materiality criteria — both quantitative (own funds, solvency ratio impact) and qualitative (governance, systems). Back-testing metrics against historical changes recommended. Log of historical and potential changes maintained.
- **E3.3**: Engage with BMA when formulating the policy to ensure alignment.

---

## 26. Liquidity Risk Management (Para 31)

13 minimum requirements for insurance groups using SBA (or standard approach with lapse/liquidity exposure):

(i) Clear ownership of key elements. (ii) Annual review or more frequently. (iii) First/second-line roles defined with conflict mitigation. (iv) Sources and dynamics of cash supply/demand understood. (v) Risk appetite informed by stress testing, Board-approved. (vi) Integrated into wider risk framework and day-to-day operations. (vii) Liquidity metrics and target thresholds defined. (viii) Cash needs and sources register maintained. (ix) **Liquidity buffer** — pool of highly liquid assets set aside. (x) Stressed scenarios of varying severity including fast-moving and sustained; reverse stress tests. (xi) **Contingency plan** with clear triggers, regularly tested via dry-run simulations. (xii) Encumbered sources identified and reflected in stress testing. (xiii) Forward-looking liquidity reporting with early warning indicators.

---

## 27. Affiliate/Related/Connected Party Assets (Para 33)

**Prior written BMA approval** required for all assets with counterparty exposure to affiliates, related parties, or connected parties — applies to both SBA and Standard Approach.

---

## 28. Risk Margin (Para 36)

**RM = CoC × Σ_{t≥0} [ModECR_t / (1 + r_{t+1})^{t+1}]**

Where:
- **CoC** = Cost-of-Capital rate prescribed by BMA (currently 6%).
- **ModECR_t** = projected ECR at time t for insurance, credit, operational, and material non-hedgeable market risks only. Calculated at t=0 and annually until all obligations settled.
- **r_t** = risk-free discount rate (BMA-prescribed, without illiquidity adjustment) for maturity t.

Calculated net of reinsurance, at aggregate level, separately for general/long-term business. Run-off period covers full liability lifetime. *(Para 36(1)–(7))*

### LT_TransitionalFactor (Handbook D19.6)
Factor increases from **10% in 2024** in equal 10 percentage point increments to **100% in 2033**. For Risk Margin projection, SBA users may keep this factor fixed at its valuation-date value. *(D35.46)*

---

## 29. BSCR Integration — SBA Offset (Handbook D22.10)

SBA users receive an offset to the interest rate and liquidity risk capital charge:

**C_Interest = max{ max(Shock_IR,Down, Shock_IR,Up) − Offset_ScenarioBased, 0 }**

Where:

**Offset_ScenarioBased = min(0.5 × (BEL_WorstScenario − BEL_BaseScenario), 0.75 × C_Interest_WithoutOffset)**

This recognises that interest rate risk is already partially captured within the SBA BEL calculation.

---

## 30. Technical Provisions as a Whole (Para 37)

Where future cash flows can be **reliably replicated** using financial instruments with observable market values:
- TP = market value of replicating instruments (no separate BEL + RM required).
- Replication must match uncertainty in amount, timing, and all risks in all scenarios.
- May **unbundle** policies: some parts valued as-a-whole, others via BEL calculation.
- **Non-replicable cash flows** (per Para 37(5)): policyholder option-dependent, mortality/morbidity-dependent, and all servicing expenses.
- Replicating instruments must be traded in **active, deep, liquid, transparent** markets.

---

## 31. Lapse/Liquidity/SBA Reporting (Para 32)

Annual completion of BMA-prescribed reporting template required for: all SBA users, and non-SBA users with lapse/liquidity exposure.

---

## 32. Transitional Provisions (Instructions Affecting Schedule XXV)

**Applies only to**: long-term business written on or before **31 December 2015** for which the **Standard Approach** has been applied. **Does NOT apply to SBA business.**

16-year phase-in via linear interpolation:

**TP_y = [Min(y−2015, 1)/16] × EBS_TP_y + [1 − Min(y−2015, 1)/16] × CurrentRes_y**

Where y ∈ {2016, 2017, ..., 2031}, EBS_TP_y = TP under current Rules, CurrentRes_y = reserves under 2015 methodology.

Risk margin allocation: where unable to directly attribute RM to pre/post-2015 business, allocate via documented process. Application under Section 6D(7) may be made. *(Para (4))*

---

## Summary Checklist for Implementation

Based on this guide, an SBA benchmark model must handle:

1. **Yield curve construction** — forward rate extraction, future spot curve building, 9 scenario adjustments (Para 28(8)).
2. **Idiosyncratic spread calibration** — asset-level spread adjustment preserving initial market values (E9.5).
3. **Multi-scenario projection** — 9 prescribed paths with at-least-annual cash flow matching.
4. **Dynamic rebalancing** — disinvestment waterfall (with D&D-adjusted proceeds, unsellable constraints) and reinvestment (3+ tenor buckets, rating-specific, grade-in asymmetry).
5. **Default and downgrade costs** — cumulative/marginal conversion, non-negative marginal floor, issuer ratings, government debt treatment, phase-in for pre-2024 business.
6. **Transaction costs** — effective bid-ask (not marginal), liquid vs. illiquid, grade-in, back-testing.
7. **Asset tier enforcement** — 10% limited-basis cap (+ 0.5% single-asset), unsellable classification, LTIC mechanics.
8. **BEL determination** — max asset requirement across all 9 scenarios; biting scenario per fungible block.
9. **Lapse cost** — LapC formula within BEL; modified discounting options for mass lapse stress.
10. **Risk Margin** — CoC × discounted ModECR; LT_TransitionalFactor handling.
11. **BSCR offset** — interest rate risk charge reduction formula (D22.10).
12. **Stress testing** — combined spread+lapse, one-notch downgrade, no-reinvestment-into-limited stresses.
13. **Audit trail** — rule traceability, biting scenario selection logging, immutable run configuration.
