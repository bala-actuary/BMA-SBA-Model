# PHASE 1: Reference Architecture Index

**Phase Overview**: Understanding the SBA model design, components, and mechanics  
**Duration**: Foundation knowledge base  
**Documents**: 10 comprehensive architecture references  
**Audience**: Architects, developers, technical leads  
**Prerequisites**: Basic understanding of fixed income, SBA modeling

---

## Document Map

### Core Architecture (Documents 1-3)
These form the foundation for understanding how all components fit together.

**[PHASE1-001-Architecture-Overview](PHASE1-001-Architecture-Overview.md)**
- System-wide architecture diagram
- Component interdependencies
- Data flow from inputs to outputs
- Key modules and their responsibilities
- **Read when**: You need to understand the big picture
- **Time**: 30 minutes

**[PHASE1-002-SBA-Goal-Seeking](PHASE1-002-SBA-Goal-Seeking.md)**
- Goal-seeking algorithm detailed breakdown
- Hybrid linear interpolation + bisection approach
- Discontinuity detection mechanism
- Convergence cascade logic
- Source: sba_goal_seeker.py (lines 200-450)
- **Read when**: Understanding rebalancing targets
- **Time**: 40 minutes

**[PHASE1-003-ZSpread-Calibration](PHASE1-003-ZSpread-Calibration.md)**
- Z-spread definition and role in valuation
- Three calibration methods explained
- Mean reversion formula: z(t) = L + (S-L)×ρ^t
- Per-period projection mechanism
- Source: zspread_provider.py (full file)
- **Read when**: Understanding spread dynamics
- **Time**: 45 minutes

### Data & Business Logic (Documents 4-6)
Understanding how data flows and calculations are performed.

**[PHASE1-004-Data-Conversion-SBA-Eligibility](PHASE1-004-Data-Conversion-SBA-Eligibility.md)**
- CSV input ingestion from raw data
- Data standardization process
- SBA eligibility filtering logic
- Output standardized data format
- **Read when**: Working with input data
- **Time**: 20 minutes

**[PHASE1-005-Rebalancing-Allocation](PHASE1-005-Rebalancing-Allocation.md)**
- Swap rebalancing mechanics
- Physical rebalancing process
- DV01 calculations and matching
- Up to 3 iterations per period
- Source: swap_rebalancer.py, balance_sheet_projection.py (lines 160-195)
- **Read when**: Understanding portfolio management
- **Time**: 35 minutes

**[PHASE1-006-Projection-Valuation](PHASE1-006-Projection-Valuation.md)**
- Period-by-period projection engine
- Valuation timing and order
- Interest rate scenarios
- QuantLib integration for NPV calculations
- Source: balance_sheet_projection.py (100-200 lines)
- **Read when**: Working on projection logic
- **Time**: 40 minutes

### Configuration & Operations (Documents 7-10)
Understanding how the system is configured and operated.

**[PHASE1-007-Configuration-Parameters](PHASE1-007-Configuration-Parameters.md)**
- Settings dataclass structure (150+ parameters)
- Excel-driven configuration (alm_setup.xlsm)
- Parameter categories and dependencies
- Configuration loading mechanism
- Source: settings_loader.py, setting_enums.py
- **Read when**: Setting up new scenarios
- **Time**: 35 minutes

**[PHASE1-008-CLI-Execution-Framework](PHASE1-008-CLI-Execution-Framework.md)**
- Command-line interface options
- Dryrun mode for fast testing
- Batch processing capabilities
- Execution modes and parameters
- **Read when**: Running model projections
- **Time**: 25 minutes

**[PHASE1-009-Defaults-Downgrades](PHASE1-009-Defaults-Downgrades.md)**
- D&D (Defaults & Downgrades) formula: 1-(1-X)^t
- Application to market values AND cashflows
- Scalar multiplication mechanism
- Bounds validation and assertions
- Source: asset_projector.py (lines 161-190)
- **Read when**: Understanding credit risk application
- **Time**: 30 minutes

**[PHASE1-010-Spread-Cap-and-Outputs](PHASE1-010-Spread-Cap-and-Outputs.md)**
- SBA spread cap (35bps regulatory requirement)
- **⚠️ CRITICAL**: Parameter defined but NOT enforced (gap found)
- Output file generation
- Regulatory reporting formats
- Source: settings_loader.py (line 76)
- **Read when**: Understanding output compliance
- **Time**: 30 minutes

---

## Learning Paths

### Path 1: Complete System Understanding (3-4 hours)
For architects and technical leads:
1. [PHASE1-001](PHASE1-001-Architecture-Overview.md) - Overview (30 min)
2. [PHASE1-002](PHASE1-002-SBA-Goal-Seeking.md) - Goal-seek (40 min)
3. [PHASE1-003](PHASE1-003-ZSpread-Calibration.md) - Z-spread (45 min)
4. [PHASE1-005](PHASE1-005-Rebalancing-Allocation.md) - Rebalancing (35 min)
5. [PHASE1-006](PHASE1-006-Projection-Valuation.md) - Projection (40 min)
6. [PHASE1-007](PHASE1-007-Configuration-Parameters.md) - Configuration (35 min)

### Path 2: Data Pipeline (1.5 hours)
For data engineers:
1. [PHASE1-004](PHASE1-004-Data-Conversion-SBA-Eligibility.md) - Data conversion (20 min)
2. [PHASE1-007](PHASE1-007-Configuration-Parameters.md) - Configuration (35 min)
3. [PHASE1-008](PHASE1-008-CLI-Execution-Framework.md) - Execution (25 min)
4. [PHASE1-010](PHASE1-010-Spread-Cap-and-Outputs.md) - Outputs (30 min)

### Path 3: Risk/Compliance (1.5 hours)
For compliance and risk teams:
1. [PHASE1-001](PHASE1-001-Architecture-Overview.md) - Overview (30 min)
2. [PHASE1-009](PHASE1-009-Defaults-Downgrades.md) - D&D (30 min)
3. [PHASE1-010](PHASE1-010-Spread-Cap-and-Outputs.md) - Spread cap ⚠️ (30 min)
4. [PHASE1-003](PHASE1-003-ZSpread-Calibration.md) - Z-spread (45 min)

---

## Key Concepts Explained

| Concept | Document | Key Points |
|---------|----------|-----------|
| **SBA Goal-Seeking** | [PHASE1-002](PHASE1-002-SBA-Goal-Seeking.md) | Finds optimal asset/liability proportion; hybrid linear/bisection |
| **Rebalancing** | [PHASE1-005](PHASE1-005-Rebalancing-Allocation.md) | Up to 3 iterations/period; swap first, then physical |
| **KRD Matching** | [PHASE1-005](PHASE1-005-Rebalancing-Allocation.md) | Targets: Asset KRD - Liability KRD = 0 (risk-free basis) |
| **D&D Formula** | [PHASE1-009](PHASE1-009-Defaults-Downgrades.md) | cumulative = 1-(1-annual_rate)^t; applied as scalar |
| **Z-Spread** | [PHASE1-003](PHASE1-003-ZSpread-Calibration.md) | Calibrated at t=0; projected with mean reversion |
| **Projection Engine** | [PHASE1-006](PHASE1-006-Projection-Valuation.md) | 101 years annual; iterative rebalancing each period |
| **Configuration** | [PHASE1-007](PHASE1-007-Configuration-Parameters.md) | 150+ parameters; Excel-driven; Settings dataclass |

---

## Critical Files Referenced

| File | Purpose | Key Lines |
|------|---------|-----------|
| liability_cashflow.py | KRD calculation | 92-97 |
| sba_goal_seeker.py | Goal-seeking algorithm | 200-450 |
| balance_sheet_projection.py | Projection & rebalancing | 160-195 |
| swap_rebalancer.py | DV01-based swaps | 240+ |
| asset_projector.py | D&D application | 161-190 |
| zspread_provider.py | Z-spread logic | Full file |
| settings_loader.py | Configuration loading | 76, 132+ |
| setting_enums.py | Configuration enums | 25-35 |
| alm_setup.xlsm | Master configuration file | All tabs |

---

## Important Findings from Phase 1

✅ **Verified**: KRD calculated using Risk_Free method (correct per spec)  
✅ **Verified**: Goal-seeking algorithm is hybrid linear/bisection (matches spec)  
✅ **Verified**: D&D formula is exact match: 1-(1-X)^t  
✅ **Verified**: Rebalancing runs up to 3 iterations per period  
✅ **Verified**: Z-spread mean reversion formula implemented  

🔴 **Gap Found**: Spread cap parameter defined (line 76) but NOT used anywhere  
⚠️ **Note**: Callable bond validation marked "ongoing" in specification  

---

## Next Steps

**After completing Phase 1**:
- Move to [PHASE2-INDEX](../PHASE2-INDEX.md) for code verification
- Move to [PHASE3-INDEX](../PHASE3-INDEX.md) for testing
- Or jump to [00-PROJECT-MASTER-INDEX](../00-PROJECT-MASTER-INDEX.md) for role-based navigation

**Recommended Sequence**:
1. Tech team: Complete Phase 1 path
2. All team: Review Phase 2 findings (especially spread cap gap)
3. QA team: Execute Phase 3 test suite

---

**Index Created**: March 24, 2026  
**Phase Status**: ✅ Complete & Verified  
**Documents**: 10 architecture references  
**Verified Components**: 5 of 6 (83%)
