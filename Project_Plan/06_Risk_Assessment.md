# BMA SBA Benchmark Model - Risk Assessment

## Document Control

| Field | Value |
|---|---|
| **Project Name** | BMA SBA Benchmark Model |
| **Document Type** | Risk Assessment |
| **Version** | 1.0 (Draft for Discussion) |
| **Date** | March 2026 |

---

## 1. Risk Scoring Framework

| Probability | Score | Definition |
|---|---|---|
| Low | 1 | Unlikely to occur (<20% chance) |
| Medium | 2 | May occur (20-60% chance) |
| High | 3 | Likely to occur (>60% chance) |

| Impact | Score | Definition |
|---|---|---|
| Low | 1 | Adds <2 weeks to timeline |
| Medium | 2 | Adds 2-6 weeks to timeline |
| High | 3 | Adds >6 weeks or threatens project viability |

**Risk Rating = Probability x Impact** (1-3 = Low, 4-6 = Medium, 7-9 = High)

---

## 2. Risk Register

### R1: QuantLib Learning Curve

| Attribute | Value |
|---|---|
| **Category** | Technical |
| **Probability** | High (3) |
| **Impact** | Medium (2) |
| **Risk Rating** | **6 (Medium)** |
| **Description** | Team has no QuantLib-Python experience. QuantLib has poor documentation, a steep learning curve, and cryptic error messages (C++ exceptions surfacing through Python bindings). |
| **Timeline Impact** | +3-5 weeks |
| **Trigger** | Developers unable to construct yield curves or price bonds correctly within first 2 weeks of Phase 2 |
| **Mitigation** | 1. Phase 0 includes a QuantLib bootcamp with Jupyter notebooks. 2. Phase 1 deliberately avoids QuantLib (uses simple math). 3. Hiring a Senior Dev with financial library experience. 4. Budget for a QuantLib consultant (2 months) if needed. 5. Maintain the QuantLib Cookbook notebook as a living reference. |
| **Contingency** | If QuantLib proves too difficult, fall back to pure Python yield curve implementation using numpy interpolation. This is less credible but fully functional. |
| **Owner** | Senior Python Developer |

---

### R2: BMA Rule Interpretation Ambiguity

| Attribute | Value |
|---|---|
| **Category** | Regulatory |
| **Probability** | High (3) |
| **Impact** | Medium-High (2.5) |
| **Risk Rating** | **7.5 (High)** |
| **Description** | BMA rules contain areas open to interpretation. Key ambiguities include: exact reinvestment mechanics, treatment of callable bonds under stress, how to project Modified ECR for Risk Margin, LCR eligibility criteria for complex assets. Different interpretations can produce materially different BEL results. |
| **Timeline Impact** | +2-4 weeks |
| **Trigger** | Team encounters a rule that could be interpreted two ways, and the difference in BEL exceeds 1% |
| **Mitigation** | 1. Document every interpretation decision with the rule reference and reasoning. 2. Flag ambiguities early for BMA Prudential Standards team clarification. 3. The @rule_ref decorator creates an audit trail of exactly which interpretation was coded. 4. Build sensitivity tests showing the impact of alternative interpretations. 5. Maintain a "Regulatory Interpretation Register" alongside the code. |
| **Contingency** | If BMA cannot clarify in time, implement the more conservative interpretation (higher BEL) as default, with the alternative as a configurable option. |
| **Owner** | Project Lead (Actuarial) |

---

### R3: MBS/ABS Modeling Complexity

| Attribute | Value |
|---|---|
| **Category** | Technical |
| **Probability** | Medium (2) |
| **Impact** | High (3) |
| **Risk Rating** | **6 (Medium)** |
| **Description** | QuantLib's MBS/ABS support is limited and poorly documented. Prepayment modeling (PSA, CPR) requires either the AbsBox library or custom implementation. Structured securities (CLOs, CDOs) have complex waterfall cash flows that may not fit the standard bond model. |
| **Timeline Impact** | +3-6 weeks |
| **Trigger** | During Phase 3, team cannot produce accurate MBS cash flows that match market analytics |
| **Mitigation** | 1. Start with simplified approach: model MBS as amortizing bonds with a PSA prepayment speed assumption. 2. Defer full waterfall modeling to a Phase 2 release. 3. Evaluate AbsBox as an alternative to QuantLib for structured products. 4. For the benchmark model, a simpler MBS model may actually be preferable (more transparent, easier to audit). |
| **Contingency** | If MBS/ABS modeling proves too complex, implement a "proxy" approach: model structured products as corporate bonds with adjusted credit assumptions. Document the simplification and its impact. |
| **Owner** | Senior Python Developer |

---

### R4: Senior Python Developer Hiring Difficulty

| Attribute | Value |
|---|---|
| **Category** | Resource |
| **Probability** | Medium-High (2.5) |
| **Impact** | High (3) |
| **Risk Rating** | **7.5 (High)** |
| **Description** | The intersection of "senior Python developer" + "financial library experience (QuantLib)" + "willing to work in Bermuda or remotely for BMA" is a small talent pool. This is the critical-path hire. |
| **Timeline Impact** | +N weeks (project cannot start until this person is hired) |
| **Trigger** | No suitable candidate found within 6 weeks of posting |
| **Mitigation** | 1. Start recruiting immediately, before other decisions are finalized. 2. Accept remote work if Bermuda-based candidates are scarce. 3. Consider contract-to-permanent arrangement. 4. Widen search to adjacent profiles: quantitative developers from banks/hedge funds, actuarial software developers, open-source QuantLib contributors. 5. Prepare a compelling job description emphasizing the unique regulatory impact of the role. |
| **Contingency** | Hire a strong senior Python developer WITHOUT QuantLib experience, and separately engage a QuantLib consultant for Phases 0-2. The consultant knowledge-transfers to the full-time developer. |
| **Owner** | Project Lead + BMA HR |

---

### R5: Scope Creep

| Attribute | Value |
|---|---|
| **Category** | Management |
| **Probability** | Medium (2) |
| **Impact** | High (3) |
| **Risk Rating** | **6 (Medium)** |
| **Description** | Stakeholders may request additional features: web UI, stochastic modeling, real-time processing, integration with other BMA systems, additional stress tests beyond the prescribed 3. Each addition could add months to the timeline. |
| **Timeline Impact** | +2-6 months if unchecked |
| **Trigger** | Stakeholder requests a feature not in the Project Charter scope |
| **Mitigation** | 1. The Project Charter clearly defines in-scope and out-of-scope. 2. Phased delivery means working software is available at each milestone. 3. Use a "future enhancements" backlog rather than adding to current scope. 4. Every scope request must be evaluated against the question: "Does this help BMA verify a company's SBA submission?" |
| **Contingency** | If scope must expand, extend the timeline proportionally rather than compressing quality. A benchmark model with incomplete audit trails is worse than a late benchmark model. |
| **Owner** | Project Lead |

---

### R6: Copilot-Generated Code Quality

| Attribute | Value |
|---|---|
| **Category** | Technical |
| **Probability** | High (3) |
| **Impact** | Low-Medium (1.5) |
| **Risk Rating** | **4.5 (Medium)** |
| **Description** | GitHub Copilot will generate plausible-looking but incorrect code, particularly for QuantLib API calls (wrong class names, wrong parameter order), regulatory calculations (subtly wrong formulas), and financial conventions (wrong day count, wrong compounding). |
| **Timeline Impact** | +1-2 weeks (debugging time) |
| **Trigger** | Test failures traced to Copilot-generated code that was accepted without review |
| **Mitigation** | 1. Mandatory code review for ALL code, especially QuantLib and regulatory logic. 2. Write detailed docstrings explaining what a function should do BEFORE asking Copilot to fill it in. 3. Never accept Copilot output for @rule_ref functions without tracing to the BMA rule. 4. Cross-validate all financial calculations against an independent source (Excel, Bloomberg). |
| **Contingency** | N/A - this risk is managed through process, not contingency. |
| **Owner** | All developers |

---

### R7: QuantLib-Python Windows Installation Issues

| Attribute | Value |
|---|---|
| **Category** | Technical / Infrastructure |
| **Probability** | Medium (2) |
| **Impact** | Low-Medium (1.5) |
| **Risk Rating** | **3 (Low)** |
| **Description** | QuantLib-Python installation on Windows can fail due to missing C++ build tools, version conflicts with conda/pip, or BMA IT security restrictions on installing compiled packages. |
| **Timeline Impact** | +1 week |
| **Trigger** | `pip install QuantLib-Python` or `conda install -c conda-forge quantlib-python` fails on BMA workstations |
| **Mitigation** | 1. Test installation on BMA workstations in Week 1 (before any code is written). 2. Use conda-forge (pre-compiled binaries, more reliable than pip). 3. Coordinate with BMA IT to whitelist the package. 4. Have a WSL2 (Windows Subsystem for Linux) fallback if native Windows installation fails. |
| **Contingency** | If QuantLib cannot be installed at all, use a pure Python yield curve and bond pricing implementation. This is more work but eliminates the dependency. |
| **Owner** | Senior Python Developer + BMA IT |

---

### R8: Data Dependency Delays

| Attribute | Value |
|---|---|
| **Category** | External |
| **Probability** | Medium (2) |
| **Impact** | Medium (2) |
| **Risk Rating** | **4 (Medium)** |
| **Description** | The model requires several external data inputs: BMA published credit cost tables (machine-readable), BMA risk-free curve data, company submission template specification, and EPL format specification from the liability team. Delays in any of these block specific phases. |
| **Timeline Impact** | +2-4 weeks per blocked phase |
| **Trigger** | Required data not available when the corresponding phase starts |
| **Mitigation** | 1. Initiate data requests early (during hiring phase). 2. Use the BMA illustrative calculation as a self-contained test case that requires no external data. 3. Create synthetic/mock data for development; replace with real data when available. 4. The phased approach means only Phase 1 data is needed immediately; later phases can wait. |
| **Contingency** | Develop and test with synthetic data. Flag in documentation that production data has not yet been validated. |
| **Owner** | Project Lead |

---

### R9: Key-Person Risk (Project Lead)

| Attribute | Value |
|---|---|
| **Category** | Resource |
| **Probability** | Low (1) |
| **Impact** | High (3) |
| **Risk Rating** | **3 (Low)** |
| **Description** | The Project Lead holds the actuarial knowledge and BMA rule interpretations. If they leave or are unavailable for an extended period, the project loses its actuarial anchor. |
| **Timeline Impact** | +2-6 months (finding and onboarding replacement) |
| **Trigger** | Project Lead unavailable for >4 weeks |
| **Mitigation** | 1. @rule_ref decorator documents every interpretation in code. 2. Rule traceability matrix provides a complete map. 3. "Regulatory Interpretation Register" documents judgment calls. 4. If Option C team is chosen, the second actuary provides redundancy. 5. Jupyter notebooks serve as executable knowledge transfer. |
| **Contingency** | The second actuary (if hired) can serve as interim project lead. Otherwise, engage an external actuarial consultant with BMA SBA experience. |
| **Owner** | BMA Senior Management |

---

### R10: Performance at Scale

| Attribute | Value |
|---|---|
| **Category** | Technical |
| **Probability** | Low (1) |
| **Impact** | Low (1) |
| **Risk Rating** | **1 (Low)** |
| **Description** | Large portfolios (5,000+ assets) with long projection horizons (50+ years) across 9 scenarios might be slow. |
| **Timeline Impact** | +1-2 weeks if optimization needed |
| **Trigger** | A single company run takes >30 minutes |
| **Mitigation** | 1. Profile early (Phase 3). 2. The 9 scenarios can run in parallel (ProcessPoolExecutor). 3. numpy vectorization for cash flow aggregation. 4. Realistic estimate: 9 scenarios x 50 years x 1000 assets = 450K calculations. pandas handles this in seconds. |
| **Contingency** | If truly performance-bound, pre-compute and cache scenario curves; vectorize the projection loop with numpy arrays instead of DataFrames. |
| **Owner** | Senior Python Developer |

---

### R11: Callable Bond Hull-White Instability

| Attribute | Value |
|---|---|
| **Category** | Technical |
| **Probability** | Medium (2) |
| **Impact** | Low (1) |
| **Risk Rating** | **2 (Low)** |
| **Description** | QuantLib's `TreeCallableFixedRateBondEngine` (Hull-White one-factor model) can be numerically unstable for certain parameter combinations, producing NaN or unreasonable prices. |
| **Timeline Impact** | +1-2 weeks |
| **Trigger** | Callable bond pricing produces NaN or prices that fail cross-validation |
| **Mitigation** | 1. Test with a range of callable bond parameters early in Phase 3. 2. Have an OAS-based fallback: price callable bonds as bullet bonds with an OAS spread adjustment. 3. For a benchmark model, the simpler OAS approach may actually be preferable (more transparent). |
| **Contingency** | Use OAS-based pricing for callable bonds throughout. Document the simplification. |
| **Owner** | Senior Python Developer |

---

## 3. Risk Heat Map

```
                    IMPACT
                Low (1)    Medium (2)    High (3)
           ┌──────────┬────────────┬────────────┐
High (3)   │  R6      │  R1        │  R2        │
           │ Copilot  │ QuantLib   │ BMA Rule   │
           │          │ Learning   │ Ambiguity  │
           ├──────────┼────────────┼────────────┤
Medium (2) │  R11     │  R8        │  R3, R4, R5│
           │ Callable │  Data      │  MBS, Hire,│
           │          │  Delays    │  Scope     │
           ├──────────┼────────────┼────────────┤
Low (1)    │  R10     │  R7        │  R9        │
           │ Perform  │  Windows   │  Key Person│
           │          │  Install   │            │
           └──────────┴────────────┴────────────┘

PRIORITY: Address top-right corner first (R2, R3, R4, R5)
```

---

## 4. Top 5 Risks by Priority

| Rank | Risk | Rating | First Action |
|---|---|---|---|
| 1 | **R2: BMA Rule Ambiguity** | 7.5 | Start documenting interpretation decisions in Phase 0; flag known ambiguities to BMA Prudential Standards immediately |
| 2 | **R4: Senior Dev Hiring** | 7.5 | Begin recruiting immediately; widen search to remote candidates |
| 3 | **R1: QuantLib Learning Curve** | 6 | Phase 0 bootcamp; hire Senior Dev with financial library experience |
| 4 | **R3: MBS/ABS Complexity** | 6 | Start with simplified model; defer full waterfall to Phase 2 release |
| 5 | **R5: Scope Creep** | 6 | Enforce Project Charter scope boundaries; maintain "future enhancements" backlog |

---

## 5. Risk Review Schedule

| Frequency | Activity | Participants |
|---|---|---|
| Weekly (during development) | Review active risks in standup | Development team |
| Monthly | Full risk register review | Project Lead + BMA Senior Management |
| At each phase gate | Risk re-assessment before proceeding | All stakeholders |
| Quarterly | Independent risk review | BMA internal audit (if applicable) |
