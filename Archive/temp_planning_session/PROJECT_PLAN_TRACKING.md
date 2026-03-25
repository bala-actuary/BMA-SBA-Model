# PROJECT_PLAN: Enhanced Illustrative SBA BEL Calculation

**Status**: READY TO IMPLEMENT (Phase 0: Planning Complete)  
**Last Updated**: March 25, 2026  
**Current Phase**: None yet (awaiting start signal)  
**Time Spent**: ~3 hours (research, planning, gap analysis)  
**Time Remaining**: 18-24 hours (implementation)

---

## Phase Checklist

### PHASE A: Foundation (Enhanced) — Estimated 2.5 hours
**Purpose**: Strengthen pedagogical base; introduce SII framework
**Status**: ⬜ NOT STARTED
- [ ] Section 1: Update intro + add SII vs. SBA Sidebar #1
- [ ] Sections 2-6: Enhance with Z-spread mean reversion + simple KRD
- [ ] Add SII comparison sidebars #2-5 (liability CF + scenarios + reinvestment + portfolio eligibility)
- [ ] Quality check: Pedagogical clarity maintained

**Time**: ___ hours | **Actual**: 0/2.5 ✓

---

### PHASE B: Comprehensive Portfolio + All 9 Scenarios — Estimated 8-10 hours
**Purpose**: Full realism; all mechanics demonstrated
**Status**: ⬜ NOT STARTED

#### Section 3: Setup (2-3 pages)
- [ ] Portfolio definition + initial KRD positions
- [ ] Configuration parameters table (10-15 key params from alm_setup.xlsm)
- [ ] All 9 BMA scenarios table with year-by-year shocks (EXECUTIVE-SUMMARY p3)
- [ ] Add SII sidebars #6-7 (Tier constraints + scenario standardization)
- **Subtask Time**: ___ hours

#### Section 4: Scenario A (Base Case) Detail (4-5 pages)
- [ ] Year 1 Period 1 full mechanics (2-2.5 pages):
  - [ ] D&D cumulative formula: Cumulative = 1-(1-annual_rate)^t
  - [ ] KRD calculation (10 buckets, Asset vs. Liability)
  - [ ] Swap rebalancing mechanics
  - [ ] TAA rebalancing with transaction costs
  - [ ] Convergence iterations (3 passes)
- [ ] Years 2-10 summary with tables
- [ ] Add SII sidebars #8-9 (hedging philosophy + multi-year projection)
- **Subtask Time**: ___ hours

#### Section 5: Scenarios B-I (10-12 pages)
- [ ] C0 results table (all 9 scenarios)
- [ ] Scenario 2 (Down 25, **BINDING**) detailed walkthrough (3-4 pages)
- [ ] Scenarios 3-8 summaries (1 page each, 6 pages total)
- [ ] All-scenarios comparison chart
- [ ] Add SII sidebar #9 (why down-rate scenarios bite)
- **Subtask Time**: ___ hours

**Time**: ___ hours | **Actual**: 0/8-10 ✓

---

### PHASE C: SBA BEL & Output — Estimated 2-3 hours
**Purpose**: Final calculation; highlight 35bps cap gap
**Status**: ⬜ NOT STARTED

#### Section 6: Goal-Seek Algorithm & SBA BEL
- [ ] Goal-seek overview (hybrid linear + bisection, attempts 1-20)
- [ ] Binding scenario selection
- [ ] SBA BEL calculation: NPV_Liab(RF) + C0
- [ ] SBA Spread derivation: Find spread where NPV_Liab(RF + spread) = BEL
- [ ] **🔴 Apply 35bps cap + FLAG GAP**: Currently NOT enforced in code
- [ ] Output metrics (BEL, C0, spread, binding scenario)
- [ ] Add SII sidebars #10-11 (binding constraint + final liability value)

**Time**: ___ hours | **Actual**: 0/2-3 ✓

---

### PHASE D: Advanced Topics & Integration — Estimated 1.5-2 hours
**Purpose**: Edge cases; connect to full model
**Status**: ⬜ NOT STARTED

#### Section 7: Edge Cases (1 page)
- [ ] Extended rate scenario (C0 nonlinearity)
- [ ] Callable bonds (wave-off logic)
- [ ] Refinancing cliff
- [ ] Concentration & Tier 3 constraints
- [ ] Add SII sidebar #12 (rate option treatment)

#### Section 8: Full Model Connection (0.75 pages)
- [ ] 10-year illustration vs. 100-year full model
- [ ] GENPRU 2R.3 submission requirements
- [ ] Asset eligibility check

**Time**: ___ hours | **Actual**: 0/1.5-2 ✓

---

### PHASE E: Appendices — Estimated 1.5 hours
**Purpose**: Reference & traceability
**Status**: ⬜ NOT STARTED

- [ ] **Appendix A**: KRD mathematics (Duration formula, KRD per tenor, swap mechanics)
- [ ] **Appendix B**: Configuration parameters (tolerances, D&D rates, costs, Tier limits)
- [ ] **Appendix C**: Regulatory compliance mapping (Article 28, GENPRU 2R.3)
- [ ] **Appendix D**: SII vs. SBA summary table (consolidate 12 sidebars)
- [ ] **Appendix E**: Known gaps (🔴 spread cap, 🟡 callables, D&D limits; severity flags)
- [ ] **Appendix F**: Glossary & cross-references to project docs

**Time**: ___ hours | **Actual**: 0/1.5 ✓

---

### VALIDATION & POLISH — Estimated 1.5 hours
**Status**: ⬜ NOT STARTED

- [ ] Verify all numbers (no arithmetic errors)
- [ ] Cross-check all 12 SII sidebars for consistency
- [ ] Verify all project doc cross-references
- [ ] Peer review for clarity
- [ ] Final formatting + TOC

**Time**: ___ hours | **Actual**: 0/1.5 ✓

---

## Verification Checklist (Final Quality Gates)

When complete, document must satisfy:

- [ ] All 9 BMA scenarios demonstrated (Scenarios 0-8)
- [ ] KRD calculations with 10 duration buckets shown
- [ ] Rebalancing mechanics explained (KRD + TAA + 3 iterations)
- [ ] Cumulative D&D formula shown
- [ ] Mean-reverting Z-spreads shown
- [ ] SBA BEL calculation demonstrated
- [ ] SBA spread cap (35bps) explained & gap flagged
- [ ] Binding scenario identified (Scenario 2, Down 25)
- [ ] All 12 SII vs. SBA sidebars present & consistent
- [ ] Regulatory mapping complete (Article 28, GENPRU 2R.3)
- [ ] Known gaps listed with severity flags
- [ ] Configuration parameters shown
- [ ] Cross-references to project docs throughout
- [ ] No arithmetic errors

---

## Related Session Documentation

**Repository Memory** (persists across all sessions):
- `PROJECT_METADATA.md` — Project metadata & summary

**Session Memory** (lives during this conversation only):
- `/memories/session/plan.md` — Detailed 7-step implementation plan (18-24 hrs)
- `/memories/session/GAP-ANALYSIS-Illustrative-Calculation.md` — Full gap analysis (critical/medium/lower)

---

## Next Steps

**When ready to start implementation**:
1. [ ] Signal "Proceed with Step 1" or "Start PHASE A"
2. [ ] I will gather project doc specs (1 hour)
3. [ ] Begin Phase A implementation
4. [ ] Update this file with phase progress (check boxes, actual times)

**To pause or pivot**:
- [ ] Update this file with current status + blocker
- [ ] I'll adjust plan or escalate

---

## Notes

- Document will be created as: **`BMA_SBA_BEL_Illustrative_Calculation_Comprehensive_v2.md`**
- Location: **`02. Model/ReferenceDocuments/`**
- Original doc will remain as reference
- Estimated page count: **28-32 pages**