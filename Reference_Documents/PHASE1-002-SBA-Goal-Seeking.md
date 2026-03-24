# 01-SBA-Goal-Seeking.md

# SBA GOAL SEEKING MODULE - COMPREHENSIVE DEEP DIVE

**File**: [site-packages/alm/model/projection/sba_goal_seeker.py](../../Athora_BEL_Model_v0.1.8/site-packages/alm/model/projection/sba_goal_seeker.py)

**Purpose**: Implements the Solvency Business Assessment (SBA) optimization algorithm that iteratively adjusts asset/liability proportions to find balance over a 100-year projection horizon.

---

## Table of Contents
1. Class Structure & Data Models
2. Core Algorithm (Detailed Pseudocode)
3. Iteration Control Parameters  
4. Asset Scalar Strategies
5. Convergence Detection
6. Data Flow (Input/Output)
7. Edge Cases & Error Handling
8. Debug Output & Reporting
9. Performance Characteristics
10. Verification Checklist

---

## 1. Class Structure & Data Models

### OptimiseMode Enum
```python
class OptimiseMode(Enum):
    NotDefined = auto()          # Initial state
    Asset = auto()               # Scale down assets (liab > assets)
    Liability = auto()           # Scale down liabilities (assets > liab)
```

**Meaning**: 
- **Asset mode**: Assets are worth more than liabilities. To balance, reduce amount of assets held. Strategy: sell down asset portfolio.
- **Liability mode**: Liabilities have higher PV than assets in absolute value. To balance, reduce amount of liability modeled. Strategy: use only portion of actual liabilities.

### OptimiseScore Dataclass
```python
@dataclass
class OptimiseScore:
    """Convergence metric calculated at end of each projection"""
    
    idx_asset: int
        # Time period where asset value becomes less than 1
        # (meaning asset position scaled down significantly)
        
    idx_liab: int
        # Critical time period (index of biting constraint)
        # The period where balance is tightest
        
    value_asset: float                # €
        # Asset market value at idx_liab period
        # Used for asset-side balance check
        
    value_liab: float                 # €
        # Liability present value at idx_liab 
        # Typically negative (liability = obligation)
        
    has_pos_cash_all_periods: float  # €, minimum
        # Safeguard: minimum cash balance across ALL projection periods
        # Should be ≥ -1€ (positive cash requirement)
        
    value_liab_t_zero: float          # €
        # Liability value at t=0 (reference for scaling checks)
        
    within_tolerance_optimal: bool
        # abs(value_asset + value_liab) <= goal_seek_tolerance_optimal
        # e.g., abs(net_value) <= €500k
        
    within_tolerance_absolute: bool
        # abs(value_asset + value_liab) <= goal_seek_tolerance_absolute
        # e.g., abs(net_value) <= €5M (fallback tolerance)
    
    @property
    def value_net(self) -> float:
        """Net asset + liability value"""
        return self.value_asset + self.value_liab
    
    @property
    def success_optimal(self) -> bool:
        """Did we converge to optimal?"""
        return (
            self.value_net >= 0 and 
            self.has_pos_cash_all_periods > -1 and 
            self.within_tolerance_optimal
        )
    
    @property
    def success_absolute(self) -> bool:
        """Did we converge to absolute fallback?"""
        return (
            self.value_net >= 0 and 
            self.has_pos_cash_all_periods > -1 and 
            self.within_tolerance_absolute
        )
```

**Interpretation**:
- `value_net = 0`: Perfect balance (asset value offsets liability obligation)
- `value_net > 0`: Surplus (more assets than needed for liabilities)
- `value_net < 0`: Deficit (insufficient assets to cover liabilities)
- `has_pos_cash_all_periods > 0`: Portfolio stays solvent throughout projection
- `has_pos_cash_all_periods ≤ -1`: Portfolio goes insolvent at some point (rejection criterion)

### Attempt Dataclass
```python
@dataclass
class Attempt:
    """Record of one goal-seek iteration"""
    
    proportion: float
        # Asset or liability proportion tested (0.0 to 1.0)
        # E.g., 0.95 = test at 95% of original amount
        
    results: ProjectionResultsLiability
        # Full projection output (all 101 periods)
        
    score: OptimiseScore
        # Convergence metrics from this attempt
        
    discontinuous: bool
        # Flag: did value_net jump dramatically?
        # Indicates numerical instability or model change
        
    approach: Literal["", "L", "B"]
        # "" = initial or sensitivity test
        # "L" = Linear interpolation approach
        # "B" = Binary (bisection) search approach
    
    @property
    def is_feasible(self) -> bool:
        return (
            score.value_net >= 0 and 
            score.has_pos_cash_all_periods > -1
        )
```

### SBAGoalSeeker Class
```python
@dataclass
class SBAGoalSeeker:
    """Main optimizer class"""
    
    settings: Settings
        # Global configuration (tolerances, limits, etc.)
        
    balance_sheet_projector: BalanceSheetProjection
        # Engine that projects balance sheet over full horizon
    
    @property
    def alm_rebalancing(self):
        """Shortcut to rebalancing strategy from projector"""
        return self.balance_sheet_projector.alm_rebalancing
```

---

## 2. Core Algorithm (Detailed Pseudocode)

### MAIN ENTRY POINT: perform_alm()

```
FUNCTION: perform_alm(
    balance_sheet: BalanceSheet,
    max_sba_liability_ratio: float,      # (0, 1], typically 1.0
    perform_alm_goal_seeking: bool
) -> ProjectionResultsLiability:

┌─────────────────────────────────────────────────────────────────────┐
│ PURPOSE:                                                            │
│ Find minimum amount of liabilities such that assets can cover      │
│ them over full projection horizon, subject to rebalancing limits.  │
│                                                                    │
│ INPUT: Balance sheet with assets/liabilities at 100%              │
│ OUTPUT: ProjectionResults with optimized proportions              │
│         (asset_proportion, sba_liability_proportion)              │
└─────────────────────────────────────────────────────────────────────┘

  ╔════════════════════════════════════════════════════════════════╗
  ║ STEP 1: INITIAL PROJECTION (Full run at 100% asset, X% liab)  ║
  ╚════════════════════════════════════════════════════════════════╝
  
  project_initial ← project_balance_sheet(
      asset_proportion = 1.0,
      sba_liability_proportion = max_sba_liability_ratio,
      show_progressbar = True
  )
  // SLOW: This projects 101 years, with rebalancing each year
  // Typical duration: 5-10 seconds
  
  IF NOT perform_alm_goal_seeking:
    RETURN project_initial    // Skip optimization, use base case
  
  
  ╔════════════════════════════════════════════════════════════════╗
  ║ STEP 2: SCORE INITIAL PROJECTION                              ║
  ╚════════════════════════════════════════════════════════════════╝
  
  score_initial ← _score(project_initial)
  
  // Sanity check: liability should be liability (negative value)
  IF score_initial.value_liab_t0 > 0:
    ERROR: "Liability is positive (asset), unsuitable for SBA"
    RETURN error_result
  
  
  ╔════════════════════════════════════════════════════════════════╗
  ║ STEP 3: DETERMINE OPTIMIZATION DIRECTION                      ║
  ╚════════════════════════════════════════════════════════════════╝
  
  // Question: Are assets sufficient?
  feasible_initial = (
      score_initial.value_net >= 0 AND
      score_initial.has_pos_cash_all_periods > -1
  )
  
  IF feasible_initial:
    // Assets are more than enough
    // Goal: Find MINIMUM assets needed
    // Strategy: Scale DOWN assets gradually
    mode ← OptimiseMode.Asset
    message ← "Assets exceed requirements; optimizing downward"
  
  ELSE:
    // Assets are insufficient
    // Goal: Find MAXIMUM liabilities we can support
    // Strategy: Scale DOWN liabilities gradually
    mode ← OptimiseMode.Liability
    message ← "Assets insufficient; reducing liability"
  
  LOG(message)
  
  
  ╔════════════════════════════════════════════════════════════════╗
  ║ STEP 4: SENSITIVITY TEST (At -3% for stability check)         ║
  ╚════════════════════════════════════════════════════════════════╝
  
  test_proportion ← initial_proportion * 0.97
  
  result_test ← project_balance_sheet(
      asset_proportion = test_proportion,
      sba_liability_proportion = **,
      show_progressbar = False
  )
  // SLOW: Another full 5-10 second projection
  
  score_test ← _score(result_test)
  
  attempts ← [
      Attempt(proportion=1.00, results=project_initial, score=score_initial, 
              approach="", discontinuous=False),
      Attempt(proportion=0.97, results=result_test, score=score_test,
              approach="", discontinuous=False)
  ]
  
  
  ╔════════════════════════════════════════════════════════════════╗
  ║ STEP 5: ITERATIVE OPTIMIZATION (Bisection/Linear Search)      ║
  ╚════════════════════════════════════════════════════════════════╝
  
  result_final ← _optimise_proportion(
      mode = mode,
      results_initial = project_initial,
      balance_sheet = balance_sheet,
      asset_proportion_initial = 1.0,
      sba_liability_proportion_initial = max_sba_liability_ratio,
      attempts = attempts
  )
  // Core iterative algorithm (see next section)
  
  
  ╔════════════════════════════════════════════════════════════════╗
  ║ STEP 6: RETURN RESULT                                          ║
  ╚════════════════════════════════════════════════════════════════╝
  
  LOG(f"Goal seek complete: {len(attempts)} iterations")
  RETURN result_final
```

### ITERATIVE OPTIMIZATION: _optimise_proportion()

```
FUNCTION: _optimise_proportion(
    mode: OptimiseMode,
    results_initial: ProjectionResultsLiability,
    balance_sheet: BalanceSheet,
    asset_proportion_initial: float = 1.0,
    sba_liability_proportion_initial: float = 1.0,
    attempts: list[Attempt] = []
) -> ProjectionResultsLiability:

PURPOSE: 
  Iteratively adjust proportion using bisection or linear search until
  either: (1) converged within tolerance, or (2) max iterations reached.

┌──────────────────────────────────────────────────────────────────────┐
│ TERMINATION CONDITIONS (Checked in priority order)                  │
├──────────────────────────────────────────────────────────────────────┤
│ 1. OPTIMAL CONVERGENCE (Preferred):                                  │
│    - Any attempt meets: success_optimal                              │
│    - AND attempts_count >= goal_seek_attempts_min                    │
│    → EXIT with that attempt                                          │
│                                                                      │
│ 2. ABSOLUTE CONVERGENCE (Fallback):                                  │
│    - Any feasible attempt meets: success_absolute                    │
│    - AND attempts_count >= goal_seek_attempts_limit_optimal          │
│    → EXIT with that attempt                                          │
│                                                                      │
│ 3. HARD STOP (Final fallback):                                       │
│    - attempts_count >= goal_seek_attempts_limit_absolute             │
│    - Use best_feasible attempt found so far                          │
│    → EXIT (may not be converged)                                     │
└──────────────────────────────────────────────────────────────────────┘

ALGORITHM:

  // Initialize search bounds
  x_upper ← initial_proportion * 1.0    // Upper bound
  x_lower ← 0.0                         // Lower bound
  
  x0, x1 ← initial_proportion, sensitivity_proportion  // Last two x tested
  a0, a1 ← attempt[0], attempt[1]       // Last two attempts
  
  best_feasible ← find(attempts | success_optimal OR success_absolute)
  best_feasible ← best_feasible OR find(attempts | is_feasible)
  best_feasible ← best_feasible OR attempts[0]  // Fallback to initial
  
  
  WHILE iterations < goal_seek_attempts_limit_absolute:
    
    ┌──────────────────────────────────────────────────────────────┐
    │ CHECK TERMINATION CONDITIONS                                 │
    ├──────────────────────────────────────────────────────────────┤
    
    best_optimal ← find(attempts | success_optimal)
    
    IF best_optimal AND attempts.count >= goal_seek_attempts_min:
      LOG(f"Optimal convergence at iteration {len(attempts)}")
      RETURN best_optimal.results
    
    best_absolute ← find(attempts | success_absolute)
    
    IF best_absolute AND attempts.count >= goal_seek_attempts_limit_optimal:
      LOG(f"Absolute convergence at iteration {len(attempts)}")
      RETURN best_absolute.results
    
    IF attempts.count >= goal_seek_attempts_limit_absolute:
      LOG(f"HARD STOP: Max iterations reached, using best feasible")
      RETURN best_feasible.results
    
    
    ┌──────────────────────────────────────────────────────────────┐
    │ UPDATE SEARCH BOUNDS (Bisection bounds)                      │
    ├──────────────────────────────────────────────────────────────┤
    
    // Determine if x1 is "upper bound" or "lower bound" in search space
    is_x1_upper_bound ← determine_bound(x1, a1.score, mode)
    
      HELPER FUNCTION: determine_bound(x, score, mode)
      
        feasible ← score.is_feasible
        converged ← score.within_tolerance_optimal
        
        IF mode == Asset:
          // In Asset mode, we reduce assets
          // Upper bound = solution is at SMALLER x (more reduction needed)
          // Lower bound = solution is at LARGER x (less reduction needed)
          
          IF converged:
            RETURN "upper"  // x is lower than needed, can reduce more
          ELSE IF feasible:
            RETURN "upper"  // Feasible but not converged, need more reduction
          ELSE:
            RETURN "lower"  // Infeasible, can't reduce this much
        
        ELSE (mode == Liability):
          // In Liability mode, we reduce liabilities
          // Similar logic but reversed
          ...
        
        END HELPER
    
    IF is_x1_upper_bound:
      x_upper ← min(x1, x_upper)
    ELSE:
      x_lower ← max(x1, x_lower)
    
    // Safety check: bounds converged?
    IF x_upper - x_lower < 0.001:
      LOG("Search bounds converged")
      best_recent ← argmin(attempts[last 5:] | |score.value_net|)
      RETURN best_recent.results
    
    
    ┌──────────────────────────────────────────────────────────────┐
    │ DECIDE SEARCH APPROACH (Linear vs Bisection)                 │
    ├──────────────────────────────────────────────────────────────┤
    
    **SPEC REFERENCE** (Section 3.3.7):
    "At each iteration the algorithm switches between linear and bisection 
     methods. Linear method is faster and normally preferred, however it is 
     usable only if the optimization function is continuous and differentiable 
     in the current neighbourhood. The algorithm identifies if there are 
     discontinuity and jumps at every iteration and uses bisection method 
     which is slower but more robust."
    
    approach ← "L"  // Default to linear
    
    // DISCONTINUITY DETECTION - Switch to bisection if:
    
    // Condition 1: Iteration count threshold
    IF attempts.count >= 8:
      approach ← "B"  // After 8 attempts, use bisection (empirical stability point)
    
    // Condition 2: Found both feasible and infeasible solutions
    //  This indicates a discontinuity/jump in the function
    IF (exists x_high AND exists x_low 
        where x_high.score.feasible AND x_low.score.infeasible):
      approach ← "B"  // Discontinuity detected: bracket has feasible on one side, infeasible on other
      // Indicates non-smooth behavior (e.g., due to asset rebalancing rules or cash constraints)
    
    // Asset explosion check
    IF asset_count_current > min(asset_count_prev * 3, asset_count_prev + 2000):
      LOG("Asset explosion detected, terminating early")
      RETURN best_feasible.results
    
    
    ┌──────────────────────────────────────────────────────────────┐
    │ CALCULATE NEXT PROPORTION TO TEST (x2)                       │
    ├──────────────────────────────────────────────────────────────┤
    
    IF approach == "L":  // Linear Interpolation
      
      // Goal: Find x where value_net ≈ 0
      // Linear formula: x2 = x1 + (x1 - x0) * (target - y1) / (y1 - y0)
      
      target ← min(
        goal_seek_tolerance_optimal * 0.5,
        abs(value_liab_t0) * 0.02 / 100  // 2bps of liability
      )
      
      y0 ← a0.score.value_net
      y1 ← a1.score.value_net
      
      IF abs(y1 - y0) < 1.0:  // Slopes too flat, switch to bisection
        approach ← "B"
        GOTO bisection_calc
      
      // Uncapped linear step
      uncapped_change ← (x1 - x0) * (target - y1) / (y1 - y0)
      
      // Cap the change to ±15% to avoid wild swings
      capped_change ← CLAMP(uncapped_change, -0.15, +0.15)
      
      x2 ← x1 + capped_change
      
      // Ensure x2 stays within bounds
      x2 ← CLAMP(x2, [
        x1 * 0.1,           // Don't drop below 10% of last x
        x_lower,            // Respect lower bisection bound
        x_upper             // Respect upper bisection bound
      ])
      
      // If we hit a bound, switch to bisection (can't move in direction)
      IF x2 == x_lower OR x2 == x_upper:
        approach ← "B"
    
    ELSE (approach == "B"):  // Bisection
    
      bisection_calc:
      // Find middle of search space, weighted toward best guess
      
      closest ← argmin(attempts | |score.value_net|)  // Best attempt so far
      
      IF closest.approach == "L":  // Came from linear search
        
        // Weight toward the side of closest attempt
        is_closer_to_lower ← abs(closest.proportion - x_lower) 
                             < abs(closest.proportion - x_upper)
        
        IF is_closer_to_lower:
          weight ← 0.2  // 20% from lower, 80% from upper
        ELSE:
          weight ← 0.8  // 80% from lower, 20% from upper
        
        x2 ← x_lower + (x_upper - x_lower) * weight
      
      ELSE:  // Standard bisection
        x2 ← (x_upper + x_lower) / 2
      
      x2 ← CLAMP(x2, [x_lower, x_upper])
    
    
    ┌──────────────────────────────────────────────────────────────┐
    │ EXECUTE PROJECTION AT x2                                     │
    ├──────────────────────────────────────────────────────────────┤
    
    a2 ← project_and_score(x2, balance_sheet)
    // SLOW: Full projection at new proportion
    
    attempts.append(a2)
    LOG(f"Iteration {len(attempts)}: x={x2:.4f}, value_net={a2.score.value_net:.0f}")
    
    // Update for next iteration
    x0, x1 ← x1, x2
    a0, a1 ← a1, a2
    
    // Track best feasible (in case we hit hard stop)
    IF a2.is_feasible:
      IF (best_feasible is None OR 
          abs(a2.score.value_net) < abs(best_feasible.score.value_net)):
        best_feasible ← a2
  
  END WHILE
  
  // Final return
  RETURN best_feasible.results
```

---

## 3. Iteration Control Parameters

All parameters loaded from Excel Parameters sheet or overridden via CLI:

| Parameter | Type | Excel Cell | Default | Purpose |
|-----------|------|-----------|---------|---------|
| `goal_seek_tolerance_optimal` | float | D120 | €500,000 | Target net value (optimal) |
| `goal_seek_tolerance_absolute` | float | D121 | €5,000,000 | Fallback tolerance (absolute) |
| `goal_seek_attempts_min` | int | D122 | 5 | Min iterations before optimal acceptable |
| `goal_seek_attempts_limit_optimal` | int | D123 | 10 | Max iterations to achieve optimal |
| `goal_seek_attempts_limit_absolute` | int | D124 | 20 | Hard stop (max iterations ever) |
| `goal_seek_method` | enum | D125 | `ReduceAllAssetsEqually` | Asset scaling strategy |

### Parameter Interpretation

**Tolerance Cascade**:
1. Optimal: `abs(value_net) ≤ €500k` (primary target)
2. Absolute: `abs(value_net) ≤ €5M` (acceptable fallback, 10× wider)
3. Hard Stop: `attempts ≥ 20` (give up, use best)

**Why cascade?**
- Goal seek is slow (each iteration = 5-10 seconds)
- Perfect convergence might not be achievable
- Fallback allows graceful degradation

**Typical Flow**:
```
Iteration 1-4:   Rough search
Iteration 5:     Check optimal at min threshold
Iteration 6-9:   Refine, still checking optimal
Iteration 10:    If not optimal yet, switch to absolute tolerance
Iteration 11-19: Refine toward absolute
Iteration 20:    If still not absolute, STOP and use best found
```

---

## 4. Asset Scalar Strategies

### Strategy 1: ReduceAllAssetsEqually

```
SIMPLE SCALING:

scalar(asset) = asset_proportion  ∀ assets

Example:
  asset_proportion = 0.95
  → ALL assets scaled by 95%
  → Corporate bonds reduced, cash reduced, swaps reduced, etc.
```

**Pros**: Simple, maintains actual portfolio structure
**Cons**: May violate allocation targets (SAA), ignores asset class preferences

### Strategy 2: ReduceCashToSAAFirst

```
INTELLIGENT SCALING:

Goal: Reduce assets while maintaining Strategic Asset Allocation

Example:
  SAA target:     {Corporate: 40%, Sovereign: 30%, Cash: 30%}
  Current (CAA):  {Corporate: 45%, Sovereign: 25%, Cash: 30%}
  
  To reduce 5% of total portfolio:
  → Reduce from Corporate first (45% → 42.5%)
  → Leave Sovereign and Cash in SAA proportion
  → Result: meet SAA target while shrinking

Algorithm:

  rate_cash_saa ← SAA[cash]  // e.g., 0.30
  mv_cash_caa ← Σ(market_values of cash assets)
  mv_non_cash_caa ← Σ(market_values of non-cash assets)
  mv_total ← mv_cash_caa + mv_non_cash_caa
  
  // Total reduction needed
  target_reduction_total ← (1 - asset_proportion) * mv_total
  
  // How much can we reduce cash without violating SAA minimum?
  max_cash_reduction_allowed ← mv_cash_caa  // Can't go negative
  
  // If we reduce <X from cash, what proportion remains?
  rate_cash_min ← (mv_cash_caa - max_cash_reduction_allowed) / mv_cash_caa
                = 0.0  // Can reduce all cash if needed
  
  // Solve: what scalar keeps cash at its SAA proportion?
  IF mv_cash * scalar / mv_total == rate_cash_saa:
    scalar_cash = rate_cash_saa * mv_total / (mv_total - mv_cash + mv_cash * scalar)
    
    // Algebra gives:
    scalar_cash = rate_cash_saa * mv_non_cash / 
                  ((1 - rate_cash_saa) * mv_cash)
  
  // Final scalars
  scalar_cash ← CLAMP(scalar_cash, rate_cash_min, 1.0)
  scalar_non_cash ← (mv_total * asset_proportion) / 
                    (mv_cash * scalar_cash + mv_non_cash)
  
  RETURN lambda asset:
    IF asset.is_cash:
      return scalar_cash
    ELSE:
      return scalar_non_cash
```

**Effect**:
```
Before: Assets = {Corp: €45M, Sov: €25M, Cash: €30M} (100% = €100M)
Target: asset_proportion = 0.95 (reduce to €95M)

Option A (ReduceAllAssetsEqually):
  Scalars = {Corp: 0.95, Sov: 0.95, Cash: 0.95}
  After:   {Corp: €42.75M, Sov: €23.75M, Cash: €28.5M} (€95M)
  Allocation: {Corp: 45%, Sov: 25%, Cash: 30%} - UNCHANGED

Option B (ReduceCashToSAAFirst):
  Aim for SAA: {Corp: 40%, Sov: 30%, Cash: 30%}
  Scalars ≈ {Corp: 0.94, Sov: 1.0, Cash: 0.92}
  After:   {Corp: €42.3M, Sov: €25.0M, Cash: €27.6M} (€95M)
  Allocation: {Corp: 44.5%, Sov: 26.3%, Cash: 29.1%} - CLOSER TO SAA
```

---

## 5. Convergence Detection

### Converged: OptimiseScore Conditions

```
OPTIMAL CONVERGENCE SUCCESS:
  ✓ abs(value_asset + value_liab) ≤ goal_seek_tolerance_optimal  (~€500k)
  ✓ value_asset + value_liab ≥ 0  (no deficit)
  ✓ min(cash_balance | t ∈ [0..T]) ≥ -1€  (positive cash always)
  ✓ attempts_count ≥ goal_seek_attempts_min  (min iterations reached)
  
  → EXIT: Use this attempt

ABSOLUTE CONVERGENCE SUCCESS:
  ✓ abs(value_asset + value_liab) ≤ goal_seek_tolerance_absolute  (~€5M)
  ✓ value_asset + value_liab ≥ 0
  ✓ min(cash_balance | t ∈ [0..T]) ≥ -1€
  ✓ attempts_count ≥ goal_seek_attempts_limit_optimal  (10+ iterations)
  
  → EXIT: Use this attempt

HARD STOP:
  ✓ attempts_count ≥ goal_seek_attempts_limit_absolute  (20+ iterations)
  
  → EXIT: Use best feasible found (may not be converged)
```

### Non-Converged: Rejection Criteria

```
REJECT ATTEMPT IF:
  ✗ value_asset + value_liab < 0  (deficit)
  ✗ min(cash_balance) < -1€  (insolvency detected)
  ✗ Contains NaN or Inf  (numerical error)
  ✗ Asset count exploded (3x growth or 2000+ new assets)

  → Mark attempt infeasible, exclude from best_feasible
```

---

## 6. Data Flow (Input/Output)

### INPUT to perform_alm()

```python
perform_alm(
    balance_sheet: BalanceSheet,
    max_sba_liability_ratio: float,
    perform_alm_goal_seeking: bool
)
```

**balance_sheet** contains:
```
Assets (market values at 100%):
  ├─ Corporate bonds: €45M
  ├─ Sovereign bonds: €25M
  ├─ Interest rate swaps: ±€5M (net PV)
  ├─ Swaptions: €2M (premium)
  └─ Cash: €30M

Liabilities (cashflows at 100%):
  ├─ Insurance obligations: Stream(t=0: €0, t=1: €5M, t=2: €4.5M, ...)
  ├─ Z-spread settings (for liability accumulation)
  └─ SBA flags (eligible yes/no)
```

**max_sba_liability_ratio** (typically 1.0):
- Percentage of actual liability to model in SBA
- E.g., 0.5 = only use 50% of actual liability (more conservative)

**perform_alm_goal_seeking** (typically True):
- If False: project at 100% and return (no optimization)
- If True: run iterative goal seek

### OUTPUT from perform_alm()

```python
ProjectionResultsLiability:
  asset_proportion: float              # E.g., 0.974 (97.4% of original)
  sba_liability_proportion: float      # E.g., 0.753 (75.3% of original)
  
  balance_sheet_runoff: DataFrame
    | Period | Asset_Value | Liability_PV | Net | Cash_Min | ...
    |--------|-------------|--------------|-----|----------|
    | 0      | €97.4M      | -€97.4M      | €0  | €1.2M    | ...
    | 1      | €96.8M      | -€96.5M      | €0.3M| €0.8M   | ...
    | ...    | ...         | ...          | ... | ...      | ...
    | 100    | €87.2M      | -€87.1M      | €0.1M| €0.1M   | ...
  
  portfolio_runoff: DataFrame
    | Period | Asset   | Model_Type | MV_Start | MV_End | DV01 | ...
    |--------|---------|-----------|----------|--------|------|
    | 0      | ISIN1   | FixedBond | €5.0M    | €5.05M | -€2k | ...
    | 0      | ISIN2   | Swap      | €-1.2M   | €-1.1M | €8k  | ...
    | ...    | ...     | ...       | ...      | ...    | ...  | ...
  
  sba_check_asset_liability_runoff_together: DataFrame
    | Metric              | Value      |
    |---------------------|-----------|
    | Converged           | TRUE      |
    | Value_Net (t=0)     | €50k      |
    | Proportions_Applied | 97.4% / 75.3% |
    | Attempts            | 8         |
    | Best_Tolerance_Met  | Optimal   |
```

---

## 7. Edge Cases & Error Handling

### Case 1: Liability is Positive Value
```
Scenario: Liability has PV = €5M (asset-like)

Error: "Liability is positive, unsuitable for SBA"

Cause: Data error or unusual pricing model

Solution: Check Excel parameters for liability definition
```

### Case 2: No Convergence After 20 Attempts
```
Scenario: Optimal NOT reached, absolute NOT reached

Behavior: Return best_feasible attempt (may have |value_net| = €2M)

Logging: "HARD STOP: Max iterations reached, using best feasible"

Root Cause:
  - Rebalancing constraints too tight
  - Asset class allocations mismatched to liability duration
  - Numerical precision issues

Solution: Manually review goal_seek_tolerance_absolute value
```

### Case 3: Asset Explosion
```
Scenario: Asset count grows from 1000 → 3000+ during projection

Trigger: asset_count_current > min(count_prev * 3, count_prev + 2000)

Behavior: TERMINATE early, return best_feasible

Logging: "Asset explosion detected, terminating early"

Root Cause:
  - Reinvestment creating too many bond positions
  - Swap rebalancer increasing legs
  - Memory/performance concern

Solution: Check market projection for realistic cashflow amounts
```

### Case 4: Negative Cash in Some Periods
```
Scenario: Portfolio goes insolvent in year 47

Check: min(cash_balance | t=0..100) = -€5M

Decision: REJECT this attempt (infeasible)

Move On: Try different proportion or scaling approach

Recovery: Goal seeker adjusts proportion, tries again
```

### Case 5: Discontinuity Detected
```
Scenario: value_net jumps from €-100k to €+€400k between iterations

Flag: discontinuous = True

Cause:
  - Rebalancing threshold crossed (buy/sell triggers)
  - Swap rebalancer deleted position
  - Asset matured
  - Rate curve extrapolation changed

Action: Log warning, switch to bisection (more stable than linear)
```

---

## 8. Debug Output & Reporting

### Log Output Example

```
[00:00:30] Starting SBA Goal Seek...
[00:00:31] Initial projection (asset%=100%, liab%=100%)...
[00:00:36] Initial score: value_asset=€97.5M, value_liab=-€97.3M, net=€0.2M
[00:00:36] Feasible: YES (cash_min=€1.2M)
[00:00:36] Mode: ASSET (need to reduce assets)
[00:00:36] Testing sensitivity at 97%...
[00:00:42] Sensitivity score: net=-€0.1M
[00:00:42]
[00:00:42] Iteration 1: x=0.9850 (98.5%), approach=L
[00:00:47] Score: net=€0.05M, cash_min=€1.1M, feasible=YES
[00:00:47] Bounds: x_lower=0.900, x_upper=0.985
[00:00:48]
[00:00:48] Iteration 2: x=0.9738 (97.38%), approach=L
[00:00:53] Score: net=€0.01M, cash_min=€1.0M, feasible=YES
[00:00:53] CONVERGED TO OPTIMAL (within €500k)
[00:00:53] Attempts: 4, Time: 23 seconds
[00:00:54] Final proportions: asset_pct=97.38%, liability_pct=100%
```

### Attempts Summary Table

```python
attempts_df = pd.DataFrame([
    {
        'Iteration': 0,
        'Approach': '',
        'Proportion': 1.0000,
        'Value_Asset_€m': 97.5,
        'Value_Liab_€m': -97.3,
        'Value_Net_€m': 0.2,
        'Cash_Min_€m': 1.2,
        'Feasible': 'YES',
        'Within_Optimal': 'NO',
        'Within_Absolute': 'YES',
    },
    {
        'Iteration': 1,
        'Approach': 'L',
        'Proportion': 0.9850,
        'Value_Asset_€m': 96.0,
        'Value_Liab_€m': -96.0,
        'Value_Net_€m': 0.05,
        'Cash_Min_€m': 1.1,
        'Feasible': 'YES',
        'Within_Optimal': 'YES',
        'Within_Absolute': 'YES',
    },
    # ... more rows
])
```

---

## 9. Performance Characteristics

### Time Complexity

```
Per Iteration: O(T × N)
  where T = projection length (100 years)
        N = number of assets (1000 typical)
  
  Assumes: O(1) per asset per year (QuantLib cached)
  
Real-world: ~5-10 seconds per iteration

Total Goal Seek: O(K × T × N)
  where K = number of iterations (5-20 typical)
  
  Typical: 8 iterations × 5 seconds = 40 seconds
  Worst: 20 iterations × 10 seconds = 200 seconds (3+ minutes)
```

### Memory Usage

```
Per Projection Result: ~50-100 MB
  - 101 time periods
  - 1000 assets × metadata
  - Scenario comparisons

Total In Memory: ~1 GB
  - 8-10 attempts × 100 MB
  - Market curves cached
  - Asset parameter tables

Swap Out: Results written to disk after each iteration
```

### Optimization Tips

1. **Enable Cache**: `enable_cache=TRUE` saves 30% time
2. **Disable Rebalancing**: `rebalance_enable=FALSE` saves 20% time
3. **Reduce Projection Term**: 50 years vs 100 years saves 50% time
4. **Dryrun Mode**: `--dryrun` flag: 1 year, no goal seek = 0.5 seconds (validation only)

---

## 10. Verification Checklist

**When implementing or reviewing SBA goal seeking, verify**:

- [ ] Goal seek parameters match User Guide section 2.2
  - [ ] goal_seek_attempts_min ≥ 3 (prevent premature exit)
  - [ ] goal_seek_tolerance_absolute > goal_seek_tolerance_optimal (cascade works)
  - [ ] goal_seek_attempts_limit_optimal ≤ goal_seek_attempts_limit_absolute
  
- [ ] Asset scalar strategies implemented correctly
  - [ ] ReduceAllAssetsEqually: all assets get same scalar
  - [ ] ReduceCashToSAAFirst: cash treated separately, SAA preserved
  
- [ ] Convergence detection logic
  - [ ] Optimal: net value ≤ tolerance_optimal AND cash positive
  - [ ] Absolute: net value ≤ tolerance_absolute AND cash positive
  - [ ] Hard stop: max iterations reached
  
- [ ] Iteration algorithm
  - [ ] Linear search used initially (faster convergence)
  - [ ] Bisection used after 8 iterations (stability)
  - [ ] Bounds updated correctly each iteration
  - [ ] Asset explosion detection prevents runaway
  
- [ ] Error handling
  - [ ] Positive liability value flagged as error
  - [ ] Negative cash periods cause rejection
  - [ ] Discontinuities trigger switch to bisection
  
- [ ] Output reconciliation
  - [ ] asset_proportion + sba_liability_proportion align with final iteration
  - [ ] balance_sheet_runoff shows net ≈ 0 throughout projection
  - [ ] Portfolio value scales exactly by proportions
  
- [ ] Performance
  - [ ] Single iteration < 15 seconds (if > 20s, check rebalancing)
  - [ ] Total goal seek < 5 minutes (if > 10 min, try dryrun)
  - [ ] Memory < 2 GB (if OOM, reduce assets or disable cache)

---

## References

**Excel Parameters**: Setup workbook → Parameters sheet (rows 120-125)
**User Guide**: "04.04 _50 SBA Python Model User Guide v0.3.pdf", Section 2.2
**Code**: [site-packages/alm/model/projection/sba_goal_seeker.py](../../Athora_BEL_Model_v0.1.8/site-packages/alm/model/projection/sba_goal_seeker.py)
