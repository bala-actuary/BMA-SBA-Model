# BMA SBA Benchmark Model

## Project Overview

This repository contains the planning, design, and reference documents for the **BMA (Bermuda Monetary Authority) SBA Regulatory Benchmark Model** - an independent calculation engine that BMA will use to verify and challenge insurance companies' Scenario-Based Approach (SBA) submissions.

The model is being built as a **transparent, auditable Python calculation engine** using **QuantLib-Python** for yield curve construction and bond pricing. Liability cash flows (EPL - External Projected Liabilities) are provided as external input; the model projects asset cash flows under BMA's 9 prescribed interest rate scenarios and determines the Best Estimate Liability (BEL), Risk Margin, Technical Provision, Liquidity Coverage Ratio, and mandatory stress test results.

### What Makes This a Benchmark Model

Unlike a company compliance model, this is a **regulator's independent verification tool**:

- **Rule Traceability** - Every calculation is tagged with its BMA rule paragraph reference via a `@rule_ref` decorator
- **Company Challenge** - Import a company's SBA submission and compare line-by-line against benchmark results
- **Rule Versioning** - BMA rule changes are handled through versioned rule modules (v2024, v2025, ...) without code rewrites
- **Complete Audit Trail** - Every run produces SQLite metadata, Parquet cash flow logs, and JSON rule traces
- **Transparent Projection Logic** - All projection logic is in pure Python (no black boxes); QuantLib is used only for curves and pricing
- **Multi-Company Batch Runs** - Run the benchmark for multiple companies in a single batch

## Key Capabilities

- **9 BMA Interest Rate Scenarios** - Base, Decrease, Increase, Down-Up, Up-Down, and 4 twist scenarios with prescribed shifts
- **Full Asset Universe** - Government/corporate/municipal bonds, callable bonds, floating-rate, amortizing, MBS, ABS, structured securities
- **BEL Calculation** - Biting scenario determination, initial cash buffer (C0) via root-finding
- **Risk Margin** - 6% Cost of Capital method with projected Modified ECR
- **LCR** - Liquidity Coverage Ratio (>= 105%) with fast-moving and sustained deterioration scenarios
- **Stress Tests** - Combined mass lapse + credit spread widening, one-notch downgrade, no reinvestment
- **Regulatory-Quality Excel Output** - Multi-tab workbook with summary, cash flow detail, ALM metrics, audit trail, and company comparison

## Technology Stack

| Component | Technology |
|---|---|
| Language | Python 3.11+ |
| Yield Curves & Bond Pricing | QuantLib-Python |
| Data Manipulation | pandas, NumPy |
| Data Validation | Pydantic v2 |
| Excel Output | openpyxl |
| Audit Trail | SQLite + Parquet + JSON Lines |
| Configuration | YAML |
| Testing | pytest |
| IDE | VS Code + GitHub Copilot |

## Repository Structure

```
BMA-SBA-Model/
├── README.md                    # This file
│
├── Project_Plan/                # ** Project planning documents (current approach) **
│   ├── 01_Project_Charter.md        # Vision, scope, governance, success criteria
│   ├── 02_Architecture_Design.md    # Package structure, data flow, rule traceability
│   ├── 03_Team_And_Hiring_Guide.md  # Team options, role descriptions, hiring priority
│   ├── 04_Timeline_And_Budget.md    # Phased timeline, milestones, cost framework
│   ├── 05_Technical_Specifications.md # BMA scenarios, asset schemas, algorithms
│   ├── 06_Risk_Assessment.md        # Risk register with mitigations
│   └── 07_Technology_Stack.md       # Technology choices and justifications
│
├── BMA_doc/                     # BMA regulatory reference documents
│   ├── BMA_SBA_Consolidated_Guide.md    # Comprehensive guide to BMA SBA rules
│   ├── BMA_SBA_Illustrative_Calculation.md # Worked example (integration test target)
│   ├── BMA_SBA_vs_UK_MA_Comparison.md   # BMA SBA vs UK Matching Adjustment
│   └── Rules&Handbook/                  # Official BMA regulatory PDFs
│
├── NAIC_Doc/                    # Comparative regulatory documentation
│   ├── NAIC_CFT_ComprehensiveGuide.md       # NAIC Cash Flow Testing guide
│   └── BMA_SBA_vs_NAIC_CFT_Comparison.md    # BMA SBA vs NAIC CFT comparison
│
├── Archive/                     # Superseded earlier documents
│   └── (earlier summary documents)
│
└── [Legacy Planning Docs]       # Prior architecture explorations (for reference only)
    ├── Python_Implementation/       # Initial Python-only design docs
    ├── Julia_Implementation/        # Initial Julia-only design docs
    └── Hybrid_Implementation/       # Hybrid Python+Julia analysis
```

> **Note:** The `Python_Implementation/`, `Julia_Implementation/`, and `Hybrid_Implementation/` folders contain earlier architecture explorations that have been superseded by the current approach documented in `Project_Plan/`. They are retained for reference and historical context.

## Architecture Summary

The model follows a **Prophet-inspired architecture** with clean separation of concerns:

| Layer | Description | Location |
|---|---|---|
| **Model Points** | Asset and liability data (CSV/Excel), validated by Pydantic schemas | `model_points/` |
| **Assumptions** | Versioned YAML/CSV files (BMA risk-free curve, credit tables, spreads) | `assumptions/` |
| **Rules** | BMA rules encoded as versioned Python modules with `@rule_ref` traceability | `rules/` |
| **Projection** | Pure Python period-by-period cash flow projection (no QuantLib) | `projection/` |
| **Curves** | QuantLib yield curve construction and bond pricing (with audit wrapper) | `curves/` |
| **Calculations** | BEL, Risk Margin, LCR, stress tests | `calculations/` |
| **Challenge** | Import company submissions and compare against benchmark | `challenge/` |
| **Output** | Regulatory-quality multi-tab Excel workbooks | `output/` |

## Current Status

The project has completed the planning phase. A comprehensive set of project documents is ready for team review in `Project_Plan/`. Key decisions made:

- **Architecture:** Python desktop calculation engine with QuantLib for curves/pricing
- **Scope:** Full BMA SBA regulatory suite (BEL + LCR + stress tests + company challenge)
- **Team:** To be assembled (3-6 people depending on timeline requirements)
- **Timeline:** 5-12 months depending on team size

## Next Steps

1. **Team Review** - Discuss the Project Plan documents with the wider BMA team
2. **Team Assembly** - Hire Senior Python Developer (critical path), then additional developers
3. **IT Setup** - Confirm Python/QuantLib installation on BMA workstations
4. **Phase 0** - Project setup and QuantLib bootcamp with Jupyter notebooks
5. **Phase 1** - Reproduce BMA illustrative calculation (the golden integration test)

## Key Reference Documents

| Document | Purpose |
|---|---|
| [Project Charter](Project_Plan/01_Project_Charter.md) | Start here - project vision and scope |
| [Architecture Design](Project_Plan/02_Architecture_Design.md) | Technical architecture and data flow |
| [BMA SBA Consolidated Guide](BMA_doc/BMA_SBA_Consolidated_Guide.md) | Authoritative BMA rule reference |
| [BMA Illustrative Calculation](BMA_doc/BMA_SBA_Illustrative_Calculation.md) | Worked example - the primary integration test |
