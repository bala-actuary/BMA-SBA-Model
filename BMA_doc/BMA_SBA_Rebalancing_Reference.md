# BMA SBA — ALM Rebalancing Mechanics: An Educational Reference for BMA Actuaries

**Document:** `BMA_SBA_Rebalancing_Reference.md`
**Purpose:** Educational reference — how insurers implement KRD matching and TAA rebalancing in their SBA models, and how BMA actuaries can challenge such submissions
**Audience:** BMA actuaries reviewing company SBA submissions
**Related Document:** `BMA_SBA_Illustrative_Calculation_Comprehensive.md` — the primary BMA benchmark illustrative calculation
**Date:** March 2026
**Version:** 1.0

---

> **Important Framing**
>
> The **BMA SBA Benchmark Model** does **NOT** implement KRD rebalancing or TAA rebalancing. It uses pure cash-flow adequacy — the projection simply tests whether assets generate sufficient cash flows to cover liabilities, period by period, without any dynamic trading between periods.
>
> This document explains what **some insurers** do in *their* SBA models when they incorporate dynamic portfolio management. Understanding their approach equips BMA actuaries to ask the right questions and challenge submissions where rebalancing assumptions are unrealistic or incorrectly specified.

---

## Contents

1. [What Is Rebalancing and Why Do Insurers Do It?](#section-1)
2. [Key Rate Duration (KRD) — What It Measures](#section-2)
3. [KRD Matching via Interest Rate Swaps](#section-3)
4. [TAA Physical Rebalancing](#section-4)
5. [Iteration: Up to Three Passes Per Period](#section-5)
6. [Why the BMA Benchmark Model Does NOT Implement KRD Rebalancing](#section-6)
7. [Red Flags When Reviewing Company Submissions](#section-7)

---

## Section 1: What Is Rebalancing and Why Do Insurers Do It? {#section-1}

### 1.1 The ALM Problem in an SBA Context

Under the BMA SBA framework, an insurer must demonstrate that its asset portfolio can cover liability cash flows across all nine prescribed interest rate scenarios. When an insurer holds a static, un-rebalanced portfolio:

- Rising rates: bond prices fall → forced-sale proceeds are lower → cash shortfalls emerge earlier
- Falling rates: coupon maturities are reinvested at lower yields → income declines → the portfolio eventually cannot meet outflows
- Rate twists: short-end vs long-end movements cause duration mismatches → the portfolio is no longer hedged across the liability runoff

Without rebalancing, even a well-constructed initial portfolio drifts away from ALM alignment as each projection year passes. This drift tends to **increase C0** (the minimum starting cash buffer), which in turn **increases the SBA BEL**.

### 1.2 The Insurer's Motivation

Insurers rebalance their modelled portfolios within the SBA projection to reflect what they claim to do in practice: actively manage their ALM position each year. If the SBA model incorporates this active management, the projection produces a lower C0 — and therefore a lower BEL and lower required capital.

This creates a direct incentive to include rebalancing in the SBA model, even if the assumed rebalancing is optimistic (e.g., no transaction costs, perfect execution, unrestricted liquidity).

### 1.3 Two Types of Rebalancing

Insurers typically implement two complementary rebalancing mechanisms:

| Type | Instrument | Purpose |
|---|---|---|
| **KRD Matching** (derivative) | Interest rate swaps | Align the interest rate sensitivity (duration profile) of assets with liabilities at each maturity bucket |
| **TAA Rebalancing** (physical) | Buy/sell bonds | Restore portfolio asset class weights to board-approved strategic targets |

These two mechanisms interact: physical rebalancing changes the portfolio composition, which changes KRDs, which triggers another round of swap rebalancing. Insurers therefore run them in sequence — up to three passes per projection period — until convergence.

**Note for BMA:** When a company states their SBA model incorporates "dynamic ALM management" or "active duration matching," they almost certainly mean some combination of these two mechanisms. The sections below explain each in detail.

---

## Section 2: Key Rate Duration (KRD) — What It Measures {#section-2}

### 2.1 Definition

Key Rate Duration (KRD) at tenor *k* is the sensitivity of portfolio market value to a 1 basis point (0.01%) shock applied **only at that specific tenor**, with all other tenors held constant.

```
KRD(k) = [MV(RF_curve with +1bp at tenor k) - MV(RF_curve with -1bp at tenor k)] / (2 × 0.0001)
```

Equivalently, KRD(k) is the dollar change in portfolio MV per 1bp parallel shock at tenor k. It is also written as **DV01** (Dollar Value of a Basis Point) at each key rate.

### 2.2 Key Rate Tenors

A typical insurer SBA model uses **5 key rate tenors** for derivative rebalancing:

| Tenor | Rationale |
|---|---|
| 5 years | Captures mid-duration exposure |
| 10 years | Core liability duration bucket |
| 15 years | Long-duration annuity runoff |
| 20 years | Extended liability tail |
| 30 years | Ultra-long tail (whole-of-life, deferred annuities) |

Some models use up to 9 tenors (adding 1Y, 2Y, 3Y, 7Y) for more granular matching, particularly when the liability cash flow profile has irregular timing.

### 2.3 Asset KRD vs. Liability KRD — A Critical Distinction

This is the most important technical point in this document. Asset and liability KRDs use **different discount curves**:

| Component | Discount Curve | Reason |
|---|---|---|
| **Asset KRD** | Risk-free rate + asset z-spread | Reflects actual bond pricing — each bond discounted at its market yield |
| **Liability KRD** | **Risk-free rate ONLY — no spread** | Hedging target must be set on a spread-neutral basis |

**Why liability KRD uses risk-free only:**

The insurer uses swaps to hedge the interest rate sensitivity of liabilities. The cost of the swap reflects the risk-free (swap) curve, not the SBA spread. If liability KRD were computed at risk-free + SBA spread, the hedge ratio would be sized against the wrong target — the swaps would be under- or over-sized, creating residual unhedged exposure.

This is not a minor implementation detail. PHASE2-004 (the Spec vs. Implementation reconciliation report for Athora's model) flagged this as a **Critical Gap** — the original reference documentation did not specify that liability KRDs must use risk-free only, creating risk that developers would implement it incorrectly.

> **BMA Challenge Point:** Ask the company to confirm explicitly that liability KRDs in their SBA model are computed at the risk-free rate only, with z-spread = 0 on the liability side.

### 2.4 Illustrative KRD Calculation

Using the extended portfolio from `BMA_SBA_Illustrative_Calculation_Comprehensive.md`:

**Asset KRDs** (base scenario, RF = flat 3%):

| Asset | Tenor 5Y KRD | Tenor 7Y KRD | Tenor 10Y KRD |
|---|---|---|---|
| Govt Bond (10Y, 3.2% yield) | $0 | $0 | $272 |
| IG Corp (7Y, 4.8% yield) | $0 | $185 | $0 |
| HY Corp (5Y, 7.5% yield) | $43 | $0 | $0 |
| **Total Asset KRD** | **$43** | **$185** | **$272** |

**Liability KRDs** (10-year $800/year annuity, discounted at RF = 3%):

| Tenor | Liability KRD |
|---|---|
| 5Y | $140 |
| 7Y | $160 |
| 10Y | $195 |

**KRD Delta** (asset minus liability — the swap target):

| Tenor | Asset KRD | Liability KRD | Delta | Swap Required |
|---|---|---|---|---|
| 5Y | $43 | $140 | **-$97** | PAYER swap (increase asset sensitivity) |
| 7Y | $185 | $160 | **+$25** | RECEIVER swap (reduce asset sensitivity) |
| 10Y | $272 | $195 | **+$77** | RECEIVER swap (reduce asset sensitivity) |

A positive delta means assets are *more* rate-sensitive than liabilities at that tenor — the portfolio is over-hedged; the insurer enters a RECEIVER swap to reduce sensitivity. A negative delta means under-hedged — the insurer enters a PAYER swap to increase sensitivity.

---

## Section 3: KRD Matching via Interest Rate Swaps {#section-3}

### 3.1 Swap Types

Two types of plain-vanilla interest rate swaps are used:

| Swap Type | Fixed Leg | Floating Leg | Effect on Portfolio Duration |
|---|---|---|---|
| **PAYER swap** (pay fixed) | Insurer pays fixed rate | Insurer receives floating (IBOR/SOFR) | **Increases** duration at that tenor (adds asset-like sensitivity) |
| **RECEIVER swap** (receive fixed) | Insurer receives fixed rate | Insurer pays floating | **Decreases** duration at that tenor (offsets excess asset sensitivity) |

### 3.2 Rebalancing Entry Algorithm

The rebalancing proceeds **tenor by tenor, longest maturity first** (to minimize interaction effects between buckets):

```
FOR each tenor k in [30Y, 20Y, 15Y, 10Y, 7Y, 5Y, 3Y, 2Y, 1Y]:

    delta(k) = Asset_KRD(k) - Liability_KRD(k)

    IF delta(k) > tolerance:
        # Asset over-hedged at this tenor — RECEIVE fixed
        → Enter RECEIVER swap, notional sized to reduce KRD(k) by delta(k)
        → Net effect: portfolio KRD(k) ≈ Liability_KRD(k)

    ELIF delta(k) < -tolerance:
        # Asset under-hedged at this tenor — PAY fixed
        → Enter PAYER swap, notional sized to increase KRD(k) by |delta(k)|
        → Net effect: portfolio KRD(k) ≈ Liability_KRD(k)

    ELSE:
        # Within tolerance — no swap required at this tenor
        PASS
```

**Swap notional sizing:**

The notional amount N of the swap at tenor k is determined by:

```
N = delta(k) / DV01_per_unit(k)
```

where `DV01_per_unit(k)` is the DV01 of a unit notional swap at tenor k (approximately equals the swap duration × 0.0001 × notional).

### 3.3 Swap Cashflow Treatment Within the Projection

Each period, the swap generates cashflows that must be modelled within the SBA projection:

- **RECEIVER swap:** Insurer receives fixed coupon (cash inflow), pays floating rate (cash outflow)
- **PAYER swap:** Insurer pays fixed coupon (cash outflow), receives floating rate (cash inflow)

These cashflows occur **before** the portfolio valuation step within each projection period. They affect the period cash balance directly — a large PAYER swap position creates ongoing fixed-rate outflows that reduce available cash for liability payments.

**Collateral:**

IR swaps require cash collateral (initial and variation margin). In a well-specified model:
- Collateral outflows reduce the available cash pool
- Collateral earns the overnight rate (close to, but not identical to, the scenario risk-free rate)
- Collateral posted must be modelled explicitly — not netted or ignored

> **BMA Challenge Point:** Ask the company whether swap collateral requirements are modelled explicitly in the projection. If collateral is treated as off-balance-sheet or simply ignored, the projection understates cash outflows and overstates the portfolio's cash-generating ability — lowering C0 artificially.

### 3.4 Transaction Costs on Swaps

Swap entry carries a bid-offer spread, typically **2–10bps on the swap notional** depending on maturity and market conditions. This cost is typically capitalised (deducted from initial portfolio value) rather than spread over the life of the swap.

Transaction costs on swaps:
- Are proportional to the frequency of rebalancing (more rebalancing → more costs)
- Increase with the size of the KRD mismatch (larger deltas → larger notionals → higher costs)
- Are not zero — a company claiming zero swap transaction costs is almost certainly wrong

**Example from the extended portfolio:**

For the 5Y PAYER swap (delta = -$97):
- Assume DV01_per_unit at 5Y = $0.00450 per unit notional
- Required notional N = $97 / $0.00450 = approximately $21,600
- Bid-offer cost at 5bps: $21,600 × 0.0005 = $10.80

Over a 10-year projection with annual rebalancing, transaction costs on swaps can accumulate to a material amount relative to the C0 buffer.

### 3.5 BMA Scenario Interaction

A subtlety: KRD rebalancing is computed using the **current scenario curve** for that projection period, not the base curve. In a rising-rate scenario, asset KRDs shrink (bond prices fall, duration shortens slightly), while liability KRDs also adjust. The required swap position therefore changes each period — even if the physical portfolio is unchanged.

> **BMA Challenge Point:** Does the company's model recompute KRD positions at each period's scenario curve, or does it use a static base-curve KRD computed once at t=0? Using t=0 KRDs throughout the projection is incorrect and will understate rebalancing costs.

---

## Section 4: TAA Physical Rebalancing {#section-4}

### 4.1 The Allocation Problem

Over time, market movements and cash flows cause the portfolio's actual asset class weights to drift from the board-approved **Strategic Asset Allocation (SAA)**. Without rebalancing, the portfolio may become concentrated in long-duration assets (bond prices rose) or depleted of a specific credit tier (bonds matured).

TAA (Tactical Asset Allocation) rebalancing restores the portfolio to target weights each period. Unlike KRD matching (which uses derivatives), TAA rebalancing involves **physical trades** — selling assets that are overweight, buying assets that are underweight.

### 4.2 The Two-Phase Algorithm

**Phase 1 — SELL overweight positions:**

```
FOR each asset class i:
    drift(i) = actual_weight(i) - target_weight(i)
    IF drift(i) > tolerance:
        → SELL proportional to drift(i)
        → Proceeds deposited to cash
        → Apply transaction cost (bid-ask spread on sale)
```

Transaction costs on physical sales are typically **5–25bps** depending on the asset type (sovereigns at the low end, high yield or private assets at the high end).

**Phase 2 — BUY underweight positions:**

Once sale proceeds are available as cash, the model must decide how to reinvest. This is an optimisation problem:

```
Minimize:    Σ (target_rate(i) - reinvestment_rate(i))²
Subject to:  Σ trade(i) = 0                            (cash-neutral)
             duration_limit: duration(new_bond) ≤ 1.30 × liability_duration
             tier_constraint: Tier3_allocation ≤ 10% of total
             bounds(i).lb ≤ trade(i) ≤ bounds(i).ub    (allocation floor/ceiling)
```

Solver: **SLSQP** (Sequential Least Squares Programming), a gradient-based constrained optimiser available in `scipy.optimize.minimize`.

The optimiser selects the maturity, rating, and spread of newly purchased "synthetic bonds" that best match the target allocation while respecting regulatory constraints and the reinvestment yield environment.

**Key constraint — Tier 3 assets cannot be sold:**

Assets classified as Tier 3 (high yield, below-investment-grade, certain real assets) must be treated as **hold-only** in the disinvestment waterfall. They cannot appear as SELL candidates in Phase 1. This is a BMA regulatory requirement — see Schedule XXV and the CLAUDE.md domain context.

If a company's TAA model allows Tier 3 assets to be sold during rebalancing, this is non-compliant.

### 4.3 Allocation Strategies

Companies may use different reference points for the target allocation:

| Strategy | Reference | Description |
|---|---|---|
| **SAA (Strategic Asset Allocation)** | Board-approved static targets | Fixed percentage targets by asset class — most common |
| **CAA (Current Asset Allocation)** | Market-weighted at each period | Dynamic — target shifts with market values |
| **Most Onerous** | Conservative across SAA/CAA | Uses the binding combination — most conservative approach |

> **BMA Challenge Point:** Which allocation strategy does the company use? If they use CAA (market-weighted), the "target" drifts with asset prices, potentially allowing allocation to concentrated positions that would be rejected under a static SAA. Ask to see the SAA on file with the board and verify it matches the SAA loaded in the SBA model.

### 4.4 Worked Example

**Setup:** Extended portfolio at end of Year 1, after coupons received.

| Asset | MV (Start) | MV (After Y1 Return) | Target Weight | Actual Weight | Drift |
|---|---|---|---|---|---|
| Govt Bond | $2,983 | $3,073 | 54.1% | 55.5% | +1.4% |
| IG Corp | $2,023 | $2,050 | 36.8% | 37.0% | +0.2% |
| HY Corp | $490 | $494 | 8.9% | 8.9% | +0.0% |
| Cash (C0) | $2,893 | $2,893 | — | — | — |

*In this simple base-case example, drift is small. In a falling-rate scenario (Scenario B), HY Corp appreciates more rapidly, causing larger drift toward Tier 3.*

**Phase 1 — SELL:**
- Govt Bond is overweight by 1.4% → SELL $77 of Govt Bond (MV-weighted)
- Transaction cost: 5bps on $77 = $0.04
- Proceeds to cash: $76.96

**Phase 2 — BUY:**
- IG Corp is underweight by 0.2% → BUY $76.96 of new IG Corp 5Y bond at scenario reinvestment spread
- Scenario base rate at 5Y = 3.0%; reinvestment IG spread = 150bps → new bond yield = 4.5%
- Transaction cost on purchase: 5bps on $76.96 = $0.04
- Net cash impact: $76.96 received - $76.96 spent = $0 (cash-neutral)
- Net transaction costs: $0.08 (reduces cash balance directly)

**Duration check:**
- Liability duration at Year 1 ≈ 5.2 years (remaining 9-year annuity discounted at 3%)
- Duration limit: 1.30 × 5.2 = 6.76 years
- New IG Corp 5Y bond: duration ≈ 4.5 years → PASS

### 4.5 Reinvestment Spread Constraints

The yield at which new bonds can be purchased within the model is constrained:

- Must be consistent with the credit quality of the asset class being purchased
- Must not exceed the **35bps SBA spread cap** for any single reinvestment (though this is applied at the portfolio level, not per-bond level, in most implementations)
- Must reflect realistic market conditions in the scenario (not the base curve in a rising-rate scenario)

---

## Section 5: Iteration — Up to Three Passes Per Period {#section-5}

### 5.1 Why Iteration Is Necessary

KRD rebalancing and TAA rebalancing interact:
- Entering swap positions changes the portfolio's effective KRD profile
- TAA physical trades change the portfolio composition → change asset KRDs
- Changed asset KRDs require revised swap positions

A single pass of "KRD rebalance, then TAA rebalance" leaves residual mismatches. Sophisticated models therefore iterate until convergence.

### 5.2 The Three-Pass Sequence

```
REPEAT up to 3 times:

    PASS 1 — Derivative Rebalancing (KRD Matching):
        → Compute current Asset KRD and Liability KRD (at risk-free)
        → Compute delta at each tenor
        → Enter/adjust IR swaps to close KRD deltas
        → Deduct swap transaction costs

    PASS 2 — Physical TAA Rebalancing:
        → Compute current allocation vs. SAA targets
        → Execute SELL (overweight) → BUY (underweight) trades
        → Deduct physical transaction costs
        → Note: TAA trades change asset mix → change asset KRDs

    PASS 3 — Derivative Rebalancing (Residual Correction):
        → Recompute Asset KRD post-TAA
        → Close any residual KRD delta introduced by Phase 2 TAA trades
        → Deduct swap transaction costs

    CHECK CONVERGENCE:
        IF max(|KRD_delta(k)|) < tolerance AND max(|allocation_drift(i)|) < tolerance:
            → EXIT loop (converged)
        ELSE:
            → Continue to next pass (or exit after 3rd pass regardless)
```

### 5.3 Cost Implications of Three-Pass Iteration

Each pass incurs transaction costs. In a heavily rebalanced portfolio:
- 3 passes per period × 10 periods = 30 sets of transaction costs over the projection
- Swap costs: proportional to notional rebalanced
- Physical costs: proportional to nominal traded (5–25bps)

**Example accumulation:** If swap costs average $10/period and physical costs average $8/period, total transaction costs over 10 years ≈ 3 passes × ($10 + $8) × 10 years = **$540**.

This is material relative to a C0 of $2,893. A model that sets transaction costs to zero overstates the portfolio's cash-generating ability and understates the true BEL.

> **BMA Challenge Point:** Ask the company to show the sensitivity of C0 to transaction cost assumptions. If setting transaction costs to zero reduces C0 by more than 2–5%, the transaction cost assumption deserves scrutiny.

### 5.4 Convergence Tolerances

Typical convergence thresholds:
- KRD delta tolerance: < $1 per tenor (i.e., residual duration mismatch < 1bp DV01)
- Allocation drift tolerance: < 0.5% per asset class

If the model exits after three passes without reaching convergence — a "partial rebalance" — the residual mismatch is absorbed as basis risk. Larger basis risk means the portfolio is not fully duration-matched, which increases vulnerability to rate moves in subsequent periods.

---

## Section 6: Why the BMA Benchmark Model Does NOT Implement KRD Rebalancing {#section-6}

### 6.1 Design Intent

The BMA SBA Benchmark Model is a **regulator's independent verification tool**, not an insurer's capital optimisation model. These different purposes lead to deliberately different designs:

| Consideration | Insurer's SBA Model | BMA Benchmark Model |
|---|---|---|
| **Primary purpose** | Minimise regulatory capital (optimise BEL) | Independent cash-flow adequacy verification |
| **Rebalancing** | Dynamic KRD + TAA each period | None — static disinvestment waterfall + reinvestment strategy (ALGO-002) |
| **Transparency** | Model-dependent; hard to audit externally | Fully traceable — every calculation tied to a BMA rule via `@rule_ref` |
| **C0 direction** | Rebalancing *reduces* C0 (lowers capital) | No rebalancing benefit; conservative baseline |
| **Auditability** | Swap positions, SLSQP optimiser are black-box | Pure Python projection; no optimisation black boxes |
| **Reproducibility** | Depends on SAA file, optimiser convergence, costs | Deterministic; same inputs → same output every time |

### 6.2 The Regulator's Perspective on Rebalancing

From BMA's perspective, a company's claimed "dynamic rebalancing" in their SBA model is an **assumption that must be verified**, not a given. The key questions are:

1. Does the board-approved SAA match the SAA used in the SBA model?
2. Are transaction costs realistic?
3. Are Tier 3 assets correctly excluded from sell decisions?
4. Is liability KRD computed at risk-free only?
5. Is the rebalancing strategy actually executable under stressed market conditions?

The BMA benchmark model produces a C0 **without any rebalancing benefit**. When a company's C0 is materially lower than the benchmark C0, the difference is (partly) attributable to their rebalancing assumption. This gap is a natural starting point for challenge.

### 6.3 When Rebalancing Assumptions Are Legitimate

BMA does not prohibit companies from including rebalancing in their SBA models. If a company can demonstrate:
- The strategy is documented in a board-approved ALM policy
- Transaction costs are calibrated to actual recent trading evidence
- The strategy is executable under all nine BMA scenarios (including stressed market conditions)
- Tier 3 assets are excluded from all sell decisions
- Liability KRDs are computed at risk-free only

...then the rebalancing assumption may be acceptable. The BMA benchmark C0 provides the **floor**: any company C0 materially below this level requires justification.

### 6.4 Summary: Benchmark C0 vs. Company C0

```
BMA Benchmark C0  =  Conservative baseline (no rebalancing, deterministic waterfall)
                  ≥  (usually)
Company C0        =  Lower (rebalancing reduces required buffer)

Difference        ≈  Rebalancing benefit + model assumption differences
```

The BMA actuary's job is to determine whether the company's rebalancing benefit is **earned** (realistic assumptions, verifiable evidence) or **manufactured** (zero costs, unrestricted Tier 3 trades, wrong liability KRD curve).

---

## Section 7: Red Flags When Reviewing Company Submissions {#section-7}

The following table provides a practical checklist for BMA actuaries reviewing company SBA submissions that include rebalancing.

### 7.1 KRD and Swap Rebalancing

| Red Flag | What to Look For | Challenge Question |
|---|---|---|
| **Wrong liability KRD discount rate** | Liability KRDs computed at risk-free + SBA spread | "Confirm that liability KRDs are computed at the risk-free rate only, with z-spread = 0 on the liability side." |
| **Static KRD rebalancing** | KRD computed once at t=0 and held constant | "Are swap positions re-optimised at each projection period using the period's scenario curve, or are t=0 KRDs used throughout?" |
| **Zero swap transaction costs** | Swap costs set to zero or not disclosed | "What bid-offer spreads were applied to swap entries? Provide evidence from recent executed trades." |
| **Collateral ignored** | No mention of variation margin or collateral cash flows | "Are swap collateral requirements (initial and variation margin) modelled explicitly as cash outflows in the projection?" |
| **Wrong swap direction** | PAYER entered when delta is positive (should be RECEIVER) | "For each tenor with a positive KRD delta, confirm the model enters a RECEIVER (not PAYER) swap." |

### 7.2 TAA Physical Rebalancing

| Red Flag | What to Look For | Challenge Question |
|---|---|---|
| **Tier 3 assets in sell waterfall** | HY, below-IG, or illiquid assets sold during rebalancing | "How are Tier 3 assets treated in the TAA sell phase? Confirm they are excluded from sale decisions." |
| **SAA mismatch** | SAA in model differs from board-approved SAA on file | "Provide the board-approved SAA and confirm it is the exact file used in the SBA model (file hash or version control reference)." |
| **Zero physical transaction costs** | No bid-ask spread or explicit cost assumption | "What bid-ask spreads were assumed for physical trades? For IG Corp bonds, industry norms are 5–15bps." |
| **Unrealistic reinvestment yields** | New bonds purchased at yields inconsistent with scenario | "In Scenario B (rates down), what yield is assumed for new IG Corp purchases in Year 5?" |
| **Duration limit breach** | New bonds have duration > 1.30 × liability duration | "What is the maximum duration assumed for reinvestment assets? Is a 1.30× liability duration cap applied?" |

### 7.3 Iteration and Convergence

| Red Flag | What to Look For | Challenge Question |
|---|---|---|
| **Only one rebalancing pass** | Model runs KRD once, TAA once — no iteration | "How many rebalancing iteration passes are run per projection period? Show the convergence check logic." |
| **Non-convergence not flagged** | Model runs 3 passes and exits silently even if not converged | "If convergence tolerance is not met after 3 passes, how is this logged and what is its impact on C0?" |

### 7.4 Capital Sensitivity

| Red Flag | What to Look For | Challenge Question |
|---|---|---|
| **Large gap vs. BMA benchmark** | Company C0 is >10% below benchmark C0 | "Provide the C0 with and without rebalancing. Quantify the rebalancing benefit and show sensitivity to transaction cost assumptions." |
| **35bps spread cap not applied** | SBA spread > 35bps in any scenario | "Confirm the 35bps spread cap was applied in all nine scenarios. Show the uncapped and capped spreads." |
| **Best-case scenario assumption** | Rebalancing modelled only in favourable scenarios | "Does the model apply rebalancing in all nine scenarios, including rate-down (Scenario B) which is typically the biting scenario?" |

---

## Appendix: Source References

This document draws on prior-art specifications from the Athora SBA model (v1.0.0), now superseded by BMA's own Algorithm_Specs. The following files were used as primary sources:

| Source | Key Content Used |
|---|---|
| `Reference_Documents/PHASE1-005-Rebalancing-Allocation.md` | TAA algorithm (two-phase SELL/BUY), SLSQP optimiser, SAA/CAA/Most-Onerous strategies, duration limit (1.30×), Tier 3 exclusion, convergence structure |
| `Reference_Documents/IMPLEMENTATION-SPECIFICATION-SBA-BEL-Developer.md` | Algorithm 3 (derivative rebalancing), KRD matching algorithm, RECEIVER/PAYER swap entry logic, liability KRD = risk-free only, 5 key rate tenors (5Y, 10Y, 15Y, 20Y, 30Y), 3-pass iteration, 35bps cap not enforced (flagged) |
| `Reference_Documents/PHASE2-004-Spec-vs-Implementation-Report.md` | Critical gap: liability KRD discount rate not documented; callable bonds not validated; D&D validation missing; spread cap mechanism absent from reference docs |

**BMA regulatory basis:**

- Schedule XXV, Para 28(7)(a-i) — 9 prescribed interest rate scenarios
- Schedule XXV, Para 28(10) — Tier 3 constraints (10% cap, hold-only)
- `BMA_doc/BMA_SBA_Consolidated_Guide.md` — authoritative rule reference

**BMA benchmark model specifications:**

- `Algorithm_Specs/ALGO-002_Projection_Engine.md` — pure cash-flow adequacy (no KRD rebalancing)
- `Algorithm_Specs/ALGO-004_Spread_Cap_Enforcement.md` — 35bps cap, enforced from day one
- `CLAUDE.md` — project overview and design constraints

---

*End of Document*
