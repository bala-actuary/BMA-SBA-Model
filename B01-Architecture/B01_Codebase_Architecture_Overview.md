# B01: Codebase Architecture Overview

**Part B (Codebase Mastery)** — Module 1 of 14  
**Checklist Items**: [B1.1–B1.3](../../../../ATHORA_CODEBASE_MASTERY_CHECKLIST.md#b1-codebase-architecture-overview)  
**Time Estimate**: 2–3 hours  
**Mastery Level Target**: Level 2–3  
**Prerequisite**: Basic Python package structure understanding  
**Reading Time**: 45–60 minutes  
**Status**: ✅ COMPLETE — Comprehensive walkthrough with code organization, data flow, and entry points

---

## 🎯 Learning Objectives

By completing this module, you will be able to:

1. **Navigate confidently** — Locate any module in the codebase and understand its role
2. **Explain the data flow** — Trace how data moves from Excel input → Python objects → SBA projection → Excel output
3. **Understand dependencies** — Know which modules depend on which, and why data flows in one direction
4. **Invoke execution modes** — Know when to use `run-client` vs. `run-server` vs. `convert1/convert2`
5. **Draw from memory** — Sketch the 6 core module dependency graph without references

---

## Section 0: Problem Statement & Why Architecture Matters

### The Challenge: Managing Complexity at Scale

The Athora SBA model is a **production-grade financial projection engine**, not a simple script:

- **100+ user-configurable parameters** (rates, scenarios, constraints, hedging rules)
- **9 regulatory scenarios** per Article 28(7) (base + 8 market shocks)
- **5 business units** (different portfolios per company)
- **3,000+ lines of core projection code** (non-trivial mathematics)
- **Multi-year, multi-scenario execution** (output volumes: 100+ worksheets per run)

**The Risk**: Without clear architecture, this complexity becomes **unmaintainable**:
- ❌ Where does configuration come from? (If not centralized = inconsistent)
- ❌ How do modules interact? (If not clear = unexpected dependencies)
- ❌ What's the execution flow? (If not documented = hard to debug)
- ❌ How do I add a new feature? (If architecture unknown = fragile changes)

### The Athora Solution: Modular Architecture

**Design Principle**: Separate concerns into 6 independent modules that connect via clear interfaces:

```
EXCEL CONFIG → SETTINGS LOADER → ASSET LOADER → PROJECTION ENGINE → REBALANCER → EXCEL OUTPUT
                     ↓                 ↓              ↓ (core)              ↓
              [loaders/]         [instruments/]  [projection/]      [markets/, valuation/]
```

**Why This Matters**:
- ✅ Each module has **one responsibility** (single responsibility principle)
- ✅ Modules connect via **defined interfaces** (DataClass objects, not global state)
- ✅ Easy to **test in isolation** (no tangled dependencies)
- ✅ Easy to **replace a module** without breaking others (e.g., swap rebalancer algorithm)
- ✅ Clear **data flow** for auditing and debugging

### For Your Learning

**B01 goal**: Understand this architecture so you can:
1. Navigate the codebase confidently (know where to find anything)
2. Trace data flow end-to-end (input → output understanding)
3. Understand dependencies (what needs what)
4. Predict impact of changes (modify one module, what else breaks?)

---

## Section 1: Repository Structure & Navigation

### 1.1 Root Directory Layout

The Athora SBA model lives in an executable package with the following structure:

```
02. Model/
├── Athora_BEL_Model_v0.1.8.pyz             # Zipped executable (distribution format)
└── Athora_BEL_Model_v0.1.8/                # Or unzipped directory (development format)
    ├── __main__.py                          # Entry point (defines how to run)
    ├── environment.json                     # Python version + environment config
    └── site-packages/alm/                   # ← MAIN PACKAGE (all code lives here)
        ├── main.py                          # CLI dispatcher (routes to modes)
        ├── data_conversion/                 # Data preprocessing pipeline
        ├── model/                           # Core valuation engine
        ├── packing/                         # Package distribution
        └── post_proc/                       # Post-processing (merge, extract, etc.)
```

**KEY INSIGHT**: Don't look for code at the package root! All implementation is inside `site-packages/alm/`.

### 1.2 The Core Engine: site-packages/alm/model/

The heart of the system is the `model/` subdirectory, which contains 6 functional modules:

```
site-packages/alm/model/
├── loaders/                 # Excel configuration input
├── instruments/             # Asset/liability definitions
├── projection/              # **CORE ENGINE** — SBA BEL calculation
├── markets/                 # Market data provider
├── valuation/               # Instrument pricing
├── writer/                  # Excel output generation
├── main_client.py           # Orchestration logic
├── main_server.py           # Multi-scenario orchestration
├── tests/                   # Unit tests
└── utils/                   # Shared utilities
```

### 1.3 The Support Modules: Top-Level alm/

```
site-packages/alm/
├── data_conversion/         # Data standardization pipeline
├── packing/                 # .pyz package creation
├── post_proc/               # Merge outputs, extract metrics
└── main.py                  # CLI dispatcher
```

### 1.4 Code References: Key File Locations

**File Locations for Quick Navigation**:

| Concept | File Path | Context |
|---------|-----------|---------|
| **Execution Entry** | `__main__.py` | Bootstrap (lines 1-2) → invokes _bootstrap |
| **CLI Dispatcher** | `alm/main.py` | Main command router (lines 37-95) |
| **Single Scenario** | `alm/model/main_client.py` | run-client handler (line 901+) |
| **Multi-Scenario** | `alm/model/main_server.py` | run-server handler |
| **Phase Control** | `alm/model/main_client.py:run_client()` | 5-phase orchestration (lines 72-100) |
| **Settings Loading** | `alm/model/loaders/settings_loader.py` | Excel → Settings object |
| **Asset Loading** | `alm/model/loaders/asset_loader.py` | Excel → Portfolio object |
| **SBA Optimization** | `alm/model/projection/sba_goal_seeker.py` | Core algorithm (~1000+ lines) |
| **Rebalancing** | `alm/model/projection/alm_rebalancing.py` | Hedging execution |
| **Excel Output** | `alm/model/writer/writer_excel.py` | Results export |

---

## Section 2: The 6 Core Modules Explained

### 2.1 Module Roles & Responsibilities

| # | Module | What It Does | Key Question |
|---|--------|---|---|
| 1 | **loaders/** | Read Excel setup/assets → Python objects | "How do I load configuration from Excel?" |
| 2 | **instruments/** | Define bonds, swaps, liabilities | "What does each asset/liability know?" |
| 3 | **projection/** | **CORE** — Calculate SBA BEL via goal-seeking | "How is theta/BEL calculated?" |
| 4 | **markets/** | Supply interest rates, FX, volatility | "What market data feeds the projection?" |
| 5 | **valuation/** | Price instruments using rates | "What's the PV of this bond's cashflows?" |
| 6 | **writer/** | Write results to Excel | "How are results exported?" |

---

## Section 3: Module Description & Dependencies

### 3.1 Loaders Module (Input Layer)

**Purpose**: Bridge Excel configuration → Python dataclass objects

**Key Files**:
- `settings_loader.py` — Reads `alm_setup_*.xlsm` → Settings object
- `asset_loader.py` — Reads `asset_data_*.xlsx` → Portfolio object
- `asset_parameter_provider.py` — Lookup tables for asset parameters
- (4 files, ~1000 lines total)

**What It Does**:
1. Opens Excel workbook
2. Reads named ranges + tables
3. Validates data types + ranges
4. Converts to strongly-typed Python dataclasses
5. Returns validated Settings + Portfolio objects

**Example Usage** (conceptual):
```python
settings = SettingsLoader("alm_setup_00_Base.xlsm").load()
portfolio = AssetLoader("asset_data_2023-Q4.xlsx").load()
# Now we have type-safe Python objects, not raw Excel
```

**Key Outputs**: `Settings` object (parameters) + `Portfolio` object (assets/liabilities)

**Mastery Target**: Level 2 — Understand the types of objects created + configuration flow

---

### 3.2 Instruments Module (Data Definition Layer)

**Purpose**: Define what each asset/liability is and how to value it

**Key Files**:
- `asset.py` — Base Asset class
- `bond.py` — FixedRateBond, CallableFixedRateBond, FloatingRateBond subclasses
- `swap.py` — VanillaSwap, Swaption subclasses
- `liability_cashflow.py` — Liability cashflow definitions
- (7+ files, ~1500 lines total)

**What It Does**:
- Defines each instrument type (Bond, Swap, etc.)
- Stores static attributes (coupon, maturity, rating, etc.)
- Implements pricing/valuation logic per instrument
- Tracks lifecycle events (maturity, call dates, etc.)

**Example** (conceptual):
```python
bond = FixedRateBond(
    isin="XS987654",
    coupon=2.5,
    maturity="2028-12-31",
    rating="A"
)
# Bond knows how to price itself, when it matures, if it's callable, etc.
```

**Key Outputs**: Strongly-typed asset/liability objects that projection can use

**Mastery Target**: Level 2 — Understand what attributes/methods each instrument has

---

### 3.3 Projection Module (Core Engine) 🔴 MOST COMPLEX

**Purpose**: Calculate SBA BEL through multi-year, multi-scenario projection with rebalancing

**Key Files**:
- `sba_goal_seeker.py` — Optimal asset/liability mix algorithm (hybrid linear/bisection)
- `balance_sheet_projection.py` — Project 100 years forward
- `alm_rebalancing.py` — Rebalancing strategy execution
- `swap_rebalancer.py` — Swap-based hedging
- (6+ files, ~3000+ lines total)

**What It Does** (High-Level Orchestration):
1. Load market scenarios (base + 9 stress cases per Article 28(7))
2. Initialize portfolio (assets + liabilities at valuation date)
3. For each scenario:
   - For each year (100 years forward):
     - Apply market movements (interest rates, FX, etc.)
     - Rebalance portfolio (match durations, execute swaps)
     - Check constraints (GSP limits, caps, etc.)
     - Recalculate values
4. Calculate SBA theta (goal-seek to find optimal hedge)
5. Output SBA BEL (present value under Article 28 formula)

**Core Algorithm** (SBA Goal-Seeking):
```
Goal: Find theta (liability hedging proportion) that minimizes Variance
This involves:
  1. Linear approximation (quick estimate)
  2. Bisection refinement (fine-tuning)
  3. Convergence check (tolerance-based stop)
```

**Key Inputs**: Settings + Portfolio objects (from loaders/instruments)

**Key Outputs**: 
- theta (optimal hedging level)
- SBA BEL (projected liability value)
- Rebalancing actions (buy/sell/swap amounts)

**Mastery Target**: Level 2-3 — Understand the 3-phase projection flow; **NOT** the mathematical optimization details

---

### 3.4 Markets Module (External Data Provider)

**Purpose**: Supply interest rate curves, FX rates, volatility for all scenarios

**Key Files**:
- `market_data.py` — Current market data (spot rates, FX, spreads)
- `market_projection.py` — Project market movements under 9 BMA scenarios
- (2-3 files, ~800 lines total)

**What It Does**:
1. Load base interest rate curve (from market data file or Excel)
2. For each scenario (Base, stress 1-8):
   - Apply market shocks (rates up/down, credit spreads, etc.)
   - Generate 100 forward spot curves (one per year)
3. Provide rates on demand to valuation module

**Example**:
```python
market = MarketProjection()
curve_year5_scenario2 = market.get_spot_curve(
    scenario="SBA_2_EquityShock",
    year=5
)
# Returns array of rates [0.5%, 0.6%, 0.8%, 1.2%, ...]
```

**Key Outputs**: 9 × 100 spot curves (rates for all scenarios + years)

**Mastery Target**: Level 1-2 — Know that market data drives valuations; understand the 9 BMA scenarios

---

### 3.5 Valuation Module (Pricing Engine)

**Purpose**: Calculate present value of instrument cashflows using market rates

**Key Files**:
- `cashflows.py` — Generate bond/swap/liability cashflow schedules
- `asset_update.py` — Revalue assets as market conditions change
- (2 files, ~600 lines total)

**What It Does**:
1. Receive cashflow schedule + market rates
2. Discount future cashflows to present value
3. Handle embedded options (callable bonds: if call option is in-the-money)
4. Return PV for each asset/liability

**Example** (Valuation of 2.5% bond):
```
Cashflows:     [2.5, 2.5, 2.5, ..., 102.5] (semi-annual coupons + principal)
Market Rates:  [0.5%, 0.6%, 0.8%, ...] (spot curve)
PV = 2.5/(1+0.5%)^0.5 + 2.5/(1+0.6%)^1.0 + ... + 102.5/(1+...)^10.0
   = 98.50 (price per 100 par)
```

**Key Outputs**: PV for each asset/liability (used by projection for rebalancing)

**Mastery Target**: Level 1-2 — Basic DCF concept; code uses QuantLib for complex features

---

### 3.6 Writer Module (Output Generation)

**Purpose**: Export SBA BEL results + intermediate data to Excel

**Key Files**:
- `writer_excel.py` — Workbook generation (5+ sheets with results)
- (1 file, ~400 lines total)

**What It Does**:
1. Create workbook with multiple sheets:
   - Summary (theta, SBA BEL, valuation date, scenario)
   - Assets (asset values by year)
   - Rebalancing (buy/sell/swap actions)
   - Allocation (weight % by asset class)
   - Validation checks (constraints, limits, warnings)
2. Format cells (colors, fonts, number formatting)
3. Add pivot tables / charts (if configured)
4. Save to `04. Outputs/[scenario]/summary_*.xlsx`

**Key Inputs**: Projection results (theta, valuations, rebalancing)

**Key Outputs**: Excel workbook ready for analysis

**Mastery Target**: Level 1 — Know that output is Excel; technical details less important

---

## Section 4: Data Flow (End-to-End)

### 4.1 The Complete Pipeline

```
LAYER 1: INTERFACE & INPUT
═══════════════════════════════════════════════════════════════
  Excel Files (User Configuration)
    ├─ alm_setup_00_Base.xlsm (99 parameters: risk factors, limits, etc.)
    ├─ alm_setup_01_No_Derivatives.xlsm (alternative scenario)
    └─ asset_data_2023-Q4.xlsx (portfolio: 500+ bonds, swaps, etc.)
         │
         ↓ [data_conversion/ — optional preprocessing]
         │   (If data is from vendor: normalize + SBA filter)
         │
         ↓ User runs: python -m alm run-client --run alm_setup_00_Base.xlsm


LAYER 2: CONFIGURATION LOADING
═══════════════════════════════════════════════════════════════
  loaders/ Module
    ├─ SettingsLoader("alm_setup_00_Base.xlsm")
    │   ├─ Reads named ranges (risk_free_rate, liquidity_premium, etc.)
    │   ├─ Reads lookup tables (asset classes, rating limits, targets)
    │   └─ → Returns Settings object (100+ typed parameters)
    │
    └─ AssetLoader("asset_data_2023-Q4.xlsx")
        ├─ Reads "Assets" sheet (500 rows)
        ├─ For each row: creates Bond/Swap/Liability object
        └─ → Returns Portfolio object (500+ instruments)


LAYER 3: INSTRUMENT DEFINITION
═══════════════════════════════════════════════════════════════
  instruments/ Module
    ├─ Bond objects (with coupon, maturity, rating, callable status)
    ├─ Swap objects (with fixed rate, tenor, collateral terms)
    ├─ Liability objects (cashflow schedule per business unit)
    └─ All objects are strongly typed + validated


LAYER 4: MARKET DATA PROVIDER
═══════════════════════════════════════════════════════════════
  markets/ Module
    ├─ Load base interest rate curve
    ├─ Generate 9 scenario × 100-year spot curves
    │   (Base + 8 BMA stress scenarios per Article 28(7))
    └─ → Curves ready for valuation


LAYER 5: PROJECTION (CORE ENGINE)
═══════════════════════════════════════════════════════════════
  projection/ Module (main_client.py orchestrates)
    │
    For each scenario (Base, Stress1-8):
      For each year (0 to 100):
        │
        ├─ valuation/ Module
        │   ├─ Calculate PV of all assets (using market rates)
        │   └─ Calculate PV of all liabilities
        │
        ├─ Rebalancing Strategy
        │   ├─ KRD matching (align asset/liability durations)
        │   ├─ TAA (tactical asset allocation per constraints)
        │   └─ Swap execution (if needed for hedging)
        │
        └─ Check constraints
            ├─ GSP limits (global solvency premium)
            ├─ Rating requirements
            └─ Proceed to next year
    │
    After all years projected:
    ├─ Goal-Seeker (sba_goal_seeker.py)
    │   ├─ Find optimal hedge ratio (theta)
    │   └─ Calculate minimum variance solution
    │
    └─ → Returns SBA BEL (present value per Article 28)


LAYER 6: OUTPUT
═══════════════════════════════════════════════════════════════
  writer/ Module
    ├─ Format results into Excel workbook
    ├─ Create sheets:
    │   ├─ Summary (theta, SBA BEL, key metrics)
    │   ├─ Asset Values (PV evolution by year)
    │   ├─ Rebalancing (actions taken each year)
    │   ├─ Allocation (portfolio weight% over time)
    │   └─ Validation (constraint checks)
    │
    └─ → Save to 04. Outputs/00_Base/summary_2023-12-31.xlsx
```

### 4.2 Data Flow in Words

1. **User provides**: Two Excel files (settings + asset data)
2. **Loaders read** Excel → Type-safe Python objects
3. **Instruments** represent those objects in memory
4. **Markets** provides interest rate scenarios
5. **Projection** runs 900-year scenarios (9 scenarios × 100 years):
   - For each year: value assets + liabilities using market rates
   - Rebalance portfolio to meet constraints
   - Track all changes
6. **Valuation** calculates PV whenever needed
7. **Goal-Seeker** finds optimal hedge after all years projected
8. **Writer** exports results to Excel

---

## Section 5: Module Dependencies (Dependency Graph)

### 5.1 Unidirectional Data Flow

**CRITICAL PRINCIPLE**: Data flows in ONE direction (no circular dependencies).

```
                main.py (dispatcher)
                    ↓
              main_client.py (orchestrator)
                    ↓
Loaders ←─────────▶ Settings + Portfolio
  ↓                           ↓  ↓
  └──────────→ Instruments ◀──┘  │
               ↓                   │
         Projection ◀─────────────┘
           ↙       ↘
       Markets   Valuation ◀──→ Instruments
           ↓          ↓
        (rates)   (PV calculations)
                    ↓
              Rebalancing
                    ↓
                Writer
                    ↓
              Excel Output
```

### 5.2 What This Means

- ✅ **Loaders** → Everything (initialization)
- ✅ **Instruments** → Projection (definitions of what to value)
- ✅ **Markets** → Valuation (supplies rates)
- ✅ **Valuation** → Projection (prices assets/liabilities)
- ✅ **Projection** → Writer (results to export)
- ❌ **NO** backwards dependencies (can't write first, then project)

---

## Section 5a: Governance Gaps & Known Issues

While the architecture is sound, there are 3 governance gaps that should be tracked for Phase 3 improvement:

### GAP-1: Module Dependency Validation Missing 🟠 MEDIUM

**Issue ID**: B01.X-GAP-MODULE-DEPENDENCIES  
**What's Missing**: No runtime validation that modules are loaded in the correct order

**Evidence**:
- **File**: `main_client.py` lines 70-85 (orchestration logic)
- **Current**: Assumes developer won't call phases out of sequence
- **Impact**: If module order violated, cascading failures occur

**BMA Rule**: Article 28 compliance requires complete, validated initialization

**Current Behavior**: If someone manually calls Phase 4 before Phase 1, code crashes with cryptic error

**Expected Behavior**: init_pipeline() should validate:
```
IF Phase1_Settings is None AND Phase2_Portfolio is None:
    RAISE ConfigurationError("Must call load_settings()")
```

**Severity**: 🟠 MEDIUM (affects usability, not correctness in normal flow)

**Phase 3 Recommendation**: Add initialization guards in orchestration

---

### GAP-2: Execution Mode Validation Missing 🟡 LOWER

**Issue ID**: B01.X-GAP-EXECUTION-MODES  
**What's Missing**: No validation that chosen execution mode (run-client vs run-server) matches setup file

**Evidence**:
- **File**: `main.py` lines 40-95 (CLI dispatcher)
- **Current**: User can pass run-server flag with single setup file (wasteful)

**BMA Rule**: Efficient resource usage expected in production

**Phase 3 Recommendation**: Warn if run-server used with 1 setup file (use run-client instead)

---

### GAP-3: Configuration Completeness Check Missing 🟠 MEDIUM

**Issue ID**: B01.X-GAP-CONFIG-COMPLETENESS  
**What's Missing**: No validation that all required settings sheets exist in Excel (e.g., Parameters, Market Data, Overrides)

**Evidence**:
- **File**: `loaders/settings_loader.py` (referenced from B03 audit)
- **Current**: Individual loaders look for sheets independently
- **Impact**: If Parameters sheet missing, user sees confusing openpyxl error, not application error

**BMA Rule**: Article 28 requires complete, valid configuration

**Phase 3 Recommendation**: Pre-flight check in run_client() that validates all required sheets exist before processing

---

## Section 5b: Regulatory Context — Why This Architecture Matters for SBA Compliance

### BMA Article 28: Enforcement Through Modularity

**Article 28** (BMA 2024 Insurance Group Solvency Requirement Amendment Rules) requires:
1. **Complete configuration** (Article 28: all parameters must be specified)
2. **Registered assets only** (Article 28(6): only eligible assets in SBA calculation)
3. **Correct calculations** (Article 28(8-10): specific SBA formulas)
4. **Multi-scenario projection** (Article 28(7): 9 scenarios including 8 market shocks)

**How Athora's architecture enforces this:**

| Requirement | Enforced By | Module |
|-----------|-----------|--------|
| Complete config | Settings Loader validates all parameters | `loaders/settings_loader.py` |
| Eligible assets | Asset Loader filters per Article 28(6) | `loaders/asset_loader.py` |
| Correct calculations | Projection engine implements SBA math | `projection/sba_goal_seeker.py` |
| 9 scenarios | Markets module generates 9 curves | `markets/market_projection.py` |

**Why Architecture → Auditability**: Because modules are separated, an auditor can:
- Verify asset loader enforces eligibility rules (inspect one file)
- Verify projection implements correct formulas (inspect project module, not entire codebase)
- Trace configuration → assume → result (clean data flow)

**Gap for Phase 3**: Architecture enforces structure, but validation is weak (see GAP-1, GAP-2, GAP-3 above)

---

## Section 6: CLI Entry Points & Execution Modes

The model has multiple execution modes. All are accessed via `main.py`.

### 6.1 Command List

```bash
python -m alm --help
```

This shows 6+ modes:

| Mode | Purpose | When to Use |
|------|---------|-----------|
| `run-client` | Single scenario valuation (one setup file) | Standard SBA BEL calculation |
| `run-server` | Multi-scenario batch with orchestration | Large portfolio runs (multiple BUs) |
| `convert1` | Data standardization (vendor → standard) | Data prep before first run |
| `convert2` | SBA eligibility filtering | Data prep before first run |
| `merge` | Consolidate multiple output files | Combine results across scenarios |
| `extract` | Extract specific metrics from output | Reporting/analytics |
| `package` | Create .pyz distribution | Deployment |

### 6.2 Standard User Workflow

**Scenario 1: Single Valuation (Most Common)**
```bash
# Run 1 scenario (e.g., Base case)
python -m alm run-client --run alm_setup_00_Base.xlsm

# Output: 04. Outputs/00_Base/summary_*.xlsx
```

**Scenario 2: Multi-Scenario Batch**
```bash
# Create runs.txt listing all setup files (one per line):
# alm_setup_00_Base.xlsm
# alm_setup_01_No_Derivatives.xlsm
# ... (9 total for BMA scenarios)

python -m alm run-server --runs runs.txt --log execution.log --queue

# Output: 04. Outputs/00_Base/, 01_No_Derivatives/, etc. (one per scenario)
```

**Scenario 3: Data Preparation**
```bash
# Step 1: Standardize vendor CSV
python -m alm convert1 --input vendor_data.csv --output standardized.xlsx

# Step 2: Apply SBA filters
python -m alm convert2 --input standardized.xlsx

# Output: SBA-eligible dataset ready for projection
```

### 6.3 Running the Model in Development

For development/debugging (running code directly without .pyz):

```bash
# If you extract the .pyz or have source code:
cd extracted_dir
python -m alm run-client --run alm_setup_00_Base.xlsm --debug
```

---

## Section 7: The 5-Phase Execution Pipeline (Inside run-client)

**This section explains** what happens when you run:
```bash
python -m alm run-client --run alm_setup_00_Base.xlsm
```

### 7.1 Five-Phase Model with Caching

Inside `main_client.py:run_client()` (lines 72–100), the model executes in phases with **caching behavior**:

```
PHASE 1: Load Settings (One-time, cached)
├─ File: main_client.py, line 75
├─ Action: SettingsLoader("alm_setup_00_Base.xlsm") → Settings object
├─ Cache Flag: initialised_settings (prevents reload)
└─ Output: All Excel parameters in memory

    ↓
    
PHASE 2: Initialize Model (One-time, cached)
├─ File: main_client.py, line 82
├─ Action: Create instruments, markets, balance sheet
├─ Components Created:
│   ├─ AssetParameterProvider (lookup tables)
│   ├─ Asset Factory (Bond, Swap, ..., 100+ objects)
│   ├─ MarketProjection (90 spot curves: 9 scenarios × 10 years)
│   ├─ BalanceSheet (container)
│   └─ Rebalancing components
├─ Cache Flag: initialised_model (prevents reload)
└─ Output: All components ready to project

    ↓
    
PHASE 3: Calibrate Model (One-time, follows Phase 2)
├─ File: main_client.py, line 83
├─ Action: Calibrate z-spreads, interest rate models
├─ Purpose: Fit real-world data to model parameters
└─ Output: Calibrated models ready for projection

    ↓
    
PHASE 4A: Valuation — Liabilities (Iterative, runs many times)
├─ File: main_client.py, line 85–87
├─ Action: Project liability forward 100 years × 9 scenarios
├─ Called By: SB Goal-Seeker (as part of optimization loop)
├─ Runs: Multiple times as goal-seeker tries different hedge ratios
└─ Output: Liability values under each scenario/year

    + (Parallel operation)
    
PHASE 4B: Valuation — Assets (Iterative, runs many times)
├─ File: main_client.py, line 89
├─ Action: Project assets forward, apply rebalancing, calculate SBA BEL
├─ Called By: SBA Goal-Seeker (as part of optimization loop)
├─ Key Sub-Steps:
│   ├─ Value all assets (using market rates)
│   ├─ Execute rebalancing (buy/sell/swap)
│   ├─ Check constraints (GSP, ratings, limits)
│   └─ Calculate interim SBA BEL
├─ Runs: Multiple times as goal-seeker iterates
└─ Output: Optimal asset allocation + final SBA BEL

    ↓ (After both 4A & 4B complete)
    
PHASE 5: Write Results (One-time, final)
├─ File: main_client.py, line 91
├─ Action: Format results into Excel workbook
├─ Sheets Created:
│   ├─ Summary (theta, SBA BEL, key metrics)
│   ├─ Asset Values (evolution by year)
│   ├─ Rebalancing (trades executed)
│   ├─ Allocation (portfolio weights)
│   └─ Validation (constraint checks)
└─ Output: 04. Outputs/00_Base/summary_*.xlsx
```

### 7.2 Why Caching Matters

**Performance Implication**:
- Phases 1-3 run **once** (expensive setup)
- Phases 4A-4B run **many times** (goal-seek loop, e.g., 50-100 iterations)
- **Without caching**: Phases 1-3 would re-run 50-100 times = 50-100x slower

**Code Example** (pseudocode):

```python
def run_client(siv):
    global initialised_settings, initialised_model
    
    # Phase 1: Load settings (if not already done)
    if not initialised_settings:
        initalise_settings(siv)           # First call: initialize
        initialised_settings = True        # Mark done
    # If called again: skip (cached)
    
    # Phase 2-3: Initialize & calibrate (if not already done)
    if not initialised_model:
        initialise_model(siv)              # First call: setup
        calibrate_model(siv)               # First call: calibrate
        initialised_model = True           # Mark done
    # If called again: skip (cached)
    
    # Phase 4A-4B: Valuation (runs every time, NOT cached)
    theta = project_liability(siv)         # Goal-seek loop
    theta = project_assets(siv)            # Goal-seek loop
    # These run 50-100 times per model run
    
    # Phase 5: Write (runs once at end)
    write_results(siv)
```

### 7.3 Key Insight: Where Goal-Seeking Happens

**The goal-seeking loop** is embedded in **Phase 4B** (asset projection):

```
Goal-Seeker Algorithm (inside project_assets):

    FOR hedge_ratio FROM 0% TO 100% (step 1%):
        Set portfolio hedging to hedge_ratio%
        Project balance sheet forward 100 years
        Calculate SBA BEL under this hedge ratio
        Record result
    
    Find hedge_ratio that minimizes SBA BEL variance
    → This is theta (optimal hedge)
    → Return theta + final SBA BEL
```

This is why Phase 4 runs many times — the goal-seeker tests multiple hedge ratios to find the optimal one.

### 7.4 Execution Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│  User Command: python -m alm run-client --run setup.xlsx   │
└──────────────────────────┬──────────────────────────────────┘
                           │
                    ┌──────▼─────┐
                    │ Bootstrap  │
                    └──────┬─────┘
                           │
              ┌────────────▼────────────┐
              │ main.py (CLI dispatcher)│
              │ Parses args, route      │
              └────────────┬────────────┘
                           │
                  ┌────────▼──────┐
                  │ run-client()  │
                  │ main_client   │
                  └────────┬──────┘
                           │
        ┌──────────────────┼──────────────────┐
        │ PHASE 1: PHASE 2-3: PHASE 4: PHASE 5:
        │ Load Settings Init+Calib Value Results
        │ (once)  (once) (50-100x) (once)
        │  │         │       │        │
        │  ├─────────┤       │        │
        └──┼─────────┼───────┼────────┼─────┐
           ▼         ▼       │        │     │
        Cache   Cache  Loop  Output  Done  │
     Settings  Model  Goal-  Excel   ✓     │
              & Init  Seek         Cached  │
              Calib
                    │        │
                    └────────┘
                   (Reuse cached
                    Phase 1-3)
```

### 7.5 Code References for Phase Management

| Phase | File | Function | Lines | Purpose |
|-------|------|----------|-------|---------|
| 1 | main_client.py | initalise_settings() | ~50-60 | Parse Excel → Settings |
| 2 | main_client.py | initialise_model() | ~82-150 | Create all components |
| 3 | main_client.py | calibrate_model() | - | Fit model parameters |
| 4A | main_client.py | project_liability() | - | Discount liability CF |
| 4B | main_client.py | project_assets() + SBAGoalSeeker | ~80-300 (complex) | Core optimization |
| 5 | writer/ | write_results() | - | Export to Excel |

---

## Section 8: Key Takeaways (Mastery Checkpoint)

By now, you should understand:

- ✅ **6 core modules** + their roles:
  - Loaders (input) → Instruments (definitions) → Projection (**core**) ← Markets (rates)
  - ↓ Valuation (pricing) → Rebalancing ↓
  - Writer (output)

- ✅ **Data flow**: Excel → Loaders → Instruments → Projection + Markets + Valuation → Writer → Excel

- ✅ **5-phase execution**: Load settings (1×) → Init+Calibrate (1×) → Value (50-100×) → Write (1×)

- ✅ **Caching behavior**: Phases 1-3 cached to avoid re-initialization; Phase 4 iterates for goal-seek

- ✅ **3 support modules**: data_conversion (preprocessing), packing (distribution), post_proc (merge/extract)

- ✅ **CLI modes**: 7 execution modes with clear purposes (run-client, run-server, convert1/2, etc.)

- ✅ **No circular dependencies**: Data flows one direction; this keeps architecture clean

---

## Section 9: Testing Your Mastery

**Can you answer these questions?**

1. ❓ **Module roles**: What does `projection/` do that `valuation/` doesn't?
   - ➜ Valuation prices individual assets; Projection orchestrates the multi-year, multi-scenario simulation
   
2. ❓ **Data flow**: How do assets get from Excel into Python objects that projection can use?
   - ➜ Excel → AssetLoader → Asset() objects → projection receives as Portfolio parameter
   
3. ❓ **Dependencies**: Which module depends on markets/?
   - ➜ Valuation (uses interest rates) and Projection (passes rates to valuation)
   
4. ❓ **Execution**: When would you use `run-server` vs `run-client`?
   - ➜ run-client: single scenario; run-server: multiple scenarios with batch orchestration
   
5. ❓ **Phases**: Why does Phase 4 run many times but Phase 1 doesn't?
   - ➜ Goal-seeker iterates over asset allocations (Phase 4); settings only needed once (Phase 1)
   
6. ❓ **Draw from memory**: Sketch the 5-phase execution + caching behavior without this document

---

## Section 10: Next Steps by Interest

Choose where to go next based on what you're curious about:

### 🔍 **If Curious About Data Flow & Configuration**
→ Study **[B03: Configuration Loading Pipeline](../B03-Loaders/B03_Configuration_Loading_Pipeline.md)** next
- How Excel setup.xlsm becomes Settings object
- How asset data becomes Portfolio object
- Parameter validation & error handling

### 💰 **If Curious About Assets & Instruments**
→ Study **[B04: Asset Instrument Definitions](../B04-Instruments/B04_Asset_Instrument_Definitions.md)** next
- What Bond, Swap, Equity, Liability classes contain
- How each instrument is valued
- Instrument-specific features (callability, floating rates, etc.)

### 📊 **If Curious About SBA Calculation** ⭐ (MOST IMPORTANT)
→ Study **[B05: ALM Projection and SBA BEL Calculation](../B05-Projection/B05_ALM_Projection_and_SBA_BEL_Calculation.md)** next
- This is where the CORE algorithm lives
- Goal-seeking optimization
- SBA BEL calculation per Article 28
- ⚠️ Most complex module (3000+ lines)

### 📈 **If Curious About Market Scenarios & Interest Rates**
→ Study **[B06: Market Data and Scenarios](../B06-Markets/B06_Market_Data_and_Scenarios.md)** next
- How 9 BMA stress scenarios are defined
- How spot curves are generated
- Market projection logic

### 📄 **If Curious About Output & Reporting**
→ Study **[B08: Output Generation and Excel Writer](../B08-Output_Generation_and_Excel_Writer.md)** next
- How results are formatted for Excel
- What sheets are created
- Post-processing and data validation

### ⚙️ **If Curious About Orchestration & Control Flow**
→ Study **[B09: Model Orchestration and Execution](../B09-Orchestration/B09_Model_Orchestration_and_Execution.md)** next
- How main_client.py coordinates phases
- How run-server orchestrates multi-scenario batch
- Logging and error handling

---

## Section 11: References

### Key Documents
- [B01 Enhancement Guide: Code Findings & Suggestions](B01_ENHANCEMENT_GUIDE_Code_Findings_and_Suggestions.md) — Detailed findings from Explore agent; shows what enhancements were made
- [Entry Points & Initialization Architecture](_References/CODE_ANALYSIS/ENTRY_POINTS_AND_INITIALIZATION_ARCHITECTURE.md) — Deep technical reference (exact line numbers, class names)

### Related Learning Modules
- [Athora SBA Model Codebase Exploration](../_References/CODE_ANALYSIS/ATHORA_SBA_MODEL_CODEBASE_EXPLORATION.md) — Original architectural overview
- [Module Dependency Graph Explanation](../_References/CODE_ANALYSIS/) — Module relationships

### Regulatory Context
- [BMA SBA Consolidated Guide](../../../05.%20Learning/01_Regulatory_Framework/BMA_SBA_Consolidated_Guide.md) — Article 28 requirements (regulatory authority)
- [Athora Codebase Mastery Checklist B01 Items](../../../../ATHORA_CODEBASE_MASTERY_CHECKLIST.md#b1-codebase-architecture-overview) — B1.1, B1.2, B1.3

---

## Section 12: Verification Checklist (Level 2-3 Mastery)

After completing B01, verify your understanding with this checklist:

**Level 2 (Architecture Understanding)**
- [ ] Can explain the role of each 6 core modules in 1-2 sentences each
- [ ] Can draw the dependency graph (modules + data flow)
- [ ] Can explain where Settings object comes from + where Portfolio object comes from
- [ ] Can name 3-4 types of instruments (Bond, Swap, etc.)
- [ ] Can explain why markets are needed in projection
- [ ] Understand CLI modes: run-client, run-server, convert1/2

**Level 3 (Execution Understanding)**
- [ ] Can explain the 5-phase execution model
- [ ] Can explain caching behavior (why Phase 1 runs once, Phase 4 runs many times)
- [ ] Can locate main_client.py and find line ~82 (initialise_model call)
- [ ] Can explain what goal-seeking does (find optimal hedge ratio)
- [ ] Can trace data: Excel → Loaders → Instruments → Projection → Writer → Excel
- [ ] Can explain why one-directional data flow matters (no circular dependencies)

**Next Action After Mastery**:
→ Move to **B02: Data Conversion Pipeline** using the same learning approach

---

## APPENDIX: Common Mistakes When Learning B01

❌ **Mistake 1**: "Phase 4 only runs once"
✅ **Reality**: Phase 4 runs multiple times inside the goal-seek loop (typically 50-100 times)

❌ **Mistake 2**: "All 6 modules run in single run-client call"
✅ **Reality**: Modules 1 (data_conversion) and 6.5 (post_proc) are separate commands. run-client uses loaders, instruments, projection, markets, valuation, writer

❌ **Mistake 3**: "I can read main.py to understand full execution"
✅ **Reality**: main.py is just the CLI dispatcher (95 lines). True execution logic is in main_client.py (900+ lines)

❌ **Mistake 4**: "Initialization order doesn't matter"
✅ **Reality**: 8-step initialization order is strict. Each step depends on prior steps (Settings → Parameters → Assets → Markets → ...)

❌ **Mistake 5**: "Caching is an optimization detail"
✅ **Reality**: Caching determines whether model runs in seconds or minutes. Phase 1 reload would slow execution 50-100×

---

**Status**: ✅ COMPLETE — Enhanced with High Priority improvements  
**Last Updated**: March 31, 2026  
**Improvements Made**:
1. ✅ Added CLI Dispatch Table (Section 1.4)
2. ✅ Added Code References throughout
3. ✅ Added 5-Phase Execution Model (Section 7, with caching details)
4. ✅ Added Execution Flow Diagram
5. ✅ Added "Next Steps by Interest" navigation (Section 10)
6. ✅ Added Verification Checklist (Section 12)

**Mastery Level After This Module**: Level 2–3 (can explain architecture + trace data flow + understand execution phases)



**Status**: ✅ COMPLETE — Ready for learning or reference  
**Last Updated**: March 31, 2026  
**Mastery Level After This Module**: Level 2–3 (can explain architecture + trace data flow)

**What**: Reads Excel configuration files and converts them to Python dataclass objects

**When Used**: At model initialization (every run)

**Key Components**:
- `settings_loader.py` — Reads Parameters sheet from setup.xlsm
- `asset_loader.py` — Reads Asset sheets
- `asset_parameter_provider.py` — Provides D&D rates, durations, spreads from lookup tables
- `lookup_table/` — Dynamic parameter lookup system

**Example**:
```python
settings = SettingsLoader("setup.xlsm).load()
# Result: Settings object with all parameters populated
```

**Importance**: Getting this right determines model behavior. A configuration error here corrupts all downstream calculations.

---

### 2. Instruments (Data Structure Layer)

**What**: Defines what each asset/liability is and how it's valued

**When Used**: During projection (every time step)

**Key Components**:
- `asset.py` — Base Asset class with valuation logic
- `assets/bond.py` — Bond implementation (coupons, maturity, callability)
- `assets/swap.py` — Interest rate swap implementation
- `assets/equity_index.py` — Stock/equity implementation
- `instruments/liability_cashflow.py` — Liability cashflow streams
- `asset_generation.py` — Factory for creating instrument types

**Each Instrument Can**:
- Calculate its present value (given market rates)
- Generate its cashflow schedule
- Evaluate constraints (e.g., callable bonds may call)

**Example**:
```python
bond = Bond(isin="XS987654", coupon=2.5, maturity="2028-12-31", rating="A")
pv = bond.present_value(market_rates=[0.5%, 0.6%, ...])  # Uses market rates
```

---

### 3. Projection (Business Logic — CORE ENGINE)

**What**: Runs the SBA BEL calculation — projects balance sheet forward 100 years × 9 scenarios

**When Used**: During main calculation (the heart of the model)

**Key Components**:
- `sba_goal_seeker.py` — Hybrid linear/bisection algorithm to find optimal theta (asset proportion)
- `balance_sheet_projection.py` — Recursive projection loop (time=0 to T)
- `alm_rebalancing.py` — Tactical rebalancing strategies (KRD matching, SAA, CAA, etc.)
- `swap_rebalancer.py` — Synthetic swap generation for hedging
- `dv01_exposure.py` — Duration/convexity calculations
- `asset_projector.py` — Projects individual assets forward in time
- `market_projection.py` — Manages market data projection through scenarios
- `zspread_provider.py` — Z-spread calibration and projection

**Workflow**:
```
1. Input: Portfolio + Liabilities + 9 Scenarios
2. For each scenario:
   a. Run goal-seeking algorithm (binary search for optimal theta)
   b. For each year (t=1 to 100):
      - Project market rates (Hull-White scenarios)
      - Project asset/liability values
      - Generate cashflows
      - Run rebalancing (if enabled)
      - Check constraints
   c. Output: theta, SBA BEL, spread
3. Result: 9 × (theta, BEL, spread) combinations
```

**Critical Calculations**:
- **Theta Scaling**: How much of portfolio is reinvested (vs. held to maturity)
- **Goal-Seek**: Finds theta that minimizes F(theta) = |Portfolio_PV(theta) − Liability_PV|
- **Rebalancing**: Annual KRD matching to maintain hedge effectiveness

---

### 4. Markets (External Data Layer)

**What**: Supplies market rates, FX, volatility; generates spot curves for all scenarios

**When Used**: During projection (every time step needs rates)

**Key Components**:
- `market_data.py` — MarketData class holding rates, volatilitie s, FX
- `market_projection.py` — Hull-White interest rate model for scenario generation
- `currency_mapper.py` — Maps currencies to base currency
- `index_manager.py` — Manages stock index fixings

**Example Output**: 90 spot curves (9 scenarios × 10 years) for term structure valuation

**Critical**: Market data quality directly affects SBA BEL calculation accuracy.

---

### 5. Valuation (Calculation Layer)

**What**: Calculates present value of asset/liability cashflows using market rates

**When Used**: During projection (pricing happens at every time step)

**Key Components**:
- `cashflows.py` — Cashflow stream representation and accumulation
- `asset_update.py` — Updates asset values during projection
- `asset_flows.py` — Asset cash flow calculations (interest, principal, trades)

**Uses**: QuantLib 1.30 for sophisticated instrument pricing (callables, floaters, etc.)

**Simple Example**:
```
Bond with coupon 2.5%, maturity 10 years, market rate 2.0%
PV = 2.5/(1+0.02)^1 + 2.5/(1+0.02)^2 + ... + 102.5/(1+0.02)^10
   ≈ 104.52
```

---

### 6. Writer (Output Layer)

**What**: Serializes results to Excel workbooks with multiple analysis sheets

**When Used**: After projection completes

**Output Sheets**:
- **Summary** — SBA BEL, spreads, success flags, matching quality metrics
- **Asset Values** — Per-asset valuations across scenarios/years
- **Allocation** — Asset class weights and concentrations
- **Rebalancing** — Trades executed (dates, quantities, costs)
- **Liability** — Liability projections and sensitivities

**Key Components**:
- `writer_excel.py` — Main Excel output writer
- `writer_collection.py` — Coordinates multiple writers
- `writer_inner_loop.py` — Manages output during projection

---

## Data Flow (Complete Picture)

```
INPUT LAYER (Loaders)
│ Excel setup.xlsm + assets.xlsm
│ ↓
│ SettingsLoader, AssetLoader
│ ↓
PYTHON OBJECTS
│
CALCULATION LAYER (Projection)
├─ Markets module provides spot curves per scenario/year
├─ Instruments module defines all assets/liabilities
├─ Projection module runs goal-seek + forward projection
├─ For each year in each scenario:
│  ├─ Valuation module calculates PV of each instrument
│  ├─ Rebalancing module optimizes allocation
│  └─ Results accumulated
└─ Final output: theta, BEL, spread per scenario

OUTPUT LAYER (Writer)
│ ↓
│ Excel output.xlsx (5+ sheets)
│
END RESULT
│ SBA BEL values, allocation, rebalancing trades, metrics
```

---

## Support Modules (Specialized)

### data_conversion/

**Purpose**: Transform raw portfolio data (from vendor) into standardized format

**When Used**: Before first model run (pre-processing step)

**Execution**: `python -m alm convert1 --input vendor_data.csv --output standardized.xlsx`

**Why Separate**: 
- Handles vendor-specific formats (Bloomberg, Reuters, etc.)
- Applies SBA eligibility filtering
- Decouples data standardization from model initialization

**Data Flow**:
```
Vendor Export CSV ──convert1──> Standardized Excel
    ↓                                ↓
Raw format                    SBA-eligible assets
    ↓                                ↓
                     ──convert2──> Final Data
                                    ↓
                            Loaders read this
```

### packing/

**Purpose**: Create .pyz distribution package (make executable)

**When Used**: During deployment/distribution

**Not needed** for running the model locally.

### post_proc/

**Purpose**: Post-processing tools (merge outputs, extract metrics)

**When Used**: After running multiple scenarios/BUs

**Commands**:
- `merge` — Combine multiple output files
- `extract` — Extract specific metrics from results

---

## Dependency Graph (Simplified)

```
[EXCEL INPUT]
     ↓
[loaders/] ← Reads Excel
     ↓
[instruments/] ← Creates Python objects
     ↓
[projection/] ← CORE: runs goal-seek + forward projection
  ├─ markets/ ← provides rates
  ├─ valuation/ ← values each instrument
  └─ alm_rebalancing/ ← rebalances portfolio
     ↓
[writer/] ← Outputs results
     ↓
[EXCEL OUTPUT]
```

**Key Rule**: Data flows one direction. Projection depends on loaders, instruments, markets, valuation. Write happens after projection completes.

---

## Execution Entry Points

### Standard Workflow
```bash
python -m alm run-client --config setup.xlsm
```

Invokes:
1. `main.py` — CLI dispatcher
2. `main_client.py` — Single scenario orchestration
3. Calls loaders/ → instruments/ → projection/ → writer/ → output

### Batch Processing
```bash
python -m alm run-server --runs runs.txt --queue
```

Invokes multi-scenario orchestration (main_server.py).

### Data Preprocessing
```bash
python -m alm convert1 --input raw.csv --output standardized.xlsx
python -m alm convert2 --input standardized.xlsx
```

---

## Key Insights for Phase 1 Mastery

1. **Loaders are critical**: Configuration errors here corrupt everything downstream
2. **Projection is where SBA BEL happens**: 80% of model logic is here
3. **Markets & Valuation are support**: They provide data projection uses
4. **Writer is output**: Multi-sheet Excel generation from calculated values
5. **Data flows one direction**: No feedback loops; deterministic calculations

---

## Verification Checklist

- [ ] Can you name the 6 core modules?
- [ ] Can you draw the dependency graph?
- [ ] Can you explain what each module does in 1 sentence?
- [ ] Can you trace data flow from Excel input to output?
- [ ] Can you identify which module handles your question: "How is theta optimized?" (Answer: projection/sba_goal_seeker.py)

---

## Deep-Dive References

For detailed code analysis and module-specific walkthroughs, see:

- [ATHORA_SBA_MODEL_CODEBASE_EXPLORATION.md](../_References/CODE_ANALYSIS/ATHORA_SBA_MODEL_CODEBASE_EXPLORATION.md) — Complete architectural overview
- [MODULE_AUDIT_data_conversion.md](../_References/MODULE_AUDITS/MODULE_AUDIT_data_conversion.md)
- [MODULE_AUDIT_model_loaders.md](../_References/MODULE_AUDITS/MODULE_AUDIT_model_loaders.md)
- [MODULE_AUDIT_model_instruments.md](../_References/MODULE_AUDITS/MODULE_AUDIT_model_instruments.md)
- [MODULE_AUDIT_model_projection.md](../_References/MODULE_AUDITS/MODULE_AUDIT_model_projection.md)
- [MODULE_AUDIT_model_markets.md](../_References/MODULE_AUDITS/MODULE_AUDIT_model_markets.md)
- [MODULE_AUDIT_model_valuation.md](../_References/MODULE_AUDITS/MODULE_AUDIT_model_valuation.md)
- [MODULE_AUDIT_model_writer.md](../_References/MODULE_AUDITS/MODULE_AUDIT_model_writer.md)

---

## Related Checklists

- [ATHORA_CODEBASE_MASTERY_CHECKLIST.md](../../../../ATHORA_CODEBASE_MASTERY_CHECKLIST.md) — Full 25-item mastery checklist
- [B02-B14 Learning Guides](../) — Deep dives into each module

---

**Last Updated**: March 31, 2026  
**Written For**: Phase 1, Part B (Codebase Mastery)  
**Status**: Core foundation complete; refer to module audits for detailed code analysis
