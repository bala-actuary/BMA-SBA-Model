# 00-Architecture-Overview.md

# ATHORA BEL MODEL v0.1.8 - ARCHITECTURE & SYSTEM OVERVIEW

## Executive Summary

The Athora BEL Model is a sophisticated insurance liability-asset valuation engine implementing Solvency II regulatory metrics (SBA - Solvency Business Assessment). It performs multi-period (typically 100-year) stochastic projections of insurance balance sheets with dynamic asset rebalancing, liability matching optimization, and scenario analysis.

**Core Purpose**: Given assets and liabilities at t=0, find the minimum proportion of assets/liabilities such that they remain in balance over the projection period, subject to rebalancing constraints.

---

## System Architecture (High Level)

```
USER INPUT
  ├─ Setup Workbook (Excel)
  │   ├─ Parameters tab (all model settings)
  │   ├─ Rates, Scenarios, Liability CFs
  │   ├─ Asset allocation targets
  │   └─ Rebalancing constraints
  │
  └─ Asset Data File (Excel)
      ├─ Bonds, Swaps, Cash (SBA-eligible)
      ├─ Market values, spreads, coupons, maturities
      └─ Collateral, derivatives (may be excluded)

              │
              ▼

CONFIGURATION LOADING (LoadSettingsLauncher)
  ├─ Parse Excel parameters
  ├─ Load market data (rates, vols, spreads, FX)
  ├─ Load liability cashflows
  ├─ Validate business unit settings
  └─ Verify parameter consistency

              │
              ▼

ALM MODEL INITIALIZATION
  ├─ Build asset parameter lookup tables
  ├─ Create asset projector (QuantLib valuation engine)
  ├─ Calibrate market curves & volatility surfaces
  ├─ Create rebalancing strategy (SAA/CAA/Most Onerous)
  ├─ Create swap rebalancer (hedging logic)
  ├─ Create z-spread provider (spread calibration)
  └─ Create SBA goal seeker (optimization engine)

              │
              ▼

MAIN PROJECTION (SBAGoalSeeker.perform_alm)
  │
  ├─ Step 1: INITIAL PROJECTION
  │   └─ Project balance sheet at 100% asset, X% liability
  │
  ├─ Step 2: SCORE INITIAL
  │   └─ Check if converged (asset ≈ liability in PV terms)
  │
  ├─ Step 3: Determine optimization direction
  │   ├─ IF passed: reduce asset holdings
  │   └─ IF failed: reduce liability holdings
  │
  └─ Step 4: ITERATIVE GOAL SEEKING (bisection/linear search)
      ├─ Adjust proportion by 1-3% each iteration
      ├─ Project full balance sheet (SLOW)
      │   ├─ 100-101 time periods
      │   ├─ Per period: calibrate assets, project liability, rebalance
      │   └─ Record all cashflows
      ├─ Score result (convergence measure)
      ├─ REPEAT until:
      │   ├─ Optimal: Within optimal tolerance + min attempts
      │   ├─ Absolute: Within absolute tolerance + optimal attempts
      │   └─ Hard Stop: Reached max attempts
      └─ Return best result

              │
              ▼

RESULTS AGGREGATION & OUTPUT
  ├─ Extract summary table (by scenario)
  ├─ Export to Excel workbook
  │   ├─ Summary sheet (SBA ratios, convergence)
  │   ├─ Balance sheet runoff (period-by-period)
  │   ├─ Portfolio runoff (asset movements)
  │   ├─ Rebalancing events (buy/sell log)
  │   └─ Scenario comparison
  └─ Write log file (execution trace)

              │
              ▼

USER OUTPUT
  ├─ Results_{run_name}_{scenario}.xlsx
  │   ├─ Solvency margins (SBA ratios)
  │   ├─ ALM convergence metrics
  │   └─ Detailed projections
  └─ Logs/{run_name}/execution.log
```

---

## Data Flow Diagram

```
BASE DATA FLOW (Per BU, Per Scenario)
═════════════════════════════════════

INPUT FILES:
┌─────────────────────┐
│   Setup.xlsm        │
├─────────────────────┤
│ Parameters sheet    │
│ - Goal seek limits  │
│ - Rebalancing flags │
│ - Output settings   │
├─────────────────────┤
│ Rates sheet         │
│ - Curves (market,   │
│   corporate)        │
│ - FX movements      │
├─────────────────────┤
│ Liability sheet     │
│ - CFs by scenario   │
├─────────────────────┤
│ Asset allocation    │
│ - SAA targets       │
└─────────────────────┘
         │
         ▼
┌──────────────────────┐
│ Athora_Assets.xlsx   │
├──────────────────────┤
│ ISIN, Notional,      │
│ Market Value,        │
│ Z-Spread,            │
│ Maturity Date,       │
│ ... (all fields)     │
└──────────────────────┘
         │
         ▼
      PARSING
      ├─ Validate column names
      ├─ Load as DataFrame
      ├─ Type convert
      ├─ Check for missing data
      └─ Flag ineligible assets
         │
         ▼
    ASSET PROCESSING
      ├─ Classify asset type (Bond, Swap, etc.)
      ├─ Extract parameters (coupon, maturity, etc.)
      ├─ Calculate theoretical values (QuantLib)
      ├─ Apply stresses (interest rate shocks)
      ├─ Group by asset class (for SAA)
      └─ Store in memory as Asset objects
         │
         ▼
  BALANCE SHEET ASSEMBLY
      ├─ Assets side
      │   ├─ Bond portfolio (sum MV)
      │   ├─ Swap positions (net PV)
      │   └─ Cash account
      │
      └─ Liabilities side
          ├─ Insurance liabilities (SBA)
          ├─ Insurance liabilities (Standard)
          ├─ Derivative collateral (if used)
          └─ Margin requirement
         │
         ▼
    ONE PROJECTION RUN
      ├─ FOR t = 0 to 100 years:
      │   ├─ Evolve market data (rates, FX)
      │   ├─ Revalue assets (QuantLib)
      │   ├─ Accrue liability cashflows
      │   ├─ Rebalance portfolio (if triggered)
      │   ├─ Apply margin calls (derivatives)
      │   └─ Record snapshot
      │
      └─ Return: ProjectionResults(asset_MV, liability_PV, proportions, ...)
         │
         ▼
    GOAL SEEK LOOP
      - Iterations: 1 to 20 (typical)
      - Each iteration: One full projection (slow)
      - Adjusts proportion by bisection or linear search
      - Targets: asset_MV ≈ -liability_PV
         │
         ▼
    RESULTS ASSEMBLY
      ├─ Extract convergence metrics
      ├─ Pivot results by scenario
      ├─ Calculate SBA ratios
      └─ Format for output
         │
         ▼
    EXCEL OUTPUT
      ├─ Write summary table
      ├─ Write period-by-period runoff
      ├─ Write rebalancing events
      ├─ Write scenario comparison
      └─ Apply formatting & formulas
```

---

## Module Dependency Graph

```
EXTERNAL DEPENDENCIES
├─ QuantLib (bond/swap valuation)
├─ NumPy/SciPy (linear algebra, optimization)
├─ Pandas (DataFrames)
├─ OpenPyXL (Excel I/O)
├─ Click (CLI)
└─ PySimpleGUI (Progress bars)


INTERNAL MODULE HIERARCHY
─────────────────────────

alm/main.py
├─ CLI router → dispatch to commands
└─ main_server.py (orchestration)
    ├─ main_client.py (single scenario worker)
    │   └─ run_client()
    │       ├─ initialise_settings() → settings_loader.py ★
    │       │   └─ Settings (all parameters)
    │       ├─ initialise_model() → creates all components
    │       │   ├─ AssetParameterProvider ← asset_params.py
    │       │   ├─ BalanceSheet ← balance_sheet.py
    │       │   ├─ MarketProjection ← market_projection.py
    │       │   └─ Asset objects ← instruments/assets/*.py
    │       ├─ calibrate_model()
    │       │   ├─ calibrate_all_asset_values()
    │       │   └─ apply_t0_stresses()
    │       │
    │       └─ project_liability()
    │           └─ SBAGoalSeeker.perform_alm()  ← sba_goal_seeker.py ★★★
    │               │
    │               ├─ project_balance_sheet() (x20 times in goal seek)
    │               │   └─ BalanceSheetProjection.project()  ← balance_sheet_projection.py ★★★
    │               │       │
    │               │       ├─ FOR t=0..100:
    │               │       │   └─ _project_single_time_period()
    │               │       │       ├─ MarketProjection.project(t)
    │               │       │       ├─ AssetProjector.project()
    │               │       │       ├─ ZSpreadProvider  ← zspread_provider.py ★★
    │               │       │       ├─ RebalancingStrategy.rebalance()  ← alm_rebalancing.py ★★
    │               │       │       │   ├─ AssetAllocator  ← asset_allocator.py
    │               │       │       │   └─ RebalancingEvents
    │               │       │       ├─ SwapRebalancer.project()
    │               │       │       └─ apply_margin_calls()
    │               │       │
    │               │       └─ ProjectionResultsBuilder
    │               │
    │               ├─ _score() → OptimiseScore ★
    │               ├─ _optimise_proportion() → bisection/linear ★★★
    │               └─ Return best results
    │
    └─ write_results() ← writer_excel.py / writer_csv.py
        └─ Export ProjectionResults to Excel

server_state_machine.py
├─ ScenarioStep enum (5 stages)
├─ BuState (per BU × Scenario)
└─ RunState (master state)

data_conversion/
├─ data_1_convert_UBS_Delta.py ← convert1 command
│   └─ Convert CSV → Standardised format
├─ data_2_finalise_data.py ← convert2 command  ★
│   └─ Apply SBA eligibility rules (lines 613-635)
└─ data_conversion_globals.py
    └─ Column definitions & asset type mappings


★ = Critical for SBA verification
★★ = High importance for understanding
★★★ = Very high importance
```

---

## Key Initialization & Calibration Sequence

```
TIMELINE OF EXECUTION
══════════════════════

t = 0s:  Application Started
         load_program_state(runs_file)
           → Parse run definitions
           → Create SingleInstanceVariables for each BU/Scenario
           → Queue jobs

t = 0.1s: Worker processes spawned
          ProcessPool creates subprocesses
          Each worker (1, 2, 3, ...) receives job

Per Worker t = 0.2s: run_client() START
          │
          ├─ LoadSettingsLoader
          │   load_settings(setup.xlsm)
          │     → Settings dataclass populated from Excel
          │     → All parameters loaded (goal seek limits, spreads, etc.)
          │     → Market data loaded (rates, vols, spreads, FX)
          │     → Liability CFs loaded
          │
          ├─ Initialise Model Components
          │   initialise_model(settings)
          │     ├─ AssetParameterProvider.load_asset_table()
          │     ├─ Load .xlsx asset file
          │     │   → Classify: eligible vs ineligible
          │     │   → Create Asset objects (Bonds, Swaps, etc.)
          │     │
          │     ├─ Create MarketProjection
          │     │   → Prepare rate curves for t=0..T
          │     │   → Swaption vol surface calibration
          │     │   → FX movement matrix
          │     │
          │     ├─ Create AssetProjector
          │     │   → QuantLib valuation engine for each asset type
          │     │
          │     ├─ Create ZSpreadProvider
          │     │   → Prepare for t=0 calibration (will run later)
          │     │
          │     ├─ Create RebalancingStrategy
          │     │   → Load SAA from settings
          │     │   → Identify cash_account_asset_class
          │     │
          │     ├─ Create SwapRebalancer
          │     │   → Key rates, bid-offer spreads
          │     │
          │     └─ Create SBAGoalSeeker
          │         └─ Holds reference to BalanceSheetProjection
          │
          ├─ Calibrate Model (ONE-TIME)
          │   calibrate_model(settings, balance_sheet)
          │     │
          │     ├─ calibrate_all_asset_values()
          │     │   FOR each asset:
          │     │     asset_projector.calibrate(asset, md_t0)
          │     │       → QuantLib valuation
          │     │       → Calculate PV, duration, DV01, etc.
          │     │
          │     ├─ apply_t0_stresses()
          │     │   FOR each asset:
          │     │     → Mark for stress calculation (key-rate DV01)
          │     │
          │     └─ alm_rebalancing.calibrate()
          │         ├─ Extract SAA from stock assets
          │         ├─ Extract CAA from actual assets
          │         ├─ Calculate spreads by asset class
          │         └─ Determine most onerous (SAA vs CAA)
          │
          ├─ Execute Projection
          │   project_liability()
          │     └─ SBAGoalSeeker.perform_alm(balance_sheet)
          │         │
          │         ├─ project_balance_sheet(asset%=100%, liab%=X%)
          │         │   └─ One full projection (t=0..100 years)
          │         │
          │         ├─ _score() → evaluate convergence
          │         │
          │         ├─ LOOP (bisection / linear search)
          │         │   │ Iteration 0-20 (max)
          │         │   ├─ Choose next proportion
          │         │   ├─ project_balance_sheet() [SLOW]
          │         │   ├─ _score() → check convergence
          │         │   └─ UNTIL converged OR max attempts
          │         │
          │         └─ RETURN best ProjectionResults
          │
          ├─ Write Results
          │   write_results()
          │     └─ ResultsWriter.write_excel()
          │         └─ Output/{run_name}/{scenario}_{bu}.xlsx
          │
          └─ run_client() EXIT

Per Job t = (variable): STATE MACHINE TRANSITIONS
          server_state_machine.tick()
          ├─ Step 0 → Step 1 (value assets) [if enabled]
          ├─ Step 1 → Step 2 (value liability) [if enabled]
          ├─ Step 2 → Step 3 (minimize liability) [if SBA minimise enabled]
          ├─ Step 3 → Step 4 (write results)
          └─ Step 4 → Step 5 (complete)

Main Thread: Orchestrates all workers via ProcessPool.tick()
             When all jobs done → merge results → exit
```

---

## Cross-Cutting Concerns

### Performance Optimization

**Slow operations**:
1. Full projection run: ~1 minute each (20 iterations × 5 seconds per iteration)
2. Asset valuation (QuantLib): ~100-200ms per asset × 1000 assets × 101 periods
3. Goal seek bisection: Major bottleneck

**Speed-ups**:
- Cache enabled (enable_cache=TRUE): Store projected rates, cache asset valuations
- Dryrun mode: Only project 1 year, disable goal seek, 1000x faster
- Rebalancing disabled: 20% faster (skip optimization)
- Parallel workers: 8 scenarios × 8 CPUs = ~8x wallclock speedup

### Error Handling

```
Try/Catch Hierarchy:
  
  run_server (main.py)
    └─ ProcessPool.tick()
         └─ run_client (worker subprocess)
               ├─ initialise_settings()
               │   └─ ERROR → JobState = Error
               ├─ initialise_model()
               │   └─ ERROR → JobState = Error
               ├─ project_liability()
               │   └─ ERROR → Log stack trace, return
               └─ write_results()
                   └─ ERROR → Mark job failed
```

Critical errors (halt):
- Negative asset valuation (model bug)
- NaN in projection (numerical error)
- Asset file not found (config error)
- Liability has positive PV (unsuitable for SBA)

Recoverable errors (skip & continue):
- Goal seek didn't converge (log warning, use best attempt)
- Rebalancing failed (log event, constrain portfolio)
- Swap rebalancer diverged (skip swaps this period)

---

## Testing & Debugging

### Small Test Case (2 minutes)

```
Dryrun settings:
  projection_term_years: 1        (vs 100)
  goal_seek_perform: False        (vs True)
  num_assets: 100                 (vs 1000)
  num_scenarios: 1                (vs 9)
  rebalance_enable: False         (vs True)
  
Result: Execution finishes in 2-5 seconds
        Can verify data loading, market projection, basic valuation
```

### Debug Output

Enable in logs/execution.log:
```
[00:00:10] Loading settings from {setup_file}
[00:00:15] Loaded 342 assets (287 eligible, 55 ineligible)
[00:00:16] Initializing QuantLib market curves
[00:00:20] Calibrating asset valuations (100ms per asset)
[00:00:30] Starting goal seek for scenario SBA_0_Base
           Iteration 0: proportion=100%, net_value=-€500K (need to tighten)
           Iteration 1: proportion=98%, net_value=-€200K
           Iteration 2: proportion=96%, net_value=+€50K (narrowed)
           ...
           Iteration 8: proportion=95.3%, net_value=-€100 (CONVERGED)
[00:00:40] Writing results to outputs/{scenario}_{bu}.xlsx
[00:00:42] Job complete
```

---

## User Guide Cross-References

All parameters documented in "04.04 _50 SBA Python Model User Guide v0.3.pdf":

| Model Component | User Guide Section | Key Parameters |
|-----------------|-------------------|----|
| Goal Seek | Section 2.2 | goal_seek_attempts_min, goal_seek_tolerance_optimal |
| Z-Spread | Section 2.2 | sba_zspread_calibration_method, sba_spread_cap |
| Rebalancing | Section 2.2 | rebalance_enable, rebalance_taa_source |
| Rates Projection | Section 2.2 | rates_sheet, sba_ir_10yr_plus_methodology |
| Liability CFs | Section 2.2 | liability_column_sba_eligible |
| Output | Section 4 | output_balance_sheet_runoff, output_trades |

---

## Quick Start: Verification Process

**For SBA Implementation Review**:

1. **Data Flow** (understand inputs/outputs)
   - Assets → Eligibility → Z-spread weights
   - Parameters → Market data → Projection
   
2. **Core Algorithm** (verify logic)
   - Goal seek convergence criteria
   - Z-spread mean reversion formula
   - Asset/liability matching algorithm
   
3. **Implementation** (check code)
   - sba_goal_seeker.py - goal seek algorithm
   - zspread_provider.py - spread calibration
   - balance_sheet_projection.py - main loop
   
4. **Results** (validate outputs)
   - SBA ratios make sense
   - Convergence achieved
   - Balance sheet balanced throughout

---

## Next Steps

1. **Start Here**: Read this document (0) for overview
2. **Go Deeper**: Pick a component (1-7) based on your focus
3. **Understand Flow**: Trace path for a single scenario end-to-end
4. **Verify Implementation**: Use Verification Checklist in each component doc
5. **Debug Specific Issue**: Find module, read deep-dive, check code

**Navigation**: See `/memories/session/model_reference_index.md` for quick links to all 7 detailed component guides.

---

## Summary Table: All Components

| Component | File | Purpose | Criticality |
|-----------|------|---------|-------------|
| SBA Goal Seeking | sba_goal_seeker.py | Iterative optimization | ★★★ Critical |
| Z-Spread Calibration | zspread_provider.py | Spread extraction & projection | ★★ High |
| Data Conversion | data_2_finalise_data.py | Eligibility classification | ★★ High |
| Rebalancing | alm_rebalancing.py | Portfolio allocation matching | ★★ High |
| Projection Loop | balance_sheet_projection.py | 100-year valuation | ★★★ Critical |
| Configuration | settings_loader.py | Parameter loading & validation | ★ Important |
| CLI/Execution | main_server.py | Multi-worker orchestration | ★ Important |

---

---

## Spread Cap & BEL Calculation

The Pythora model applies a **35 basis point regulatory cap** on the derived SBA spread for each scenario:

**Uncapped Spread Calculation**:
- Goal-seek the spread S such that: PV(Liability CFs, RF + S) = MV(Assets scaled)
- S can exceed 35bps in stressed scenarios

**Spread Cap Application**:
- IF uncapped_spread > 35bps:
  - Apply regulatory cap: capped_spread = 35bps
  - Recalculate BEL: BEL_capped = PV(Liability CFs, RF + 35bps)
- ELSE:
  - Use uncapped spread as-is

**Output Reporting**:
- Report both uncapped and capped values per scenario
- Use capped BEL as primary reporting figure
- Most onerous: Select highest capped BEL across 9 scenarios

**For ANL (Ineligible EPC)**:
- EPC liabilities are not SBA-eligible
- Calculate SBA BEL for EPC-excluding liabilities using same capped spread

---

## Current Model Limitations & Validation Status

### Not Yet Fully Validated (v1.0.0 - Oct 2024)

- ⚠️ **Defaults & Downgrades (D&D)**: Implemented but not fully tested; alternative expected loss method used for validation
- ⚠️ **Callable Bonds (Dynamic Call)**: Not validated; using MaturityDate assumption as fallback
- ⚠️ **Asset Optionalities**: Further development recognized as needed

For full details, see [08-Defaults-Downgrades.md](08-Defaults-Downgrades.md) and [09-Spread-Cap-and-Outputs.md](09-Spread-Cap-and-Outputs.md)
