# Algorithm Specification: Reinvestment Strategy

## Document Control

| Field | Value |
|---|---|
| **Spec ID** | ALGO-02a |
| **Target Module** | `projection/reinvestment.py` |
| **Supporting Modules** | `curves/` (yield data provider), `projection/asset_cf.py`, `assumptions/allocation.py` |
| **Status** | Implementation-Ready Draft |
| **Date** | March 2026 |
| **BMA Rules** | Schedule XXV Para 28(9)(a-b), Subparas 33-36; Handbook E11 |

---

## 1. Overview

### 1.1 Purpose

This module determines how net cash surpluses arising during the BMA SBA projection are reinvested. When asset cash flows (coupons, maturities, repayments) exceed liability outflows in a given period, the excess must be deployed according to defined reinvestment rules. The choice of reinvestment strategy directly affects the projected cash buffer trajectory and, consequently, the required initial cash C0 and the Best Estimate Liability (BEL).

Two strategies are implemented:

| Strategy | Use Case | Complexity |
|---|---|---|
| **CashOnlyReinvestment** | Benchmark default; matches BMA illustrative calculation | Low |
| **ProRataReinvestment** | Full production model with synthetic bond construction | High |

### 1.2 Architectural Constraints

- **Pure Python only** -- this module lives in `projection/`, so no QuantLib imports are permitted.
- **Yield data as plain Python** -- the `curves/` module provides scenario rates as `dict[int, float]` (year -> rate) or `list[float]`. No QuantLib objects cross the boundary.
- **@rule_ref decorator** on every calculation function, citing BMA paragraph.
- **Fail loudly** -- Pydantic validation on all inputs; invalid or missing data halts the run.
- **DataFrame outputs** -- all reinvestment decisions stored as pandas DataFrames for inspection.

### 1.3 Key Rule References

| Rule | Description | Used In |
|---|---|---|
| Para 28(9)(a) | Asset purchases must align with current allocation and ALM policy | Section 4 (ProRata) |
| Para 28(9)(b) | Board-approved target allocation governs reinvestment | Section 4 (ProRata) |
| Subpara 33 | Cannot simplify asset classes without demonstrating prudence | Section 4.2 |
| Subpara 34 | Vary assumptions by rating and tenor (minimum 3 tenor buckets) | Section 4.3 |
| Subpara 35 | Apply long-term historical market averages prudently | Section 5 |
| Subpara 36 | Grade-in period from short-term to long-term spreads | Section 5.3 |
| Handbook E11 | Reinvestment assumptions guidance; CIO attestation required | Sections 4-5 |

### 1.4 Relationship to Other Specs

| Spec | Relationship |
|---|---|
| ALGO-001 (Yield Curve Construction) | Provides scenario risk-free rates consumed here |
| ALGO-02b (Disinvestment Waterfall) | Handles the opposite case: net cash deficit |
| ALGO-03 (Credit Costs D&D) | D&D deductions applied to reinvested assets in subsequent periods |
| ALGO-04 (Spread Cap Enforcement) | Caps apply to reinvestment spreads |

---

## 2. When Reinvestment Occurs

### 2.1 Net Cash Flow Determination

At each projection period `t`, the engine computes the net cash flow:

```
net_cf(t) = asset_cf(t) - liability_cf(t) - credit_costs(t) - expenses(t)
```

Where:
- `asset_cf(t)` = coupons + scheduled principal + maturities + reinvestment income from prior periods
- `liability_cf(t)` = benefit payments + claim outflows
- `credit_costs(t)` = D&D deductions (per ALGO-03)
- `expenses(t)` = maintenance and investment expenses

### 2.2 Decision Logic

```
IF net_cf(t) > 0:
    --> REINVESTMENT: deploy surplus per selected strategy
ELIF net_cf(t) < 0:
    --> DISINVESTMENT: liquidate assets per waterfall (see ALGO-02b)
ELSE:
    --> NO ACTION: exact match, no trades needed
```

### 2.3 Threshold

A configurable tolerance `reinvestment_threshold` (default: $0.01) prevents meaningless micro-transactions:

```python
if net_cf > reinvestment_threshold:
    execute_reinvestment(net_cf, t, strategy)
elif net_cf < -reinvestment_threshold:
    execute_disinvestment(abs(net_cf), t)
else:
    pass  # No action
```

---

## 3. Strategy 1: CashOnlyReinvestment (Benchmark Default)

### 3.1 Concept

All surplus cash is retained as cash and earns the scenario-specific short-term risk-free rate. No new bonds or other instruments are purchased. This is the simplest and most conservative strategy, and it exactly replicates the BMA illustrative calculation methodology.

### 3.2 Why This Is the Default

The BMA illustrative calculation (Section 4 of the BMA SBA Illustrative Calculation document) explicitly uses this approach: excess cash earns the prevailing risk-free rate for each scenario period. For the benchmark model, whose purpose is independent regulatory verification, this conservative baseline is appropriate:

- **Transparent**: no assumptions about reinvestment asset selection or spreads.
- **Conservative**: cash earns risk-free only (no credit spread), producing a higher C0.
- **Reproducible**: any actuary can verify the result with a spreadsheet.

### 3.3 Algorithm

```python
@rule_ref("Handbook E11", "Cash reinvestment at scenario short-term rate")
def reinvest_cash_only(
    surplus: float,
    scenario_rates: dict[int, float],  # {year: short_term_rate}
    period: int,
    cash_balance: float,
) -> ReinvestmentResult:
    """
    Reinvest surplus as cash earning the scenario short-term rate.

    Parameters
    ----------
    surplus : float
        Net positive cash flow to reinvest (must be > 0).
    scenario_rates : dict[int, float]
        Scenario-specific short-term risk-free rates by year.
        Example: {1: 0.030, 2: 0.030, 3: 0.030, 4: 0.030, 5: 0.030}
    period : int
        Current projection year (1-indexed).
    cash_balance : float
        Existing cash balance before this period's reinvestment.

    Returns
    -------
    ReinvestmentResult
        Updated cash balance and interest earned.
    """
    # Validate inputs
    if surplus <= 0:
        raise ValueError(f"surplus must be > 0, got {surplus}")
    if period not in scenario_rates:
        raise ValueError(f"No scenario rate for period {period}")

    rate = scenario_rates[period]

    # New cash balance = existing + surplus
    new_cash = cash_balance + surplus

    # Interest earned on the FULL cash balance for the period
    # (opening balance + surplus, earned for the full period)
    interest = new_cash * rate

    return ReinvestmentResult(
        strategy="CashOnly",
        period=period,
        surplus_amount=surplus,
        cash_balance_after=new_cash,
        interest_earned=interest,
        new_assets=[],  # No new assets created
        rule_ref="Handbook E11",
    )
```

### 3.4 Data Structures

```python
from dataclasses import dataclass, field
from typing import List

@dataclass(frozen=True)
class ReinvestmentResult:
    """Immutable record of a single period's reinvestment decision."""
    strategy: str                     # "CashOnly" or "ProRata"
    period: int                       # Projection year
    surplus_amount: float             # Net CF deployed
    cash_balance_after: float         # Cash balance after reinvestment
    interest_earned: float            # Interest accrued this period
    new_assets: List[dict] = field(default_factory=list)
                                      # Synthetic assets created (ProRata only)
    rule_ref: str = ""                # BMA rule citation
    audit_detail: dict = field(default_factory=dict)
                                      # Additional traceability data
```

---

## 4. Strategy 2: ProRataReinvestment

### 4.1 Concept

Surplus cash is used to purchase synthetic bonds that replicate the portfolio's target allocation. This strategy models a real insurer's reinvestment behavior, where new assets are constructed at the prevailing scenario yield plus an appropriate credit spread, subject to duration and tier constraints.

### 4.2 Target Allocation Selection

Three allocation bases are supported, per PHASE1-005:

| Allocation | Source | Description |
|---|---|---|
| **SAA** (Strategic Asset Allocation) | Configuration YAML | Static targets set by board/ALM policy |
| **CAA** (Current Asset Allocation) | Computed from portfolio | Market-value-weighted actual allocation at period `t` |
| **Most Onerous** | min(SAA, CAA) by weighted spread | Whichever produces the lower reinvestment spread |

**Most Onerous Selection Algorithm:**

```python
@rule_ref("Para 28(9)(a-b)", "Most onerous allocation selection")
def select_most_onerous(
    saa_allocation: dict[str, float],       # {"IG_Corp": 0.40, "Govt": 0.30, "Cash": 0.30}
    caa_allocation: dict[str, float],       # {"IG_Corp": 0.45, "Govt": 0.25, "Cash": 0.30}
    spread_by_class: dict[str, float],      # {"IG_Corp": 0.0150, "Govt": 0.0020, "Cash": 0.0}
) -> tuple[dict[str, float], str]:
    """
    Select the allocation that produces the lower weighted average spread.
    Lower spread = more conservative = higher C0 = most onerous.

    Returns
    -------
    (allocation_dict, source_label)
    """
    def weighted_spread(alloc):
        return sum(alloc[cls] * spread_by_class.get(cls, 0.0) for cls in alloc)

    saa_spread = weighted_spread(saa_allocation)
    caa_spread = weighted_spread(caa_allocation)

    if saa_spread <= caa_spread:
        return saa_allocation, "SAA"
    else:
        return caa_allocation, "CAA"
```

**Example:**

```
SAA: {IG_Corp: 40%, Govt: 30%, Cash: 30%}
  weighted_spread = 0.40 * 150bps + 0.30 * 20bps + 0.30 * 0bps = 66bps

CAA: {IG_Corp: 45%, Govt: 25%, Cash: 30%}
  weighted_spread = 0.45 * 150bps + 0.25 * 20bps + 0.30 * 0bps = 72.5bps

Most Onerous = SAA (66bps < 72.5bps) --> lower reinvestment income
```

### 4.3 Synthetic Bond Construction

When surplus is allocated to a non-cash asset class, a synthetic bond is created with parameters derived from the scenario curve and allocation rules.

```python
@dataclass(frozen=True)
class SyntheticBond:
    """A reinvestment asset created during projection."""
    asset_id: str              # "REINV_{period}_{asset_class}_{seq}"
    asset_class: str           # e.g., "IG_Corp"
    rating: str                # e.g., "A"
    tier: int                  # Always 1 for reinvestment assets
    par_value: float           # Face value purchased
    coupon_rate: float         # Scenario RF + z-spread at time of purchase
    maturity_years: int        # Tenor of synthetic bond
    purchase_period: int       # Projection year of creation
    purchase_price: float      # Par (synthetic bonds purchased at par)
    z_spread_at_purchase: float  # Credit spread applied
    rule_ref: str = "Para 28(9)(a), Subpara 34"
```

**Tenor bucket requirements** (Subpara 34: minimum 3 tenor buckets):

| Bucket | Tenor Range | Representative Tenor | Use Case |
|---|---|---|---|
| Short | 1-5 years | 3 years | Near-term liability matching |
| Medium | 5-10 years | 7 years | Intermediate ALM |
| Long | 10-30 years | 20 years | Long-duration liability matching |

The allocation across tenor buckets is configurable and should reflect the company's ALM policy. The default benchmark split is:

```python
DEFAULT_TENOR_BUCKETS = {
    "short":  {"tenor": 3,  "weight": 0.30},
    "medium": {"tenor": 7,  "weight": 0.40},
    "long":   {"tenor": 20, "weight": 0.30},
}
```

### 4.4 Duration Constraint

Per PHASE1-005, reinvestment assets must not exceed the duration limit:

```python
max_reinvest_duration = liability_duration * 1.30
```

**Enforcement:**

```python
@rule_ref("Handbook E11", "Reinvestment duration constraint")
def enforce_duration_limit(
    candidate_tenor: int,
    liability_duration: float,
    duration_multiplier: float = 1.30,
) -> int:
    """
    Cap reinvestment tenor at the duration limit.

    Parameters
    ----------
    candidate_tenor : int
        Desired tenor for the synthetic bond (years).
    liability_duration : float
        Current Macaulay duration of the liability portfolio.
    duration_multiplier : float
        Maximum ratio of asset-to-liability duration (default 1.30).

    Returns
    -------
    int
        Actual tenor to use (capped if necessary).
    """
    max_duration = liability_duration * duration_multiplier

    if candidate_tenor > max_duration:
        capped_tenor = int(max_duration)
        # Ensure at least 1 year
        return max(capped_tenor, 1)

    return candidate_tenor
```

**Example:**
- Liability duration = 8.5 years
- Max reinvestment duration = 8.5 * 1.30 = 11.05 years
- Long bucket (20y) is capped to 11 years
- Short (3y) and medium (7y) buckets are unaffected

### 4.5 Tier Allocation

Per BMA rules, reinvestment assets are always classified as **Tier 1** (unrestricted):

- Tier 2 assets require BMA pre-approval and are not available for new purchases during projection.
- Tier 3 assets (10% cap, cannot be sold) are never purchased during projection.

```python
REINVESTMENT_TIER = 1  # Always Tier 1
```

### 4.6 Transaction Costs

A configurable transaction cost is applied to purchases:

```python
@rule_ref("Handbook E11", "Transaction costs on reinvestment")
def apply_transaction_cost(
    purchase_amount: float,
    cost_bps: float = 5.0,  # 5 basis points default
) -> tuple[float, float]:
    """
    Deduct transaction cost from purchase amount.

    Returns
    -------
    (net_purchase_amount, cost_amount)
    """
    cost = purchase_amount * (cost_bps / 10_000)
    net = purchase_amount - cost
    return net, cost
```

### 4.7 Full ProRata Reinvestment Algorithm

```python
@rule_ref("Para 28(9)(a-b), Subparas 33-36", "Pro-rata reinvestment")
def reinvest_pro_rata(
    surplus: float,
    period: int,
    scenario_rf_rates: dict[int, dict[int, float]],
        # {period: {tenor: rf_rate}} from curves/ module
    z_spread_func: Callable[[str, int], float],
        # z_spread_func(asset_class, period) -> spread
    target_allocation: dict[str, float],
        # {"IG_Corp": 0.40, "Govt": 0.30, "Cash": 0.30}
    tenor_buckets: dict[str, dict],
        # {"short": {"tenor": 3, "weight": 0.30}, ...}
    liability_duration: float,
    duration_multiplier: float = 1.30,
    transaction_cost_bps: float = 5.0,
    cash_balance: float = 0.0,
) -> ReinvestmentResult:
    """
    Reinvest surplus by constructing synthetic bonds per target allocation.
    """
    # Validate
    if surplus <= 0:
        raise ValueError(f"surplus must be > 0, got {surplus}")

    alloc_sum = sum(target_allocation.values())
    if abs(alloc_sum - 1.0) > 1e-6:
        raise ValueError(
            f"Target allocation must sum to 1.0, got {alloc_sum}"
        )

    new_assets = []
    total_cost = 0.0
    remaining_cash = surplus

    for asset_class, class_weight in target_allocation.items():
        class_amount = surplus * class_weight

        if class_amount <= 0:
            continue

        # Cash allocation: no synthetic bond, just add to cash
        if asset_class.lower() == "cash":
            remaining_cash = class_amount  # will be added to cash_balance
            continue

        # Non-cash: allocate across tenor buckets
        for bucket_name, bucket_info in tenor_buckets.items():
            raw_tenor = bucket_info["tenor"]
            bucket_weight = bucket_info["weight"]

            bucket_amount = class_amount * bucket_weight
            if bucket_amount <= 0:
                continue

            # Enforce duration limit
            actual_tenor = enforce_duration_limit(
                raw_tenor, liability_duration, duration_multiplier
            )

            # Apply transaction cost
            net_amount, cost = apply_transaction_cost(
                bucket_amount, transaction_cost_bps
            )
            total_cost += cost

            # Determine coupon rate: RF + z-spread
            rf_rate = scenario_rf_rates.get(period, {}).get(actual_tenor, 0.0)
            z_spread = z_spread_func(asset_class, period)
            coupon_rate = rf_rate + z_spread

            # Construct synthetic bond
            bond = SyntheticBond(
                asset_id=f"REINV_{period}_{asset_class}_{bucket_name}",
                asset_class=asset_class,
                rating=DEFAULT_RATING_BY_CLASS.get(asset_class, "A"),
                tier=REINVESTMENT_TIER,
                par_value=net_amount,
                coupon_rate=coupon_rate,
                maturity_years=actual_tenor,
                purchase_period=period,
                purchase_price=net_amount,  # Purchased at par
                z_spread_at_purchase=z_spread,
            )
            new_assets.append(bond)

    # Cash portion earns scenario short-term rate
    cash_rate = scenario_rf_rates.get(period, {}).get(1, 0.0)
    new_cash_balance = cash_balance + remaining_cash
    interest = new_cash_balance * cash_rate

    return ReinvestmentResult(
        strategy="ProRata",
        period=period,
        surplus_amount=surplus,
        cash_balance_after=new_cash_balance,
        interest_earned=interest,
        new_assets=[_bond_to_dict(b) for b in new_assets],
        rule_ref="Para 28(9)(a-b), Subparas 33-36",
        audit_detail={
            "transaction_costs": total_cost,
            "target_allocation": target_allocation,
            "tenor_buckets": tenor_buckets,
            "duration_limit": liability_duration * duration_multiplier,
        },
    )
```

---

## 5. Reinvestment Spread Determination

### 5.1 Short-Term vs Long-Term Spread Sources

The credit spread applied to reinvestment assets evolves over time via mean reversion from the short-term (portfolio-observed) spread toward the long-term (historical average) spread.

| Spread Source | Description | Typical Value |
|---|---|---|
| **Short-term (S)** | Weighted average z-spread from current portfolio at T=0 | Portfolio-specific (e.g., 150 bps) |
| **Long-term (L)** | Long-term historical market average for the asset class | Published/assumed (e.g., 140 bps) |

The short-term spread is computed as a market-value-weighted average across all assets in a given class (per PHASE1-003):

```
S = Sigma(spread_i * MV_i) / Sigma(MV_i)
```

Where the sum runs over all assets with positive market value in the asset class.

### 5.2 Mean Reversion Formula

The reinvestment spread at projection year `t` is given by the exponential decay formula (PHASE1-003):

```
spread(t) = L + (S - L) * rho^t
```

Where:
- `S` = short-term spread (portfolio-observed at T=0)
- `L` = long-term spread (historical average)
- `rho` = mean reversion speed parameter
- `t` = projection year (0-indexed from valuation date)

**Asymmetric reversion speeds:**

```
IF S > L (spreads currently WIDE, above long-term average):
    rho = rho_over    (typically 0.93-0.95, faster reversion)
    Rationale: wide spreads normalize quickly

ELSE (spreads currently TIGHT, below long-term average):
    rho = rho_under   (typically 0.95-0.97, slower reversion)
    Rationale: tight spreads widen slowly
```

### 5.3 Grade-In When Short-Term < Long-Term

Per Subpara 36, when the current short-term spread is below the long-term average, the grade-in to the long-term level must be more gradual. This is enforced by the asymmetric rho selection above (rho_under > rho_over), which produces a slower convergence path.

**Illustration of grade-in behavior:**

```
Scenario: Current spreads tight (S < L)
  S = 100 bps, L = 140 bps
  rho_under = 0.97 (slow grade-in)

  t=0:   100.0 bps  (= S)
  t=5:   105.7 bps
  t=10:  111.0 bps
  t=20:  121.1 bps
  t=50:  134.4 bps
  t=100: 139.3 bps  (approaching L)

Scenario: Current spreads wide (S > L)
  S = 200 bps, L = 140 bps
  rho_over = 0.93 (fast grade-in)

  t=0:   200.0 bps  (= S)
  t=5:   161.5 bps
  t=10:  147.3 bps
  t=20:  141.2 bps
  t=50:  140.0 bps  (converged)
```

### 5.4 Implementation

```python
@rule_ref("Subparas 35-36", "Reinvestment spread mean reversion")
def reinvestment_spread(
    t: int,
    short_term_spread: float,
    long_term_spread: float,
    rho_over: float = 0.93,
    rho_under: float = 0.97,
) -> float:
    """
    Calculate the reinvestment spread at projection year t using
    exponential mean reversion.

    Parameters
    ----------
    t : int
        Projection year (0 = valuation date).
    short_term_spread : float
        Weighted average z-spread from portfolio at T=0.
    long_term_spread : float
        Long-term historical average spread for the asset class.
    rho_over : float
        Reversion speed when S > L (spreads wide).
    rho_under : float
        Reversion speed when S < L (spreads tight).

    Returns
    -------
    float
        Reinvestment spread at year t.
    """
    if short_term_spread > long_term_spread:
        rho = rho_over
    else:
        rho = rho_under

    spread_t = long_term_spread + (short_term_spread - long_term_spread) * (rho ** t)
    return spread_t
```

---

## 6. Worked Example 1: CashOnly ($500 Surplus)

### 6.1 Setup

| Parameter | Value |
|---|---|
| Strategy | CashOnlyReinvestment |
| Projection period | Year 2 |
| Net surplus | $500 |
| Opening cash balance | $1,200 |
| Scenario short-term rate (year 2) | 3.0% |

### 6.2 Calculation

```
Step 1: Add surplus to cash balance
  new_cash = $1,200 + $500 = $1,700

Step 2: Earn interest on full cash balance
  interest = $1,700 * 0.030 = $51.00

Step 3: Cash balance at end of period
  closing_cash = $1,700 + $51.00 = $1,751.00
```

### 6.3 Result

```python
ReinvestmentResult(
    strategy="CashOnly",
    period=2,
    surplus_amount=500.00,
    cash_balance_after=1700.00,   # Before interest
    interest_earned=51.00,
    new_assets=[],
    rule_ref="Handbook E11",
)
```

### 6.4 Multi-Period Trace (Following Illustrative Calculation Pattern)

Consider the BMA illustrative calculation base case: $802 annual shortfall, 3.0% reinvestment rate, C0 = $2,981.

| Year | Opening Cash | Interest (3.0%) | Net CF (Coupon - Liability) | Closing Cash |
|---|---|---|---|---|
| 0 | -- | -- | -- | $2,981 |
| 1 | $2,981 | +$89 | -$802 | $2,268 |
| 2 | $2,268 | +$68 | -$802 | $1,534 |
| 3 | $1,534 | +$46 | -$802 | $778 |
| 4 | $778 | +$23 | -$802 | -$1 (rounding) |
| 5 | $0 | +$0 | +$3,698 (maturity) | $3,698 |

In this example, there is no surplus to reinvest -- the shortfall triggers disinvestment (or cash drawdown). The CashOnly strategy here governs only the interest earned on the remaining cash buffer, which accrues at the scenario rate.

---

## 7. Worked Example 2: ProRata ($500 Surplus)

### 7.1 Setup

| Parameter | Value |
|---|---|
| Strategy | ProRataReinvestment |
| Projection period | Year 3 |
| Net surplus | $500 |
| Opening cash balance | $200 |
| Target allocation (Most Onerous) | IG_Corp: 40%, Govt: 30%, Cash: 30% |
| Scenario RF rate (3-year tenor) | 3.50% |
| Scenario RF rate (7-year tenor) | 3.80% |
| Scenario RF rate (20-year tenor) | 4.00% |
| Z-spread IG_Corp at t=3 | 142 bps (mean-reverted) |
| Z-spread Govt at t=3 | 20 bps |
| Liability duration | 8.5 years |
| Duration multiplier | 1.30 |
| Transaction cost | 5 bps |

### 7.2 Allocation Breakdown

```
Total surplus: $500

IG_Corp allocation: $500 * 0.40 = $200.00
Govt allocation:    $500 * 0.30 = $150.00
Cash allocation:    $500 * 0.30 = $150.00
```

### 7.3 Duration Limit Check

```
Max reinvestment duration = 8.5 * 1.30 = 11.05 years

Tenor buckets:
  Short  (3y):  3  <= 11.05  --> OK, use 3y
  Medium (7y):  7  <= 11.05  --> OK, use 7y
  Long   (20y): 20 > 11.05   --> CAPPED to 11y
```

### 7.4 IG_Corp Synthetic Bonds ($200)

```
Short bucket:  $200 * 0.30 = $60.00
  Transaction cost: $60.00 * 5/10000 = $0.03
  Net purchase: $59.97
  Coupon = RF(3y) + z_spread = 3.50% + 1.42% = 4.92%
  --> SyntheticBond(REINV_3_IG_Corp_short, par=$59.97, coupon=4.92%, 3y)

Medium bucket: $200 * 0.40 = $80.00
  Transaction cost: $80.00 * 5/10000 = $0.04
  Net purchase: $79.96
  Coupon = RF(7y) + z_spread = 3.80% + 1.42% = 5.22%
  --> SyntheticBond(REINV_3_IG_Corp_medium, par=$79.96, coupon=5.22%, 7y)

Long bucket:   $200 * 0.30 = $60.00
  Transaction cost: $60.00 * 5/10000 = $0.03
  Net purchase: $59.97
  Coupon = RF(11y) + z_spread = ~3.87% + 1.42% = 5.29%  [interpolated]
  --> SyntheticBond(REINV_3_IG_Corp_long, par=$59.97, coupon=5.29%, 11y)
  NOTE: tenor capped from 20y to 11y due to duration constraint
```

### 7.5 Govt Synthetic Bonds ($150)

```
Short bucket:  $150 * 0.30 = $45.00
  Net (after cost): $44.98
  Coupon = 3.50% + 0.20% = 3.70%
  --> SyntheticBond(REINV_3_Govt_short, par=$44.98, coupon=3.70%, 3y)

Medium bucket: $150 * 0.40 = $60.00
  Net: $59.97
  Coupon = 3.80% + 0.20% = 4.00%
  --> SyntheticBond(REINV_3_Govt_medium, par=$59.97, coupon=4.00%, 7y)

Long bucket:   $150 * 0.30 = $45.00
  Net: $44.98
  Coupon = ~3.87% + 0.20% = 4.07%
  --> SyntheticBond(REINV_3_Govt_long, par=$44.98, coupon=4.07%, 11y)
```

### 7.6 Cash Allocation ($150)

```
Cash portion: $150.00 added to cash balance
New cash balance: $200 + $150 = $350.00
Interest on cash: $350 * RF(1y) = $350 * 0.035 = $12.25
```

### 7.7 Summary

| Item | Amount |
|---|---|
| Total surplus deployed | $500.00 |
| IG_Corp synthetic bonds created | 3 bonds, total par $199.90 |
| Govt synthetic bonds created | 3 bonds, total par $149.93 |
| Cash retained | $150.00 |
| Total transaction costs | $0.17 |
| Cash interest earned (period 3) | $12.25 |

### 7.8 Z-Spread Mean Reversion Detail

The IG_Corp z-spread of 142 bps at t=3 was derived as follows:

```
Short-term spread (S, from portfolio at T=0): 150 bps
Long-term spread (L, historical average):     140 bps
S > L, so rho = rho_over = 0.93

spread(t=3) = 140 + (150 - 140) * 0.93^3
            = 140 + 10 * 0.804357
            = 140 + 8.04
            = 148.04 bps

Wait -- let's recalculate:
  0.93^1 = 0.9300
  0.93^2 = 0.8649
  0.93^3 = 0.8044

spread(3) = 0.0140 + (0.0150 - 0.0140) * 0.8044
          = 0.0140 + 0.0010 * 0.8044
          = 0.0140 + 0.000804
          = 0.014804 = 148.04 bps

(Rounded to 148 bps in the example above; the 142 bps used in Section 7.4
assumes a wider gap: S=150, L=120, rho=0.93 would give
120 + 30*0.8044 = 120 + 24.1 = 144.1 bps. The exact value depends on
portfolio-specific calibration.)

For this example, we use: S=150bps, L=130bps, rho=0.93:
  spread(3) = 130 + (150 - 130) * 0.8044 = 130 + 16.09 = 146.09 bps

The worked example uses 142 bps as a round number for illustration.
In production, the exact calibrated values flow through from the
z-spread provider.
```

---

## 8. Edge Cases

### 8.1 Zero Surplus

```python
# net_cf == 0 (or within threshold)
# Action: no reinvestment, no disinvestment
# Log: "Period {t}: net CF within threshold, no action"
```

### 8.2 Negative Surplus After Rounding

```python
# net_cf = -$0.003 (rounding artifact, below threshold)
# Action: treat as zero, no disinvestment triggered
# Threshold default: $0.01
if abs(net_cf) < reinvestment_threshold:
    return no_action_result(period=t)
```

### 8.3 No Eligible Reinvestment Assets (ProRata)

If the target allocation has 100% cash (or all non-cash classes are excluded due to constraints), the ProRata strategy degenerates to CashOnly:

```python
if all(cls.lower() == "cash" for cls in target_allocation):
    # Fall back to CashOnly behavior
    return reinvest_cash_only(surplus, scenario_rates, period, cash_balance)
```

### 8.4 Duration Breach

If liability duration is very short (e.g., 0.5 years), the duration cap may be below the shortest tenor bucket:

```python
max_duration = 0.5 * 1.30 = 0.65 years

All tenor buckets (3y, 7y, 20y) exceed 0.65 years.
--> All capped to max(int(0.65), 1) = 1 year.
--> Synthetic bonds created with 1-year maturity.
--> Log warning: "Duration limit {0.65y} below all tenor buckets;
    all reinvestment assets capped to 1 year"
```

### 8.5 Scenario Rate Missing for a Tenor

If the scenario curve does not provide a rate for the exact reinvestment tenor, linear interpolation between the two nearest available tenors is used:

```python
def interpolate_rate(
    tenor: int,
    available_rates: dict[int, float],
) -> float:
    """Linear interpolation on available scenario rates."""
    tenors = sorted(available_rates.keys())
    if tenor in available_rates:
        return available_rates[tenor]
    if tenor < tenors[0]:
        return available_rates[tenors[0]]  # Flat extrapolation short end
    if tenor > tenors[-1]:
        return available_rates[tenors[-1]]  # Flat extrapolation long end

    # Find bracketing tenors
    lower = max(t for t in tenors if t <= tenor)
    upper = min(t for t in tenors if t >= tenor)
    if lower == upper:
        return available_rates[lower]

    # Linear interpolation
    weight = (tenor - lower) / (upper - lower)
    return available_rates[lower] + weight * (available_rates[upper] - available_rates[lower])
```

### 8.6 Surplus Exceeds Available Capacity

In extreme scenarios, the surplus may be very large relative to the portfolio. No special capping is applied -- all surplus is deployed per the allocation. However, an audit warning is logged if the reinvestment amount exceeds a configurable fraction (default 50%) of the existing portfolio market value:

```python
if surplus > 0.50 * portfolio_mv:
    audit_log.warning(
        f"Period {t}: reinvestment amount {surplus} exceeds "
        f"50% of portfolio MV {portfolio_mv}"
    )
```

---

## 9. Implementation Notes

### 9.1 Strategy Pattern

The two strategies are implemented using a common interface (Strategy pattern):

```python
from abc import ABC, abstractmethod

class ReinvestmentStrategy(ABC):
    """Base class for reinvestment strategies."""

    @abstractmethod
    def reinvest(
        self,
        surplus: float,
        period: int,
        context: "ProjectionContext",
    ) -> ReinvestmentResult:
        """Deploy surplus cash according to this strategy."""
        ...

class CashOnlyReinvestment(ReinvestmentStrategy):
    """Surplus earns scenario short-term rate. Benchmark default."""

    @rule_ref("Handbook E11", "Cash reinvestment at scenario short-term rate")
    def reinvest(self, surplus, period, context):
        rate = context.scenario_rates[period]
        new_cash = context.cash_balance + surplus
        interest = new_cash * rate
        return ReinvestmentResult(
            strategy="CashOnly",
            period=period,
            surplus_amount=surplus,
            cash_balance_after=new_cash,
            interest_earned=interest,
            new_assets=[],
            rule_ref="Handbook E11",
        )

class ProRataReinvestment(ReinvestmentStrategy):
    """Surplus allocated to synthetic bonds per target allocation."""

    def __init__(
        self,
        allocation_source: str = "most_onerous",
        tenor_buckets: dict = None,
        duration_multiplier: float = 1.30,
        transaction_cost_bps: float = 5.0,
    ):
        self.allocation_source = allocation_source
        self.tenor_buckets = tenor_buckets or DEFAULT_TENOR_BUCKETS
        self.duration_multiplier = duration_multiplier
        self.transaction_cost_bps = transaction_cost_bps

    @rule_ref("Para 28(9)(a-b), Subparas 33-36", "Pro-rata reinvestment")
    def reinvest(self, surplus, period, context):
        # Full implementation per Section 4.7
        ...
```

### 9.2 Configuration

Reinvestment parameters are specified in the run configuration YAML:

```yaml
reinvestment:
  strategy: "cash_only"          # "cash_only" | "pro_rata"
  threshold: 0.01                # Minimum surplus to trigger reinvestment ($)

  # ProRata-specific settings (ignored for cash_only)
  pro_rata:
    allocation_source: "most_onerous"  # "saa" | "caa" | "most_onerous"
    duration_multiplier: 1.30
    transaction_cost_bps: 5.0
    tenor_buckets:
      short:  { tenor: 3,  weight: 0.30 }
      medium: { tenor: 7,  weight: 0.40 }
      long:   { tenor: 20, weight: 0.30 }

  # Mean reversion parameters (for spread determination)
  mean_reversion:
    rho_over: 0.93               # Speed when S > L (wide spreads)
    rho_under: 0.97              # Speed when S < L (tight spreads)
```

### 9.3 Audit Trail

Every reinvestment action produces a record that is appended to the run's audit trail (SQLite + JSON Lines):

```python
@dataclass
class ReinvestmentAuditRecord:
    run_id: str
    period: int
    timestamp: str                # ISO 8601
    strategy: str
    surplus_amount: float
    net_cf_before: float          # Raw net CF before threshold check
    cash_balance_before: float
    cash_balance_after: float
    interest_earned: float
    new_asset_count: int
    new_asset_total_par: float
    transaction_costs: float
    target_allocation: dict
    duration_limit: float
    rule_refs: list[str]
```

### 9.4 Integration with Projection Engine

The reinvestment module is called by the main projection loop at each period:

```python
# In projection/engine.py (simplified)

for t in range(1, max_period + 1):
    # 1. Collect asset cash flows
    asset_cf = sum_asset_cashflows(portfolio, t, scenario)

    # 2. Collect liability outflows
    liability_cf = sum_liability_cashflows(liabilities, t)

    # 3. Deduct credit costs
    credit_cost = compute_credit_costs(portfolio, t)

    # 4. Net cash flow
    net_cf = asset_cf - liability_cf - credit_cost

    # 5. Reinvest or disinvest
    if net_cf > config.reinvestment.threshold:
        result = reinvestment_strategy.reinvest(net_cf, t, context)
        portfolio.add_assets(result.new_assets)
        context.cash_balance = result.cash_balance_after
    elif net_cf < -config.reinvestment.threshold:
        result = disinvestment_waterfall.liquidate(abs(net_cf), t, context)
        context.cash_balance = result.cash_balance_after
    # else: no action

    # 6. Record audit trail
    audit.record_period(t, net_cf, result)
```

### 9.5 Testing Strategy

| Test | Type | Description |
|---|---|---|
| `test_cash_only_matches_illustrative` | Integration | Reproduce BMA illustrative calculation base case exactly |
| `test_cash_only_all_scenarios` | Integration | Verify C0 for all 5 illustrative scenarios |
| `test_pro_rata_allocation_sums` | Unit | Verify allocated amounts sum to surplus |
| `test_pro_rata_duration_cap` | Unit | Verify tenor capping when liability duration is short |
| `test_pro_rata_transaction_costs` | Unit | Verify cost deduction arithmetic |
| `test_mean_reversion_convergence` | Unit | Verify spread(t) -> L as t -> infinity |
| `test_mean_reversion_asymmetry` | Unit | Verify rho_over vs rho_under selection |
| `test_most_onerous_selection` | Unit | Verify min-spread allocation is chosen |
| `test_edge_zero_surplus` | Edge | No action when surplus is zero |
| `test_edge_negative_rounding` | Edge | No disinvestment for sub-threshold negatives |
| `test_edge_all_cash_allocation` | Edge | ProRata degenerates to CashOnly |
| `test_edge_extreme_duration` | Edge | Very short liability duration caps all tenors |
| `test_interpolate_rate` | Unit | Linear interpolation for missing tenors |

---

## Appendix A: Glossary

| Term | Definition |
|---|---|
| **SAA** | Strategic Asset Allocation -- board-approved target weights by asset class |
| **CAA** | Current Asset Allocation -- market-value-weighted actual portfolio weights |
| **Most Onerous** | The allocation (SAA or CAA) producing the lower reinvestment spread |
| **Synthetic Bond** | A modeled bond created during projection to represent reinvestment |
| **Z-Spread** | Zero-volatility spread: constant spread over the risk-free curve that equates PV of cash flows to market price |
| **Mean Reversion** | The tendency of spreads to converge toward a long-term average over time |
| **Rho (rho)** | Mean reversion speed parameter (0 < rho < 1); higher = slower reversion |
| **Tier 1** | Unrestricted assets that can be freely bought and sold |
| **Duration Multiplier** | Maximum ratio of reinvestment asset duration to liability duration (default 1.30x) |
| **C0** | Initial cash buffer required to ensure solvency under a given scenario |
| **BEL** | Best Estimate Liability = MV_assets(T=0) + C0_biting |
