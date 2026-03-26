# SBA BEL Methodology - Executive Summary for Actuary On-boarding

**Audience**: Actuaries, team leads, auditors (technical, math-comfortable)  
**Duration**: 2-3 hours read + 2-3 hours deep-dive (Phase docs)  
**Purpose**: Understand SBA BEL business logic, regulatory context, implementation approach  
**Version**: v1.0 | Created: March 24, 2026  

---

## Page 1: Regulatory Context & Business Purpose

### What is SBA?

The **Scenario-Based Approach (SBA)** is a regulatory requirement from the **Bermuda Monetary Authority (BMA)** under Schedule XXV of the 2024 Amended Rules. It determines the **Best Estimate Liability (BEL)** for insurance portfolios by calculating the minimum asset value required to cover projected liability cash flows across a range of economic stress scenarios.

### Why SBA Matters

**Formula (Conceptually)**:
$$\text{SBA BEL} = \max_{s \in \text{Scenarios}} \left( \text{Assets needed in scenario } s \text{ to cover all liability cashflows} \right)$$

**Translation**: SBA answers "What's the worst-case amount of assets we need to keep our promises to policyholders, across 9 different economic scenarios?"

### Regulatory Requirement

Under **Article 28 of Schedule XXV (BMA 2024 Amendment Rules)**:

| Requirement | Your Implementation |
|-------------|-------------------|
| Use current asset portfolio + reinvestment assumptions | ✅ Assets at t=0, reinvestment rules defined |
| Calculate assets needed for **base** + **8 stress scenarios** | ✅ 9 total scenarios modeled |
| Take **highest asset requirement** across scenarios | ✅ Goal-seek identifies maximum |
| Apply interest rate shocks per Article 28(7) | ✅ 8 IR scenarios: ±25bps, curve shifts, twists, flattens |
| Document methodology per Article 28(38) | ✅ This package (21 docs + executive summary) |

### Governance at Athora

| Role | Responsibility |
|------|-----------------|
| Group Chief Actuary | Approve methodology, attest to appropriateness |
| Group Model & Assumptions Committee | Develop & endorse model |
| Group Actuarial | Implement & maintain model |
| Group Risk | Validate model |
| Board | Final approval |

---

## Pages 2-3: The Algorithm - How SBA Works

### High-Level Process

```
INPUTS:
├─ Assets at time 0 (market values by asset class)
├─ Liabilities (cashflows in base + 8 scenarios, includes policyholder optionality)
├─ Interest rates (Euribor 6M swap curve + 8 stress scenarios)
├─ Spreads (mean-reverting credit spreads)
├─ Parameters (150+ settings: tolerances, limits, assumptions)
└─ Derivatives (IR swaps for hedging, FX forwards, swaptions)

PROCESS (for each of 9 scenarios):
  FOR year = 1 to 100:
    1. Receive liability cashflows for the year
    2. Receive asset cashflows (coupons, maturities, net of D&D)
    3. Pay/receive swap cashflows
    4. Pay investment fees
    5. Roll forward to end-of-year (accrue investment returns)
    6. Update interest rates (base + scenario shock)
    7. Revalue all assets at new rates + spreads
    8. Rebalance portfolio:
       - Physical: Rebalance to Target Asset Allocation (TAA)
       - Derivatives: Rebalance swaps to match KRD targets
       - Incur transaction costs
  AFTER total projection:
    Run GOAL-SEEK: Find minimum assets (at t=0) such that:
      - Projected assets never go negative (no borrowing)
      - All liability cashflows are covered in full

OUTPUT:
├─ Asset amount required = SBA BEL (highest across 9 scenarios)
├─ SBA Spread = Constant spread above risk-free rates that values liabilities
├─ Plus: 35bps cap applied to SBA spread ⚠️ (currently NOT enforced - gap)
└─ Supporting metrics: KRD positions, rebalancing history, stress impacts

```

### The 9 Scenarios

**Base Case** (1 scenario):
- Interest rates follow Euribor 6M swap curve
- Spreads apply mean reversion: $z(t) = L + (S_0 - L) \times \rho^t$
- No shocks applied

**Stress Scenarios** (8 scenarios per BMA Article 28(7)):

| Scenario | Interest Rate Adjustment | Economic Meaning |
|----------|------------------------|-------------------|
| Up 25bps | All rates +25bps | Tightening environment |
| Down 25bps | All rates -25bps | Easing environment |
| Up 25bps flipped | +25bps then -25bps (curve twist) | Short rates up, long rates down |
| Down 25bps flipped | -25bps then +25bps (curve twist) | Short rates down, long rates up |
| Flattening | Front end +50bps, back end -50bps | Yield curve flattens |
| Steepening | Front end -50bps, back end +50bps | Yield curve steepens |
| Short up | +50bps for first 5 years, then +25bps | Front-end stress |
| Long up | -25bps for first 5 years, then +50bps | Back-end stress |

**For each scenario**: Shocks are MAXIMUM in year 1, then converge to steady-state by year 10. Years 11-100 use year 10 shock level.

### Goal-Seek Algorithm: Finding the SBA BEL

**Why needed**: We need to find the EXACT starting asset amount that:
1. ✅ Covers all liability cashflows (no shortfall)
2. ✅ Never goes negative (no borrowing allowed)
3. ✅ Is minimized (most conservative estimate)

**Athora's approach**: **Hybrid Linear + Bisection** algorithm

```
Algorithm: Hybrid_Goal_Seek(target_spread)
  attempts = 0
  while attempts < 20:
    
    // Try linear interpolation first (faster)
    if attempts < 8:
      linear_estimate = interpolate_linearly(attempts)
      run_projection(linear_estimate)
      if converged_or_bracketed():
        return solution
      
    // After 8 attempts (or discontinuity detected), switch to bisection
    else:
      upper = 110% of current estimate
      lower = 90% of current estimate
      while not converged():
        mid = (upper + lower) / 2
        run_projection(mid)
        if cash_positive_everywhere():
          lower = mid
        else:
          upper = mid
      return solution
    
    attempts += 1
  
  // Last resort: absolute tolerance (looser)
  return best_effort_with_absolute_tolerance()
```

**Why hybrid?** 
- Linear is fast when the problem is smooth
- Bisection is robust when there are discontinuities (e.g., rebalancing thresholds)
- Falls back to loose tolerance if 20 attempts insufficient

---

## Pages 4-6: Business Rules & Constraints

### Asset Eligibility: What Can Back Liabilities?

**Principle**: Only **fixed-income** assets with **defined cashflows** can back SBA liabilities.

#### Unrestricted Assets (No Limits)

| Asset Class | Example | Rating Requirement | Approval Status |
|-------------|---------|-------------------|-----------------|
| Cash & equivalents | Bank deposits, MMFs | IG | ✅ No approval needed |
| Government bonds | EUR Sovereigns | Any | ✅ Acceptable per BMA |
| Municipal bonds | Local authority bonds | IG | ✅ Acceptable per BMA |
| Corporate bonds (IG) | Bank bonds, corporates | Investment Grade | ✅ Acceptable per BMA |
| EMD (IG) | Emerging market sovereigns | Investment Grade | ✅ Acceptable per BMA |

**Total from these**: No limit. Use as much as available.

#### Limited Assets (Subject to Caps)

| Asset Class | Limit | Details | Approval Status |
|-------------|-------|---------|-----------------|
| Private debt, sub-IG | 10% of SBA assets | Internally rated MML, Large Cap DL, Infra, ABL | ✅ Approved YE23 |
| Dutch RMLs, High LTV (>80%) | 10% of SBA assets | Retail mortgages <90 days arrears | ✅ Approved |
| Dutch RMLs, Low LTV (≤80%) | 20% of SBA assets (incremental) | Quality mortgages | ✅ Approved |
| CLOs (IG) | 10% cap (general limited bucket) | Collateralized loan obligations | 🟡 Planned YE24 |
| ABS (IG) | 10% cap (general limited bucket) | Asset-backed securities | 🟡 Planned YE24 |
| Commercial mortgage loans (IG) | Per sub-IG bucket | NAIC-rated mortgages | 🟡 Planned YE24 |

**Combined limit for limited assets**:
```
Sub-IG assets + High LTV RMLs ≤ 10% of total SBA assets
Sub-IG assets + High LTV RMLs + Low LTV RMLs ≤ 20% of total SBA assets

If exceeded: Scale down pro-rata until compliant
```

#### Individual Asset Limit

**Each single asset** (by ISIN or contract): ≤ 0.5% of total SBA asset portfolio

**Ineligible Assets** (cannot use):
- ❌ Equities
- ❌ Real estate (non-mortgage)
- ❌ Unit-linked assets
- ❌ Structured products (unless approved ABS)
- ❌ Private equity

---

### Liability Eligibility: What Needs SBA Valuation?

#### IN SCOPE (Valued via SBA)

| Liability Type | Characteristics | Coverage |
|---|---|---|
| **ANL Traditional** | Fixed-rate annuities, with-profit | Belgium & Netherlands |
| **AB Traditional** | Fixed-rate liabilities, Belgian book | Belgium entity |
| **ARE Business** | Reinsurance liabilities | All treaties |

**Key feature**: Includes **policyholder optionality** (surrender rights, profit sharing) modeled through scenario-dependent cashflows

#### OUT OF SCOPE (Not in SBA)

| Type | Reason |
|-----|--------|
| Unit-linked | Policyholder bears investment risk; BEL not sensitive to asset choices |
| Participating (DE/IT) | Too variable due to smoothing/profit participation |
| Savings mortgages | Ineligible backing assets |
| Separate accounts | Managed separately outside SBA scope |

---

### Key Actuarial Mechanics

#### 1. Interest Rate Modeling

**Base Case**:
- Use **Euribor 6M swap curve** (market standard, risk-free proxy for EUR)
- Interpolate illiquid tenors linearly between liquid points
- Liquid tenors: 1-15Y (yearly), then 20Y, 25Y, ..., 50Y (5-year), then flat beyond 50Y
- **No extrapolation beyond 50Y** (hold rate flat to 100Y)

**Rationale**: 
- Swap rates capture current risk-free yields
- Flat beyond 50Y is conservative (assumes no further changes)
- Consistent with spread measurement (spreads also relative to swaps)

#### 2. Spread Modeling

**Formula** (per Athora methodology):
$$z(t) = L + (z_0 - L) \times \rho^t$$

Where:
- $z(t)$ = Spread in year $t$
- $L$ = Long-term ("convergence") spread
- $z_0$ = Starting spread (calibrated at t=0)
- $\rho$ = Mean reversion rate (0 < ρ < 1)

**Interpretation**: Spreads stay high initially then revert to long-term average

**At Athora**: 
- Calibrated to market prices at t=0
- Three methods available (configurable):
  1. **CalibrateToMarketPrice** (goal-seek z to match observed prices)
  2. **UseRateInAssetFile** (read from asset data file)
  3. **UseRateInThisTable** (use preset table)
- Default: CalibrateToMarketPrice

#### 3. Defaults & Downgrades (D&D)

**Formula** (cumulative impact):
$$\text{Cumulative Default Rate}(t) = 1 - (1 - r_{\text{annual}})^t$$

**Applied as**: Multiplicative scalar to both **market value** and **cashflows** each period

**Example**:
- Annual default rate: 1%
- Year 1: Cumulative = 1-(1-0.01)¹ = 1% → 99% of assets remain
- Year 5: Cumulative = 1-(1-0.01)⁵ = 4.9% → 95.1% of assets remain
- Year 10: Cumulative = 1-(1-0.01)¹⁰ = 9.6% → 90.4% of assets remain

**Usage**: Reflects expected credit losses in projection

#### 4. Rebalancing: Two-Layer approach

**Layer 1: Physical Rebalancing** (3x per year max)
- Target: Return to **Target Asset Allocation** (TAA)
- Example TAA: 60% bonds, 25% private lending, 15% mortgages
- Sell overweight, buy underweight
- Incur transaction costs (2-5 bps typical)
- Purpose: Maintain strategic portfolio positioning

**Layer 2: Derivative Rebalancing** (swaps)
- Target: **Key Rate Duration (KRD) match**: 
  $$\text{Asset KRD} - \text{Liability KRD} = 0$$
  (i.e., no net interest rate sensitivity)
- Use IR swaps to adjust duration
- Example: If assets shorter than liabilities, enter receive-fixed (short-rate) swaps
- Rebalance up to 3 times per period

**Why two layers?**
- Physical rebalancing changes the actual assets (longer-term strategic)
- Swap rebalancing adjusts hedges (period-by-period tactical)

---

## Pages 7-8: Key Assumptions & Configuration Parameters

### Critical Parameters (Define Model Behavior)

| Parameter | Value | Meaning | Reference |
|-----------|-------|---------|-----------|
| **goal_seek_tolerance_optimal** | 0.001% | Relative tolerance for goal-seek convergence | alm_setup.xlsm |
| **goal_seek_tolerance_absolute** | 0.1% | Absolute fallback tolerance | alm_setup.xlsm |
| **max_goal_seek_attempts** | 20 | Maximum linear+bisection attempts | sba_goal_seeker.py |
| **liability_krd_method** | Risk_Free | Method for KRD: Risk_Free (✅) or Risk_Free+SBA_Spread (❌) | liability_cashflow.py:92-97 |
| **sba_spread_cap** | 35 bps | Regulatory cap on SBA spread (🔴 NOT ENFORCED) | settings_loader.py:76 |
| **projection_years** | 100 | Model runs 100-year projection | balance_sheet_projection.py |
| **projection_frequency** | Annual | Year-by-year steps (not monthly) | balance_sheet_projection.py |
| **rebalance_iterations_max** | 3 | Max swaps + physical per period | balance_sheet_projection.py:160-195 |
| **investment_expense_bps** | Dynamic | Calculated from asset mix (% of AUM) | config settings |

### Input Data Requirements

| Data | Format | Source | Frequency |
|------|--------|--------|-----------|
| Asset holdings (t=0) | CSV + Excel | Bloomberg/custodian feeds | Daily |
| Asset characteristics | ISIN, rating, cashflows, spreads | UBS Delta, Bloomberg | Daily |
| Liability cashflows | By scenario (base + 8) | BU systems (BE, NL, ARE) | Quarterly |
| Yield curves | Euribor 6M swap rates by tenor | Bloomberg, published daily | Daily |
| Credit spreads | By asset class, rating | Market data feeds | Daily |

### Output Delivery

| Output | Purpose | Users |
|--------|---------|-------|
| SBA BEL amount | Main regulatory metric | CFO, regulators, board |
| SBA Spread | Discounting rate for liabilities | Risk team, actuaries |
| Scenario results | Sensitivity analysis | Risk analytics, board |
| KRD positions | Hedging effectiveness | ALM team |
| Rebalancing history | execution tracking | Operations |

---

## Page 9: Known Issues, Gaps & Risk Mitigation

### 🔴 Critical Gap: Spread Cap NOT Enforced

**Issue**: 
- **Specification requirement** (BMA Article 28, Athora Methodology Section 15.2): SBA spread must not exceed 35bps
- **Current status**: Parameter `sba_spread_cap` defined at line 76 of `settings_loader.py` but zero references in code
- **Impact**: If market spreads exceed 35bps, the 35bps cap is silently ignored → BEL calculation non-compliant

**Evidence**:
- File: settings_loader.py, line 76
- Grep search: "spread_cap" appears 1 time (definition) | 0 times (enforcement logic)
- File: liability_cashflow.py, lines 156-210 (where spread is applied to liability valuation)
- **No cap logic present**

**Severity**: 🔴 **HIGH** (Regulatory non-compliance)

**Mitigation Options**:
1. ✅ **Implement the cap** (v0.2.0 target)
   - Add enforcement logic after spread calculation
   - Apply: `sba_spread_final = min(sba_spread_calculated, 0.0035)`
   - Effort: 4-8 hours dev + 2 hours testing
   
2. ✅ **Document corporate exception** (if cap not enforceable)
   - Get BMA written approval
   - File in regulatory dossier
   - Note: Higher priority if spreads historically > 35bps
   
3. ✅ **Monitor and test** (current approach)
   - Phase 3 edge case E4.5 tests spread cap behavior
   - Manual verification each reporting date
   - Document in Phase 3 edge case test results

**Action**: Prioritize based on spread environment; coordinate with CFO + compliance

---

### ⚠️ Medium Priority: Callable Bond Validation Incomplete

**Issue**:
- Specification Section 3.3.8.4 marks callable bond handling as "validation is ongoing"
- Implementation exists (wave-off logic) but not fully stress-tested in extreme scenarios
- Option pricing may be incorrect under extreme IR moves

**Expected behavior**: 
- When interest rates fall dramatically, bondholder exercises call option
- Bond value capped at strike (issuer calls it back)
- Currently modeled but not production-validated

**Mitigation**: 
- Test case E3.3 (PHASE3-004) covers callable bond behavior
- Post-deployment monitoring (Q1 2024) for real portfolio impacts
- If issues found: Escalate to dev team for refinement

**Status**: Not blocking; monitor in validation phase

---

### ⚠️ Medium Priority: D&D Stress Testing Limited

**Issue**:
- D&D formula uses annual default rates only
- No intra-year defaults modeled (conservative but limited)
- Extreme annual rates (>50%) not extensively tested

**Mitigation**:
- Edge cases E5.1-E5.3 (PHASE3-004) test annual rates from 0% to extreme
- Expected behavior documented with caveats
- Use in production with annual default rates only

**Status**: Known limitation; edge cases define expected behavior

---

## Page 10: How to Use This Document + Learning Path

### "I'm new, what should I read first?"

```
Week 1 (Days 1-4): Regulatory Foundation
├─ This document, Pages 1-3 → Understand BMA requirement + algorithm
├─ This document, Pages 4-6 → Learn business rules
└─ PHASE1-001-Architecture → See how it's implemented at Athora

Week 2 (Days 5-10): Deep Implementation
├─ IMPLEMENTATION-SPECIFICATION (see companion doc)
├─ Code review: liability_cashflow.py, sba_goal_seeker.py
└─ Configuration: alm_setup.xlsm settings

Week 3 (Days 11-15): Validation & Testing
├─ PHASE3-001 → Test architecture
├─ PHASE3-007 → Execute 54 test cases
└─ PHASE3-006 → Understand known gaps
```

### Quick Finding Guide

| Question | Answer Location |
|----------|-----------------|
| **"Why does BMA require SBA?"** | Page 1 |
| **"How does goal-seek find the BEL?"** | Pages 2-3 |
| **"What assets can we use?"** | Pages 4-5 |
| **"What's the D&D formula?"** | Page 6 |
| **"What are the critical parameters?"** | Pages 7-8 |
| **"What could go wrong?"** | Page 9 |
| **"How do I code this?"** | IMPLEMENTATION-SPECIFICATION document |
| **"How do I test this?"** | PHASE3-INDEX.md (then PHASE3-007) |
| **"Where's the detailed architecture?"** | PHASE1-001 to PHASE1-010 |
| **"What's been verified?"** | PHASE2-004 (verification findings) |

### Recommended Reading Order (By Role)

**Actuaries** (like you):
1. Pages 1-9 of this document (executive summary)
2. [PHASE1-001-Architecture](PHASE1-001-Architecture-Overview.md)
3. [PHASE1-002-SBA-Goal-Seeking](PHASE1-002-SBA-Goal-Seeking.md)
4. [PHASE1-005-Rebalancing](PHASE1-005-Rebalancing-Allocation.md)
5. IMPLEMENTATION-SPECIFICATION (companion doc)

**Developers**:
1. Pages 2-3 of this document (algorithm overview)
2. IMPLEMENTATION-SPECIFICATION (companion document)
3. [PHASE1-001-Architecture](PHASE1-001-Architecture-Overview.md)
4. Code review: sba_goal_seeker.py, liability_cashflow.py
5. [PHASE3-007-Test-Execution](PHASE3-007-Test-Execution-Guide.md)

**Risk/Compliance**:
1. Page 1 (regulatory context)
2. Pages 4-9 (business rules + gaps)
3. [PHASE3-006-Known-Gaps](PHASE3-006-Known-Gaps-Limitations.md)
4. [PHASE2-004-Verification](PHASE2-004-Spec-vs-Implementation-Report.md)

**Auditors/Regulators**:
1. Entire document (Pages 1-10)
2. [PHASE2-005-Verification](PHASE2-005-Spec-vs-Implementation-Report.md)
3. All Phase 1 docs (if comprehensive review needed)

---

## Reference Matrix: This Document ↔ Complete Documentation Set

| Topic | This Doc | Phase 1 | Phase 2 | Phase 3 |
|-------|----------|---------|---------|---------|
| Regulatory context | ✅ Pages 1 | [001](PHASE1-001-Architecture-Overview.md) | [002](PHASE2-002-Checklist-Final.md) | N/A |
| Algorithm overview | ✅ Pages 2-3 | [001-002](PHASE1-002-SBA-Goal-Seeking.md) | [001](PHASE2-001-Code-Verification-Findings.md) | [001](PHASE3-001-Test-Plan.md) |
| Asset eligibility | ✅ Pages 4-5 | [referenced in 001](PHASE1-001-Architecture-Overview.md) | [001](PHASE2-001-Code-Verification-Findings.md) | N/A |
| Rebalancing | ✅ Page 6 | [005](PHASE1-005-Rebalancing-Allocation.md) | [004](PHASE2-004-Spec-vs-Implementation-Report.md) | [002](PHASE3-002-Component-Unit-Tests.md) |
| D&D formula | ✅ Page 6 | [009](PHASE1-009-Defaults-Downgrades.md) | [001](PHASE2-001-Code-Verification-Findings.md) | [004](PHASE3-004-Edge-Case-Stress-Tests.md) |
| Parameters | ✅ Pages 7-8 | [007](PHASE1-007-Configuration-Parameters.md) | N/A | N/A |
| Known gaps | ✅ Page 9 | [010](PHASE1-010-Spread-Cap-and-Outputs.md) | [004](PHASE2-004-Spec-vs-Implementation-Report.md) | [006](PHASE3-006-Known-Gaps-Limitations.md) |
| Testing | N/A | N/A | N/A | [001-007](PHASE3-001-Test-Plan.md) |

---

## Summary

**SBA is fundamentally**: 
- Risk measurement framework (BMA requirement) ✅
- Scenario analysis across 9 economic cases ✅  
- Deterministic 100-year projection model ✅
- Goal-seek to find minimum backing assets ✅
- Result: SBA BEL = highest asset requirement + SBA spread cap at 35bps ✅

**Your immediate next steps**:
1. Read Pages 1-3 (regulatory + algorithm) today
2. Read Pages 4-9 (business rules + gaps) tomorrow
3. Move to IMPLEMENTATION-SPECIFICATION document (companion)
4. Then dive into PHASE1 architecture docs

**Key thing to remember**: 
- 🟢 5 of 6 components verified as spec-compliant
- 🔴 1 critical gap: Spread cap NOT enforced (need to fix or document)
- ⚠️ 2 medium items: Callable bonds, D&D stress testing

---

**Document**: SBA BEL Executive Summary for Actuary On-boarding  
**Status**: Ready for use  
**Next**: Read IMPLEMENTATION-SPECIFICATION document (developer-focused technical specs)
