# ANTHORA SBA Project — Metadata & Tracking

**Project**: Enhanced Illustrative SBA BEL Calculation Document
**Workspaces**: `c:\BMADev\Anthora_SBA`
**Current Document**: `02. Model/ReferenceDocuments/BMA_SBA_Illustrative_Calculation.md`
**Start Date**: March 25, 2026
**Status**: PLANNING (ready to implement)

---

## Project Summary

**Goal**: Transform existing pedagogical illustrative calculation into comprehensive, production-ready reference demonstrating **all 9 BMA scenarios, KRD/rebalancing, SBA spread calculation, with SII vs. SBA BEL comparisons**.

**Current Document Status**: 45% complete (5 of 9 scenarios, missing KRD/rebalancing, static spreads)

**Deliverable**: New standalone document, 28-32 pages, 5 phases

---

## Critical Gaps to Fix

🔴 **CRITICAL**:
- Missing 4 BMA scenarios (only 5 of 9 shown)
- Missing KRD & rebalancing mechanics (core SBA feature)
- Missing Z-spread calibration & mean reversion
- Missing SBA spread calculation & 35bps cap enforcement
- Missing goal-seek algorithm detail

🟡 **MEDIUM**:
- D&D formula simplified (static annual vs. cumulative)
- Interest rate curves simplified (flat vs. Euribor curve)
- Configuration parameters hardcoded

⚪ **LOWER**:
- No regulatory mapping to Article 28
- No complex instruments (callables, FX, swaptions)

---

## Implementation Plan

**Timeline**: 18-24 hours (complete & rigorous)

**Phases**:
1. **A**: Foundation (Enhanced) — 2.5 hrs
2. **B**: Comprehensive Portfolio + All 9 Scenarios — 8-10 hrs
3. **C**: SBA BEL & Output — 2-3 hrs
4. **D**: Advanced Topics — 1.5-2 hrs
5. **E**: Appendices — 1.5 hrs
6. **Validation & Polish** — 1.5 hrs

**Key Features**:
- All 9 BMA scenarios (exact year-by-year shocks)
- KRD with 10 duration buckets (detailed)
- Full rebalancing: KRD matching + TAA + 3 iterations
- Cumulative D&D formula: 1-(1-r)^t
- Mean-reverting Z-spreads
- SBA spread solve + 35bps cap (with gap flagged)
- 12 integrated SII vs. SBA BEL comparison sidebars
- Regulatory compliance mapping (Article 28, GENPRU 2R.3)

---

## Session Information

**Session Memory Files** (live during conversation):
- `/memories/session/plan.md` — Detailed 7-step implementation plan
- `/memories/session/GAP-ANALYSIS-Illustrative-Calculation.md` — Full gap analysis

**Requirements Confirmed**:
- Scope: Full enhancement (all gaps)
- SII Comparison: Yes (12 sidebars)
- KRD Depth: Detailed (10 buckets, full math)
- Format: New standalone document
- Timeline: Complete & rigorous (18-24 hours)

---

## Next Steps (When Ready to Implement)

1. [ ] Step 1: Gather specifications from EXECUTIVE-SUMMARY p3, IMPLEMENTATION-SPECIFICATION, PHASE1-002/003/005/009 (1 hr)
2. [ ] Step 2: Write PHASE A (Foundation enhanced) (2.5 hrs)
3. [ ] Step 3: Write PHASE B (Comprehensive portfolio) (8-10 hrs)
4. [ ] Step 4: Write PHASE C (SBA BEL & output) (2-3 hrs)
5. [ ] Step 5: Write PHASE D (Advanced topics) (1.5-2 hrs)
6. [ ] Step 6: Write PHASE E (Appendices) (1.5 hrs)
7. [ ] Step 7: Validate & polish (1.5 hrs)

**Verification Checklist**: 14 items (see session plan)

---

## Cross-Session Reference

**Found previous progress?** Check this file + session memory files for plan continuity.

**Starting fresh?** This file summarizes project scope + plan; session files have detailed implementation steps.

**To resume**: 
1. Read this file for context
2. Check `/memories/session/plan.md` for detailed steps
3. Update `PROJECT_PLAN_TRACKING.md` with current phase checklist

---