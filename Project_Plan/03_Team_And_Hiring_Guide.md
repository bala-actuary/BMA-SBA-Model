# BMA SBA Benchmark Model - Team & Hiring Guide

## Document Control

| Field | Value |
|---|---|
| **Project Name** | BMA SBA Benchmark Model |
| **Document Type** | Team Composition & Hiring Guide |
| **Version** | 1.0 (Draft for Discussion) |
| **Date** | March 2026 |

---

## 1. Team Philosophy

A regulatory benchmark model's value lies in **transparency and coherence** - not engineering sophistication. Unlike a commercial software product where more developers means faster delivery, a benchmark model benefits from:

- **Fewer people who each understand the whole model** over many specialists with fragmented knowledge
- **Actuarial-led development** where the person who understands the BMA rules also understands (or directly guides) the code
- **Independent validation** built in from the start, not bolted on at the end

The project lead (actuarial owner) will be the single most important team member. The rest of the team translates their actuarial knowledge into production-quality Python code.

---

## 2. Team Options

### Option A: Lean Team (3 people) - Recommended for Quality

**Best for:** Maximum auditability, long-term maintainability, budget-conscious

| Role | Type | Commitment | Module Ownership |
|---|---|---|---|
| **Project Lead (Actuarial Owner)** | Existing BMA staff | Full-time | All actuarial logic, rule interpretation, acceptance testing, model governance |
| **Senior Python Developer** | New hire or contract | Full-time | `curves/`, `projection/`, `engine/`, architecture, code reviews |
| **Python Developer (Mid-level)** | New hire or contract | Full-time | `model_points/`, `assumptions/`, `output/`, `challenge/`, test writing |

| Metric | Value |
|---|---|
| **Timeline** | 10-12 months |
| **Relative Cost** | Low |
| **Coordination Overhead** | Minimal (3-person standup) |
| **Knowledge Concentration** | High (everyone knows everything) |
| **Key Risk** | Single point of failure on Senior Dev |

### Option B: Moderate Team (4-5 people) - Balanced

**Best for:** Balanced speed and quality, allows parallel workstreams

| Role | Type | Commitment | Module Ownership |
|---|---|---|---|
| **Project Lead (Actuarial Owner)** | Existing BMA staff | Full-time | Actuarial logic, rules, acceptance, governance |
| **Senior Python Developer** | New hire or contract | Full-time | `curves/`, `engine/`, architecture, QuantLib integration |
| **Python Developer (Quant)** | New hire or contract | Full-time | `projection/`, `calculations/`, asset modeling |
| **Python Developer (Data/Reporting)** | New hire or contract | Full-time | `model_points/`, `assumptions/`, `output/`, `challenge/` |
| **QA / Test Engineer** | New hire or contract | Part-time (50%) | Test design, golden files, integration tests, CI |

| Metric | Value |
|---|---|
| **Timeline** | 7-9 months |
| **Relative Cost** | Medium |
| **Coordination Overhead** | Moderate (daily standups, sprint planning) |
| **Knowledge Concentration** | Medium (each person owns 2-3 modules) |
| **Key Risk** | Knowledge fragmentation; strong code review needed |

### Option C: Full Team (6 people) - Fastest

**Best for:** Hard deadline, budget available, highest quality assurance

| Role | Type | Commitment | Module Ownership |
|---|---|---|---|
| **Project Lead (Actuarial Owner)** | Existing BMA staff | Full-time | Project direction, rule interpretation, governance |
| **Senior Python Developer** | New hire or contract | Full-time | `curves/`, `engine/`, architecture |
| **Python Developer (Quant)** | New hire or contract | Full-time | `projection/`, asset modeling |
| **Python Developer (Data/Reporting)** | New hire or contract | Full-time | `model_points/`, `assumptions/`, `output/`, `challenge/` |
| **Second Actuary** | New hire or internal transfer | Full-time | `calculations/`, `rules/`, stress tests, independent logic review |
| **QA / Test Engineer** | New hire or contract | Full-time | All testing, CI/CD, regression suites |

| Metric | Value |
|---|---|
| **Timeline** | 5-7 months |
| **Relative Cost** | High |
| **Coordination Overhead** | Significant (need sprint planning, clear ownership boundaries) |
| **Knowledge Concentration** | Lower (mitigation: pair programming, documentation) |
| **Key Advantage** | Second actuary provides independent four-eyes on ALL actuarial logic |

---

## 3. Role Descriptions and Skill Requirements

### 3.1 Project Lead / Actuarial Owner (You)

**This is not a traditional project manager role.** You are the actuarial brain of the model and the ultimate authority on whether the code correctly implements BMA rules.

**Day-to-Day Responsibilities:**
- Translate BMA rules into precise specifications for the development team
- Review every piece of actuarial logic in the code for correctness
- Make judgment calls on ambiguous BMA rule interpretations (and document them)
- Write and review test cases that validate regulatory calculations
- Own the rule traceability matrix (every calculation maps to a BMA rule)
- Present model to BMA senior management and stakeholders

**Skills You Already Have:**
- Deep actuarial domain knowledge
- Prophet experience (model points, assumption tables, projection bases)
- Understanding of BMA SBA rules

**Skills to Develop (Before or During Phase 0):**
- Python basics: variables, functions, classes, modules, imports
- pandas fundamentals: DataFrames, read_csv, groupby, merge
- pytest basics: writing and running tests
- Git basics: commit, branch, pull request
- VS Code navigation and debugging

**Recommended Preparation:**
- 2-3 week Python crash course (many free online resources)
- Focus on pandas (your Prophet tabular data experience transfers directly)
- Install VS Code + Python extension + Copilot before the team starts

### 3.2 Senior Python Developer (CRITICAL HIRE)

**This is the make-or-break hire.** This person architects the codebase, writes the most complex modules, mentors the team on Python best practices, and serves as your Python translator.

**Key Responsibilities:**
- Design and implement the core architecture (package structure, module boundaries)
- Implement `curves/` module (QuantLib integration with audit wrapper)
- Implement `engine/` module (run orchestration, audit trail)
- Code review all pull requests
- Mentor you and the mid-level developer on Python patterns
- Debug QuantLib issues (these can be cryptic)
- Set up CI/CD (GitHub Actions), testing infrastructure

**Required Skills:**

| Skill | Level | Why |
|---|---|---|
| Python | Expert (5+ years production) | Architects the entire system |
| Financial libraries | Strong (QuantLib, RiskEngine, or similar) | QuantLib integration is the technical bottleneck |
| Fixed-income math | Working knowledge | Must understand yield curves, bond pricing, day counts |
| pandas / numpy | Expert | Core data manipulation |
| pytest | Strong | Designs the test infrastructure |
| Git / GitHub | Strong | Code review, branching strategy, CI/CD |
| Software architecture | Strong | Package design, dependency management, SOLID principles |

**Nice-to-Have Skills:**
- Actuarial or insurance domain knowledge
- Experience with regulatory/compliance systems
- Pydantic experience
- Experience mentoring junior developers

**Where to Find This Person:**
- Quantitative development teams at banks, hedge funds, or fintech companies
- Financial software companies (Bloomberg, Refinitiv, MSCI)
- Actuarial software companies (FIS/Prophet, Moody's/AXIS, Milliman)
- Open-source QuantLib contributors (check GitHub)

**Interview Focus Areas:**
1. "Build a yield curve from these swap rates using QuantLib-Python" (live coding)
2. "How would you structure a Python package for regulatory calculations that need audit trails?" (architecture discussion)
3. "Walk me through debugging a QuantLib pricing discrepancy" (problem-solving)
4. "How would you explain QuantLib's Handle pattern to someone new to the library?" (mentoring ability)

**Compensation Benchmark:**
This role commands a premium due to the QuantLib + financial engineering combination. Expect to pay 15-25% above a standard senior Python developer rate.

### 3.3 Python Developer - Quantitative Focus (Option B/C only)

**Key Responsibilities:**
- Implement `projection/` module (the core period-by-period engine)
- Implement `calculations/` module (BEL, stress tests, LCR)
- Asset modeling (all bond types, MBS/ABS)
- Numerical testing and validation

**Required Skills:**

| Skill | Level | Why |
|---|---|---|
| Python | Strong (3+ years) | Implements core calculation modules |
| Numerical computing | Strong (numpy, scipy) | Root-finding (brentq), financial math |
| pandas | Strong | Period-by-period cash flow manipulation |
| Fixed-income math | Working knowledge | Bond pricing, amortization schedules, duration |
| pytest | Competent | Writes unit and integration tests |
| Git | Competent | Daily workflow |

**Interview Focus:**
1. "Implement a function that finds the initial cash buffer C0 such that cash never goes negative over a 10-year projection" (numerical problem-solving)
2. "Write a pandas operation that generates a period-by-period cash flow table from a list of bonds" (pandas proficiency)

### 3.4 Python Developer - Data & Reporting Focus

**Key Responsibilities:**
- Implement `model_points/` module (data loading, Pydantic schemas, validation)
- Implement `assumptions/` module (assumption loading, versioning)
- Implement `output/` module (Excel report generation)
- Implement `challenge/` module (company comparison)
- Data pipeline quality and error handling

**Required Skills:**

| Skill | Level | Why |
|---|---|---|
| Python | Competent (2+ years) | Data pipeline and reporting code |
| pandas | Strong | Data loading, transformation, export |
| Pydantic | Competent (can learn on the job) | Schema validation |
| openpyxl | Competent (can learn on the job) | Excel workbook generation |
| Excel/CSV data handling | Strong | Parsing real-world messy data files |
| pytest | Competent | Test writing |

**Interview Focus:**
1. "Given this messy CSV file, write a pandas pipeline that validates, cleans, and normalizes the data" (real-world data handling)
2. "Create an openpyxl workbook with formatted headers, conditional formatting, and multiple tabs" (Excel generation)

### 3.5 Second Actuary (Option C only)

**Key Responsibilities:**
- Own `calculations/` module: stress tests, LCR, risk margin
- Independently review all actuarial logic implemented by the Quant Developer
- Write actuarial test cases (the "would an actuary agree with this number?" tests)
- Co-own the rule traceability matrix with the Project Lead
- Serve as the model's built-in independent validator

**Required Skills:**

| Skill | Level | Why |
|---|---|---|
| Actuarial qualification | Fellow or near-Fellow | Regulatory credibility, independent judgment |
| BMA regulatory knowledge | Strong | Interprets rules independently of Project Lead |
| Python | Basic to Competent | Enough to read code and write tests (not architect) |
| Prophet or similar | Preferred | Shares mental model with Project Lead |

**Why This Role is Transformative:**
Without a second actuary, the model's actuarial correctness depends entirely on the Project Lead. With one, every piece of actuarial logic has independent four-eyes review - which is exactly what regulators expect of regulatory models (per SR 11-7 and BMA's own model validation requirements).

### 3.6 QA / Test Engineer

**Key Responsibilities:**
- Design and maintain the test suite
- Write integration tests (end-to-end scenario runs)
- Maintain golden file regression tests
- Set up and maintain CI/CD pipeline (GitHub Actions)
- Performance testing for large portfolios

**Required Skills:**

| Skill | Level | Why |
|---|---|---|
| pytest | Strong | Core testing framework |
| Python | Competent | Writing test code |
| Numerical testing | Competent | Float comparison, tolerance-based assertions |
| CI/CD (GitHub Actions) | Competent | Automated test pipeline |
| Data analysis | Basic | Verifying large DataFrames against expected outputs |

---

## 4. Hiring Priority and Sequencing

Regardless of team size chosen, hire in this order:

```
Month 0          Month 1          Month 2          Month 3
   │                │                │                │
   v                v                v                v
┌─────────┐  ┌──────────┐  ┌──────────────┐  ┌───────────────┐
│ Project  │  │ Senior   │  │ Mid-level    │  │ QA + Second   │
│ Lead     │  │ Python   │  │ Python Dev   │  │ Actuary       │
│ ramps up │  │ Dev      │  │ (Data/Report │  │ (if Option    │
│ on       │  │ starts   │  │  OR Quant)   │  │  B or C)      │
│ Python   │  │          │  │              │  │               │
└─────────┘  └──────────┘  └──────────────┘  └───────────────┘
     │              │              │                  │
     v              v              v                  v
  Learning      Phase 0:       Joins for          Joins for
  pandas,       Foundation     Phase 1            Phase 2+
  pytest,       together
  Git
```

**Why this order?**
1. **You first** (Month 0): Spend 2-3 weeks learning Python basics before anyone else arrives. This way you can meaningfully review code from day one.
2. **Senior Dev second** (Month 1): You and the Senior Dev set up the project together in Phase 0. This is your most productive pairing - they learn the actuarial requirements, you learn Python patterns.
3. **Additional developers third** (Month 2): By now the architecture is in place and there are clear tasks to assign.
4. **QA and second actuary last** (Month 3): Testing infrastructure is meaningful only after there is code to test.

---

## 5. Engagement Models

### Full-Time Hires vs. Contractors

| Factor | Full-Time Hire | Contractor |
|---|---|---|
| **Institutional knowledge retention** | High - stays with BMA | Low - leaves when contract ends |
| **Cost flexibility** | Fixed | Variable (higher rate, shorter duration) |
| **Hiring speed** | Slow (4-8 weeks typical) | Fast (1-3 weeks) |
| **Long-term maintenance** | Built-in | Must re-engage or train replacement |
| **IP/Security** | Simpler governance | Requires clear contractual terms |

**Recommendation:**
- **Senior Python Developer**: Full-time hire preferred (institutional knowledge is critical for long-term maintenance). If not possible, 12-month minimum contract with knowledge transfer period.
- **Mid-level developers**: Either model works. Contract is fine for initial build if a full-time hire maintains the code afterwards.
- **Second Actuary**: Full-time hire or internal transfer preferred (actuarial judgment should stay in-house).
- **QA Engineer**: Can be contract.

---

## 6. Team Working Model

### Daily Workflow
- **Daily standup** (15 min): What I did, what I'm doing, any blockers
- **Code review**: Every pull request reviewed by at least one other person. Actuarial logic PRs also reviewed by the Project Lead.
- **Weekly demo**: Show working functionality to the team (keeps momentum visible)

### Development Practices
- **Git branching**: Feature branches off `main`, PRs for all changes
- **Test-driven**: Write failing test first (especially for BMA rule implementations)
- **Pair programming**: Senior Dev + Project Lead pair frequently in early phases
- **Jupyter for exploration**: Try things in notebooks first, then move to modules

### Communication
- Team of 3: In-person / ad-hoc is sufficient
- Team of 4-6: Daily standup + shared task board (GitHub Projects or simple Kanban)

---

## 7. Knowledge Transfer Plan

### Risk: What if the Senior Developer leaves?

| Mitigation | Details |
|---|---|
| **Architecture documentation** | `02_Architecture_Design.md` ensures the design is not in anyone's head |
| **Code comments on "why"** | Every non-obvious decision has a code comment explaining the reasoning |
| **Jupyter notebooks** | The 5 notebooks serve as executable documentation and tutorials |
| **Rule traceability matrix** | Maps every function to its BMA rule - any actuary can verify correctness |
| **Pair programming** | Project Lead pairs with Senior Dev regularly, building your own Python competence |
| **Comprehensive test suite** | If all tests pass, the model is correct - regardless of who maintains it |

### Risk: What if the Project Lead (you) rotates?

| Mitigation | Details |
|---|---|
| **Rule traceability matrix** | The next actuary can trace every calculation to BMA rules |
| **@rule_ref decorators** | Every function is tagged with its regulatory basis |
| **Assumption governance doc** | Documents why each assumption was chosen |
| **Test suite with BMA illustrative calc** | The golden test proves the model works correctly |
| **Second actuary** (if hired) | Provides continuity and independent knowledge |

---

## 8. Cost Estimation Framework

Exact salaries depend on BMA's pay scales and the Bermuda market. Below is a relative framework:

| Role | Relative Cost (Annual) | Notes |
|---|---|---|
| Project Lead | Existing staff | No incremental cost |
| Senior Python Developer (QuantLib) | $$$$$ | Premium for financial library expertise |
| Python Developer (Quant) | $$$$ | Standard senior developer rate |
| Python Developer (Data/Reporting) | $$$ | Standard mid-level developer rate |
| Second Actuary | $$$$ | Qualified actuary rate |
| QA / Test Engineer | $$$ | Standard QA rate |

### Total Cost by Option (10-12 month project)

| Team Option | Headcount | Relative Annual Cost |
|---|---|---|
| A: Lean | 2 new (+ you) | $$$$$$$$ |
| B: Moderate | 3-4 new (+ you) | $$$$$$$$$$$$ |
| C: Full | 5 new (+ you) | $$$$$$$$$$$$$$$$ |

**Note:** Option A may appear cheapest, but takes 3-5 months longer than Option C. Factor in the opportunity cost of delayed regulatory capability when comparing.

---

## 9. Decision Matrix

| Factor | Weight | Option A (3) | Option B (4-5) | Option C (6) |
|---|---|---|---|---|
| Timeline | 25% | 6/10 (10-12 mo) | 8/10 (7-9 mo) | 10/10 (5-7 mo) |
| Cost | 20% | 10/10 | 7/10 | 4/10 |
| Quality / Auditability | 25% | 9/10 | 8/10 | 9/10 (2nd actuary) |
| Long-term Maintainability | 15% | 10/10 | 7/10 | 6/10 |
| Key-person Risk | 15% | 4/10 (1 dev) | 7/10 | 9/10 |
| **Weighted Score** | 100% | **7.85** | **7.55** | **7.65** |

**All three options are viable.** Option A scores highest on quality and cost but lowest on timeline and key-person risk. Option C is fastest and most resilient but most expensive. Option B is a solid middle ground.

**Recommendation:** Start with **Option A** (3 people). If timeline pressure increases or if the Senior Developer position proves hard to fill, expand to Option B by adding a Quant Developer.
