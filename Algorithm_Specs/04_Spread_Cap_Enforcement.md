# Algorithm Specification 04: Spread Cap Enforcement

**Target Module:** `calculations/bel_calculator.py`
**Status:** Implementation-Ready
**Version:** 1.0
**Date:** 2026-03-24

---

## 1. Overview

### 1.1 Purpose

This specification defines the algorithm for enforcing the BMA-mandated 35 basis point (bps) spread cap on the implied SBA spread when determining the Best Estimate Liability (BEL). The spread cap is a regulatory constraint ensuring that insurers do not use an unreasonably high discount rate to deflate their reported liabilities.

The spread cap mechanism sits at the final stage of the BEL calculation pipeline. After the projection engine has determined C0 (initial cash buffer) for each of the 9 scenarios via goal-seeking, this module:

1. Derives the implied SBA spread for each scenario.
2. Checks whether the spread exceeds the 35bps cap.
3. If the cap binds, recalculates BEL using the capped spread.
4. Selects the most onerous scenario based on the **capped** BEL values.

### 1.2 Target Module

`calculations/bel_calculator.py` -- orchestrates BEL determination. Consumes results from:
- `projection/` -- C0 per scenario from goal-seeking
- `curves/` -- risk-free curves and PV calculations (QuantLib boundary)

### 1.3 Regulatory References

| Reference | Description |
|-----------|-------------|
| Rules Schedule XXV Para 28(10)(a-c) | BEL = highest asset requirement across all 9 scenarios |
| Rules Schedule XXV Para 28(7)(a-i) | Definition of the 9 interest rate scenarios |
| Handbook E9 | Risk-free curve guidance, spread consistency |
| Handbook D19.1 | BEL definition and calculation |
| Subpara. 11 | Biting scenario = scenario requiring most assets |

### 1.4 Why This Matters

**The Pythora reference model defined the 35bps spread cap as a configuration parameter (`sba_spread_cap` in `settings_loader.py`, line 76) but never enforced it in the calculation code.** This was their single largest implementation gap. The cap was loaded, stored, and available -- but no code path ever compared the derived spread against it or triggered a BEL recalculation. Our implementation must close this gap with explicit, auditable enforcement logic.

---

## 2. The Spread Cap Rule

### 2.1 Regulatory Requirement

The BMA mandates that the SBA spread used to discount liability cash flows must not exceed **35 basis points (0.0035)** above the risk-free rate. This cap ensures:

- Liabilities are not discounted too aggressively (which would understate them).
- BEL values remain conservative across all scenarios.
- Consistency across companies -- no firm can claim an outsized spread advantage.

### 2.2 Economic Intuition

When the implied spread is high (e.g., 50bps), it means assets are generating substantial excess return over risk-free. Discounting liabilities at this high rate produces a lower PV (smaller liability). The cap forces the insurer to use a more conservative (lower) discount rate, which increases the PV of liabilities, producing a higher (more conservative) BEL.

**Key relationship:**
```
Higher spread  -->  Higher discount rate  -->  Lower PV(liabilities)  -->  Lower BEL
Cap enforced   -->  Lower discount rate   -->  Higher PV(liabilities) -->  Higher BEL (more conservative)
```

### 2.3 Cap Threshold

```
SPREAD_CAP = 0.0035  (35 basis points)
```

This value is a regulatory constant, not a model assumption. It should be stored as a named constant, not buried in configuration. However, the implementation must support an override parameter for sensitivity analysis and future regulatory changes.

---

## 3. Algorithm Specification

### 3.1 Inputs

| Input | Source | Description |
|-------|--------|-------------|
| `scenario_results[1..9]` | `projection/` | Goal-seek results per scenario, containing C0, converged asset proportion |
| `liability_cashflows` | `model_points/` | Liability cash flow schedule (annual, 100-year horizon) |
| `risk_free_curves[1..9]` | `curves/` | Scenario-specific risk-free term structures |
| `asset_mv_t0` | `model_points/` | Market value of assets at T=0 |
| `spread_cap` | Configuration | Default 0.0035 (35bps); overridable for sensitivity |

### 3.2 Outputs (Per Scenario)

| Output Field | Type | Description |
|--------------|------|-------------|
| `sba_spread_uncapped` | float | Implied spread from goal-seek (can be > 35bps) |
| `sba_spread_capped` | float | min(uncapped, 35bps) |
| `sba_BEL_uncapped` | float | BEL calculated using uncapped spread |
| `sba_BEL_capped` | float | BEL calculated using capped spread (= uncapped if cap does not bind) |
| `sba_avg_dis_rate_uncapped` | float | Average discount rate using uncapped spread |
| `sba_avg_dis_rate_capped` | float | Average discount rate using capped spread |
| `cap_binding` | bool | True if uncapped spread > cap |
| `bel_impact_of_cap` | float | sba_BEL_capped - sba_BEL_uncapped (0 if cap not binding) |

### 3.3 Aggregate Outputs

| Output Field | Type | Description |
|--------------|------|-------------|
| `biting_scenario_id` | int | Index of scenario with highest capped BEL |
| `biting_scenario_name` | str | Name of biting scenario (e.g., "SBA_2_Down") |
| `BEL_final` | float | Highest capped BEL = BEL from biting scenario |
| `C0_biting` | float | C0 from the biting scenario |
| `scenarios_where_cap_binds` | list[int] | List of scenario indices where spread > 35bps |

### 3.4 Complete Algorithm (Step-by-Step)

```
ALGORITHM: SpreadCapEnforcement

INPUT:
  scenario_results[1..9]   -- from projection engine (C0, asset proportions)
  liability_cfs             -- liability cash flow schedule
  rf_curves[1..9]          -- scenario-specific risk-free curves
  asset_mv_t0              -- market value of assets at T=0
  spread_cap = 0.0035      -- 35bps default

OUTPUT:
  scenario_metrics[1..9]   -- per-scenario spread/BEL metrics
  biting_scenario          -- most onerous scenario (highest capped BEL)
  BEL_final                -- regulatory BEL

PROCEDURE:

  -- Phase 1: Per-Scenario Spread Derivation and Cap Check
  FOR each scenario s in [1..9]:

    -- Step 1: Retrieve goal-seek results
    C0[s] = scenario_results[s].initial_cash_buffer
    asset_proportion[s] = scenario_results[s].asset_proportion
    MV_scaled[s] = asset_mv_t0 * asset_proportion[s]

    -- Step 2: Calculate uncapped BEL
    BEL_uncapped[s] = MV_scaled[s] + C0[s]

    -- Step 3: Derive implied SBA spread via goal-seek
    -- Find spread S such that PV(liability_cfs, rf_curve[s] + S) = BEL_uncapped[s]
    S_uncapped[s] = solve_for_spread(
        target_pv   = BEL_uncapped[s],
        cashflows   = liability_cfs,
        base_curve  = rf_curves[s]
    )

    -- Step 4: Apply spread cap
    IF S_uncapped[s] > spread_cap THEN
      cap_binding[s] = TRUE
      S_capped[s] = spread_cap

      -- Step 5: Recalculate BEL with capped spread
      BEL_capped[s] = PV(liability_cfs, rf_curves[s] + spread_cap)

    ELSE
      cap_binding[s] = FALSE
      S_capped[s] = S_uncapped[s]
      BEL_capped[s] = BEL_uncapped[s]
    END IF

    -- Step 6: Compute reporting metrics
    avg_dis_rate_uncapped[s] = weighted_avg_rate(rf_curves[s], S_uncapped[s], liability_cfs)
    avg_dis_rate_capped[s]   = weighted_avg_rate(rf_curves[s], S_capped[s], liability_cfs)
    bel_impact[s] = BEL_capped[s] - BEL_uncapped[s]

    -- Store per-scenario results
    scenario_metrics[s] = {
      sba_spread_uncapped:       S_uncapped[s],
      sba_spread_capped:         S_capped[s],
      sba_BEL_uncapped:          BEL_uncapped[s],
      sba_BEL_capped:            BEL_capped[s],
      sba_avg_dis_rate_uncapped: avg_dis_rate_uncapped[s],
      sba_avg_dis_rate_capped:   avg_dis_rate_capped[s],
      cap_binding:               cap_binding[s],
      bel_impact_of_cap:         bel_impact[s]
    }

  END FOR

  -- Phase 2: Select Most Onerous Scenario
  biting_scenario = argmax(BEL_capped[s] for s in [1..9])
  BEL_final = BEL_capped[biting_scenario]
  C0_biting = C0[biting_scenario]

  RETURN scenario_metrics, biting_scenario, BEL_final, C0_biting
```

---

## 4. Implied SBA Spread Calculation (Goal-Seek Derivation)

### 4.1 Problem Statement

Given the uncapped BEL for a scenario (which equals the scaled asset MV plus C0), find the constant spread S above the risk-free curve such that discounting the liability cash flows at (rf + S) produces a present value equal to the uncapped BEL.

**Formally:**

```
Find S such that:

  SUM over t=1..T [ CF(t) / PRODUCT over k=1..t [ 1 + rf(k) + S ] ] = BEL_uncapped

Where:
  CF(t)  = liability cash flow at time t
  rf(k)  = risk-free forward rate for period k (from scenario curve)
  S      = constant parallel spread (the unknown)
  T      = projection horizon (typically 100 years)
```

### 4.2 Solution Method: Brentq Root-Finding

Define the objective function:

```python
def spread_objective(S, liability_cfs, rf_curve, target_bel):
    """
    Returns 0 when PV(liab_cfs, rf + S) == target_bel.
    Monotonically decreasing in S (higher spread -> lower PV).
    """
    pv = compute_pv(liability_cfs, rf_curve, spread=S)
    return pv - target_bel
```

Use `scipy.optimize.brentq` to solve:

```python
from scipy.optimize import brentq

uncapped_spread = brentq(
    f=spread_objective,
    a=-0.05,        # Lower bound: -500bps (negative spread possible)
    b=0.20,         # Upper bound: 2000bps (generous ceiling)
    args=(liability_cfs, rf_curve, bel_uncapped),
    xtol=1e-8,      # Tolerance: 0.001bps precision
    maxiter=100
)
```

### 4.3 Why Brentq Works Here

- The PV function is **monotonically decreasing** in S: higher spread -> higher discount rate -> lower PV.
- Brentq is guaranteed to converge for monotonic functions with a sign change in [a, b].
- No derivatives needed (unlike Newton-Raphson).
- Convergence is superlinear, typically < 20 iterations.

### 4.4 Implementation: `compute_pv`

The PV calculation uses the scenario-specific risk-free curve plus the spread:

```python
@rule_ref("E9", "Risk-free curve + spread discounting")
def compute_pv(
    liability_cfs: pd.DataFrame,   # columns: [period, cashflow]
    rf_curve: YieldCurve,          # from curves/ module
    spread: float                  # parallel shift in decimal (e.g., 0.0035)
) -> float:
    """
    Compute present value of liability cash flows using rf + spread.

    This function delegates curve construction to the curves/ module
    (which is the ONLY module that imports QuantLib). The shifted curve
    is: rf_curve parallel-shifted by +spread.

    Returns:
        Present value (positive number representing liability value).
    """
    shifted_curve = rf_curve.parallel_shift(spread)
    discount_factors = shifted_curve.discount_factors(liability_cfs["period"])
    return (liability_cfs["cashflow"] * discount_factors).sum()
```

### 4.5 Average Discount Rate Calculation

The average discount rate is the single flat rate that reproduces the same PV:

```python
def weighted_avg_rate(
    rf_curve: YieldCurve,
    spread: float,
    liability_cfs: pd.DataFrame
) -> float:
    """
    Weighted average discount rate, where weights are the
    PV-weighted contribution of each cash flow.
    """
    shifted_curve = rf_curve.parallel_shift(spread)
    rates = shifted_curve.zero_rates(liability_cfs["period"])
    discount_factors = shifted_curve.discount_factors(liability_cfs["period"])
    pv_weights = liability_cfs["cashflow"] * discount_factors
    total_pv = pv_weights.sum()
    if total_pv == 0:
        return 0.0
    return (rates * pv_weights).sum() / total_pv
```

---

## 5. BEL Recalculation When Cap Binds

### 5.1 When Does the Cap Bind?

The cap binds when `S_uncapped > 0.0035`. This typically occurs in scenarios where:

- Asset yields are significantly above risk-free rates (high-spread environment).
- Stressed scenarios push risk-free rates down while asset MV remains high.
- The portfolio has a high proportion of credit assets with wide spreads.

### 5.2 Recalculation Procedure

When the cap binds for scenario s:

```
Step 1: Retrieve the scenario's risk-free curve: rf_curves[s]
Step 2: Set the discount curve to: rf_curves[s] + 0.0035
Step 3: Compute: BEL_capped[s] = PV(liability_cfs, rf_curves[s] + 0.0035)
```

**Critical detail:** The capped BEL is computed purely from the liability cash flows and the capped discount rate. It is NOT derived from the asset side. When the cap binds, the BEL is decoupled from the asset goal-seek result.

### 5.3 Relationship Between Capped and Uncapped BEL

When the cap binds (uncapped spread > 35bps):

```
S_uncapped > S_capped = 0.0035

Therefore:
  Discount rate (uncapped) > Discount rate (capped)
  PV at uncapped rate < PV at capped rate

So:
  BEL_capped > BEL_uncapped   (always, when cap binds)
```

The cap **always increases** BEL when it binds. This is the conservative direction -- the regulator forces a higher liability value.

### 5.4 Pseudocode for Capped BEL

```python
@rule_ref("E9", "Spread cap enforcement - BEL recalculation")
def calculate_capped_bel(
    liability_cfs: pd.DataFrame,
    rf_curve: YieldCurve,
    uncapped_spread: float,
    spread_cap: float = 0.0035
) -> tuple[float, float, bool]:
    """
    Apply spread cap and return capped BEL.

    Returns:
        (bel_capped, spread_capped, cap_binding)
    """
    if uncapped_spread > spread_cap:
        # Cap binds: recalculate BEL at capped rate
        bel_capped = compute_pv(liability_cfs, rf_curve, spread=spread_cap)
        return bel_capped, spread_cap, True
    else:
        # Cap does not bind: uncapped BEL stands
        bel_uncapped = compute_pv(liability_cfs, rf_curve, spread=uncapped_spread)
        return bel_uncapped, uncapped_spread, False
```

---

## 6. Most Onerous Scenario Selection

### 6.1 Regulatory Requirement

Per Rules Para 28(10): BEL equals the **highest** asset requirement across all 9 scenarios. The "biting scenario" is the one that produces this maximum.

**Critical:** The selection is based on **capped** BEL values, not uncapped. This is because the capped BEL is the regulatory value.

### 6.2 Selection Algorithm

```python
@rule_ref("Para 28(10)", "Biting scenario = highest capped BEL")
def select_biting_scenario(
    scenario_metrics: dict[int, ScenarioMetrics]
) -> tuple[int, str, float]:
    """
    Select the most onerous scenario based on highest capped BEL.

    Returns:
        (scenario_id, scenario_name, BEL_final)
    """
    biting_id = max(
        scenario_metrics.keys(),
        key=lambda s: scenario_metrics[s].sba_BEL_capped
    )
    return (
        biting_id,
        SCENARIO_NAMES[biting_id],
        scenario_metrics[biting_id].sba_BEL_capped
    )
```

### 6.3 Scenario Naming Convention

```python
SCENARIO_NAMES = {
    0: "SBA_0_Base",        # Base case, no rate shock
    1: "SBA_1_Decrease",    # Rates decrease
    2: "SBA_2_Increase",    # Rates increase
    3: "SBA_3_DownUp",      # Down then up
    4: "SBA_4_UpDown",      # Up then down
    5: "SBA_5_Twist1",      # Twist variant 1
    6: "SBA_6_Twist2",      # Twist variant 2
    7: "SBA_7_Twist3",      # Twist variant 3
    8: "SBA_8_Twist4",      # Twist variant 4
}
```

### 6.4 Final BEL and Technical Provision

```
BEL_final = max(BEL_capped[s] for s in 1..9)
C0_biting = C0[biting_scenario]

Technical_Provision = BEL_final + Risk_Margin
Risk_Margin = 6% * CoC_capital  (cost of capital method, separate calculation)
```

---

## 7. Output Reporting Fields

### 7.1 Per-Scenario Detail Table

The following DataFrame is produced with one row per scenario:

| Column | Type | Units | Description |
|--------|------|-------|-------------|
| `scenario_id` | int | -- | 0-8 |
| `scenario_name` | str | -- | Human-readable name |
| `C0` | float | $ | Initial cash buffer from goal-seek |
| `asset_proportion` | float | ratio | Converged asset proportion |
| `sba_spread_uncapped` | float | decimal | Implied spread (e.g., 0.0042 = 42bps) |
| `sba_spread_uncapped_bps` | float | bps | Same, in basis points (e.g., 42.0) |
| `sba_spread_capped` | float | decimal | min(uncapped, cap) |
| `sba_spread_capped_bps` | float | bps | Same, in basis points |
| `sba_BEL_uncapped` | float | $ | BEL before cap |
| `sba_BEL_capped` | float | $ | BEL after cap (regulatory value) |
| `bel_impact_of_cap` | float | $ | Capped - Uncapped (0 if not binding) |
| `cap_binding` | bool | -- | True if spread > 35bps |
| `sba_avg_dis_rate_uncapped` | float | decimal | Weighted avg discount rate (uncapped) |
| `sba_avg_dis_rate_capped` | float | decimal | Weighted avg discount rate (capped) |
| `convergence_status` | str | -- | "optimal" / "absolute" / "not_converged" |

### 7.2 Summary Record

| Field | Value |
|-------|-------|
| `biting_scenario_id` | int |
| `biting_scenario_name` | str |
| `BEL_final` | float (highest capped BEL) |
| `C0_biting` | float |
| `count_scenarios_cap_binding` | int |
| `scenarios_cap_binding` | list[str] |
| `spread_range_uncapped` | (min, max) in bps |
| `spread_range_capped` | (min, max) in bps |

### 7.3 Audit Trail Requirements

Every calculation must be tagged with:
- `@rule_ref` decorator linking to the BMA rule paragraph.
- Input hash (from `RunConfig`) for reproducibility.
- Timestamp and model version.

Both uncapped and capped values are always reported, even when the cap does not bind. This allows auditors to see how close each scenario is to the cap threshold.

---

## 8. Worked Example

### 8.1 Setup

**Portfolio:**
- Liability: 5-year annuity paying $1,000/year
- Assets: Corporate bond portfolio, MV(T=0) = $4,480
- Risk-free rate: varies by scenario
- Spread cap: 35bps (0.0035)

We demonstrate 3 scenarios to illustrate the cap binding, not binding, and most-onerous selection.

### 8.2 Scenario A: Base Case (Cap Does NOT Bind)

**Inputs:**
- Risk-free rate: 3.00% flat
- Goal-seek result: C0 = $2,981, asset_proportion = 1.00
- BEL_uncapped = MV_scaled + C0 = $4,480 + $2,981 = **$7,461**

**Step 1: Derive implied spread.**

Find S such that PV(liability_cfs, 3.00% + S) = $7,461.

Liability cash flows: $1,000 per year for 5 years.

```
PV(CF, r) = 1000/(1+r) + 1000/(1+r)^2 + 1000/(1+r)^3 + 1000/(1+r)^4 + 1000/(1+r)^5
```

At r = 3.00% + S:
- We need PV = $7,461. But the maximum PV of $5,000 (undiscounted) is far less than $7,461.

**Correction:** The BEL in the SBA framework equals `MV_assets(T=0) + C0`. The spread derivation relates to the *liability-only* cash flows. Let us re-state: the implied spread is the rate at which the PV of liability cash flows equals the portion of BEL attributable to liabilities.

For this worked example, we use a more realistic setup:

**Revised Setup (Realistic Scale):**
- Liability cash flows: $10,000/year for 30 years (simplified level annuity)
- Asset MV(T=0): $150,000
- Risk-free rate: 3.00% flat (Scenario A), 2.00% flat (Scenario B), 4.00% flat (Scenario C)
- Spread cap: 35bps

---

#### Scenario A: Base Case (Cap Does NOT Bind)

**Goal-seek result:** C0 = $12,500, asset_proportion = 0.98

```
MV_scaled = $150,000 * 0.98 = $147,000
BEL_uncapped = $147,000 + $12,500 = $159,500
```

**Step 1: Derive implied spread.**

PV of $10,000/year for 30 years at rate r is:

```
PV(r) = 10,000 * [1 - (1+r)^(-30)] / r
```

We need PV(rf + S) = $159,500, with rf = 3.00%.

At r = 3.00% (S = 0):
```
PV(0.03) = 10,000 * [1 - 1.03^(-30)] / 0.03
         = 10,000 * [1 - 0.41199] / 0.03
         = 10,000 * 19.600 = $196,004
```

At r = 3.30% (S = 30bps):
```
PV(0.033) = 10,000 * [1 - 1.033^(-30)] / 0.033
          = 10,000 * [1 - 0.37689] / 0.033
          = 10,000 * 18.882 = $188,820
```

At r = 3.80% (S = 80bps):
```
PV(0.038) = 10,000 * [1 - 1.038^(-30)] / 0.038
          = 10,000 * [1 - 0.32523] / 0.038
          = 10,000 * 17.757 = $177,573
```

At r = 4.60% (S = 160bps):
```
PV(0.046) = 10,000 * [1 - 1.046^(-30)] / 0.046
          = 10,000 * [1 - 0.26141] / 0.046
          = 10,000 * 16.057 = $160,566
```

At r = 4.65% (S = 165bps):
```
PV(0.0465) = 10,000 * [1 - 1.0465^(-30)] / 0.0465
           = 10,000 * [1 - 0.25735] / 0.0465
           = 10,000 * 15.972 = $159,716
```

At r = 4.67% (S = 167bps):
```
PV(0.0467) = 10,000 * [1 - 1.0467^(-30)] / 0.0467
           ≈ $159,500   (close enough for illustration)
```

**Implied spread: S_uncapped = 167bps = 0.0167**

**Step 2: Check cap.**
```
167bps > 35bps?  NO WAIT -- 167bps IS greater than 35bps. Cap BINDS.
```

This scenario shows a high uncapped spread. Let us adjust the example to show a case where the cap does NOT bind. We need the BEL to be closer to the undiscounted PV, meaning a small spread.

---

**Let us use concrete, clean numbers designed to illustrate all three cases clearly.**

#### Revised Worked Example Setup

**Common assumptions:**
- Liability: Level annuity of $10,000/year for 20 years
- Asset portfolio MV(T=0): $150,000
- Spread cap: 35bps (0.0035)

**PV formula for a level annuity:**
```
PV(r) = PMT * [1 - (1+r)^(-n)] / r
```

Where PMT = $10,000, n = 20.

**Reference PVs at various rates (for quick lookup):**

| Rate | PV |
|------|-----|
| 3.00% | $148,775 |
| 3.10% | $147,967 |
| 3.20% | $147,168 |
| 3.30% | $146,377 |
| 3.35% | $145,984 |
| 3.50% | $143,018 |
| 4.00% | $135,903 |
| 4.50% | $129,244 |
| 5.00% | $124,622 |

---

### Scenario 1: Base Case (Cap Does NOT Bind)

**Assumptions:**
- Risk-free rate: 3.10% flat
- Goal-seek result: C0 = $1,500, asset_proportion = 0.983

**Calculations:**

```
MV_scaled = $150,000 * 0.983 = $147,450
BEL_uncapped = $147,450 + $1,500 = $148,950
```

**Derive implied spread:**

Need PV(0.031 + S) = $148,950.

From the reference table, PV(3.10%) = $147,967, which is less than $148,950. We need a rate slightly below 3.10%, meaning the spread S is slightly negative. But this would be unusual.

**Let us simplify further. Use a clean example where BEL is given, and we just show the spread derivation and cap logic.**

---

### 8.2 Final Worked Example (Three Scenarios)

#### Common Setup

| Parameter | Value |
|-----------|-------|
| Liability | Level annuity, $10,000/year for 20 years |
| Spread cap | 35bps (0.0035) |

The PV of this annuity at rate r is:
```
PV(r) = 10,000 * [1 - (1+r)^(-20)] / r
```

#### Scenario 1: Decrease Scenario -- Cap Does NOT Bind

| Input | Value |
|-------|-------|
| Risk-free rate (flat) | 3.50% |
| BEL_uncapped (from goal-seek) | $141,700 |

**Step 1: Derive implied spread.**

We solve for S in: PV(0.035 + S) = $141,700.

```
PV(3.50%) = 10,000 * [1 - 1.035^(-20)] / 0.035 = $142,124
PV(3.60%) = 10,000 * [1 - 1.036^(-20)] / 0.036 = $141,272
```

Linear interpolation:
```
S_uncapped ≈ (141,700 - 142,124) / (141,272 - 142,124) * 0.10%
           = (-424) / (-852) * 0.10%
           ≈ 0.050% above 3.50%
           = 0.50% - 3.50% ...
```

Let us work from rate directly:
```
Target PV = $141,700
PV(3.50%) = $142,124   (rate too low, PV too high)
PV(3.60%) = $141,272   (rate too high, PV too low)
```

Interpolating for PV = $141,700:
```
rate = 3.50% + (142,124 - 141,700) / (142,124 - 141,272) * 0.10%
     = 3.50% + 424/852 * 0.10%
     = 3.50% + 0.050%
     = 3.550%
```

Implied spread: S_uncapped = 3.550% - 3.50% = **5.0bps (0.0005)**

**Step 2: Check cap.**
```
5.0bps <= 35bps  -->  Cap does NOT bind.
```

**Step 3: Results for Scenario 1.**

| Metric | Value |
|--------|-------|
| sba_spread_uncapped | 0.0005 (5.0bps) |
| sba_spread_capped | 0.0005 (5.0bps) |
| sba_BEL_uncapped | $141,700 |
| sba_BEL_capped | $141,700 |
| cap_binding | FALSE |
| bel_impact_of_cap | $0 |

---

#### Scenario 2: Increase Scenario -- Cap DOES Bind

| Input | Value |
|-------|-------|
| Risk-free rate (flat) | 3.50% |
| BEL_uncapped (from goal-seek) | $138,800 |

**Step 1: Derive implied spread.**

Solve for S in: PV(0.035 + S) = $138,800.

```
PV(3.50%) = $142,124
PV(4.00%) = $135,903
```

Interpolating for PV = $138,800:
```
rate = 3.50% + (142,124 - 138,800) / (142,124 - 135,903) * 0.50%
     = 3.50% + 3,324 / 6,221 * 0.50%
     = 3.50% + 0.267%
     = 3.767%
```

Refining:
```
PV(3.77%) ≈ 10,000 * [1 - 1.0377^(-20)] / 0.0377
          ≈ $138,726   (close to $138,800)

PV(3.76%) ≈ $138,845   (slightly above target)
```

Implied spread: S_uncapped = 3.76% - 3.50% = **26bps (0.0026)**

Wait -- 26bps < 35bps, so the cap does NOT bind. We need a scenario where the spread is above 35bps. Let us use a lower BEL_uncapped.

| Input | Value |
|-------|-------|
| Risk-free rate (flat) | 3.50% |
| BEL_uncapped (from goal-seek) | $136,500 |

```
PV(3.50%) = $142,124
PV(4.00%) = $135,903
```

Interpolating for PV = $136,500:
```
rate = 3.50% + (142,124 - 136,500) / (142,124 - 135,903) * 0.50%
     = 3.50% + 5,624 / 6,221 * 0.50%
     = 3.50% + 0.452%
     = 3.952%
```

Implied spread: S_uncapped = 3.952% - 3.50% = **45.2bps (0.00452)**

**Step 2: Check cap.**
```
45.2bps > 35bps  -->  Cap BINDS.
S_capped = 35bps (0.0035)
```

**Step 3: Recalculate BEL at capped rate.**

```
Capped discount rate = 3.50% + 0.35% = 3.85%
BEL_capped = PV(3.85%) = 10,000 * [1 - 1.0385^(-20)] / 0.0385
```

Computing:
```
1.0385^20 = e^(20 * ln(1.0385)) = e^(20 * 0.03777) = e^(0.75548) = 2.1286
1/2.1286 = 0.46981
[1 - 0.46981] / 0.0385 = 0.53019 / 0.0385 = 13.7712

BEL_capped = 10,000 * 13.7712 = $137,712
```

**Step 4: Results for Scenario 2.**

| Metric | Value |
|--------|-------|
| sba_spread_uncapped | 0.00452 (45.2bps) |
| sba_spread_capped | 0.0035 (35.0bps) |
| sba_BEL_uncapped | $136,500 |
| sba_BEL_capped | **$137,712** |
| cap_binding | TRUE |
| bel_impact_of_cap | **+$1,212** |

Note: The capped BEL ($137,712) is **higher** than the uncapped BEL ($136,500). The cap increased BEL by $1,212 because the discount rate was lowered from 3.952% to 3.85%.

---

#### Scenario 3: Twist Scenario -- Cap Does NOT Bind (Largest BEL)

| Input | Value |
|-------|-------|
| Risk-free rate (flat) | 3.50% |
| BEL_uncapped (from goal-seek) | $141,200 |

**Step 1: Derive implied spread.**

```
PV(3.50%) = $142,124
PV(3.60%) = $141,272
```

Interpolating for PV = $141,200:
```
rate = 3.50% + (142,124 - 141,200) / (142,124 - 141,272) * 0.10%
     = 3.50% + 924/852 * 0.10%
```

That ratio > 1, so the rate is above 3.60%. Let us check:
```
PV(3.60%) = $141,272   (still above $141,200)
PV(3.70%) = $140,428
```

Interpolating between 3.60% and 3.70%:
```
rate = 3.60% + (141,272 - 141,200) / (141,272 - 140,428) * 0.10%
     = 3.60% + 72/844 * 0.10%
     = 3.60% + 0.0085%
     = 3.609%
```

Implied spread: S_uncapped = 3.609% - 3.50% = **10.9bps (0.00109)**

**Step 2: Check cap.**
```
10.9bps <= 35bps  -->  Cap does NOT bind.
```

**Step 3: Results for Scenario 3.**

| Metric | Value |
|--------|-------|
| sba_spread_uncapped | 0.00109 (10.9bps) |
| sba_spread_capped | 0.00109 (10.9bps) |
| sba_BEL_uncapped | $141,200 |
| sba_BEL_capped | $141,200 |
| cap_binding | FALSE |
| bel_impact_of_cap | $0 |

---

### 8.3 Most Onerous Scenario Selection

Now we compare the **capped** BELs across all three scenarios:

| Scenario | BEL_uncapped | S_uncapped (bps) | Cap Binds? | BEL_capped |
|----------|-------------|-------------------|------------|------------|
| 1 (Decrease) | $141,700 | 5.0 | No | **$141,700** |
| 2 (Increase) | $136,500 | 45.2 | **Yes** | $137,712 |
| 3 (Twist) | $141,200 | 10.9 | No | $141,200 |

**Most onerous (highest capped BEL): Scenario 1 at $141,700.**

```
BEL_final = $141,700
Biting scenario = Scenario 1 (Decrease)
```

### 8.4 Key Observations from the Worked Example

1. **The cap changed the BEL for Scenario 2** from $136,500 to $137,712 (an increase of $1,212). Without the cap, Scenario 2 would have been the least onerous; with the cap, it remains the least onerous but the gap narrows.

2. **The most onerous scenario was NOT the one where the cap binds.** Scenario 1 has the highest capped BEL even though its spread (5bps) is well below the cap. This illustrates that the cap is a floor on conservatism, not the primary driver of the biting scenario.

3. **If the cap had NOT been enforced** (the Pythora gap), the results would have been identical for Scenarios 1 and 3, but Scenario 2's BEL would have been reported as $136,500 instead of $137,712. In this example the biting scenario selection would not change, but in a real portfolio with tighter margins, the missing cap could cause the wrong scenario to be selected as biting or understate the final BEL.

4. **Both uncapped and capped values are always reported.** This gives auditors visibility into the spread environment and how close each scenario is to the cap threshold.

---

## 9. Edge Cases

### 9.1 Negative Implied Spread

**Situation:** Asset MV + C0 exceeds the undiscounted sum of liability cash flows, or risk-free rates are very high. The implied spread becomes negative.

**Handling:**
```
IF S_uncapped < 0 THEN
    -- Spread is negative: assets more than sufficient at RF rate.
    -- Cap does not apply (negative < 35bps).
    -- Report S_uncapped as-is (negative number).
    -- BEL_capped = BEL_uncapped (no recalculation needed).
    cap_binding = FALSE
END IF
```

This is a legitimate outcome. It means the portfolio is well-funded relative to liabilities even without any spread credit.

### 9.2 Spread Exactly Equals Cap

```
IF S_uncapped == spread_cap THEN
    -- Boundary case: cap does NOT bind (inequality is strict >).
    -- Use ≤ for "no cap" to avoid unnecessary recalculation.
    cap_binding = FALSE
    BEL_capped = BEL_uncapped
END IF
```

Use strict inequality: `cap_binding = (S_uncapped > spread_cap)`.

### 9.3 Goal-Seek Did Not Converge

**Situation:** The projection engine could not find a convergent C0 for a scenario (e.g., assets insufficient even at 100% proportion).

**Handling:**
```
IF scenario_results[s].convergence_status == "not_converged" THEN
    -- Flag scenario as non-convergent.
    -- Still attempt spread derivation using best available C0.
    -- Mark all outputs with convergence_status = "not_converged".
    -- Include in most-onerous selection with a WARNING.
    -- DO NOT silently exclude from BEL determination.
    LOG WARNING: "Scenario {s} did not converge. BEL may be approximate."
END IF
```

**Fail loudly:** Per the architectural constraint, non-convergence should be surfaced prominently, not suppressed.

### 9.4 Spread Root-Finding Fails

**Situation:** `brentq` cannot find a root in the bracket [-500bps, +2000bps].

**Handling:**
```
IF brentq raises ValueError (no sign change in bracket) THEN
    -- The BEL is outside the range of PV achievable with any spread in the bracket.
    -- This indicates a data issue or extreme scenario.
    RAISE SpreadDerivationError(
        scenario=s,
        bel_target=BEL_uncapped,
        pv_at_lower_bound=PV(rf - 0.05),
        pv_at_upper_bound=PV(rf + 0.20),
        message="Cannot derive implied spread. Check liability CFs and asset MV."
    )
END IF
```

### 9.5 Zero or Near-Zero Liability Cash Flows

**Situation:** All liability cash flows are zero (empty block) or near-zero.

**Handling:**
```
IF sum(liability_cfs["cashflow"]) < EPSILON THEN
    -- No meaningful liabilities to discount.
    -- BEL = 0, spread is undefined.
    -- Skip spread derivation; report BEL = 0 for this block.
    S_uncapped = NaN
    S_capped = NaN
    BEL_uncapped = 0
    BEL_capped = 0
END IF
```

### 9.6 All 9 Scenarios Have Cap Binding

**Situation:** Every scenario produces a spread above 35bps. All BELs are recalculated at the capped rate.

**Handling:** This is normal operation -- no special logic needed. The most onerous is still the highest capped BEL. However, this situation should be flagged in the audit report as it may indicate the portfolio has unusually high credit exposure or the asset-liability match is poor.

```
IF all(cap_binding[s] for s in 1..9) THEN
    LOG INFO: "Spread cap binds in ALL 9 scenarios. "
              "Uncapped spreads range from {min}bps to {max}bps. "
              "All BELs recalculated at 35bps cap."
END IF
```

### 9.7 Spread Cap Value Change (Regulatory Update)

The 35bps value is the current BMA requirement. The implementation must:
- Store as a named constant: `DEFAULT_SPREAD_CAP_BPS = 35`
- Accept an override via `RunConfig` for sensitivity analysis.
- Log the spread cap value used in every run for audit traceability.
- Validate that the override is reasonable (0bps to 500bps range).

---

## 10. Implementation Notes

### 10.1 Lessons from Pythora's Failure

The Pythora reference model is the most important cautionary tale for this module. Their failure was not a design error -- it was a gap between design and implementation:

1. **The parameter existed.** `sba_spread_cap` was defined in `settings_loader.py` (line 76) with a default value. The configuration infrastructure was built correctly.

2. **The parameter was loaded.** It was read from the Excel setup file and stored in the `Settings` dataclass. No loading error.

3. **The parameter was never used.** No code path in `alm_projection.py`, `main_client.py`, or any other module ever compared the derived spread against `sba_spread_cap`. The goal-seek derived a spread, and that uncapped spread flowed directly into BEL reporting.

4. **The BEL was never recalculated.** Even if the comparison had been made, there was no `recalculate_bel_at_capped_spread()` function. The recalculation logic simply did not exist.

5. **The output fields were present.** The Excel writer had columns for both `sba_spread_capped` and `sba_BEL_capped`, but they were always populated with the uncapped values -- giving the false appearance of enforcement.

### 10.2 Our Enforcement Strategy

To prevent a similar gap, the implementation must include:

**A. Explicit enforcement function.** The cap check and BEL recalculation must be a single, named function (`enforce_spread_cap`) that is impossible to skip:

```python
@rule_ref("E9", "35bps spread cap enforcement")
def enforce_spread_cap(
    scenario_id: int,
    bel_uncapped: float,
    liability_cfs: pd.DataFrame,
    rf_curve: YieldCurve,
    spread_cap: float = 0.0035
) -> SpreadCapResult:
    """
    MANDATORY spread cap enforcement. Called for every scenario.
    Returns both uncapped and capped values.
    """
    uncapped_spread = derive_implied_spread(bel_uncapped, liability_cfs, rf_curve)

    if uncapped_spread > spread_cap:
        capped_bel = compute_pv(liability_cfs, rf_curve, spread=spread_cap)
        return SpreadCapResult(
            spread_uncapped=uncapped_spread,
            spread_capped=spread_cap,
            bel_uncapped=bel_uncapped,
            bel_capped=capped_bel,
            cap_binding=True
        )
    else:
        return SpreadCapResult(
            spread_uncapped=uncapped_spread,
            spread_capped=uncapped_spread,
            bel_uncapped=bel_uncapped,
            bel_capped=bel_uncapped,
            cap_binding=False
        )
```

**B. Integration test.** A dedicated test that:
- Constructs a scenario with a known spread > 35bps.
- Verifies that `bel_capped > bel_uncapped`.
- Verifies that `spread_capped == 0.0035`.
- Verifies that the BEL recalculation matches an independent PV calculation.

```python
def test_spread_cap_binds():
    """Golden test: cap must increase BEL when spread > 35bps."""
    result = enforce_spread_cap(
        scenario_id=2,
        bel_uncapped=136_500,
        liability_cfs=ANNUITY_10K_20Y,
        rf_curve=FLAT_CURVE_350BPS,
        spread_cap=0.0035
    )
    assert result.cap_binding is True
    assert result.spread_capped == 0.0035
    assert result.bel_capped > result.bel_uncapped
    assert result.bel_capped == pytest.approx(137_712, rel=0.01)

def test_spread_cap_does_not_bind():
    """Golden test: cap has no effect when spread <= 35bps."""
    result = enforce_spread_cap(
        scenario_id=1,
        bel_uncapped=141_700,
        liability_cfs=ANNUITY_10K_20Y,
        rf_curve=FLAT_CURVE_350BPS,
        spread_cap=0.0035
    )
    assert result.cap_binding is False
    assert result.bel_capped == result.bel_uncapped
```

**C. Assertion guard in the BEL pipeline.** Add an assertion in the main BEL calculation that the spread cap has been applied:

```python
def calculate_final_bel(scenario_metrics: dict) -> float:
    for s, metrics in scenario_metrics.items():
        # Guard: spread cap must have been evaluated
        assert metrics.sba_spread_capped is not None, (
            f"Scenario {s}: spread cap was not applied. "
            f"This is a critical enforcement gap."
        )
        assert metrics.sba_spread_capped <= SPREAD_CAP + 1e-10, (
            f"Scenario {s}: capped spread {metrics.sba_spread_capped} "
            f"exceeds cap {SPREAD_CAP}. Enforcement failed."
        )
    return max(m.sba_BEL_capped for m in scenario_metrics.values())
```

**D. Output validation.** The Excel writer must verify that `sba_BEL_capped != sba_BEL_uncapped` for at least one scenario where `cap_binding == True`. If `cap_binding` is True but the two BEL values are equal, raise an error -- this would indicate the Pythora-style bug of populating capped fields with uncapped values.

### 10.3 Data Flow Diagram

```
projection/                    calculations/bel_calculator.py
  goal_seek()                    enforce_spread_cap()
  per scenario  ──────────────>  per scenario
    C0[s]                          derive_implied_spread()
    asset_proportion[s]              └─ brentq root-finding
                                   check cap (> 35bps?)
curves/                            if yes: recalculate BEL
  rf_curves[s]  ──────────────>      compute_pv(liab_cfs, rf + 35bps)
                                   store both uncapped & capped
model_points/
  liability_cfs ──────────────>  select_biting_scenario()
  asset_mv_t0                      max(BEL_capped[s]) across all s
                                   └─ BEL_final
                                 ──────────────> output/
                                                   Excel workbook
                                                   Audit trail
```

### 10.4 Performance Considerations

- The spread derivation (`brentq`) requires evaluating `compute_pv` approximately 10-20 times per scenario. With 9 scenarios, that is ~90-180 PV evaluations.
- Each PV evaluation is a dot product of cash flows and discount factors (~100 elements for a 100-year projection). This is trivially fast (microseconds).
- The capped BEL recalculation (when cap binds) is a single additional PV evaluation.
- Total computational cost of this module: negligible relative to the projection engine.

### 10.5 Pydantic Validation

```python
from pydantic import BaseModel, Field, validator

class SpreadCapConfig(BaseModel):
    spread_cap_bps: float = Field(default=35.0, ge=0.0, le=500.0)
    enforce_cap: bool = Field(default=True)

    @property
    def spread_cap_decimal(self) -> float:
        return self.spread_cap_bps / 10_000

    @validator("enforce_cap")
    def warn_if_disabled(cls, v):
        if not v:
            import warnings
            warnings.warn(
                "Spread cap enforcement is DISABLED. "
                "This is only valid for sensitivity analysis.",
                UserWarning
            )
        return v

class SpreadCapResult(BaseModel):
    scenario_id: int
    scenario_name: str
    spread_uncapped: float
    spread_capped: float
    bel_uncapped: float
    bel_capped: float
    cap_binding: bool
    bel_impact_of_cap: float = 0.0
    avg_dis_rate_uncapped: float
    avg_dis_rate_capped: float
    convergence_status: str

    @validator("bel_impact_of_cap", always=True)
    def compute_impact(cls, v, values):
        return values.get("bel_capped", 0) - values.get("bel_uncapped", 0)

    @validator("bel_capped")
    def capped_ge_uncapped_when_binding(cls, v, values):
        if values.get("cap_binding") and v < values.get("bel_uncapped", 0):
            raise ValueError(
                "When cap binds, capped BEL must be >= uncapped BEL. "
                f"Got capped={v}, uncapped={values.get('bel_uncapped')}"
            )
        return v
```

### 10.6 @rule_ref Decorator Usage

Every function in this module must carry a `@rule_ref` tag:

| Function | Rule Reference | Description |
|----------|---------------|-------------|
| `derive_implied_spread` | E9 | Spread derivation from RF curve |
| `enforce_spread_cap` | E9 | 35bps cap enforcement |
| `compute_pv` | E9 | PV calculation using RF + spread |
| `select_biting_scenario` | Para 28(10) | Most onerous scenario selection |
| `calculate_final_bel` | Para 28(10)(a-c) | Final BEL determination |

---

## Appendix A: Summary of Formulas

| Formula | Expression |
|---------|------------|
| PV of liabilities | `PV(S) = SUM[ CF(t) / PROD(1 + rf(k) + S, k=1..t) ]` for t=1..T |
| Implied spread | `S* such that PV(S*) = BEL_uncapped` |
| Capped spread | `S_cap = min(S*, 0.0035)` |
| Capped BEL | `BEL_capped = PV(S_cap)` when cap binds; else `BEL_uncapped` |
| BEL final | `BEL_final = max(BEL_capped[s])` across s = 1..9 |
| BEL (full) | `BEL = MV_assets(T=0) + C0_biting` |
| Technical Provision | `TP = BEL + Risk_Margin` |
| Cap impact | `Delta = BEL_capped - BEL_uncapped` (>= 0 when cap binds) |

## Appendix B: Scenario Quick Reference

| ID | Name | Rate Shock |
|----|------|-----------|
| 0 | SBA_0_Base | No adjustment |
| 1 | SBA_1_Decrease | Rates decrease to -1.5% by year 10 |
| 2 | SBA_2_Increase | Rates increase |
| 3 | SBA_3_DownUp | Down then up |
| 4 | SBA_4_UpDown | Up then down |
| 5 | SBA_5_Twist1 | Twist variant 1 |
| 6 | SBA_6_Twist2 | Twist variant 2 |
| 7 | SBA_7_Twist3 | Twist variant 3 |
| 8 | SBA_8_Twist4 | Twist variant 4 |
