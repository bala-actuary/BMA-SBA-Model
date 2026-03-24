# Algorithm Specification 08: End-to-End Data Flow and Run Orchestration

## Document Control

| Field | Value |
|---|---|
| **Spec ID** | ALGO-008 |
| **Target Module** | `engine/run_orchestrator.py` |
| **Supporting Modules** | `engine/run_config.py`, `engine/batch.py`, `engine/audit_trail.py`, all upstream modules |
| **Status** | Implementation-Ready Draft |
| **Date** | March 2026 |
| **Cross-References** | [ALGO-001](01_Yield_Curve_Construction.md), [ALGO-002](02_Projection_Engine.md), [ALGO-002a](02a_Reinvestment_Strategy.md), [ALGO-002b](02b_Disinvestment_Waterfall.md), [ALGO-003](03_Credit_Costs_DD.md), [ALGO-004](04_Spread_Cap_Enforcement.md) |

---

## 1. Overview

### 1.1 Purpose

This specification defines the end-to-end data flow for the BMA SBA Benchmark Model -- from raw input files to the final regulatory Excel workbook and audit trail. It is the "map" that ties all other Algorithm Specs together.

The target module, `engine/run_orchestrator.py`, is the conductor. It does not perform any calculation itself. Instead, it sequences calls to `model_points/`, `assumptions/`, `curves/`, `projection/`, `calculations/`, `challenge/`, and `output/` in the correct order, managing state transitions, error handling, and audit logging throughout.

### 1.2 Scope

| In Scope | Out of Scope |
|---|---|
| Single-company run sequence (8 steps) | Internal algorithm details (see ALGO-001 through ALGO-004) |
| Batch mode (parallel multi-company) | QuantLib curve math (see ALGO-001) |
| Challenge mode (benchmark vs submission) | Projection loop internals (see ALGO-002) |
| Audit trail (SQLite + Parquet + JSON Lines) | Credit cost formulas (see ALGO-003) |
| Output workbook generation | Spread cap enforcement logic (see ALGO-004) |
| CLI interface (`run`, `batch`, `challenge`) | |
| Error handling and recovery | |

### 1.3 Design Principles

From the Architecture Design Document:

- **Immutable runs:** `RunConfig` frozen after creation; all inputs hashed for reproducibility.
- **Fail loudly:** Pydantic validation at every data boundary; no silent defaults.
- **Rule traceability:** Every calculation tagged with its BMA rule paragraph.
- **Everything is a DataFrame:** All intermediate results are pandas DataFrames.
- **QuantLib boundary:** Only `curves/` imports QuantLib; everything else is pure Python.

---

## 2. Module Dependency Graph

### 2.1 Package-Level Data Flow

```
                          CLI (click/typer)
                               |
                               v
                    +---------------------+
                    | engine/             |
                    |   run_orchestrator  |  <-- THIS MODULE
                    |   run_config        |      (conducts everything)
                    |   batch             |
                    |   audit_trail       |
                    +----+----+----+------+
                         |    |    |
            +------------+    |    +-------------+
            |                 |                  |
            v                 v                  v
   +----------------+  +-------------+  +----------------+
   | model_points/  |  | assumptions/|  | rules/         |
   |  assets.py     |  |  loader.py  |  |  registry.py   |
   |  liabilities.py|  |  economic.py|  |  v2024/        |
   |  schemas.py    |  |  credit.py  |  |  v2025/        |
   |  portfolio.py  |  |  tables/    |  +----------------+
   +-------+--------+  +------+------+
           |                  |
           +--------+---------+
                    |
                    v
           +----------------+
           | curves/         |          *** QUANTLIB BOUNDARY ***
           |  yield_curve.py |
           |  scenario_curves|
           |  bond_pricing   |
           |  ql_audit.py    |
           +-------+--------+
                   |
                   | (plain Python objects: arrays, dicts, callables)
                   | (NO QuantLib objects cross this boundary)
                   v
           +----------------+
           | projection/     |          *** PURE PYTHON ***
           |  cashflow_engine|  <-- ALGO-002
           |  reinvestment   |  <-- ALGO-002a
           |  disinvestment  |  <-- ALGO-002b
           |  credit_costs   |  <-- ALGO-003
           +-------+--------+
                   |
                   v
           +----------------+
           | calculations/   |
           |  bel_calculator |  <-- ALGO-004 (spread cap)
           |  risk_margin    |
           |  lcr            |
           |  stress_tests   |
           |  lapse_cost     |
           +-------+--------+
                   |
          +--------+--------+
          |                 |
          v                 v
   +-------------+  +----------------+
   | output/      |  | challenge/     |
   |  excel_writer|  |  compare.py    |
   |  summary     |  |  variance      |
   |  detail      |  |  challenge_rpt |
   |  audit_sheet |  +----------------+
   +-------------+
```

### 2.2 Data Types Crossing Module Boundaries

| Boundary | From | To | Data Type |
|---|---|---|---|
| Input files -> model_points | CSV/Excel files | `model_points/` | Raw file paths |
| model_points -> engine | `model_points/` | `engine/` | `List[AssetModelPoint]`, `List[EPLCashFlow]` (Pydantic-validated) |
| assumptions -> engine | `assumptions/` | `engine/` | `AssumptionSet` (YAML/CSV loaded into typed objects) |
| rules -> engine | `rules/` | `engine/` | `RuleSet` (versioned rule module) |
| engine -> curves | `engine/` | `curves/` | Spot rates (list/array), scenario definitions |
| curves -> projection | `curves/` | `projection/` | `dict[int, float]` (year -> rate), `list[float]` (discount factors), `Callable[[float], float]` (rate at tenor). **No QuantLib objects.** |
| curves -> projection (prices) | `curves/bond_pricing` | `projection/disinvestment` | `Callable[[AssetModelPoint, int], float]` (price function) |
| projection -> calculations | `projection/` | `calculations/` | `ScenarioResult` per scenario: C0, min_balance, cash flow DataFrame, portfolio snapshots |
| calculations -> output | `calculations/` | `output/` | `RunResult` containing BEL, RM, TP, LCR, stress test results, all scenario DataFrames |
| calculations -> challenge | `calculations/` | `challenge/` | `RunResult` (benchmark) + company `SubmissionData` |

---

## 3. RunConfig (Immutable Configuration)

### 3.1 Definition

```python
from dataclasses import dataclass, field
from datetime import date
from pathlib import Path
from typing import Optional
import hashlib, json

@dataclass(frozen=True)
class RunConfig:
    """Immutable configuration for a single model run.

    Once created, no field can be modified. All input files are hashed
    at creation time to ensure reproducibility.
    """

    # === Required fields ===
    run_id: str                          # UUID, auto-generated
    company_id: str                      # e.g., "ACME_RE"
    valuation_date: date                 # e.g., 2024-12-31
    rule_year: int                       # e.g., 2024 -> loads rules/v2024/

    # === Input file paths ===
    asset_file: Path                     # CSV/Excel asset portfolio
    epl_file: Path                       # CSV/Excel EPL cash flows
    market_data_file: Path               # CSV spot rates
    assumption_dir: Path                 # Directory of assumption tables

    # === Optional overrides ===
    reinvestment_strategy: str = "cash_only"   # "cash_only" or "pro_rata"
    projection_horizon: int = 100              # T_max in years
    c0_tolerance: float = 0.01                 # brentq xtol for C0 solve
    c0_upper_bound_factor: float = 2.0         # Upper bound = factor * sum(EPL)
    sba_spread_cap_bps: float = 35.0           # 35 bps cap (ALGO-004)

    # === Company submission (challenge mode only) ===
    submission_file: Optional[Path] = None

    # === Computed at creation ===
    input_hash: str = field(default="", init=False)
    created_at: str = field(default="", init=False)

    def __post_init__(self):
        # frozen=True prevents direct assignment; use object.__setattr__
        import uuid
        from datetime import datetime

        if not self.run_id:
            object.__setattr__(self, 'run_id', str(uuid.uuid4()))
        object.__setattr__(self, 'created_at', datetime.utcnow().isoformat())
        object.__setattr__(self, 'input_hash', self._compute_input_hash())

    def _compute_input_hash(self) -> str:
        """SHA-256 of all input file contents. Ensures reproducibility."""
        h = hashlib.sha256()
        for path in [self.asset_file, self.epl_file, self.market_data_file]:
            h.update(path.read_bytes())
        # Hash assumption directory contents
        for f in sorted(self.assumption_dir.glob("**/*")):
            if f.is_file():
                h.update(f.read_bytes())
        return h.hexdigest()
```

### 3.2 Immutability Contract

- `RunConfig` is a frozen dataclass. Any attempt to modify a field after creation raises `FrozenInstanceError`.
- The `input_hash` captures the exact byte content of all inputs at run start.
- Two runs with the same `input_hash` and `rule_year` must produce identical results.
- `RunConfig` is serialized to JSON and stored in the audit trail SQLite database.

### 3.3 Input Validation at Creation

`RunConfig.__post_init__` performs these checks (fail loudly):

| Check | Error if |
|---|---|
| `asset_file` exists and is readable | `FileNotFoundError` |
| `epl_file` exists and is readable | `FileNotFoundError` |
| `market_data_file` exists and is readable | `FileNotFoundError` |
| `assumption_dir` exists and is a directory | `NotADirectoryError` |
| `rule_year` has a matching `rules/v{year}/` module | `ValueError("No rules for year {year}")` |
| `valuation_date` is a valid business date | `ValueError` |
| `projection_horizon` is in [1, 150] | `ValueError` |
| `submission_file` exists (if provided) | `FileNotFoundError` |

---

## 4. Single Company Run Sequence

### 4.1 State Machine

The orchestrator tracks run progress through a linear state machine:

```
NotStarted
    |
    v  [Step 1: Load & validate inputs]
LoadingInputs -----> FAIL: ValidationError
    |
    v  [Step 2: Build curves]
BuildingCurves -----> FAIL: CurveConstructionError
    |
    v  [Step 3: Run projection x9 scenarios]
RunningProjection --> FAIL: ProjectionError
    |
    v  [Step 4: Assemble BEL]
AssemblingBEL ------> FAIL: BELCalculationError
    |
    v  [Step 5: Supporting metrics]
ComputingMetrics ---> FAIL: MetricError
    |
    v  [Step 6: Stress tests]
RunningStress ------> FAIL: StressTestError
    |
    v  [Step 7: Generate output]
WritingResults -----> FAIL: OutputError
    |
    v  [Step 8: Write audit trail]
WritingAudit -------> FAIL: AuditError
    |
    v
Complete
```

Any failure transitions to a `Failed` terminal state. The state, timestamp, and error details are recorded in the audit trail.

### 4.2 Orchestrator Entry Point

```python
class RunOrchestrator:
    """Conducts a single-company BMA SBA benchmark run.

    This class does not perform calculations. It sequences calls to
    the appropriate modules and manages state, errors, and audit logging.
    """

    def __init__(self, config: RunConfig):
        self.config = config
        self.state = RunState.NOT_STARTED
        self.audit = AuditTrail(config)
        self.result: Optional[RunResult] = None

    def execute(self) -> RunResult:
        """Run all 8 steps. Returns RunResult or raises."""
        try:
            inputs = self._step1_load_inputs()
            curves = self._step2_build_curves(inputs)
            scenario_results = self._step3_run_projections(inputs, curves)
            bel_result = self._step4_assemble_bel(scenario_results, curves)
            metrics = self._step5_compute_metrics(bel_result, inputs, curves)
            stress = self._step6_run_stress_tests(inputs, curves, bel_result)
            workbook_path = self._step7_generate_output(
                bel_result, metrics, stress, inputs
            )
            self._step8_write_audit()
            self.state = RunState.COMPLETE
            return self.result
        except Exception as e:
            self.state = RunState.FAILED
            self.audit.log_failure(self.state, e)
            raise
```

---

### 4.3 Step 1: Load and Validate Inputs

**Modules called:** `model_points/assets.py`, `model_points/liabilities.py`, `model_points/portfolio.py`, `assumptions/loader.py`, `rules/registry.py`

```
Input Files                    Pydantic Validation             Validated Objects
-----------                    -------------------             -----------------
assets.csv/xlsx  ----------->  AssetModelPoint schema  ------> List[AssetModelPoint]
epl.csv/xlsx     ----------->  EPLCashFlow schema      ------> List[EPLCashFlow]
market_data.csv  ----------->  MarketDataPoint schema  ------> List[MarketDataPoint]
assumptions/tables/ -------->  AssumptionSet loader     ------> AssumptionSet
rules/v{year}/   ----------->  registry.get_rules()    ------> RuleSet
```

**Validation checks (fail loudly -- any failure halts the run):**

| Check | Module | Error Type |
|---|---|---|
| All required columns present in asset file | `model_points/assets.py` | `ValidationError` |
| All field types and ranges valid (Pydantic) | `model_points/schemas.py` | `ValidationError` |
| No duplicate `asset_id` values | `model_points/assets.py` | `ValidationError` |
| EPL periods are contiguous (1, 2, 3, ...) | `model_points/liabilities.py` | `ValidationError` |
| Total EPL > 0 | `model_points/liabilities.py` | `ValidationError` |
| Currency consistency (assets match EPL) | `model_points/portfolio.py` | `ValidationError` |
| Tier 3 assets <= 10% of total portfolio MV | `model_points/portfolio.py` | `ValidationError` |
| All Tier 2 assets have `bma_approved=True` | `model_points/portfolio.py` | `ValidationError` |
| Assumption tables present and version-matched | `assumptions/loader.py` | `FileNotFoundError` |
| Credit floor tables have all required ratings | `assumptions/credit.py` | `ValidationError` |
| Spot rates cover required tenor range | `assumptions/economic.py` | `ValidationError` |

**Output of Step 1:**

```python
@dataclass
class ValidatedInputs:
    assets: List[AssetModelPoint]             # Pydantic-validated
    epl: List[EPLCashFlow]                    # Pydantic-validated
    market_data: List[MarketDataPoint]        # Pydantic-validated
    assumptions: AssumptionSet                # Typed assumption objects
    rules: RuleSet                            # Versioned rule module
    portfolio_summary: pd.DataFrame           # MV by tier, rating, type
    total_mv: float                           # Sum of asset market values
    total_epl: float                          # Sum of liability cash flows
```

---

### 4.4 Step 2: Build Curves

**Module called:** `curves/curve_builder.py`, `curves/scenario_curves.py`, `curves/ql_audit.py`

**This is the QuantLib boundary.** All QuantLib calls happen here and only here.

```
                    market_data (spot rates)
                           |
                           v
                  +------------------+
                  | curve_builder.py |     QuantLib: ql.PiecewiseYieldCurve
                  +--------+---------+
                           |
                    base risk-free curve
                           |
                  +--------+---------+
                  | scenario_curves  |     QuantLib: apply shifts per ALGO-001
                  +--------+---------+
                           |
              +------------+------------+--- ... ---+
              |            |            |            |
              v            v            v            v
          Scenario 1   Scenario 2   Scenario 3   Scenario 9
          (Base)       (Decrease)   (Increase)   (Inc+NegTwist)
```

**What crosses the QuantLib boundary (output of Step 2):**

```python
@dataclass
class ScenarioCurveSet:
    """Plain Python representation of all 9 scenario curves.
    No QuantLib objects -- only arrays, dicts, and callables.
    """

    base_discount_factors: np.ndarray         # DF[t] for t=0..T_max
    scenario_rates: dict[int, dict[int, float]]
        # scenario_id -> {year -> annual_rate}
        # e.g., {1: {1: 0.042, 2: 0.043, ...}, 2: {1: 0.027, ...}, ...}

    scenario_discount_factors: dict[int, np.ndarray]
        # scenario_id -> DF array

    bond_pricer: Callable[[AssetModelPoint, int, int], float]
        # (asset, scenario_id, year) -> clean_price
        # Wraps QuantLib bond pricing; callable from projection/

    base_portfolio_mv: float
        # MV of starting portfolio under base curve (for BEL formula)
```

**All QuantLib calls are logged by `ql_audit.py`:**
- Input: curve type, tenor points, rates
- Output: discount factors, bond prices
- Timestamp, call duration, QuantLib version

---

### 4.5 Step 3: Run Projection for Each Scenario

**Modules called:** `projection/cashflow_engine.py` (ALGO-002), `projection/credit_costs.py` (ALGO-003), `projection/reinvestment.py` (ALGO-002a), `projection/disinvestment.py` (ALGO-002b)

**Pure Python. No QuantLib imports.**

```
For each scenario s in [1..9]:

  +-------+     +-------+     +---------+
  | Assets|     |  EPL  |     | Scenario|
  | (copy)|     |       |     | Curve s |
  +---+---+     +---+---+     +----+----+
      |             |              |
      +------+------+--------------+
             |
             v
    +--------+---------+
    | C0 Root-Finder   |     scipy.optimize.brentq
    | (find C0 where   |
    |  min_balance = 0)|     xtol = config.c0_tolerance (default 0.01)
    +--------+---------+     bracket = [0, c0_upper_bound_factor * sum(EPL)]
             |
             | trial C0
             v
    +--------+---------+
    | run_projection() |     ALGO-002: period-by-period loop
    |                  |
    |  for t=1..T_max: |
    |    1. Collect CFs |     coupons, maturities, prepayments
    |    2. Credit adj  |     ALGO-003: D&D deductions
    |    3. Net CF      |     adjusted_cf + cash_interest - EPL[t]
    |    4. Reinvest    |     ALGO-002a: if surplus
    |       or Divest   |     ALGO-002b: if shortfall
    |    5. Txn costs   |     bid-ask spread model
    |    6. Update state|     portfolio, cash balance
    |    7. Audit log   |     rule refs, decisions
    +--------+---------+
             |
             v
    ScenarioResult(
        scenario_id = s,
        c0 = solved_C0,
        min_balance = 0.0,    (by definition -- brentq target)
        min_balance_period = t_min,
        cashflow_df = DataFrame[period, asset_cf, credit_adj,
                                net_cf, reinvest, divest,
                                txn_cost, cash_balance, portfolio_mv],
        portfolio_snapshots = dict[int, DataFrame],  # periodic snapshots
        rule_refs_triggered = List[RuleRef],
    )
```

**Parallelization:** The 9 scenarios are independent and can run in parallel. Use `concurrent.futures.ProcessPoolExecutor` with `max_workers = min(cpu_count - 1, 9)`.

```python
def _step3_run_projections(self, inputs, curves) -> dict[int, ScenarioResult]:
    self.state = RunState.RUNNING_PROJECTION
    self.audit.log_state(self.state)

    with ProcessPoolExecutor(max_workers=min(cpu_count() - 1, 9)) as pool:
        futures = {
            pool.submit(
                run_single_scenario,
                scenario_id=s,
                assets=inputs.assets,        # deep copy inside worker
                epl=inputs.epl,
                rates=curves.scenario_rates[s],
                discount_factors=curves.scenario_discount_factors[s],
                bond_pricer=curves.bond_pricer,
                rules=inputs.rules,
                config=self.config,
            ): s
            for s in range(1, 10)
        }

    results = {}
    for future in as_completed(futures):
        s = futures[future]
        results[s] = future.result()      # raises if worker failed
        self.audit.log_scenario_complete(s, results[s].c0)

    return results
```

**Note on bond_pricer serialization:** The `bond_pricer` callable wraps QuantLib, which cannot be pickled across processes. Two implementation options:

1. **Pre-compute price grids** in Step 2: for each scenario, each asset, each year, compute the clean price and store as `dict[tuple[str, int, int], float]`. This is a plain dict, fully serializable.
2. **Use ThreadPoolExecutor** instead of ProcessPool. Threads share memory, so the callable works. GIL contention is minimal because the projection loop is compute-bound on NumPy/pandas.

Recommended: Option 1 (pre-computed price grids) for maximum auditability -- every price used in the projection is pre-logged by `ql_audit.py`.

---

### 4.6 Step 4: Assemble BEL

**Module called:** `calculations/bel_calculator.py` (ALGO-004)

```
scenario_results: dict[int, ScenarioResult]
         |
         v
+--------+---------+
| For each scenario |
| derive implied    |
| SBA spread        |
+--------+---------+
         |
         v
+--------+---------+
| Enforce 35bps     |   ALGO-004: spread cap
| spread cap        |
+--------+---------+
         |
         v
+--------+---------+
| Select biting     |   biting = argmax(C0[s]) after cap adjustment
| scenario          |
+--------+---------+
         |
         v
BELResult(
    c0_by_scenario = {1: 5.2M, 2: 8.1M, ...},
    spread_by_scenario = {1: 0.0028, 2: 0.0041, ...},
    cap_applied = {1: False, 2: True, ...},
    capped_c0_by_scenario = {1: 5.2M, 2: 9.3M, ...},  # after cap
    biting_scenario_id = 2,
    c0_biting = 9.3M,
    base_portfolio_mv = 100.0M,
    bel = 109.3M,       # MV_assets(T=0) + C0_biting
)
```

**BEL formula (Rules Para 28(10)):**

```
BEL = MV_assets(T=0) + C0_biting

where:
  C0_biting = max(capped_C0[s] for s in 1..9)
  MV_assets(T=0) = sum of asset market values at valuation date under base curve
```

---

### 4.7 Step 5: Calculate Supporting Metrics

**Modules called:** `calculations/risk_margin.py`, `calculations/lcr.py`, `calculations/lapse_cost.py`

These calculations are independent of each other and can run in parallel.

```
bel_result + inputs + curves
         |
         +---> Risk Margin
         |       CoC = 6%
         |       RM = 0.06 * SUM(Modified_ECR[t] / (1+r[t+1])^(t+1))
         |       Uses base curve discount factors
         |       --> RiskMarginResult(rm, ecr_projection_df)
         |
         +---> Technical Provision
         |       TP = BEL + RM
         |       --> tp: float
         |
         +---> Liquidity Coverage Ratio
         |       LCR = Available_Liquidity / Potential_Surrender >= 105%
         |       Haircuts by liquidity tier (Level 1/2A/2B)
         |       --> LCRResult(lcr, available, outflows, passes)
         |
         +---> Lapse Cost
                 Surrender cost under mass lapse scenario
                 --> LapseCostResult(lapse_cost, affected_policies_df)
```

**Output of Step 5:**

```python
@dataclass
class MetricsResult:
    risk_margin: RiskMarginResult
    technical_provision: float             # BEL + RM
    lcr: LCRResult
    lapse_cost: LapseCostResult
```

---

### 4.8 Step 6: Run Stress Tests

**Module called:** `calculations/stress_tests.py`

Three mandatory stress tests (Handbook E5.6h):

```
+----------------------------------------------------+
| Stress Test 1: Combined Mass Lapse + Credit Spread  |
| Handbook E5.6h(i)                                   |
|                                                     |
|  1. Apply mass lapse: max(20%, BSCR lapse shock)    |
|  2. Apply credit spread widening:                   |
|       AAA +277bps, AA +350bps, A +450bps,           |
|       BBB +600bps, BB +900bps, B+ below +1200bps    |
|  3. Re-run biting scenario projection               |
|  4. Report stressed BEL vs base BEL                 |
+----------------------------------------------------+

+----------------------------------------------------+
| Stress Test 2: One-Notch Downgrade                  |
| Handbook E5.6h(ii)                                  |
|                                                     |
|  1. Downgrade every asset by one notch              |
|  2. Recalculate credit costs with new ratings       |
|  3. Re-run biting scenario projection               |
|  4. Report impact on BEL                            |
+----------------------------------------------------+

+----------------------------------------------------+
| Stress Test 3: No Tier 3 Reinvestment               |
| Handbook E5.6h(iii)                                 |
|                                                     |
|  1. Prohibit reinvestment into Tier 3 assets        |
|  2. All reinvestment -> Tier 1 or approved Tier 2   |
|  3. Re-run biting scenario projection               |
|  4. Report impact on BEL                            |
+----------------------------------------------------+
```

**Output:**

```python
@dataclass
class StressTestResult:
    test_name: str
    base_bel: float
    stressed_bel: float
    impact: float                          # stressed - base
    impact_pct: float                      # impact / base * 100
    passes: bool                           # positive surplus maintained
    detail_df: pd.DataFrame                # period-by-period stressed projection
```

---

### 4.9 Step 7: Generate Output Workbook

**Module called:** `output/excel_writer.py` (via openpyxl)

See Section 8 for full workbook structure.

```python
def _step7_generate_output(self, bel_result, metrics, stress, inputs):
    self.state = RunState.WRITING_RESULTS
    self.audit.log_state(self.state)

    self.result = RunResult(
        config=self.config,
        bel=bel_result,
        metrics=metrics,
        stress_tests=stress,
        inputs_summary=inputs.portfolio_summary,
    )

    output_dir = Path(f"output/{self.config.company_id}/{self.config.run_id}/")
    output_dir.mkdir(parents=True, exist_ok=True)

    workbook_path = output_dir / f"{self.config.company_id}_{self.config.valuation_date}_SBA.xlsx"
    excel_writer.write_workbook(workbook_path, self.result)

    return workbook_path
```

---

### 4.10 Step 8: Write Audit Trail

**Module called:** `engine/audit_trail.py`

See Section 7 for full audit trail structure.

```python
def _step8_write_audit(self):
    self.state = RunState.WRITING_AUDIT
    self.audit.log_state(self.state)

    output_dir = Path(f"output/{self.config.company_id}/{self.config.run_id}/")

    # SQLite: run metadata
    self.audit.write_sqlite(output_dir / "audit.db")

    # Parquet: period-by-period cash flows for all 9 scenarios
    self.audit.write_parquet(output_dir / "cashflows.parquet")

    # JSON Lines: rule references and decision log
    self.audit.write_jsonl(output_dir / "decisions.jsonl")
```

---

## 5. Batch Mode (Multiple Companies)

### 5.1 Architecture

```
CLI: python -m bma_sba_benchmark batch --companies companies.csv --date 2024-12-31

companies.csv:
  company_id, asset_file, epl_file, market_data_file, assumption_dir
  ACME_RE,    data/acme/assets.csv, data/acme/epl.csv, ...
  BETA_LIFE,  data/beta/assets.csv, data/beta/epl.csv, ...
  GAMMA_INS,  data/gamma/assets.csv, data/gamma/epl.csv, ...

                          +------------------+
                          | engine/batch.py  |
                          +--------+---------+
                                   |
                    +--------------+--------------+
                    |              |              |
                    v              v              v
            RunOrchestrator  RunOrchestrator  RunOrchestrator
            (ACME_RE)        (BETA_LIFE)      (GAMMA_INS)
            Steps 1-8       Steps 1-8        Steps 1-8
```

### 5.2 Batch Runner

```python
class BatchRunner:
    def __init__(self, companies_file: Path, valuation_date: date,
                 rule_year: int, max_workers: Optional[int] = None):
        self.companies = self._load_companies(companies_file)
        self.valuation_date = valuation_date
        self.rule_year = rule_year
        self.max_workers = max_workers or max(1, cpu_count() - 1)

    def execute(self) -> BatchResult:
        results = {}
        failures = {}

        with ProcessPoolExecutor(max_workers=self.max_workers) as pool:
            futures = {}
            for company in self.companies:
                config = RunConfig(
                    run_id="",    # auto-generated
                    company_id=company.company_id,
                    valuation_date=self.valuation_date,
                    rule_year=self.rule_year,
                    asset_file=company.asset_file,
                    epl_file=company.epl_file,
                    market_data_file=company.market_data_file,
                    assumption_dir=company.assumption_dir,
                )
                futures[pool.submit(run_company, config)] = company.company_id

            for future in as_completed(futures):
                cid = futures[future]
                try:
                    results[cid] = future.result()
                except Exception as e:
                    failures[cid] = str(e)

        # Write batch summary
        self._write_batch_summary(results, failures)
        return BatchResult(results=results, failures=failures)
```

### 5.3 Batch Isolation

Each company runs in its own process with:
- Its own `RunConfig` (independent `run_id`, `input_hash`)
- Its own output directory (`output/{company_id}/{run_id}/`)
- Its own audit trail
- No shared mutable state between companies

A failure in one company does not affect others. The batch summary reports which companies succeeded and which failed (with error details).

---

## 6. Challenge Mode (Benchmark vs Company Submission)

### 6.1 Purpose

Challenge mode compares the benchmark model's results against a company's SBA submission to identify discrepancies that warrant regulatory follow-up.

```
CLI: python -m bma_sba_benchmark challenge --company ACME --submission acme_sba_2024.xlsx

                    +-------------------+
                    | Benchmark Run     |
                    | (Steps 1-8)      |
                    +--------+----------+
                             |
                         RunResult
                             |
                             v
                    +--------+----------+
                    |  challenge/       |
                    |  compare.py       |
                    +--------+----------+
                             ^
                             |
                      SubmissionData
                             |
                    +--------+----------+
                    | challenge/        |
                    | import_submission |
                    +-------------------+
                             ^
                             |
                    acme_sba_2024.xlsx
                    (company submission)
```

### 6.2 Comparison Metrics

```python
@dataclass
class ComparisonResult:
    # Top-level metrics
    bel_benchmark: float
    bel_company: float
    bel_difference: float
    bel_difference_pct: float

    biting_scenario_benchmark: int
    biting_scenario_company: int
    biting_scenario_match: bool

    # Per-scenario C0 comparison
    c0_comparison: pd.DataFrame
    # Columns: scenario_id, c0_benchmark, c0_company, difference, difference_pct

    # Materiality flags
    flags: List[ChallengeFlag]
```

### 6.3 Materiality Thresholds

Differences are flagged when they exceed configurable thresholds:

| Metric | Threshold | Flag Level |
|---|---|---|
| BEL difference | > 2% | WARNING |
| BEL difference | > 5% | CRITICAL |
| C0 difference (any scenario) | > 5% | WARNING |
| Biting scenario mismatch | Different scenario | CRITICAL |
| Credit cost difference | > 10% | WARNING |
| LCR difference | > 5pp | WARNING |

### 6.4 Variance Drill-Down

When a top-level flag triggers, `challenge/variance_analysis.py` performs a period-by-period comparison of the biting scenario cash flows:

```
Period | Benchmark CF | Company CF | Diff   | Primary Driver
-------|-------------|-----------|--------|----------------------------
   1   |    5,200,000 |  5,150,000 |  50K   | Credit cost assumption
   2   |    4,900,000 |  4,700,000 | 200K   | Reinvestment yield diff
  ...
```

---

## 7. Audit Trail Structure

### 7.1 Three-Layer Audit Trail

Each run produces three audit artifacts, optimized for different query patterns:

```
output/{company_id}/{run_id}/
  |
  +-- audit.db            SQLite: run metadata, parameters, timestamps
  +-- cashflows.parquet   Parquet: period-by-period data for all 9 scenarios
  +-- decisions.jsonl     JSON Lines: rule references, decision log
  +-- {company}_SBA.xlsx  Output workbook
```

### 7.2 SQLite: Run Metadata (`audit.db`)

**Table: `runs`**

| Column | Type | Description |
|---|---|---|
| `run_id` | TEXT PK | UUID |
| `company_id` | TEXT | Company identifier |
| `valuation_date` | TEXT | ISO date |
| `rule_year` | INTEGER | Rule version year |
| `input_hash` | TEXT | SHA-256 of all inputs |
| `config_json` | TEXT | Full RunConfig serialized |
| `state` | TEXT | Final state (COMPLETE / FAILED) |
| `started_at` | TEXT | ISO timestamp |
| `completed_at` | TEXT | ISO timestamp |
| `duration_seconds` | REAL | Wall-clock time |
| `error_message` | TEXT | NULL if successful |

**Table: `scenario_results`**

| Column | Type | Description |
|---|---|---|
| `run_id` | TEXT FK | References `runs.run_id` |
| `scenario_id` | INTEGER | 1-9 |
| `scenario_name` | TEXT | e.g., "Decrease" |
| `c0` | REAL | Solved initial cash buffer |
| `capped_c0` | REAL | After spread cap (ALGO-004) |
| `implied_spread_bps` | REAL | Derived SBA spread |
| `cap_applied` | INTEGER | 0/1 boolean |
| `min_balance_period` | INTEGER | Period where min balance occurs |
| `solve_iterations` | INTEGER | brentq iterations |
| `solve_duration_ms` | REAL | Time to solve C0 |

**Table: `summary_results`**

| Column | Type | Description |
|---|---|---|
| `run_id` | TEXT FK | References `runs.run_id` |
| `bel` | REAL | Best Estimate Liability |
| `risk_margin` | REAL | Risk Margin (6% CoC) |
| `technical_provision` | REAL | TP = BEL + RM |
| `biting_scenario_id` | INTEGER | Worst scenario |
| `lcr` | REAL | Liquidity Coverage Ratio |
| `lcr_passes` | INTEGER | 0/1 (>= 105%) |
| `base_portfolio_mv` | REAL | Starting portfolio MV |
| `total_epl` | REAL | Sum of liability cash flows |

**Table: `state_log`**

| Column | Type | Description |
|---|---|---|
| `run_id` | TEXT FK | |
| `timestamp` | TEXT | ISO timestamp |
| `state` | TEXT | State name |
| `message` | TEXT | Optional detail |

### 7.3 Parquet: Period-by-Period Cash Flows (`cashflows.parquet`)

One row per (scenario, period). Columnar format for efficient analytical queries.

| Column | Type | Description |
|---|---|---|
| `scenario_id` | int8 | 1-9 |
| `period` | int16 | 1 to T_max |
| `asset_cf` | float64 | Gross asset cash flows |
| `default_cost` | float64 | Default deduction (ALGO-003) |
| `downgrade_cost` | float64 | Downgrade deduction (ALGO-003) |
| `adjusted_cf` | float64 | Asset CF after credit adj |
| `cash_interest` | float64 | Interest on cash balance |
| `liability_cf` | float64 | EPL outflow |
| `net_cf` | float64 | Surplus or shortfall |
| `reinvestment_amount` | float64 | Amount reinvested |
| `disinvestment_amount` | float64 | Amount from asset sales |
| `transaction_cost` | float64 | Bid-ask costs |
| `cash_balance` | float64 | End-of-period cash |
| `portfolio_mv` | float64 | Portfolio market value |
| `portfolio_count` | int32 | Number of active assets |

**Size estimate:** 9 scenarios x 100 periods = 900 rows, ~15 columns. Approximately 50-100 KB per run in compressed Parquet. Highly efficient for analytical tools (pandas, DuckDB, Power BI).

### 7.4 JSON Lines: Rule References and Decision Log (`decisions.jsonl`)

One JSON object per line. Captures every regulatory rule invoked and every material decision made.

```json
{"ts":"2026-03-24T10:00:01Z","type":"rule_ref","rule":"Para 28(7)(b)","module":"curves.scenario_curves","scenario":2,"detail":"Applied -150bps decrease shift at year 5"}
{"ts":"2026-03-24T10:00:02Z","type":"rule_ref","rule":"Para 28(22)","module":"projection.credit_costs","scenario":2,"period":1,"detail":"Default cost deduction: 12,500 on asset BOND_042 (rating A, year 1)"}
{"ts":"2026-03-24T10:00:02Z","type":"decision","module":"projection.disinvestment","scenario":2,"period":15,"detail":"Shortfall 250,000: sold BOND_007 (Govt, Tier 1) for 248,750 net of txn cost"}
{"ts":"2026-03-24T10:00:03Z","type":"rule_ref","rule":"Para 28(15)","module":"projection.disinvestment","scenario":2,"period":15,"detail":"Tier 3 asset BOND_099 excluded from liquidation"}
{"ts":"2026-03-24T10:00:04Z","type":"decision","module":"calculations.bel_calculator","detail":"Spread cap applied to scenario 2: implied 41.2bps capped to 35.0bps"}
{"ts":"2026-03-24T10:00:04Z","type":"rule_ref","rule":"Para 28(10)","module":"calculations.bel_calculator","detail":"Biting scenario: 2 (Decrease), capped C0 = 9,300,000"}
```

**JSON Lines format chosen because:**
- Append-only (no corruption risk on crash)
- One line per event (grep-friendly, streamable)
- Human-readable without special tools
- Loadable into pandas with `pd.read_json(path, lines=True)`

---

## 8. Output Workbook Structure

### 8.1 Tab Layout

The output Excel workbook is generated by `output/excel_writer.py` using openpyxl.

| Tab # | Tab Name | Content | Key Columns/Fields |
|---|---|---|---|
| 1 | **Summary** | Top-level regulatory results | BEL, RM, TP, biting scenario, base MV, LCR, lapse cost |
| 2 | **Scenario Results** | C0 by scenario, pre- and post-cap | scenario_id, name, c0, implied_spread, cap_applied, capped_c0 |
| 3 | **Cash Flow Detail** | Period-by-period for biting scenario | period, asset_cf, credit_adj, net_cf, reinvest, divest, cash_bal |
| 4 | **Asset Portfolio** | Starting portfolio snapshot | asset_id, type, tier, rating, par, mv, bv, coupon, maturity, duration |
| 5 | **Credit Costs** | D&D costs by period and rating bucket | period, rating, default_cost, downgrade_cost, total_dd, phase_in |
| 6 | **Stress Tests** | Results of 3 mandatory tests | test_name, base_bel, stressed_bel, impact, impact_pct, passes |
| 7 | **LCR** | Liquidity coverage breakdown | tier, asset_category, mv, haircut, available_liquidity, outflows, lcr |
| 8 | **Audit Trail** | Rule references and run metadata | rule_ref, module, count, description; plus RunConfig summary |
| 9 | **Comparison** | Benchmark vs company (if challenge) | metric, benchmark, company, difference, difference_pct, flag |

### 8.2 Formatting Standards

| Element | Format |
|---|---|
| Monetary values | `#,##0.00` (2 decimal places, comma separator) |
| Rates | `0.000000` (6 decimal places) |
| Basis points | `0.00 "bps"` |
| Ratios (LCR) | `0.0000` (4 decimal places) |
| Percentages | `0.00%` |
| Dates | `YYYY-MM-DD` |
| Headers | Bold, frozen top row |
| Biting scenario row | Highlighted (light yellow fill) |
| Failed stress tests | Red fill on "passes" cell |
| Challenge flags (CRITICAL) | Red fill |
| Challenge flags (WARNING) | Orange fill |

### 8.3 Summary Tab Detail

```
+-------------------------------------------+
|  BMA SBA Benchmark Model - Run Summary    |
+-------------------------------------------+
| Company:          ACME_RE                 |
| Valuation Date:   2024-12-31              |
| Rule Year:        2024                    |
| Run ID:           a1b2c3d4-...            |
| Input Hash:       sha256:7f8a...          |
+-------------------------------------------+
|                                           |
| Best Estimate Liability (BEL)  109,300,000|
| Risk Margin (RM)                 4,200,000|
| Technical Provision (TP)       113,500,000|
|                                           |
| Biting Scenario:   2 (Decrease)          |
| C0 (biting):       9,300,000             |
| Base Portfolio MV:  100,000,000          |
| Spread Cap Applied: Yes (41.2 -> 35.0bps)|
|                                           |
| LCR:               112.3% (PASS)        |
| Lapse Cost:         2,100,000            |
|                                           |
| Stress Test 1:     PASS (impact +3.2%)  |
| Stress Test 2:     PASS (impact +1.8%)  |
| Stress Test 3:     PASS (impact +0.9%)  |
+-------------------------------------------+
```

---

## 9. CLI Interface

### 9.1 CLI Framework

Built with `click` (or `typer`). Entry point: `python -m bma_sba_benchmark`.

### 9.2 Commands

#### `run` -- Single Company

```bash
python -m bma_sba_benchmark run \
    --company ACME_RE \
    --date 2024-12-31 \
    --rules 2024 \
    --assets data/acme/assets.csv \
    --epl data/acme/epl.csv \
    --market-data data/acme/market_data.csv \
    --assumptions data/acme/assumptions/ \
    --reinvestment cash_only \
    --output output/
```

| Argument | Required | Default | Description |
|---|---|---|---|
| `--company` | Yes | -- | Company identifier |
| `--date` | Yes | -- | Valuation date (YYYY-MM-DD) |
| `--rules` | Yes | -- | Rule year (2024, 2025, ...) |
| `--assets` | Yes | -- | Asset portfolio file |
| `--epl` | Yes | -- | EPL cash flow file |
| `--market-data` | Yes | -- | Spot rates file |
| `--assumptions` | Yes | -- | Assumption directory |
| `--reinvestment` | No | `cash_only` | `cash_only` or `pro_rata` |
| `--horizon` | No | `100` | Projection horizon (years) |
| `--output` | No | `output/` | Output directory |
| `--verbose` | No | `False` | Detailed console logging |

#### `batch` -- Multiple Companies

```bash
python -m bma_sba_benchmark batch \
    --companies companies.csv \
    --date 2024-12-31 \
    --rules 2024 \
    --workers 4 \
    --output output/
```

| Argument | Required | Default | Description |
|---|---|---|---|
| `--companies` | Yes | -- | CSV listing companies and their input files |
| `--date` | Yes | -- | Valuation date |
| `--rules` | Yes | -- | Rule year |
| `--workers` | No | `cpu_count - 1` | Max parallel workers |
| `--output` | No | `output/` | Output directory |

**companies.csv format:**

```csv
company_id,asset_file,epl_file,market_data_file,assumption_dir
ACME_RE,data/acme/assets.csv,data/acme/epl.csv,data/acme/market.csv,data/acme/assumptions/
BETA_LIFE,data/beta/assets.csv,data/beta/epl.csv,data/beta/market.csv,data/beta/assumptions/
```

#### `challenge` -- Benchmark vs Company Submission

```bash
python -m bma_sba_benchmark challenge \
    --company ACME_RE \
    --date 2024-12-31 \
    --rules 2024 \
    --assets data/acme/assets.csv \
    --epl data/acme/epl.csv \
    --market-data data/acme/market_data.csv \
    --assumptions data/acme/assumptions/ \
    --submission data/acme/acme_sba_submission_2024.xlsx \
    --output output/
```

Same arguments as `run`, plus:

| Argument | Required | Description |
|---|---|---|
| `--submission` | Yes | Company's SBA submission Excel file |

### 9.3 Console Output

```
BMA SBA Benchmark Model v0.1.0
================================
Company:    ACME_RE
Date:       2024-12-31
Rules:      v2024
Run ID:     a1b2c3d4-e5f6-7890-abcd-ef1234567890

[1/8] Loading inputs...                    OK  (1.2s)
  Assets: 347 model points validated
  EPL: 100 periods, total 523,000,000
  Assumptions: v2024Q4 loaded
[2/8] Building curves...                   OK  (3.5s)
  Base curve: 30 tenors, max 100y
  9 scenario curves constructed
  Bond price grid: 347 assets x 9 scenarios x 100 years
[3/8] Running projections (9 scenarios)... OK  (45.2s)
  Scenario 1 (Base):              C0 =   5,200,000  (4.8s)
  Scenario 2 (Decrease):          C0 =   8,100,000  (5.1s)
  Scenario 3 (Increase):          C0 =   3,900,000  (4.9s)
  Scenario 4 (Down-Up):           C0 =   6,700,000  (5.0s)
  Scenario 5 (Up-Down):           C0 =   4,100,000  (4.7s)
  Scenario 6 (Dec+PosTwist):      C0 =   7,500,000  (5.3s)
  Scenario 7 (Dec+NegTwist):      C0 =   8,400,000  (5.2s)
  Scenario 8 (Inc+PosTwist):      C0 =   4,300,000  (4.9s)
  Scenario 9 (Inc+NegTwist):      C0 =   3,700,000  (5.1s)
[4/8] Assembling BEL...                    OK  (0.3s)
  Spread cap applied to scenarios: 2, 7
  Biting scenario: 2 (Decrease)
  BEL = 100,000,000 + 9,300,000 = 109,300,000
[5/8] Computing metrics...                 OK  (2.1s)
  Risk Margin:  4,200,000
  TP:           113,500,000
  LCR:          112.3% (PASS)
[6/8] Running stress tests...              OK  (15.3s)
  Mass Lapse + Credit Spread:  PASS (+3.2%)
  One-Notch Downgrade:         PASS (+1.8%)
  No Tier 3 Reinvestment:      PASS (+0.9%)
[7/8] Generating output workbook...        OK  (1.8s)
  output/ACME_RE/a1b2c3d4-.../ACME_RE_2024-12-31_SBA.xlsx
[8/8] Writing audit trail...               OK  (0.5s)
  audit.db, cashflows.parquet, decisions.jsonl

================================
Run completed in 69.9s
```

---

## 10. Error Handling and Recovery

### 10.1 Error Categories

| Category | Examples | Handling |
|---|---|---|
| **Input Validation** | Missing columns, invalid types, duplicate IDs, Tier 3 > 10% | `ValidationError` -- halt immediately, report all violations |
| **File I/O** | File not found, permission denied, corrupt Excel | `FileNotFoundError` / `IOError` -- halt with clear path info |
| **Curve Construction** | QuantLib failure, degenerate curve, negative rates | `CurveConstructionError` -- halt, log QuantLib error details |
| **Projection** | brentq non-convergence, infinite loop, NaN values | `ProjectionError` -- halt with scenario/period context |
| **Calculation** | Division by zero in LCR, negative RM | `CalculationError` -- halt with formula context |
| **Output** | Disk full, Excel write failure | `OutputError` -- halt, audit trail may still be written |

### 10.2 Error Propagation

```
Module raises specific exception
    |
    v
RunOrchestrator catches in execute()
    |
    +-- Logs to audit trail (state_log table + decisions.jsonl)
    +-- Sets state = FAILED
    +-- Re-raises with context: run_id, company_id, step, original error
    |
    v
CLI catches and displays user-friendly message
    |
    +-- If --verbose: full traceback
    +-- Always: step number, error type, actionable message
```

### 10.3 brentq Non-Convergence

The most likely runtime failure is `scipy.optimize.brentq` failing to converge when finding C0:

| Cause | Symptom | Resolution |
|---|---|---|
| Upper bound too low | `ValueError: f(a) and f(b) must have different signs` | Increase `c0_upper_bound_factor` (default 2.0) |
| Discontinuous objective | brentq exceeds `maxiter` | Check for discrete jumps in disinvestment (e.g., selling a large bond) |
| Zero EPL | C0 = 0 trivially | Detect in Step 1 validation |
| All assets mature early | Cash goes negative in later periods regardless of C0 | Report as a genuine model finding (portfolio-liability mismatch) |

**Fallback:** If brentq fails after `maxiter=200`, log the failure, record the best bracket found, and continue with remaining scenarios. The failed scenario is flagged in the output.

### 10.4 Partial Results on Failure

If the run fails at Step N (where N > 3), the orchestrator attempts to write whatever results are available:

| Failed At | What Is Written |
|---|---|
| Step 1-2 | Audit trail only (state_log with error) |
| Step 3 | Audit trail + completed scenario results (partial) |
| Step 4-6 | Audit trail + all scenario results + whatever metrics computed |
| Step 7 | Audit trail + all results (workbook write failed, data is intact) |
| Step 8 | Workbook exists; audit trail write failed (log to stderr) |

---

## 11. Implementation Notes

### 11.1 File Layout

```
bma_sba_benchmark/engine/
    __init__.py
    run_config.py          # RunConfig frozen dataclass (Section 3)
    run_orchestrator.py    # RunOrchestrator class (Section 4)
    batch.py               # BatchRunner class (Section 5)
    audit_trail.py         # AuditTrail class (Section 7)
    states.py              # RunState enum
    errors.py              # Custom exception hierarchy
```

### 11.2 Dependency Injection

The orchestrator does not import QuantLib. It calls `curves/` through a well-defined interface:

```python
# engine/run_orchestrator.py -- NO QuantLib import

from bma_sba_benchmark.curves.curve_builder import build_base_curve
from bma_sba_benchmark.curves.scenario_curves import build_scenario_curves

# These functions return plain Python objects (ScenarioCurveSet)
# QuantLib is imported only inside curves/ modules
```

### 11.3 Serialization for Parallel Workers

For `ProcessPoolExecutor` (both within a single company for 9 scenarios and in batch mode for multiple companies), all data crossing process boundaries must be picklable:

| Object | Picklable? | Solution |
|---|---|---|
| `RunConfig` | Yes | Frozen dataclass with primitive types |
| `List[AssetModelPoint]` | Yes | Pydantic models serialize natively |
| `List[EPLCashFlow]` | Yes | Pydantic models serialize natively |
| Scenario rate dicts | Yes | `dict[int, float]` |
| Discount factor arrays | Yes | `np.ndarray` |
| Bond price grid | Yes | `dict[tuple, float]` (pre-computed in Step 2) |
| `RuleSet` | Yes | Module reference resolved by worker |
| QuantLib objects | **No** | Never sent to workers; pre-compute prices instead |

### 11.4 Memory Budget

Estimated per-company memory usage:

| Component | Estimate | Notes |
|---|---|---|
| Asset model points | ~500 assets x 1 KB = 0.5 MB | Pydantic objects |
| EPL cash flows | ~100 periods x 0.1 KB = 0.01 MB | |
| Price grid | 500 x 9 x 100 x 8 bytes = 36 MB | Pre-computed float64 |
| Projection DataFrames | 9 x 100 x 15 cols x 8 bytes = 0.1 MB | Period-by-period |
| Portfolio snapshots | 9 x 20 snapshots x 500 x 0.5 KB = 45 MB | Periodic (every 5 years) |
| **Total per company** | **~80-100 MB** | Fits comfortably on standard workstation |

For batch mode with 4 parallel workers: ~400 MB. Well within typical 16 GB workstation RAM.

### 11.5 Timing Budget

Estimated wall-clock time for a single company (500 assets, 100-year projection):

| Step | Estimate | Notes |
|---|---|---|
| Step 1: Load inputs | 1-2s | File I/O + Pydantic validation |
| Step 2: Build curves | 3-5s | QuantLib curve construction + price grid |
| Step 3: Projections (9 scenarios) | 30-60s | ~5s per scenario x 9, parallelized |
| Step 4: Assemble BEL | <1s | Arithmetic on 9 C0 values |
| Step 5: Metrics | 1-3s | RM, LCR, lapse cost |
| Step 6: Stress tests | 10-20s | 3 tests, each re-runs biting projection |
| Step 7: Output workbook | 1-2s | openpyxl write |
| Step 8: Audit trail | <1s | SQLite + Parquet + JSONL writes |
| **Total** | **~50-90s** | On modern workstation (8+ cores) |

### 11.6 Testing Strategy

| Test Type | Target | Description |
|---|---|---|
| **Golden test** | `test_illustrative_calc.py` | End-to-end: BMA illustrative calculation inputs -> expected BEL, RM, TP |
| **Unit tests** | Each step method | Mock upstream outputs, verify step logic |
| **Integration test** | `run` command | CLI invocation with test fixture files |
| **Batch test** | `batch` command | 3 test companies, verify isolation |
| **Challenge test** | `challenge` command | Known-difference submission, verify flags |
| **Error tests** | Each error category | Invalid inputs, corrupt files, brentq failure |
| **Reproducibility** | Same inputs, two runs | Identical `input_hash`, identical results |
| **Audit trail** | SQLite/Parquet/JSONL | Query audit DB, load Parquet, parse JSONL |

### 11.7 Cross-Reference to Other Specs

| Step | Primary Spec | Section |
|---|---|---|
| Step 1: Load inputs | (Schema spec -- future) | -- |
| Step 2: Build curves | [ALGO-001](01_Yield_Curve_Construction.md) | Sections 2-4 |
| Step 3: Projection | [ALGO-002](02_Projection_Engine.md) | Sections 2-5 |
| Step 3: Reinvestment | [ALGO-002a](02a_Reinvestment_Strategy.md) | Sections 3-5 |
| Step 3: Disinvestment | [ALGO-002b](02b_Disinvestment_Waterfall.md) | Sections 3-7 |
| Step 3: Credit costs | [ALGO-003](03_Credit_Costs_DD.md) | Sections 2-5 |
| Step 4: BEL / Spread cap | [ALGO-004](04_Spread_Cap_Enforcement.md) | Sections 2-5 |
| Step 5: Risk Margin | (Risk Margin spec -- future) | -- |
| Step 5: LCR | (LCR spec -- future) | -- |
| Step 6: Stress tests | Tech Specs Section 7 | -- |

---

*End of specification.*
