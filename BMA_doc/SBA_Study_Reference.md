# Bermuda Scenario-Based Approach (SBA): A Comprehensive Study Reference

> Based on J.P. Morgan Asset Management: *"Untangling Complexity — A Step-by-Step Guide to Crafting Optimal Portfolios under the Bermuda Scenario Based Approach"* and supporting mathematical formulations.

> **Important disclaimer (added 2026-04-08):** This document reflects **J.P. Morgan's asset management perspective** on the SBA, not the BMA's regulatory requirements. It was created from a marketing paper aimed at institutional investors building SBA portfolios. Key differences from the official BMA Rules and Handbook include:
>
> - The "discounting liabilities at RFR + illiquidity premium" framing is a conceptual simplification. The BMA Rules (Para 28(10)) define BEL as the **highest asset requirement across 9 scenarios** — an asset-side projection exercise, not a liability-discounting exercise. The illiquidity premium is a derived quantity, not an input.
> - The theta/lambda scaling and portfolio optimisation framework (Sections 7–10) is **J.P. Morgan's proprietary methodology** for portfolio construction, not the regulatory BEL calculation. The BMA Rules simply require determining the assets needed to cover liabilities under each scenario.
> - Several regulatory details are missing or simplified (yield curve construction methodology, idiosyncratic spread adjustment, default/downgrade cost mechanics, governance requirements, BSCR SBA offset). For implementation, refer to `BMA_SBA_Consolidated_Guide.md` (v2.0).
>
> Corrections to specific inaccuracies have been applied inline and marked with **[Corrected]** tags.

---

## Table of Contents

1. [The Big Picture — Why the SBA Exists](#1-the-big-picture)
2. [Overview of the SBA Framework](#2-overview-of-the-sba-framework)
3. [Illiquidity Premium — Concept and Measurement](#3-illiquidity-premium)
4. [Key Technical Distinction — IRR vs Z-Spread](#4-irr-vs-z-spread)
5. [Asset Admissibility — The 258 Categories](#5-asset-admissibility)
6. [SBA Model Implementation](#6-sba-model-implementation)
7. [BEL Calculation — Mathematical Formulation](#7-bel-calculation)
8. [Theta Scaling — Goal Seek Logic](#8-theta-scaling)
9. [Liability Scaling — SBA vs Standard Carve-Out](#9-liability-scaling)
10. [Portfolio Optimisation](#10-portfolio-optimisation)
11. [BSCR, MCR and the Standard Approach](#11-bscr-mcr-and-standard-approach)
12. [Bermuda vs UK Matching Adjustment](#12-bermuda-vs-uk-matching-adjustment)
13. [The SBA Business Model — End-to-End](#13-the-sba-business-model)
14. [Duration, KRD and DV01 — The Rate Sensitivity Toolkit](#14-duration-krd-and-dv01)

---

## 1. The Big Picture

### Why Does the SBA Exist?

Life insurers holding long-dated, predictable liabilities (e.g. pension annuities) can invest in illiquid fixed income assets and hold them to maturity. By doing so, they earn a yield premium over the risk-free rate — the **illiquidity premium** — without being exposed to mark-to-market losses, because they never need to sell.

The SBA is the regulatory framework that allows Bermuda insurers to **recognise the illiquidity premium in the liability valuation**. **[Corrected — note on framing]** While this is often described as "discounting liabilities at a higher rate," the BMA Rules actually define BEL as the highest asset requirement across 9 interest rate scenarios (Para 28(10)) — an asset-side projection exercise. The illiquidity premium is the residual yield advantage that emerges from the asset portfolio after default adjustments, not a prescribed discount rate input. The economic effect is similar (lower BEL, improved solvency position), but the regulatory mechanism is fundamentally different from simple liability discounting.

### The Core Value Proposition

```
Higher yielding illiquid credit portfolio
            ↓
Default-adjusted asset CFs match liability CFs
            ↓
Market Value of portfolio < Risk-free discounted BEL
            ↓
SBA BEL is LOWER than Standard BEL
            ↓
Lower liabilities → Higher surplus → More capital efficiency
```

### Regulatory Balance Sheet

```
ASSETS                              LIABILITIES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SBA Ringfenced Portfolio            SBA BEL
(258C + 258E + 258F assets)        (discounted at RFR + Illiq. Prem)
Valued at Market Value
                                    Standard BEL
Other Assets                        (discounted at RFR only)
(Non-SBA portfolio)
                                    BSCR Capital Requirement

                                    Surplus (Free Capital)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Numerical Illustration of SBA Benefit

| Approach | BEL | Required Assets | Surplus |
|---|---|---|---|
| Standard (RFR only) | £950m | £950m | £50m |
| SBA (RFR + 179bps) | £800m | £800m | £200m |
| **SBA Advantage** | **£150m lower** | **£150m less needed** | **£150m more** |

---

## 2. Overview of the SBA Framework

### 2.1 Purpose

The SBA enables insurers to determine BEL through asset-liability cash flow projection under 9 interest rate scenarios, which in practice results in a lower BEL than the Standard Approach due to the illiquidity premium embedded in the asset portfolio. **[Corrected — see disclaimer at top for framing note]**

The framework rests on three key conditions:

1. The liabilities must have **predictable and stable cash flows**
2. The assets must be managed on a **buy-and-maintain basis** and **ringfenced**
3. The asset cash flows, **after adjusting for defaults and downgrades**, must match the liability cash flows

### 2.2 Regulatory Oversight

- Insurers must **apply for BMA approval** to use the SBA (introduced in 2024 amendments)
- Existing SBA users may continue but require approval for any **major model changes**
- The BMA review process takes approximately **4–8 weeks**
- Approval requires: full model documentation, reporting templates, liquidity management evidence, stress tests, and governance documentation

### 2.3 Liability Eligibility

Liabilities are eligible for SBA if they either:
- Contain **no policyholder options** (illiquid liabilities), or
- Where options exist, the **residual risk is demonstrated to be immaterial** through adequate modelling

Where policyholder options exist, three additional tests must be passed:

| Test | Description |
|---|---|
| Lapse Cost (LapC) | Added to SBA BEL: `(1 SD lapse rate / BSCR lapse stress) × lapse capital requirement` |
| 100% ECR under lapse stress | Maintain 100% ECR coverage under the **BSCR lapse up/down stresses** as prescribed under Para 44A of Schedule I **[Corrected — the "40%" is not specified in the SBA rules; it's a BSCR parameter that may change]** |
| 3-month liquidity coverage ratio | Maintain 105% liquidity coverage ratio over 3-month horizon |

**Important:** Insurance contracts cannot be split to achieve SBA eligibility — the option must stay attached to the policy.

### 2.4 Liability Eligibility by Business Line

| Business Line | SBA Eligibility | Reason |
|---|---|---|
| Pension risk transfer / annuities in payment | ✅ Generally eligible | Predictable, stable, limited policyholder options |
| Whole life | ✅ Potentially eligible | Depends on option materiality |
| Term assurance | ❌ Generally not eligible | Cash flows driven by random mortality events — not predictable or stable |
| Variable annuities | ❌ Generally not eligible | Policyholder options too complex |

---

## 3. Illiquidity Premium

### 3.1 Conceptual Foundation

Credit spreads on fixed income assets consist of multiple components:

| Component | Description | Risk to Insurer |
|---|---|---|
| Real rates + Inflation | Risk-free rate components | None if funded by matched illiquid liabilities |
| Term premium | Compensation for duration | None if matched |
| Liquidity premium | Compensation for illiquidity | None — harvested as illiquidity premium |
| Expected losses | Compensation for expected defaults | Fully at risk |
| Default volatility | Deviation of actual from expected losses | Fully at risk (capital held against this) |
| Spread volatility | Mark-to-market risk | Only at risk if sold pre-maturity |

The illiquidity premium is the component the insurer can **earn risk-free** by virtue of holding assets to maturity against matched illiquid liabilities.

### 3.2 Methods of Estimating the Illiquidity Premium

Three broad approaches exist in academic literature:

**(i) Direct / Model-Free Method**

Identify two instruments with **identical characteristics except for liquidity**. The liquidity premium is the yield difference between them.

**Key Example — Corporate Bond vs CDS:**

| Instrument | Liquidity | Yield/Spread |
|---|---|---|
| Corporate bond | Less liquid — physical bond, harder to trade | Higher spread |
| CDS (Credit Default Swap) | More liquid — synthetic derivative, standardised | Lower premium |

Both instruments expose the holder to **exactly the same credit risk** of the same issuer. Therefore:

$$\text{Illiquidity Premium} = \text{Corporate Bond Spread} - \text{CDS Premium}$$

This is the cleanest expression of the illiquidity premium — observable directly from market prices with no model assumptions required.

**The Negative Basis Trade:** An investor can simultaneously:
1. **Buy the corporate bond** — earn the full credit spread
2. **Buy CDS protection** — pay away the CDS premium, fully hedging credit risk

Net position: earn the illiquidity premium with no credit risk.

In practice, frictions prevent this being a pure arbitrage: counterparty risk on CDS, funding costs, basis risk between CDS and bond, and the ongoing need to hold the illiquid bond on balance sheet.

**(ii) Structural Models of Default**

Use models such as the Merton model to determine the theoretical credit spread. The excess observed spread corresponds to the liquidity premium (related to the "credit spread puzzle" in academic literature).

**(iii) Reduced-Form Regression Models**

Use liquidity proxies such as bid-ask spread, market depth, and trading intensity to estimate the liquidity premium component via regression analysis.

### 3.3 The SBA Approach to Illiquidity Premium

Under the SBA, the BMA prescribes default and downgrade costs derived from average historical default rates. These are used to haircut asset cash flows. The illiquidity premium can then be **derived** (not defined by the BMA) as:

$$\text{Illiquidity Premium} = \text{IRR}(\text{default-adjusted asset CFs}) - \text{IRR}(\text{liability CFs at risk-free rate})$$

**[Corrected]** Note: This formula is JP Morgan's derived measure. The BMA Rules do not define or use the term "illiquidity premium" in the SBA calculation itself — the BMA simply requires determining the highest asset requirement across 9 scenarios. The illiquidity premium is an analytical quantity useful for understanding the SBA benefit, not a regulatory input.

---

## 4. IRR vs Z-Spread

This is a critical technical distinction that is frequently misunderstood.

### 4.1 Internal Rate of Return (IRR)

| Feature | Detail |
|---|---|
| Cash flows used | **Default-adjusted** cash flows (promised CFs × (1 − default haircut)) |
| Discount rate | A **single flat rate** applied uniformly across all periods |
| Method | Solve for the single rate that equates market value to PV of default-adjusted CFs |
| Credit component | **Net of default costs** — already stripped from the cash flows |

$$\text{MV} = \sum_{t=1}^{T} \frac{\text{Promised CF}_t \times (1 - H_t)}{(1 + \text{IRR})^t}$$

Decomposition:

$$\text{IRR} = \text{RFR} + \text{Net Credit Spread} + \text{Illiquidity Premium}$$

Where "Net Credit Spread" is residual spread **after** defaults have been stripped from cash flows.

### 4.2 Z-Spread (Zero-Volatility Spread)

| Feature | Detail |
|---|---|
| Cash flows used | **Contractual** (unadjusted) cash flows — full promised amounts |
| Discount rate | **Risk-free spot curve at each maturity + constant spread** |
| Method | Goal-seek the constant spread over the full spot curve that equates MV to PV of contractual CFs |
| Credit component | **Gross** — includes expected default costs |

$$\text{MV} = \sum_{t=1}^{T} \frac{\text{Promised CF}_t}{(1 + \text{Spot Rate}_t + \text{Z-Spread})^t}$$

Decomposition:

$$\text{Z-Spread} = \text{Expected Default \& Downgrade Costs} + \text{Illiquidity Premium}$$

### 4.3 Side-by-Side Comparison

| Feature | IRR | Z-Spread |
|---|---|---|
| Cash flows | Default-adjusted | Contractual (full) |
| Discount rate | Single flat rate | Spot curve + constant spread |
| Credit component | Net of defaults | Gross, includes defaults |
| Relationship | IRR ≈ Z-Spread − Expected Default Costs | Z-Spread > IRR |

### 4.4 Direction of Causality in SBA

This is commonly misunderstood:

❌ **Wrong:** RFR + Risk Premium → Discount rate → Compute MV
✅ **Correct:** Market prices → Observe MV → Solve for IRR → Decompose into RFR + spread components

The market value of a bond is **observed directly** from market prices. The IRR is then **derived** from that observed value. The illiquidity premium is then **backed out** as the residual after stripping the risk-free rate and expected default costs.

---

## 5. Asset Admissibility

### 5.1 The 258 Asset Categories

The naming convention (258C, 258D, 258E, 258F) refers to paragraph numbers in the BMA Guidance Notes for Commercial Insurers (November 2016).

| Category | Name | Description | SBA Eligible? |
|---|---|---|---|
| 258C | Generally Acceptable Assets | Government bonds, corporate bonds, preferred stock, CDs, structured securities (with prior approval), commercial mortgage loans (with prior approval) | ✅ Yes — primary category |
| 258D | Not Acceptable Assets | Equities, equity tranches of securitised debt, instruments with ill-defined cash flows | ❌ No |
| 258E | Assets Acceptable on Limited Basis | Up to 10% of portfolio; below-IG assets, commercial real estate, credit funds; requires prior approval; max 0.5% per instrument unless approved | ✅ Yes — limited to 10% |
| 258F | Assets for Liabilities Beyond 30 Years | Otherwise ineligible assets (e.g. equities) for long-dated liabilities; yield reduced by 1 SD; assumed converted to 258C at year 30 | ✅ Special treatment |

### 5.2 Key Asset Admissibility Rules

**Bonds with Optionality (Section 2.5.2)**

Unlike the UK MA regime, callable bonds and other bonds with optionality **are permitted** under the SBA. The requirement is that cash flows must be modelled to differ appropriately across the 9 interest rate scenarios (e.g. a callable bond is assumed called when rates fall below the coupon rate).

**Portfolio Management Style (Section 2.5.4)**

- Assets must be managed on a **strict buy-and-maintain basis**
- Active trading is **not permitted** except to rebalance within existing asset allocation and duration limits
- Leverage and borrowing to meet cash flow shortfalls are **not allowed**
- Assets may only be sold to meet excess liability cash flows not met through maturities and coupons

**Affiliates and Connected Parties (Section 2.5.3)**

Assets with counterparty exposure to affiliates or related/connected parties are **not eligible** without prior BMA approval.

**Derivatives and Hedging (Section 2.5.5)**

- Permitted only for **risk mitigation** purposes with prior BMA approval
- Application must cover modelling assumptions, unhedged risks, collateral management, liquidity risk, and stress testing
- **Dynamic hedging** requiring daily or intraday trading is **not permitted**

**Ringfencing (Section 2.5.6)**

SBA assets must be **ringfenced exclusively** for meeting the policyholder liabilities to which they are assigned. They cannot be used, pledged, or redeployed for any other purpose — including backing a different block of liabilities or new business.

### 5.3 Special Treatment of 258E Assets Without Fixed Maturity

**[Corrected]** In the rare case that a 258E asset has no fixed maturity date (e.g. an open-ended fund or perpetual instrument):

- Per the Rules (Para 28(16)(f)), limited-basis assets **shall not be sold to meet cashflow shortfalls** in SBA projections. However, an exception may exist for assets with no contractual maturity date where **a different treatment has been approved by the Authority** subject to conditions set by the Authority.
- The "zero value on sale" treatment described in the JP Morgan paper is **JP Morgan's conservative modelling assumption**, not a BMA regulatory requirement. The actual regulatory position is that these assets cannot be sold for shortfalls unless the BMA has approved alternative treatment.
- **Practical approach:** Defer any sale as long as possible by prioritising 258C asset sales first; allow the 258E asset to remain in the portfolio generating cash flows until the last possible moment.

**Note on Ringfencing and New Business:** Because of ringfencing, a perpetual 258E asset in one SBA portfolio **cannot** be redeployed to back new business liabilities. New business requires a fresh, separately constructed and ringfenced SBA portfolio.

---

## 6. SBA Model Implementation

### 6.1 Interest Rate Stress Scenarios

The BMA prescribes 9 scenarios: the base case plus 8 interest rate stresses designed to capture a full range of yield curve shocks.

| Scenario | Definition |
|---|---|
| Base | Current forward rates — no stress |
| Rate fall up to year 10 | All rates decrease annually to −1.5% by year 10; unchanged thereafter |
| Rate rise up to year 10 | All rates increase annually to +1.5% by year 10; unchanged thereafter |
| Down-Up (rate fall then recovery) | All rates decrease annually to −1.5% by year 5, then **return to base by year 10** **[Corrected]** |
| Up-Down (rate rise then recovery) | All rates increase annually to +1.5% by year 5, then **return to base by year 10** **[Corrected]** |
| Decrease with positive twist | Year 1: −1.5%, Year 10: −1.0%, Year 30: −0.5% (interpolate) |
| Decrease with negative twist | Year 1: −0.5%, Year 10: −1.0%, Year 30: −1.5% (interpolate) |
| Increase with positive twist | Year 1: +0.5%, Year 10: +1.0%, Year 30: +1.5% (interpolate) |
| Increase with negative twist | Year 1: +1.5%, Year 10: +1.0%, Year 30: +0.5% (interpolate) |

**How cash flows change across scenarios:**

For plain vanilla fixed rate bonds, contractual cash flows are unchanged. However cash flows change across scenarios through:

| Mechanism | Effect |
|---|---|
| Callable bonds | Issuer calls when rates fall below coupon — cash flows accelerate |
| MBS/ABS prepayments | Falling rates trigger prepayments — principal returned early |
| Reinvestment cash flows | Surplus cash reinvested at scenario-specific rates — income differs |
| Disinvestment proceeds | Asset sale prices depend on prevailing rates — falling prices in rising rate scenarios |
| Dynamic lapses (liabilities) | Rising rates → more lapses → liability CFs accelerate; Falling rates → fewer lapses → CFs extend |

**The perfect matching principle:** The BMA notes that if assets and liabilities are perfectly matched, the interest rate stress scenarios would have no impact — the SBA would default to something similar to the UK MA. The 9 scenarios therefore serve as a **penalty mechanism for mismatching** — the worse the matching, the wider the dispersion of BEL across scenarios, and the higher the BEL (maximum across scenarios).

### 6.2 Default Adjustment (Haircut)

The BMA prescribes default and downgrade factors on their website, derived from average historical default rates, covering:
- Bank loans
- Corporate bonds: secured, senior unsecured, subordinated

For assets not covered by BMA assumptions, the insurer must derive their own assumptions (subject to BMA approval) with a **floor** equal to the prescribed senior unsecured corporate bond cost for the equivalent credit rating.

The haircut is applied at the **cash flow level**, not the market value level:

$$\text{Adjusted CF}_{t} = \text{Promised CF}_{t} \times (1 - \text{CoD}_{R,K})$$

Where:
- $\text{CoD}_{R,K}$ = BMA prescribed Cost of Default for rating $R$ and asset class $K$
- The adjustment reduces the cash flows available to meet liabilities
- A higher haircut (lower rated assets) requires a **larger portfolio** to meet the same liabilities

**Example:** A BBB corporate bond with 2.5% BMA default cost contributes only 97.5% of its coupon/principal toward meeting liabilities in the projection.

**[Corrected — important simplification warning]:** This single-period flat haircut formula is a simplified illustration. The BMA's actual requirement (Handbook E10.3) uses **cumulative loss rates** (and marginal loss rates) that increase over time. The correct implementation uses cumulative survival factors:
- `cumulative_survival(t) = (1 - cumulative_default_rate(t)) × (1 - cumulative_downgrade_rate(t) × phase_in_factor)`
- `adjusted_CF(t) = promised_CF(t) × cumulative_survival(t)`

Key differences from this simplified formula:
- Cumulative rates are **non-decreasing** over time (Handbook E10.6 — marginal loss rates floored at zero)
- Default and downgrade components are **separated** — only downgrade has a 5-year phase-in for pre-2024 business (E10.8)
- D&D costs do **NOT affect t=0 market values** or reinvestment purchase prices (E10.5)
- Government debt rated AA- or better in local currency is **exempt** from D&D costs (E10.19)
- For assets without BMA-published costs, floors = corporate senior unsecured (E10.16)

See `BMA_SBA_Consolidated_Guide_v2.0.md` Section 15 for the complete mechanics.

### 6.3 Transaction Costs

Realistic transaction costs must be applied to all assets bought and sold in the SBA projections:
- For liquid publicly traded assets: typically set at observed **bid-ask spreads**
- For less liquid assets: suitable estimates required
- Must also include all applicable **fees, commissions, and expenses**

### 6.4 SBA Projection Model

The core of the SBA is a **dynamic asset-liability projection model** that:

1. Projects asset and liability cash flows across base + 8 interest rate scenarios
2. Models optionality in both assets (callable bonds) and liabilities (dynamic lapses)
3. In each projection year: compares asset cash inflows to liability cash outflows
4. **Excess cash flows** → reinvested into new eligible assets
5. **Negative net cash flows** → assets sold (258C only) to meet shortfall immediately (cannot be rolled forward)
6. Over the full lifetime: model determines the **minimum assets required** to cover all liability cash flows through to maturity **[Corrected — the "zero surplus" / "zero assets AND zero liabilities" terminal condition is JP Morgan's optimisation framing (theta scaling to exactly meet obligations). The BMA Rules (Para 28(10)) require determining the assets needed to cover liability cash flows — conceptually equivalent but not expressed as a zero-surplus target]**

This is a significantly more complex model than the UK MA, which requires a strictly buy-and-hold portfolio with no reinvestment or disinvestment.

**Governance requirements:**
- Significant investment in quantitative functions, technology, data policies, risk management, model documentation, and validation
- Similar governance to Solvency II internal model
- Third-party software may be used, but the insurer **must not outsource** the running, maintaining, and management of the model

### 6.5 Reinvestment Sub-Model

When asset cash flows exceed liability cash flows in a given year, the surplus must be reinvested. The model must specify:
- A **future investment universe** for each year of the projection with at minimum 3 maturity groups (short, medium, long) and rating categories
- **Projected asset prices** in line with each interest rate scenario
- **Reinvestment aligned** to the insurer's board-approved ALM and investment policies

### 6.6 Disinvestment Sub-Model

When asset cash flows are insufficient to meet liability cash flows:
- Assets must be sold **immediately** to cover the shortfall (no rolling forward)
- Only **258C (generally acceptable) assets** can be sold
- **258E assets** cannot be sold to meet cash flow shortfalls
- The model must always ensure sufficient 258C assets remain available — otherwise a material mismatch exists
- The disinvestment strategy must align to the insurer's actual investment management practices

**Special case — 258E asset without fixed maturity:**
- **[Corrected]** The Rules (Para 28(16)(f)) state these assets cannot be sold for shortfalls, but the BMA may approve alternative treatment for no-maturity assets. The "zero proceeds on sale" is JP Morgan's conservative modelling choice, not a regulatory mandate.
- Mitigation: prioritise 258C sales and defer 258E sale as far as possible into the future

**Projected sale prices** must reflect the scenario-specific interest rate environment at the point of sale (e.g. bond prices fall in rising rate scenarios).

### 6.7 Liquidity Risk Management

This is a **separate and additional** framework from the 9 SBA interest rate stress scenarios. The SBA scenarios test asset-liability cash flow matching under interest rate stress; the liquidity risk management framework addresses **operational liquidity survival**.

| Feature | SBA 9 Scenarios | Liquidity Risk Management |
|---|---|---|
| Purpose | Measure BEL and illiquidity premium | Ensure operational cash availability |
| Focus | Asset-liability CF matching | Cash demand and supply dynamics |
| Output | BEL quantum | Liquidity buffer size and risk appetite |
| Governance | Regulatory model requirement | Board-level risk framework |
| Stress type | Interest rate curve movements | Liquidity demand/supply shocks (lapses, collateral calls, market dislocations) |

**Required components:**
- Board-set liquidity risk appetite (duration and severity of stress scenarios the insurer targets to withstand)
- Cash needs and sources register
- Liquidity buffer — a pool of highly liquid assets dedicated to covering cash flow deficiencies
- Liquidity stress and scenario testing including **reverse stress testing** to identify the insurer's liquidity breaking point
- Liquidity contingency plan with defined triggers, regularly reviewed and tested
- Liquidity metrics and thresholds embedded into board-level risk reporting
- Ongoing liquidity reporting

### 6.8 Well Matched Portfolios

The BMA requires formal documentation of matching quality. Insurers must define numerical thresholds that, if exceeded, would indicate the portfolio is mismatched. Key metrics:

| Metric | Description |
|---|---|
| Cost of mismatch | Difference in BEL between base scenario and the "biting" (worst) scenario |
| Dispersion across 8 scenarios | Standard deviation or range of BEL across all 8 stress scenarios |
| Regulatory capital ratios / BEL | Currency, interest rate, lapse, mortality, morbidity, longevity risk as % of BEL |
| Currency matching effectiveness | Size of assets in non-liability currency; hedging residual risks |
| Asset sales to meet shortfalls | Total asset sales as % of BEL |
| Reinvestment dependency | Graphical and sensitivity analysis of reinvestment impact on BEL across scenarios |
| Accumulated cash flow shortfall | Highest accumulated shortfall across all projection years as % of BEL |
| Duration and key rate durations | Duration gap and key rate duration gap vs internal tolerance levels |
| Fungibility | Where matching relies on fungibility across business blocks — must be documented, tested, and limited to the same legal entity |

---

## 7. BEL Calculation — Mathematical Formulation

> **[Corrected — editorial note]:** The mathematical formulation below (lambda/theta scaling, optimisation framework) is **J.P. Morgan's proprietary portfolio construction methodology**, not the BMA's prescribed BEL calculation. The BMA Rules (Para 28(10)) simply require: (a) determine assets needed under base scenario, (b) determine assets needed under each stress scenario, (c) BEL = the highest requirement across all 9 scenarios. JP Morgan's scaling approach is one valid way to find this answer, but it is not the only approach and is not prescribed by the regulator. This section remains valuable as an educational exposition of the underlying mathematics.

### 7.1 Conceptual Foundation

> **Critical insight:** In the SBA, you are **not discounting liabilities**. You are **scaling assets** until the portfolio survives all stress scenarios. The BEL is simply the **cost of that surviving asset portfolio at t=0**.

This is fundamentally different from the standard approach where:
$$\text{BEL}_{Standard} = \sum_{t=1}^{T} \frac{L_t}{(1 + \text{RFR}_t)^t}$$

### 7.2 The Master Optimisation Problem

For each scenario $s \in \{1, \ldots, 9\}$, find the **minimum scaling factor** $\lambda_s^*$ applied to a reference eligible asset portfolio $P_{ref}$ such that all constraints are satisfied:

$$\lambda_s^* = \min \{ \lambda > 0 \mid \text{Constraints}(\lambda, s) \text{ are satisfied} \}$$

The **SBA BEL** is then:

$$\text{BEL}_{\text{SBA}} = \max_{s \in \{1,\ldots,9\}} \left( \lambda_s^* \cdot \text{MV}(P_{ref}) \right)$$

**Why the maximum?** This is the conservative regulatory principle — the insurer must hold enough assets to cover liabilities under the **worst case** scenario across all 9 stress tests.

### 7.3 Precise Mathematical Formulation (Theta Notation)

Let:
- $L_t^{(s)}$ = liability cash outflow at time $t$ under scenario $s$
- $A_t^{(s)}$ = default-adjusted asset cash inflow at time $t$ under scenario $s$
- $R_t^{(s)}$ = reinvestment cash flow (when $A_t > L_t$, surplus reinvested)
- $D_t^{(s)}$ = disinvestment proceeds (when $A_t < L_t$, from selling 258C assets only)
- $\theta^{(s)}$ = scalar applied to starting asset portfolio weights — the solve-for variable

For each scenario $s$, solve for $\theta^{(s)}$ such that:

$$\sum_{t=0}^{T} \left[ \theta^{(s)} \cdot A_t^{(s)} + R_t^{(s)} - D_t^{(s)} - L_t^{(s)} \right] \cdot \text{discount}_t^{(s)} = 0$$

With constraints:
- $D_t^{(s)}$ can only come from 258C assets
- No borrowing: cumulative net cash flow $\geq 0$ at all $t$
- Final surplus = 0 at $t = T$

Then:
$$\text{BEL} = \max_{s} \left\{ \theta^{(s)} \cdot \text{MV}_0^{\text{base}} \right\}$$

Where $\text{MV}_0^{\text{base}}$ is the market value of the reference asset portfolio at $t_0$ under base rates.

### 7.4 The Recursive Cash Flow Logic

At each time step $t$, available cash is:

$$\text{Cash}_{t,s} = \underbrace{\sum_{i \in \text{Assets}} CF_{i,t,s} \cdot (1 - H_{i,t})}_{\text{Default-Adjusted Asset CF}} + \underbrace{Inv_{t-1,s} \cdot (1 + r_{\text{reinv},s})}_{\text{Reinvestment Income}} + \underbrace{Sale_{t,s}}_{\text{Disinvestment}}$$

Where:
- $CF_{i,t,s}$ = promised cash flow of asset $i$ at time $t$ in scenario $s$
- $H_{i,t}$ = BMA default/downgrade haircut (% reduction applied to cash flow, not MV)
- $Inv_{t-1,s}$ = cash surplus reinvested from period $t-1$
- $r_{\text{reinv},s}$ = reinvestment rate in scenario $s$
- $Sale_{t,s}$ = proceeds from selling 258C assets only

**Liquidity constraint (no borrowing):**

$$\text{Cash}_{t,s} \geq L_{t,s} + \text{Exp}_{t,s} \quad \forall t$$

If $\text{Cash}_{t,s} < L_{t,s} + \text{Exp}_{t,s}$, then disinvestment must be triggered immediately:

$$Sale_{t,s} \geq (L_{t,s} + \text{Exp}_{t,s}) - (\text{Organic Asset CF} + \text{Reinvestment})$$

**Terminal constraint:**

$$\text{Surplus}_{T,s} = 0$$

**[Corrected]:** The "terminal surplus = 0" constraint is JP Morgan's theta-scaling formulation. The BMA Rules (Para 28(10)) simply require determining the **minimum assets needed** to cover all liability cash flows through to maturity under each scenario. The regulatory approach is: for a given asset portfolio, find the minimum initial cash buffer (C0) such that the cash balance never goes negative at any timestep — i.e., `min(cash_balance_t) >= 0 for all t`. The two formulations are mathematically related but conceptually different: JP Morgan scales the portfolio (theta), while the regulatory approach fixes the portfolio and solves for the cash buffer (C0).

### 7.5 The Default Adjustment — Detail

$$\text{Adjusted CF}_{t,s} = \text{Promised CF}_{t,s} \times (1 - \text{CoD}_{R,K})$$

**[Corrected]:** This is the same simplified formula as Section 6.2. See the correction note there for the actual BMA cumulative survival factor mechanics (E10.3-E10.6). The single-period flat CoD understates the time-varying nature of default and downgrade costs.

**Impact on BEL:** A higher CoD (lower rated asset) reduces the adjusted cash flows available to meet liabilities, requiring a larger initial portfolio (higher $\lambda$ or $\theta$) to meet the same obligations — directly increasing the BEL.

**Relationship to illiquidity premium:** Higher yielding (lower rated) bonds require a larger haircut but offer higher market yield. The **net effect after haircut** determines the illiquidity premium — the residual excess return after expected credit losses are stripped away.

### 7.6 The SBA Illiquidity Premium

$$\text{Illiquidity Premium} = \text{IRR}(\text{default-adjusted asset CFs}) - \text{IRR}(\text{liability CFs at risk-free rate})$$

**[Corrected]** This is JP Morgan's derived analytical measure, not a BMA-defined quantity. The BMA does not prescribe or reference the illiquidity premium in the SBA rules — it is a useful way to quantify the SBA benefit but is not part of the regulatory calculation.

In the JP Morgan case study, this achieved **179bps** versus approximately **133bps** under the UK MA — a material advantage driven by access to higher-yielding assets, the 258E bucket, and the flexibility of reinvestment/disinvestment modelling.

---

## 8. Theta Scaling — Goal Seek Logic

### 8.1 Why the Equation is Recursive (Non-Linear)

The key insight is that the reinvestment cash flows $R_t$ and disinvestment proceeds $D_t$ are **themselves functions of $\theta$**:

- If $\theta$ is larger, asset cash flows are larger → different surplus/deficit at each $t$ → different $R_t$ and $D_t$ → different terminal surplus
- This creates a **chain of dependencies** across time:

```
t=1 outcome → feeds into → t=2 outcome → feeds into → t=3 → ... → t=T surplus
```

**At t=1:**
$$\text{Net CF}_1 = \theta \cdot A_1 - L_1$$
- If Net CF₁ > 0: Reinvest → generates $R_2 = \text{Net CF}_1 \times (1 + r_{\text{reinv}})$ which feeds into t=2
- If Net CF₁ < 0: Sell 258C assets → reduces 258C pool available at t=2

**At t=2:**
$$\text{Net CF}_2 = \theta \cdot A_2 + R_2 - D_2 - L_2$$
Where $R_2$ depends on t=1 surplus and $D_2$ depends on remaining 258C pool after t=1 sales.

Each subsequent period depends on all prior periods — you **cannot solve analytically**. You must run the full forward projection for every trial value of $\theta$.

### 8.2 What is Fixed and What Changes

| Variable | Status | Reason |
|---|---|---|
| $L_t$ — liability CFs | **Fixed** | Policyholder obligations — outside insurer's control |
| $A_t$ per unit — asset CFs before scaling | **Fixed** | Contractual bond cash flows |
| $H_{i,t}$ — BMA haircuts | **Fixed** | Prescribed by BMA |
| $r_{\text{reinv},s}$ — reinvestment rates | **Fixed** | Prescribed per scenario |
| Interest rate scenario $s$ | **Fixed** | One of 9 prescribed scenarios per run |
| **$\theta$** | **Variable** | The single goal-seek variable |
| Target: Terminal Surplus at $t=T$ | **= 0** | The goal-seek target |

### 8.3 Goal Seek Algorithm — Step by Step

```
Step 1: Fix scenario s and all input assumptions
Step 2: Initial guess θ = 1.0

Step 3: Run full recursive projection t = 1 to T
   For each t:
     Calculate θ × A_t    (scaled asset CF)
     Add R_t              (reinvestment from prior surplus)
     Calculate Net CF_t = θ×A_t + R_t - L_t

     If Net CF_t > 0:
       → Reinvest surplus: store as R_{t+1}
       → 258C pool unchanged

     If Net CF_t < 0:
       → Sell 258C assets: D_t = |Net CF_t|
       → Reduce remaining 258C pool

     Move to t+1

Step 4: Check terminal surplus at t=T
   If Surplus_T > 0  → θ is too high → reduce θ
   If Surplus_T < 0  → θ is too low  → increase θ
   If Surplus_T = 0  → θ* found ✓

Step 5: Repeat Steps 3–4 with updated θ until convergence
Step 6: Repeat Steps 1–5 for all 9 scenarios
Step 7: BEL = max(θ_s* × MV_0^base) across all s
```

### 8.4 Numerical Example (3-Year Projection)

Assume: 3-year projection, single scenario, no reinvestment income for simplicity.

| $t$ | $L_t$ | $A_t$ per unit |
|---|---|---|
| 1 | 50 | 30 |
| 2 | 50 | 40 |
| 3 | 50 | 80 |

**Trial 1: θ = 1.0**

- t=1: Net CF = 1.0×30 − 50 = −20 → sell 258C, D₁ = 20
- t=2: Net CF = 1.0×40 − 50 = −10 → sell 258C, D₂ = 10
- t=3: Net CF = 1.0×80 − 50 = +30 → surplus
- Terminal Surplus = +30 → θ too high, reduce

**Trial 2: θ = 0.85**

- t=1: Net CF = 0.85×30 − 50 = −24.5 → sell 258C
- t=2: Net CF = 0.85×40 − 50 = −16 → sell 258C
- t=3: Net CF = 0.85×80 − 50 = +18
- Net Terminal = 18 − 24.5 − 16 = −22.5 → θ too low, increase

**Iterate between 0.85 and 1.0 until Terminal Surplus = 0** → this is θ* for this scenario.

**[Corrected — important context]:** This numerical example illustrates JP Morgan's theta-scaling approach. While the mathematics is valid as a portfolio construction tool, it has three simplifications that do not reflect the BMA's actual requirements:

1. **No default/downgrade adjustment** — The example uses raw asset CFs. The BMA requires cumulative survival factors applied to all projected asset cash flows (Para 28(22)-(23), E10.3).
2. **No transaction costs on disinvestment** — When 258C assets are sold (D₁, D₂), the proceeds should be reduced by bid-ask costs and cumulative D&D (Para 28(30)-(31), Para 28(34)(c), E11.2).
3. **Theta scales the entire portfolio uniformly** — The BMA does not prescribe portfolio scaling. The regulatory approach is: given a fixed portfolio, find the minimum cash buffer (C0) needed so the cash balance stays non-negative at every timestep across all 9 scenarios (Para 28(10)).

For a worked example that follows the BMA's actual methodology, see `BMA_doc/BMA_SBA_Illustrative_Calculation_Comprehensive.md`.

---

## 9. Liability Scaling — SBA vs Standard Carve-Out

> **[Corrected — editorial note]:** The "liability scaling" and "SBA vs Standard carve-out" concept below is JP Morgan's practical design pattern for portfolio construction. The BMA Rules (Para 28(1)) simply state that insurance groups may apply SBA "to some or all of long-term business." The Rules do not prescribe a theta-based liability scaling mechanism — this is JP Morgan's interpretation of how to handle partial SBA coverage.

### 9.1 When Assets < Liabilities

When available SBA-eligible assets are insufficient to cover all eligible liabilities in full, the framework does **not** require scaling up assets. Instead, liabilities are **carved out** between two valuation approaches:

$$\text{Total Liabilities} = \underbrace{L_{\text{SBA}}}_{\text{Matched by available assets}} + \underbrace{L_{\text{Standard}}}_{\text{Remainder on standard approach}}$$

| Portion | Treatment | Discount Rate | BEL Impact |
|---|---|---|---|
| $L_{\text{SBA}}$ — matched by available assets | SBA — illiquidity premium applied | RFR + Illiquidity Premium | Lower BEL |
| $L_{\text{Standard}}$ — unmatched remainder | Standard approach | RFR only | Higher BEL |

### 9.2 Regulatory Logic

This is a pragmatic regulatory design — the insurer receives **partial SBA benefit** proportional to the assets actually available, rather than an all-or-nothing outcome. The SBA benefit is maximised for the matched portion; the unmatched portion remains on the standard (more conservative) approach.

The scaling variable ($\theta < 1$ in this case) reduces the **liability block** assigned to the SBA portfolio, not the assets themselves — determining the size of the SBA-eligible liability group that the available assets can support.

---

## 10. Portfolio Optimisation

### 10.1 The Optimisation Framework

The JP Morgan approach goes beyond simply finding $\theta$ for a given reference portfolio. The optimisation **simultaneously determines the optimal portfolio composition** that:

- **Minimises $\theta \cdot \text{MV}$** (minimises BEL / cost of the matching portfolio)
- Equivalently: **maximises the illiquidity premium**
- Subject to all SBA regulatory constraints and good risk management constraints

**Objective function:**

$$\min_{\text{portfolio weights}} \left[ \text{MV}_0 \text{ subject to all constraints being satisfied across all 9 scenarios} \right]$$

### 10.2 Optimisation Constraints

**Diversification constraints:**
- 3.0% limit per corporate issuer rated A and above
- 2.0% limit per corporate issuer rated BBB
- 25% limit per country (excluding UK and US) for Emerging Market Debt
- Sector mix within 5–10% of benchmark distribution
- 0.75% limit per ISIN

**Regulatory constraints:**
- Portfolio BSCR < 4% over all future years
- Positive net cumulative cash flows every year over the liability timeline
- Net assets = zero at end of liability runoff
- Positive net cash flows: must be reinvested in 258C assets or cash
- Negative net cash flows: must be met through sale of 258C assets only
- Net duration within ±1 year
- Net key rate durations within ±1 year
- Illiquid assets below 20%

**[Corrected]:** These are labelled "regulatory constraints" but they are actually a **mix of BMA rules and JP Morgan's own investment management constraints**. Of the above, only the following are BMA-mandated:
- Positive net cash flows reinvested per declared strategy (Para 28(33))
- Negative net cash flows met by selling eligible (Tier 1/2) assets only (Para 28(34)(a))
- 258E (limited-basis) assets cannot be sold for shortfalls (Para 28(16)(f))
- 258E assets capped at 10% aggregate, 0.5% per single asset (Para 28(16)(a),(d))

The BSCR < 4% target, ±1 year duration/KRD limits, 20% illiquid cap, and "net assets = zero at runoff" are JP Morgan's risk management and optimisation constraints — reasonable but not BMA-prescribed.

### 10.3 Case Study Results (JP Morgan, June 2024)

**Liability:** Sample pension liability cash flows (UK pension fund profile, extending to 2077)

**Investment universe:** ~4,000 bonds across global fixed income (public 258C assets) plus alternative assets (258E)

**Result:**

| Metric | Value |
|---|---|
| IRR of Assets | 5.68% |
| IRR of Liabilities (risk-free) | 3.90% |
| **Illiquidity Premium** | **179 bps** |
| BSCR (Year 0) | 2.47% |

**Why 179bps vs ~133bps for UK MA:**
1. Inclusion of 258E assets (high IRR alternatives: direct lending, corporate mezzanine, transportation leasing)
2. Reinvestment and disinvestment flexibility — allows better optimisation than strict buy-and-hold
3. Access to the full US securitised market

**Scale consideration:** For very large insurers, issuer concentration limits would force holdings closer to the passive index, reducing the achievable illiquidity premium. The full benefit is most accessible to small-to-mid sized insurers.

### 10.4 Ongoing Portfolio Rebalancing

Portfolio construction is not a one-time exercise. The same optimisation framework handles:
- **Rebalancing:** As liability estimates update, portfolio is reoptimised with turnover as an additional constraint
- **New premium:** Integrated into the same framework to measure aggregate impact on capital and diversification

---

## 11. BSCR, MCR and the Standard Approach

### 11.1 Granularity of Calculations

| Metric | Granularity | Notes |
|---|---|---|
| **BEL** | Liability group level | Each group has dedicated ringfenced assets; may use SBA or Standard approach depending on eligibility |
| **BSCR** | Company level | Aggregates all risks across all liability groups and business lines |
| **MCR** | Company level | Absolute regulatory floor — % of BSCR or fixed minimum, whichever is higher |

### 11.2 The Full Risk Universe — BSCR

The BSCR captures all material risks the insurer faces:

| Risk Category | Examples |
|---|---|
| Market Risk | Interest rate, credit spread, equity, currency, property |
| Insurance Risk | Longevity, mortality, lapse, morbidity |
| Credit Risk | Counterparty default |
| Operational Risk | Systems failure, fraud, process errors |
| Liquidity Risk | Cash flow mismatches |

The SBA **indirectly reduces** the BSCR because:
- Lower SBA BEL → lower liabilities → higher net assets → better solvency ratio
- BSCR capital charges on spread assets **do not increase with duration** — a key Bermuda advantage
- The 258E bucket provides access to higher-yielding alternatives at manageable BSCR capital charges

### 11.3 The Standard Approach

The Standard Approach is a Solvency II-style valuation excluding the Matching Adjustment and Volatility Adjustment:

$$\text{BEL}_{Standard} = \sum_{t=1}^{T} \frac{L_t}{(1 + \text{RFR}_t)^t}$$

Simply discounting liability cash flows at the **risk-free curve** — no illiquidity premium, no asset matching requirement.

| Feature | Standard Approach | SBA |
|---|---|---|
| Liability discount rate | Risk-free curve only | RFR + Illiquidity Premium |
| Asset-liability matching | Not required | Mandatory ringfencing |
| Cash flow projection | Simple present value | Full dynamic projection across 9 scenarios |
| Reinvestment modelling | Not required | Explicitly modelled |
| Regulatory approval | Not required | Explicit BMA approval needed |
| BEL level | Higher | Lower — benefit of illiquidity premium |

### 11.4 The Full Regulatory Framework — Layer View

```
┌────────────────────────────────────────────────────┐
│            BERMUDA REGULATORY FRAMEWORK            │
│                                                    │
│  ┌───────────────┐    ┌───────────────────────┐   │
│  │     BSCR      │    │  VALUATION (BEL)      │   │
│  │  (Capital)    │    │                       │   │
│  │               │    │  SBA Eligible Blocks  │   │
│  │ Market Risk   │    │  → SBA BEL            │   │
│  │ Insurance     │    │    (RFR + IP)         │   │
│  │ Credit Risk   │    │                       │   │
│  │ Op Risk       │    │  Other Blocks         │   │
│  │ Liq Risk      │    │  → Standard BEL       │   │
│  │               │    │    (RFR only)         │   │
│  └──────┬────────┘    └──────────┬────────────┘   │
│         │                        │                 │
│         └────────────┬───────────┘                 │
│                      ↓                             │
│            Solvency Ratio =                        │
│         Eligible Capital / BSCR                    │
│                      ↓                             │
│     Must exceed 100% ECR (Enhanced Capital Req.)   │
│     Must exceed MCR (absolute minimum floor)       │
└────────────────────────────────────────────────────┘
```

---

## 12. Bermuda vs UK Matching Adjustment

| Feature | UK Matching Adjustment | Bermuda SBA |
|---|---|---|
| Reinvestment modelling | ❌ Not allowed — strict buy and hold | ✅ Explicitly modelled and required |
| Disinvestment modelling | ❌ Not allowed | ✅ Explicitly modelled |
| US securitised market | Limited access | ✅ Full access (with prior approval) |
| BSCR spread duration charge | Increases with duration | ✅ Does not increase with duration |
| Higher-yield alternative assets | ❌ No equivalent | ✅ 258E bucket — 10% allowance |
| Bonds with optionality | ❌ Strict restrictions | ✅ Allowed with scenario modelling |
| Policyholder options | Largely excluded | ✅ Allowed with adequate modelling |
| Illiquidity premium achieved | ~133 bps (JP Morgan case study) | ✅ ~179 bps (JP Morgan case study) |
| Regulatory approval | Required (PRA) | Required (BMA) — new from 2024 |
| Governance investment | Very significant | Significant but comparable |

**Conclusion:** The Bermuda SBA provides materially more flexibility and a higher achievable illiquidity premium than the UK MA, while requiring a similar level of governance investment. This explains why Bermuda has become the dominant jurisdiction for pension risk transfer and life reinsurance globally.

---

## 13. The SBA Business Model — End-to-End

### 13.1 The Pension Risk Transfer Value Chain

```
PENSION FUND                        BERMUDA LIFE REINSURER
━━━━━━━━━━━━━━━━━━━                 ━━━━━━━━━━━━━━━━━━━━━━
Has pension liabilities    →        Takes on pension liabilities
Wants balance sheet        →        Pays certain annuities
certainty
Transfers longevity risk   →        Manages longevity + investment risk

            HOW THE REINSURER MAKES MONEY:
            ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            1. Receives premium from pension fund
            2. Invests in SBA ringfenced credit portfolio
            3. Earns illiquidity premium (e.g. 179bps over RFR)
            4. SBA BEL < premium received
            5. Difference = profit / capital release
            6. Capital recycled into writing new business
```

### 13.2 How the SBA Creates and Distributes Value

| Benefit | Mechanism |
|---|---|
| Lower BEL | Illiquidity premium reduces liability valuation — same cash flows discounted at a higher rate |
| Less capital needed | Lower BEL → lower BSCR requirement → more free capital |
| Higher investment returns | Access to illiquid private credit, securitised markets, 258E alternatives |
| Competitive pricing | Insurer can offer better terms to pension funds than competitors using standard approach |
| Balance sheet efficiency | More business underwritten per unit of deployed capital |
| Regulatory capital arbitrage vs UK | Higher illiquidity premium at lower capital charge than UK MA equivalent |

### 13.3 The Single Sentence Summary

> **The SBA allows a well-managed insurer to transform the illiquidity of long-dated credit assets into a tangible balance sheet and capital advantage — provided it has the systems, governance, and expertise to demonstrate a high degree of asset-liability matching across all regulatory stress scenarios.**

---

## 14. Duration, KRD and DV01 — The Rate Sensitivity Toolkit

### 14.1 Duration — The Foundation

#### Intuition

Duration is the **weighted average time** at which you receive the cash flows from a bond, where the weights are the **present values** of each cash flow as a proportion of the total bond price. Think of it as the **centre of gravity** of a bond's cash flow stream on a timeline.

**Simple example — two bonds, both 5-year maturity:**

| | Bond A (Zero Coupon) | Bond B (10% Coupon) |
|---|---|---|
| Cash flows | Single payment at year 5 only | Coupons years 1–5 + principal at year 5 |
| Centre of gravity | Year 5 — exactly | Earlier than year 5 — pulled forward by coupons |
| Duration | 5.0 years | ~4.1 years |

Bond B's duration is shorter than its maturity because the coupons pull the centre of gravity forward in time.

#### Why Duration Matters — Price Sensitivity

Duration directly measures **price sensitivity to interest rate changes**:

$$\frac{\Delta P}{P} \approx -D_{\text{mod}} \times \Delta y$$

Where:
- $\Delta P / P$ = percentage change in bond price
- $D_{\text{mod}}$ = modified duration
- $\Delta y$ = change in yield (interest rate)

**Numerical example:**

| Bond | Modified Duration | Rate rises 1% | Price change |
|---|---|---|---|
| Short bond | 2 years | +1% | −2% |
| Long bond | 15 years | +1% | −15% |

A bond with duration 15 years loses 15% of its value for every 1% rise in interest rates.

#### Macaulay Duration vs Modified Duration

| Type | Formula | Use |
|---|---|---|
| Macaulay Duration | $D_{mac} = \frac{\sum_{t=1}^{T} t \cdot \frac{CF_t}{(1+y)^t}}{P}$ | Weighted average time — the "centre of gravity" |
| Modified Duration | $D_{mod} = \frac{D_{mac}}{1 + y}$ | Price sensitivity — primary ALM measure |

### 14.2 The Problem with a Single Duration Number

Duration measures sensitivity to a **parallel shift** of the entire yield curve — all rates moving up or down by the same amount simultaneously. But in reality yield curves **twist and change shape** — which is precisely why the BMA prescribes twist scenarios among the 9 SBA stress tests.

**Consider this:** A portfolio with the correct overall duration could still be badly mismatched if its assets are concentrated in one maturity bucket and its liabilities in another. A twist scenario would expose this mismatch even when total duration appears balanced. This is where **Key Rate Duration** solves the problem.

### 14.3 Key Rate Duration (KRD)

KRD measures the **sensitivity of a portfolio's value to a change in yield at a specific maturity point on the curve**, holding all other maturity yields constant. Instead of one number, you get a **profile of sensitivities across the curve**:

$$\text{KRD}_k = -\frac{1}{P} \cdot \frac{\Delta P}{\Delta y_k}$$

Where $\Delta y_k$ = a small shift (typically 1 basis point) **only at maturity point $k$**, all other points unchanged.

The KRDs across all key rate points sum to approximately the total modified duration:

$$D_{\text{mod}} \approx \sum_{k} \text{KRD}_k$$

**Example — from JP Morgan Case Study (Exhibit 11):**

| Key Rate | Asset KRD | Liability KRD | Net KRD (A − L) |
|---|---|---|---|
| 6 months | — | — | −0.02 |
| 2 years | — | — | −0.22 |
| 5 years | — | — | −0.74 |
| 10 years | — | — | −0.84 |
| 20 years | — | — | +0.29 |
| 30 years | — | — | +0.81 |

Reading this:
- **Negative values:** Liabilities are more sensitive than assets at that maturity — a rate fall there increases BEL more than asset value
- **Positive values:** Assets are more sensitive than liabilities — a rate fall there increases asset value more than BEL
- Small mismatches within the ±1 year constraint are acceptable

#### KRD Profile of Pension Liabilities

Pension annuity liabilities typically have very **long duration** with KRD concentrated at long maturities, because pension payments extend decades into the future:

```
KRD
 |
4|                          ████
 |                     ████████
3|               ██████████████
 |          ████████████████████
2|     ██████████████████████████
 |  ████████████████████████████████
1|████████████████████████████████████
 |────────────────────────────────────→
  6m  2yr  5yr  10yr  15yr  20yr  30yr

Heavy concentration in long maturities
because pension payments extend decades
```

### 14.4 DV01 — Dollar Value of 1 Basis Point

#### Definition

DV01 (also called PV01 or PVBP) measures the **absolute change in market value** when rates move by **1 basis point (0.01%)**:

$$\text{DV01} = D_{\text{mod}} \times P \times 0.0001$$

Where $P$ = current market value of the bond or portfolio.

**Simple numerical example:**

- Bond market value = £10,000,000
- Modified duration = 15 years

$$\text{DV01} = 15 \times £10,000,000 \times 0.0001 = £15,000 \text{ per bp}$$

For every 1 basis point move in rates, the bond's value changes by £15,000.

#### Why DV01 is More Useful Than Duration in Practice

Duration is a **relative** measure (% change per % rate move). DV01 is an **absolute** measure (£ change per bp). Consider:

| Portfolio | Duration | Market Value | DV01 |
|---|---|---|---|
| A | 15 years | £100m | £150,000 per bp |
| B | 15 years | £500m | £750,000 per bp |

Both portfolios have the **same duration** but Portfolio B has **5× the absolute rate risk**. Duration alone does not reveal this. DV01 does — which is why risk managers, traders, and regulators work in DV01 terms when actual financial impact matters.

### 14.5 Key Rate DV01 (KR-DV01)

Just as KRD breaks duration down by maturity point, **KR-DV01** breaks DV01 down by maturity point:

$$\text{KR-DV01}_k = \text{KRD}_k \times P \times 0.0001$$

This tells you: how much does the portfolio value change if **only the yield at maturity point $k$** moves by 1bp?

**Example — Pension Annuity Portfolio (£500m):**

| Key Rate Point | KRD | KR-DV01 (£) |
|---|---|---|
| 2 years | 0.30 | £150,000 |
| 5 years | 1.20 | £600,000 |
| 10 years | 2.80 | £1,400,000 |
| 20 years | 3.50 | £1,750,000 |
| 30 years | 4.20 | £2,100,000 |
| **Total** | **12.00** | **£6,000,000** |

Reading: if the 30-year yield moves by 1bp, the portfolio changes in value by £2,100,000 — immediately showing where the biggest rate exposures sit.

### 14.6 The Relationship Triangle

Duration, KRD and DV01 all measure the **same underlying concept** — interest rate sensitivity — expressed in different units:

```
                    INTEREST RATE SENSITIVITY
                           /        \
                          /          \
              DURATION              DV01
           (Relative, %)         (Absolute, £)
           "% change per        "£ change per
            1% rate move"        1bp rate move"
                  \                  /
                   \                /
                    \              /
                    KRD / KR-DV01
                    (Broken down by
                     maturity point)
```

Key relationships:

$$\text{DV01} = D_{\text{mod}} \times P \times 0.0001$$

$$D_{\text{mod}} = \frac{\text{DV01}}{P \times 0.0001}$$

$$\text{Total DV01} = \sum_{k} \text{KR-DV01}_k$$

$$\text{Total Duration} = \sum_{k} \text{KRD}_k$$

### 14.7 Duration, KRD and DV01 in the SBA Context

#### The Core ALM Matching Objective

The SBA requires a **high degree of matching** at two levels:

**Level 1 — Total Duration / DV01 Matching (parallel shift protection):**

$$D_{\text{mod}}^{\text{Assets}} \approx D_{\text{mod}}^{\text{Liabilities}}$$
$$\text{DV01}^{\text{Assets}} \approx \text{DV01}^{\text{Liabilities}}$$

This ensures that when rates shift in parallel, both sides of the balance sheet move by approximately the same amount — net position protected.

**Level 2 — KRD / KR-DV01 Matching (twist protection):**

$$\text{KRD}_k^{\text{Assets}} \approx \text{KRD}_k^{\text{Liabilities}} \quad \forall k$$
$$\text{KR-DV01}_k^{\text{Assets}} \approx \text{KR-DV01}_k^{\text{Liabilities}} \quad \forall k$$

This ensures matching at every point on the yield curve — essential to protect against the twist scenarios.

#### The Net Duration Gap

$$\text{Duration Gap} = D_{\text{mod}}^{\text{Assets}} - D_{\text{mod}}^{\text{Liabilities}}$$

| Duration Gap | Meaning | Rate Rise Impact | Rate Fall Impact |
|---|---|---|---|
| Zero | Perfect match | Net position unchanged | Net position unchanged |
| Positive (assets longer) | Assets more rate-sensitive | Net loss | Net gain |
| Negative (assets shorter) | Liabilities more rate-sensitive | Net gain | Net loss |

In the JP Morgan case study, the net duration was **−0.72 years** — liabilities very slightly longer than assets, within the ±1 year SBA tolerance.

#### Numerical DV01 Mismatch Example

Suppose:
- Asset portfolio MV = £800m, Duration = 14.5 years
- Liability BEL = £800m, Duration = 15.2 years

$$\text{DV01}^{\text{Assets}} = 14.5 \times £800m \times 0.0001 = £1,160,000 \text{ per bp}$$

$$\text{DV01}^{\text{Liabilities}} = 15.2 \times £800m \times 0.0001 = £1,216,000 \text{ per bp}$$

$$\text{Net DV01} = £1,160,000 - £1,216,000 = -£56,000 \text{ per bp}$$

For every 1bp fall in rates, the insurer loses a net £56,000 (liabilities rise more than assets). Across the SBA scenarios involving rate falls, this mismatch is penalised through a higher BEL.

#### How Each Measure Maps to the 9 SBA Scenarios

| Scenario Type | Scenarios | Primary Metric Exposed |
|---|---|---|
| Parallel shift up/down | Scenarios 1–4 | Total Duration Gap / Net DV01 |
| Positive/Negative Twist | Scenarios 5–8 | KR-DV01 mismatch at specific maturity points |

**The perfect match principle:** If assets and liabilities are perfectly matched at every KRD/KR-DV01 point across the curve, none of the 9 scenarios would change the BEL — because any curve movement would produce an exactly equal and offsetting change on both sides. The SBA BEL would collapse to the base scenario BEL — equivalent in effect to the UK MA. The 9 stress scenarios only add to the BEL through imperfect KRD matching.

#### The Practical ALM Workflow

```
Step 1 — Check Total Duration Gap
   D(Assets) − D(Liabilities) ≈ 0?  (±1 year SBA tolerance)
   Quick sanity check — are both sides broadly aligned?
            ↓
Step 2 — Check Total DV01 Match
   DV01(Assets) − DV01(Liabilities) ≈ £0?
   Absolute £ risk — how much does balance sheet move per bp?
            ↓
Step 3 — Check KRD / KR-DV01 Profile
   Net KR-DV01(k) ≈ £0 for every maturity point k?
   Are mismatches within the ±1 year per key rate tolerance?
            ↓
Step 4 — Run the 9 SBA Scenarios
   KR-DV01 mismatches at each maturity point directly drive
   BEL dispersion across the 9 scenarios
   Well-matched portfolio → minimal BEL variation → lower max BEL
```

### 14.8 Complete Summary Table

| Measure | Formula | Unit | What It Tells You | SBA Use |
|---|---|---|---|---|
| Macaulay Duration | Weighted avg time of CFs | Years | Centre of gravity of cash flows | Conceptual understanding |
| Modified Duration | $D_{mac} / (1+y)$ | Years | % price change per 1% rate move | ALM matching ratio |
| DV01 | $D_{mod} \times P \times 0.0001$ | £ per bp | Absolute £ change per 1bp rate move | Risk limits, hedging, P&L attribution |
| KRD | Sensitivity at maturity point $k$ | Years | % price change per 1% move at point $k$ | Twist scenario matching quality |
| KR-DV01 | $\text{KRD}_k \times P \times 0.0001$ | £ per bp at $k$ | Absolute £ change per 1bp at maturity $k$ | Hedging specific curve points |

### 14.9 The One-Paragraph Intuition

> Duration and DV01 are the **same thing in different units** — duration is relative (% per %), DV01 is absolute (£ per bp). KRD and KR-DV01 are simply duration and DV01 **broken down by maturity point** instead of expressed as a single total number. In the SBA context, total duration/DV01 matching protects against the parallel shift scenarios, while KRD/KR-DV01 matching protects against the twist scenarios — together they form the complete toolkit for demonstrating the "high degree of matching" that the BMA requires.

---

## Key Formulae Reference

**Illiquidity Premium & Yield Measures**

| Formula | Description |
|---|---|
| $\text{IP} = \text{Corp Bond Spread} - \text{CDS Premium}$ | Direct / model-free illiquidity premium estimation |
| $\text{IRR} = \text{RFR} + \text{Net Credit Spread} + \text{IP}$ | IRR decomposition (default-adjusted CFs) |
| $\text{Z-Spread} = \text{Expected Default Costs} + \text{IP}$ | Z-spread decomposition (contractual CFs) |
| $\text{IRR} \approx \text{Z-Spread} - \text{Expected Default Costs}$ | Relationship between IRR and Z-spread |
| $\text{IP}_{\text{SBA}} = \text{IRR}(\text{adj. asset CFs}) - \text{IRR}(\text{liab. CFs at RFR})$ | SBA illiquidity premium definition |

**BEL & Default Adjustment**

| Formula | Description |
|---|---|
| $\text{Adjusted CF}_t = \text{Promised CF}_t \times (1 - \text{CoD}_{R,K})$ | BMA default haircut applied to cash flows |
| $\text{BEL}_{\text{SBA}} = \max_{s} (\lambda_s^* \cdot \text{MV}(P_{ref}))$ | SBA BEL as maximum scaled asset MV across 9 scenarios |
| $\text{BEL}_{\text{Standard}} = \sum_t L_t / (1 + \text{RFR}_t)^t$ | Standard approach BEL |

**Duration, KRD and DV01**

| Formula | Description |
|---|---|
| $D_{mac} = \frac{\sum_{t=1}^{T} t \cdot \frac{CF_t}{(1+y)^t}}{P}$ | Macaulay Duration — weighted average time of cash flows |
| $D_{mod} = \frac{D_{mac}}{1 + y}$ | Modified Duration — % price change per 1% rate move |
| $\frac{\Delta P}{P} \approx -D_{\text{mod}} \times \Delta y$ | Price sensitivity approximation |
| $\text{DV01} = D_{\text{mod}} \times P \times 0.0001$ | Absolute £ change per 1bp parallel rate move |
| $\text{KRD}_k = -\frac{1}{P} \cdot \frac{\Delta P}{\Delta y_k}$ | Key Rate Duration at maturity point $k$ |
| $\text{KR-DV01}_k = \text{KRD}_k \times P \times 0.0001$ | Absolute £ change per 1bp move at maturity point $k$ |
| $D_{\text{mod}} \approx \sum_{k} \text{KRD}_k$ | Total duration = sum of all KRDs |
| $\text{Total DV01} = \sum_{k} \text{KR-DV01}_k$ | Total DV01 = sum of all KR-DV01s |
| $\text{Net DV01} = \text{DV01}^{\text{Assets}} - \text{DV01}^{\text{Liabilities}}$ | ALM mismatch in absolute £ terms per bp |
| $\text{Duration Gap} = D_{\text{mod}}^{\text{Assets}} - D_{\text{mod}}^{\text{Liabilities}}$ | ALM mismatch in relative terms (SBA tolerance: ±1 year) |

---

*Document compiled from: J.P. Morgan Asset Management "Untangling Complexity — Bermuda SBA" (2024) and supporting mathematical formulations. For personal study reference only.*
