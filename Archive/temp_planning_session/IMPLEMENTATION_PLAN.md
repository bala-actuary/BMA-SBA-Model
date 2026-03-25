# Plan: Enhanced Illustrative SBA BEL Calculation Document

## TL;DR
Transform existing pedagogical document into **comprehensive, production-ready reference** demonstrating all 9 BMA scenarios, KRD/rebalancing mechanics, SBA spread calculation, with **integrated SII vs. SBA BEL comparisons** throughout. New 28-32 page document.

---

## Requirements Confirmed
✅ **Scope**: Full (all 9 scenarios, KRD, rebalancing, goalseeking, spreads, all gaps)  
✅ **SII Comparison**: 12 sidebars integrated throughout document  
✅ **KRD Detail**: Detailed (10 duration buckets, full swap math, 3 iterations)  
✅ **Format**: New standalone document  
✅ **References**: Documentation-level (no code file:line)  
✅ **Timeline**: Complete & rigorous (18-24 hours)

---

## Document Structure (28-32 pages)

### PHASE A: Foundation — Enhanced (7-8 pages)

**Section 1: Introduction** (1 page)
- Keep current intro + add scope descriptor
- **SII vs. SBA Sidebar #1**: Comparing SII TP vs. SBA BEL foundations

**Sections 2-6: Simple 5-Year Example** (6-7 pages)
- KEEP pedagogical walkthrough
- ENHANCE: Add Z-spread mean reversion + simple KRD (1 bucket)
- ADD: 4 SII comparison sidebars (#2-#5) covering:
  - Liability cash flow treatment
  - Stress scenario philosophy  
  - Reinvestment risk handling
  - Portfolio eligibility

---

### PHASE B: Comprehensive Portfolio with All 9 Scenarios (18-20 pages)

**Section 3: Portfolio & Scenarios Setup** (2-3 pages)
- Portfolio definition + init KRD positions
- **Configuration parameters table** (key 10-15 from alm_setup.xlsm)
- **All 9 BMA scenarios** table with exact year-by-year shocks (reference EXECUTIVE-SUMMARY p3)
- SII sidebars: Tier constraints + scenario standardization

**Section 4: Scenario A (Base Case) — Detailed** (4-5 pages)
- **Period 1 full mechanics** (2-2.5 pages):
  - D&D cumulative formula application: Cumulative = 1-(1-annual_rate)^t
  - KRD calculation (10 duration buckets): Asset KRD ≠ Liability KRD
  - Swap rebalancing: Execute swaps to match durations
  - TAA rebalancing: Rebalance to target allocation + transaction costs
  - Output: Portfolio value, C0 interim, KRD position
- **Years 2-10 summary** with year-by-year tables
- SII sidebars: Interest rate hedging philosophy + multi-year projection

**Section 5: Scenarios B-I — Comparative** (10-12 pages)
- **C0 results for all 9 scenarios** (1 page table)
- **Scenario 2 (Down 25 — BINDING)** (3-4 pages): Full walkthrough similar to Scenario A
- **Scenarios 3-8** (6 pages, 1 page each): Condensed summaries with key insights
- **All-scenarios comparison** (1 page): C0 chart + binding year chart
- SII sidebar: Why down-rate scenarios bite

---

### PHASE C: SBA BEL & Output (2-3 pages)

**Section 6: Goal-Seek Algorithm & SBA BEL** (2-3 pages)
- **Goal-seek overview**: Hybrid linear + bisection, attempts 1-20, convergence tolerance
- **Binding scenario selection**: Identify max C0 across 9 scenarios
- **SBA BEL calculation**: NPV_Liabilities(RF) + C0
- **SBA Spread derivation**: Solve for spread where NPV_Liab(RF + spread) = BEL
- **🔴 Apply 35bps cap** (FLAG GAP: Currently NOT enforced in code)
- **Output metrics**: BEL, C0, spread, binding scenario
- SII sidebars: Binding constraint concept + final liability value

---

### PHASE D: Advanced Topics (2-3 pages)

**Section 7: Edge Cases** (1.5 pages)
- Extended rate scenario (C0 nonlinearity)
- Callable bonds (wave-off logic, caps value)
- Refinancing cliff
- Concentration & Tier 3 constraints
- SII sidebar: Rate option treatment

**Section 8: Connection to Full Model & Submission** (0.75 pages)
- 10-year illustration vs. 100-year full model (principles identical)
- GENPRU 2R.3 submission requirements
- Asset eligibility in your example

---

### PHASE E: Appendices (3-4 pages)

- **A**: KRD mathematics (Duration formula, KRD per tenor, swap mechanics)
- **B**: Configuration parameters (tolerances, D&D rates, costs, Tier limits)
- **C**: Regulatory compliance mapping (Article 28 → sections; GENPRU 2R.3)
- **D**: SII vs. SBA summary (consolidated 12 sidebars into comparison table)
- **E**: Known gaps (🔴 spread cap, 🟡 callables, D&D limits; severity flags)
- **F**: Glossary (KRD, TAA, C0, BEL, SII TP, Tier 3, D&D + project doc cross-refs)

---

## Implementation Steps (18-24 hours total)

**Step 1**: Gather specifications (1 hr)
- [ ] 9 BMA scenarios (EXECUTIVE-SUMMARY p3)
- [ ] KRD formulas (IMPLEMENTATION-SPECIFICATION p5-7)
- [ ] D&D cumulative (PHASE1-009)
- [ ] Z-spread mean reversion (PHASE1-003)
- [ ] Goal-seek algorithm (PHASE1-002)

**Step 2**: Write PHASE A (Foundation) (2.5 hrs)
- [ ] Enhance sections 1-6 + add 5 SII sidebars

**Step 3**: Write PHASE B (Comprehensive) (8-10 hrs)
- [ ] Section 3: Setup + all 9 scenarios + SII sidebars #6-7
- [ ] Section 4: Scenario A detail (D&D, KRD buckets, swap, TAA, iterations 1-3)
- [ ] Section 5: Scenarios B-I + binding scenario + SII sidebar #9

**Step 4**: Write PHASE C (SBA BEL) (2-3 hrs)
- [ ] Section 6: Goal-seek + BEL + spread solve + cap enforcement + SII sidebars #10-11

**Step 5**: Write PHASE D (Advanced) (1.5-2 hrs)
- [ ] Sections 7-8 + SII sidebar #12

**Step 6**: Write PHASE E (Appendices) (1.5 hrs)
- [ ] A-F: Math, params, compliance, SII comparison, gaps, glossary

**Step 7**: Validate & polish (1.5 hrs)
- [ ] Verify all numbers, cross-references, SII consistency
- [ ] Final formatting

---

## Verification Checklist

✅ All 9 BMA scenarios (0-8) demonstrated  
✅ KRD with 10 duration buckets shown  
✅ Rebalancing mechanics (KRD + TAA + 3 iterations)  
✅ Cumulative D&D formula: 1-(1-r)^t  
✅ Mean-reverting Z-spreads: z(t)=L+(z0-L)ρ^t  
✅ SBA BEL & spread calculation  
✅ 35bps cap with gap flag (🔴 not enforced)  
✅ Binding scenario identified (Scenario 2)  
✅ 12 SII vs. SBA sidebars consistent  
✅ Article 28 & GENPRU 2R.3 compliance mapped  
✅ Known gaps listed (severity flags)  
✅ Configuration params shown  
✅ Project doc cross-references throughout  
✅ No arithmetic errors

---

## Deliverable

**File**: `BMA_SBA_BEL_Illustrative_Calculation_Comprehensive_v2.md`  
**Length**: 28-32 pages  
**Audience**: SBA compliance, actuaries, auditors  
**Use Cases**: Learning + reference + test case + regulatory submission