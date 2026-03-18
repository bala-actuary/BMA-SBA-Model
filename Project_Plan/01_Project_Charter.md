# BMA SBA Benchmark Model - Project Charter

## Document Control

| Field | Value |
|---|---|
| **Project Name** | BMA SBA Benchmark Model |
| **Document Type** | Project Charter |
| **Version** | 1.0 (Draft for Discussion) |
| **Date** | March 2026 |
| **Owner** | [Project Lead - Actuarial] |
| **Status** | Draft - Pending Team Review |

---

## 1. Executive Summary

The Bermuda Monetary Authority (BMA) requires an independent, internal benchmark model to verify and challenge insurance companies' Scenario Based Approach (SBA) submissions. Currently, BMA relies entirely on companies' own models (typically built in Prophet, AXIS, or similar commercial platforms) with no internal tool to independently reproduce or challenge their calculations.

This project will build **BMA's official SBA benchmark model** - a transparent, auditable Python-based calculation engine that serves as the authoritative reference implementation of BMA's own SBA rules.

---

## 2. Vision Statement

> Build a best-in-class regulatory benchmark model that enables BMA to independently assess the soundness of participating insurance companies' SBA submissions through transparent, rule-traceable, and reproducible calculations.

---

## 3. Project Objectives

### Primary Objectives

| # | Objective | Success Criteria |
|---|---|---|
| O1 | **Independent SBA Calculation** | Model can calculate BEL, Risk Margin, and Technical Provision for any company's portfolio using BMA's prescribed assumptions |
| O2 | **Company Challenge Capability** | Model can import a company's SBA submission and produce a line-by-line comparison report identifying material variances |
| O3 | **Full Regulatory Compliance** | Model implements all 9 prescribed interest rate scenarios, 3 mandatory stress tests, LCR calculation, and Risk Margin |
| O4 | **Audit Readiness** | Every calculation is traceable to a specific BMA rule paragraph via the audit trail |
| O5 | **Long-Term Sustainability** | Model handles annual BMA rule changes through versioned rule modules without code rewrites |

### Secondary Objectives

| # | Objective | Success Criteria |
|---|---|---|
| O6 | **Multi-Company Batch Processing** | Model can run benchmark calculations for multiple companies in a single batch |
| O7 | **Industry Analysis** | Aggregated results across companies enable cross-industry comparison and trend analysis |
| O8 | **Knowledge Transfer** | Model serves as a training tool for new BMA actuarial staff learning SBA rules |

---

## 4. Scope

### In Scope

| Area | Details |
|---|---|
| **Asset Cash Flow Projection** | All BMA-eligible asset types: government/corporate/municipal bonds, callable bonds, floating-rate bonds, amortizing bonds, MBS, ABS, structured securities |
| **Liability Cash Flows** | External input (EPL - External Projected Liabilities). Model does NOT project liabilities - they are provided by the company or BMA's liability team. |
| **9 BMA Interest Rate Scenarios** | Base, Decrease, Increase, Down-Up, Up-Down, and 4 twist scenarios with prescribed shift amounts |
| **BEL Calculation** | Biting scenario determination, initial cash buffer (C0) calculation via root-finding |
| **Risk Margin** | 6% Cost of Capital method with projected Modified ECR |
| **Technical Provision** | TP = BEL + Risk Margin |
| **LCR** | Liquidity Coverage Ratio (>= 105%) with fast-moving and sustained deterioration scenarios |
| **Stress Tests** | Combined mass lapse + credit spread widening, one-notch downgrade, no reinvestment |
| **Company Comparison** | Import company submissions, compare against benchmark, produce challenge reports |
| **Rule Versioning** | Support for annual BMA rule changes (v2024, v2025, etc.) |
| **Audit Trail** | Complete logging of every calculation step traceable to BMA rule references |
| **Reporting** | Regulatory-quality multi-tab Excel workbook output |

### Out of Scope

| Area | Rationale |
|---|---|
| **Liability Projection** | Handled by separate BMA models or provided by companies as EPL |
| **Web Application / UI** | Model is a desktop calculation engine; BMA analysts use Excel for review |
| **Real-Time / Online Processing** | Batch processing is sufficient for regulatory review cycles |
| **Stochastic Modeling** | Deterministic scenario-based approach per BMA SBA rules. Stochastic extensions may be a future phase. |
| **BSCR Calculation** | Separate BMA model; this model feeds into BSCR, not calculates it |
| **Company Data Collection** | Data submission and collection processes are separate from the benchmark model |

---

## 5. Strategic Alignment

### Why This Model Matters for BMA

1. **Regulatory Credibility**: BMA can demonstrate to international peers (IAIS, PRA, NAIC) that it independently verifies company submissions, not just accepts them at face value.

2. **Supervisory Effectiveness**: Material variances between company submissions and the benchmark trigger targeted supervisory actions, reducing the risk of undetected model errors.

3. **Rule Consistency**: A single reference implementation ensures BMA applies its own rules consistently across all participating companies.

4. **Institutional Knowledge**: The model encodes BMA's interpretation of its own rules in executable, testable form - reducing key-person risk when staff rotate.

5. **International Equivalence**: Supports BMA's Solvency II equivalence status by demonstrating robust supervisory tools comparable to PRA's internal model review capability.

### Comparable Regulatory Models

| Regulator | Model | Purpose |
|---|---|---|
| UK PRA | Matching Adjustment benchmark | Independently assess MA applications |
| NAIC | PBR prescribed assumptions | Floor/validation for company-specific reserves |
| EIOPA | Long-Term Guarantee measures assessment | EU-wide benchmark for Solvency II calibration |
| **BMA (this project)** | **SBA Benchmark Model** | **Independent verification of company SBA submissions** |

---

## 6. Key Stakeholders

| Stakeholder | Role in Project | Interest |
|---|---|---|
| **BMA Senior Management** | Sponsor / Approval | Regulatory credibility, resource allocation |
| **BMA Actuarial Team** | Primary Users | Day-to-day model operation, company reviews |
| **BMA Insurance Supervision** | Key Consumer | Supervisory actions informed by benchmark results |
| **BMA IT / Infrastructure** | Support | Desktop deployment, data security, backups |
| **External Auditors** | Periodic Review | Model governance, audit trail verification |
| **Participating Insurance Companies** | Subject of Analysis | Understand what BMA's benchmark expects |

---

## 7. Governance Structure

### Model Governance (per SR 11-7 / SS1/23 best practices)

| Governance Element | Approach |
|---|---|
| **Model Owner** | Project Lead (Actuarial) |
| **Model Validation** | Independent review every 3 years (BMA's existing validation framework) |
| **Change Control** | All changes via Git version control; major changes require documented approval |
| **Assumption Governance** | All assumptions versioned in YAML/CSV with change logs; BMA published tables as floor |
| **Audit Trail** | Every run logged with full inputs, outputs, and rule traces |
| **Documentation** | Rule traceability matrix maintained; every calculation maps to BMA rule paragraph |

### Project Governance

| Decision | Authority |
|---|---|
| Scope changes | Project Lead + BMA Senior Management |
| Technology choices | Project Lead + Senior Python Developer |
| BMA rule interpretations | Project Lead (actuarial judgment, documented) |
| Hiring decisions | Project Lead + BMA HR |
| Budget allocation | BMA Senior Management |

---

## 8. Success Criteria

### Phase 1 (Core - Month 3)
- [ ] Model reproduces the BMA illustrative calculation exactly (C0_base=2981, C0_down=3091, BEL=7571)
- [ ] All 9 interest rate scenarios implemented and verified

### Phase 2 (Full Calculations - Month 7)
- [ ] All BMA asset types supported (bonds, callable, MBS/ABS, structured)
- [ ] BEL + Risk Margin + Technical Provision calculated correctly
- [ ] LCR and all 3 stress tests implemented
- [ ] Regulatory-quality Excel output produced

### Phase 3 (Benchmark Ready - Month 10)
- [ ] Company comparison module operational
- [ ] Multi-company batch runs working
- [ ] Complete audit trail with rule traceability
- [ ] Independent actuarial validation completed
- [ ] Rule versioning tested (v2024 + v2025)

---

## 9. Constraints and Assumptions

### Constraints
- Development tooling: VS Code + GitHub Copilot (Claude Code not permitted)
- Budget: To be determined based on team size decision
- No existing codebase to build on (greenfield development)
- Must run on standard BMA workstations (Windows, no cloud/Docker dependency)

### Assumptions
- EPL (liability cash flows) will be provided in a standardized format
- BMA published credit cost tables and risk-free curves are available in machine-readable form
- Company submissions follow a standardized BMA template
- Team members can be hired within a reasonable timeframe
- Python 3.11+ and QuantLib-Python are approved for BMA use

---

## 10. Dependencies

| Dependency | Provider | Risk if Delayed |
|---|---|---|
| EPL format specification | BMA Liability Team | Cannot build data loaders |
| BMA credit cost tables (machine-readable) | BMA Prudential Standards | Cannot validate credit cost module |
| Company submission template | BMA Insurance Supervision | Cannot build challenge module |
| BMA risk-free curve data | BMA Market Risk | Cannot build yield curve module |
| Hiring approval and budget | BMA Senior Management | Cannot start development |
| IT approval for Python/QuantLib installation | BMA IT | Cannot develop on BMA machines |

---

## 11. Next Steps

1. **Team Discussion** - Review this charter and all supporting documents with the wider BMA team
2. **Team Size Decision** - Select from Option A (3 people), B (4-5), or C (6) based on timeline and budget
3. **Hiring** - Begin recruitment for Senior Python Developer (critical path)
4. **IT Coordination** - Confirm Python/QuantLib can be installed on BMA workstations
5. **Data Dependencies** - Initiate conversations with Liability Team, Prudential Standards, and Insurance Supervision for input data formats

---

## Appendix: Related Documents

| Document | Location |
|---|---|
| Architecture Design | `Project_Plan/02_Architecture_Design.md` |
| Team & Hiring Guide | `Project_Plan/03_Team_And_Hiring_Guide.md` |
| Timeline & Budget | `Project_Plan/04_Timeline_And_Budget.md` |
| Technical Specifications | `Project_Plan/05_Technical_Specifications.md` |
| Risk Assessment | `Project_Plan/06_Risk_Assessment.md` |
| Technology Stack | `Project_Plan/07_Technology_Stack.md` |
| BMA SBA Consolidated Guide | `BMA_doc/BMA_SBA_Consolidated_Guide.md` |
| BMA SBA Illustrative Calculation | `BMA_doc/BMA_SBA_Illustrative_Calculation.md` |
