# SBA BEL Model - Implementation Specification for Developers

**Audience**: Developers, QA engineers, technical architects  
**Duration**: 3-4 hours comprehension + 8-10 hours code review  
**Purpose**: Understand algorithm implementation, data flows, edge cases, testing requirements  
**Version**: v1.0 | Created: March 24, 2026  
**Status**: Active development + validation phase  

---

## Page 1: High-Level System Architecture

### Component Stack

```
┌─────────────────────────────────────────────────────────────┐
│                     INPUT LAYER                              │
│  Assets (CSV)  │  Liabilities (cashflows)  │  Config (Excel) │
└────────────────┬────────────────────────────┬────────────────┘
                 │                            │
┌────────────────▼─────────────────────────┬──▼────────────────┐
│         DATA LOADING & VALIDATION         │  CONFIGURATION    │
│  • CSV parsing                            │  LOADER           │
│  • Cashflow aggregation                   │  • Settings class │
│  • Asset classification                   │  • Parameter      │
│  • Eligibility checking (150 units)       │    validation     │
└────────────────┬─────────────────────────┴──┬────────────────┘
                 │                            │
┌────────────────▼──────────────────────────────▼───────────────┐
│            PROJECTION ENGINE (Core Model)                     │
│                                                               │
│  FOR scenario = 0 to 8:  // Base + 8 IR scenarios           │
│    FOR year = 1 to 100:  // Annual projection               │
│      ├─ Calculate liability cashflows (scenario-dependent)   │
│      ├─ Project asset cashflows (coupons, maturities)       │
│      ├─ Apply defaults & downgrades (D&D scalar)            │
│      ├─ Receive/pay derivative flows (swaps, FX, etc)       │
│      ├─ Accrue investment return                            │
│      ├─ Update IR curves (base + scenario shock)            │
│      ├─ Update spreads (mean reversion)                     │
│      ├─ Revalue assets at new rates                         │
│      └─ Rebalance (TAA + KRD matching)                      │
│                                                               │
│  After projection completion:                                │
│    └─ Record: Final asset value in scenario + cash balance  │
└────────────────┬──────────────────────────────────────────────┘
                 │
┌────────────────▼──────────────────────────────────────────────┐
│          GOAL-SEEK ENGINE (Find Minimum Assets)              │
│                                                               │
│  Input: Projection cashflow series (9 scenarios)             │
│  Task: Find minimum t=0 assets such that:                    │
│    • Assets never go negative (no borrowing)                 │
│    • All liabilities covered                                 │
│  Algorithm: Hybrid linear + bisection                        │
│  Output: Asset amount = SBA BEL                              │
└────────────────┬──────────────────────────────────────────────┘
                 │
┌────────────────▼──────────────────────────────────────────────┐
│          SPREAD CALCULATION & REPORTING                       │
│                                                               │
│  Calculate: SBA spread (inverse NPV)                         │
│  Apply: 35bps cap (currently NOT implemented 🔴)             │
│  Output: SBA BEL + SBA spread + metrics                      │
└──────────────────────────────────────────────────────────────┘
```

### Key Classes/Modules

| Module | File | Responsibility |
|--------|------|-----------------|
| **Settings** | settings_loader.py | Load 150+ parameters, validate ranges |
| **Asset** | asset_projector.py | Hold asset data, apply D&D, project cashflows |
| **Liability** | liability_cashflow.py | Calculate liability NPV, KRD, interest rate sensitivity |
| **Interest Rates** | zspread_provider.py | Manage yield curves, apply spreads, mean reversion |
| **Swaps** | swap_rebalancer.py | DV01 calculations, rebalancing logic, target KRD matching |
| **Projection** | balance_sheet_projection.py | Main loop: iterate years/scenarios, rebalance, accrue returns |
| **Goal-Seek** | sba_goal_seeker.py | Hybrid linear/bisection algorithm to find minimum assets |

---

## Pages 2-4: Core Algorithm Specifications

### Algorithm 1: Goal-Seek (Find Minimum Assets)

**Purpose**: Find minimum asset amount (at t=0) such that liability cashflows are covered across all years and scenarios

**Input**:
- $\text{Scenarios} = \{base, stress_1, ..., stress_8\}$ (9 total)
- $\text{Years} = \{1, 2, ..., 100\}$ 
- For each (scenario, year): **liability_cf[scenario][year]** (known)

**Output**:
- $A_0$ = Required t=0 assets
- $\text{biting_scenario}$ = Which scenario required maximum assets

**Algorithm**:

```python
def goal_seek(scenarios, years, liability_cf, settings):
    """
    Hybrid linear + bisection goal-seek
    """
    attempts = 0
    linear_estimates = []
    
    while attempts < settings.max_attempts:  # Default: 20
        
        # Phase 1: Try LINEAR INTERPOLATION (attempts 0-7)
        if attempts < 8:
            # Estimate based on prior attempts
            if attempts == 0:
                # Initial guess: total liabilities / projection period
                estimated_assets = sum(liability_cf[base]) / 100
            else:
                # Interpolate between two prior attempts
                estimated_assets = linear_interpolate(linear_estimates[-2:])
            
            # Run projection with this estimate
            projection = run_projection(scenarios, estimated_assets)
            
            # Check convergence
            min_cash = min(projection.cash_balance_all_periods_all_scenarios)
            
            if abs(min_cash) < settings.tolerance_optimal:
                # Converged!
                return estimated_assets, scenarios[projection.max_scenario]
            
            # Check for discontinuity (sign change in bracket)
            linear_estimates.append((estimated_assets, min_cash))
            if len(linear_estimates) >= 2:
                prev_assets, prev_cash = linear_estimates[-2]
                if prev_cash < 0 < min_cash:
                    # Bracket found! Switch to bisection
                    attempts = 8  # Force bisection
                    continue
        
        # Phase 2: BISECTION (attempts 8+)
        if attempts >= 8:
            # Setup: find bracket [lower, upper]
            if attempts == 8:
                # Create initial bracket from linear attempts
                lower = min(est[0] for est in linear_estimates) * 0.9
                upper = max(est[0] for est in linear_estimates) * 1.1
            
            # Standard bisection loop
            for bisection_step in range(settings.bisection_max_steps):
                mid = (lower + upper) / 2
                projection = run_projection(scenarios, mid)
                min_cash = min(projection.cash_balance_all_periods_all_scenarios)
                
                if abs(min_cash) < settings.tolerance_optimal:
                    return mid, scenarios[projection.max_scenario]
                
                if min_cash < 0:
                    lower = mid  # Need more assets
                else:
                    upper = mid  # Can use fewer assets
                
                # Check bisection convergence
                if (upper - lower) / mid < settings.tolerance_optimal:
                    return mid, scenarios[projection.max_scenario]
        
        attempts += 1
    
    # Phase 3: FALLBACK (if 20 attempts exceeded)
    # Use loose tolerance
    final_estimate = estimate_with_absolute_tolerance(settings.tolerance_absolute)
    return final_estimate, biting_scenario
```

**Key details**:
- **Discontinuity detection**: Linear method detects sign change → triggers bisection
- **Tolerance cascade**: Optimal (0.001%) → Absolute (0.1%) → Hard stop
- **Max attempts**: 20 (hybrid attempts, then bisection)
- **Convergence criterion**: `abs(min_cash_balance) < tolerance`

**Related code**:
- File: `sba_goal_seeker.py` lines 200-450
- Test: [PHASE3-002](PHASE3-002-Component-Unit-Tests.md) tests U6-U10

---

### Algorithm 2: Annual Projection Within Each Scenario

**Purpose**: Project cashflows, values, and rebalancing for one scenario over 100 years

**Pseudo-code**:

```python
def project_scenario(scenario_id, initial_assets, settings):
    """
    Project one scenario (base or stress) for 100 years
    Returns: final_asset_value, end_of_year_cash_balances
    """
    
    # Initialize
    assets = {spread: initial_assets}  # Asset holdings by instrument
    cash = 0
    ir_curve = settings.get_ir_curve(scenario_id, year=0)  # Euribor 6M
    spread = settings.get_spread(scenario_id, year=0)  # At t=0
    
    for year in range(1, 101):
        
        # Step 1: Receive liability cashflows (and policy expenses)
        cf_liability = scenarios[scenario_id].liability_cf[year]
        cf_expense = scenarios[scenario_id].expense_cf[year]
        cash += cf_liability + cf_expense
        
        # Step 2: Receive asset cashflows (coupons, prepayments, maturities)
        for asset in assets:
            cf_asset = asset.get_coupon_and_maturity(year)
            cash += cf_asset
        
        # Step 3: Apply defaults and downgrades
        d_and_d_rate = settings.get_d_and_d_rate(asset.rating, year)
        cumulative_dd = 1 - (1 - d_and_d_rate) ** year
        
        # Apply to market values AND cashflows
        for asset in assets:
            asset.market_value *= (1 - cumulative_dd)
            cf_received *= (1 - cumulative_dd)
        
        # Step 4: Pay/receive swap cashflows
        for swap in swaps:
            cf_swap = swap.get_cf(year)
            cash += cf_swap
        
        # Step 5: Pay investment management fees
        total_aum = sum(asset.market_value for asset in assets)
        fee_bps = settings.investment_expense_bps
        cash -= total_aum * fee_bps / 10000
        
        # Step 6: Accrue investment return (roll cash forward)
        # Assume cash earns risk-free rate
        rate_for_cash = ir_curve[year] + spread
        cash *= (1 + rate_for_cash)
        
        # Step 7: End of year
        # Update IR curve for next year (apply scenario shock after year 1)
        ir_curve = settings.get_ir_curve_shocked(scenario_id, year + 1)
        
        # Update spreads (mean reversion)
        spread = settings.get_spread_mean_reverted(scenario_id, year + 1)
        
        # Step 8: Revalue all assets
        for asset in assets:
            npv = asset.calculate_npv(ir_curve, spread)
            asset.market_value = npv
        
        # Step 9: REBALANCING (up to 3 iterations)
        for rebalance_iteration in range(3):
            
            # 9a: Swap rebalancing (KRD matching)
            #     Target: Asset KRD - Liability KRD = 0
            asset_krd = calculate_asset_krd(assets, ir_curve, spread)
            liability_krd = calculate_liability_krd(liabilities, ir_curve, method='Risk_Free')
            target_krd = asset_krd - liability_krd
            
            if abs(target_krd) > settings.krd_tolerance:
                # Execute swaps to hedge
                swaps_executed = rebalance_swaps_for_krd(target_krd, ir_curve)
                if not swaps_executed:
                    break  # Convergence achieved
            
            # 9b: Physical rebalancing (TAA)
            #     Target: Return to Target Asset Allocation
            current_allocation = {asset: asset.market_value / total_aum for asset in assets}
            target_allocation = settings.get_taa()
            
            rebalance_needed = max(abs(current_allocation[a] - target_allocation[a]) 
                                   for a in assets)
            
            if rebalance_needed > settings.taa_tolerance:
                # Sell overweight, buy underweight
                trades_executed = rebalance_to_taa(current_allocation, target_allocation)
                
                # Incur transaction costs
                transaction_cost_bps = settings.transaction_cost_bps
                cash -= total_aum * transaction_cost_bps / 10000
                
                if not trades_executed:
                    break  # Convergence achieved
        
        # Step 10: Record period end
        record_metrics(year, scenario_id, cash, assets, ir_curve, spread)
    
    # End of scenario projection
    final_asset_value = sum(asset.market_value for asset in assets) + cash
    return {
        'final_asset_value': final_asset_value,
        'cash_balances': cash_balances_by_period,
        'asset_values': asset_values_by_period,
        'krd_positions': krd_positions_by_period
    }
```

**Key Decision Points**:
1. **Line "Step 1"**: Liability CF is SCENARIO-DEPENDENT (includes policyholder optionality)
2. **Line "Step 3"**: D&D applies to BOTH market value AND cashflows (dual application)
3. **Line "Step 9a"**: KRD method = Risk_Free (zspread = 0) per liability_cashflow.py:92-97
4. **Line "Step 9"**: Up to 3 iterations; early exit if convergence achieved
5. **Line "Step 9b"**: TAA rebalancing; 🔴 NOTE: 35bps spread cap NOT applied here

**Related code**:
- File: `balance_sheet_projection.py` lines 100-250
- Test: [PHASE3-003](PHASE3-003-Integration-Test-Scenarios.md) tests I1-I8

---

### Algorithm 3: Derivative Rebalancing (KRD Matching)

**Purpose**: Execute swaps to keep interest rate duration matched between assets and liabilities

**Formula**:

$$\text{Target IR Swap Position} = \text{Asset DV01} - \text{Liability DV01}$$

Where:
- $\text{DV01} = \frac{\partial \text{NPV}}{\partial r}$ (change in NPV per 1bp IR move)
- If Asset DV01 > Liability DV01 → Need receive-fixed swap (short rates)
- If Asset DV01 < Liability DV01 → Need pay-fixed swap (long rates)

**Implementation**:

```python
def rebalance_swaps_for_krd(target_dv01, ir_curve, settings):
    """
    Execute swaps to reach target DV01 balance
    """
    # Calculate current KRDs
    asset_krd = sum(asset.dv01(ir_curve) for asset in assets)
    liability_krd = liability.calculate_krd(ir_curve, method='Risk_Free')
    
    current_dv01_gap = asset_krd - liability_krd
    
    # Calculate swap requirement
    dv01_to_cover = target_dv01 - current_dv01_gap
    
    # Calculate swap terms
    if abs(dv01_to_cover) < settings.dv01_tolerance:
        return False  # Converged, no swap needed
    
    # Size the swap (typical: standardized swaps with fixed DV01)
    swap_size = dv01_to_cover / settings.standard_swap_dv01
    
    if swap_size > 0:
        # Need receive-fixed (hedge against rising rates)
        swap = IRSwap(
            notional=swap_size,
            fixed_rate=ir_curve[5],  # 5Y rate assumed
            pv01=settings.standard_swap_dv01,
            direction='Receive'
        )
    else:
        # Need pay-fixed (hedge against falling rates)
        swap = IRSwap(
            notional=abs(swap_size),
            fixed_rate=ir_curve[5],
            pv01=settings.standard_swap_dv01,
            direction='Pay'
        )
    
    # Add to portfolio
    swaps.append(swap)
    return True  # Swap executed
```

**Key Parameters**:
- **liability_dv01_calibration_method**: Must be `Risk_Free` (line 92-97 of liability_cashflow.py)
- **dv01_tolerance**: Convergence threshold (e.g., 0.001 = 1bp sensitivity)
- **standard_swap_characteristics**: Tenor, fixed rate, notional conventions

**Related code**:
- File: `swap_rebalancer.py` lines 240-350
- Test: [PHASE3-002](PHASE3-002-Component-Unit-Tests.md) test U16-U20

---

## Pages 5-7: Component-Level Specifications

### Component 1: Interest Rate Curves & Scenarios

**Input**:
- Euribor 6M swap rates (15 annual + 8 5-year tenors to 50Y)
- Interpolation method (linear)
- Extrapolation beyond 50Y (flat)

**Processing**:

```python
class YieldCurve:
    def __init__(self, swap_rates_dict):
        # swap_rates_dict: {tenor_year: rate}
        # Example: {1: 0.025, 2: 0.030, ..., 50: 0.035}
        
        # Interpolate illiquid tenors
        self.spot_rates = self.interpolate_spot_curve(swap_rates_dict)
        
        # Extrapolate beyond 50Y
        for year in range(51, 101):
            self.spot_rates[year] = self.spot_rates[50]  # Flat
    
    def get_forward_rate(self, from_year, to_year):
        """Calculate implied forward rate"""
        # Forward = (1 + spot_to)^to / (1 + spot_from)^from - 1
        return ((1 + self.spot_rates[to_year]) ** to_year / 
                (1 + self.spot_rates[from_year]) ** from_year) ** (1 / (to_year - from_year)) - 1
    
    def apply_scenario_shock(self, scenario_id, year):
        """Apply BMA-specified IR shock per scenario"""
        shock_bps = settings.get_shock(scenario_id, year)
        return YieldCurve({tenor: rate + shock_bps/10000 for tenor, rate in self.spot_rates.items()})
```

**Scenarios (per BMA Article 28(7))**:

| Scenario | Year 1-10 Shock | Year 11-100 Shock |
|----------|---|---|
| 0 (Base) | 0 bps | 0 bps |
| 1 (Up 25) | +25 bps all | +25 bps all |
| 2 (Down 25) | -25 bps all | -25 bps all |
| 3 (Up 25 flipped) | +25 bps then -25 | -25 bps | 
| 4 (Down 25 flipped) | -25 bps then +25 | +25 bps |
| 5 (Flattening) | Front +50, Back -50 | Blend to 0 |
| 6 (Steepening) | Front -50, Back +50 | Blend to 0 |
| 7 (Short up) | +50 bps (0-5Y), +25 bps (5Y+) | +25 bps |
| 8 (Long up) | -25 bps (0-5Y), +50 bps (5Y+) | +50 bps |

**Output**:
- Spot yield curve by year and scenario
- Forward rates for period-by-period discounting

**Related code**:
- File: `zspread_provider.py` (full file)
- File: `settings_loader.py` lines 45-110 (scenario definitions)
- Test: [PHASE3-002](PHASE3-002-Component-Unit-Tests.md) tests U21-U26

---

### Component 2: Liability Cashflows & KRD

**Input**:
- Cashflow schedule by scenario, by year
- Interest rate sensitivity (duration)

**Processing**:

```python
class LiabilityCashflow:
    def __init__(self, cf_dict, settings):
        self.cf = cf_dict  # {year: amount, ...}
        self.krd_method = settings.liability_dv01_calibration_method
    
    def calculate_npv(self, ir_curve, spread, zspread):
        """
        Calculate liability NPV
        ⚠️  CRITICAL: zspread must be 0 for Risk_Free method
        """
        if self.krd_method == LiabilityDV01CalibrationMethod.Risk_Free:
            zspread = 0.0  # ✅ Correct
        elif self.krd_method == LiabilityDV01CalibrationMethod.Risk_Free_plus_SBA_Spread:
            zspread = zspread  # ❌ Wrong for SBA (but option available)
        
        npv = 0
        for year, cf_amount in self.cf.items():
            discount_rate = ir_curve[year] + zspread
            npv += cf_amount / ((1 + discount_rate) ** year)
        
        return npv
    
    def calculate_krd(self, ir_curve, stress_bps=1):
        """
        Key Rate Duration: Sensitivity to 1Y shock at each tenor
        Formula: KRD = (NPV_up - NPV_down) / (2 * stress_bps * total_cf)
        """
        krd_by_tenor = {}
        for tenor in range(1, 51):
            # Bump this tenor's rate
            ir_curve_up = ir_curve.copy()
            ir_curve_up[tenor] += stress_bps / 10000
            npv_up = self.calculate_npv(ir_curve_up, 0)  # zspread=0 per Risk_Free
            
            ir_curve_down = ir_curve.copy()
            ir_curve_down[tenor] -= stress_bps / 10000
            npv_down = self.calculate_npv(ir_curve_down, 0)
            
            krd_tenor = (npv_up - npv_down) / (2 * stress_bps / 10000)
            krd_by_tenor[tenor] = krd_tenor
        
        return krd_by_tenor
```

**Critical decision**:
- Line "if self.krd_method == Risk_Free": `zspread = 0` ✅
- File location: `liability_cashflow.py` lines 92-97
- Test: [PHASE3-002](PHASE3-002-Component-Unit-Tests.md) tests U11-U15
- **Verified**: Risk_Free configured in alm_setup.xlsm ✅

**Related code**:
- File: `liability_cashflow.py` lines 85-210
- Test: [PHASE3-002](PHASE3-002-Component-Unit-Tests.md)

---

### Component 3: Defaults & Downgrades (D&D)

**Formula**:

$$\text{Cumulative Default Rate}(t) = 1 - (1 - r_{\text{annual}})^t$$

**Application** (DUAL):
1. **To market values**: `MV_t = MV_0 \times (1 - D\&D(t))`
2. **To cashflows**: `CF_t = CF_{\text{scheduled}} \times (1 - D\&D(t))`

**Implementation**:

```python
def apply_d_and_d(annual_default_rate, year):
    """
    Calculate cumulative D&D scalar for given year
    """
    cumulative_dd_rate = 1 - (1 - annual_default_rate) ** year
    
    # Bounds validation
    assert 0 <= cumulative_dd_rate <= 1, f"D&D out of bounds: {cumulative_dd_rate}"
    
    # Return scalar to apply to both MV and CF
    return 1 - cumulative_dd_rate  # Remaining fraction

# Usage:
for asset in assets:
    dd_scalar = apply_d_and_d(asset.default_rate, year)
    asset.market_value *= dd_scalar
    asset.cf_coupon *= dd_scalar
```

**Verification**:
- Formula matches spec exactly: $1-(1-X)^t$ ✅
- Applied to both MV and CF (dual application) ✅
- Assertions for bounds checking ✅
- File: `asset_projector.py` lines 161-190

**Related code**:
- File: `asset_projector.py` lines 161-190
- Test: [PHASE3-002](PHASE3-002-Component-Unit-Tests.md) tests U1-U5

---

### Component 4: SBA Spread Calculation & Capping

**Purpose**: Calculate constant spread above risk-free rates that values liabilities = assets

**Formula**:

$$\text{Assets}(t=0) = \text{NPV}_{\text{Liabilities}} \left( r_{\text{risk-free}} + s_{\text{SBA}} \right)$$

Where: $s_{\text{SBA}}$ = SBA spread (solve for this)

**Implementation** (currently missing cap 🔴):

```python
def calculate_sba_spread(asset_value_total, liability_cf, ir_curve, settings):
    """
    Solve for SBA spread via goal-seek
    """
    # Goal: Find spread such that NPV_liab(IR + spread) = asset_value
    
    def npv_diff(spread_candidate):
        npv_liab = calculate_npv(liability_cf, ir_curve, spread_candidate)
        return npv_liab - asset_value_total
    
    # Goal-seek solve
    sba_spread = bisect(npv_diff, left=0, right=0.10)  # 0-1000 bps
    
    # 🔴 CRITICAL GAP: Spread cap NOT enforced
    # Should be: sba_spread = min(sba_spread, 0.0035)  # 35 bps cap
    # But this line is MISSING from the code!
    
    return sba_spread
```

**Known Gap**:
- Parameter `sba_spread_cap` = 0.0035 (35 bps) defined
- But NEVER USED in calculation above 🔴
- File: `settings_loader.py` line 76 (definition only)
- Grep: "spread_cap" appears 1 time (definition), 0 times (enforcement)
- **Impact**: Regulatory non-compliance if spreads > 35bps

**Mitigation Options**:
1. ✅ Add line: `sba_spread = min(sba_spread, settings.sba_spread_cap)`
2. ✅ Document exception if cap not needed per risk assessment
3. ✅ Test via edge case E4.5 (PHASE3-004)

**Related code**:
- File: `settings_loader.py` line 76 (cap defined but not used 🔴)
- File: `liability_cashflow.py` lines 156-210 (spread application)
- Test: [PHASE3-004](PHASE3-004-Edge-Case-Stress-Tests.md) test E4.5

---

## Pages 8-9: Data Structures & Parameters

### Critical Configuration Parameters (alm_setup.xlsm)

| Parameter | Type | Range | Default | Impact |
|-----------|------|-------|---------|--------|
| **goal_seek_tolerance_optimal** | float | 0.0001%-0.01% | 0.001% | Goal-seek convergence precision |
| **goal_seek_tolerance_absolute** | float | 0.01%-1% | 0.1% | Fallback if optimal doesn't converge |
| **max_goal_seek_attempts** | int | 10-50 | 20 | Hard stop for goal-seek iterations |
| **liability_krd_method** | enum | {Risk_Free, Risk_Free_plus_SBA_Spread} | Risk_Free | ✅ MUST BE Risk_Free |
| **sba_spread_cap** | float | 0-1000 bps | 35 bps | 🔴 Defined but NOT used |
| **investment_expense_bps** | float | 0-100 bps | Asset-driven | Annual fee on AUM |
| **transaction_cost_bps** | float | 0-50 bps | 2-5 bps | Cost per rebalancing trade |
| **dv01_tolerance** | float | 0.0001-0.01 | 0.001 | KRD convergence threshold |
| **taa_tolerance** | float | 0.01-1% | 0.1% | TAA rebalancing trigger |
| **projection_years** | int | 50-200 | 100 | Projection horizon |
| **spread_mean_reversion_rho** | float | 0.8-0.99 | 0.95 | Mean reversion speed (per year) |

### Input Data Structures

**Asset Portfolio** (CSV format):
```
ISIN,AssetName,Type,Rating,Maturity,Coupon,MarketValue,Spread
DE0001141400,Bund,Sovereign,AAA,2027-06-30,0.0050,1000000,0.0005
DE0005140008,DBU Senior,Corporate,A,2030-12-31,0.0175,2500000,0.0085
NL0009015000,ABN Dutch RML,Mortgage,Unrated,2035-03-31,0.0250,5000000,0.0015
...
```

**Liability Cashflows** (By scenario, provided by Business Units):
```
Year,ScenarioBase,ScenarioUp25,ScenarioDown25,...
1,5000000,4950000,5050000,...
2,5100000,5000000,5200000,...
...
100,1000000,950000,1050000,...
```

**Configuration Sheet** (in alm_setup.xlsm):
```
Parameter,Value
goal_seek_tolerance_optimal,0.001%
liability_krd_method,Risk_Free
sba_spread_cap,35bps
dryrun_mode,FALSE
...
```

### Output Data Structures

**SBA Results**:
```json
{
  "sba_bel": 1250000,
  "sba_spread": 0.0032,  // 32 bps (below 35 bps cap, if enforced)
  "biting_scenario": "Down 25bps",
  "scenario_results": {
    "Base": {"assets_required": 1200000, "final_spread": 0.0030},
    "Up 25": {"assets_required": 1180000, "final_spread": 0.0028},
    ...
  },
  "krd_position": {
    "asset_krd": [-50, -45, -40, ...],  // By tenor
    "liability_krd": [-50, -45, -38, ...],
    "swap_hedges": [{"tenor": "5Y", "notional": 500000, "direction": "Receive"}]
  },
  "validation": {
    "cash_never_negative": true,
    "all_liabilities_covered": true,
    "convergence_achieved": true,
    "goal_seek_attempts": 12
  }
}
```

---

## Pages 10-11: Edge Case Handling & Error Scenarios

### Edge Case 1: Negative Cash Balance During Projection

**Scenario**: Despite t=0 asset amount, intermediate year goes negative

**Handling**:
```python
min_cash = min(cash_ledger[all_years])
if min_cash < 0:
    # This scenario infeasible; goal-seek needs more assets
    return {
        'feasible': False,
        'deficit': abs(min_cash),
        'deficit_year': year_with_min_cash
    }
```

**Related test**: [PHASE3-004](PHASE3-004-Edge-Case-Stress-Tests.md) edge case E7

---

### Edge Case 2: Goal-Seek Non-Convergence

**Scenario**: After 20 attempts, convergence not achieved

**Handling**:
```python
if attempts >= settings.max_goal_seek_attempts:
    # Fall back to loose tolerance
    final_estimate = best_estimate_with_tolerance(
        settings.tolerance_absolute
    )
    log.warning(f"Goal-seek reached max attempts; using absolute tolerance")
    return {
        'estimate': final_estimate,
        'confidence': 'LOW',
        'tolerance_used': settings.tolerance_absolute
    }
```

**Related test**: [PHASE3-004](PHASE3-004-Edge-Case-Stress-Tests.md) edge case E10

---

### Edge Case 3: KRD Rebalancing Impossible

**Scenario**: Cannot achieve exact KRD match (e.g., insufficient swap liquidity)

**Handling**:
```python
krd_gap = abs(asset_krd - liability_krd)
if krd_gap > settings.krd_tolerance * 10:  # Exceptionally large
    log.error(f"KRD gap {krd_gap} exceeds acceptable threshold")
    # Early exit from rebalancing; record gap
    return {
        'converged': False,
        'remaining_gap': krd_gap,
        'action': 'Escalate to risk management'
    }
```

**Related test**: [PHASE3-004](PHASE3-004-Edge-Case-Stress-Tests.md) edge case E18

---

### Edge Case 4: D&D Rate Extreme

**Scenario**: Annual default rate > 50% or near 100%

**Boundaries**:
```python
annual_dd_rate = asset.get_default_rate()
assert 0 <= annual_dd_rate <= 1.0, f"D&D rate invalid: {annual_dd_rate}"

cumulative_dd = 1 - (1 - annual_dd_rate) ** year
# Mathematically valid for any rate in [0, 1]
# Year 100 with 100% annual: Cumulative → 100% (entire asset lost)
# Year 100 with 1% annual: Cumulative → 63.4% (most lost)

return cumulative_dd  # Always valid
```

**Related test**: [PHASE3-004](PHASE3-004-Edge-Case-Stress-Tests.md) edge cases E5.1-E5.3

---

## Pages 12: Testing & Validation Requirements

### Test Coverage Summary

**Unit Tests** (26 tests):
- D&D formula: 5 tests
- Goal-seek algorithm: 5 tests  
- KRD calculation: 5 tests
- Rebalancing: 5 tests
- Z-spread: 6 tests

**Integration Tests** (8 tests):
- 6-month projection: I1
- 1-year projection: I2
- 101-year full term: I3
- Market stress scenario: I4
- Credit event: I5
- Data integrity: I6
- Rebalancing convergence: I7
- End-to-end update: I8

**Edge Case Tests** (20 tests):
- Portfolio extremes: E1-E3
- Market stress: E4-E6
- Data anomalies: E7-E9
- Numerical edge cases: E10-E14
- Configuration bounds: E15-E17
- Error handling: E18-E20

**Regression Framework**: Baseline capture + drift detection

---

## Summary: Developer Checklist

Before going to production:

- [ ] **Goal-seek algorithm**: Test with 100+ scenarios; verify convergence
- [ ] **KRD matching**: Confirm Risk_Free method is ALWAYS used (line 92-97)
- [ ] **Spread cap**: ADD enforcement logic or document exception
- [ ] **D&D formula**: Verify $1-(1-X)^t$ applied to both MV and CF
- [ ] **Rebalancing**: Test up-to-3-iteration loop with early exit
- [ ] **Edge cases**: Run all 20 edge case tests; document failures
- [ ] **Regressions**: Setup baseline; monitor for drift
- [ ] **Known gaps**: Document spread cap, callable bonds, D&D limits

---

**Document**: SBA BEL Implementation Specification  
**Status**: Active development  
**Last Updated**: March 24, 2026  
**Next**: Execute PHASE3 test suite
