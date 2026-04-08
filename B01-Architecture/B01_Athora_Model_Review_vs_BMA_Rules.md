# Athora SBA Model — Design Review Against BMA Rules & Regulations

> **Date:** 2026-04-08
> **Source documents reviewed:** `B01_Codebase_Architecture_Overview.md`, `B01_ENHANCEMENT_GUIDE_Code_Findings_and_Suggestions.md`
> **Validated against:** `BMA_SBA_Consolidated_Guide_v2.0.md` (verified against BMA Rules Schedule XXV and 2024 Handbook)

---

## 1. Executive Summary

The Athora model (`Athora_BEL_Model_v0.1.8.pyz`) is an **insurer's own SBA portfolio optimisation engine** — it finds the optimal hedge ratio (theta) that minimises BEL for Athora's specific portfolio. It is architecturally sound (modular, unidirectional data flow, typed objects) but its core purpose differs fundamentally from a regulator's benchmark model.

This review identifies **7 areas of concern** against BMA Rules and Handbook, plus **3 documented governance gaps** that the Athora team themselves flagged.

---

## 2. Architecture Overview

### 2.1 What Athora Built

| Component | Description | Files |
|-----------|-------------|-------|
| **loaders/** | Excel → Python typed objects (Settings, Portfolio) | ~4 files, ~1000 lines |
| **instruments/** | Bond/Swap/Liability definitions with pricing logic | ~7+ files, ~1500 lines |
| **projection/** | Core SBA engine — goal-seeking, balance sheet projection, rebalancing | ~6+ files, ~3000+ lines |
| **markets/** | Base curve + 9 scenario x spot curves | ~2-3 files, ~800 lines |
| **valuation/** | DCF pricing, z-spread calibration, QuantLib for callable/floater pricing | ~2 files, ~600 lines |
| **writer/** | Results → Excel (5 sheets: Summary, Assets, Rebalancing, Allocation, Validation) | ~1 file, ~400 lines |

### 2.2 Execution Pipeline

| Phase | Description | Frequency |
|-------|-------------|-----------|
| 1. Load Settings | Excel named ranges → `Settings` dataclass (100+ params) | Once (cached) |
| 2. Initialize Model | Build instruments, market curves (9 scenarios), balance sheet | Once (cached) |
| 3. Calibrate | Z-spread calibration, interest rate model fitting | Once |
| 4. Valuation | Project assets + liabilities, rebalance, goal-seek theta | **50-100 iterations** |
| 5. Write Results | Format to Excel workbook | Once |

### 2.3 What Athora Gets Right

- **Modular separation of concerns** with unidirectional data flow — no circular dependencies.
- **Strongly-typed dataclasses** from Excel input onward — no raw Excel data downstream.
- **QuantLib confined** to the valuation module for callable/floater pricing.
- **9 regulatory scenarios** per Article 28(7) drive the markets module.
- **Caching** of Phases 1-3 for performance (Phase 4 iterates 50-100x).

---

## 3. Regulatory Alignment Issues

### ISSUE-1: Theta Hedge-Ratio Optimisation vs BMA's BEL Definition (HIGH)

**What Athora does:** Goal-seeks "theta" — an optimal hedge ratio (0% to 100%) — to minimise SBA BEL variance across scenarios. The algorithm iterates through hedge ratios, projects the balance sheet under each, and finds the minimum-variance solution.

**What the BMA requires:** Para 28(10) prescribes a straightforward 3-step process:
1. Determine assets required to cover liability cash flows under the base scenario
2. Determine revised assets required under each of the 8 stress scenarios
3. BEL = the highest asset requirement across all 9 scenarios

**The concern:** The BMA does not prescribe or permit optimisation of a hedge ratio within the BEL calculation. The BEL should be determined for a **given** asset portfolio under each scenario — not by optimising the portfolio composition. Theta optimisation is a legitimate tool for an insurer's pre-submission portfolio construction, but embedding it within the BEL calculation itself conflates portfolio optimisation with regulatory valuation.

**Regulatory reference:** Rules Para 28(10)(a)-(c); Consolidated Guide Section 10.2.

---

### ISSUE-2: Active Rebalancing Within Projections (HIGH)

**What Athora does:** At each projection timestep, the model executes:
- KRD (Key Rate Duration) matching — realigns asset durations to liability durations
- TAA (Tactical Asset Allocation) — adjusts portfolio per allocation constraints
- Swap rebalancing — executes interest rate swap trades for hedging

**What the BMA requires:**
- Para 22(4): "Use of management actions **shall not apply** to reinvestment and disinvestment assumptions in the scenario-based approach."
- Para 28(33): Reinvestment must follow the insurer's **declared** asset allocation, board-approved ALM and investment policies — not be dynamically optimised.
- Para 28(34): Disinvestment only to meet excess liability cash flows; must maintain existing allocation within duration limits.

**The concern:** KRD matching and TAA within projections appear to constitute discretionary management actions. The BMA explicitly prohibits management actions in SBA reinvestment/disinvestment. The prescribed reinvestment/disinvestment must follow static, pre-declared rules — not dynamic optimisation at each timestep.

**Regulatory reference:** Rules Para 22(4), Para 28(33)-(34); Consolidated Guide Sections 12-13.

---

### ISSUE-3: Swap Rebalancer as Core Module (MEDIUM)

**What Athora does:** A dedicated `swap_rebalancer.py` module executes swap trades within the SBA projection as part of the rebalancing strategy.

**What the BMA requires:**
- Handbook E8: Derivatives permitted only for **risk mitigation** with BMA pre-approval.
- Handbook E8 introduction: "Dynamic hedging is not allowed in the SBA, namely daily or intra-day hedging."
- Handbook E8.3b: Strategy must explicitly state whether derivatives are buy-and-hold or whether future trading is expected.
- Handbook E8.3e: Residual risks quantified to 1 standard deviation.

**The concern:** If the swap rebalancer is executing new swap trades at each projection timestep based on prevailing conditions, this could constitute dynamic hedging. Permitted use would be modelling pre-approved, static hedge positions that were in place at valuation date. The B01 documentation does not clarify which pattern Athora uses.

**Regulatory reference:** Handbook E8.1-E8.3i; Consolidated Guide Section 20.

---

### ISSUE-4: Scenario Curve Generation — 90 vs 900 Curves (MEDIUM)

**What Athora does:** The markets module generates "9 x 10 = 90 spot curves" (9 scenarios x 10 years) per the B01 documentation.

**What the BMA requires:** Para 28(8) specifies building future spot rate curves at "years 1, 2, 3, etc." — i.e., at every annual projection step. For a 100-year projection, this means 9 scenarios x 100 years = 900 spot curves.

**The concern:** If Athora only generates 10 annual curves per scenario and interpolates/extrapolates for years 11-100, this may not comply with the prescribed methodology. The B01 documentation states both "10 years" and "100 years" without reconciling the discrepancy.

**Possible explanation:** This may be a B01 documentation error — the code may generate curves for all projection years. This needs verification against the actual codebase.

**Regulatory reference:** Rules Para 28(8)(a)-(c); Consolidated Guide Section 8.

---

### ISSUE-5: No Idiosyncratic Spread Adjustment (MEDIUM)

**What Athora documents:** Z-spread calibration in Phase 3, but no mention of the Handbook E9.5 per-asset idiosyncratic spread adjustment.

**What the BMA requires:** Handbook E9.5 mandates that for each asset, a spread adjustment must be determined such that:

> PV(projected asset cash flows, using chosen risk-free rates + applicable spreads + spread adjustment) = actual market value of the asset

This adjustment must be carried through all SBA projections for that specific asset. It ensures the choice of risk-free curve does not distort initial asset market values.

**The concern:** If Athora's z-spread calibration achieves the same economic result (calibrating spreads to match market values), this may be functionally equivalent. But the specific E9.5 requirement — that the adjustment is explicitly determined and carried through projections — is not documented in B01.

**Regulatory reference:** Handbook E9.5; Consolidated Guide Section 9.

---

### ISSUE-6: No Mention of Mandatory Stress Tests (MEDIUM)

**What Athora documents:** The B01 architecture covers the core BEL calculation pipeline (Phases 1-5) but does not reference the 3 mandatory application stress tests.

**What the BMA requires:** Handbook E5.6h mandates three stress tests:
1. Combined mass lapse + credit spread widening (AAA +277bps through CCC/C +2346bps)
2. One-notch credit downgrade of all assets
3. No reinvestment into limited-basis (258E) assets

**The concern:** These tests may exist outside the core BEL engine (e.g., in a separate reporting module not covered by B01). However, they are a regulatory requirement for SBA eligibility and should be part of the documented architecture.

**Regulatory reference:** Handbook E5.6h(i)-(iii); Consolidated Guide Section 19.

---

### ISSUE-7: Audit Trail Limited to Excel (LOW)

**What Athora does:** Results exported to a single Excel workbook with 5 sheets (Summary, Asset Values, Rebalancing, Allocation, Validation).

**What the BMA requires:** Para 28(38): "Documentation shall allow a knowledgeable third party to understand the design and details of the model." Para 28(41)(g)(ix): "Detailed model validation reports" required.

**The concern:** Excel output is adequate for submission to the BMA, but lacks the granularity needed for independent verification. Period-by-period cash flow traces across 9 scenarios x 100 years would be difficult to capture in Excel alone. This is not a regulatory violation but a practical limitation for model governance and challenge purposes.

**Regulatory reference:** Rules Para 28(38), Para 28(41)(g)(ix); Consolidated Guide Sections 22-23.

---

## 4. Governance Gaps (Documented by Athora Team)

These were identified in the B01 documents themselves:

| Gap | Severity | Description | BMA Relevance |
|-----|----------|-------------|---------------|
| **GAP-1** | Medium | No runtime validation of module load order. Phases called out of sequence produce cryptic errors. | Para 28(41)(d)(ii): "Software, computer code, algorithms... shall undergo rigorous quality control." |
| **GAP-2** | Lower | No validation that execution mode matches setup file. | Para 28(40)(i): "Adequate and effective controls in place regarding the model's operation." |
| **GAP-3** | Medium | No pre-flight check that all required Excel sheets exist. Missing sheets produce openpyxl errors, not application-level errors. | Para 28(39): Data policy must ensure "completeness, accuracy, appropriateness" of inputs. |

---

## 5. Lessons for the BMA Benchmark Model

### 5.1 What to Adopt from Athora

| Pattern | Athora Implementation | BMA Benchmark Adaptation |
|---------|----------------------|--------------------------|
| Modular architecture | 6 core modules with single responsibility | Already adopted in Algorithm Specs (curves/, projection/, calculations/) |
| Unidirectional data flow | No circular dependencies | Already adopted |
| Typed data objects | Settings/Portfolio dataclasses | Enhanced with Pydantic v2 (fail-loudly validation) |
| Phase caching | Expensive setup cached, iteration in Phase 4 only | Adopted — curves built once, 9 scenario projections independent |
| QuantLib isolation | Used in valuation module | Tighter boundary — only in `curves/` |

### 5.2 What NOT to Adopt from Athora

| Pattern | Why Not |
|---------|---------|
| **Theta hedge-ratio optimisation** | BMA prescribes a given-portfolio projection, not optimisation. Our model uses C0 root-finding per scenario. |
| **Active rebalancing (KRD/TAA)** | Prohibited as management actions under Para 22(4). Our model uses prescribed reinvestment/disinvestment rules. |
| **Swap rebalancer within projection** | Risk of constituting dynamic hedging. Our model does not execute new derivative trades within projections. |
| **Excel-only audit trail** | Insufficient for regulator's challenge function. Our model uses SQLite + Parquet + JSON Lines. |
| **Global variable caching** | Fragile. Our model uses frozen RunConfig with explicit state machine. |
| **Implicit initialization order** | Source of GAP-1. Our model uses explicit state machine (NotStarted → ... → Complete). |

### 5.3 Open Question — The 35bps Spread Cap

Algorithm Spec ALGO-004 implements a 35bps spread cap and identifies Athora's failure to enforce it as "their single largest implementation gap." However:

- The BMA Rules (Schedule XXV) do not prescribe a spread cap.
- The Handbook (Sections E1-E11) does not mention a spread cap.
- The Consolidated Guide v2.0 (verified against source PDFs) does not include a spread cap.
- The B01 Athora architecture documents do not mention a spread cap.

**The 35bps figure likely originates from the archived Athora/Pythora reference material (PHASE1 documents).** It may be:
- An Athora-specific condition in their BMA approval letter
- An internal Athora risk management parameter
- A now-superseded BMA requirement from pre-2024 rules

**Recommendation:** Before implementing ALGO-004, verify the source and applicability of the 35bps cap. If it cannot be traced to a current BMA regulation, it should be either removed or refactored as an optional company-specific parameter (not a universal regulatory constraint).

---

## 6. Summary

| Category | Count | Items |
|----------|-------|-------|
| **High severity** | 2 | ISSUE-1 (theta optimisation), ISSUE-2 (active rebalancing) |
| **Medium severity** | 4 | ISSUE-3 (swap rebalancer), ISSUE-4 (curve count), ISSUE-5 (idiosyncratic spread), ISSUE-6 (stress tests) |
| **Low severity** | 1 | ISSUE-7 (Excel-only audit trail) |
| **Governance gaps** | 3 | GAP-1 (init order), GAP-2 (mode validation), GAP-3 (config completeness) |

The Athora model is a capable production system with good architecture, but its core design pattern (hedge-ratio optimisation with active rebalancing) is **not aligned with what the BMA Rules prescribe** for the SBA BEL calculation. The BMA benchmark model should learn from Athora's modular structure while implementing the straightforward projection-based approach mandated by Para 28(9)-(10).
