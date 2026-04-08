# BMA SBA Quick Reference Cheat Sheet

> Every entry below is traced to the exact rule paragraph from the **Insurance (Prudential Standards) (Insurance Group Solvency Requirement) Amendment Rules 2024** and the **2024 Year-End Insurance Group Instructions Handbook**.

---

## 1. What Is SBA?

A **cash-flow adequacy test**: project your actual asset portfolio forward under 9 interest rate scenarios and find the one requiring the most starting assets. That amount = BEL.

**Not** a liability discounting method. **Not** stochastic. **Not** model-dependent.

> *Para 28(1); Handbook E1*

---

## 2. The BEL Formula

```
BEL = highest asset requirement across all 9 scenarios
    = MV_assets(t=0) + C0_biting
```

Where **C0** = minimum additional cash so that cash balance >= 0 at every time step under the biting scenario.

> *Para 28(10)(a-c), p. 160*

---

## 3. The 9 Interest Rate Scenarios

| ID | Name | Short Description | Max Shift |
|----|------|-------------------|-----------|
| a | **Base** | No adjustment | 0 bps |
| b | **Decrease** | All rates down | **-150 bps** by Y10 |
| c | **Increase** | All rates up | **+150 bps** by Y10 |
| d | **Down-Up** | Down then back | -150 bps by Y5, back by Y10 |
| e | **Up-Down** | Up then back | +150 bps by Y5, back by Y10 |
| f | **Decrease + Positive Twist** | Short down more, long down less | Y1: -1.5%, Y10: -1.0%, Y30: -0.5% |
| g | **Decrease + Negative Twist** | Short down less, long down more | Y1: -0.5%, Y10: -1.0%, Y30: -1.5% |
| h | **Increase + Positive Twist** | Short up less, long up more | Y1: +0.5%, Y10: +1.0%, Y30: +1.5% |
| i | **Increase + Negative Twist** | Short up more, long up less | Y1: +1.5%, Y10: +1.0%, Y30: +0.5% |

Shifts are **annual ramps** (not instantaneous shocks). Unchanged after year 10 for parallel; interpolated for twists.

> *Para 28(7)(a-i), pp. 159-160*

---

## 4. Curve Construction

1. Convert initial **spot rates** to **forward rates**
2. Build future spot curves using those forwards (year 1, 2, 3...)
3. Apply scenario shifts from Section 3 above

> *Para 28(8)(a-c), p. 160*

---

## 5. Projection Mechanics (Each Scenario)

At each time step (at least annually):

| Step | What Happens | Rule |
|------|-------------|------|
| 1 | Compare asset cash flows vs liability cash flows | Para 28(9)(b) |
| 2 | **Shortfall?** Sell assets at scenario yields | Para 28(9)(c) |
| 3 | **Excess?** Buy assets at scenario yields per investment guidelines | Para 28(9)(d) |
| 4 | Reduce asset cash flows by D&D costs | Para 28(22-23) |

**No management actions allowed** in reinvestment/disinvestment assumptions.

> *Para 22(4), p. 157 — "Use of management actions shall not apply to reinvestment and disinvestment assumptions in the scenario-based approach."*

---

## 6. Asset Tiers

| Tier | Name | What's Included | Limit | Can Sell? |
|------|------|----------------|-------|-----------|
| **1** | Acceptable | Govt bonds, muni bonds, public corporate bonds, cash (all **IG**) | No limit | Yes |
| **2** | Prior Approval | Private assets, structured securities (MBS/ABS/CLO), mortgages, IG preferred stock | BMA approval required | Yes |
| **3** | Limited Basis | Below-IG versions of Tier 1/2, commercial real estate, credit funds | **10% aggregate, 0.5% single asset** | **NO — cannot be sold to cover shortfalls** |

> *Tier 1: Para 28(13); Tier 2: Para 28(14); Tier 3: Para 28(15-16), pp. 160-161*

---

## 7. Disinvestment Waterfall (When Cash Is Short)

Sell in this order to cover shortfalls:
1. **Cash** (no cost)
2. **Government / Municipal bonds** (Tier 1 — tightest spreads)
3. **IG Corporate bonds** (Tier 1)
4. **Tier 2 assets** (structured, mortgages, etc.)
5. **Tier 3 — BLOCKED** (cannot be sold; Para 28(16)(f))

All sales priced at **scenario-prevailing yields**, not par.

> *Para 28(9)(c), 28(16)(f); Handbook E11*

---

## 8. Default & Downgrade (D&D) Costs

- **Applied by reducing projected asset cash flows** (not as a separate charge)
- Default baseline + uncertainty margin (downgrade)
- Company assumptions **cannot be lower** than BMA published tables
- **Phase-in** (for business in force as of 31 Dec 2023):

| Year | Default | Downgrade |
|------|---------|-----------|
| 2024 | 100% | **20%** |
| 2025 | 100% | 40% |
| 2026 | 100% | 60% |
| 2027 | 100% | 80% |
| 2028+ | 100% | **100%** |

New business post-2023 uses 100% from inception.

**Additional D&D mechanics (added 2026-04-08):**
- D&D costs do **NOT affect initial t=0 market values** or reinvestment purchase prices — only projected future CFs (E10.5)
- BMA provides annualised costs → convert to **cumulative loss rates** (and marginal); same rate for all CFs within a period (E10.3)
- **Marginal loss rate must be non-negative** (cumulative non-decreasing); floor at zero with knock-on adjustment (E10.6)
- **Issuer ratings** should be used (not issue ratings), unless demonstrated no less conservative (E10.12-14)
- **Government debt exemption:** AA- or better in local currency = zero D&D; also A- or better with reserve-currency status and full fiscal/monetary independence (E10.19)
- **Floors** for non-BMA-published assets = corporate senior unsecured for corresponding rating; structured assets at tranche level (E10.16)
- Beyond published tenors: costs kept constant at last published values (implies increasing cumulative losses) (E10.17)

> *Para 28(22-26), pp. 162-163; Handbook E10, pp. 331-334*

---

## 9. Biting Scenario

The scenario producing the **highest aggregate asset requirement**. For fungible liability blocks, one biting scenario applies. Non-fungible blocks must be tested separately.

> *Para 28(11), p. 160*

---

## 9a. Idiosyncratic Spread Adjustment (added 2026-04-08)

For each asset, a spread adjustment is determined such that:

```
PV(projected CFs, RF + applicable spreads + adjustment) = actual market value
```

This ensures the choice of risk-free curve does **not distort initial asset market values**. The adjustment is carried through all SBA projections for that specific asset. Required whenever the market benchmark curve underlying the asset's value differs from the SBA risk-free curve.

> *Handbook E9.5*

---

## 10. SBA Spread & 35bps Cap

After computing BEL, the **implied SBA spread** S is defined as:

```
PV(Liability CFs, RF + S) = BEL
```

If S > 35 bps, the spread is **capped at 35 bps** and BEL is increased accordingly. This prevents insurers from using overly optimistic discount rates.

> **Verification note (added 2026-04-08):** The 35bps spread cap is referenced in the Athora/Pythora prior-art implementation and the BMA Illustrative Calculation document, but **cannot be traced to a specific paragraph in the 2024 BMA Rules or Handbook**. It may originate from a company-specific BMA approval condition or a pre-2024 regulatory requirement. Before relying on this as a universal rule, verify with the BMA directly.

> *Handbook E9, pp. 330-331*

---

## 11. Risk Margin

```
RM = CoC × sum[ ModifiedECR(t) / (1 + r_{t+1})^{t+1} ]
```

CoC rate = **6%** (prescribed by BMA).

**Technical Provision = BEL + Risk Margin**

> *Para 36(4), pp. 179-180; Handbook D38.5, p. 314*

---

## 12. Liquidity Coverage Ratio (LCR)

```
LCR = Eligible Liquid Assets / Surrender Outflows >= 105%
```

Tier 3 assets are **not eligible** for LCR.

> *Para 29(2)(iii), p. 175; Handbook D22, p. 253*

---

## 13. Key Stress Tests (Application Package)

| Stress Test | What It Does | Ref |
|-------------|-------------|-----|
| Combined credit spread + mass lapse | Instantaneous spread widening (AAA: +277bps ... CCC: +2346bps) + mass lapse (higher of 20% or BSCR shock) | Handbook E5.6h(i) |
| One-notch downgrade | Downgrade ALL assets by one notch | Handbook E5.6h(ii) |
| No reinvestment into limited-basis | Cannot reinvest into Tier 3 assets | Handbook E5.6h(iii) |

> *Handbook E5.6h, p. 320*

---

## 14. Governance Requirements

| Requirement | Who | Ref |
|------------|-----|-----|
| Board approval of SBA use and major model changes | Board | Para 40(a) |
| Annual attestation on model adequacy | CRO, CIO, Chief Actuary, CEO | Handbook E2 |
| Model change policy (major vs minor) | Board | Handbook E3; Para 40(e) |
| Independent model validation | External | Every 3 years; Para 41 |
| 105% LCR maintenance | Ongoing | Para 29(2)(iii) |

---

## 15. What SBA Is NOT

| Common Misconception | Reality |
|---------------------|---------|
| BEL = PV of liabilities + buffer | BEL = total assets needed (asset adequacy measure) |
| Scenarios are instantaneous shocks | Scenarios are **annual ramps** over 10 years |
| Active portfolio rebalancing is required | Management actions are **prohibited** in projections |
| Tier 3 cap is 35% | Tier 3 cap is **10%** (0.5% per asset) |
| Any asset can be sold in a shortfall | Tier 3 assets **cannot be sold** |
| D&D is fully loaded from day one | Downgrade component phases in 20%→100% over 2024-2028 |

---

*Last verified: 2026-03-25 against BMA Rules PDF (Schedule XXV, Para 28) and Handbook (Sections E1-E11).*
*Updated: 2026-04-08 — added E9.5 idiosyncratic spread, E10.5-E10.19 D&D detail, 35bps cap verification note.*
