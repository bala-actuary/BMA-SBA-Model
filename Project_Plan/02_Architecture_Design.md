# BMA SBA Benchmark Model - Architecture Design Document

## Document Control

| Field | Value |
|---|---|
| **Project Name** | BMA SBA Benchmark Model |
| **Document Type** | Architecture Design Document |
| **Version** | 1.0 (Draft for Discussion) |
| **Date** | March 2026 |

---

## 1. Architecture Overview

The BMA SBA Benchmark Model is a **Python desktop calculation engine** - not a web application, not a microservices platform, and not a database-backed system. It is designed to be:

- **Transparent**: Every calculation readable by an actuary, every number traceable to a BMA rule
- **Auditable**: Complete audit trail of every run, every decision, every cash flow
- **Reproducible**: Same inputs always produce same outputs; all inputs are versioned and hashed
- **Portable**: Runs on a standard Windows workstation with no infrastructure dependencies

### Design Principles

| Principle | Meaning | Why It Matters for a Benchmark Model |
|---|---|---|
| **Transparency over Sophistication** | Prefer readable Python over optimized C++ | Auditors and actuaries must be able to read and verify every calculation |
| **Separation of Concerns** | Data, assumptions, logic, and output are in separate modules | Mirrors Prophet's architecture; enables independent versioning of rules and assumptions |
| **Rule Traceability** | Every calculation annotated with its BMA rule reference | The defining feature of a benchmark model - every number has a regulatory basis |
| **Immutable Runs** | Once a run starts, all inputs are snapshotted | Ensures reproducibility; enables comparison of runs months apart |
| **Fail Loudly** | Invalid data stops the run with a clear message | No silent defaults or implicit assumptions that could mislead a regulatory review |
| **Everything is a DataFrame** | Intermediate results are pandas DataFrames | Inspectable in notebooks, exportable to CSV, testable in pytest |

---

## 2. Prophet-to-Python Concept Mapping

For team members with Prophet experience, this table shows how familiar Prophet concepts map to the Python architecture:

| Prophet Concept | Description in Prophet | Python Equivalent | Location |
|---|---|---|---|
| **Model Points** | CSV files of policy/asset records loaded via DCS | pandas DataFrames validated by Pydantic schemas | `model_points/` |
| **Assumption Tables** | Managed in Assumptions Manager; versioned, controlled | YAML/CSV files with version metadata; loaded into typed Python objects | `assumptions/` |
| **Projection Basis** | The calculation logic (formulas, conditions) | Pure Python functions that take model points + assumptions and return results | `projection/` |
| **Product** | A liability type (e.g., annuity, term life) | A liability block configuration | `model_points/liabilities.py` |
| **Run** | An execution: specific model points + assumptions + basis | `RunConfig` dataclass + orchestrator | `engine/` |
| **Global Variables** | Valuation date, run ID, etc. | `RunConfig` fields: `company_id`, `valuation_date`, `rule_year` | `engine/run_config.py` |
| **Output Tables** | Result arrays written to database/files | pandas DataFrames written to Excel via openpyxl | `output/` |
| **Run Comparison** | Comparing two runs side-by-side | `challenge/compare.py` with tolerance-based flagging | `challenge/` |
| **Assumptions Manager** | Controlled environment for assumption governance | Git version control + `manifest.yaml` metadata | `assumptions/tables/` |

---

## 3. Package Structure

```
bma_sba_benchmark/
│
├── pyproject.toml                     # Package metadata, dependencies
├── README.md                          # Getting started guide
│
├── bma_sba_benchmark/                 # ======= MAIN PACKAGE =======
│   │
│   ├── rules/                         # [1] BMA RULES (versioned, traceable)
│   │   ├── registry.py                #     Maps rule_year -> rule module
│   │   ├── v2024/                     #     Rules effective 2024 year-end
│   │   │   ├── scenarios.py           #       9 scenario definitions
│   │   │   ├── asset_tiers.py         #       Tier 1/2/3 classification
│   │   │   ├── credit_costs.py        #       Default/downgrade tables
│   │   │   ├── transaction_costs.py   #       Bid-ask spread grading
│   │   │   ├── stress_tests.py        #       3 mandatory stress tests
│   │   │   ├── lcr.py                 #       LCR rules (105% threshold)
│   │   │   ├── risk_margin.py         #       6% CoC parameters
│   │   │   ├── reinvestment.py        #       Reinvestment/disinvestment rules
│   │   │   └── parameters.py          #       All numeric params in one place
│   │   └── v2025/                     #     Year 2 overrides only
│   │       └── credit_costs.py        #       40% phase-in (was 20%)
│   │
│   ├── model_points/                  # [2] DATA LAYER
│   │   ├── schemas.py                 #     Pydantic schemas for all inputs
│   │   ├── assets.py                  #     Asset model point loader
│   │   ├── liabilities.py             #     EPL loader
│   │   ├── portfolio.py               #     Combined portfolio + validation
│   │   └── company_submission.py      #     Company SBA submission parser
│   │
│   ├── assumptions/                   # [3] ASSUMPTIONS LAYER
│   │   ├── loader.py                  #     Load assumption sets
│   │   ├── economic.py                #     Risk-free curves, spreads
│   │   ├── credit.py                  #     Default rates, transitions
│   │   ├── reinvestment.py            #     Reinvestment strategy params
│   │   └── tables/                    #     Versioned data files
│   │       ├── bma_rfr_2024Q4.csv
│   │       ├── bma_credit_floors.csv
│   │       ├── spread_widening.csv
│   │       └── manifest.yaml
│   │
│   ├── curves/                        # [4] QUANTLIB LAYER (surgical use)
│   │   ├── yield_curve.py             #     ql.PiecewiseYieldCurve
│   │   ├── scenario_curves.py         #     Apply 9 scenario shifts
│   │   ├── bond_pricing.py            #     ql.FixedRateBond etc.
│   │   ├── mbs_pricing.py             #     MBS/ABS CF generation
│   │   └── ql_audit.py                #     Audit wrapper for all QL calls
│   │
│   ├── projection/                    # [5] PROJECTION LOGIC (pure Python)
│   │   ├── asset_cf.py                #     Asset cash flows per period
│   │   ├── liability_cf.py            #     EPL cash flows per period
│   │   ├── net_cf.py                  #     Net CF = asset - liability
│   │   ├── reinvestment.py            #     Reinvest excess cash
│   │   ├── disinvestment.py           #     Liquidation waterfall
│   │   ├── credit_adjustment.py       #     Default/downgrade costs
│   │   ├── transaction_cost.py        #     Bid-ask / transaction costs
│   │   ├── scenario_runner.py         #     One scenario end-to-end
│   │   └── multi_scenario.py          #     All 9 + biting scenario
│   │
│   ├── calculations/                  # [6] REGULATORY CALCULATIONS
│   │   ├── bel.py                     #     Best Estimate Liability
│   │   ├── risk_margin.py             #     Risk Margin
│   │   ├── technical_provision.py     #     TP = BEL + RM
│   │   ├── lcr.py                     #     Liquidity Coverage Ratio
│   │   ├── stress_tests.py            #     3 mandatory stress tests
│   │   └── lapse_cost.py              #     Lapse cost adjustment
│   │
│   ├── engine/                        # [7] RUN ORCHESTRATION
│   │   ├── run_config.py              #     RunConfig dataclass
│   │   ├── runner.py                  #     Main orchestrator
│   │   ├── batch.py                   #     Multi-company batch runner
│   │   └── audit_trail.py             #     SQLite + Parquet + JSON logs
│   │
│   ├── challenge/                     # [8] COMPANY COMPARISON
│   │   ├── import_submission.py       #     Parse company workbook
│   │   ├── compare.py                 #     Metric-by-metric comparison
│   │   ├── variance_analysis.py       #     CF-level drill-down
│   │   └── challenge_report.py        #     Challenge report generator
│   │
│   ├── output/                        # [9] REPORTING
│   │   ├── excel_writer.py            #     Multi-tab Excel workbook
│   │   ├── summary.py                 #     Summary dashboard
│   │   ├── detail.py                  #     Period-by-period detail
│   │   ├── audit_sheet.py             #     Audit trail sheet
│   │   └── comparison_sheet.py        #     Benchmark vs company
│   │
│   └── utils/                         # [10] SHARED UTILITIES
│       ├── rule_ref.py                #     @rule_ref decorator
│       ├── dates.py                   #     Date helpers
│       └── validation.py              #     Common validators
│
├── tests/                             # ======= TEST SUITE =======
│   ├── test_illustrative_calc.py      #     Golden test (BMA example)
│   ├── test_scenarios.py
│   ├── test_asset_pricing.py
│   ├── test_projection.py
│   ├── test_credit_costs.py
│   ├── test_bel.py
│   ├── test_lcr.py
│   ├── test_stress_tests.py
│   ├── test_rule_versioning.py
│   ├── test_audit_trail.py
│   └── fixtures/
│
├── notebooks/                         # ======= JUPYTER NOTEBOOKS =======
│   ├── 01_illustrative_calculation.ipynb
│   ├── 02_quantlib_curves.ipynb
│   ├── 03_scenario_visualization.ipynb
│   ├── 04_projection_walkthrough.ipynb
│   └── 05_company_comparison.ipynb
│
└── docs/                              # ======= DOCUMENTATION =======
    ├── architecture.md
    ├── rule_traceability_matrix.md
    ├── user_guide.md
    └── assumption_governance.md
```

---

## 4. Data Flow

### 4.1 High-Level Data Flow

```
 ┌─────────────────────────────────────────────────────────────────┐
 │                          INPUTS                                 │
 ├─────────────────┬──────────────────┬────────────────────────────┤
 │  Model Points   │   Assumptions    │      Run Config            │
 │  - Assets.csv   │   - BMA RFR      │      - Company ID          │
 │  - EPL.csv      │   - Credit floors│      - Valuation date      │
 │                 │   - Spreads      │      - Rule year (2024)    │
 └────────┬────────┴────────┬─────────┴──────────────┬─────────────┘
          │                 │                        │
          v                 v                        v
 ┌─────────────────────────────────────────────────────────────────┐
 │                    VALIDATION LAYER                             │
 │  model_points/schemas.py + validators                          │
 │  - Pydantic schema validation                                  │
 │  - BMA asset eligibility checks (Tier 1/2/3)                   │
 │  - 10% limited-basis cap enforcement                           │
 │  - Currency matching verification                              │
 │  *** FAIL LOUDLY if invalid ***                                │
 └────────────────────────────┬────────────────────────────────────┘
                              │
                              v
 ┌─────────────────────────────────────────────────────────────────┐
 │                    CURVE CONSTRUCTION                           │
 │  curves/yield_curve.py + curves/scenario_curves.py             │
 │  *** QUANTLIB USED HERE ***                                    │
 │                                                                 │
 │  Base curve (ql.PiecewiseYieldCurve)                           │
 │       │                                                         │
 │       ├─> Scenario 1: Base (no shift)                          │
 │       ├─> Scenario 2: Decrease (-150bps by Y10)                │
 │       ├─> Scenario 3: Increase (+150bps by Y10)                │
 │       ├─> Scenario 4: Down-Up (-150 Y5, return Y10)            │
 │       ├─> Scenario 5: Up-Down (+150 Y5, return Y10)            │
 │       ├─> Scenario 6: Decrease + Positive Twist                │
 │       ├─> Scenario 7: Decrease + Negative Twist                │
 │       ├─> Scenario 8: Increase + Positive Twist                │
 │       └─> Scenario 9: Increase + Negative Twist                │
 │                                                                 │
 │  All calls wrapped by ql_audit.py (inputs/outputs logged)      │
 └────────────────────────────┬────────────────────────────────────┘
                              │
                              v
 ┌─────────────────────────────────────────────────────────────────┐
 │               PROJECTION ENGINE (per scenario)                  │
 │  projection/ *** PURE PYTHON - NO QUANTLIB ***                 │
 │                                                                 │
 │  For t = 1, 2, ..., T_max:                                     │
 │    1. asset_cf[t] = contractual CFs due this period             │
 │    2. credit_adj[t] = default_cost + downgrade_cost  [@rule_ref]│
 │    3. adjusted_cf[t] = asset_cf[t] - credit_adj[t]             │
 │    4. net_cf[t] = adjusted_cf[t] - epl[t]                      │
 │    5. IF net_cf > 0: reinvest(excess)                [@rule_ref]│
 │       IF net_cf < 0: liquidate(shortfall)            [@rule_ref]│
 │    6. transaction_cost[t] on any trades              [@rule_ref]│
 │    7. Update portfolio state                                    │
 │    8. Log: balances, actions, rule refs -> audit trail          │
 │                                                                 │
 │  Solve for C0: scipy.optimize.brentq                           │
 │    (find initial cash where min balance across all t = 0)       │
 └────────────────────────────┬────────────────────────────────────┘
                              │
                              v
 ┌─────────────────────────────────────────────────────────────────┐
 │              BITING SCENARIO + REGULATORY CALCS                 │
 │                                                                 │
 │  Biting = scenario requiring max(C0) across all 9              │
 │  BEL = MV_assets(T=0) + C0_biting                              │
 │  RM = 6% × Σ(Modified ECR_t / (1+r)^t)                        │
 │  TP = BEL + RM                                                 │
 │  LCR = Available Liquidity / Potential Surrender >= 105%       │
 │  Stress Tests: mass lapse+spread, 1-notch, no reinvest         │
 └────────────────────────────┬────────────────────────────────────┘
                              │
                              v
 ┌───────────────────────────────────────┬─────────────────────────┐
 │           OUTPUTS                     │     AUDIT TRAIL         │
 │                                       │                         │
 │  Excel Workbook:                      │  SQLite: run metadata   │
 │   Tab 1: Summary Dashboard            │  Parquet: period CFs    │
 │   Tab 2: BEL by Scenario              │  JSON: rule traces      │
 │   Tab 3: Cash Flow Detail             │                         │
 │   Tab 4: ALM Metrics                  │  All files portable,    │
 │   Tab 5: Stress Test Results          │  archivable, and        │
 │   Tab 6: LCR Results                  │  inspectable without    │
 │   Tab 7: Audit Summary                │  special software       │
 │   Tab 8: Company Comparison*          │                         │
 └───────────────────────────────────────┴─────────────────────────┘
 * Only if company submission provided
```

### 4.2 QuantLib Boundary

QuantLib is used **only** in the `curves/` module. This is a deliberate design choice:

```
┌───────────────────────────────────────────────────────────────┐
│                    PURE PYTHON ZONE                           │
│  (100% transparent, readable by any actuary)                 │
│                                                               │
│  model_points/  assumptions/  projection/  calculations/     │
│  engine/        challenge/    output/      rules/            │
│                                                               │
│         ┌──────────────────────────────────┐                 │
│         │      QUANTLIB ZONE               │                 │
│         │  (industry-standard instrument   │                 │
│         │   math, wrapped with audit log)  │                 │
│         │                                  │                 │
│         │  curves/yield_curve.py           │                 │
│         │  curves/scenario_curves.py       │                 │
│         │  curves/bond_pricing.py          │                 │
│         │  curves/mbs_pricing.py           │                 │
│         │  curves/ql_audit.py <-- logs     │                 │
│         │         all I/O                  │                 │
│         └──────────────────────────────────┘                 │
│                                                               │
│  QuantLib provides: discount factors, bond prices, forward   │
│  rates. The projection/ module uses these as INPUTS, but     │
│  all decision logic (reinvest, liquidate, apply credit       │
│  costs) is in pure Python.                                   │
└───────────────────────────────────────────────────────────────┘
```

**Why this boundary?**
- Projection logic is where BMA rules live. It MUST be transparent.
- Bond pricing math (day counts, compounding, curve interpolation) is standardized. Using a recognized library adds credibility and avoids reimplementing well-known formulas.
- The audit wrapper (`ql_audit.py`) ensures that even QuantLib's inputs/outputs are logged, so an auditor can verify what the library was asked and what it returned.

---

## 5. Rule Traceability System

### 5.1 The `@rule_ref` Decorator

Every function that implements a BMA rule carries a decorator:

```python
# utils/rule_ref.py

@dataclass
class RuleReference:
    source: str          # "Rules" or "Handbook"
    section: str         # e.g., "Sched XXV Para 28(7)(a-i)"
    page: str = ""       # e.g., "p. 159"
    description: str = "" # Human-readable summary

def rule_ref(*refs: RuleReference):
    """Decorator: tags function with BMA rule references,
    logs every invocation to the audit trail."""
    ...
```

### 5.2 Usage Example

```python
# rules/v2024/scenarios.py

SCENARIO_DECREASE_RULE = RuleReference(
    source="Rules",
    section="Sched XXV Para 28(7)(b)",
    description="Decrease scenario: rates grade to -150bps by year 10"
)

@rule_ref(SCENARIO_DECREASE_RULE)
def decrease_scenario_shift(projection_year: int) -> float:
    """Returns the bps shift for the Decrease scenario at a given year."""
    if projection_year >= 10:
        return -150.0
    return -15.0 * projection_year  # Linear grade
```

### 5.3 Rule Traceability Matrix

A living document (`docs/rule_traceability_matrix.md`) maps every BMA rule to its implementing function:

```
| BMA Rule                          | Module                        | Function                    |
|-----------------------------------|-------------------------------|-----------------------------|
| Sched XXV Para 28(7)(a)          | rules/v2024/scenarios.py      | base_scenario_shift()       |
| Sched XXV Para 28(7)(b)          | rules/v2024/scenarios.py      | decrease_scenario_shift()   |
| Sched XXV Para 28(9)(c-d)        | projection/disinvestment.py   | sell_assets()               |
| Sched XXV Para 28(13-16)         | rules/v2024/asset_tiers.py    | classify_tier()             |
| Sched XXV Para 28(22-26)         | rules/v2024/credit_costs.py   | apply_credit_costs()        |
| Handbook E5.6h(i)                | calculations/stress_tests.py  | combined_lapse_spread()     |
| Rules Para 29(2)(iii)            | calculations/lcr.py           | calculate_lcr()             |
| Rules Subpara 36(4)              | calculations/risk_margin.py   | calculate_risk_margin()     |
```

---

## 6. Rule Versioning System

### 6.1 How It Works

```
rules/
├── registry.py          # Selects rule version based on RunConfig.rule_year
├── v2024/               # Complete rule set for 2024
│   ├── scenarios.py
│   ├── credit_costs.py  # 20% downgrade phase-in
│   └── ...
└── v2025/               # ONLY overrides what changed
    └── credit_costs.py  # 40% downgrade phase-in (override)
```

The registry loads v2024 as the base, then applies v2025 overrides:

```python
# rules/registry.py
def get_rules(rule_year: int) -> RuleSet:
    """Load rules for a given year.
    v2025 inherits from v2024 and only overrides changed modules."""
    base = import_module("bma_sba_benchmark.rules.v2024")
    if rule_year >= 2025:
        overrides = import_module("bma_sba_benchmark.rules.v2025")
        # Override only the modules that exist in v2025
        ...
    return RuleSet(...)
```

### 6.2 Benefits
- **No code duplication**: v2025 only contains what changed
- **Side-by-side comparison**: Run the same portfolio under v2024 and v2025 rules to see the impact of rule changes
- **Audit-friendly**: `RunConfig.rule_year` is logged in every run; the exact rules used are unambiguous

---

## 7. Audit Trail Architecture

### 7.1 Three-Layer Design

| Layer | Storage | Contents | Use Case |
|---|---|---|---|
| **Run Metadata** | SQLite database | Run ID, company, date, inputs (hashed), BEL result, biting scenario | "What runs have we done? What were the results?" |
| **Period Cash Flows** | Parquet files | Period-by-period: opening/closing balances, asset CFs, liability CFs, reinvestment/liquidation actions, credit costs | "Show me the cash flow detail for Scenario 2, Year 5" |
| **Rule Traces** | JSON Lines files | Every `@rule_ref` invocation: function name, inputs, outputs, timestamp, rule references | "Which BMA rule produced this credit cost number?" |

### 7.2 Why These Technologies

- **SQLite**: Zero-infrastructure database. A single `.db` file. Readable by any SQL tool, Python, or even Excel (via ODBC). The most widely deployed database engine in the world.
- **Parquet**: Columnar format that stores DataFrames efficiently. A single file per scenario per run. Readable by pandas in one line: `pd.read_parquet("run_001_scenario_2.parquet")`.
- **JSON Lines**: One JSON object per line. Simple, human-readable, grep-able. Stores the rule trace log.

### 7.3 No Server Required
All three formats are **files**. No database server to install, configure, or maintain. An auditor can copy these files to their laptop, open them in Python/pandas/Excel, and verify any number.

---

## 8. Company Challenge Architecture

### 8.1 Challenge Workflow

```
Step 1: Import                Step 2: Benchmark Run         Step 3: Compare
┌─────────────────┐          ┌────────────────────┐       ┌─────────────────┐
│ Company's SBA    │          │ Run benchmark with │       │ Side-by-side    │
│ Submission       │ ──────>  │ company's portfolio│ ───>  │ comparison      │
│ (Excel workbook) │          │ + BMA assumptions  │       │ with tolerances │
└─────────────────┘          └────────────────────┘       └─────────────────┘
                                                                   │
                                                                   v
                                                          ┌─────────────────┐
                                                          │ Challenge Report│
                                                          │ (Excel workbook)│
                                                          │ - Pass/Fail     │
                                                          │ - Variances     │
                                                          │ - Rule refs     │
                                                          └─────────────────┘
```

### 8.2 Comparison Logic

```python
# challenge/compare.py
def compare(benchmark: RunResult, company: CompanySubmission) -> ComparisonReport:
    """Compare benchmark results against company submission.

    For each metric:
    - Calculate absolute and percentage difference
    - Apply configurable tolerance (e.g., 2% for BEL, 5% for credit costs)
    - Flag PASS/FAIL
    - Include rule reference explaining the benchmark calculation
    """
```

### 8.3 Multi-Company Batch

```python
# engine/batch.py
def batch_run(companies: List[CompanyPortfolio], run_config: RunConfig) -> BatchReport:
    """Run benchmark for multiple companies.
    Produces:
    - Individual company reports
    - Cross-company comparison dashboard
    - Industry-level summary statistics
    """
```

---

## 9. Key Design Patterns

| Pattern | Where | Why |
|---|---|---|
| **Strategy** | `projection/reinvestment.py` | Different reinvestment strategies (pro-rata, target-duration, cash-only) can be swapped without changing the projection engine |
| **Factory** | `curves/bond_pricing.py` | Maps asset type to the correct QuantLib instrument constructor |
| **Pure Functions** | All of `projection/` | Given same inputs, produce same outputs. No global state. Trivial to test. |
| **Decorator** | `utils/rule_ref.py` | Cross-cutting audit concern applied declaratively, not woven into business logic |
| **Immutable Run** | `engine/run_config.py` | RunConfig is frozen after creation. Inputs are hashed. Ensures reproducibility. |
| **Composition** | Throughout | Simple function composition, not deep class hierarchies. Approachable for Python newcomers. |

---

## 10. Error Handling Philosophy

### Fail Loudly at Boundaries

```python
# model_points/schemas.py (example)
class AssetModelPoint(BaseModel):
    asset_id: str
    asset_type: Literal["GOVT_BOND", "CORP_BOND", "CALLABLE", "FLOATING", "AMORTIZING", "MBS", "ABS"]
    par_value: float = Field(gt=0, description="Par/face value, must be positive")
    coupon_rate: float = Field(ge=0, le=1, description="Annual coupon rate as decimal")
    maturity_date: date
    rating: Literal["AAA", "AA+", "AA", "AA-", "A+", "A", "A-", "BBB+", "BBB", "BBB-",
                     "BB+", "BB", "BB-", "B+", "B", "B-", "CCC", "CC", "C", "D"]
    currency: str = Field(pattern=r"^[A-Z]{3}$")
    tier: Optional[Literal[1, 2, 3]] = None  # Computed by rules/asset_tiers.py
```

If a model point file contains an asset with `par_value = -100` or `rating = "XYZ"`, Pydantic raises a clear validation error before any calculation begins. No silent data quality issues.

### Trust Internal Code

Inside the package, functions trust their callers. Validation happens at the boundary (data loading), not at every function call. This keeps the code simple and fast.

---

## 11. Module Dependency Diagram

```
                    rules/
                      │
                      v
model_points/ ──> projection/ ──> calculations/ ──> engine/runner.py
                      ^                                    │
assumptions/ ─────────┘                                    │
                                                           v
curves/ ──> projection/ (for pricing during liquidation)  output/
                                                           │
                                              challenge/ <──┘
```

**Key constraints:**
- `projection/` depends on `rules/`, `model_points/`, `assumptions/`, and `curves/`
- `projection/` does NOT depend on `engine/`, `output/`, or `challenge/`
- `curves/` is the ONLY module that imports QuantLib
- `challenge/` depends on `output/` and `engine/` but NOT on `projection/` directly

This means the projection logic can be tested in complete isolation.

---

## 12. Deployment Model

### Target Environment
- Standard BMA Windows workstation
- Python 3.11+ installed via conda (recommended) or pip
- No internet access required at runtime
- No database server
- No Docker containers

### Installation
```bash
# One-time setup
conda create -n bma_sba python=3.11
conda activate bma_sba
pip install bma-sba-benchmark  # Or install from local wheel
```

### Running
```bash
# Single company run
python -m bma_sba_benchmark run --company ACME --date 2024-12-31 --rules 2024

# Multi-company batch
python -m bma_sba_benchmark batch --companies companies.csv --date 2024-12-31

# Company comparison
python -m bma_sba_benchmark challenge --company ACME --submission acme_sba_2024.xlsx
```

### Output
All output goes to a timestamped directory:
```
output/
└── 2024-12-31_ACME_run_001/
    ├── benchmark_report.xlsx       # Main Excel workbook
    ├── audit/
    │   ├── run_metadata.db         # SQLite
    │   ├── scenario_1.parquet      # Period-by-period CFs
    │   ├── scenario_2.parquet
    │   ├── ...
    │   └── rule_traces.jsonl       # Rule invocation log
    └── comparison_report.xlsx      # If company submission provided
```
