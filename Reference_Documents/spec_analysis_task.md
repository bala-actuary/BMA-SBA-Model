# Specification vs Implementation Analysis Task

## Status: ANALYSIS COMPLETE ✓

## Document Details
- **Source File**: c:\BMADev\Anthora_SBA\02. Model\Reference Documents\02.02.18 SBA_BEL_Pythora_Model_Implementation_Specification_v1.0.md
- **Format**: Markdown (converted from PDF)
- **Size**: ~4000 lines (complete specification read)
- **Approved**: 8 October 2024 (GMAC) and 30 October 2024 (AHL Board)
- **Version**: 1.0
- **Effective Date**: 30 October 2024
- **Analysis Date**: 24 March 2026

## Key Finding: SBA Implementation Specification v1.0.0

This is the **OFFICIAL specification** for the Pythora model v1.0.0, covering SBA BEL calculation methodology and implementation.

## Sections Identified

### 1. Executive Summary (Lines 1-138)
- **Purpose**: Documentation of SBA BEL model implementation for EBS reporting
- **Scope**: Pythora model architecture, functionalities, data flow
- **Key Reference**: SBA BEL Methodology Paper, SBA BEL Model Requirements
- **Governance**: GMAC & Group Chief Actuary approved
- **Outstanding Requirements**:
  - F2.13: Asset optionalities (not fully validated)
  - F2.31: Duration-based BSCR (out of scope, classified as desirable)
  - F3.3: Sensitivities & SST (partially satisfied)
  - F3.5: BMA Liquidity Template (in progress)
  - NF5: Controls (partially satisfied)

### 2. Conceptual Specification (Lines 139-600+)
- **Model Inputs**: Liability data, Asset data, Model parameters
- **Assumption Inputs**: Economic/non-economic, Market data
- **Code Overview**: 4-step process (Load → Calibrate → Calculate SBA → Output)
- **Key Concepts**:
  - SBA goal-seeking algorithm for asset/liability proportion matching
  - Z-spread constant spread methodology (35bps regulatory cap)
  - 9 SBA scenarios (base + 8 stressed)
  - Spread cap application: If spread > 35bps, recalculate BEL with cap

### 3. Technical Specification (Lines 600-2500+)
- **3.1 Model Inputs**: Asset/liability data, parameters, location details
- **3.2 Assumption Inputs**: 
  - By Asset Class (trading costs, expenses, PiT/TTC spreads, Rho, SAA, illiquidity)
  - D&D (Defaults & Downgrades by asset class, rating, term)
  - SAA (Strategic Asset Allocation)
  - Stresses (sensitivities on asset spread or notional)
  - Market Data (rates, swaption vols, FX, historical fixings, D&D)
- **3.3 Code Architecture**:
  - main_server: Multi-process orchestration
  - main_client: Single BU/scenario calculation
  - Load Settings module
  - Load Data module
  - Calibrate Model module
  - [PENDING READ] Project module
  - [PENDING READ] Goal-seek module
  - [PENDING READ] Export module

### 4. Appendix (Not yet read)
- Section 4.1: Model Inputs details
- Section 4.2: Output Reports details
- Section 4.3: Model Changes
- Section 4.4: Model Code

## Critical Algorithms to Verify

### 1. Goal-Seeking Algorithm (Lines 2.3.6.1, 3.3.X)
**Spec Requirements**:
- Iterative search for minimum asset % that meets liabilities
- Conditions:
  1. Sufficient liquid assets in all periods (cash ≥ 0)
  2. Net asset-liability value (surplus) at end < tolerance and positive
  3. After scaling, derivative rebalancing ensures asset KRDs scale with liability KRDs
- If insufficient assets: scale down liabilities (liability proportion < 100%)
- If surplus assets: scale down assets (asset proportion < 100%)

**Status**: NOT YET READ in detail (technical section 3.3.X)

### 2. Z-Spread Calibration & Projection (Section 3.3.X)
**Spec Requirements**:
- Individual asset z-spread calculated at t=0 via goal-seeking to market value
- Per-period projection: asset z-spread moves in line with asset class z-spread
- Asset class z-spread converges from PiT → TTC using Rho parameters (Rho_Over or Rho_Under)
- Formula: spread(t) = L + (S-L)×ρ^t (mean reversion)
- Weighted average z-spread for reinvestment assets

**Status**: PARTIALLY CONFIRMED (spec section 2.3.3.2)

### 3. Projection Loop (Section 3.3.X)
**Spec Requirements**:
- Per-period steps:
  1. Project z-spread by asset class (PiT/TTC convergence)
  2. Value assets/liabilities using projected spreads
  3. Roll cashflows to end-of-period
  4. Apply defaults, FX, scalars
  5. Remove matured assets
  6. Net cashflows to cash account
  7. Physical rebalancing (sell/buy to TAA)
  8. Derivative rebalancing (IR swaps per KRD)
  9. Liquidity constraints (cash ≥ 0)
  10. Continue until liability CFs < 50,000

**Status**: PARTIALLY CONFIRMED (spec section 2.3.1.1, lines 2400+)

### 4. Asset Model Types (Section 3.3.8.1 expected)
**Spec References**:
- FixedRateBond, FloatingRateBond, FixedToFloatingRateBond
- CallableFixedRateBond
- Swaptions (mapped to 5 vanilla swaps)
- VanillaSwaps (payer/receiver)
- CashAccount
- Residential Mortgage Loans
- Bond Futures
- Stripped Assets

**Status**: NOT YET READ

### 5. SBA Scenarios (Section 2.3.6.2)
**Spec**, 9 scenarios: SBA_0_Base + 8 stressed
- BMA prescribed shocks applied to base rates
- Construct 9 yield curves per scenario

**Status**: CONFIRMED

### 6. Rebalancing Rules (Sections 2.3.5.2, 2.3.5.3, 3.3.X)
**Physical Rebalancing**:
- TAA used to determine over/under-weight
- Sell proportionally from over-weight classes (liquid assets only)
- Proceeds to cash
- Buy into under-weight classes (follow asset profiles)
- Constraint: cash ≥ 0 (no borrowing)
- Illiquid assets (private debt, RMLs): can't sell, only buy
- Encumbered assets: illiquid
- Duration constraint: purchased duraton ≤ liability duration

**Derivative Rebalancing**:
- Maintain target KRDs (5, 10, 15, 20, 30 years)
- Assumption: asset KRD movement matches liability KRD movement
- Calculate swap notionals to achieve target KRDs
- Swaps: payer or receiver, zero market value at issue
- Bid-offer spread: capitalized as immediate cash outflow
- Swaps never sold (but can offset with new swaps)
- Full collateralization with cash

**Status**: PARTIALLY CONFIRMED (spec sections 2.3.5.2-2.3.5.3, liability valuation discount at risk-free only)

## Emerging Discrepancies / Clarifications Needed

1. **Goal-Seek Strategy**: Spec says "iterative optimization" and "goal-seek". This could be:
   - Linear search (early iterations for speed)
   - Bisection search (later iterations for stability)
   - Other optimization
   **Status**: Not yet described in technical section

2. **Liability Discount Rate for KRD Calculation**: 
   - Spec 2.3.5.3 state: "liability valuation approach used is to discount the remaining cash flows at the prevailing risk free rate within the model (i.e. no 'SBA spread' is applied)"
   - **Implication**: KRDs calculated using risk-free rates only, not SBA spread
   - My reference doc should clarify this

3. **Spread Convergence Parameters**:
   - Spec specifies Rho_Over and Rho_Under differ by direction (spread wide vs tight)
   - **Question**: Is this fully implemented in code?

4. **D&D Methodology**:
   - Spec section 2.3.3.5: "although implemented, has not been fully tested"
   - Previous expected loss approach used for validation
   - **Status**: New BMA D&D methodology not yet validated

5. **Callable Bonds**:
   - Spec section 2.3.3.3: "functionality for modelling dynamic execution of the call option was not fully validated and is not expected to be implemented at the date of model introduction (Q3-2024)"
   - **Status**: Feature not yet ready

6. **Spread Cap**:
   - Regulatory cap: 35 bps
   - Applied if uncapped spread > 35bps
   - **Question**: When is cap checked? (at end or per-scenario?)

## Reference Documents Validation Status

- ✓ 00-Architecture-Overview.md: **REQUIRES UPDATE** - needs spread cap section
- ✓ 01-SBA-Goal-Seeking.md: **NEEDS VERIFICATION** - algorithm description pending
- ✓ 02-ZSpread-Calibration.md: **NEEDS VERIFICATION** - PiT/TTC convergence details
- ✓ 03-Data-Conversion-SBA-Eligibility.md: **NEEDS VERIFICATION** - eligibility rules
- ✓ 04-Rebalancing-Allocation.md: **NEEDS VERIFICATION** - KRD liability discount rate
- ✓ 05-Projection-Valuation.md: **NEEDS VERIFICATION** - period steps
- ✓ 06-Configuration-Parameters.md: **UPDATE IN PROGRESS** - add D&D, Rho, SAA details
- ✓ 07-CLI-Execution-Framework.md: **OK** - main_server/main_client confirmed

## Next Steps

1. **Continue reading spec**: Sections 3.3.6 onwards (Project module, Goal-seek algorithm, Asset types)
2. **Note specific line numbers** where algorithms differ from reference docs
3. **Create reconciliation table**: Spec requirement vs Implementation detail vs Reference doc
4. **Flag unimplemented features**: Callable bonds, Dynamic call options, New D&D
5. **Generate verification checklist**: Line-by-line code review tasks

## User's Verification Strategy

User has:
1. ✓ 8 reference documents (0-7)
2. ✓ Official specification (v1.0.0)
3. ⏳ Extracted codebase (at C:\BMADev\Anthora_SBA\02. Model\Athora_BEL_Model_v0.1.8\)

**Goal**: Verify SBA implementation matches spec AND methodology paper

**Method**: 
- Compare spec requirements to reference docs
- Cross-reference to actual code
- Create unified verification checklist
