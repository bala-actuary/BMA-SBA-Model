# Gap Analysis: Existing Illustrative Calculation vs. Projectwide Documentation

**Date**: March 25, 2026  
**Assessment Scope**: BMA_SBA_Illustrative_Calculation.md vs. 30-document project package  
**Purpose**: Identify gaps and propose comprehensive enhancement plan

---

## Executive Summary

**Current Document Status**: ✅ Good pedagogical foundation (5-year simple example + 10-year extended example)

**Completeness Score**: 45% of comprehensive needs

**Critical Gaps**: 
- 🔴 Missing actual 9 BMA scenarios (only 5 simplified)
- 🔴 Missing KRD/rebalancing mechanics (core model feature)
- 🔴 Missing Z-spread calibration & mean reversion
- 🔴 Missing goal-seek algorithm detail (hybrid linear/bisection)
- 🔴 Missing spread cap calculation & 35bps cap enforcement
- 🟡 D&D formula not detailed (using static rates, not cumulative formula)
- 🟡 No interest rate curve management
- 🟡 No configuration parameters shown
- 🟡 No regulatory mapping to Article 28 requirements
- ⚪ No complex instruments (callables, FX, swaptions)

---

## Detailed Gap Analysis

### ✅ What Your Document Does Well

| Strength | Location | Assessment |
|----------|----------|-----------|
| **Pedagogical clarity** | Sections 1-3 | Clear intro to SBA as solvency test |
| **Simple example** | Sections 2-6 | Good 5-year annuity + bond setup |
| **Cash flow walkthrough** | Tables in sections 4 & 7 | Well-structured period-by-period |
| **Binding scenario concept** | Section 5.2 | Correctly identifies "Rates Down" as biting |
| **Reinvestment risk** | Section 6 | Good insight on how rates drive C0 |
| **Multi-asset example** | Section 7 | Extended with 3 assets, D&D, Tier 3 |
| **Forced asset sales** | Section 7.9 | Excellent illustration of mark-to-market penalties |
| **Key learning points** | Section 7.10 | Insightful summary of portfolio dynamics |

**Score**: 40-45% complete for pedagogical purposes ✅

---

### 🔴 CRITICAL GAPS

#### Gap 1: Missing 9 BMA Scenarios (Only 5 Shown)

**Current State**:
- Your doc shows: Base, Rates Down, Rates Up, Twist-Steepener, Twist-Flattener (5 scenarios)
- All use simplified, flat or linear rate structures
- No evidence they follow BMA Article 28(7) specifications

**What's Missing (from our project documents)**:
Per BMA Article 28(7) & IMPLEMENTATION-SPECIFICATION, the 9 scenarios are:

| # | Scenario | Year 1-10 Shock | Year 11-100 Shock | Your Doc | Project Reference |
|---|----------|---|---|---|---|
| 0 | Base | 0 bps | 0 bps | ✅ Covered | EXECUTIVE-SUMMARY p2-3 |
| 1 | Up 25 | +25 all | +25 all | ❌ Missing | EXECUTIVE-SUMMARY p3 |
| 2 | Down 25 | -25 all | -25 all | ✅ As "Rates Down" | Your section B |
| 3 | Up 25 flipped | +25 then -25 | -25 | ❌ Missing | EXECUTIVE-SUMMARY p3 |
| 4 | Down 25 flipped | -25 then +25 | +25 | ❌ Missing | EXECUTIVE-SUMMARY p3 |
| 5 | Flattening | Front +50, Back -50 | Blend to 0 | ❌ Simplified | Your section D |
| 6 | Steepening | Front -50, Back +50 | Blend to 0 | ❌ Simplified | Your section E |
| 7 | Short up | +50 (0-5Y), +25 (5Y+) | +25 | ❌ Missing | EXECUTIVE-SUMMARY p3 |
| 8 | Long up | -25 (0-5Y), +50 (5Y+) | +50 | ❌ Missing | EXECUTIVE-SUMMARY p3 |

**Impact**: Your illustration covers ~55% of BMA scenarios. A comprehensive version must show all 9.

**How to Fix**:
- Add 4 missing scenarios (Up 25, Up 25 flipped, Down 25 flipped, Short up, Long up)
- Clarify year 1-10 vs. year 11-100 shock levels for each
- Show that shocks converge to steady-state by year 10
- Reference: EXECUTIVE-SUMMARY-SBA-BEL pages 2-3, IMPLEMENTATION-SPECIFICATION page 4

---

#### Gap 2: Missing KRD & Rebalancing Mechanics

**Current State**: Your doc completely ignores rebalancing. Assets are held to maturity; no swaps, no TAA adjustments.

**What's Missing**:

1. **Swap Rebalancing (KRD matching)**:
   - Each period: Calculate Asset KRD vs. Liability KRD
   - Execute swaps to match: Asset KRD - Liability KRD = 0
   - Cost: Implied swap rates differ across scenarios
   - Currently: NOT shown in calculation

2. **Physical Rebalancing (TAA)**:
   - Each period: Rebalance back to Target Asset Allocation
   - Transaction costs: 2-5 bps per trade
   - Currently: NOT shown in calculation

3. **Rebalancing Iterations**:
   - Up to 3 iterations per period (swap, then physical, then swap again)
   - Early exit if convergence achieved
   - Currently: NOT shown in calculation

4. **Impact on C0**:
   - Rebalancing changes the required C0
   - Adds transaction cost drag
   - Affects binding year identification

**How to Fix**:
- Add sub-step after "Revalue assets" in each period: "Rebalance to target KRD (1 iteration shown)"
- Show swap calculation: Asset DV01 = ? | Liability DV01 = ? | Swap needed = ?
- Add TAA rebalancing step with transaction costs
- Add note: "Simplified to 1 rebalancing pass; full model uses up to 3"
- References: 
  - EXECUTIVE-SUMMARY pages 6
  - IMPLEMENTATION-SPECIFICATION pages 3, 5-7
  - PHASE1-005 (detailed)
  - PHASE3-002 tests U16-U20 (KRD tests)

---

#### Gap 3: Missing Z-Spread Calibration & Mean Reversion

**Current State**: Your doc uses static spreads. No discussion of how spreads are calibrated or evolved.

**What's Missing**:

1. **Initial Calibration (t=0)**:
   - Bond yield = Risk-free rate + Z-spread
   - Three calibration methods (market-price goal-seek is default)
   - Currently: NOT shown

2. **Mean Reversion Formula**:
   $$z(t) = L + (z_0 - L) \times \rho^t$$
   - z(t) = spread in year t
   - L = long-term (convergence) spread
   - z₀ = starting spread
   - ρ = mean reversion rate (e.g., 0.95/year)
   - Currently: NOT shown; static spread used instead

3. **Per-Scenario Projection**:
   - Each scenario has different risk-free rates
   - But spreads evolve the SAME way across scenarios
   - This creates cross-scenario spread dynamics
   - Currently: NOT considered

**How to Fix**:
- Add section: "2.4 Z-Spread Calibration & Mean Reversion"
- Show: Initial bond yield = 4.5% = 3.0% (risk-free at t=0) + 1.5% (Z-spread)
- Show: For five-year bond, Z-spread path per mean reversion
- Add column to cash flow tables: "Z-spread in Year Y"
- Reference:
  - EXECUTIVE-SUMMARY pages 6
  - IMPLEMENTATION-SPECIFICATION pages 5-7
  - PHASE1-003 (detailed Z-spread)
  - PHASE3-002 tests U21-U26 (Z-spread tests)

---

#### Gap 4: Missing Goal-Seek Algorithm Detail

**Current State**: Your doc solves C0 backwards "working backward" but doesn't explain the algorithm.

**What's Missing**:

1. **Hybrid Linear + Bisection Algorithm**:
   - Years 1-8 of attempts: Use linear interpolation (fast)
   - If discontinuity detected: Switch to bisection (robust)
   - Attempts 1-20 max before fallback
   - Currently: NOT explained

2. **Discontinuity Detection**:
   - Example: Rebalancing threshold crossed → sharp change in feasibility
   - Linear method may miss this → triggers bisection
   - Currently: NOT shown

3. **Convergence Criteria**:
   - Optimal tolerance: 0.001% (1bp relative error)
   - Absolute tolerance: 0.1% (fallback)
   - Max attempts: 20
   - Currently: Not specified in your doc

4. **Pseudo-Code**:
   - Not shown; only backward calculation shown

**How to Fix**:
- Add sub-section: "Solving for C0 — Goal-Seek Algorithm"
- Show semi-pseudo-code of hybrid linear/bisection approach
- Explain why hybrid is needed (discontinuities from rebalancing)
- Show convergence tolerances
- Note: Your backward calculation is mathematically equivalent to bisection
- Reference:
  - IMPLEMENTATION-SPECIFICATION pages 2-3
  - PHASE1-002 (detailed algorithm)
  - PHASE3-002 tests U6-U10

---

#### Gap 5: Missing SBA Spread Calculation & 35bps Cap

**Current State**: Your doc calculates final BEL as Bond MV + C0, but doesn't calculate SBA spread.

**What's Missing**:

1. **SBA Spread Calculation**:
   ```
   Find spread such that:
   NPV_Liabilities(Risk_Free_Rate + SBA_Spread) = BEL
   ```
   - Currently: Not calculated

2. **35bps Cap Enforcement**:
   ```
   SBA_Spread_Final = min(SBA_Spread_Calculated, 0.0035)
   ```
   - 🔴 **CRITICAL GAP IN PROJECT**: Cap is defined but NOT enforced in code!
   - Your doc should show the cap AND note the gap
   - Currently: Not mentioned

3. **Output Metrics**:
   - Final SBA spread (should be ≤ 35bps)
   - If spread capped, note: "Capped at 35bps per BMA requirement"
   - Currently: Not shown

**How to Fix**:
- Add subsection: "Step 8: Calculate SBA Spread & Apply Regulatory Cap"
- Show iterative solve for spread
- Show: "SBA Spread = 32bps (before cap) → 32bps (after cap, no binding)"
- Or: "SBA Spread = 45bps (before cap) → 35bps (after cap, binding)"
- Note: "⚠️ Implementation Gap: Spread cap currently not enforced in code. Add: `sba_spread = min(sba_spread, 0.0035)`"
- Reference:
  - IMPLEMENTATION-SPECIFICATION pages 6
  - PHASE1-010 (spread cap reference)
  - PHASE2-004 (gap identified)
  - PHASE3-006 (gap documented)

---

### 🟡 MEDIUM PRIORITY GAPS

#### Gap 6: D&D Formula Not Detailed

**Current State**: Your section 7.1.3 shows "$4/year" for IG BBB credit cost, but doesn't explain the cumulative formula.

**What's Wrong**:
- Year 1 D&D = 0.20%, Year 2 = 0.20% → cumulative should be 1-(1-0.002)² = 0.39% (not 0.40%)
- Your doc uses static rates; correct approach uses cumulative: $\text{Cumulative} = 1 - (1 - r_{annual})^t$

**What's Missing**:
- Formula: cumulative = 1 - (1 - annual_rate)^t
- Explanation: Why cumulative matters (exponential decay of survival)
- Example: "Year 5 with 1% annual default → Cumulative = 1-(0.99)⁵ = 4.9%"
- Application to both market values AND cashflows

**How to Fix**:
- Add section: "2.5 Defaults & Downgrades (D&D) Formula"
- Show formula with example
- Explain: "Year Y cumulative D&D = 1 - (1 - annual_dd%)^Y"
- Show in table as: "D&D% applied to coupons & par"
- Reference:
  - EXECUTIVE-SUMMARY pages 6
  - IMPLEMENTATION-SPECIFICATION page 7
  - PHASE1-009 (detailed)
  - PHASE3-002 tests U1-U5

---

#### Gap 7: Interest Rate Curve Management Not Shown

**Current State**: Your scenarios show a single "Reinvestment Rate" per year (e.g., 3.0% flat).

**What's Missing**:
1. **Base curve**: Euribor 6M swap curve (market standard)
2. **Curve structure**: 15 annual tenors + 5-year intervals to 50Y (liquid); flat extrapolation beyond 50Y
3. **Interpolation**: Linear between illiquid tenors
4. **Scenario application**: 
   - Each scenario applies a shock to the base curve
   - Example: "Rates Up" = all rates +25bps
   - "Flattening" = front-end +50, back-end -50
5. **Per-period curves**: Different curve for each projection year (yields change as time evolves)

**Impact**: Your simplified flat rates obscure important curve dynamics. In reality, KRD calculations depend on the entire curve shape, not just one rate.

**How to Fix**:
- Add sub-section: "2.1 Interest Rate Curves — Base Case & Scenarios"
- Show sample Euribor curve (5Y = 2.5%, 10Y = 2.8%, 20Y = 3.0%, 50Y = 3.2%)
- Explain: "For scenarios, we apply a parallel shift (e.g., +25bps) or non-parallel twist"
- Note: "Reinvestment rates in this illustration are simplified from full curve; production model uses full term structure"
- Reference:
  - EXECUTIVE-SUMMARY pages 2-3
  - IMPLEMENTATION-SPECIFICATION pages 4-5
  - PHASE1-003 (Z-spread curve)
  - Methodology doc section 6

---

#### Gap 8: No Regulatory Mapping to BMA Article 28

**Current State**: Your doc doesn't explicitly show compliance with BMA requirements.

**What's Missing**:
- Mapping section showing which section of your doc demonstrates compliance with each Article 28 requirement
- Example: "Article 28(10): Sections 5 & 7 demonstrate calculation under base + stress scenarios"

**How to Fix**:
- Add appendix: "Regulatory Compliance Mapping — Article 28 vs. This Illustration"
- Map each Article 28 paragraph to sections that address it
- Reference: PHASE2-002 (full compliance mapping)

---

#### Gap 9: No Configuration Parameters Shown

**Current State**: Your doc hardcodes assumptions (3.0% base rate, 1.5% down rate, etc.). No reference to the 150+ parameters in alm_setup.xlsm.

**What's Missing**:
- Tolerance settings (goal_seek_tolerance_optimal = 0.001%)
- D&D rates by asset class
- Transaction cost assumptions
- Investment expense assumptions
- Other parameters that affect C0

**How to Fix**:
- Add appendix: "A. Configuration Parameters Used in This Illustration"
- Show table of key parameters and their values
- Reference: PHASE1-007, EXECUTIVE-SUMMARY pages 7-8

---

### ⚪ LOWER PRIORITY (NICE-TO-HAVE)

#### Gap 10: No Complex Instruments

Your doc uses only straight bonds. Real portfolios include:
- Callable bonds (wave-off logic when rates fall)
- FX forwards (currency risk)
- Swaptions (optionality)
- Structured products

**When to address**: Optional for v1. Could add as appendix "Appendix C: Complex Instruments".

---

## Completeness Scorecard

| Component | Your Doc | Project Docs | Gap Priority |
|-----------|----------|--------------|--------------|
| **Pedagogical intro** | ✅ Excellent | EXECUTIVE-SUMMARY p1 | Covered |
| **Simple example (5Y)** | ✅ Excellent | LEARNING-PATH | Covered |
| **All 9 BMA scenarios** | ⚠️ 5 of 9 only | EXECUTIVE-SUMMARY p3 | 🔴 CRITICAL |
| **KRD/rebalancing** | ❌ Missing | PHASE1-005, IMPLEMENTATION-SPEC p5-7 | 🔴 CRITICAL |
| **Z-spread calibration** | ❌ Missing | PHASE1-003, IMPLEMENTATION-SPEC p5-7 | 🔴 CRITICAL |
| **D&D formula** | ⚠️ Static only | PHASE1-009, IMPLEMENTATION-SPEC p7 | 🟡 MEDIUM |
| **Goal-seek algorithm** | ⚠️ Backward calc only | PHASE1-002, IMPLEMENTATION-SPEC p2-3 | 🟡 MEDIUM |
| **SBA spread + cap** | ❌ Missing | IMPLEMENTATION-SPEC p6, PHASE3-006 | 🟡 MEDIUM |
| **Interest rate curves** | ⚠️ Simplified | PHASE1-003, IMPLEMENTATION-SPEC p4-5 | 🟡 MEDIUM |
| **Configuration params** | ❌ Not referenced | PHASE1-007, EXECUTIVE-SUMMARY p7-8 | ⚪ LOWER |
| **Regulatory mapping** | ❌ Not explicit | PHASE2-002, PHASE2-004 | ⚪ LOWER |
| **Complex instruments** | ❌ None | PHASE1-* (various) | ⚪ OPTIONAL |
| **Expected output metrics** | ✅ BEL, biting scenario | IMPLEMENTATION-SPEC p9 | Covered |
| **Forced asset sales** | ✅ Excellent | IMPLEMENTATION-SPEC p10-11 | Covered |
| **Multi-asset portfolio** | ✅ Good (3 assets) | LEARNING-PATH | Covered |

**Overall Completeness**: 45% → Target with plan: 95%

---

## Proposed Enhancement Plan

### Vision

Create a **comprehensive, production-ready illustrative calculation** that:
1. ✅ Demonstrates full SBA logic (all 9 scenarios, KRD, rebalancing, spreads)
2. ✅ Bridges pedagogical clarity with technical precision
3. ✅ References every piece of project documentation
4. ✅ Highlights critical gaps (spread cap, callable bonds, D&D limits)
5. ✅ Serves as both an educational tool AND a test case for model validation

### Proposed Structure (25-30 pages)

#### Phase A: Foundation (5-6 pages) — Keep & Enhance Existing
**Goal**: Strengthen pedagogical base

1. **Introduction** (1 page)
   - Current section 1: KEEP (good intro)
   - ADD: "This illustration demonstrates all core SBA mechanics per BMA Article 28"
   - ADD: Cross-reference to 30-doc project

2. **Simple 5-Year Example** (4-5 pages)
   - Current sections 2-6: KEEP (excellent pedagogical)
   - ENHANCE: 
     - Add all 5 simple scenarios (currently have 5 of 5) ✅
     - Clarify BMA scenario names (e.g., "Scenario B: Rates Down" → "Scenario 2: Down 25bps (Parallel)")
     - Add Z-spread mean reversion (currently flat spread)
     - ADD sub-step: Show KRD calculation (simplified, 1 duration bucket)

#### Phase B: Comprehensive Multi-Asset + All 9 Scenarios (15-18 pages) — Replace Simple Section 7
**Goal**: Full realism

1. **Portfolio Setup** (2 pages)
   - Section 7.1: KEEP, ENHANCE with:
   - ADD: Configuration parameters table (from PHASE1-007)
   - ADD: Tier constraint explanation (already partially there)
   - ADD: Initial KRD positions (asset vs. liability durations)

2. **All 9 BMA Scenarios Specified** (1 page)
   - NEW: Show all 9 scenarios with year-by-year rate shocks
   - Reference: EXECUTIVE-SUMMARY p3, IMPLEMENTATION-SPECIFICATION p4
   - Example table: Scenario | Y1-10 Shock | Y11-100 Shock | Example Reinvestment Rate Path

3. **Detailed Scenario A Walkthrough (Base Case)** (2-3 pages)
   - REDO current section 7.3 to include:
   - Step 1: Calculate opening cash over years (KEEP)
   - Step 2: Apply D&D using cumulative formula: 1-(1-r)^t (NEW)
   - Step 3: Calculate asset KRD vs liability KRD (NEW)
   - Step 4: Rebalance swaps (1 iteration shown; note up to 3 in full model) (NEW)
   - Step 5: Rebalance TAA with transaction costs (NEW)
   - Step 6: Revalue assets after rebalancing (NEW)
   - Output: Final C0, binding year, KRD hedge positions

4. **Scenarios B-I Summary** (8-10 pages)
   - Condensed tables for remaining 8 scenarios (Scenarios B-I)
   - Show: C0 required, binding year(s), KRD evolution, rebalance trades
   - Highlighting: Scenario B (Rates Down) as biting scenario
   - Reference: IMPLEMENTATION-SPECIFICATION algorithm

5. **SBA BEL & Spread Calculation** (1-2 pages)
   - NEW: Show spread solve formula
   - Example: "SBA Spread solves: NPV_Liab(RF + spread) = BEL"
   - Apply cap: "SBA Spread = min(38bps calculated, 35bps cap) = 35bps"
   - Note 🔴 Gap: "Implementation will need to enforce this cap"
   - Reference: IMPLEMENTATION-SPECIFICATION p6, PHASE2-004

#### Phase C: Advanced Topics & Integration (3-4 pages) — Enhance Section 7.10 + Add
**Goal**: Connect to production model

1. **Edge Cases & Stress Scenarios** (1-2 pages)
   - Current section 7.9 (forced sales): KEEP (excellent)
   - ADD: 
     - Callable bond example (if rates fall, issuer calls → cap value)
     - Refinancing cliff (what if many bonds mature in year 5?)
     - Concentration limits (holding > 0.5% of one asset)
   - Reference: PHASE3-004 edge case tests E3.3, E7-E10

2. **Configuration Sensitivity** (1 page)
   - NEW: Show how C0 changes with:
   - Different goal_seek_tolerance
   - Different D&D rates
   - Different transaction costs
   - Reference: PHASE1-007

3. **Connection to Broader SBA Model** (0.5-1 page)
   - NEW: How does this 10-year illustration connect to full 100-year model?
   - How does it connect to regulatory requirements?
   - Reference: LEARNING-PATH, EXECUTIVE-SUMMARY p1, PHASE2-002 (compliance mapping)

#### Phase D: Appendices (2-3 pages) — Add
**Goal**: Reference & traceability

1. **Appendix A: Configuration Parameters**
   - Table of 150+ key params used (simplified subset)
   - Reference: PHASE1-007

2. **Appendix B: Regulatory Mapping**
   - Article 28(10) → Sections X & Y show compliance
   - Article 28(7) → Sections X & Y show all 9 scenarios
   - Reference: PHASE2-002, PHASE2-004

3. **Appendix C: Known Gaps & Production Considerations**
   - ADD: Spread cap not enforced (with code location) 🔴
   - ADD: Callable bond validation incomplete ⚠️
   - ADD: D&D formula details (cumulative, not annual)
   - Reference: PHASE3-006

4. **Appendix D: Glossary & Cross-References to Project Docs**
   - D&D, KRD, TAA, Tier 1/3, etc.
   - Every major term links to relevant Phase doc

---

## Implementation Roadmap

### Step 1: Gap Analysis Validation (1 hour)
- [ ] You review this gap analysis
- [ ] Confirm priorities (CRITICAL > MEDIUM > LOWER)
- [ ] Decide: Any gaps you want to defer or add?

### Step 2: Gather Precise Specs (1 hour)
- From project docs:
  - [ ] Exact 9 BMA scenarios with year-by-year shocks → EXECUTIVE-SUMMARY p3
  - [ ] KRD calculation formulas → IMPLEMENTATION-SPECIFICATION p5
  - [ ] Z-spread mean reversion formula → PHASE1-003
  - [ ] D&D cumulative formula → PHASE1-009
  - [ ] SBA spread solve & cap logic → IMPLEMENTATION-SPECIFICATION p6
  - [ ] Rebalancing algorithm → PHASE1-005
  - [ ] Goal-seek algorithm → PHASE1-002

### Step 3: Create Enhanced Document Structure (2-3 hours)
- [ ] Outline with section headings (Phase A/B/C/D)
- [ ] Decide: How many pages per section?
- [ ] Gather all project cross-references

### Step 4: Write Phase A (Foundation) — Enhanced (2-3 hours)
- [ ] Keep sections 1-6 (mostly as-is)
- [ ] Add Z-spread to simple example
- [ ] Add simple KRD (1 bucket duration)
- [ ] Clarify BMA scenario names

### Step 5: Write Phase B (Comprehensive) — NEW (8-10 hours)
- [ ] Rewrite section 7.1-7.10 with:
   - [ ] All 9 scenarios with exact year-by-year shocks
   - [ ] Full KRD calculation (by-tenor)
   - [ ] Swap rebalancing mechanics (1 iteration shown)
   - [ ] TAA rebalancing with transaction costs
   - [ ] D&D cumulative formula (not static rates)
   - [ ] Output: C0, binding years, KRD positions
- [ ] Show Scenarios A & B in detail (3-4 pages each)
- [ ] Summarize Scenarios C-I (condensed, 1 page each)

### Step 6: Write Phase C & D (Advanced + Appendices) — NEW (3-4 hours)
- [ ] Edge cases (callables, refinancing, concentration)
- [ ] Configuration sensitivity
- [ ] Regulatory mapping appendix
- [ ] Known gaps documentation (with code references)
- [ ] Glossary & cross-references

### Step 7: Validation & Polish (2 hours)
- [ ] Cross-check all numbers (no arithmetic errors)
- [ ] Verify all cross-references to project docs
- [ ] Peer review for clarity
- [ ] Final editing

**Total Effort**: 19-24 hours

---

## Success Criteria for Enhanced Document

When complete, the new document should:

- ✅ Demonstrate all 9 BMA scenarios (not just 5)
- ✅ Show KRD calculations and swap rebalancing in every period
- ✅ Use cumulative D&D formula (not static annual rates)
- ✅ Clearly show SBA spread calculation AND the 35bps cap
- ✅ Reference every key piece of project documentation
- ✅ Highlight all known gaps (spread cap, callable bonds, D&D limits)
- ✅ Provide exact cross-references to implementation code (file:line)
- ✅ Be usable as both pedagogical tool AND test case for model validation
- ✅ Serve as basis for regulatory submission document

---

## Questions for You

Before I proceed, I need your input:

1. **Scope**: Should I implement the full enhancement plan (Phases A-D, 25-30 pages)?
   - Or focus on just CRITICAL gaps (Phase B rewrite, 15-18 pages)?

2. **Depth of Rebalancing**: How much detail on KRD/swaps?
   - Simple (1 KRD bucket, 1 swap shown): 2-3 pages
   - Detailed (10 KRD buckets, full swap math): 5-7 pages

3. **Complexity**: Use the existing 10-year, 3-asset portfolio?
   - Or create a new portfolio design optimized for teaching all concepts?

4. **Code References**: Should I include actual file:line references to code?
   - Example: "See: sba_goal_seeker.py:250 for discontinuity detection"

5. **Approval**: Should I create the document as a standalone file, or integrate it into your existing one?
   - Standalone: Easier to manage; completely fresh structure
   - Integrated: Keep your pedagogical sections; insert new detail sections

---

**My Recommendation**:
1. Implement FULL plan (all 9 scenarios, KRD, rebalancing, spreads, gaps)
2. Use existing 3-asset portfolio (already good structure)
3. Detailed KRD (full bucketing, realistic complexity)
4. Include code references (supports your codebase auditing)
5. Create as NEW document (easier to structure; keep original as reference)

**Estimated Delivery**: 18-24 hours of work → ready for review

---

