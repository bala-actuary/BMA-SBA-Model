# Algorithm Spec 07: Data Schemas and Sample Inputs

## Document Control

| Field | Value |
|---|---|
| **Spec ID** | ALGO-07 |
| **Target Modules** | `model_points/schemas.py`, `model_points/assets.py`, `model_points/liabilities.py`, `model_points/portfolio.py`, `assumptions/loader.py`, `assumptions/economic.py`, `assumptions/credit.py` |
| **BMA Rule References** | Rules Sched XXV Para 28(9)-(16), 28(22)-(26); Handbook E2, E4, E10 |
| **Status** | Implementation-Ready Draft |
| **Date** | March 2026 |
| **Cross-References** | [ALGO-002](02_Projection_Engine.md) (projection engine), [ALGO-003](03_Credit_Costs_DD.md) (D&D credit costs), [ALGO-004](04_Spread_Cap_Enforcement.md) (spread cap) |

---

## 1. Overview

### 1.1 Purpose

This specification defines every data schema consumed by the BMA SBA Benchmark Model -- asset model points, liability EPL cash flows, market data, credit cost tables, and run configuration. It provides complete Pydantic v2 models with field types, validators, and cross-field constraints. A self-consistent sample dataset is included that can serve as the golden integration test input.

### 1.2 Design Principles

| Principle | Implementation |
|---|---|
| **Fail loudly** | Every schema is a Pydantic `BaseModel` with strict validation. Invalid data raises `ValidationError` with field-level detail. No silent defaults for required fields. |
| **Everything is a DataFrame** | Loaders parse CSV/Excel into lists of Pydantic models, then convert to pandas DataFrames for downstream consumption. |
| **Immutable inputs** | All model classes are `frozen=True`. Once loaded, input data cannot be mutated. |
| **Rule traceability** | Schema constraints map to specific BMA paragraphs (noted in field descriptions). |

### 1.3 Module Responsibilities

| Module | Responsibility |
|---|---|
| `model_points/schemas.py` | All Pydantic model definitions (this spec) |
| `model_points/assets.py` | Load asset CSV/Excel into `List[AssetModelPoint]`, convert to DataFrame |
| `model_points/liabilities.py` | Load EPL CSV/Excel into `List[EPLCashFlow]`, convert to DataFrame |
| `model_points/portfolio.py` | Combine assets + liabilities, cross-validate (currency match, tier caps) |
| `assumptions/loader.py` | Load versioned assumption sets from `assumptions/tables/` |
| `assumptions/economic.py` | Risk-free curve loader and validator |
| `assumptions/credit.py` | D&D table loader with cumulative-to-incremental conversion |

---

## 2. Asset Model Point Schema

### 2.1 Enumerations

```python
# model_points/schemas.py

from enum import Enum
from typing import Optional, List, Literal
from datetime import date
from decimal import Decimal
from pydantic import BaseModel, Field, field_validator, model_validator


class AssetType(str, Enum):
    """BMA asset type classification.

    Maps to QuantLib instrument types in curves/bond_pricing.py.
    See Tech Spec Section 2.3 for QuantLib mapping.
    """
    GOVT_BOND = "GOVT_BOND"              # Government/sovereign bonds (Tier 1)
    MUNI_BOND = "MUNI_BOND"              # Municipal bonds (Tier 1)
    CORP_BOND_IG = "CORP_BOND_IG"        # Investment-grade corporate bonds (Tier 1)
    CORP_BOND_HY = "CORP_BOND_HY"        # High-yield corporate bonds (Tier 3)
    CALLABLE_BOND = "CALLABLE_BOND"      # Callable fixed-rate bonds (Tier 1 or 2)
    FLOATING_RATE = "FLOATING_RATE"      # Floating-rate notes (Tier 1 or 2)
    AMORTIZING = "AMORTIZING"            # Amortizing/sinking fund bonds (Tier 1 or 2)
    MBS_AGENCY = "MBS_AGENCY"            # Agency MBS (Tier 2)
    MBS_NON_AGENCY = "MBS_NON_AGENCY"    # Non-agency MBS (Tier 2)
    ABS = "ABS"                          # Asset-backed securities (Tier 2)
    CLO = "CLO"                          # Collateralized loan obligations (Tier 2)
    PREFERRED_STOCK = "PREFERRED_STOCK"  # IG preferred stock (Tier 2)
    MORTGAGE_LOAN = "MORTGAGE_LOAN"      # Residential/commercial mortgage loans (Tier 2)
    COMMERCIAL_RE = "COMMERCIAL_RE"      # Commercial real estate (Tier 3)
    CASH = "CASH"                        # Cash and cash equivalents (Tier 1)


class CreditRating(str, Enum):
    """Credit rating enum ordered from highest to lowest quality.

    Used for D&D table lookups (ALGO-03) and disinvestment waterfall
    priority (ALGO-02b).
    """
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


# Investment-grade boundary: BBB- and above
INVESTMENT_GRADE_RATINGS = {
    CreditRating.AAA, CreditRating.AA_PLUS, CreditRating.AA,
    CreditRating.AA_MINUS, CreditRating.A_PLUS, CreditRating.A,
    CreditRating.A_MINUS, CreditRating.BBB_PLUS, CreditRating.BBB,
    CreditRating.BBB_MINUS,
}
```

### 2.2 Supporting Types

```python
class CallDate(BaseModel):
    """A single call option in a callable bond's call schedule."""
    model_config = {"frozen": True}

    call_date: date = Field(description="Date on which the issuer may call the bond")
    call_price: float = Field(
        gt=0,
        description="Call price as percentage of par (e.g., 100.0 = at par, 102.0 = 2% premium)"
    )

    @field_validator("call_price")
    @classmethod
    def call_price_reasonable(cls, v: float) -> float:
        if v < 90.0 or v > 120.0:
            raise ValueError(f"Call price {v} outside reasonable range [90, 120]")
        return v


class AmortPayment(BaseModel):
    """A single scheduled amortization payment."""
    model_config = {"frozen": True}

    payment_date: date = Field(description="Date of scheduled principal payment")
    principal_amount: float = Field(
        gt=0,
        description="Scheduled principal repayment amount"
    )
```

### 2.3 AssetModelPoint

```python
class AssetModelPoint(BaseModel):
    """One record per asset in the starting portfolio.

    This is the primary input schema for the asset side.
    Each field is validated individually; cross-field constraints
    are enforced by model_validator.

    Rule Ref: Para 28(13)-(16) for tier classification.
    Rule Ref: Para 28(9) for asset cash flow projection inputs.
    """
    model_config = {"frozen": True}

    # --- Identification ---
    asset_id: str = Field(
        min_length=1, max_length=50,
        description="Unique identifier within the portfolio"
    )
    asset_type: AssetType = Field(
        description="BMA asset type classification"
    )
    issuer_name: str = Field(
        min_length=1, max_length=200,
        description="Issuer or counterparty name (for reporting)"
    )

    # --- Valuation ---
    par_value: float = Field(
        gt=0,
        description="Face/par/notional value. Must be > 0."
    )
    book_value: float = Field(
        gt=0,
        description="Carrying (book) value on the insurer's balance sheet"
    )
    market_value: float = Field(
        gt=0,
        description="Mark-to-market value at valuation date"
    )

    # --- Coupon / Cash Flow ---
    coupon_rate: float = Field(
        ge=0.0, le=0.30,
        description="Annual coupon rate as decimal (e.g., 0.05 = 5%). "
                    "Range [0, 0.30]. CASH assets use 0.0."
    )
    coupon_frequency: int = Field(
        description="Coupon payments per year. "
                    "Allowed: 0 (zero-coupon), 1 (annual), 2 (semi-annual), 4 (quarterly)"
    )

    # --- Dates ---
    issue_date: date = Field(
        description="Original issue date of the instrument"
    )
    maturity_date: date = Field(
        description="Final maturity date. CASH uses a far-future sentinel (9999-12-31)."
    )

    # --- Classification ---
    currency: str = Field(
        min_length=3, max_length=3, pattern=r"^[A-Z]{3}$",
        description="ISO 4217 currency code (e.g., 'USD', 'BMD')"
    )
    rating: CreditRating = Field(
        description="Credit rating. CASH and GOVT_BOND typically AAA."
    )
    tier: Literal[1, 2, 3] = Field(
        description="BMA tier classification. "
                    "1 = unrestricted, 2 = BMA pre-approval, 3 = limited basis (10% cap, cannot sell). "
                    "Rule Ref: Para 28(13)-(16)"
    )

    # --- Optional: Callable bonds ---
    is_callable: bool = Field(
        default=False,
        description="True if the bond has an embedded call option"
    )
    call_schedule: Optional[List[CallDate]] = Field(
        default=None,
        description="Call dates and prices. Required if is_callable=True."
    )

    # --- Optional: Amortizing instruments ---
    is_amortizing: bool = Field(
        default=False,
        description="True if the bond has a scheduled amortization"
    )
    amort_schedule: Optional[List[AmortPayment]] = Field(
        default=None,
        description="Scheduled principal payments. Required if is_amortizing=True."
    )

    # --- Optional: MBS/ABS ---
    prepayment_speed: Optional[float] = Field(
        default=None, ge=0.0, le=1000.0,
        description="PSA prepayment speed (e.g., 150 = 150% PSA). "
                    "Required for MBS_AGENCY and MBS_NON_AGENCY."
    )

    # --- Metadata ---
    sector: str = Field(
        default="",
        description="Sector classification for reporting (e.g., 'Financials', 'Energy')"
    )
    bma_approved: bool = Field(
        default=True,
        description="True if the asset has BMA pre-approval. "
                    "Tier 2 assets MUST have bma_approved=True. "
                    "Rule Ref: Para 28(14)"
    )

    # ---- Field-level validators ----

    @field_validator("coupon_frequency")
    @classmethod
    def validate_coupon_frequency(cls, v: int) -> int:
        if v not in (0, 1, 2, 4):
            raise ValueError(
                f"coupon_frequency must be 0, 1, 2, or 4; got {v}"
            )
        return v

    # ---- Cross-field validators ----

    @model_validator(mode="after")
    def validate_cross_field(self) -> "AssetModelPoint":
        errors = []

        # Maturity must be after issue
        if self.maturity_date <= self.issue_date:
            errors.append(
                f"maturity_date ({self.maturity_date}) must be after "
                f"issue_date ({self.issue_date})"
            )

        # CASH must have coupon_rate=0, coupon_frequency=0
        if self.asset_type == AssetType.CASH:
            if self.coupon_rate != 0.0:
                errors.append("CASH assets must have coupon_rate=0.0")
            if self.coupon_frequency != 0:
                errors.append("CASH assets must have coupon_frequency=0")

        # Callable bonds must have a call schedule
        if self.is_callable and not self.call_schedule:
            errors.append("is_callable=True requires a non-empty call_schedule")
        if not self.is_callable and self.call_schedule:
            errors.append("call_schedule provided but is_callable=False")

        # Amortizing bonds must have an amort schedule
        if self.is_amortizing and not self.amort_schedule:
            errors.append("is_amortizing=True requires a non-empty amort_schedule")
        if not self.is_amortizing and self.amort_schedule:
            errors.append("amort_schedule provided but is_amortizing=False")

        # MBS must have prepayment speed
        if self.asset_type in (AssetType.MBS_AGENCY, AssetType.MBS_NON_AGENCY):
            if self.prepayment_speed is None:
                errors.append(
                    f"{self.asset_type.value} requires prepayment_speed"
                )

        # Tier 2 must be BMA-approved
        if self.tier == 2 and not self.bma_approved:
            errors.append(
                "Tier 2 assets must have bma_approved=True "
                "(Rule Ref: Para 28(14))"
            )

        # Tier 3 cannot be investment-grade (except COMMERCIAL_RE)
        if self.tier == 3 and self.asset_type != AssetType.COMMERCIAL_RE:
            if self.rating in INVESTMENT_GRADE_RATINGS:
                errors.append(
                    "Tier 3 non-CRE assets must be below investment grade "
                    "(Rule Ref: Para 28(15))"
                )

        # Tier consistency with asset type
        TIER_1_ONLY = {AssetType.GOVT_BOND, AssetType.MUNI_BOND, AssetType.CASH}
        TIER_2_ONLY = {
            AssetType.MBS_AGENCY, AssetType.MBS_NON_AGENCY,
            AssetType.ABS, AssetType.CLO, AssetType.PREFERRED_STOCK,
            AssetType.MORTGAGE_LOAN,
        }
        TIER_3_ONLY = {AssetType.CORP_BOND_HY, AssetType.COMMERCIAL_RE}

        if self.asset_type in TIER_1_ONLY and self.tier != 1:
            errors.append(
                f"{self.asset_type.value} must be Tier 1; got Tier {self.tier}"
            )
        if self.asset_type in TIER_2_ONLY and self.tier != 2:
            errors.append(
                f"{self.asset_type.value} must be Tier 2; got Tier {self.tier}"
            )
        if self.asset_type in TIER_3_ONLY and self.tier != 3:
            errors.append(
                f"{self.asset_type.value} must be Tier 3; got Tier {self.tier}"
            )

        if errors:
            raise ValueError("; ".join(errors))
        return self
```

### 2.4 Tier Classification Rules

The following table defines the mandatory tier for each asset type, per Para 28(13)-(16):

| Asset Type | Required Tier | Rationale |
|---|---|---|
| `GOVT_BOND` | 1 | Government bonds are unrestricted |
| `MUNI_BOND` | 1 | Municipal bonds are unrestricted |
| `CORP_BOND_IG` | 1 | Investment-grade corporates are unrestricted |
| `CASH` | 1 | Cash is unrestricted |
| `CALLABLE_BOND` | 1 or 2 | Depends on underlying credit; validator accepts both |
| `FLOATING_RATE` | 1 or 2 | Depends on underlying credit; validator accepts both |
| `AMORTIZING` | 1 or 2 | Depends on underlying credit; validator accepts both |
| `MBS_AGENCY` | 2 | Structured securities require BMA approval |
| `MBS_NON_AGENCY` | 2 | Structured securities require BMA approval |
| `ABS` | 2 | Structured securities require BMA approval |
| `CLO` | 2 | Structured securities require BMA approval |
| `PREFERRED_STOCK` | 2 | Requires BMA approval |
| `MORTGAGE_LOAN` | 2 | Requires BMA approval |
| `CORP_BOND_HY` | 3 | Below-IG; limited basis |
| `COMMERCIAL_RE` | 3 | Limited basis |

**Note on `CALLABLE_BOND`, `FLOATING_RATE`, `AMORTIZING`:** These types can be either Tier 1 (if the underlying issuer is IG and no BMA approval needed) or Tier 2 (if structured or requiring approval). The model validator does not enforce a single tier for these types; the user must classify correctly.

---

## 3. Liability EPL Schema

### 3.1 EPLCashFlow Model

```python
class EPLCashFlow(BaseModel):
    """External Projected Liabilities - one record per projection period.

    Liability cash flows are treated as fixed hurdles in the SBA.
    The projection engine (ALGO-002) must generate sufficient asset
    cash flows to meet these outflows each period.

    Rule Ref: Para 28(9) - cash flow matching requirement.
    """
    model_config = {"frozen": True}

    period: int = Field(
        ge=1,
        description="Projection year, sequential from 1. "
                    "Period 1 = first year after valuation date."
    )
    liability_cf: float = Field(
        description="Net liability cash outflow for this period. "
                    "Positive = cash leaving the portfolio (claims, benefits, expenses). "
                    "Negative values are allowed (e.g., premium inflows) but unusual for SBA."
    )
    block_id: str = Field(
        default="DEFAULT",
        min_length=1, max_length=100,
        description="Liability block identifier. Each block is analyzed independently. "
                    "Use 'DEFAULT' for a single-block portfolio."
    )
    currency: str = Field(
        default="USD",
        min_length=3, max_length=3, pattern=r"^[A-Z]{3}$",
        description="ISO 4217 currency code. Must match asset portfolio currency."
    )
    description: str = Field(
        default="",
        description="Optional description of the cash flow (for reporting)"
    )
```

### 3.2 EPL Collection Validator

```python
class EPLCashFlowSet(BaseModel):
    """Validated collection of EPL cash flows for one liability block.

    Enforces contiguity (no gaps in period sequence) and non-zero total.
    """
    model_config = {"frozen": True}

    cash_flows: List[EPLCashFlow] = Field(
        min_length=1,
        description="Ordered list of EPL records"
    )

    @model_validator(mode="after")
    def validate_collection(self) -> "EPLCashFlowSet":
        errors = []

        # Group by block_id
        blocks: dict[str, list[EPLCashFlow]] = {}
        for cf in self.cash_flows:
            blocks.setdefault(cf.block_id, []).append(cf)

        for block_id, block_cfs in blocks.items():
            # Sort by period
            sorted_cfs = sorted(block_cfs, key=lambda x: x.period)
            periods = [cf.period for cf in sorted_cfs]

            # Must start at 1
            if periods[0] != 1:
                errors.append(
                    f"Block '{block_id}': periods must start at 1, "
                    f"got {periods[0]}"
                )

            # Must be contiguous (no gaps)
            expected = list(range(periods[0], periods[-1] + 1))
            if periods != expected:
                missing = set(expected) - set(periods)
                errors.append(
                    f"Block '{block_id}': non-contiguous periods. "
                    f"Missing: {sorted(missing)}"
                )

            # No duplicate periods
            if len(periods) != len(set(periods)):
                errors.append(
                    f"Block '{block_id}': duplicate periods detected"
                )

            # Total liability must be positive
            total = sum(cf.liability_cf for cf in sorted_cfs)
            if total <= 0:
                errors.append(
                    f"Block '{block_id}': total liability_cf must be > 0, "
                    f"got {total:.2f}"
                )

            # All records in block must have same currency
            currencies = {cf.currency for cf in sorted_cfs}
            if len(currencies) > 1:
                errors.append(
                    f"Block '{block_id}': mixed currencies {currencies}"
                )

        if errors:
            raise ValueError("; ".join(errors))
        return self

    @property
    def t_max(self) -> int:
        """Maximum projection period across all blocks."""
        return max(cf.period for cf in self.cash_flows)

    @property
    def block_ids(self) -> List[str]:
        """Unique block identifiers."""
        return sorted({cf.block_id for cf in self.cash_flows})
```

---

## 4. Market Data Schema

### 4.1 Risk-Free Curve Input

```python
class RiskFreeRatePoint(BaseModel):
    """A single point on the risk-free yield curve.

    The base risk-free curve is the starting point for all 9 BMA
    scenarios. Scenario shifts are applied by curves/scenario_curves.py
    (ALGO-001).
    """
    model_config = {"frozen": True}

    tenor_years: float = Field(
        gt=0.0, le=100.0,
        description="Tenor in years (e.g., 0.25 = 3-month, 30.0 = 30-year)"
    )
    rate: float = Field(
        ge=-0.02, le=0.25,
        description="Annualized continuously-compounded rate as decimal. "
                    "Range [-0.02, 0.25] to catch data errors."
    )
    instrument_type: str = Field(
        default="SWAP",
        description="Source instrument type: 'SWAP', 'GOVT', 'DEPOSIT', 'FUTURE'. "
                    "Used by QuantLib curve bootstrap (ALGO-001)."
    )
    date: date = Field(
        description="Observation date (must equal valuation date for all points)"
    )


class RiskFreeCurve(BaseModel):
    """Complete risk-free yield curve for one valuation date."""
    model_config = {"frozen": True}

    points: List[RiskFreeRatePoint] = Field(
        min_length=5,
        description="Curve points. Minimum 5 tenors required for interpolation."
    )

    @model_validator(mode="after")
    def validate_curve(self) -> "RiskFreeCurve":
        errors = []

        tenors = [p.tenor_years for p in self.points]
        dates = {p.date for p in self.points}

        # All points must have the same date
        if len(dates) > 1:
            errors.append(f"Mixed observation dates: {dates}")

        # No duplicate tenors
        if len(tenors) != len(set(tenors)):
            errors.append("Duplicate tenors detected")

        # Must include short end (< 1y) and long end (>= 20y)
        if min(tenors) >= 1.0:
            errors.append(
                "Curve must include at least one tenor < 1 year "
                f"(shortest is {min(tenors)}y)"
            )
        if max(tenors) < 20.0:
            errors.append(
                "Curve must include at least one tenor >= 20 years "
                f"(longest is {max(tenors)}y)"
            )

        if errors:
            raise ValueError("; ".join(errors))
        return self

    @property
    def valuation_date(self) -> date:
        return self.points[0].date

    @property
    def max_tenor(self) -> float:
        return max(p.tenor_years for p in self.points)
```

---

## 5. Credit Cost Table Schema

### 5.1 D&D Table Format

As specified in [ALGO-003](03_Credit_Costs_DD.md), the model requires two separate tables: one for cumulative default rates and one for cumulative downgrade rates. Both follow the same schema.

```python
class CreditCostTable(BaseModel):
    """BMA D&D cumulative rate table.

    Provides cumulative default or downgrade rates by rating and year.
    These are the BMA FLOOR values -- company assumptions must meet
    or exceed them (Para 28(24)).

    The table is indexed by CreditRating (rows) and projection year (columns).
    Rates are cumulative probabilities over the given time horizon.
    """
    model_config = {"frozen": True}

    table_type: Literal["DEFAULT", "DOWNGRADE"] = Field(
        description="Identifies whether this is a default or downgrade table"
    )
    version: str = Field(
        min_length=1,
        description="Version identifier (e.g., '2024Q4', 'BMA_2025')"
    )
    years: List[int] = Field(
        min_length=1,
        description="Column headers: projection years (e.g., [1, 2, 3, 5, 10, 20, 30])"
    )
    rates: dict[str, List[float]] = Field(
        description="Mapping of rating string to list of cumulative rates, "
                    "aligned with the 'years' field. "
                    "E.g., {'AAA': [0.0001, 0.0003, ...], 'AA+': [0.0002, ...]}"
    )

    @model_validator(mode="after")
    def validate_table(self) -> "CreditCostTable":
        errors = []

        # Years must be positive and sorted ascending
        if self.years != sorted(self.years) or any(y <= 0 for y in self.years):
            errors.append("years must be positive integers in ascending order")

        # Every rating row must have same length as years
        for rating_str, rate_list in self.rates.items():
            if len(rate_list) != len(self.years):
                errors.append(
                    f"Rating '{rating_str}': expected {len(self.years)} rates, "
                    f"got {len(rate_list)}"
                )

            # Rates must be non-negative and non-decreasing (cumulative)
            for i, r in enumerate(rate_list):
                if r < 0.0:
                    errors.append(
                        f"Rating '{rating_str}', year {self.years[i]}: "
                        f"negative rate {r}"
                    )
                if i > 0 and r < rate_list[i - 1]:
                    errors.append(
                        f"Rating '{rating_str}': cumulative rates must be "
                        f"non-decreasing (year {self.years[i-1]}={rate_list[i-1]} "
                        f"> year {self.years[i]}={r})"
                    )

            # Rates must be < 1.0 (except possibly D which can be 1.0)
            if rating_str != "D" and any(r >= 1.0 for r in rate_list):
                errors.append(
                    f"Rating '{rating_str}': cumulative rate >= 1.0 "
                    f"(only 'D' may reach 1.0)"
                )

            # Validate rating string is a known CreditRating
            try:
                CreditRating(rating_str)
            except ValueError:
                errors.append(f"Unknown rating: '{rating_str}'")

        if errors:
            raise ValueError("; ".join(errors))
        return self
```

### 5.2 Phase-In Configuration

```python
class PhaseInConfig(BaseModel):
    """Transitional phase-in schedule for downgrade costs.

    Rule Ref: Para 28(25)-(26).
    Phase-in applies ONLY to the downgrade component, ONLY for
    business in force as of Dec 31, 2023.
    """
    model_config = {"frozen": True}

    schedule: dict[int, float] = Field(
        description="Mapping of valuation year to phase-in factor. "
                    "E.g., {2024: 0.20, 2025: 0.40, 2026: 0.60, 2027: 0.80, 2028: 1.00}"
    )
    cutoff_date: date = Field(
        default=date(2023, 12, 31),
        description="Business in force before this date gets the phase-in. "
                    "New business after this date: phase_in = 1.0 always."
    )

    @field_validator("schedule")
    @classmethod
    def validate_schedule(cls, v: dict[int, float]) -> dict[int, float]:
        for year, factor in v.items():
            if not (0.0 <= factor <= 1.0):
                raise ValueError(
                    f"Phase-in factor for {year} must be in [0, 1]; got {factor}"
                )
        return v
```

---

## 6. Configuration Schema

### 6.1 RunConfig

```python
class RunConfig(BaseModel):
    """Immutable configuration for a single model run.

    Frozen after creation. All inputs are hashed for reproducibility.
    """
    model_config = {"frozen": True}

    # --- Run identification ---
    run_id: str = Field(
        description="Unique run identifier (auto-generated UUID if not provided)"
    )
    company_id: str = Field(
        min_length=1,
        description="Company identifier (e.g., 'ACME_RE')"
    )
    valuation_date: date = Field(
        description="Valuation date (e.g., 2024-12-31)"
    )
    rule_year: int = Field(
        ge=2024,
        description="BMA rule year to apply (e.g., 2024, 2025). "
                    "Determines which versioned rule module is loaded."
    )

    # --- Input file paths ---
    asset_file: str = Field(
        description="Path to asset portfolio CSV/Excel file"
    )
    epl_file: str = Field(
        description="Path to EPL (liability) CSV/Excel file"
    )
    market_data_file: str = Field(
        description="Path to risk-free curve CSV file"
    )

    # --- Assumption versions ---
    assumption_version: str = Field(
        default="latest",
        description="Assumption set version from assumptions/tables/manifest.yaml"
    )

    # --- Projection parameters ---
    projection_currency: str = Field(
        default="USD",
        min_length=3, max_length=3, pattern=r"^[A-Z]{3}$",
        description="Currency for all calculations"
    )
    reinvestment_strategy: Literal[
        "pro_rata", "cash_only", "duration_match"
    ] = Field(
        default="cash_only",
        description="Reinvestment strategy. 'cash_only' is the conservative benchmark default."
    )
    transaction_cost_enabled: bool = Field(
        default=True,
        description="Whether to apply bid-ask transaction costs"
    )
    rate_floor: float = Field(
        default=0.0,
        ge=-0.01,
        description="Minimum interest rate floor applied to scenario curves. "
                    "Default 0.0 (rates cannot go negative per BMA guidance). "
                    "Subject to BMA confirmation."
    )

    # --- Output ---
    output_dir: str = Field(
        description="Directory for output files (Excel, Parquet, audit logs)"
    )
```

### 6.2 YAML Configuration File Format

The `RunConfig` can be loaded from a YAML file. Example structure:

```yaml
# config/run_config.yaml
run_id: "RUN_2024Q4_ACME_001"
company_id: "ACME_RE"
valuation_date: "2024-12-31"
rule_year: 2024

# Input file paths (relative to project root or absolute)
asset_file: "data/input/acme_assets_2024Q4.csv"
epl_file: "data/input/acme_epl_2024Q4.csv"
market_data_file: "data/input/rfr_curve_2024Q4.csv"

# Assumption set
assumption_version: "2024Q4"

# Projection parameters
projection_currency: "USD"
reinvestment_strategy: "cash_only"
transaction_cost_enabled: true
rate_floor: 0.0

# Output
output_dir: "data/output/ACME_2024Q4"
```

---

## 7. Validation Rules

### 7.1 Within-Schema Validation

These are enforced by the Pydantic `field_validator` and `model_validator` decorators defined in Sections 2-6. Summary:

| Rule | Schema | Validator Type |
|---|---|---|
| `par_value > 0` | AssetModelPoint | field (gt=0) |
| `coupon_frequency in {0, 1, 2, 4}` | AssetModelPoint | field_validator |
| `maturity_date > issue_date` | AssetModelPoint | model_validator |
| CASH must have coupon_rate=0, frequency=0 | AssetModelPoint | model_validator |
| Callable requires call_schedule | AssetModelPoint | model_validator |
| MBS requires prepayment_speed | AssetModelPoint | model_validator |
| Tier 2 requires bma_approved=True | AssetModelPoint | model_validator |
| Tier 3 non-CRE must be sub-IG | AssetModelPoint | model_validator |
| Asset type must match required tier | AssetModelPoint | model_validator |
| EPL periods start at 1 | EPLCashFlowSet | model_validator |
| EPL periods contiguous, no gaps | EPLCashFlowSet | model_validator |
| Total liability_cf > 0 per block | EPLCashFlowSet | model_validator |
| Curve tenors include short + long end | RiskFreeCurve | model_validator |
| D&D rates non-negative, non-decreasing | CreditCostTable | model_validator |

### 7.2 Cross-Schema Validation (Portfolio Level)

These are enforced in `model_points/portfolio.py` after all individual schemas are loaded:

```python
class PortfolioValidator:
    """Cross-schema validation rules applied after loading all inputs."""

    @staticmethod
    def validate(
        assets: List[AssetModelPoint],
        epl: EPLCashFlowSet,
        curve: RiskFreeCurve,
        config: RunConfig,
    ) -> List[str]:
        """Returns list of error messages. Empty list = valid."""
        errors = []

        # 1. Currency consistency
        asset_currencies = {a.currency for a in assets}
        epl_currencies = {cf.currency for cf in epl.cash_flows}
        all_currencies = asset_currencies | epl_currencies | {config.projection_currency}
        if len(all_currencies) > 1:
            errors.append(
                f"Currency mismatch: assets={asset_currencies}, "
                f"liabilities={epl_currencies}, config={config.projection_currency}"
            )

        # 2. Unique asset IDs
        ids = [a.asset_id for a in assets]
        if len(ids) != len(set(ids)):
            dupes = [x for x in ids if ids.count(x) > 1]
            errors.append(f"Duplicate asset_id values: {set(dupes)}")

        # 3. Tier 3 cap: limited-basis assets <= 10% of total MV
        #    Rule Ref: Para 28(16)
        total_mv = sum(a.market_value for a in assets)
        tier3_mv = sum(a.market_value for a in assets if a.tier == 3)
        if total_mv > 0 and (tier3_mv / total_mv) > 0.10:
            errors.append(
                f"Tier 3 assets = {tier3_mv/total_mv:.1%} of portfolio MV, "
                f"exceeds 10% cap (Rule Ref: Para 28(16))"
            )

        # 4. Valuation date consistency
        if curve.valuation_date != config.valuation_date:
            errors.append(
                f"Curve valuation date ({curve.valuation_date}) != "
                f"run config valuation date ({config.valuation_date})"
            )

        # 5. Curve must extend to at least T_max
        if curve.max_tenor < epl.t_max:
            errors.append(
                f"Curve max tenor ({curve.max_tenor}y) < "
                f"liability horizon ({epl.t_max}y). "
                f"Extrapolation required or extend curve."
            )

        # 6. Asset maturity vs liability horizon (warning, not error)
        max_asset_mat = max(
            (a.maturity_date for a in assets if a.asset_type != AssetType.CASH),
            default=config.valuation_date,
        )
        liability_end = date(
            config.valuation_date.year + epl.t_max,
            config.valuation_date.month,
            config.valuation_date.day,
        )
        # This is informational -- not a hard error

        return errors
```

### 7.3 Error Handling Policy

All validation errors are collected and reported together (not fail-on-first). The run is aborted if any errors are present. The error messages include:
- Field name and value that failed
- The constraint that was violated
- The BMA rule reference where applicable

Example error output:
```
ValidationError: 3 errors in portfolio validation:
  [1] asset_id='BOND_099': Tier 2 assets must have bma_approved=True (Rule Ref: Para 28(14))
  [2] Block 'ANNUITY': non-contiguous periods. Missing: {15, 16}
  [3] Tier 3 assets = 14.2% of portfolio MV, exceeds 10% cap (Rule Ref: Para 28(16))
```

---

## 8. Sample Input Dataset

This section provides a complete, self-consistent sample dataset suitable for integration testing. All values are realistic but simplified. The dataset covers all 3 asset tiers, multiple asset types, and includes a 30-year liability payout pattern.

### 8.1 Sample Asset Portfolio

**Valuation date:** 2024-12-31

| asset_id | asset_type | issuer_name | par_value | book_value | market_value | coupon_rate | coupon_frequency | issue_date | maturity_date | currency | rating | tier | is_callable | is_amortizing | prepayment_speed | sector | bma_approved |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| GOVT_001 | GOVT_BOND | US Treasury | 5000000 | 4975000 | 5050000 | 0.0425 | 2 | 2022-06-15 | 2032-06-15 | USD | AAA | 1 | false | false | | Government | true |
| GOVT_002 | GOVT_BOND | US Treasury | 3000000 | 2980000 | 2950000 | 0.035 | 2 | 2021-01-15 | 2031-01-15 | USD | AAA | 1 | false | false | | Government | true |
| MUNI_001 | MUNI_BOND | State of Texas GO | 2000000 | 2010000 | 2025000 | 0.04 | 2 | 2023-03-01 | 2043-03-01 | USD | AA+ | 1 | false | false | | Municipal | true |
| CORP_001 | CORP_BOND_IG | Apple Inc | 4000000 | 3985000 | 4080000 | 0.0475 | 2 | 2023-09-15 | 2033-09-15 | USD | AA+ | 1 | false | false | | Technology | true |
| CORP_002 | CORP_BOND_IG | JPMorgan Chase | 3000000 | 2990000 | 3015000 | 0.05 | 2 | 2022-03-01 | 2029-03-01 | USD | A+ | 1 | false | false | | Financials | true |
| CORP_003 | CORP_BOND_IG | Johnson & Johnson | 2500000 | 2510000 | 2530000 | 0.0375 | 2 | 2020-07-15 | 2030-07-15 | USD | AAA | 1 | false | false | | Healthcare | true |
| CORP_004 | CORP_BOND_IG | Duke Energy | 2000000 | 1995000 | 1980000 | 0.055 | 2 | 2024-01-15 | 2044-01-15 | USD | BBB+ | 1 | false | false | | Utilities | true |
| MBS_001 | MBS_AGENCY | FNMA Pool 2024-01 | 3000000 | 2985000 | 2970000 | 0.045 | 12 | 2024-02-01 | 2054-02-01 | USD | AA+ | 2 | false | true | 150 | Structured | true |
| CORP_HY_001 | CORP_BOND_HY | Carnival Corp | 1500000 | 1480000 | 1450000 | 0.085 | 2 | 2023-06-15 | 2030-06-15 | USD | BB+ | 3 | false | false | | Consumer | true |
| CRE_001 | COMMERCIAL_RE | 100 Main St Office | 500000 | 500000 | 490000 | 0.065 | 4 | 2023-01-01 | 2033-01-01 | USD | BBB | 3 | false | false | | Real Estate | true |
| CASH_001 | CASH | Cash Account | 2000000 | 2000000 | 2000000 | 0.0 | 0 | 2024-12-31 | 9999-12-31 | USD | AAA | 1 | false | false | | Cash | true |

**Portfolio summary:**
- **Total market value:** $28,540,000
- **Tier 1 market value:** $23,630,000 (82.8%)
- **Tier 2 market value:** $2,970,000 (10.4%)
- **Tier 3 market value:** $1,940,000 (6.8%) -- under the 10% cap
- **Average coupon (ex-cash):** ~4.7%
- **Currency:** USD (uniform)

### 8.2 Sample Liability EPL (30-Year Payout Pattern)

This represents a single annuity block with a typical declining payout pattern: higher outflows in early years (active claims), tapering over 30 years.

| period | liability_cf | block_id | currency | description |
|---|---|---|---|---|
| 1 | 1800000 | ANNUITY_BLK | USD | Year 1 benefits + expenses |
| 2 | 1750000 | ANNUITY_BLK | USD | Year 2 benefits + expenses |
| 3 | 1700000 | ANNUITY_BLK | USD | Year 3 benefits + expenses |
| 4 | 1650000 | ANNUITY_BLK | USD | Year 4 benefits |
| 5 | 1600000 | ANNUITY_BLK | USD | Year 5 benefits |
| 6 | 1500000 | ANNUITY_BLK | USD | Year 6 benefits |
| 7 | 1400000 | ANNUITY_BLK | USD | Year 7 benefits |
| 8 | 1300000 | ANNUITY_BLK | USD | Year 8 benefits |
| 9 | 1200000 | ANNUITY_BLK | USD | Year 9 benefits |
| 10 | 1100000 | ANNUITY_BLK | USD | Year 10 benefits |
| 11 | 1000000 | ANNUITY_BLK | USD | Year 11 benefits |
| 12 | 900000 | ANNUITY_BLK | USD | Year 12 benefits |
| 13 | 800000 | ANNUITY_BLK | USD | Year 13 benefits |
| 14 | 700000 | ANNUITY_BLK | USD | Year 14 benefits |
| 15 | 600000 | ANNUITY_BLK | USD | Year 15 benefits |
| 16 | 500000 | ANNUITY_BLK | USD | Year 16 benefits |
| 17 | 450000 | ANNUITY_BLK | USD | Year 17 benefits |
| 18 | 400000 | ANNUITY_BLK | USD | Year 18 benefits |
| 19 | 350000 | ANNUITY_BLK | USD | Year 19 benefits |
| 20 | 300000 | ANNUITY_BLK | USD | Year 20 benefits |
| 21 | 250000 | ANNUITY_BLK | USD | Year 21 benefits |
| 22 | 220000 | ANNUITY_BLK | USD | Year 22 benefits |
| 23 | 190000 | ANNUITY_BLK | USD | Year 23 benefits |
| 24 | 160000 | ANNUITY_BLK | USD | Year 24 benefits |
| 25 | 130000 | ANNUITY_BLK | USD | Year 25 benefits |
| 26 | 100000 | ANNUITY_BLK | USD | Year 26 benefits |
| 27 | 80000 | ANNUITY_BLK | USD | Year 27 benefits |
| 28 | 60000 | ANNUITY_BLK | USD | Year 28 benefits |
| 29 | 40000 | ANNUITY_BLK | USD | Year 29 benefits |
| 30 | 20000 | ANNUITY_BLK | USD | Year 30 final payment |

**Liability summary:**
- **Total undiscounted outflow:** $22,250,000
- **Weighted average duration:** ~8.7 years
- **Block:** ANNUITY_BLK (single block)
- **Currency:** USD

### 8.3 Sample Risk-Free Curve

Base risk-free yield curve as of 2024-12-31. Rates are annualized continuously-compounded. This is a realistic upward-sloping USD curve.

| tenor_years | rate | instrument_type | date |
|---|---|---|---|
| 0.25 | 0.0440 | DEPOSIT | 2024-12-31 |
| 0.50 | 0.0435 | DEPOSIT | 2024-12-31 |
| 1.00 | 0.0425 | SWAP | 2024-12-31 |
| 2.00 | 0.0410 | SWAP | 2024-12-31 |
| 3.00 | 0.0400 | SWAP | 2024-12-31 |
| 4.00 | 0.0395 | SWAP | 2024-12-31 |
| 5.00 | 0.0390 | SWAP | 2024-12-31 |
| 7.00 | 0.0395 | SWAP | 2024-12-31 |
| 10.00 | 0.0405 | SWAP | 2024-12-31 |
| 12.00 | 0.0410 | SWAP | 2024-12-31 |
| 15.00 | 0.0415 | SWAP | 2024-12-31 |
| 20.00 | 0.0420 | SWAP | 2024-12-31 |
| 25.00 | 0.0422 | SWAP | 2024-12-31 |
| 30.00 | 0.0425 | SWAP | 2024-12-31 |
| 40.00 | 0.0425 | SWAP | 2024-12-31 |

**Curve characteristics:**
- Short end (0.25y): 4.40% -- mildly inverted relative to 5y
- Belly (5y): 3.90% -- curve trough
- Long end (30y): 4.25% -- upward-sloping beyond 5y
- 15 tenors spanning 0.25y to 40y

### 8.4 Sample D&D Tables

#### Cumulative Default Rates (BMA Floor)

Rates represent cumulative probability of default over the given time horizon. Values are illustrative and based on long-run historical averages scaled to BMA guidance.

| rating | year_1 | year_2 | year_3 | year_5 | year_10 | year_20 | year_30 |
|---|---|---|---|---|---|---|---|
| AAA | 0.0001 | 0.0003 | 0.0006 | 0.0013 | 0.0040 | 0.0120 | 0.0200 |
| AA+ | 0.0002 | 0.0005 | 0.0010 | 0.0022 | 0.0065 | 0.0180 | 0.0300 |
| AA | 0.0003 | 0.0008 | 0.0015 | 0.0030 | 0.0090 | 0.0250 | 0.0400 |
| AA- | 0.0004 | 0.0010 | 0.0020 | 0.0040 | 0.0120 | 0.0330 | 0.0520 |
| A+ | 0.0005 | 0.0013 | 0.0025 | 0.0055 | 0.0170 | 0.0450 | 0.0700 |
| A | 0.0006 | 0.0016 | 0.0032 | 0.0070 | 0.0220 | 0.0580 | 0.0900 |
| A- | 0.0008 | 0.0020 | 0.0042 | 0.0095 | 0.0300 | 0.0750 | 0.1100 |
| BBB+ | 0.0012 | 0.0030 | 0.0060 | 0.0140 | 0.0420 | 0.1000 | 0.1450 |
| BBB | 0.0018 | 0.0045 | 0.0090 | 0.0200 | 0.0600 | 0.1350 | 0.1900 |
| BBB- | 0.0030 | 0.0075 | 0.0150 | 0.0320 | 0.0900 | 0.1800 | 0.2450 |
| BB+ | 0.0060 | 0.0150 | 0.0300 | 0.0620 | 0.1500 | 0.2700 | 0.3400 |
| BB | 0.0090 | 0.0220 | 0.0440 | 0.0900 | 0.2000 | 0.3300 | 0.4000 |
| BB- | 0.0130 | 0.0320 | 0.0630 | 0.1250 | 0.2600 | 0.3900 | 0.4600 |
| B+ | 0.0200 | 0.0480 | 0.0920 | 0.1700 | 0.3200 | 0.4500 | 0.5200 |
| B | 0.0300 | 0.0700 | 0.1300 | 0.2200 | 0.3800 | 0.5100 | 0.5800 |
| B- | 0.0450 | 0.1000 | 0.1750 | 0.2800 | 0.4400 | 0.5700 | 0.6400 |
| CCC | 0.0800 | 0.1600 | 0.2500 | 0.3600 | 0.5200 | 0.6500 | 0.7100 |
| CC | 0.1500 | 0.2500 | 0.3500 | 0.4500 | 0.6000 | 0.7200 | 0.7800 |
| C | 0.3000 | 0.4000 | 0.5000 | 0.5800 | 0.7000 | 0.8000 | 0.8500 |
| D | 1.0000 | 1.0000 | 1.0000 | 1.0000 | 1.0000 | 1.0000 | 1.0000 |

#### Cumulative Downgrade Rates (BMA Floor)

Rates represent cumulative spread widening loss from credit migration (downgrade without default).

| rating | year_1 | year_2 | year_3 | year_5 | year_10 | year_20 | year_30 |
|---|---|---|---|---|---|---|---|
| AAA | 0.0000 | 0.0001 | 0.0002 | 0.0005 | 0.0015 | 0.0040 | 0.0065 |
| AA+ | 0.0001 | 0.0002 | 0.0004 | 0.0010 | 0.0030 | 0.0070 | 0.0110 |
| AA | 0.0001 | 0.0003 | 0.0006 | 0.0014 | 0.0040 | 0.0095 | 0.0150 |
| AA- | 0.0002 | 0.0004 | 0.0008 | 0.0018 | 0.0055 | 0.0130 | 0.0200 |
| A+ | 0.0002 | 0.0005 | 0.0010 | 0.0025 | 0.0075 | 0.0180 | 0.0280 |
| A | 0.0003 | 0.0007 | 0.0014 | 0.0032 | 0.0095 | 0.0230 | 0.0350 |
| A- | 0.0004 | 0.0009 | 0.0018 | 0.0042 | 0.0125 | 0.0300 | 0.0450 |
| BBB+ | 0.0005 | 0.0012 | 0.0025 | 0.0058 | 0.0170 | 0.0400 | 0.0580 |
| BBB | 0.0007 | 0.0018 | 0.0036 | 0.0082 | 0.0240 | 0.0540 | 0.0760 |
| BBB- | 0.0012 | 0.0030 | 0.0060 | 0.0130 | 0.0360 | 0.0720 | 0.0980 |
| BB+ | 0.0025 | 0.0060 | 0.0120 | 0.0250 | 0.0600 | 0.1050 | 0.1350 |
| BB | 0.0035 | 0.0085 | 0.0170 | 0.0350 | 0.0800 | 0.1300 | 0.1600 |
| BB- | 0.0050 | 0.0120 | 0.0240 | 0.0480 | 0.1000 | 0.1550 | 0.1850 |
| B+ | 0.0070 | 0.0170 | 0.0330 | 0.0650 | 0.1250 | 0.1800 | 0.2100 |
| B | 0.0100 | 0.0240 | 0.0460 | 0.0850 | 0.1500 | 0.2050 | 0.2350 |
| B- | 0.0140 | 0.0340 | 0.0630 | 0.1100 | 0.1750 | 0.2300 | 0.2600 |
| CCC | 0.0200 | 0.0480 | 0.0850 | 0.1400 | 0.2100 | 0.2600 | 0.2900 |
| CC | 0.0300 | 0.0650 | 0.1100 | 0.1700 | 0.2400 | 0.2900 | 0.3200 |
| C | 0.0500 | 0.0900 | 0.1400 | 0.2000 | 0.2800 | 0.3300 | 0.3600 |
| D | 0.0000 | 0.0000 | 0.0000 | 0.0000 | 0.0000 | 0.0000 | 0.0000 |

**Note:** D-rated assets have zero downgrade cost (already in default; default table captures all loss). These are illustrative values; replace with BMA published tables when available in machine-readable form.

### 8.5 Sample Configuration YAML

```yaml
# tests/data/sample_run_config.yaml
#
# Golden integration test configuration.
# Uses the sample dataset from ALGO-07 Section 8.

run_id: "TEST_GOLDEN_001"
company_id: "SAMPLE_CO"
valuation_date: "2024-12-31"
rule_year: 2024

# Input files (relative to project root)
asset_file: "tests/data/sample_assets.csv"
epl_file: "tests/data/sample_epl.csv"
market_data_file: "tests/data/sample_rfr_curve.csv"

# Assumptions
assumption_version: "2024Q4_TEST"

# Projection
projection_currency: "USD"
reinvestment_strategy: "cash_only"
transaction_cost_enabled: true
rate_floor: 0.0

# Output
output_dir: "tests/output/golden_test"
```

### 8.6 Sample Phase-In Configuration

```yaml
# assumptions/tables/phase_in_2024.yaml
schedule:
  2024: 0.20
  2025: 0.40
  2026: 0.60
  2027: 0.80
  2028: 1.00
  2029: 1.00
  2030: 1.00
cutoff_date: "2023-12-31"
```

---

## 9. File Format Specifications

### 9.1 Asset Portfolio File

**Supported formats:** CSV (`.csv`), Excel (`.xlsx`)

**CSV column headers** (first row, exact names):

```
asset_id,asset_type,issuer_name,par_value,book_value,market_value,coupon_rate,coupon_frequency,issue_date,maturity_date,currency,rating,tier,is_callable,is_amortizing,prepayment_speed,sector,bma_approved
```

**Excel format:** Single tab named `Assets`. Same column headers as CSV. Dates formatted as `YYYY-MM-DD` or Excel date serial.

**Type parsing rules:**

| Column | CSV String Format | Python Type |
|---|---|---|
| asset_id | Plain text | str |
| asset_type | Enum value (e.g., `GOVT_BOND`) | AssetType |
| issuer_name | Plain text | str |
| par_value | Numeric, no commas | float |
| book_value | Numeric | float |
| market_value | Numeric | float |
| coupon_rate | Decimal (e.g., `0.05`) | float |
| coupon_frequency | Integer (0, 1, 2, 4) | int |
| issue_date | `YYYY-MM-DD` | date |
| maturity_date | `YYYY-MM-DD` | date |
| currency | 3-letter ISO (e.g., `USD`) | str |
| rating | Rating string (e.g., `AA+`) | CreditRating |
| tier | Integer (1, 2, 3) | int |
| is_callable | `true` / `false` (case-insensitive) | bool |
| is_amortizing | `true` / `false` | bool |
| prepayment_speed | Numeric or empty | Optional[float] |
| sector | Plain text or empty | str |
| bma_approved | `true` / `false` | bool |

**Missing/empty values:** Only `prepayment_speed`, `sector`, and `description` fields may be empty. All other fields are required. An empty required field triggers a `ValidationError`.

### 9.2 EPL File

**Supported formats:** CSV (`.csv`), Excel (`.xlsx`)

**CSV column headers:**

```
period,liability_cf,block_id,currency,description
```

**Excel format:** Single tab named `EPL`. Same column headers.

**Notes:**
- `block_id` defaults to `"DEFAULT"` if column is absent
- `currency` defaults to `"USD"` if column is absent
- `description` is optional

### 9.3 Market Data File

**Supported formats:** CSV (`.csv`)

**CSV column headers:**

```
tenor_years,rate,instrument_type,date
```

**Notes:**
- `instrument_type` defaults to `"SWAP"` if column is absent
- All rows must have the same `date` value (the valuation date)
- Rates are expressed as decimals (e.g., `0.0425`, not `4.25%`)

### 9.4 D&D Table Files

**Supported formats:** CSV (`.csv`)

**Default table CSV structure:**

```
rating,year_1,year_2,year_3,year_5,year_10,year_20,year_30
AAA,0.0001,0.0003,0.0006,0.0013,0.0040,0.0120,0.0200
AA+,0.0002,0.0005,0.0010,0.0022,0.0065,0.0180,0.0300
...
```

**Column naming convention:** `year_N` where N is the projection year. The loader parses these column names to extract the year integers.

**File names:**
- `assumptions/tables/bma_default_floors.csv` -- cumulative default rates
- `assumptions/tables/bma_downgrade_floors.csv` -- cumulative downgrade rates

### 9.5 Assumption Manifest

**Format:** YAML

**File:** `assumptions/tables/manifest.yaml`

```yaml
# assumptions/tables/manifest.yaml
#
# Tracks versions and metadata for all assumption files.

version: "2024Q4"
effective_date: "2024-12-31"
description: "BMA SBA assumption set for 2024 Q4 valuation"

files:
  risk_free_curve:
    path: "bma_rfr_2024Q4.csv"
    source: "BMA published RFR curve"
    as_of_date: "2024-12-31"
    sha256: ""  # Populated at load time for audit trail

  default_floors:
    path: "bma_default_floors.csv"
    source: "BMA published D&D tables"
    version: "2024"
    sha256: ""

  downgrade_floors:
    path: "bma_downgrade_floors.csv"
    source: "BMA published D&D tables"
    version: "2024"
    sha256: ""

  spread_widening:
    path: "spread_widening.csv"
    source: "BMA stress test parameters"
    version: "2024"
    sha256: ""

  phase_in:
    path: "phase_in_2024.yaml"
    source: "BMA transitional arrangement"
    version: "2024"
    sha256: ""
```

---

## 10. Implementation Notes

### 10.1 Pydantic v2 Configuration

All models use Pydantic v2 features:

```python
from pydantic import ConfigDict

# Applied to all schema classes:
model_config = ConfigDict(
    frozen=True,           # Immutable after creation
    str_strip_whitespace=True,  # Strip leading/trailing whitespace from strings
    validate_default=True,      # Validate default values
    use_enum_values=False,      # Keep enum instances (not raw values) for type safety
    extra="forbid",             # Reject unknown fields -- fail loudly
)
```

**Why `extra="forbid"`:** If a CSV has an unexpected column, the loader should reject it rather than silently ignoring it. This catches schema mismatches early.

### 10.2 Loader Pattern

All loaders follow the same pattern:

```python
# model_points/assets.py

import pandas as pd
from pathlib import Path
from typing import List
from .schemas import AssetModelPoint


def load_assets(file_path: str | Path) -> tuple[List[AssetModelPoint], pd.DataFrame]:
    """Load and validate asset model points from CSV or Excel.

    Returns:
        Tuple of (list of validated Pydantic models, pandas DataFrame).
        The DataFrame is for downstream consumption by projection engine.

    Raises:
        FileNotFoundError: If file_path does not exist.
        ValueError: If file format is not CSV or Excel.
        pydantic.ValidationError: If any row fails validation.
            Error message includes row number and field details.
    """
    path = Path(file_path)

    if path.suffix == ".csv":
        df = pd.read_csv(path, parse_dates=["issue_date", "maturity_date"])
    elif path.suffix in (".xlsx", ".xls"):
        df = pd.read_excel(
            path, sheet_name="Assets",
            parse_dates=["issue_date", "maturity_date"],
        )
    else:
        raise ValueError(f"Unsupported file format: {path.suffix}")

    # Validate each row, collecting all errors
    models = []
    errors = []
    for idx, row in df.iterrows():
        try:
            model = AssetModelPoint(**row.to_dict())
            models.append(model)
        except Exception as e:
            errors.append(f"Row {idx + 2} (asset_id={row.get('asset_id', '?')}): {e}")

    if errors:
        error_msg = f"{len(errors)} validation error(s) in {path.name}:\n"
        error_msg += "\n".join(f"  [{i+1}] {e}" for i, e in enumerate(errors))
        raise ValueError(error_msg)

    return models, df
```

**Key design decisions:**
- Return both Pydantic models (for type-safe access) and DataFrame (for vectorized operations).
- Collect all errors before raising -- do not fail on the first bad row.
- Error messages include the CSV row number (offset by 2: 1 for header, 1 for 0-indexed).

### 10.3 DataFrame Column Types

When converting to DataFrame, enforce these dtypes:

| Column | pandas dtype | Notes |
|---|---|---|
| asset_id | `string` | Not object; proper string dtype |
| par_value, book_value, market_value | `float64` | Standard float precision |
| coupon_rate | `float64` | Decimal form |
| coupon_frequency | `int64` | |
| issue_date, maturity_date | `datetime64[ns]` | pandas Timestamp |
| tier | `int64` | Not categorical |
| is_callable, is_amortizing, bma_approved | `bool` | |
| prepayment_speed | `Float64` (nullable) | Use pandas nullable float |

### 10.4 Call Schedule and Amort Schedule Handling

For CSV input, call schedules and amort schedules cannot be represented inline. Two approaches:

**Option A (recommended): Separate files.** Main asset CSV contains only the flat fields. Callable bonds reference a companion file:
- `assets_call_schedule.csv` with columns: `asset_id`, `call_date`, `call_price`
- `assets_amort_schedule.csv` with columns: `asset_id`, `payment_date`, `principal_amount`

The loader joins these on `asset_id` after loading the main file.

**Option B: Excel with multiple tabs.**
- Tab `Assets`: main model points
- Tab `CallSchedule`: call dates/prices keyed by asset_id
- Tab `AmortSchedule`: amort payments keyed by asset_id

### 10.5 Testing Strategy

The sample dataset in Section 8 is the foundation for integration tests:

```python
# tests/test_data_loading.py

def test_load_sample_assets():
    """Verify that sample_assets.csv loads and validates successfully."""
    models, df = load_assets("tests/data/sample_assets.csv")
    assert len(models) == 11
    assert df.shape[0] == 11

    # Tier distribution
    tier_counts = df.groupby("tier").size()
    assert tier_counts[1] == 8   # 6 bonds + 1 muni + 1 cash
    assert tier_counts[2] == 1   # MBS
    assert tier_counts[3] == 2   # HY bond + CRE

    # Tier 3 cap
    total_mv = df["market_value"].sum()
    tier3_mv = df.loc[df["tier"] == 3, "market_value"].sum()
    assert tier3_mv / total_mv < 0.10


def test_load_sample_epl():
    """Verify that sample_epl.csv loads with 30 contiguous periods."""
    cfs, df = load_epl("tests/data/sample_epl.csv")
    epl_set = EPLCashFlowSet(cash_flows=cfs)
    assert epl_set.t_max == 30
    assert len(epl_set.block_ids) == 1
    assert epl_set.block_ids[0] == "ANNUITY_BLK"


def test_load_sample_curve():
    """Verify that sample_rfr_curve.csv loads with required tenor range."""
    curve = load_risk_free_curve("tests/data/sample_rfr_curve.csv")
    assert len(curve.points) == 15
    assert curve.max_tenor == 40.0
    assert curve.valuation_date == date(2024, 12, 31)


def test_portfolio_cross_validation():
    """Verify cross-schema validation passes for the sample dataset."""
    assets, _ = load_assets("tests/data/sample_assets.csv")
    cfs, _ = load_epl("tests/data/sample_epl.csv")
    epl_set = EPLCashFlowSet(cash_flows=cfs)
    curve = load_risk_free_curve("tests/data/sample_rfr_curve.csv")
    config = load_config("tests/data/sample_run_config.yaml")

    errors = PortfolioValidator.validate(assets, epl_set, curve, config)
    assert errors == [], f"Unexpected validation errors: {errors}"
```

### 10.6 Negative Test Cases

The test suite must also verify that invalid inputs are rejected:

| Test | Input Defect | Expected Error |
|---|---|---|
| `test_missing_required_field` | Remove `par_value` from one row | `ValidationError` mentioning par_value |
| `test_negative_par_value` | `par_value = -1000` | `gt=0` constraint |
| `test_invalid_coupon_frequency` | `coupon_frequency = 3` | "must be 0, 1, 2, or 4" |
| `test_maturity_before_issue` | `maturity_date < issue_date` | "must be after issue_date" |
| `test_tier2_not_approved` | Tier 2 with `bma_approved=false` | "must have bma_approved=True" |
| `test_mbs_no_prepayment` | MBS_AGENCY with no prepayment_speed | "requires prepayment_speed" |
| `test_epl_gap` | Periods 1,2,4 (skip 3) | "non-contiguous periods" |
| `test_epl_zero_total` | All liability_cf = 0 | "total liability_cf must be > 0" |
| `test_tier3_over_cap` | Tier 3 = 15% of MV | "exceeds 10% cap" |
| `test_currency_mismatch` | Assets in USD, liabilities in EUR | "Currency mismatch" |
| `test_curve_no_long_end` | Max tenor = 10y | "at least one tenor >= 20 years" |
| `test_dd_rates_decreasing` | Year_5 < Year_3 in default table | "non-decreasing" |
| `test_unknown_extra_column` | CSV has `bonus_field` column | "extra fields not permitted" |

---

## Appendix A: Enum Reference Quick Table

For quick lookup during implementation:

| Enum | Values | Used By |
|---|---|---|
| `AssetType` | GOVT_BOND, MUNI_BOND, CORP_BOND_IG, CORP_BOND_HY, CALLABLE_BOND, FLOATING_RATE, AMORTIZING, MBS_AGENCY, MBS_NON_AGENCY, ABS, CLO, PREFERRED_STOCK, MORTGAGE_LOAN, COMMERCIAL_RE, CASH | AssetModelPoint, disinvestment waterfall, QuantLib mapping |
| `CreditRating` | AAA, AA+, AA, AA-, A+, A, A-, BBB+, BBB, BBB-, BB+, BB, BB-, B+, B, B-, CCC, CC, C, D | AssetModelPoint, D&D tables, stress tests |

## Appendix B: Cross-Reference to Other Specs

| Schema / Field | Used By Spec | Module |
|---|---|---|
| AssetModelPoint | ALGO-002 (projection engine) | `projection/cashflow_engine.py` |
| AssetModelPoint.tier | ALGO-002b (disinvestment waterfall) | `projection/disinvestment.py` |
| AssetModelPoint.rating | ALGO-003 (D&D credit costs) | `projection/credit_costs.py` |
| AssetModelPoint.asset_type | ALGO-001 (QuantLib bond pricing) | `curves/bond_pricing.py` |
| EPLCashFlow | ALGO-002 (projection engine) | `projection/cashflow_engine.py` |
| RiskFreeCurve | ALGO-001 (yield curve construction) | `curves/yield_curve.py` |
| CreditCostTable | ALGO-003 (D&D credit costs) | `assumptions/credit_tables.py` |
| RunConfig | Engine orchestration | `engine/run_config.py` |
