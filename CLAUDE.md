# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

BMA SBA Benchmark Model — a **regulator's independent verification tool** for the Bermuda Monetary Authority (BMA) to verify insurance companies' Scenario-Based Approach (SBA) submissions. This is a Python desktop calculation engine (not a web app), designed to run on standard Windows workstations with no infrastructure dependencies.

**Current status:** Planning phase. No source code exists yet. The repo contains project planning docs (`Project_Plan/`), regulatory reference material (`BMA_doc/`), implementation-ready algorithm specifications (`Algorithm_Specs/`), and prior-art reference from another SBA model (`Reference_Documents/`). Code implementation has not started.

## Target Architecture

When implementation begins, the package will be `bma_sba_benchmark/` with this structure:

- **rules/** — Versioned BMA rule modules (v2024, v2025...) with `@rule_ref` decorator for audit traceability. v2025+ only overrides what changed from the base year.
- **model_points/** — Pydantic-validated asset and liability data loaders (CSV/Excel input).
- **assumptions/** — Versioned YAML/CSV assumption tables (risk-free curves, credit floors, spreads).
- **curves/** — **The ONLY module that imports QuantLib.** Yield curve construction, bond pricing, scenario curve shifts. All QuantLib calls wrapped by `ql_audit.py` for logging.
- **projection/** — **Pure Python, no QuantLib.** Period-by-period cash flow projection, reinvestment/disinvestment, credit adjustments. This is where BMA rules live and must be fully transparent.
- **calculations/** — BEL, Risk Margin, LCR, stress tests.
- **engine/** — Run orchestration, batch processing, audit trail (SQLite + Parquet + JSON Lines).
- **challenge/** — Import company SBA submissions and compare against benchmark results.
- **output/** — Multi-tab regulatory Excel workbooks via openpyxl.

### Key Architectural Constraints

- **QuantLib boundary:** Only `curves/` imports QuantLib. All projection logic is pure Python for auditability.
- **Fail loudly:** Pydantic validation at data boundaries; no silent defaults. Invalid data stops the run.
- **Rule traceability:** Every calculation tagged with BMA rule paragraph via `@rule_ref` decorator.
- **Immutable runs:** `RunConfig` frozen after creation, inputs hashed for reproducibility.
- **Everything is a DataFrame:** Intermediate results are pandas DataFrames for inspectability.

## Technology Stack

Python 3.11+, QuantLib-Python (pinned version, conda-forge), pandas 2.x, NumPy, SciPy (brentq for C0 root-finding), Pydantic v2, openpyxl, PyYAML, PyArrow (Parquet), pytest + hypothesis, click/typer CLI.

## Development Commands (planned)

```bash
# Environment setup
conda create -n bma_sba python=3.11
conda activate bma_sba
conda install -c conda-forge quantlib-python
pip install -e ".[dev]"

# Run tests
pytest
pytest tests/test_illustrative_calc.py  # Golden integration test
pytest --cov=bma_sba_benchmark          # With coverage

# Run model
python -m bma_sba_benchmark run --company ACME --date 2024-12-31 --rules 2024
python -m bma_sba_benchmark batch --companies companies.csv --date 2024-12-31
python -m bma_sba_benchmark challenge --company ACME --submission acme_sba_2024.xlsx
```

## Domain Context

- **9 BMA interest rate scenarios** (base, decrease, increase, down-up, up-down, 4 twist variants) defined in Rules Schedule XXV Para 28(7)(a-i).
- **Biting scenario** = the one requiring the highest initial cash buffer (C0).
- **BEL** = MV_assets(T=0) + C0_biting. **TP** = BEL + Risk Margin (6% CoC).
- **Asset tiers:** Tier 1 (unrestricted), Tier 2 (BMA pre-approval), Tier 3 (10% cap, cannot be sold).
- **Credit costs:** Default + downgrade with phase-in (20% in 2024 → 100% by 2028).
- **Disinvestment waterfall:** Cash → Govt bonds → IG corp → Muni → Tier 2. Tier 3 cannot be sold.
- The BMA illustrative calculation (`BMA_doc/BMA_SBA_Illustrative_Calculation_Comprehensive.md`) is the golden integration test target (standalone, supersedes the earlier `BMA_SBA_Illustrative_Calculation.md`).

## Repository Layout

- `Algorithm_Specs/` — **Implementation-ready algorithm specifications** (10 docs). Each maps to a target code module with pseudocode, BMA rule references, and worked numerical examples. Start here when implementing any module.
- `Project_Plan/` — Project planning docs (01 through 07). Start with `01_Project_Charter.md`.
- `BMA_doc/` — BMA regulatory reference. `BMA_SBA_Consolidated_Guide.md` is the authoritative rule reference. `BMA_SBA_Illustrative_Calculation_Comprehensive.md` is the golden integration test target (standalone; includes simple 5-year intro case, full 10-year multi-asset example, SII vs SBA comparison notes, and all 15 sections). `BMA_SBA_Rebalancing_Reference.md` is an educational reference on ALM rebalancing mechanics (KRD/TAA) for challenging company submissions.
- `Reference_Documents/` — Prior-art reference from Pythora v1.0.0 (another SBA BEL model). Use for context only — NOT specs to implement directly. Key lessons already extracted into `Algorithm_Specs/`.
- `NAIC_Doc/` — Comparative regulatory docs (BMA SBA vs NAIC CFT).
- `Archive/`, `Python_Implementation/`, `Julia_Implementation/`, `Hybrid_Implementation/` — Superseded earlier architecture explorations. Retained for reference only.
