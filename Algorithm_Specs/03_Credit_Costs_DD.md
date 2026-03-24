# Algorithm Spec 03: Default & Downgrade (D&D) Credit Costs

## Document Control

| Field | Value |
|---|---|
| **Spec ID** | ALGO-03 |
| **Target Modules** | `projection/credit_costs.py`, `assumptions/credit_tables.py` |
| **BMA Rule References** | Rules Sched XXV Para 28(22)-(26); Handbook E10 |
| **Status** | Implementation-Ready Draft |
| **Date** | March 2026 |

---

## 1. Overview

### 1.1 Purpose

This specification defines the algorithm for computing Default and Downgrade (D&D) credit cost deductions applied to projected asset cash flows in the BMA SBA Benchmark Model. D&D costs reduce asset cash flows to reflect expected credit losses over the projection horizon, ensuring that the Best Estimate Liability (BEL) calculation accounts for the realistic possibility that not all promised asset cash flows will be received.

### 1.2 Target Modules

| Module | Responsibility |
|---|---|
| `assumptions/credit_tables.py` | Load, validate, and serve D&D lookup tables (Pydantic models) |
| `projection/credit_costs.py` | Compute per-asset, per-period D&D deductions (pure Python, no QuantLib) |

### 1.3 Key Rule References

| Rule | Subject |
|---|---|
| Para 28(22) | Projected cash flows must be reduced for defaults and rating migration |
| Para 28(23) | Methodology: realized average default losses + uncertainty margin |
| Para 28(24) | BMA published tables are **floors** (companies may use higher, never lower) |
| Para 28(25)-(26) | Transitional phase-in for downgrade component |
| Handbook E10 | Detailed guidance on D&D assumptions and attestation |

### 1.4 Architectural Constraints

- **Pure Python only** -- this module lives in `projection/`, so no QuantLib imports.
- **@rule_ref decorator** on every calculation function, citing BMA paragraph.
- **Fail loudly** -- Pydantic validation on all table loads; missing or invalid data halts the run.
- **DataFrame outputs** -- all intermediate D&D results stored as pandas DataFrames for inspection.

---

## 2. BMA Regulatory Requirements

### 2.1 What the Rules Actually Say

The BMA requires that projected asset cash flows be **reduced** to account for two distinct credit risks:

1. **Default costs (baseline):** The expected loss from outright defaults, based on realized average default losses by rating and duration. This represents the "best estimate" component.

2. **Downgrade costs (uncertainty margin):** An additional margin reflecting the expected loss from rating migration (credit deterioration short of default, causing spread widening and market value loss). This represents the "uncertainty" or "prudence" component.

Together, these two components form the total D&D deduction.

### 2.2 BMA Floor Constraint

Para 28(24): The D&D assumptions used by a company **cannot be lower** than the BMA published tables. The benchmark model uses the BMA floors directly. When comparing against company submissions, the `challenge/` module must verify that company assumptions meet or exceed these floors.

### 2.3 Transitional Phase-In

Para 28(25)-(26): For business in force as of December 31, 2023, the **downgrade cost component only** is phased in over 5 years:

| Valuation Year | Phase-In Factor | Notes |
|---|---|---|
| 2024 | 20% | First year of SBA regime |
| 2025 | 40% | |
| 2026 | 60% | |
| 2027 | 80% | |
| 2028+ | 100% | Fully phased in |

**Important distinctions:**
- The phase-in applies ONLY to the downgrade component, NOT to default costs.
- The phase-in applies ONLY to business in force as of Dec 31, 2023. New business written after that date gets 100% downgrade costs from inception.
- Default costs are always applied at 100% regardless of vintage or year.

### 2.4 Limited Basis Assets (Tier 3)

Para 28(15)-(16) and Handbook E10: For assets acceptable on a limited basis (Tier 3), the uncertainty adjustment (downgrade component) must be at least 1 standard deviation of the default cost distribution. This imposes a higher D&D charge on riskier, less liquid assets.

---

## 3. Formula Specification

### 3.1 Reconciling BMA vs Pythora Approaches

Two formulations exist in our reference material. This section reconciles them and selects the approach for implementation.

**BMA / Technical Spec Approach (Additive):**
```
default_cost(t)   = incremental_default_rate(rating, t) * par_remaining(t)
downgrade_cost(t) = incremental_downgrade_rate(rating, t) * par_remaining(t) * phase_in
total_dd_cost(t)  = default_cost(t) + downgrade_cost(t)
adjusted_cf(t)    = gross_cf(t) - total_dd_cost(t)
```

**Pythora Approach (Multiplicative):**
```
MV_adj(t) = MV(t) * (1 - X)^t
CF_adj(t) = CF(t) * (1 - X)^t
```
where X = annualized D&D cost from lookup table by (asset_class, rating, years_since_inception).

### 3.2 Analysis of the Two Approaches

| Dimension | BMA/Additive | Pythora/Multiplicative |
|---|---|---|
| Conceptual model | Deduct dollar cost from cash flows | Apply survival factor to cash flows |
| Phase-in handling | Naturally separates default and downgrade components | Requires splitting X into two parts |
| Compounding | Implicit in cumulative rate table | Explicit via (1-X)^t |
| Auditability | Each cost line visible | Single factor, less transparent |
| BMA alignment | Directly matches rule language | Mathematically equivalent if rates are small |

### 3.3 Selected Approach: Hybrid

We implement the **multiplicative survival factor** approach (consistent with Pythora and standard actuarial practice) but **separate the default and downgrade factors** to support the phase-in transitional arrangement and audit transparency.

**Core formulas:**

```
# Per-asset, per-period survival factors
default_survival(t)   = (1 - d_annual(rating, asset_class, t))^t
downgrade_survival(t) = (1 - g_annual(rating, asset_class, t))^t

# Effective downgrade survival with phase-in
downgrade_survival_eff(t) = 1 - phase_in(year, vintage) * (1 - downgrade_survival(t))

# Combined survival factor
combined_survival(t) = default_survival(t) * downgrade_survival_eff(t)

# Application to cash flows
adjusted_cf(t) = gross_cf(t) * combined_survival(t)
```

Where:
- `d_annual(rating, asset_class, t)` = annualized default rate from BMA table for year t
- `g_annual(rating, asset_class, t)` = annualized downgrade rate from BMA table for year t
- `phase_in(year, vintage)` = phase-in factor (0.0 to 1.0)
- `t` = projection year (years from valuation date for existing assets, or years since purchase for reinvested assets)

### 3.4 Derivation of Annualized Rates from Cumulative Tables

BMA tables provide **cumulative** default/downgrade rates. We need annualized rates:

```
# BMA table gives: cumulative_rate(rating, T) for T = 1, 2, 3, ..., 30
# This is the total cumulative probability of default/downgrade over T years

# Annualized rate for use in survival factor:
# (1 - d_annual)^T = 1 - cumulative_rate(T)
# Therefore:
d_annual(T) = 1 - (1 - cumulative_rate(T))^(1/T)
```

This annualized rate is **term-dependent** -- a 5-year annualized rate differs from a 3-year annualized rate because cumulative default probabilities are not perfectly linear.

### 3.5 Alternative: Incremental Period Costs

For maximum precision, we can also compute **incremental** (marginal) costs per period:

```
# Incremental default cost in period t:
incremental_survival(t) = (1 - cumulative_rate(t)) / (1 - cumulative_rate(t-1))

# This is the conditional survival probability for period t,
# given survival through period t-1.

# Cumulative survival through period T:
cumulative_survival(T) = PRODUCT(incremental_survival(t) for t = 1 to T)
                       = 1 - cumulative_rate(T)   # By construction
```

**Implementation decision:** Use the **incremental approach** for the projection engine because it naturally handles the period-by-period structure and avoids floating-point issues with large exponents.

### 3.6 Final Implemented Formulas

For each asset `a` in each projection period `t`:

```
# Step 1: Look up cumulative rates from BMA tables
cum_def_rate(t)   = BMA_default_table[a.rating][a.asset_class][t]
cum_def_rate(t-1) = BMA_default_table[a.rating][a.asset_class][t-1]   # 0 if t=1

cum_dg_rate(t)    = BMA_downgrade_table[a.rating][a.asset_class][t]
cum_dg_rate(t-1)  = BMA_downgrade_table[a.rating][a.asset_class][t-1]  # 0 if t=1

# Step 2: Compute incremental survival factors
inc_def_survival(t) = (1 - cum_def_rate(t)) / (1 - cum_def_rate(t-1))
inc_dg_survival(t)  = (1 - cum_dg_rate(t)) / (1 - cum_dg_rate(t-1))

# Step 3: Apply phase-in to downgrade component
phi = phase_in_factor(valuation_year, a.vintage)
inc_dg_survival_eff(t) = 1 - phi * (1 - inc_dg_survival(t))

# Step 4: Compute cumulative survival through period T
cum_def_survival(T)    = PRODUCT(inc_def_survival(t), t=1..T)
cum_dg_survival_eff(T) = PRODUCT(inc_dg_survival_eff(t), t=1..T)

# Step 5: Combined survival factor
combined_survival(T) = cum_def_survival(T) * cum_dg_survival_eff(T)

# Step 6: Adjust cash flows
adjusted_cf(t) = gross_cf(t) * combined_survival(t)
```

**Equivalence check:** When phase_in = 1.0 (fully phased in), the combined survival simplifies to:
```
combined_survival(T) = (1 - cum_def_rate(T)) * (1 - cum_dg_rate(T))
```
which is approximately `1 - cum_def_rate(T) - cum_dg_rate(T)` for small rates, consistent with the additive BMA formulation.

---

## 4. D&D Lookup Table Structure

### 4.1 Table Schema

Two separate tables: one for default costs, one for downgrade costs.

```python
# assumptions/credit_tables.py

from pydantic import BaseModel, Field, field_validator
from typing import Dict, Literal
from enum import Enum
import pandas as pd


class CreditRating(str, Enum):
    AAA = "AAA"
    AA_PLUS = "AA+"
    AA = "AA"
    AA_MINUS = "AA-"
    A_PLUS = "A+"
    A = "A"
    A_MINUS = "A-"
    BBB_PLUS = "BBB+"
    BBB = "BBB"
    BBB_MINUS = "BBB-"
    BB_PLUS = "BB+"
    BB = "BB"
    BB_MINUS = "BB-"
    B_PLUS = "B+"
    B = "B"
    B_MINUS = "B-"
    CCC = "CCC"
    CC = "CC"
    C = "C"
    D = "D"
    NR = "NR"


class AssetClass(str, Enum):
    GOVT_SOVEREIGN = "GOVT_SOVEREIGN"
    IG_CORPORATE_SNR = "IG_CORPORATE_SNR"
    IG_CORPORATE_SUB = "IG_CORPORATE_SUB"
    HY_CORPORATE = "HY_CORPORATE"
    MUNICIPAL = "MUNICIPAL"
    COVERED_BOND = "COVERED_BOND"
    ABS = "ABS"
    RMBS = "RMBS"
    CMBS = "CMBS"
    CLO = "CLO"
    PREFERRED_STOCK = "PREFERRED_STOCK"
    MORTGAGE_LOAN = "MORTGAGE_LOAN"


class CreditCostRow(BaseModel):
    """One row in the D&D lookup table."""
    asset_class: AssetClass
    rating: CreditRating
    year_1: float = Field(ge=0.0, le=1.0)
    year_2: float = Field(ge=0.0, le=1.0)
    year_3: float = Field(ge=0.0, le=1.0)
    year_4: float = Field(ge=0.0, le=1.0)
    year_5: float = Field(ge=0.0, le=1.0)
    year_6: float = Field(ge=0.0, le=1.0)
    year_7: float = Field(ge=0.0, le=1.0)
    year_8: float = Field(ge=0.0, le=1.0)
    year_9: float = Field(ge=0.0, le=1.0)
    year_10: float = Field(ge=0.0, le=1.0)
    year_15: float = Field(ge=0.0, le=1.0)
    year_20: float = Field(ge=0.0, le=1.0)
    year_30: float = Field(ge=0.0, le=1.0)

    @field_validator("*", mode="before")
    @classmethod
    def coerce_pct_strings(cls, v):
        """Handle '0.17%' style strings from Excel/CSV."""
        if isinstance(v, str) and v.endswith("%"):
            return float(v.rstrip("%")) / 100.0
        return v

    def cumulative_rate(self, year: int) -> float:
        """Return cumulative rate for a given year, interpolating if needed."""
        ...  # See Section 4.3
```

### 4.2 CSV File Format

File: `assumptions/tables/bma_default_floors.csv`

```csv
asset_class,rating,year_1,year_2,year_3,year_4,year_5,year_6,year_7,year_8,year_9,year_10,year_15,year_20,year_30
IG_CORPORATE_SNR,AAA,0.0001,0.0003,0.0006,0.0010,0.0015,0.0022,0.0030,0.0040,0.0048,0.0055,0.0100,0.0160,0.0260
IG_CORPORATE_SNR,AA,0.0003,0.0008,0.0015,0.0024,0.0035,0.0048,0.0062,0.0078,0.0095,0.0110,0.0190,0.0300,0.0470
IG_CORPORATE_SNR,A,0.0005,0.0015,0.0030,0.0050,0.0075,0.0100,0.0130,0.0160,0.0195,0.0230,0.0400,0.0600,0.0900
IG_CORPORATE_SNR,BBB,0.0015,0.0040,0.0075,0.0120,0.0170,0.0225,0.0285,0.0350,0.0415,0.0480,0.0800,0.1150,0.1600
IG_CORPORATE_SNR,BB,0.0080,0.0200,0.0370,0.0550,0.0740,0.0920,0.1100,0.1270,0.1430,0.1580,0.2200,0.2700,0.3400
IG_CORPORATE_SNR,B,0.0350,0.0700,0.1050,0.1400,0.1700,0.1980,0.2230,0.2460,0.2670,0.2860,0.3500,0.4000,0.4600
```

File: `assumptions/tables/bma_downgrade_floors.csv` -- same format.

**Note:** The values above are **illustrative placeholders**. Actual BMA published values must be substituted when available in machine-readable form. The table structure and code are designed to accept the real values as a drop-in replacement.

### 4.3 Interpolation for Non-Tabulated Years

The BMA table provides rates at years 1-10, 15, 20, 30. For intermediate years (e.g., year 12), linear interpolation on cumulative rates is used:

```python
def cumulative_rate(self, year: int) -> float:
    """
    Return cumulative D&D rate for a given projection year.
    Interpolates linearly between table knot points.
    For years > 30, extrapolate flat (use year_30 value).
    """
    KNOTS = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 15, 20, 30]
    values = [
        self.year_1, self.year_2, self.year_3, self.year_4, self.year_5,
        self.year_6, self.year_7, self.year_8, self.year_9, self.year_10,
        self.year_15, self.year_20, self.year_30,
    ]

    if year <= 0:
        return 0.0
    if year >= 30:
        return self.year_30

    # Find bracketing knots
    for i in range(len(KNOTS) - 1):
        if KNOTS[i] <= year <= KNOTS[i + 1]:
            t_lo, t_hi = KNOTS[i], KNOTS[i + 1]
            v_lo, v_hi = values[i], values[i + 1]
            frac = (year - t_lo) / (t_hi - t_lo)
            return v_lo + frac * (v_hi - v_lo)

    return self.year_30  # Fallback (should not reach here)
```

### 4.4 Table Loader with Validation

```python
class CreditCostTable(BaseModel):
    """Complete D&D table for one cost type (default or downgrade)."""
    cost_type: Literal["default", "downgrade"]
    rows: list[CreditCostRow]
    _index: Dict[tuple, CreditCostRow] = {}   # (asset_class, rating) -> row

    def model_post_init(self, __context):
        """Build lookup index and validate completeness."""
        self._index = {}
        for row in self.rows:
            key = (row.asset_class, row.rating)
            if key in self._index:
                raise ValueError(
                    f"Duplicate D&D entry for {key} in {self.cost_type} table"
                )
            self._index[key] = row

    def lookup(self, asset_class: AssetClass, rating: CreditRating,
               year: int) -> float:
        """
        Return cumulative D&D rate for given asset_class, rating, year.
        Raises KeyError with descriptive message if not found.
        """
        key = (asset_class, rating)
        if key not in self._index:
            raise KeyError(
                f"No {self.cost_type} cost entry for asset_class={asset_class}, "
                f"rating={rating}. Check assumptions/tables/bma_{self.cost_type}_floors.csv"
            )
        return self._index[key].cumulative_rate(year)


def load_credit_table(filepath: str, cost_type: str) -> CreditCostTable:
    """
    Load a BMA D&D floor table from CSV.
    Validates every row via Pydantic. Fails loudly on any error.

    @rule_ref: Para 28(24) - BMA tables are floors
    """
    df = pd.read_csv(filepath)
    required_cols = {"asset_class", "rating", "year_1", "year_2", "year_3",
                     "year_4", "year_5", "year_6", "year_7", "year_8",
                     "year_9", "year_10", "year_15", "year_20", "year_30"}
    missing = required_cols - set(df.columns)
    if missing:
        raise ValueError(f"Missing columns in {filepath}: {missing}")

    rows = [CreditCostRow(**row) for row in df.to_dict(orient="records")]
    return CreditCostTable(cost_type=cost_type, rows=rows)
```

### 4.5 Floor Enforcement

When the `challenge/` module compares company assumptions against the benchmark:

```python
def validate_company_dd_floors(
    company_table: CreditCostTable,
    bma_floor_table: CreditCostTable,
) -> list[str]:
    """
    Check that company D&D assumptions >= BMA floors at every point.
    Returns list of violation descriptions (empty = pass).

    @rule_ref: Para 28(24) - assumptions cannot be lower than BMA tables
    """
    violations = []
    for key, bma_row in bma_floor_table._index.items():
        if key not in company_table._index:
            violations.append(f"Missing entry for {key}")
            continue
        co_row = company_table._index[key]
        for year in [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 15, 20, 30]:
            co_rate = co_row.cumulative_rate(year)
            bma_rate = bma_row.cumulative_rate(year)
            if co_rate < bma_rate - 1e-8:  # Tolerance for float comparison
                violations.append(
                    f"{key} year {year}: company={co_rate:.6f} < "
                    f"BMA floor={bma_rate:.6f}"
                )
    return violations
```

---

## 5. Application in Projection

### 5.1 When D&D Is Applied

D&D is applied in **Step 2** of the projection engine (per Technical Spec section 4.1), AFTER collecting gross asset cash flows and BEFORE netting against liabilities. The sequence within each projection period `t`:

```
1. Collect gross asset cash flows for period t
2. >>> APPLY D&D DEDUCTION <<<
3. Add interest on cash balance
4. Net against liability cash flows
5. Reinvest surplus or liquidate to cover shortfall
6. Log audit trail
```

### 5.2 Per-Asset D&D Calculation (Pseudocode)

```python
@rule_ref("Para 28(22)-(26)", "D&D credit cost deduction")
def compute_dd_adjustment(
    asset: AssetModelPoint,
    period: int,
    gross_cf: float,
    default_table: CreditCostTable,
    downgrade_table: CreditCostTable,
    valuation_year: int,
    vintage: str,   # "pre_2024" or "post_2023"
) -> DDResult:
    """
    Compute D&D-adjusted cash flow for one asset in one period.

    Args:
        asset: The asset model point
        period: Projection year (1, 2, 3, ...)
        gross_cf: Unadjusted cash flow for this period (coupons + principal)
        default_table: BMA default floor table
        downgrade_table: BMA downgrade floor table
        valuation_year: Year of valuation (e.g., 2024)
        vintage: Whether asset is pre-2024 business (phase-in applies)

    Returns:
        DDResult with adjusted_cf, default_cost, downgrade_cost, survival factor
    """
    if asset.is_derivative:
        # Derivatives exempt -- fully collateralized
        return DDResult(
            adjusted_cf=gross_cf,
            default_cost=0.0,
            downgrade_cost=0.0,
            combined_survival=1.0,
        )

    # Look up cumulative rates
    cum_def_t   = default_table.lookup(asset.asset_class, asset.rating, period)
    cum_def_t_1 = default_table.lookup(asset.asset_class, asset.rating, period - 1)
    cum_dg_t    = downgrade_table.lookup(asset.asset_class, asset.rating, period)
    cum_dg_t_1  = downgrade_table.lookup(asset.asset_class, asset.rating, period - 1)

    # Incremental survival factors
    inc_def_surv = (1 - cum_def_t) / (1 - cum_def_t_1)
    inc_dg_surv  = (1 - cum_dg_t) / (1 - cum_dg_t_1)

    # Phase-in for downgrade
    phi = phase_in_factor(valuation_year, vintage)
    inc_dg_surv_eff = 1 - phi * (1 - inc_dg_surv)

    # Cumulative survival (tracked on the asset across periods)
    # asset._cum_def_survival and asset._cum_dg_survival are
    # initialized to 1.0 at period 0
    asset._cum_def_survival *= inc_def_surv
    asset._cum_dg_survival  *= inc_dg_surv_eff

    combined_survival = asset._cum_def_survival * asset._cum_dg_survival

    # Compute dollar costs for audit trail
    default_cost   = gross_cf * (1 - asset._cum_def_survival)
    downgrade_cost = gross_cf * (asset._cum_def_survival - combined_survival)
    adjusted_cf    = gross_cf * combined_survival

    return DDResult(
        adjusted_cf=adjusted_cf,
        default_cost=default_cost,
        downgrade_cost=downgrade_cost,
        combined_survival=combined_survival,
    )
```

### 5.3 Portfolio-Level Aggregation

```python
@rule_ref("Para 28(22)", "Aggregate D&D across portfolio")
def apply_dd_to_portfolio(
    active_assets: list[AssetModelPoint],
    period: int,
    gross_cfs: dict[str, float],  # asset_id -> gross cash flow
    default_table: CreditCostTable,
    downgrade_table: CreditCostTable,
    valuation_year: int,
) -> PortfolioDDResult:
    """
    Apply D&D to all assets in the portfolio for one period.
    Returns total adjusted cash flow and detailed breakdown.
    """
    results = {}
    total_gross = 0.0
    total_adjusted = 0.0
    total_default_cost = 0.0
    total_downgrade_cost = 0.0

    for asset in active_assets:
        gross_cf = gross_cfs.get(asset.asset_id, 0.0)
        vintage = "pre_2024" if asset.issue_date <= date(2023, 12, 31) else "post_2023"

        dd = compute_dd_adjustment(
            asset, period, gross_cf,
            default_table, downgrade_table,
            valuation_year, vintage,
        )
        results[asset.asset_id] = dd
        total_gross += gross_cf
        total_adjusted += dd.adjusted_cf
        total_default_cost += dd.default_cost
        total_downgrade_cost += dd.downgrade_cost

    return PortfolioDDResult(
        period=period,
        total_gross_cf=total_gross,
        total_adjusted_cf=total_adjusted,
        total_default_cost=total_default_cost,
        total_downgrade_cost=total_downgrade_cost,
        asset_details=results,
    )
```

### 5.4 D&D Same Across All 9 Scenarios

D&D cost assumptions are **identical across all 9 interest rate scenarios**. The D&D rates depend only on credit rating and duration, not on the interest rate path. The same D&D tables are passed to all 9 scenario projections.

---

## 6. Phase-In Transitional Arrangement

### 6.1 Formula

```python
@rule_ref("Para 28(25)-(26)", "Downgrade phase-in transitional arrangement")
def phase_in_factor(valuation_year: int, vintage: str) -> float:
    """
    Return the downgrade cost phase-in factor.

    Args:
        valuation_year: The year of valuation (e.g., 2024, 2025, ...)
        vintage: "pre_2024" for business in force as of Dec 31, 2023
                 "post_2023" for new business written after Dec 31, 2023

    Returns:
        Float between 0.0 and 1.0 representing the fraction of downgrade
        cost to apply.
    """
    # New business always gets full downgrade costs
    if vintage == "post_2023":
        return 1.0

    # Pre-2024 business: phased in over 5 years
    PHASE_IN = {
        2024: 0.20,
        2025: 0.40,
        2026: 0.60,
        2027: 0.80,
    }
    # 2028 and beyond: fully phased in
    if valuation_year >= 2028:
        return 1.0

    if valuation_year in PHASE_IN:
        return PHASE_IN[valuation_year]

    # Before 2024: SBA regime not yet in effect
    raise ValueError(
        f"Valuation year {valuation_year} predates SBA regime (2024+)"
    )
```

### 6.2 Phase-In Examples

**Example 1: Valuation year 2026, pre-2024 BBB bond**
- Default survival factor at year 5: `1 - 0.0170 = 0.9830` (full, no phase-in)
- Downgrade survival factor at year 5: `1 - 0.0085 = 0.9915` (hypothetical)
- Phase-in factor: `0.60`
- Effective downgrade survival: `1 - 0.60 * (1 - 0.9915) = 1 - 0.60 * 0.0085 = 0.9949`
- Combined survival: `0.9830 * 0.9949 = 0.9780`
- Without phase-in: `0.9830 * 0.9915 = 0.9746`
- Difference: 0.34% of cash flow retained due to phase-in relief

**Example 2: Valuation year 2026, post-2023 BBB bond**
- Phase-in factor: `1.00` (no relief for new business)
- Combined survival: `0.9830 * 0.9915 = 0.9746`

---

## 7. Special Cases

### 7.1 Derivatives: Exempt

```python
DERIVATIVE_TYPES = {"SWAP", "SWAPTION", "OPTION", "FORWARD", "FUTURES"}

def is_derivative(asset: AssetModelPoint) -> bool:
    """Derivatives are exempt from D&D -- fully collateralized per BMA spec."""
    return asset.asset_type in DERIVATIVE_TYPES
```

**Rationale:** Per BMA guidance and Pythora spec (section 3.3.8.11), derivatives are considered fully collateralized and therefore not subject to credit default/downgrade risk in the SBA projection.

### 7.2 Reinvested Assets

Assets purchased during the projection (from reinvestment of excess cash flows) ARE subject to D&D costs. Key assumptions:

```python
@rule_ref("Para 28(22)", "D&D for reinvested assets")
def dd_for_reinvested_asset(
    new_asset: AssetModelPoint,
    purchase_period: int,
    current_period: int,
    default_table: CreditCostTable,
    downgrade_table: CreditCostTable,
    valuation_year: int,
) -> DDResult:
    """
    For reinvested assets:
    - years_since_inception = current_period - purchase_period
    - Rating = rating assigned at purchase (per reinvestment strategy)
    - Vintage = "post_2023" (always, since purchased during projection)
    - D&D starts from inception (not from T=0)
    """
    years_held = current_period - purchase_period
    if years_held <= 0:
        return DDResult(adjusted_cf=gross_cf, default_cost=0.0,
                        downgrade_cost=0.0, combined_survival=1.0)

    # Use years_held as the lookup period, not current_period
    # Vintage is always "post_2023" for reinvested assets
    return compute_dd_adjustment(
        new_asset, years_held, gross_cf,
        default_table, downgrade_table,
        valuation_year, vintage="post_2023",
    )
```

**Reinvested asset rating assumption:** The reinvestment strategy module determines the rating of newly purchased assets. For the benchmark model's conservative approach, reinvested assets are assumed to carry the weighted-average rating of the existing portfolio in that asset class.

### 7.3 Tier 3 (Limited Basis) Assets

For Tier 3 assets, the BMA requires the downgrade component (uncertainty adjustment) to be at least 1 standard deviation of the default cost distribution:

```python
@rule_ref("Para 28(15)-(16)", "Enhanced D&D for limited basis assets")
def tier3_downgrade_floor(
    standard_downgrade_rate: float,
    default_rate_std_dev: float,
) -> float:
    """
    For Tier 3 assets, downgrade rate must be >= 1 std dev of defaults.
    Returns the higher of the standard downgrade rate and the std dev floor.
    """
    return max(standard_downgrade_rate, default_rate_std_dev)
```

**Note:** The standard deviation of default costs is not directly available from BMA tables. It must either be:
1. Published by BMA as a separate table (preferred), or
2. Estimated from historical default data (e.g., Moody's Annual Default Study).

Until BMA publishes this, the benchmark model will use a configurable multiplier on the base default rate as a proxy (default: 1.5x the base default rate).

### 7.4 Rating Changes During Projection

The benchmark model uses a **static rating assumption**: each asset retains its initial rating throughout the projection. This is the standard approach because:
- BMA scenarios are interest rate scenarios, not credit scenarios
- Credit migration is captured probabilistically via the D&D tables (the downgrade component represents the expected loss from migration)
- One-notch downgrade is handled separately as a stress test (Handbook E5.6h(ii))

However, for the **one-notch downgrade stress test**, all ratings shift down one notch and D&D is recalculated with the new ratings.

### 7.5 Cash and Cash Equivalents

Cash earns the risk-free rate and has **zero D&D cost** (no credit risk). In the D&D lookup, cash is excluded:

```python
DD_EXEMPT_TYPES = {"CASH", "SWAP", "SWAPTION", "OPTION", "FORWARD", "FUTURES"}
```

### 7.6 Sovereign / Government Bonds

Government bonds from G7 countries rated AAA/AA are typically assigned very low D&D rates in BMA tables. The table accommodates this -- the rates are simply small. No special-case logic is needed; the table values drive the calculation.

---

## 8. Worked Example

### 8.1 Setup

A single BBB-rated investment-grade corporate bond (senior unsecured), 5-year projection.

| Parameter | Value |
|---|---|
| Asset ID | BOND_001 |
| Asset Class | IG_CORPORATE_SNR |
| Rating | BBB |
| Par Value | $1,000,000 |
| Market Value (T=0) | $1,010,000 |
| Coupon Rate | 5.0% annual |
| Coupon Frequency | Annual |
| Maturity | 5 years from valuation |
| Tier | 1 |
| Issue Date | Before Dec 31, 2023 (pre-2024 vintage) |
| Valuation Year | 2026 |

### 8.2 BMA Table Values (Illustrative)

**Default cumulative rates (IG Corporate Senior Unsecured, BBB):**

| Year | Cumulative Default Rate |
|---|---|
| 0 | 0.0000 |
| 1 | 0.0015 |
| 2 | 0.0040 |
| 3 | 0.0075 |
| 4 | 0.0120 |
| 5 | 0.0170 |

**Downgrade cumulative rates (IG Corporate Senior Unsecured, BBB):**

| Year | Cumulative Downgrade Rate |
|---|---|
| 0 | 0.0000 |
| 1 | 0.0010 |
| 2 | 0.0025 |
| 3 | 0.0045 |
| 4 | 0.0068 |
| 5 | 0.0095 |

### 8.3 Period-by-Period Calculation

**Phase-in factor:** Valuation year 2026, vintage pre-2024 --> phi = 0.60

#### Period 1

```
Gross cash flow = $1,000,000 * 5.0% = $50,000 (coupon only)

Default:
  cum_def(1) = 0.0015, cum_def(0) = 0.0000
  inc_def_surv(1) = (1 - 0.0015) / (1 - 0.0000) = 0.99850
  cum_def_survival(1) = 0.99850

Downgrade:
  cum_dg(1) = 0.0010, cum_dg(0) = 0.0000
  inc_dg_surv(1) = (1 - 0.0010) / (1 - 0.0000) = 0.99900
  inc_dg_surv_eff(1) = 1 - 0.60 * (1 - 0.99900) = 1 - 0.60 * 0.00100 = 0.99940
  cum_dg_survival(1) = 0.99940

Combined survival(1) = 0.99850 * 0.99940 = 0.99790

Adjusted CF(1) = $50,000 * 0.99790 = $49,895.05
Default cost    = $50,000 * (1 - 0.99850) = $50,000 * 0.00150 = $75.00
Downgrade cost  = $50,000 * (0.99850 - 0.99790) = $50,000 * 0.00060 = $30.00
Total D&D cost  = $105.00
```

#### Period 2

```
Gross cash flow = $50,000

Default:
  cum_def(2) = 0.0040, cum_def(1) = 0.0015
  inc_def_surv(2) = (1 - 0.0040) / (1 - 0.0015) = 0.99600 / 0.99850 = 0.99750
  cum_def_survival(2) = 0.99850 * 0.99750 = 0.99601

Downgrade:
  cum_dg(2) = 0.0025, cum_dg(1) = 0.0010
  inc_dg_surv(2) = (1 - 0.0025) / (1 - 0.0010) = 0.99750 / 0.99900 = 0.99850
  inc_dg_surv_eff(2) = 1 - 0.60 * (1 - 0.99850) = 1 - 0.60 * 0.00150 = 0.99910
  cum_dg_survival(2) = 0.99940 * 0.99910 = 0.99850

Combined survival(2) = 0.99601 * 0.99850 = 0.99452

Adjusted CF(2) = $50,000 * 0.99452 = $49,725.89
Default cost    = $50,000 * (1 - 0.99601) = $199.50
Downgrade cost  = $50,000 * (0.99601 - 0.99452) = $74.61
Total D&D cost  = $274.11
```

#### Period 3

```
Gross cash flow = $50,000

Default:
  cum_def(3) = 0.0075, cum_def(2) = 0.0040
  inc_def_surv(3) = (1 - 0.0075) / (1 - 0.0040) = 0.99250 / 0.99600 = 0.99649
  cum_def_survival(3) = 0.99601 * 0.99649 = 0.99252

Downgrade:
  cum_dg(3) = 0.0045, cum_dg(2) = 0.0025
  inc_dg_surv(3) = (1 - 0.0045) / (1 - 0.0025) = 0.99550 / 0.99750 = 0.99800
  inc_dg_surv_eff(3) = 1 - 0.60 * (1 - 0.99800) = 1 - 0.60 * 0.00200 = 0.99880
  cum_dg_survival(3) = 0.99850 * 0.99880 = 0.99730

Combined survival(3) = 0.99252 * 0.99730 = 0.98984

Adjusted CF(3) = $50,000 * 0.98984 = $49,492.07
Default cost    = $50,000 * (1 - 0.99252) = $374.00
Downgrade cost  = $50,000 * (0.99252 - 0.98984) = $134.07
Total D&D cost  = $508.07
```

#### Period 4

```
Gross cash flow = $50,000

Default:
  cum_def(4) = 0.0120, cum_def(3) = 0.0075
  inc_def_surv(4) = (1 - 0.0120) / (1 - 0.0075) = 0.98800 / 0.99250 = 0.99547
  cum_def_survival(4) = 0.99252 * 0.99547 = 0.98802

Downgrade:
  cum_dg(4) = 0.0068, cum_dg(3) = 0.0045
  inc_dg_surv(4) = (1 - 0.0068) / (1 - 0.0045) = 0.99320 / 0.99550 = 0.99769
  inc_dg_surv_eff(4) = 1 - 0.60 * (1 - 0.99769) = 1 - 0.60 * 0.00231 = 0.99861
  cum_dg_survival(4) = 0.99730 * 0.99861 = 0.99592

Combined survival(4) = 0.98802 * 0.99592 = 0.98399

Adjusted CF(4) = $50,000 * 0.98399 = $49,199.44
Default cost    = $50,000 * (1 - 0.98802) = $599.00
Downgrade cost  = $50,000 * (0.98802 - 0.98399) = $201.44
Total D&D cost  = $800.44
```

#### Period 5 (Maturity: Coupon + Principal)

```
Gross cash flow = $50,000 + $1,000,000 = $1,050,000

Default:
  cum_def(5) = 0.0170, cum_def(4) = 0.0120
  inc_def_surv(5) = (1 - 0.0170) / (1 - 0.0120) = 0.98300 / 0.98800 = 0.99494
  cum_def_survival(5) = 0.98802 * 0.99494 = 0.98302

Downgrade:
  cum_dg(5) = 0.0095, cum_dg(4) = 0.0068
  inc_dg_surv(5) = (1 - 0.0095) / (1 - 0.0068) = 0.99050 / 0.99320 = 0.99728
  inc_dg_surv_eff(5) = 1 - 0.60 * (1 - 0.99728) = 1 - 0.60 * 0.00272 = 0.99837
  cum_dg_survival(5) = 0.99592 * 0.99837 = 0.99430

Combined survival(5) = 0.98302 * 0.99430 = 0.97742

Adjusted CF(5) = $1,050,000 * 0.97742 = $1,026,294.39
Default cost    = $1,050,000 * (1 - 0.98302) = $17,829.00
Downgrade cost  = $1,050,000 * (0.98302 - 0.97742) = $5,876.61
Total D&D cost  = $23,705.61
```

### 8.4 Summary Table

| Period | Gross CF | Cum Def Surv | Cum DG Surv (eff) | Combined Surv | Adjusted CF | Default Cost | Downgrade Cost | Total D&D |
|---|---|---|---|---|---|---|---|---|
| 1 | $50,000 | 0.99850 | 0.99940 | 0.99790 | $49,895.05 | $75.00 | $30.00 | $105.00 |
| 2 | $50,000 | 0.99601 | 0.99850 | 0.99452 | $49,725.89 | $199.50 | $74.61 | $274.11 |
| 3 | $50,000 | 0.99252 | 0.99730 | 0.98984 | $49,492.07 | $374.00 | $134.07 | $508.07 |
| 4 | $50,000 | 0.98802 | 0.99592 | 0.98399 | $49,199.44 | $599.00 | $201.44 | $800.44 |
| 5 | $1,050,000 | 0.98302 | 0.99430 | 0.97742 | $1,026,294.39 | $17,829.00 | $5,876.61 | $23,705.61 |
| **Total** | **$1,250,000** | | | | **$1,224,606.84** | **$19,076.50** | **$6,316.73** | **$25,393.23** |

**Total D&D impact:** $25,393.23 reduction from $1,250,000 gross cash flows = **2.03%** total credit cost over 5 years.

**If phase-in were 100% (fully phased in, or post-2023 business):** The downgrade cost would be approximately $10,528 (= $6,317 / 0.60), making the total D&D approximately $29,604 or **2.37%**.

### 8.5 Verification Cross-Check

Independent verification using the Pythora multiplicative formula with a single combined rate:

```
Combined annual rate X = annualized from cumulative = 1 - (1 - 0.0170 - 0.60*0.0095)^(1/5)
                       = 1 - (1 - 0.0227)^0.2
                       = 1 - 0.99541^0.2
                       = 1 - 0.99908
                       = 0.000459 per year (approximate)

This approximation breaks down because it conflates the two components.
The incremental approach used above is more precise.
```

The worked example confirms that the incremental approach is the correct implementation path for the benchmark model.

---

## 9. Edge Cases and Validation Rules

### 9.1 Validation at Load Time

```python
def validate_credit_tables(
    default_table: CreditCostTable,
    downgrade_table: CreditCostTable,
) -> None:
    """
    Validate D&D tables at load time. Raises ValueError on any violation.
    Called by the engine before starting any projection.
    """
    for table in [default_table, downgrade_table]:
        for row in table.rows:
            # V1: Cumulative rates must be monotonically non-decreasing
            rates = [row.cumulative_rate(y) for y in range(1, 31)]
            for i in range(1, len(rates)):
                if rates[i] < rates[i - 1] - 1e-10:
                    raise ValueError(
                        f"{table.cost_type} table {row.asset_class}/{row.rating}: "
                        f"cumulative rate decreases at year {i+1} "
                        f"({rates[i]:.6f} < {rates[i-1]:.6f})"
                    )

            # V2: Cumulative rates must be in [0, 1)
            for y in [1, 5, 10, 20, 30]:
                r = row.cumulative_rate(y)
                if not (0.0 <= r < 1.0):
                    raise ValueError(
                        f"{table.cost_type} table {row.asset_class}/{row.rating}: "
                        f"rate at year {y} = {r:.6f} outside [0, 1)"
                    )

            # V3: Higher ratings must have lower or equal default rates
            # (cross-row validation -- handled separately)

    # V4: Both tables must cover the same (asset_class, rating) combinations
    def_keys = set(default_table._index.keys())
    dg_keys = set(downgrade_table._index.keys())
    if def_keys != dg_keys:
        missing_in_dg = def_keys - dg_keys
        missing_in_def = dg_keys - def_keys
        raise ValueError(
            f"Default and downgrade tables cover different keys. "
            f"Missing in downgrade: {missing_in_dg}. "
            f"Missing in default: {missing_in_def}."
        )
```

### 9.2 Edge Case: Cumulative Rate Approaches 1.0

If `cum_def_rate(t-1)` is very close to 1.0, the incremental survival formula divides by a near-zero number. Guard:

```python
def safe_incremental_survival(cum_rate_t: float, cum_rate_t_1: float) -> float:
    """Compute incremental survival with guard against division by near-zero."""
    denominator = 1.0 - cum_rate_t_1
    if denominator < 1e-12:
        # Asset is essentially in default -- survival is 0
        return 0.0
    return (1.0 - cum_rate_t) / denominator
```

### 9.3 Edge Case: Asset Matures Mid-Period

For the benchmark model with annual projection steps, an asset that matures in period `t` has its final cash flow (coupon + principal) occur in period `t`. The D&D factor applied is the cumulative survival through period `t`. No special handling needed -- the formulas apply uniformly.

### 9.4 Edge Case: Rating Not in Table

If an asset's rating is not found in the D&D lookup table, the engine must **fail loudly**:

```python
# KeyError raised by CreditCostTable.lookup() with descriptive message.
# The engine catches this and halts the run, reporting the missing rating.
# NO silent fallback to a "similar" rating or zero cost.
```

### 9.5 Edge Case: Year Exceeds Table Maximum

For projections longer than 30 years, cumulative rates are held flat at the year-30 value:

```python
# In CreditCostRow.cumulative_rate():
if year >= 30:
    return self.year_30
```

This is conservative (underestimates D&D for very long durations) but acceptable for the benchmark model where most SBA projections are under 30 years.

### 9.6 Edge Case: Zero Coupon Bonds / Discount Bonds

D&D applies to ALL cash flows including principal. A zero-coupon bond has no coupon cash flows, but its principal at maturity is still reduced by the cumulative survival factor. No special handling needed.

### 9.7 Edge Case: Amortizing Securities

For amortizing bonds (MBS, ABS, amortizing corporates), D&D applies to each period's scheduled cash flow (interest + principal return). The survival factor at period `t` applies to whatever gross cash flow occurs in period `t`, regardless of the coupon/principal split.

---

## 10. Implementation Notes

### 10.1 Data Structures

```python
from dataclasses import dataclass
from typing import Optional


@dataclass(frozen=True)
class DDResult:
    """Result of D&D calculation for one asset in one period."""
    adjusted_cf: float         # Cash flow after D&D deduction
    default_cost: float        # Dollar amount of default deduction
    downgrade_cost: float      # Dollar amount of downgrade deduction
    combined_survival: float   # Cumulative combined survival factor


@dataclass(frozen=True)
class PortfolioDDResult:
    """Aggregated D&D result for the entire portfolio in one period."""
    period: int
    total_gross_cf: float
    total_adjusted_cf: float
    total_default_cost: float
    total_downgrade_cost: float
    asset_details: dict[str, DDResult]   # asset_id -> DDResult


@dataclass
class AssetDDState:
    """
    Mutable state tracking cumulative survival factors for one asset
    across projection periods. Initialized at period 0 and updated
    each period by compute_dd_adjustment().
    """
    asset_id: str
    cum_def_survival: float = 1.0
    cum_dg_survival: float = 1.0
```

### 10.2 Performance Considerations

- D&D calculation is O(N) per period where N = number of active assets.
- With typical portfolios of 1,000-10,000 assets and 30-60 periods, this is at most 600,000 lookups per scenario -- trivially fast in pure Python.
- The D&D lookup table is small (at most ~200 rows for 10 asset classes x 20 ratings) and fits in a Python dict. No database or caching needed.
- All 9 scenarios use the same D&D tables, so tables are loaded once and shared.

### 10.3 Audit Trail

Every D&D calculation must be logged to the audit trail (SQLite + JSON Lines):

```python
# Example audit record for one asset, one period:
{
    "run_id": "RUN-2026-001",
    "period": 3,
    "asset_id": "BOND_001",
    "rule_ref": "Para 28(22)-(26)",
    "gross_cf": 50000.00,
    "cum_def_rate": 0.0075,
    "cum_dg_rate": 0.0045,
    "phase_in": 0.60,
    "inc_def_survival": 0.99649,
    "inc_dg_survival_eff": 0.99880,
    "cum_def_survival": 0.99252,
    "cum_dg_survival_eff": 0.99730,
    "combined_survival": 0.98984,
    "adjusted_cf": 49492.07,
    "default_cost": 374.00,
    "downgrade_cost": 134.07
}
```

### 10.4 Testing Strategy

| Test Type | Description |
|---|---|
| **Unit: table loading** | Load valid CSV, verify Pydantic validation catches bad data (negative rates, non-monotonic, missing columns) |
| **Unit: interpolation** | Verify linear interpolation at year 12, 17, 25; verify year 30+ flat extrapolation |
| **Unit: incremental survival** | Verify incremental survival formula against hand calculation |
| **Unit: phase-in** | Verify phase_in_factor for each year 2024-2028+, both vintages |
| **Unit: derivative exemption** | Confirm D&D returns identity (survival=1.0) for derivative types |
| **Unit: edge cases** | Near-1.0 cumulative rates, missing ratings (expect error), zero-coupon bonds |
| **Integration: worked example** | Run the BBB 5-year bond from Section 8 through the code, compare all intermediate values to the spec |
| **Integration: golden test** | Run against BMA illustrative calculation, verify D&D impact matches expected output |
| **Property: monotonicity** | Hypothesis test: for any valid inputs, combined_survival(t) <= combined_survival(t-1) |
| **Property: floor enforcement** | Hypothesis test: adjusted_cf <= gross_cf for all non-derivative assets |

### 10.5 Module Dependencies

```
assumptions/credit_tables.py
    imports: pydantic, pandas, pathlib
    exports: CreditCostTable, CreditCostRow, CreditRating, AssetClass,
             load_credit_table, validate_credit_tables

projection/credit_costs.py
    imports: assumptions.credit_tables, rules.decorators (@rule_ref)
    exports: compute_dd_adjustment, apply_dd_to_portfolio,
             phase_in_factor, DDResult, PortfolioDDResult, AssetDDState
    NO imports from: curves/ (no QuantLib dependency)

challenge/dd_comparison.py  (future)
    imports: assumptions.credit_tables
    exports: validate_company_dd_floors
```

### 10.6 Configuration

D&D table file paths are specified in the run configuration:

```yaml
# config/run_config.yaml
assumptions:
  default_table: "assumptions/tables/bma_default_floors.csv"
  downgrade_table: "assumptions/tables/bma_downgrade_floors.csv"
  # Override with company-specific tables for challenge mode:
  # company_default_table: "submissions/acme/dd_defaults.csv"
  # company_downgrade_table: "submissions/acme/dd_downgrades.csv"
```

### 10.7 Open Items

| # | Item | Status | Notes |
|---|---|---|---|
| 1 | BMA published D&D floor tables in machine-readable format | **Pending** | Placeholder values used; swap in real values when available |
| 2 | Tier 3 standard deviation floor | **Pending** | Need BMA guidance or historical data source for std dev of default rates |
| 3 | D&D for structured securities (MBS, ABS, CLO) | **Pending** | May need separate table structure reflecting tranche seniority |
| 4 | Interaction with prepayment assumptions | **Design needed** | For MBS: if prepayment reduces par faster, D&D cost also decreases proportionally |
| 5 | Pythora validation comparison | **Pending** | Once implemented, compare our incremental approach against Pythora's (1-X)^t to confirm numerical consistency |
