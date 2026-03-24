# Algorithm Specification: Yield Curve Construction

## Document Control

| Field | Value |
|---|---|
| **Spec ID** | ALGO-001 |
| **Target Modules** | `curves/curve_builder.py`, `curves/ql_audit.py` |
| **Supporting Modules** | `curves/scenario_curves.py`, `rules/v2024/scenarios.py` |
| **Status** | Implementation-Ready Draft |
| **Date** | March 2026 |
| **BMA Rules** | Schedule XXV Para 28(7)(a-i); Handbook E9, pp. 330-331 |

---

## 1. Overview

### 1.1 Purpose

This module builds the risk-free yield curves used throughout the BMA SBA projection engine. It is responsible for:

1. **Constructing the base risk-free curve** from BMA-published or market spot rates at T=0 (Handbook E9).
2. **Generating 9 scenario-shifted curves** for each projection year `t = 0..100` (Rules Para 28(7)(a-i)).
3. **Overlaying z-spread adjustments** for asset-specific discounting (Handbook E9; PHASE1-003 spec).
4. **Wrapping all QuantLib calls** through `ql_audit.py` for full auditability.

### 1.2 Architectural Boundary

Per the project's architectural constraint, **only the `curves/` package imports QuantLib**. All downstream consumers (projection, calculations, engine) receive plain Python objects: NumPy arrays of discount factors, or callables that return rates for a given tenor. The `curves/` package is the QuantLib firewall.

```
assumptions/            curves/                   projection/
  economic.py  ------>    curve_builder.py  ------>  (pure Python)
  (spot rates CSV)        scenario_curves.py        asset_cf.py
                          ql_audit.py               liability_cf.py
                          bond_pricing.py            ...
                    ONLY MODULE THAT
                    IMPORTS QUANTLIB
```

### 1.3 Key Rule References

| Rule | Description | Used In |
|---|---|---|
| Rules Sched XXV Para 28(7)(a) | Base scenario: no adjustment | Section 3.1 |
| Rules Sched XXV Para 28(7)(b) | Decrease scenario: -150 bps | Section 3.2 |
| Rules Sched XXV Para 28(7)(c) | Increase scenario: +150 bps | Section 3.2 |
| Rules Sched XXV Para 28(7)(d) | Down-Up hump scenario | Section 3.3 |
| Rules Sched XXV Para 28(7)(e) | Up-Down hump scenario | Section 3.3 |
| Rules Sched XXV Para 28(7)(f) | Decrease + Positive Twist | Section 3.4 |
| Rules Sched XXV Para 28(7)(g) | Decrease + Negative Twist | Section 3.4 |
| Rules Sched XXV Para 28(7)(h) | Increase + Positive Twist | Section 3.4 |
| Rules Sched XXV Para 28(7)(i) | Increase + Negative Twist | Section 3.4 |
| Handbook E9, pp. 330-331 | Risk-free curve specification | Section 2 |
| Rules Sched XXV Para 28(9) | Cash flow matching/projection | Section 4 (z-spread) |

---

## 2. Base Risk-Free Curve Construction

### 2.1 Input Data

The base risk-free curve is loaded from `assumptions/tables/bma_rfr_{YYYY}Q{Q}.csv`. Each row is a `(tenor_years, spot_rate)` pair.

**Example input** (annual spot rates, continuously compounded):

| Tenor (years) | Spot Rate |
|---|---|
| 0.25 | 0.0420 |
| 0.50 | 0.0415 |
| 1 | 0.0400 |
| 2 | 0.0390 |
| 3 | 0.0385 |
| 5 | 0.0380 |
| 7 | 0.0378 |
| 10 | 0.0375 |
| 15 | 0.0372 |
| 20 | 0.0370 |
| 30 | 0.0368 |
| 40 | 0.0367 |
| 50 | 0.0366 |

**Rule reference:** Handbook E9 -- "Insurers must use either the BMA published curve or a relevant market curve (e.g., Swap/Govt) without adjustments."

### 2.2 QuantLib Class Selection

**Selected class:** `ql.ZeroCurve`

**Rationale:**

| Alternative | Why Not |
|---|---|
| `ql.PiecewiseYieldCurve` | Requires market instruments (bonds, deposits, swaps) as input. We have published spot rates, not instruments. Over-complex for our use case. |
| `ql.ForwardCurve` | Forward rates are less intuitive and harder to audit. We start from spot rates. |
| `ql.FlatForward` | Single rate only; cannot represent a term structure. |
| **`ql.ZeroCurve`** | **Takes spot rates directly. Perfect for BMA-published curves. Supports interpolation and extrapolation.** |

### 2.3 Interpolation and Extrapolation

**Interpolation:** `ql.Linear` (linear interpolation on zero rates)

**Rationale:** Linear interpolation on zero rates is:
- Transparent and auditable (actuaries can replicate by hand).
- Consistent with BMA's expectation of simple, verifiable methods.
- Produces smooth forward rates (no discontinuities that would affect bond pricing).

**Extrapolation:** Enabled via `curve.enableExtrapolation()` -- flat extrapolation beyond the last tenor. Required for liabilities extending past 50 years.

**Alternative considered:** `ql.LogLinear` (log-linear on discount factors). Rejected because it is less intuitive to verify and the difference is immaterial for typical rate levels.

### 2.4 Day Count Convention and Calendar

| Parameter | Value | Rationale |
|---|---|---|
| Day count | `ql.Actual365Fixed()` | Standard for regulatory curves; consistent with BMA published rates |
| Calendar | `ql.UnitedStates(ql.UnitedStates.GovernmentBond)` | US Govt bond calendar for settlement; covers BMD-linked instruments |
| Settlement days | 2 (T+2) | Standard bond market settlement |
| Compounding | `ql.Continuous` | Internal representation; converted at boundaries as needed |
| Frequency | `ql.Annual` | BMA projection is annual |

### 2.5 Construction Algorithm

```python
# --- curves/curve_builder.py ---

import QuantLib as ql
import numpy as np
from datetime import date
from typing import List, Tuple
from .ql_audit import ql_audit_log, ql_call

@rule_ref("Handbook E9", "Base risk-free curve construction")
def build_base_curve(
    valuation_date: date,
    spot_rates: List[Tuple[float, float]],  # [(tenor_years, rate), ...]
) -> ql.ZeroCurve:
    """
    Build the base risk-free yield curve from BMA-published spot rates.

    Parameters
    ----------
    valuation_date : date
        T=0 valuation date (e.g., 2024-12-31).
    spot_rates : list of (tenor_years, rate)
        Annual spot rates, continuously compounded.
        Example: [(1, 0.04), (2, 0.039), (5, 0.038), ...]

    Returns
    -------
    ql.ZeroCurve
        QuantLib yield curve handle with extrapolation enabled.
    """
    # Step 1: Set QuantLib evaluation date
    ql_eval_date = ql_call(
        "ql.Date",
        lambda: ql.Date(valuation_date.day, valuation_date.month, valuation_date.year),
        context="Set evaluation date"
    )
    ql.Settings.instance().evaluationDate = ql_eval_date

    # Step 2: Convert tenors to QuantLib dates
    calendar = ql.UnitedStates(ql.UnitedStates.GovernmentBond)
    settlement_date = calendar.advance(ql_eval_date, ql.Period(2, ql.Days))

    dates = [settlement_date]
    rates = [spot_rates[0][1]]  # Rate at settlement = short-end rate

    for tenor_years, rate in spot_rates:
        dt = calendar.advance(
            settlement_date,
            ql.Period(int(tenor_years * 12), ql.Months)
        )
        dates.append(dt)
        rates.append(rate)

    # Step 3: Build the ZeroCurve
    day_count = ql.Actual365Fixed()

    curve = ql_call(
        "ql.ZeroCurve",
        lambda: ql.ZeroCurve(dates, rates, day_count, calendar),
        context=f"Build base RF curve, {len(spot_rates)} tenors, "
                f"val_date={valuation_date}"
    )

    # Step 4: Enable extrapolation for long-dated liabilities
    curve.enableExtrapolation()

    # Step 5: Audit -- log the curve for reproducibility
    ql_audit_log(
        operation="build_base_curve",
        inputs={
            "valuation_date": str(valuation_date),
            "num_tenors": len(spot_rates),
            "shortest_tenor": spot_rates[0][0],
            "longest_tenor": spot_rates[-1][0],
            "short_rate": spot_rates[0][1],
            "long_rate": spot_rates[-1][1],
        },
        outputs={
            "curve_type": "ZeroCurve",
            "interpolation": "Linear",
            "extrapolation": True,
            "day_count": "Actual365Fixed",
        }
    )

    return curve
```

### 2.6 Reading Rates Back from the Curve

```python
@rule_ref("Handbook E9", "Extract rates from constructed curve")
def read_zero_rate(curve: ql.ZeroCurve, tenor_years: float) -> float:
    """
    Read back a zero rate from the curve at a given tenor.

    Parameters
    ----------
    curve : ql.ZeroCurve
    tenor_years : float

    Returns
    -------
    float
        Continuously compounded zero rate.
    """
    day_count = ql.Actual365Fixed()
    dt = curve.referenceDate() + ql.Period(
        int(tenor_years * 365.25), ql.Days
    )
    rate = curve.zeroRate(dt, day_count, ql.Continuous).rate()
    return rate


def read_discount_factor(curve: ql.ZeroCurve, tenor_years: float) -> float:
    """Read discount factor at given tenor."""
    dt = curve.referenceDate() + ql.Period(
        int(tenor_years * 365.25), ql.Days
    )
    return curve.discount(dt)
```

### 2.7 Exporting to Pure-Python Objects

To enforce the QuantLib boundary, curves are exported as lookup-friendly Python objects before being passed to `projection/`:

```python
@rule_ref("Handbook E9", "Export curve for projection engine")
def export_curve_to_array(
    curve: ql.ZeroCurve,
    tenors: np.ndarray,  # e.g., np.arange(0.5, 101, 0.5)
) -> dict:
    """
    Export the QuantLib curve as a dict of NumPy arrays.

    Returns
    -------
    dict with keys:
        "tenors": np.ndarray of tenor points (years)
        "zero_rates": np.ndarray of zero rates (continuous)
        "discount_factors": np.ndarray of discount factors
    """
    zero_rates = np.array([read_zero_rate(curve, t) for t in tenors])
    discount_factors = np.array([read_discount_factor(curve, t) for t in tenors])

    return {
        "tenors": tenors,
        "zero_rates": zero_rates,
        "discount_factors": discount_factors,
    }
```

---

## 3. The 9 Scenario Curve Generation

### 3.1 Scenario 1: Base (No Shift)

**Rule reference:** Rules Sched XXV Para 28(7)(a)

No modification to the base curve. The base curve from Section 2 is used directly.

```python
# Scenario 1: Base
# shift(year, tenor) = 0 for all year, tenor
```

### 3.2 Parallel Scenarios (2: Decrease, 3: Increase)

**Rule reference:** Rules Sched XXV Para 28(7)(b-c)

These scenarios apply a uniform shift across all tenors. The shift grades in linearly over the first 10 projection years, then remains constant.

**Formula:**

```
shift(year) = max_shift * min(year / 10, 1.0)
```

Where:
- `year` = projection year (0, 1, 2, ..., 100)
- `max_shift` = -0.0150 for Decrease, +0.0150 for Increase (in decimal)

**Temporal profile:**

| Projection Year | Grading Factor | Decrease Shift (bps) | Increase Shift (bps) |
|---|---|---|---|
| 0 | 0.0 | 0 | 0 |
| 1 | 0.1 | -15 | +15 |
| 2 | 0.2 | -30 | +30 |
| 3 | 0.3 | -45 | +45 |
| 5 | 0.5 | -75 | +75 |
| 7 | 0.7 | -105 | +105 |
| 10 | 1.0 | -150 | +150 |
| 15 | 1.0 | -150 | +150 |
| 50 | 1.0 | -150 | +150 |
| 100 | 1.0 | -150 | +150 |

**Implementation:**

```python
@rule_ref("Rules Sched XXV Para 28(7)(b-c)", "Parallel scenario shift")
def parallel_shift(year: int, max_shift_bps: float) -> float:
    """
    Compute parallel scenario shift for a given projection year.

    Parameters
    ----------
    year : int
        Projection year (0-100).
    max_shift_bps : float
        Maximum shift in basis points (-150 or +150).

    Returns
    -------
    float
        Shift in decimal (e.g., -0.0150 for -150 bps).
    """
    max_shift = max_shift_bps / 10_000  # bps to decimal
    grading = min(year / 10.0, 1.0)
    return max_shift * grading
```

The shifted rate at each tenor is:

```
shifted_rate(year, tenor) = base_rate(tenor) + shift(year)
```

### 3.3 Hump Scenarios (4: Down-Up, 5: Up-Down)

**Rule reference:** Rules Sched XXV Para 28(7)(d-e)

These scenarios apply a hump-shaped temporal pattern: the shift ramps up to peak by year 5, then returns to zero by year 10. The shift is uniform across all tenors (parallel within each projection year).

**Formula:**

```
shift(year) =
    if year <= 5:   max_shift * (year / 5)
    if year <= 10:  max_shift * (10 - year) / 5
    else:           0
```

Where:
- `max_shift` = -0.0150 for Down-Up, +0.0150 for Up-Down

**Temporal profile:**

| Projection Year | Grading Factor | Down-Up Shift (bps) | Up-Down Shift (bps) |
|---|---|---|---|
| 0 | 0.0 | 0 | 0 |
| 1 | 0.2 | -30 | +30 |
| 2 | 0.4 | -60 | +60 |
| 3 | 0.6 | -90 | +90 |
| 5 | 1.0 | -150 | +150 |
| 7 | 0.6 | -90 | +90 |
| 10 | 0.0 | 0 | 0 |
| 15 | 0.0 | 0 | 0 |
| 50 | 0.0 | 0 | 0 |

**Implementation:**

```python
@rule_ref("Rules Sched XXV Para 28(7)(d-e)", "Hump scenario shift")
def hump_shift(year: int, max_shift_bps: float) -> float:
    """
    Compute hump (down-up or up-down) scenario shift.

    Parameters
    ----------
    year : int
        Projection year (0-100).
    max_shift_bps : float
        Peak shift in basis points (-150 or +150).

    Returns
    -------
    float
        Shift in decimal.
    """
    max_shift = max_shift_bps / 10_000
    if year <= 5:
        return max_shift * (year / 5.0)
    elif year <= 10:
        return max_shift * (10.0 - year) / 5.0
    else:
        return 0.0
```

The shifted rate:

```
shifted_rate(year, tenor) = base_rate(tenor) + shift(year)
```

### 3.4 Twist Scenarios (6-9)

**Rule reference:** Rules Sched XXV Para 28(7)(f-i)

Twist scenarios apply different shifts at the short and long ends of the yield curve, with linear interpolation across the tenor spectrum. The temporal grading is the same as parallel scenarios (linear to year 10, flat after).

**Formula:**

```
tenor_shift(tenor) = short_shift + (long_shift - short_shift) * (tenor / max_tenor)

shift(year, tenor) = tenor_shift(tenor) * min(year / 10.0, 1.0)
```

Where:
- `tenor` = bond/rate tenor in years
- `max_tenor` = 20 years (the "long end" anchor; see Tech Spec 1.3)
- `short_shift`, `long_shift` = scenario-specific shifts (see table below)
- Tenors below 1 year get the `short_shift`; tenors above 20 years get the `long_shift`

**Twist parameters:**

| Scenario | ID | Short Shift (bps) | Long Shift (bps) | Description |
|---|---|---|---|---|
| Decrease + Positive Twist | 6 | -150 | -50 | Short end falls more; curve steepens |
| Decrease + Negative Twist | 7 | -50 | -150 | Long end falls more; curve flattens |
| Increase + Positive Twist | 8 | +50 | +150 | Long end rises more; curve steepens |
| Increase + Negative Twist | 9 | +150 | +50 | Short end rises more; curve flattens |

**Tenor interpolation detail:**

For Scenario 6 (Decrease + Positive Twist) at full grading (year >= 10):

| Tenor (years) | Shift Calculation | Shift (bps) |
|---|---|---|
| 0.25 | Clamped to short_shift | -150 |
| 1 | -150 + (-50 - (-150)) * (1/20) | -145 |
| 2 | -150 + 100 * (2/20) | -140 |
| 5 | -150 + 100 * (5/20) | -125 |
| 10 | -150 + 100 * (10/20) | -100 |
| 15 | -150 + 100 * (15/20) | -75 |
| 20 | -150 + 100 * (20/20) | -50 |
| 30 | Clamped to long_shift | -50 |
| 50 | Clamped to long_shift | -50 |

**Implementation:**

```python
# Tenor boundaries from Tech Spec 1.3
SHORT_END_TENOR = 1.0   # years: tenors <= 1y get short_shift
LONG_END_TENOR = 20.0   # years: tenors >= 20y get long_shift

@rule_ref("Rules Sched XXV Para 28(7)(f-i)", "Twist scenario shift")
def twist_shift(
    year: int,
    tenor: float,
    short_shift_bps: float,
    long_shift_bps: float,
) -> float:
    """
    Compute twist scenario shift for a given projection year and tenor.

    Parameters
    ----------
    year : int
        Projection year (0-100).
    tenor : float
        Rate/bond tenor in years.
    short_shift_bps : float
        Shift at short end in bps (e.g., -150).
    long_shift_bps : float
        Shift at long end in bps (e.g., -50).

    Returns
    -------
    float
        Shift in decimal.
    """
    short_shift = short_shift_bps / 10_000
    long_shift = long_shift_bps / 10_000

    # Tenor interpolation: clamp to [SHORT_END_TENOR, LONG_END_TENOR]
    clamped_tenor = max(SHORT_END_TENOR, min(tenor, LONG_END_TENOR))

    # Linear interpolation across the tenor spectrum
    tenor_fraction = (clamped_tenor - SHORT_END_TENOR) / (LONG_END_TENOR - SHORT_END_TENOR)
    tenor_shift = short_shift + (long_shift - short_shift) * tenor_fraction

    # Temporal grading (same as parallel: linear to year 10, flat after)
    temporal_grading = min(year / 10.0, 1.0)

    return tenor_shift * temporal_grading
```

### 3.5 Scenario Definition Registry

All 9 scenarios are defined in `rules/v2024/scenarios.py` and consumed by `curves/scenario_curves.py`:

```python
# --- rules/v2024/scenarios.py ---

from dataclasses import dataclass
from enum import IntEnum
from typing import Optional

class ScenarioType(IntEnum):
    BASE = 1
    PARALLEL = 2
    HUMP = 3
    TWIST = 4

@dataclass(frozen=True)
class ScenarioDefinition:
    id: int
    name: str
    scenario_type: ScenarioType
    short_shift_bps: float = 0.0
    long_shift_bps: float = 0.0
    rule_ref: str = ""

# BMA 9 scenarios -- Rules Sched XXV Para 28(7)(a-i)
BMA_SCENARIOS = {
    1: ScenarioDefinition(1, "Base",                    ScenarioType.BASE,     0,    0,    "Para 28(7)(a)"),
    2: ScenarioDefinition(2, "Decrease",                ScenarioType.PARALLEL, -150, -150, "Para 28(7)(b)"),
    3: ScenarioDefinition(3, "Increase",                ScenarioType.PARALLEL, +150, +150, "Para 28(7)(c)"),
    4: ScenarioDefinition(4, "Down-Up",                 ScenarioType.HUMP,     -150, -150, "Para 28(7)(d)"),
    5: ScenarioDefinition(5, "Up-Down",                 ScenarioType.HUMP,     +150, +150, "Para 28(7)(e)"),
    6: ScenarioDefinition(6, "Decrease+PositiveTwist",  ScenarioType.TWIST,    -150, -50,  "Para 28(7)(f)"),
    7: ScenarioDefinition(7, "Decrease+NegativeTwist",  ScenarioType.TWIST,    -50,  -150, "Para 28(7)(g)"),
    8: ScenarioDefinition(8, "Increase+PositiveTwist",  ScenarioType.TWIST,    +50,  +150, "Para 28(7)(h)"),
    9: ScenarioDefinition(9, "Increase+NegativeTwist",  ScenarioType.TWIST,    +150, +50,  "Para 28(7)(i)"),
}
```

### 3.6 Building a Scenario Curve for a Given Projection Year

```python
# --- curves/scenario_curves.py ---

@rule_ref("Rules Sched XXV Para 28(7)", "Apply scenario shift to base curve")
def build_scenario_curve(
    base_curve: ql.ZeroCurve,
    scenario: ScenarioDefinition,
    projection_year: int,
    valuation_date: date,
) -> ql.ZeroCurve:
    """
    Build a shifted yield curve for a specific scenario and projection year.

    The base curve rates are read at each standard tenor, shifted per the
    scenario formula, and a new ZeroCurve is constructed.

    Parameters
    ----------
    base_curve : ql.ZeroCurve
        The T=0 base risk-free curve.
    scenario : ScenarioDefinition
        One of the 9 BMA scenarios.
    projection_year : int
        The projection year t (0-100).
    valuation_date : date
        The original T=0 date (used to set QuantLib dates).

    Returns
    -------
    ql.ZeroCurve
        A new curve with scenario shifts applied.
    """
    # Standard tenor grid
    TENORS = [0.25, 0.5, 1, 2, 3, 5, 7, 10, 15, 20, 30, 40, 50]

    day_count = ql.Actual365Fixed()
    calendar = ql.UnitedStates(ql.UnitedStates.GovernmentBond)

    # Advance the reference date by projection_year
    ql_val_date = ql.Date(valuation_date.day, valuation_date.month, valuation_date.year)
    ref_date = calendar.advance(ql_val_date, ql.Period(projection_year, ql.Years))
    settlement_date = calendar.advance(ref_date, ql.Period(2, ql.Days))

    dates = [settlement_date]
    shifted_rates = []

    # First rate at settlement
    base_rate_0 = base_curve.zeroRate(
        base_curve.referenceDate() + ql.Period(1, ql.Days),
        day_count, ql.Continuous
    ).rate()
    shift_0 = _compute_shift(scenario, projection_year, tenor=0.003)
    shifted_rates.append(max(base_rate_0 + shift_0, 0.0))

    for tenor in TENORS:
        # Read base rate at this tenor
        base_dt = base_curve.referenceDate() + ql.Period(
            int(tenor * 365.25), ql.Days
        )
        base_rate = base_curve.zeroRate(
            base_dt, day_count, ql.Continuous
        ).rate()

        # Compute shift
        shift = _compute_shift(scenario, projection_year, tenor)

        # Apply shift with floor at 0
        shifted_rate = max(base_rate + shift, 0.0)

        # Build date
        dt = calendar.advance(settlement_date, ql.Period(int(tenor * 12), ql.Months))
        dates.append(dt)
        shifted_rates.append(shifted_rate)

    # Construct new ZeroCurve
    new_curve = ql_call(
        "ql.ZeroCurve",
        lambda: ql.ZeroCurve(dates, shifted_rates, day_count, calendar),
        context=f"Scenario {scenario.id} ({scenario.name}), year {projection_year}"
    )
    new_curve.enableExtrapolation()

    # Audit log
    ql_audit_log(
        operation="build_scenario_curve",
        inputs={
            "scenario_id": scenario.id,
            "scenario_name": scenario.name,
            "projection_year": projection_year,
            "rule_ref": scenario.rule_ref,
        },
        outputs={
            "rate_at_1y": shifted_rates[3] if len(shifted_rates) > 3 else None,
            "rate_at_10y": shifted_rates[8] if len(shifted_rates) > 8 else None,
            "rate_at_30y": shifted_rates[11] if len(shifted_rates) > 11 else None,
            "min_rate": min(shifted_rates),
        }
    )

    return new_curve


def _compute_shift(
    scenario: ScenarioDefinition,
    year: int,
    tenor: float,
) -> float:
    """
    Internal dispatcher: compute the shift for a scenario/year/tenor combination.
    """
    if scenario.scenario_type == ScenarioType.BASE:
        return 0.0

    elif scenario.scenario_type == ScenarioType.PARALLEL:
        return parallel_shift(year, scenario.short_shift_bps)

    elif scenario.scenario_type == ScenarioType.HUMP:
        return hump_shift(year, scenario.short_shift_bps)

    elif scenario.scenario_type == ScenarioType.TWIST:
        return twist_shift(year, tenor, scenario.short_shift_bps, scenario.long_shift_bps)

    else:
        raise ValueError(f"Unknown scenario type: {scenario.scenario_type}")
```

### 3.7 Rate Floor

**Rule reference:** Tech Spec 1.3 -- "Minimum rate floor: 0 bps (rates cannot go negative, per BMA guidance) -- CONFIRM with BMA"

All shifted rates are floored at zero:

```python
shifted_rate = max(base_rate + shift, 0.0)
```

**Note:** This is a conservative implementation pending BMA confirmation. If the BMA allows negative rates, the floor should be removed or set to a configurable parameter.

### 3.8 Pre-computing All Scenario Curves

For efficiency, all curves for all 9 scenarios across all 101 projection years are pre-computed at run start:

```python
@rule_ref("Rules Sched XXV Para 28(7)", "Pre-compute all scenario curves")
def precompute_scenario_curves(
    base_curve: ql.ZeroCurve,
    valuation_date: date,
    max_projection_year: int = 100,
) -> dict:
    """
    Pre-compute scenario curves for all 9 scenarios and all projection years.

    Returns
    -------
    dict
        {(scenario_id, projection_year): exported_curve_dict}
        Each exported_curve_dict has keys: "tenors", "zero_rates", "discount_factors"
    """
    EXPORT_TENORS = np.arange(0.5, 51, 0.5)  # 0.5, 1.0, ..., 50.0

    result = {}
    for scenario_id, scenario in BMA_SCENARIOS.items():
        for year in range(0, max_projection_year + 1):
            ql_curve = build_scenario_curve(
                base_curve, scenario, year, valuation_date
            )
            exported = export_curve_to_array(ql_curve, EXPORT_TENORS)
            result[(scenario_id, year)] = exported

    return result
```

**Memory estimate:** 9 scenarios x 101 years x 100 tenors x 2 arrays (rates + DFs) x 8 bytes = ~14.5 MB. Acceptable for a desktop application.

---

## 4. Z-Spread Overlay

### 4.1 Purpose

The z-spread (zero-volatility spread) is added to the risk-free curve to create asset-specific discount curves. This is used for:

1. **Calibrating asset market values at T=0** -- finding the spread that reprices the bond to its observed market value.
2. **Projecting liability accumulation rates** -- the liability PV grows at (risk-free + spread) each period.

**Rule reference:** Handbook E9 -- "Spreads must be consistent with the chosen curve"; Rules Sched XXV Para 28(9).

### 4.2 Z-Spread Calibration Methods

Three methods for determining the T=0 z-spread on individual assets (from PHASE1-003):

| Method | When Used | Algorithm |
|---|---|---|
| **CalibrateToMarketPrice** | Bonds with observable market prices | Goal-seek spread `S` such that `PV(CFs, RF + S) = MV`. Uses QuantLib `CashFlows.zSpread()`. |
| **UseRateInAssetFile** | Pre-calculated spread provided in data | Read `z_spread` column directly from asset file. |
| **UseRateInThisTable** | No market price available (cash, derivatives) | Use spread from assumption table by model type. |

### 4.3 Z-Spread Calibration via QuantLib

```python
# --- curves/curve_builder.py ---

@rule_ref("Handbook E9", "Z-spread calibration to market price")
def calibrate_zspread(
    bond: ql.FixedRateBond,
    market_price: float,
    base_curve: ql.ZeroCurve,
) -> float:
    """
    Find the z-spread that reprices the bond to its market value.

    Parameters
    ----------
    bond : ql.FixedRateBond
        QuantLib bond instrument.
    market_price : float
        Observed market price (clean price, per 100 face).
    base_curve : ql.ZeroCurve
        Risk-free yield curve.

    Returns
    -------
    float
        Z-spread in decimal (e.g., 0.015 = 150 bps).

    Raises
    ------
    ValueError
        If calibration fails to converge.
    """
    day_count = ql.Actual365Fixed()
    compounding = ql.Continuous
    frequency = ql.Annual

    curve_handle = ql.YieldTermStructureHandle(base_curve)

    z_spread = ql_call(
        "ql.CashFlows.zSpread",
        lambda: ql.CashFlows.zSpread(
            bond.cashflows(),
            market_price,
            curve_handle,
            day_count,
            compounding,
            frequency,
            True,  # includeSettlementDateFlows
        ),
        context=f"Z-spread calibration, MV={market_price}"
    )

    # Bounds check: clip to [-10%, +10%] per PHASE1-003 spec
    z_spread = max(-0.10, min(z_spread, 0.10))

    ql_audit_log(
        operation="calibrate_zspread",
        inputs={"market_price": market_price},
        outputs={"z_spread": z_spread, "z_spread_bps": z_spread * 10_000},
    )

    return z_spread
```

### 4.4 Weighted Average Z-Spread

For setting the liability accumulation rate, a market-value-weighted average z-spread is computed across the portfolio:

**Formula** (from PHASE1-003, Section 4):

```
wa_spread = SUM(spread_i * MV_i) / SUM(MV_i)

where:
  spread_i = z-spread of asset i (decimal)
  MV_i     = market value of asset i (must be >= 0)
```

**Bounds:** Clip result to [-10%, +10%] (i.e., [-1000 bps, +1000 bps]).

**Edge cases:**
- Exclude assets with negative market values (short positions).
- If total weight is zero, fall back to equal-weighted average.
- If result is NaN, raise an error (do not silently continue).

### 4.5 Z-Spread Mean Reversion (Projection)

For projection years `t > 0`, the z-spread mean-reverts toward a long-term level:

**Formula** (from PHASE1-003, Section 3):

```
spread(t) = L + (S - L) * rho^t

where:
  S   = short-term spread at t=0 (from calibration)
  L   = long-term spread (TTC = Through-The-Cycle average)
  rho = reversion speed parameter
  t   = projection year
```

**Two speed parameters:**

```
if S > L:    # Spread is WIDE (above long-term average)
    rho = rho_over   # Faster reversion (0.93 - 0.95 typical)
else:        # Spread is TIGHT (below long-term average)
    rho = rho_under  # Slower reversion (0.95 - 0.97 typical)
```

**Rationale:** Wide spreads tend to tighten faster (markets normalize); tight spreads widen more gradually.

### 4.6 Building a Spread-Adjusted Curve

```python
@rule_ref("Handbook E9; Rules Para 28(9)", "Z-spread overlay on risk-free curve")
def build_spread_adjusted_curve(
    base_curve: ql.ZeroCurve,
    z_spread: float,
) -> ql.ZeroSpreadedTermStructure:
    """
    Create a yield curve = risk-free + z-spread.

    Parameters
    ----------
    base_curve : ql.ZeroCurve
        Risk-free curve (base or scenario-shifted).
    z_spread : float
        Z-spread in decimal (e.g., 0.015 for 150 bps).

    Returns
    -------
    ql.ZeroSpreadedTermStructure
        Spread-adjusted curve.
    """
    curve_handle = ql.YieldTermStructureHandle(base_curve)
    spread_handle = ql.QuoteHandle(ql.SimpleQuote(z_spread))

    adjusted_curve = ql_call(
        "ql.ZeroSpreadedTermStructure",
        lambda: ql.ZeroSpreadedTermStructure(curve_handle, spread_handle),
        context=f"Spread overlay, z_spread={z_spread:.4f} ({z_spread*10000:.1f} bps)"
    )
    adjusted_curve.enableExtrapolation()

    ql_audit_log(
        operation="build_spread_adjusted_curve",
        inputs={"z_spread": z_spread},
        outputs={"curve_type": "ZeroSpreadedTermStructure"},
    )

    return adjusted_curve
```

---

## 5. ql_audit.py Wrapping Pattern

### 5.1 Purpose

Every QuantLib call in the `curves/` package is wrapped through `ql_audit.py` to achieve:

1. **Audit trail:** Every QuantLib invocation is logged with inputs, outputs, timestamps, and BMA rule references.
2. **Error containment:** QuantLib exceptions are caught, enriched with context, and re-raised as domain exceptions.
3. **Performance tracking:** Execution time of each QL call is recorded.
4. **Reproducibility:** All inputs are captured so that any calculation can be replayed.

### 5.2 Core Implementation

```python
# --- curves/ql_audit.py ---

import logging
import time
import json
from typing import Any, Callable, Optional
from dataclasses import dataclass, field
from datetime import datetime

logger = logging.getLogger("bma_sba_benchmark.curves.ql_audit")

# Global audit log (JSON Lines format, written to engine/audit_trail)
_audit_entries: list = []


@dataclass
class QLAuditEntry:
    """One logged QuantLib operation."""
    timestamp: str
    operation: str
    ql_function: str
    context: str
    inputs: dict
    outputs: dict
    elapsed_ms: float
    success: bool
    error: Optional[str] = None


def ql_call(
    ql_function_name: str,
    fn: Callable,
    context: str = "",
) -> Any:
    """
    Execute a QuantLib function with full audit logging.

    Parameters
    ----------
    ql_function_name : str
        Name of the QuantLib function/class being called (for logging).
    fn : callable
        A zero-argument callable that performs the QuantLib operation.
    context : str
        Human-readable description of why this call is being made.

    Returns
    -------
    Any
        The result of the QuantLib call.

    Raises
    ------
    QuantLibError
        If the QuantLib call fails, wrapped with context.
    """
    start = time.perf_counter()
    try:
        result = fn()
        elapsed = (time.perf_counter() - start) * 1000

        entry = QLAuditEntry(
            timestamp=datetime.utcnow().isoformat(),
            operation="ql_call",
            ql_function=ql_function_name,
            context=context,
            inputs={},  # Captured by caller in ql_audit_log
            outputs={"type": type(result).__name__},
            elapsed_ms=round(elapsed, 3),
            success=True,
        )
        _audit_entries.append(entry)

        logger.debug(
            f"QL call: {ql_function_name} ({elapsed:.1f}ms) - {context}"
        )
        return result

    except Exception as e:
        elapsed = (time.perf_counter() - start) * 1000
        entry = QLAuditEntry(
            timestamp=datetime.utcnow().isoformat(),
            operation="ql_call",
            ql_function=ql_function_name,
            context=context,
            inputs={},
            outputs={},
            elapsed_ms=round(elapsed, 3),
            success=False,
            error=str(e),
        )
        _audit_entries.append(entry)

        logger.error(
            f"QL call FAILED: {ql_function_name} - {e} - {context}"
        )
        raise QuantLibError(
            f"QuantLib error in {ql_function_name}: {e}\n"
            f"Context: {context}"
        ) from e


def ql_audit_log(
    operation: str,
    inputs: dict,
    outputs: dict,
) -> None:
    """
    Log a high-level audit entry for a curve-building operation.

    This is called after a series of ql_call()s to record the
    composite operation with its full inputs and outputs.
    """
    entry = QLAuditEntry(
        timestamp=datetime.utcnow().isoformat(),
        operation=operation,
        ql_function="(composite)",
        context=operation,
        inputs=inputs,
        outputs=outputs,
        elapsed_ms=0,
        success=True,
    )
    _audit_entries.append(entry)

    logger.info(
        f"QL audit: {operation} | "
        f"inputs={json.dumps(inputs, default=str)} | "
        f"outputs={json.dumps(outputs, default=str)}"
    )


def flush_audit_log(filepath: str) -> None:
    """Write all audit entries to a JSON Lines file."""
    with open(filepath, "a") as f:
        for entry in _audit_entries:
            f.write(json.dumps(entry.__dict__, default=str) + "\n")
    _audit_entries.clear()


def get_audit_summary() -> dict:
    """Return summary statistics of QL calls for the current run."""
    total_calls = len([e for e in _audit_entries if e.operation == "ql_call"])
    failed = len([e for e in _audit_entries if not e.success])
    total_ms = sum(e.elapsed_ms for e in _audit_entries)
    return {
        "total_ql_calls": total_calls,
        "failed_calls": failed,
        "total_elapsed_ms": round(total_ms, 1),
    }


class QuantLibError(Exception):
    """Domain exception wrapping QuantLib errors with context."""
    pass
```

### 5.3 Audit Trail Output Format

Each entry in the audit trail is one JSON line:

```json
{"timestamp": "2024-12-31T00:00:01.234", "operation": "build_base_curve", "ql_function": "(composite)", "context": "build_base_curve", "inputs": {"valuation_date": "2024-12-31", "num_tenors": 13, "shortest_tenor": 0.25, "longest_tenor": 50, "short_rate": 0.042, "long_rate": 0.0366}, "outputs": {"curve_type": "ZeroCurve", "interpolation": "Linear", "extrapolation": true, "day_count": "Actual365Fixed"}, "elapsed_ms": 0, "success": true, "error": null}
```

### 5.4 Integration with @rule_ref Decorator

The `@rule_ref` decorator (from `utils/rule_ref.py`) tags every function with its BMA rule paragraph. The audit log captures this context:

```python
# --- utils/rule_ref.py ---

import functools

def rule_ref(rule: str, description: str = ""):
    """
    Decorator that tags a function with its BMA rule reference.

    Usage:
        @rule_ref("Rules Sched XXV Para 28(7)(b)", "Decrease scenario")
        def decrease_shift(year): ...
    """
    def decorator(fn):
        fn._rule_ref = rule
        fn._rule_description = description

        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            return fn(*args, **kwargs)

        wrapper._rule_ref = rule
        wrapper._rule_description = description
        return wrapper
    return decorator
```

---

## 6. Worked Example

### 6.1 Setup

**Valuation date:** 2024-12-31

**Base risk-free spot rates (BMA published):**

| Tenor | Rate |
|---|---|
| 1y | 4.00% |
| 5y | 3.80% |
| 10y | 3.75% |
| 20y | 3.70% |
| 30y | 3.68% |

### 6.2 Step 1: Build the Base Curve

```python
spot_rates = [
    (1, 0.0400),
    (5, 0.0380),
    (10, 0.0375),
    (20, 0.0370),
    (30, 0.0368),
]
base_curve = build_base_curve(date(2024, 12, 31), spot_rates)
```

**Verification -- read back rates:**

| Tenor | Input Rate | Read-Back Rate | Match? |
|---|---|---|---|
| 1y | 4.00% | 4.00% | Yes (knot point) |
| 3y | (interpolated) | 3.90% | Yes: 4.00 + (3.80-4.00)*(3-1)/(5-1) = 3.90 |
| 5y | 3.80% | 3.80% | Yes (knot point) |
| 10y | 3.75% | 3.75% | Yes (knot point) |
| 15y | (interpolated) | 3.725% | Yes: 3.75 + (3.70-3.75)*(15-10)/(20-10) = 3.725 |
| 20y | 3.70% | 3.70% | Yes (knot point) |
| 50y | (extrapolated) | 3.68% | Yes (flat beyond 30y) |

### 6.3 Step 2: Apply Scenario 2 (Decrease) at Projection Year 5

**Rule ref:** Para 28(7)(b)

At year 5, the temporal grading factor = min(5/10, 1.0) = 0.5.

Shift = -150 bps * 0.5 = **-75 bps** = -0.0075 (uniform across all tenors).

| Tenor | Base Rate | Shift | Shifted Rate |
|---|---|---|---|
| 1y | 4.00% | -0.75% | **3.25%** |
| 5y | 3.80% | -0.75% | **3.05%** |
| 10y | 3.75% | -0.75% | **3.00%** |
| 20y | 3.70% | -0.75% | **2.95%** |
| 30y | 3.68% | -0.75% | **2.93%** |

### 6.4 Step 3: Apply Scenario 6 (Decrease + Positive Twist) at Projection Year 10

**Rule ref:** Para 28(7)(f)

At year 10, temporal grading = min(10/10, 1.0) = 1.0 (fully graded in).

Short shift = -150 bps, Long shift = -50 bps.

Tenor interpolation (between 1y and 20y):

```
tenor_shift(tenor) = -0.0150 + (-0.0050 - (-0.0150)) * (tenor - 1) / (20 - 1)
                   = -0.0150 + 0.0100 * (tenor - 1) / 19
```

| Tenor | Base Rate | tenor_shift Calc | Shift | Shifted Rate |
|---|---|---|---|---|
| 1y | 4.00% | -150 + 100*(0/19) = -150 bps | -1.50% | **2.50%** |
| 5y | 3.80% | -150 + 100*(4/19) = -129 bps | -1.29% | **2.51%** |
| 10y | 3.75% | -150 + 100*(9/19) = -103 bps | -1.03% | **2.72%** |
| 20y | 3.70% | -150 + 100*(19/19) = -50 bps | -0.50% | **3.20%** |
| 30y | 3.68% | Clamped to long_shift = -50 bps | -0.50% | **3.18%** |

**Observation:** The curve steepens -- the short end drops more than the long end. The 1y-20y spread goes from -30 bps (base: 4.00% vs 3.70%) to +70 bps (shifted: 2.50% vs 3.20%).

### 6.5 Step 4: Apply Scenario 4 (Down-Up Hump) at Various Years

**Rule ref:** Para 28(7)(d)

| Year | Hump Formula | Shift (bps) |
|---|---|---|
| 0 | -150 * (0/5) | 0 |
| 2 | -150 * (2/5) | -60 |
| 5 | -150 * (5/5) | -150 (peak) |
| 7 | -150 * (10-7)/5 | -90 |
| 10 | -150 * (10-10)/5 | 0 (back to base) |
| 15 | 0 | 0 |

At the peak (year 5), the 10y base rate of 3.75% shifts to 3.75% - 1.50% = **2.25%**. By year 10, it returns to **3.75%**.

### 6.6 Step 5: Z-Spread Overlay

Given a corporate bond with:
- Market price: $98.50 (per $100 face)
- Coupon: 5.0%, annual
- Maturity: 10 years
- Base 10y risk-free rate: 3.75%

The calibrated z-spread via `calibrate_zspread()` would be approximately:
- Bond yield (from price) ~ 5.20%
- Z-spread ~ 5.20% - 3.75% ~ **145 bps** (approximate; actual QuantLib result accounts for full term structure)

The spread-adjusted 10y rate = 3.75% + 1.45% = **5.20%**.

Under Scenario 2 at year 10 (Decrease, full grading):
- Shifted risk-free 10y rate = 3.75% - 1.50% = 2.25%
- Spread-adjusted rate = 2.25% + 1.45% = **3.70%** (assuming spread held constant)
- In practice, the spread mean-reverts: `spread(10) = L + (S - L) * rho^10`

---

## 7. Edge Cases and Validation

### 7.1 Negative Rate Floor

**Condition:** `base_rate + shift < 0`

**Handling:** Floor at zero. Log a warning.

```python
shifted_rate = max(base_rate + shift, 0.0)
if base_rate + shift < 0:
    logger.warning(
        f"Rate floored at 0: base={base_rate:.4f}, shift={shift:.4f}, "
        f"tenor={tenor}, year={year}"
    )
```

**Example:** Base 1y rate = 1.00%, Scenario 2 at year 10 shift = -1.50%. Shifted = -0.50%, floored to **0.00%**.

### 7.2 Inverted Curve After Twist

**Condition:** A twist scenario may invert the curve at certain tenors.

**Handling:** This is expected and valid. No correction needed -- the BMA scenarios are designed to test exactly this kind of stress. Log the inversion for audit purposes.

```python
if shifted_rates[i] > shifted_rates[i+1]:
    logger.info(
        f"Curve inversion: {tenors[i]}y={shifted_rates[i]:.4f} > "
        f"{tenors[i+1]}y={shifted_rates[i+1]:.4f}"
    )
```

### 7.3 Missing Tenors in Input Data

**Condition:** The BMA-published curve may not include all tenors in the standard grid.

**Handling:** QuantLib's ZeroCurve interpolates between provided knot points. Ensure at minimum the short end (<=1y) and long end (>=20y) are present. Raise a validation error if fewer than 3 tenor points are provided.

```python
if len(spot_rates) < 3:
    raise ValueError(
        f"Insufficient tenor points: {len(spot_rates)}. "
        f"Minimum 3 required for meaningful interpolation."
    )
if spot_rates[0][0] > 1.0:
    raise ValueError(
        f"Shortest tenor is {spot_rates[0][0]}y. "
        f"Must include at least one tenor <= 1y for short-end anchoring."
    )
```

### 7.4 Z-Spread Out of Bounds

**Condition:** Calibrated z-spread exceeds [-10%, +10%].

**Handling:** Clip and log a warning. This bounds check is from PHASE1-003, Section 9.

```python
if z_spread < -0.10 or z_spread > 0.10:
    logger.warning(
        f"Z-spread clipped: raw={z_spread:.4f}, "
        f"clipped={max(-0.10, min(z_spread, 0.10)):.4f}"
    )
    z_spread = max(-0.10, min(z_spread, 0.10))
```

### 7.5 Very Long Tenors (>50 years)

**Condition:** Some liabilities extend past 50 years.

**Handling:** Extrapolation is enabled on all curves. The rate beyond the last knot point is held flat. This is conservative and appropriate for regulatory purposes.

### 7.6 Validation Checks at Curve Construction

After building any curve, run these sanity checks:

```python
def validate_curve(curve: ql.ZeroCurve, label: str) -> None:
    """Post-construction validation."""
    test_tenors = [1, 5, 10, 20, 30]
    day_count = ql.Actual365Fixed()

    rates = []
    for t in test_tenors:
        dt = curve.referenceDate() + ql.Period(t, ql.Years)
        r = curve.zeroRate(dt, day_count, ql.Continuous).rate()
        rates.append(r)

        # Check rate is non-negative
        if r < 0:
            raise ValueError(f"{label}: Negative rate {r:.4f} at {t}y")

        # Check rate is reasonable (0% to 25%)
        if r > 0.25:
            raise ValueError(f"{label}: Implausible rate {r:.4f} at {t}y (>25%)")

    # Check discount factors are monotonically decreasing
    for t in [1, 5, 10, 20, 30, 50]:
        dt = curve.referenceDate() + ql.Period(t, ql.Years)
        df = curve.discount(dt)
        if df <= 0 or df > 1:
            raise ValueError(f"{label}: Invalid discount factor {df:.6f} at {t}y")
```

---

## 8. Implementation Notes

### 8.1 What Goes in `curves/` vs `projection/`

| Responsibility | Module | QuantLib? |
|---|---|---|
| Build base risk-free ZeroCurve from spot rates | `curves/curve_builder.py` | Yes |
| Apply scenario shifts, build scenario ZeroCurves | `curves/scenario_curves.py` | Yes |
| Calibrate z-spread from bond market price | `curves/curve_builder.py` | Yes |
| Build spread-adjusted curves | `curves/curve_builder.py` | Yes |
| Price bonds given a curve | `curves/bond_pricing.py` | Yes |
| Wrap all QuantLib calls | `curves/ql_audit.py` | Yes (wrapper) |
| Export curves to NumPy arrays | `curves/curve_builder.py` | Yes (reads from QL, exports to NumPy) |
| Compute weighted-average z-spread | `projection/credit_adjustment.py` | **No** (pure Python) |
| Apply mean reversion to z-spread over time | `projection/credit_adjustment.py` | **No** (pure Python) |
| Use discount factors for cash flow PV | `projection/asset_cf.py` | **No** (uses exported arrays) |
| Determine scenario shifts (rule definitions) | `rules/v2024/scenarios.py` | **No** (pure data) |
| Select biting scenario | `projection/multi_scenario.py` | **No** (pure Python) |

### 8.2 Data Flow Summary

```
1. assumptions/tables/bma_rfr_2024Q4.csv
      |
      v
2. curves/curve_builder.py::build_base_curve()
      |  -> ql.ZeroCurve (QuantLib object, internal only)
      v
3. curves/scenario_curves.py::build_scenario_curve()
      |  -> 9 x 101 ql.ZeroCurve objects
      v
4. curves/curve_builder.py::export_curve_to_array()
      |  -> dict{"tenors": ndarray, "zero_rates": ndarray, "discount_factors": ndarray}
      v
5. projection/scenario_runner.py  (pure Python from here)
      |  -> Uses discount_factors for PV calculations
      v
6. projection/multi_scenario.py
      |  -> Runs all 9, identifies biting scenario
      v
7. calculations/bel.py
      -> BEL = MV_assets(T=0) + C0_biting
```

### 8.3 Thread Safety

Curve construction is **not** thread-safe due to QuantLib's global `Settings.instance().evaluationDate`. Scenarios must be computed sequentially, or each thread must set its own evaluation date before each curve build. The recommended approach is sequential pre-computation (Section 3.8).

### 8.4 Performance Budget

| Operation | Expected Time | Count per Run |
|---|---|---|
| Build base curve | ~1 ms | 1 |
| Build one scenario curve | ~2 ms | 9 x 101 = 909 |
| Export one curve to arrays | ~1 ms | 909 |
| Total curve pre-computation | ~3 seconds | 1 |
| Z-spread calibration (one bond) | ~5 ms | ~1000 bonds |
| Total z-spread calibration | ~5 seconds | 1 |
| **Total curves/ module time** | **~8 seconds** | **per run** |

### 8.5 Testing Strategy

| Test | What It Verifies | File |
|---|---|---|
| `test_base_curve_knot_points` | Rates read back at knot points match inputs exactly | `tests/test_curves.py` |
| `test_base_curve_interpolation` | Linear interpolation between knots is correct | `tests/test_curves.py` |
| `test_base_curve_extrapolation` | Flat extrapolation beyond last tenor | `tests/test_curves.py` |
| `test_parallel_shift_year_0` | No shift at year 0 | `tests/test_scenarios.py` |
| `test_parallel_shift_year_10` | Full shift at year 10 | `tests/test_scenarios.py` |
| `test_parallel_shift_year_50` | Full shift still at year 50 | `tests/test_scenarios.py` |
| `test_hump_peak_at_year_5` | Peak shift at year 5 | `tests/test_scenarios.py` |
| `test_hump_returns_to_zero` | Shift is 0 at year 10 and beyond | `tests/test_scenarios.py` |
| `test_twist_short_end` | Short tenors get short_shift | `tests/test_scenarios.py` |
| `test_twist_long_end` | Long tenors get long_shift | `tests/test_scenarios.py` |
| `test_twist_interpolation` | Mid tenors interpolate linearly | `tests/test_scenarios.py` |
| `test_negative_rate_floor` | Rates never go below 0 | `tests/test_curves.py` |
| `test_zspread_calibration` | Round-trip: calibrate spread, reprice, match MV | `tests/test_curves.py` |
| `test_zspread_bounds` | Spreads clipped to [-10%, +10%] | `tests/test_curves.py` |
| `test_illustrative_calc_rates` | Golden test: rates match BMA illustrative example | `tests/test_illustrative_calc.py` |
| `test_audit_log_populated` | Every QL call produces an audit entry | `tests/test_ql_audit.py` |

### 8.6 Dependencies

```
curves/curve_builder.py
  imports: QuantLib, numpy, curves.ql_audit

curves/scenario_curves.py
  imports: QuantLib, numpy, curves.curve_builder, curves.ql_audit,
           rules.v2024.scenarios

curves/ql_audit.py
  imports: logging, time, json (standard library only)

rules/v2024/scenarios.py
  imports: dataclasses, enum (standard library only; NO QuantLib)
```

---

## Appendix A: Complete Scenario Shift Tables

### A.1 Scenario 2 (Decrease) -- All Projection Years

| Year | Grading | Shift (bps) |
|---|---|---|
| 0 | 0.0 | 0 |
| 1 | 0.1 | -15 |
| 2 | 0.2 | -30 |
| 3 | 0.3 | -45 |
| 4 | 0.4 | -60 |
| 5 | 0.5 | -75 |
| 6 | 0.6 | -90 |
| 7 | 0.7 | -105 |
| 8 | 0.8 | -120 |
| 9 | 0.9 | -135 |
| 10+ | 1.0 | -150 |

### A.2 Scenario 4 (Down-Up Hump) -- All Projection Years

| Year | Phase | Grading | Shift (bps) |
|---|---|---|---|
| 0 | Ramp up | 0/5 | 0 |
| 1 | Ramp up | 1/5 | -30 |
| 2 | Ramp up | 2/5 | -60 |
| 3 | Ramp up | 3/5 | -90 |
| 4 | Ramp up | 4/5 | -120 |
| 5 | Peak | 5/5 | -150 |
| 6 | Ramp down | 4/5 | -120 |
| 7 | Ramp down | 3/5 | -90 |
| 8 | Ramp down | 2/5 | -60 |
| 9 | Ramp down | 1/5 | -30 |
| 10+ | Post-hump | 0 | 0 |

### A.3 Scenario 6 (Decrease + Positive Twist) -- Tenor Grid at Full Grading

| Tenor (y) | Clamped Tenor | tenor_fraction | Short + Diff * frac | Shift (bps) |
|---|---|---|---|---|
| 0.25 | 1.0 | 0.000 | -150 + 100*0.000 | -150.0 |
| 0.50 | 1.0 | 0.000 | -150 + 100*0.000 | -150.0 |
| 1 | 1.0 | 0.000 | -150 + 100*0.000 | -150.0 |
| 2 | 2.0 | 0.053 | -150 + 100*0.053 | -144.7 |
| 3 | 3.0 | 0.105 | -150 + 100*0.105 | -139.5 |
| 5 | 5.0 | 0.211 | -150 + 100*0.211 | -128.9 |
| 7 | 7.0 | 0.316 | -150 + 100*0.316 | -118.4 |
| 10 | 10.0 | 0.474 | -150 + 100*0.474 | -102.6 |
| 15 | 15.0 | 0.737 | -150 + 100*0.737 | -76.3 |
| 20 | 20.0 | 1.000 | -150 + 100*1.000 | -50.0 |
| 30 | 20.0 | 1.000 | (clamped) | -50.0 |
| 50 | 20.0 | 1.000 | (clamped) | -50.0 |

---

## Appendix B: Glossary

| Term | Definition |
|---|---|
| **Base curve** | The T=0 risk-free yield curve with no scenario adjustments |
| **Biting scenario** | The scenario (of 9) requiring the highest initial cash buffer C0 |
| **BPS** | Basis points; 1 bps = 0.01% = 0.0001 in decimal |
| **Day count** | Convention for measuring time between dates (Actual/365 Fixed) |
| **Discount factor** | Present value of $1 received at tenor t: DF(t) = exp(-r*t) |
| **Grading factor** | Temporal multiplier that phases in the scenario shift over time |
| **Knot point** | A tenor where the rate is explicitly defined (not interpolated) |
| **Settlement lag** | T+2: two business days between trade and settlement |
| **Tenor** | Time to maturity of a rate or bond, in years |
| **Z-spread** | The constant spread added to the risk-free curve such that the discounted cash flows equal the bond's market price |
| **ZeroCurve** | QuantLib class that builds a yield curve from zero (spot) rates |
