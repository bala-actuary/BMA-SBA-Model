# BMA SBA Benchmark Model - Timeline & Budget Estimate

## Document Control

| Field | Value |
|---|---|
| **Project Name** | BMA SBA Benchmark Model |
| **Document Type** | Timeline & Budget Estimate |
| **Version** | 1.0 (Draft for Discussion) |
| **Date** | March 2026 |

---

## 1. Phase Breakdown

### Phase 0: Foundation + First Notebook (Weeks 1-3)

| Activity | Owner | Duration | Dependencies |
|---|---|---|---|
| Project setup (pyproject.toml, venv, pytest, CI) | Senior Dev | 3 days | Python/QuantLib installed |
| Create full directory structure | Senior Dev | 1 day | - |
| Implement RunConfig dataclass | Senior Dev | 1 day | - |
| Implement @rule_ref decorator and RuleTrace | Senior Dev | 2 days | - |
| Jupyter notebook: reproduce BMA illustrative calc | Project Lead + Senior Dev | 3 days | BMA_doc/BMA_SBA_Illustrative_Calculation.md |
| Write test_illustrative_calc.py (FAILING) | Project Lead | 2 days | Notebook complete |

**Milestone:** BMA illustrative example reproduced manually in Jupyter. Failing test defines the target.

**Exit Criteria:**
- [ ] `pytest` runs (1 test, FAILING as expected)
- [ ] Notebook shows: bond=$4,480, C0_base=$2,981, C0_down=$3,091, BEL=$7,571
- [ ] @rule_ref decorator working and logging traces

---

### Phase 1: Core Projection Engine (Weeks 4-9)

| Activity | Owner | Duration | Dependencies |
|---|---|---|---|
| model_points/schemas.py (Pydantic schemas) | Data Dev | 3 days | - |
| model_points/assets.py + liabilities.py (loaders) | Data Dev | 3 days | Schemas |
| assumptions/economic.py (flat rate) | Data Dev | 2 days | - |
| rules/v2024/scenarios.py (9 scenarios) | Project Lead + Senior Dev | 5 days | BMA rules review |
| projection/asset_cf.py (bond cash flows) | Senior Dev | 3 days | Model points |
| projection/liability_cf.py (EPL flows) | Senior Dev | 2 days | Model points |
| projection/net_cf.py | Senior Dev | 1 day | Asset + liability CFs |
| projection/reinvestment.py | Senior Dev | 3 days | Net CF |
| projection/disinvestment.py | Senior Dev | 3 days | Net CF |
| projection/scenario_runner.py | Senior Dev | 3 days | All projection modules |
| projection/multi_scenario.py (biting) | Senior Dev | 2 days | Scenario runner |
| calculations/bel.py | Project Lead | 2 days | Multi-scenario |
| Unit tests for all above | All | Ongoing | - |
| Integration test: illustrative calc PASSES | All | 3 days | All modules |

**Milestone:** `test_illustrative_calc.py` PASSES. Core BEL calculation works.

**Exit Criteria:**
- [ ] C0_base=2981, C0_down=3091, biting="Rates Down", BEL=7571
- [ ] All 9 scenarios run correctly
- [ ] Unit test coverage > 80% for projection/ module

---

### Phase 2: QuantLib Integration (Weeks 10-14)

| Activity | Owner | Duration | Dependencies |
|---|---|---|---|
| Jupyter notebook: QuantLib curve building | Senior Dev | 3 days | QuantLib installed |
| curves/yield_curve.py (PiecewiseYieldCurve) | Senior Dev | 5 days | QuantLib knowledge |
| curves/scenario_curves.py (9 shifted curves) | Senior Dev | 4 days | Base curve |
| curves/bond_pricing.py (FixedRateBond + engine) | Senior Dev | 4 days | Curves |
| curves/ql_audit.py (audit wrapper) | Senior Dev | 2 days | - |
| Update projection to use QuantLib pricing | Senior Dev | 3 days | All curves modules |
| Cross-validation: QuantLib vs hand calcs | Project Lead | 3 days | - |
| Regression: Phase 1 tests still pass | All | 2 days | - |

**Milestone:** Full yield curve term structure replaces flat rates. Bond pricing via QuantLib validated.

**Exit Criteria:**
- [ ] 9 scenario curves built from BMA published rates
- [ ] Bond prices match Excel PRICE() within 0.01%
- [ ] Phase 1 integration test still passes (document any rounding changes)
- [ ] All QuantLib calls logged by ql_audit.py

---

### Phase 3: Full Asset Universe (Weeks 15-20)

| Activity | Owner | Duration | Dependencies |
|---|---|---|---|
| Extend schemas: callable, floating, amortizing, MBS, ABS | Data Dev | 5 days | - |
| Callable bond pricing (HullWhite tree) | Senior Dev | 5 days | QuantLib |
| Floating rate bond pricing | Senior Dev | 3 days | QuantLib |
| Amortizing bond pricing | Senior Dev | 3 days | QuantLib |
| MBS/ABS cash flow model (PSA prepayment) | Senior Dev | 5 days | QuantLib |
| rules/v2024/asset_tiers.py (Tier classification) | Project Lead | 3 days | - |
| rules/v2024/credit_costs.py (BMA floor tables) | Project Lead | 4 days | BMA tables |
| rules/v2024/transaction_costs.py (spread grading) | Project Lead + Dev | 3 days | - |
| projection/credit_adjustment.py | Quant Dev | 4 days | Credit rules |
| projection/transaction_cost.py | Quant Dev | 2 days | Transaction rules |
| Update disinvestment: limited-basis exclusion | Senior Dev | 2 days | Tier rules |
| 10% cap enforcement | Data Dev | 2 days | Tier rules |
| Tests for each asset type | All | Ongoing | - |

**Milestone:** All BMA-eligible asset types supported. Credit and transaction costs applied.

**Exit Criteria:**
- [ ] Every asset type in BMA Tier 1/2/3 can be modeled
- [ ] Credit costs match BMA published floor tables
- [ ] Limited-basis assets excluded from disinvestment
- [ ] 10% cap validated

---

### Phase 4: Stress Tests + LCR + Risk Margin (Weeks 21-26)

| Activity | Owner | Duration | Dependencies |
|---|---|---|---|
| calculations/stress_tests.py: mass lapse + spread | Project Lead + Dev | 5 days | Full projection engine |
| calculations/stress_tests.py: one-notch downgrade | Project Lead + Dev | 3 days | Credit module |
| calculations/stress_tests.py: no reinvestment | Project Lead + Dev | 3 days | Reinvestment module |
| calculations/lcr.py: liquidity classification | Project Lead | 3 days | Asset tiers |
| calculations/lcr.py: fast-moving scenario | Project Lead + Dev | 3 days | - |
| calculations/lcr.py: sustained scenario | Project Lead + Dev | 3 days | - |
| calculations/lcr.py: 105% threshold check | Dev | 2 days | - |
| calculations/risk_margin.py: 6% CoC | Project Lead | 4 days | BEL + projection |
| calculations/technical_provision.py: TP = BEL + RM | Dev | 1 day | BEL + RM |
| calculations/lapse_cost.py | Project Lead | 2 days | - |
| Full test suite | All | 5 days | - |

**Milestone:** Complete regulatory calculation stack. Model produces BEL, RM, TP, LCR, and all stress test results.

**Exit Criteria:**
- [ ] All 3 stress tests implemented and tested
- [ ] LCR >= 105% check operational
- [ ] Risk Margin calculation matches hand computation
- [ ] TP = BEL + RM verified

---

### Phase 5: Audit Trail + Reporting (Weeks 27-30)

| Activity | Owner | Duration | Dependencies |
|---|---|---|---|
| engine/audit_trail.py: SQLite run log | Senior Dev | 3 days | - |
| engine/audit_trail.py: Parquet period logs | Senior Dev | 2 days | - |
| engine/audit_trail.py: JSON rule traces | Senior Dev | 2 days | @rule_ref |
| output/excel_writer.py: master workbook | Data Dev | 5 days | - |
| output/summary.py: dashboard sheet | Data Dev | 3 days | - |
| output/detail.py: period-by-period sheets | Data Dev | 3 days | - |
| output/audit_sheet.py: audit in Excel | Data Dev | 2 days | - |
| engine/runner.py: main orchestrator | Senior Dev | 3 days | All modules |
| CLI interface (click/typer) | Senior Dev | 2 days | Runner |

**Milestone:** End-to-end: load data -> run model -> produce Excel report + audit trail.

**Exit Criteria:**
- [ ] Multi-tab Excel workbook produced automatically
- [ ] Audit trail captures every period, every decision, every rule reference
- [ ] CLI: `python -m bma_sba_benchmark run --company X --date Y --rules Z`

---

### Phase 6: Company Challenge Module (Weeks 31-35)

| Activity | Owner | Duration | Dependencies |
|---|---|---|---|
| challenge/import_submission.py | Data Dev | 5 days | Company template spec |
| model_points/company_submission.py | Data Dev | 3 days | Schemas |
| challenge/compare.py: metric comparison | Senior Dev | 4 days | Run results |
| challenge/variance_analysis.py: CF drill-down | Senior Dev | 3 days | - |
| challenge/challenge_report.py: report gen | Data Dev | 4 days | Compare results |
| output/comparison_sheet.py | Data Dev | 3 days | - |
| engine/batch.py: multi-company | Senior Dev | 3 days | Runner |
| Test with synthetic submissions | All | 3 days | - |

**Milestone:** Import company submission, compare against benchmark, produce challenge report.

**Exit Criteria:**
- [ ] Company Excel workbook parsed and validated
- [ ] Comparison report shows pass/fail by metric with tolerances
- [ ] Batch run processes multiple companies
- [ ] Challenge report includes rule references for each finding

---

### Phase 7: Hardening + Validation (Weeks 36-40)

| Activity | Owner | Duration | Dependencies |
|---|---|---|---|
| rules/v2025/ implementation | Project Lead | 3 days | v2024 complete |
| Regression test suite (golden files) | QA / Dev | 5 days | All modules |
| Independent actuarial review | Second Actuary / External | 10 days | All modules |
| Fix issues found in review | All | 5 days | Review complete |
| rule_traceability_matrix.md (complete) | Project Lead | 3 days | All rules modules |
| User guide | Data Dev | 3 days | CLI + reporting |
| Assumption governance document | Project Lead | 2 days | - |
| Performance testing (large portfolios) | Senior Dev | 2 days | - |

**Milestone:** Production-ready benchmark model, independently validated.

**Exit Criteria:**
- [ ] v2024 and v2025 rules both working
- [ ] Independent actuarial review signed off
- [ ] Golden file regression tests automated in CI
- [ ] Rule traceability matrix 100% complete
- [ ] User guide delivered

---

## 2. Timeline Visualization

### Option A: Lean Team (3 people)

```
Month:   1        2        3        4        5        6        7        8        9        10       11
Week:    1-4      5-8      9-12     13-16    17-20    21-24    25-28    29-32    33-36    37-40    41-44
         ├────────┼────────┼────────┼────────┼────────┼────────┼────────┼────────┼────────┼────────┤
Phase 0: ████
Phase 1:     ████████████████
Phase 2:                     ████████████
Phase 3:                                  ████████████████
Phase 4:                                                   ████████████████
Phase 5:                                                                    ██████████
Phase 6:                                                                              ████████████
Phase 7:                                                                                         ████████████
         ├────────┼────────┼────────┼────────┼────────┼────────┼────────┼────────┼────────┼────────┤
MILESTONES:
         Notebook  BEL      QuantLib  Full     Stress/  Audit/   Challenge  Hardened
         + Test    works!   curves   assets   LCR/RM   Reports  module     + Valid.
```

### Option B: Moderate Team (4-5 people) - Parallel Streams

```
Month:   1        2        3        4        5        6        7        8        9
Week:    1-4      5-8      9-12     13-16    17-20    21-24    25-28    29-32    33-36
         ├────────┼────────┼────────┼────────┼────────┼────────┼────────┼────────┤
Phase 0: ████
Phase 1:     ████████████████
Phase 2:              ████████████           ← Quant Dev starts early
Phase 3:                     ████████████    ← Senior Dev + Quant Dev parallel
Phase 4:                          ████████████
Phase 5:                               ██████████  ← Data Dev works in parallel
Phase 6:                                    ████████████
Phase 7:                                              ████████████
         ├────────┼────────┼────────┼────────┼────────┼────────┼────────┼────────┤
```

### Option C: Full Team (6 people) - Maximum Parallelism

```
Month:   1        2        3        4        5        6        7
Week:    1-4      5-8      9-12     13-16    17-20    21-24    25-28
         ├────────┼────────┼────────┼────────┼────────┼────────┤
Phase 0: ████
Phase 1:     ████████████
Phase 2:         ████████████        ← Parallel with late Phase 1
Phase 3:              ████████████   ← Senior + Quant + Data all active
Phase 4:                   ████████  ← 2nd Actuary leads this
Phase 5:                ██████████   ← Data Dev builds while Phase 4 runs
Phase 6:                     ████████████
Phase 7:                          ████████████
         ├────────┼────────┼────────┼────────┼────────┼────────┤
```

---

## 3. Key Milestones and Deliverables

| Milestone | Week | Deliverable | Demo |
|---|---|---|---|
| **M1: Notebook Ready** | 3 | Jupyter notebook reproducing BMA example | Show stakeholders the BMA math working in Python |
| **M2: BEL Works** | 9 | Illustrative calc test passes | Run model, show BEL = $7,571 |
| **M3: QuantLib Curves** | 14 | Full term structure, proper bond pricing | Show 9 yield curves, validated bond prices |
| **M4: Full Assets** | 20 | All BMA asset types modeled | Run with realistic multi-asset portfolio |
| **M5: Regulatory Stack** | 26 | BEL + LCR + stress tests + RM | Complete regulatory output |
| **M6: Excel Reports** | 30 | Regulatory-quality Excel workbook + audit trail | Present report to stakeholders |
| **M7: Challenge Module** | 35 | Company comparison operational | Demo with synthetic company submission |
| **M8: Production Ready** | 40 | Validated, documented, regression-tested | Final sign-off |

---

## 4. GitHub Copilot Impact on Timeline

### Where Copilot Saves Time

| Module | Without Copilot | With Copilot | Saving |
|---|---|---|---|
| Pydantic schemas (model_points/) | 5 days | 3 days | 2 days |
| pandas I/O (data loading) | 4 days | 3 days | 1 day |
| pytest fixtures and scaffolding | 4 days | 2 days | 2 days |
| openpyxl Excel formatting | 8 days | 5 days | 3 days |
| CLI setup (click/typer) | 2 days | 1 day | 1 day |
| Type hints and docstrings | 3 days | 1 day | 2 days |
| **Total savings** | | | **~11 days** |

### Where Copilot Has No Impact

| Module | Reason |
|---|---|
| QuantLib integration (curves/) | Copilot hallucinates QuantLib APIs; every suggestion must be verified |
| BMA rule implementation (rules/) | Requires regulatory interpretation, not code generation |
| Projection logic (projection/) | Custom actuarial logic, not pattern-matchable |
| BEL/LCR/stress calculations | Domain-specific math, not boilerplate |
| @rule_ref design | Novel design pattern for this project |
| Test case design | Requires actuarial judgment about what to test |

### Net Impact

| Metric | Value |
|---|---|
| Total project effort (Option A) | ~220 person-days |
| Copilot savings | ~11 days (~5%) |
| Effective timeline reduction | < 1 week |
| **Verdict** | Copilot is a nice-to-have, not a game-changer for this project |

---

## 5. Budget Framework

### Development Cost (Team Option A: 3 people, 11 months)

| Cost Category | Estimate |
|---|---|
| Senior Python Developer (11 months, full-time) | Market rate * 11 |
| Mid-level Python Developer (10 months, full-time) | Market rate * 10 |
| Project Lead (existing staff - no incremental) | $0 incremental |
| **Subtotal: Personnel** | Variable by market |

### Tools and Infrastructure

| Item | Cost | Notes |
|---|---|---|
| VS Code | Free | Open source |
| GitHub Copilot (2 seats) | ~$40/month/seat | Business plan |
| Python + QuantLib | Free | Open source |
| GitHub (private repo) | ~$4/month/seat | Team plan |
| Windows workstations | Existing | BMA provided |
| **Subtotal: Tools** | ~$100-150/month | Negligible |

### Optional Costs

| Item | Cost | When |
|---|---|---|
| QuantLib consultant (2 months) | Market rate | If Senior Dev lacks QuantLib experience |
| Independent actuarial review | Market rate | Phase 7 (if no second actuary hired) |
| Training: Python course for Project Lead | ~$500-2,000 | Month 0 |

---

## 6. Dependencies and Critical Path

```
                                    CRITICAL PATH
                                    ─────────────

Hire Senior Dev ──> Phase 0 ──> Phase 1 ──> Phase 2 ──> Phase 3 ──> Phase 4 ──> Phase 7
                                   │                                     │
                                   │                              Phase 5 (parallel)
                                   │                                     │
                                   │                              Phase 6 (parallel)
                                   │
                              Data Dev joins
                              (parallel data/reporting work)
```

**The critical path runs through the Senior Python Developer.** If this hire is delayed by N weeks, the entire project shifts by N weeks.

### External Dependencies

| Dependency | Needed By | Risk if Delayed |
|---|---|---|
| Senior Dev hired | Week 1 | Project cannot start |
| BMA credit cost tables (digital) | Week 15 | Phase 3 blocked |
| BMA risk-free curve data (digital) | Week 10 | Phase 2 blocked |
| Company submission template spec | Week 31 | Phase 6 blocked |
| EPL format specification | Week 4 | Phase 1 blocked (can use illustrative example as fallback) |
| IT approval for Python/QuantLib | Week 1 | Entire project blocked |
