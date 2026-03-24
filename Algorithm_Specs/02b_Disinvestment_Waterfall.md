# Algorithm Spec 02b: Disinvestment (Forced Liquidation) Waterfall

## Document Control

| Field | Value |
|---|---|
| **Spec ID** | ALGO-02b |
| **Target Module** | `projection/disinvestment.py` |
| **Supporting Modules** | `curves/bond_pricing.py` (price provider), `projection/engine.py` (caller), `assumptions/transaction_costs.py` (spread tables) |
| **BMA Rule References** | Rules Sched XXV Para 28(9)(c-d), 28(13)-(16), 28(30)-(31); Handbook E11 |
| **Status** | Implementation-Ready Draft |
| **Date** | March 2026 |

---

## 1. Overview

### 1.1 Purpose

This specification defines the algorithm for forced asset liquidation when the projection engine encounters a net cash shortfall in any projection period. The disinvestment waterfall determines **which** assets to sell, **in what order**, **at what price**, and **how much** to sell -- while respecting BMA tier restrictions and applying realistic transaction costs.

The module is the mirror image of reinvestment (ALGO-02a): where reinvestment deploys excess cash into new assets, disinvestment converts existing assets back into cash to cover shortfalls.

### 1.2 Target Module

| Module | Responsibility |
|---|---|
| `projection/disinvestment.py` | Execute the 6-tier liquidation waterfall, compute sale proceeds net of transaction costs, return sold-asset records and any residual deficit (pure Python, no QuantLib) |
| `curves/bond_pricing.py` | Provide clean prices for bonds under the active scenario curve (QuantLib boundary) |
| `assumptions/transaction_costs.py` | Load and serve bid-ask spread tables and grade-in parameters (Pydantic models) |

### 1.3 Key Rule References

| Rule | Subject | Used In |
|---|---|---|
| Para 28(9)(c-d) | Shortfalls must be met by selling assets at scenario's prevailing market yields | Section 2, 5 |
| Para 28(13) | Acceptable assets (Tier 1) -- unrestricted | Section 3 |
| Para 28(14) | Prior approval assets (Tier 2) -- require BMA approval | Section 3 |
| Para 28(15)-(16) | Limited basis assets (Tier 3) -- 10% cap, CANNOT be sold | Section 7 |
| Para 28(30)-(31) | Transaction costs (bid-ask spreads) must be applied | Section 5 |
| Handbook E11 | Best estimate transaction costs, grade-in to long-term average | Section 5 |

### 1.4 Architectural Constraints

- **Pure Python only** -- this module lives in `projection/`, so no QuantLib imports. Bond prices are obtained via a callable interface from `curves/bond_pricing.py`.
- **@rule_ref decorator** on every calculation function, citing BMA paragraph.
- **Fail loudly** -- if an asset has no valid price, the run halts (no silent zero-price assumptions).
- **DataFrame outputs** -- all liquidation actions recorded as pandas DataFrames for audit inspection.
- **No optimizer** -- the disinvestment waterfall is a deterministic priority queue, not an optimization problem. (Optimization is reserved for the combined rebalancing step in ALGO-02a.)

---

## 2. When Disinvestment Occurs

### 2.1 Trigger Condition

Disinvestment is triggered in period `t` when the net cash flow is negative after all income sources have been accounted for:

```
net_cf[t] = asset_coupons[t] + asset_maturities[t] + cash_interest[t]
           - liability_outflows[t] - credit_costs[t] - expenses[t]

IF net_cf[t] < 0:
    shortfall = abs(net_cf[t])
    execute_disinvestment(shortfall, portfolio, scenario_curve, t, rules)
```

### 2.2 Interaction with the Projection Engine

The projection engine (ALGO-02, `projection/engine.py`) calls into this module as **Step 4** of the period loop:

```
projection/engine.py (period loop)
    |
    +-- Step 1: Collect asset cash flows (coupons, maturities, amortization)
    +-- Step 2: Apply credit cost deductions (ALGO-03)
    +-- Step 3: Net against liability outflows
    +-- Step 4: IF shortfall -> disinvestment.execute(...)   <-- THIS MODULE
    |           IF surplus  -> reinvestment.execute(...)      <-- ALGO-02a
    +-- Step 5: Update portfolio state
    +-- Step 6: Record audit trail
```

The disinvestment module receives:
- `shortfall: float` -- the absolute dollar amount that must be raised
- `portfolio: PortfolioState` -- current active assets with positions and metadata
- `price_fn: Callable[[AssetModelPoint, int], float]` -- function that returns clean price for an asset at period `t` (provided by `curves/bond_pricing.py`)
- `t: int` -- current projection period
- `rules: RuleSet` -- active BMA rule version (for transaction cost parameters)

It returns:
- `sales: List[SaleRecord]` -- each sale with asset_id, quantity sold, gross proceeds, transaction cost, net proceeds
- `total_net_proceeds: float` -- sum of net proceeds across all sales
- `residual_deficit: float` -- any shortfall remaining after exhausting all eligible assets (0.0 if fully covered)

---

## 3. The 6-Tier Liquidation Waterfall

### 3.1 Priority Order

Assets are sold in strict priority order from most liquid (cheapest to liquidate) to least liquid (most expensive). The engine exhausts each tier before moving to the next.

| Priority | Asset Category | Tier | Rationale | Typical Txn Cost |
|---|---|---|---|---|
| 1 | Cash and cash equivalents | Tier 1 | Already liquid; zero transaction cost | 0 bps |
| 2 | Government bonds | Tier 1 | Most liquid fixed income; tightest bid-ask | 2-10 bps |
| 3 | Investment-grade corporate bonds | Tier 1 | Deep secondary market; moderate spread | 10-50 bps |
| 4 | Municipal bonds | Tier 1 | Less liquid than corporates; wider spreads | 10-40 bps |
| 5 | Approved structured securities (MBS, ABS, CLO) | Tier 2 | Requires BMA pre-approval; wider spreads | 25-100 bps |
| 6 | Other Tier 2 assets (mortgages, preferred stock) | Tier 2 | Least liquid eligible assets | 50-100 bps |
| **EXCLUDED** | **Tier 3 assets (below-IG, CRE, credit funds)** | **Tier 3** | **CANNOT be sold per Para 28(15)-(16)** | **N/A** |

### 3.2 Mapping AssetType to Waterfall Priority

```python
WATERFALL_PRIORITY: dict[AssetType, int] = {
    AssetType.CASH:           1,
    AssetType.GOVT_BOND:      2,
    AssetType.CORP_BOND_IG:   3,
    AssetType.CALLABLE_BOND:  3,  # IG callables treated same as IG corporates
    AssetType.FLOATING_RATE:  3,  # IG floaters treated same as IG corporates
    AssetType.AMORTIZING:     3,  # IG amortizing treated same as IG corporates
    AssetType.MUNI_BOND:      4,
    AssetType.MBS:            5,
    AssetType.ABS:            5,
    AssetType.CLO:            5,
    AssetType.PREFERRED_STOCK: 6,
    AssetType.MORTGAGE_LOAN:  6,
}

# These asset types are NEVER eligible for liquidation
TIER3_EXCLUDED: set[AssetType] = {
    AssetType.CORP_BOND_HY,
    AssetType.COMMERCIAL_RE,
}
```

**Design decision:** The mapping is a static dictionary rather than a runtime lookup to keep the waterfall deterministic and auditable. If BMA rules change the priority order, only this dictionary needs updating (versioned in `rules/v20XX/`).

---

## 4. Within-Tier Selection Logic

### 4.1 The Decision: Pro-Rata by Market Value

When a tier contains multiple assets and only a partial sale from that tier is needed, the algorithm sells **pro-rata by current market value**.

**Rationale:**

| Alternative | Pros | Cons |
|---|---|---|
| **Pro-rata by MV** (chosen) | Preserves portfolio diversification within the tier; no duration bias; simple and auditable | May sell some of a high-duration bond when a low-duration one would suffice |
| Shortest-duration-first | Minimizes price sensitivity of sold assets | Concentrates remaining portfolio in long-duration assets, increasing risk; creates selection bias |
| Largest-position-first | Reduces concentration | Arbitrary; no economic rationale |
| Lowest-yield-first | Preserves highest-yielding assets | Penalizes safer assets; gaming risk |

Pro-rata by MV is the standard regulatory benchmark assumption. It is neutral, transparent, and does not introduce optimization assumptions into what should be a mechanical liquidation process.

### 4.2 Pro-Rata Calculation

For assets `{a1, a2, ..., an}` in the current tier with market values `{mv1, mv2, ..., mvn}`:

```
tier_total_mv = sum(mv_i for i in 1..n)
sale_weight_i = mv_i / tier_total_mv

amount_to_sell_from_tier = min(remaining_shortfall_after_txn_cost_estimate, tier_total_mv)

# Each asset's share of the sale
target_sale_mv_i = amount_to_sell_from_tier * sale_weight_i
```

The actual face amount to sell is then derived from the asset's current clean price (Section 5).

---

## 5. Forced Sale Pricing

### 5.1 Clean Price at Scenario Curve

For each bond asset, the sale price is the clean price implied by the scenario's prevailing yield curve at projection period `t`. This price is **not** the book value or par -- it reflects the interest rate environment of the active scenario.

```
Interface contract (provided by curves/bond_pricing.py):

    price_fn(asset: AssetModelPoint, t: int) -> float
        Returns: clean price per 100 face value under the active scenario curve
        Raises: PricingError if asset cannot be priced

    Internally, curves/bond_pricing.py calls:
        ql.BondFunctions.cleanPrice(bond, scenario_curve_at_t)
```

The `projection/disinvestment.py` module **never imports QuantLib**. It receives `price_fn` as an injected dependency.

### 5.2 Transaction Cost Deduction

Transaction costs represent the bid-ask spread cost of selling in secondary markets.

```python
@rule_ref("Para 28(30)-(31)", "Handbook E11")
def transaction_cost(notional: float, asset_type: AssetType,
                     t: int, rules: RuleSet) -> float:
    """
    Compute one-way (sell-side) transaction cost.

    cost = notional * effective_half_spread(asset_type, t)
    """
    current_spread = rules.bid_ask_spread(asset_type, "current")  # bps
    lt_average     = rules.bid_ask_spread(asset_type, "long_term") # bps

    # Grade-in rule: if current < long-term, grade linearly over 10 years
    if current_spread < lt_average:
        grading_factor = min(t / 10.0, 1.0)
        effective_spread = current_spread + (lt_average - current_spread) * grading_factor
    else:
        effective_spread = current_spread

    half_spread_decimal = effective_spread / 10_000 / 2  # bps -> decimal, one side
    return notional * half_spread_decimal
```

**Grade-in rule (Handbook E11):** If current market spreads are tighter than long-term averages (which is common in benign markets), the model grades in to the long-term average linearly over 10 years. This prevents understating liquidation costs in the early projection years and then facing reality later.

### 5.3 Bid-Ask Spread Table

Loaded from `assumptions/tables/transaction_costs.yaml`:

| Asset Category | `AssetType` values | Current Spread (bps) | Long-Term Average (bps) |
|---|---|---|---|
| Cash | `CASH` | 0 | 0 |
| Government bonds | `GOVT_BOND` | 2-5 | 5-10 |
| IG Corporate | `CORP_BOND_IG`, `CALLABLE_BOND`, `FLOATING_RATE`, `AMORTIZING` | 10-25 | 25-50 |
| Municipal | `MUNI_BOND` | 10-20 | 20-40 |
| Structured (IG) | `MBS`, `ABS`, `CLO` | 25-50 | 50-100 |
| Other Tier 2 | `PREFERRED_STOCK`, `MORTGAGE_LOAN` | 50-100 | 100-200 |

*Note: Exact values are configurable per valuation date. The ranges above are illustrative; production values are loaded from `assumptions/tables/`.*

### 5.4 Net Proceeds Calculation

For selling face amount `F` of an asset:

```
gross_proceeds = F * clean_price / 100           # clean_price is per 100 face
txn_cost       = transaction_cost(F, asset_type, t, rules)
net_proceeds   = gross_proceeds - txn_cost

# The face amount removed from the portfolio is F
# The cash received is net_proceeds
```

### 5.5 Price Interface: How projection/ Gets Prices from curves/

The architectural boundary requires a clean interface:

```python
# In projection/disinvestment.py -- NO QuantLib imports

from typing import Protocol

class BondPricer(Protocol):
    """Interface for obtaining bond clean prices from the curves/ module."""

    def clean_price(self, asset: AssetModelPoint, period: int) -> float:
        """Return clean price per 100 face for the asset under the active scenario.

        Args:
            asset: The asset model point (contains coupon, maturity, etc.)
            period: Projection period (0 = valuation date, 1 = end of year 1, etc.)

        Returns:
            Clean price per 100 face value.

        Raises:
            PricingError: If the asset cannot be priced (missing data, expired, etc.)
        """
        ...

# The projection engine injects a concrete implementation at runtime:
#   pricer = ScenarioBondPricer(scenario_curve, valuation_date)  # from curves/
#   disinvestment.execute(shortfall, portfolio, pricer, t, rules)
```

---

## 6. Partial Liquidation

### 6.1 Why Partial Sales Are Necessary

In most periods, the shortfall will not require liquidating entire positions. The waterfall must support selling a fraction of a bond holding.

### 6.2 Mechanics

Each `AssetModelPoint` in the portfolio tracks a `face_amount` (or `par_amount`). A partial sale reduces this amount:

```python
@dataclass
class SaleRecord:
    asset_id: str
    asset_type: AssetType
    waterfall_priority: int
    face_sold: float          # par amount liquidated
    clean_price: float        # per 100 face, at scenario curve
    gross_proceeds: float     # face_sold * clean_price / 100
    txn_cost: float           # one-way bid-ask cost
    net_proceeds: float       # gross_proceeds - txn_cost
    remaining_face: float     # face_amount after this sale
    period: int
    rule_ref: str             # "Para 28(9)(c-d)"
```

After a partial sale:
- The asset remains in the portfolio with `face_amount -= face_sold`
- Future coupon payments scale proportionally: `coupon[t+1] = coupon_rate * remaining_face`
- Future maturity proceeds scale proportionally

### 6.3 Minimum Sale Granularity

For practical purposes, if a partial sale would leave a remaining face amount below a configurable minimum (default: $1,000), the entire position is sold instead. This avoids tracking economically insignificant residual positions.

```python
MIN_RESIDUAL_FACE = 1_000  # Configurable via rules

if remaining_face < MIN_RESIDUAL_FACE:
    face_to_sell = asset.face_amount  # sell the whole thing
```

---

## 7. Tier 3 Exclusion Enforcement

### 7.1 Rule

Per Para 28(15)-(16), assets acceptable on a limited basis (Tier 3) **cannot be sold** to meet cash flow shortfalls. They remain in the portfolio for the entire projection, generating their (credit-adjusted) cash flows, but are never candidates for liquidation.

### 7.2 Implementation

Tier 3 exclusion is enforced at two points:

1. **Waterfall entry:** Tier 3 assets are filtered out before the waterfall begins.
2. **Validation:** A defensive check that no `SaleRecord` references a Tier 3 asset.

```python
@rule_ref("Para 28(15)-(16)")
def filter_eligible_assets(portfolio: PortfolioState) -> List[AssetModelPoint]:
    """Return only assets eligible for liquidation (Tier 1 + approved Tier 2).

    Tier 3 assets are excluded unconditionally.
    Tier 2 assets without bma_approved=True are also excluded.
    """
    eligible = []
    for asset in portfolio.active_assets:
        if asset.asset_type in TIER3_EXCLUDED:
            continue  # Tier 3: cannot sell
        if asset.liquidity_tier == 2 and not asset.bma_approved:
            continue  # Tier 2 without approval: cannot sell
        eligible.append(asset)
    return eligible
```

### 7.3 Tier 3 Impact on Deficit

Because Tier 3 assets are locked, a portfolio with a large Tier 3 allocation may exhaust all eligible assets sooner, producing a deficit. This is by design -- it penalizes illiquid portfolios appropriately and feeds back into the C0 root-finding loop (a higher C0 is needed to compensate).

---

## 8. Deficit Handling

### 8.1 When All Eligible Assets Are Exhausted

If the waterfall runs through all 6 priority tiers and the shortfall is still not fully covered:

```
residual_deficit = remaining_shortfall  # > 0
```

This is **not** an error. It is a signal to the C0 root-finding algorithm (ALGO-02, Section 4.4) that the current trial value of C0 is too low. The root-finder will increase C0 and re-run the projection until `residual_deficit == 0` in all periods for the scenario.

### 8.2 Recording the Deficit

```python
@dataclass
class DeficitRecord:
    period: int
    shortfall_requested: float    # original shortfall amount
    total_net_proceeds: float     # what the waterfall actually raised
    residual_deficit: float       # shortfall - total_net_proceeds
    eligible_assets_remaining: int  # should be 0 if deficit occurred
    tier3_assets_locked: int      # count of Tier 3 assets that could not be sold
    rule_ref: str                 # "Para 28(9)(c-d)"
```

The deficit record is appended to the audit trail. The projection engine continues to the next period (the deficit is effectively carried as negative cash, which compounds the problem in subsequent periods -- exactly the signal the root-finder needs).

### 8.3 Interaction with C0 Root-Finding

```
C0 root-finding loop (scipy.optimize.brentq):
    for each trial C0:
        run full projection for scenario
        if any period has residual_deficit > 0:
            return +1  (C0 too low, sign for brentq)
        if final_cash_balance > 0:
            return -1  (C0 too high)
        return 0       (exact match)
```

The deficit directly drives the root-finding convergence: more deficit means C0 must increase.

---

## 9. Complete Algorithm: Pseudocode

### 9.1 Main Entry Point

```python
@rule_ref("Para 28(9)(c-d)")
def execute_disinvestment(
    shortfall: float,
    portfolio: PortfolioState,
    pricer: BondPricer,
    t: int,
    rules: RuleSet,
) -> DisinvestmentResult:
    """
    Execute the disinvestment waterfall to raise cash for a shortfall.

    Args:
        shortfall: Positive dollar amount that must be raised.
        portfolio: Current portfolio state with active asset positions.
        pricer: Injected bond pricer (from curves/ module).
        t: Current projection period.
        rules: Active BMA rule version.

    Returns:
        DisinvestmentResult with sales list, total proceeds, and any deficit.
    """
    assert shortfall > 0, "Disinvestment called with non-positive shortfall"

    # Step 1: Filter to eligible assets only (exclude Tier 3, unapproved Tier 2)
    eligible = filter_eligible_assets(portfolio)

    # Step 2: Group by waterfall priority and sort
    tiers = group_by_priority(eligible)  # dict[int, List[AssetModelPoint]]
    sorted_priorities = sorted(tiers.keys())  # [1, 2, 3, 4, 5, 6]

    sales: List[SaleRecord] = []
    remaining_shortfall = shortfall

    # Step 3: Walk down the waterfall
    for priority in sorted_priorities:
        if remaining_shortfall <= 0:
            break

        tier_assets = tiers[priority]

        if priority == 1:
            # Cash: direct draw-down, no pricing needed, no transaction cost
            cash_available = sum(a.face_amount for a in tier_assets)
            cash_used = min(cash_available, remaining_shortfall)
            if cash_used > 0:
                sales.extend(
                    sell_cash(tier_assets, cash_used, t)
                )
                remaining_shortfall -= cash_used
        else:
            # Bonds: price, compute transaction costs, sell pro-rata
            tier_sales, proceeds = sell_from_tier(
                tier_assets, remaining_shortfall, pricer, t, rules, priority
            )
            sales.extend(tier_sales)
            remaining_shortfall -= proceeds

    # Step 4: Record result
    total_net = sum(s.net_proceeds for s in sales)
    residual = max(remaining_shortfall, 0.0)

    if residual > 0:
        deficit = DeficitRecord(
            period=t,
            shortfall_requested=shortfall,
            total_net_proceeds=total_net,
            residual_deficit=residual,
            eligible_assets_remaining=0,
            tier3_assets_locked=count_tier3(portfolio),
            rule_ref="Para 28(9)(c-d)",
        )
    else:
        deficit = None

    return DisinvestmentResult(sales=sales, total_net_proceeds=total_net,
                               residual_deficit=residual, deficit_record=deficit)
```

### 9.2 Selling from a Bond Tier (Pro-Rata)

```python
def sell_from_tier(
    tier_assets: List[AssetModelPoint],
    target_amount: float,
    pricer: BondPricer,
    t: int,
    rules: RuleSet,
    priority: int,
) -> Tuple[List[SaleRecord], float]:
    """
    Sell assets from a single tier pro-rata by MV to raise target_amount.

    Returns:
        (list of SaleRecords, total net proceeds raised)
    """
    # Price all assets in the tier
    priced = []
    for asset in tier_assets:
        cp = pricer.clean_price(asset, t)  # per 100 face
        mv = asset.face_amount * cp / 100
        priced.append((asset, cp, mv))

    tier_total_mv = sum(mv for _, _, mv in priced)
    if tier_total_mv <= 0:
        return [], 0.0

    # Estimate: can this tier cover the target?
    # We need to account for transaction costs eating into proceeds.
    # Use an iterative approach: estimate gross needed, then refine.
    sales = []
    total_net = 0.0

    # Compute the fraction of the tier to sell
    # First estimate: assume ~worst-case txn cost to avoid underselling
    estimated_txn_rate = max_txn_rate_for_priority(priority, t, rules)
    gross_needed = target_amount / (1 - estimated_txn_rate)
    sell_fraction = min(gross_needed / tier_total_mv, 1.0)

    for asset, cp, mv in priced:
        target_sale_mv = mv * sell_fraction
        face_to_sell = target_sale_mv / (cp / 100)

        # Enforce minimum residual
        if asset.face_amount - face_to_sell < MIN_RESIDUAL_FACE:
            face_to_sell = asset.face_amount

        # Cap at available
        face_to_sell = min(face_to_sell, asset.face_amount)

        gross = face_to_sell * cp / 100
        txn = transaction_cost(face_to_sell, asset.asset_type, t, rules)
        net = gross - txn

        if net <= 0:
            continue  # Skip if transaction cost exceeds gross proceeds

        sale = SaleRecord(
            asset_id=asset.asset_id,
            asset_type=asset.asset_type,
            waterfall_priority=priority,
            face_sold=face_to_sell,
            clean_price=cp,
            gross_proceeds=gross,
            txn_cost=txn,
            net_proceeds=net,
            remaining_face=asset.face_amount - face_to_sell,
            period=t,
            rule_ref="Para 28(9)(c-d)",
        )
        sales.append(sale)
        total_net += net

        # Update portfolio in-place
        asset.face_amount -= face_to_sell

    return sales, total_net
```

### 9.3 Selling Cash (Priority 1)

```python
def sell_cash(
    cash_assets: List[AssetModelPoint],
    amount: float,
    t: int,
) -> List[SaleRecord]:
    """Draw down cash positions. No pricing, no transaction cost."""
    sales = []
    remaining = amount

    for asset in cash_assets:
        if remaining <= 0:
            break
        draw = min(asset.face_amount, remaining)
        sale = SaleRecord(
            asset_id=asset.asset_id,
            asset_type=AssetType.CASH,
            waterfall_priority=1,
            face_sold=draw,
            clean_price=100.0,  # cash is always at par
            gross_proceeds=draw,
            txn_cost=0.0,
            net_proceeds=draw,
            remaining_face=asset.face_amount - draw,
            period=t,
            rule_ref="Para 28(9)(c-d)",
        )
        sales.append(sale)
        remaining -= draw
        asset.face_amount -= draw

    return sales
```

---

## 10. Worked Example

### 10.1 Setup

**Period:** `t = 3`
**Shortfall:** $800 (liability outflows exceed asset cash flows by $800)
**Scenario:** Base case (no rate shift), so risk-free curve is flat at 3.0%

**Portfolio at start of period 3:**

| Asset ID | Type | Tier | Face Amount | Clean Price (per 100) | Market Value | Eligible? |
|---|---|---|---|---|---|---|
| CASH-001 | Cash | 1 | $200 | 100.00 | $200 | Yes |
| UST-5Y | Govt Bond | 1 | $300 | 98.50 | $295.50 | Yes |
| CORP-A1 | IG Corporate | 1 | $250 | 96.20 | $240.50 | Yes |
| CORP-A2 | IG Corporate | 1 | $150 | 97.10 | $145.65 | Yes |
| HY-001 | HY Corporate | 3 | $100 | 89.00 | $89.00 | **No (Tier 3)** |

**Transaction cost parameters (for this example):**

| Asset Category | Effective Half-Spread at t=3 |
|---|---|
| Cash | 0 bps |
| Govt Bond | 4.9 bps (current 3 bps, LT avg 10 bps, graded: 3 + (10-3)*0.3 = 5.1 bps, half = ~2.55 bps... see below) |
| IG Corporate | 22 bps (current 15 bps, LT avg 40 bps, graded: 15 + (40-15)*0.3 = 22.5 bps, half = 11.25 bps) |

Let us be precise. The formula is:

```
transaction_cost = notional * effective_spread / 10_000 / 2
```

For `t = 3`, `grading_factor = min(3/10, 1.0) = 0.3`:

- **Govt bonds:** current = 5 bps, LT avg = 10 bps. effective = 5 + (10-5)*0.3 = 6.5 bps. Half-spread = 6.5/2 = 3.25 bps = 0.000325 of notional.
- **IG Corporate:** current = 20 bps, LT avg = 40 bps. effective = 20 + (40-20)*0.3 = 26 bps. Half-spread = 26/2 = 13 bps = 0.0013 of notional.

### 10.2 Step-by-Step Execution

**Step 1: Filter eligible assets.**

HY-001 is Tier 3 -- excluded. Eligible portfolio:

| Asset ID | Type | Priority | Face | Clean Price | MV |
|---|---|---|---|---|---|
| CASH-001 | Cash | 1 | $200 | 100.00 | $200.00 |
| UST-5Y | Govt Bond | 2 | $300 | 98.50 | $295.50 |
| CORP-A1 | IG Corp | 3 | $250 | 96.20 | $240.50 |
| CORP-A2 | IG Corp | 3 | $150 | 97.10 | $145.65 |

**Remaining shortfall: $800.00**

---

**Step 2: Priority 1 -- Cash.**

- Cash available: $200.00
- Draw: min($200, $800) = $200.00
- Transaction cost: $0
- Net proceeds: $200.00

| Sale | Face Sold | Gross | Txn Cost | Net |
|---|---|---|---|---|
| CASH-001 | $200.00 | $200.00 | $0.00 | $200.00 |

**Remaining shortfall: $800 - $200 = $600.00**

---

**Step 3: Priority 2 -- Government bonds.**

Only UST-5Y in this tier.

- MV of tier: $295.50
- Need to raise: $600.00 (exceeds tier MV, so sell entire position)
- Face to sell: $300 (entire position)

Compute proceeds:
```
gross_proceeds = 300 * 98.50 / 100 = $295.50
txn_cost       = 300 * 0.000325    = $0.10
net_proceeds   = 295.50 - 0.10     = $295.40
```

| Sale | Face Sold | Gross | Txn Cost | Net |
|---|---|---|---|---|
| UST-5Y | $300.00 | $295.50 | $0.10 | $295.40 |

**Remaining shortfall: $600.00 - $295.40 = $304.60**

---

**Step 4: Priority 3 -- IG Corporate bonds.**

Two assets in this tier: CORP-A1 (MV $240.50) and CORP-A2 (MV $145.65).

- Tier total MV: $240.50 + $145.65 = $386.15
- Need to raise: $304.60 (less than tier total, so partial sale)

**Pro-rata allocation:**
```
CORP-A1 weight = 240.50 / 386.15 = 0.6228
CORP-A2 weight = 145.65 / 386.15 = 0.3772
```

We need $304.60 net. Estimate gross needed (accounting for ~13 bps txn cost rate on face):
```
# Approximate: gross_needed ~ 304.60 / (1 - 0.0013) ~ 305.00
# But we sell by MV fraction, so:
sell_fraction = 305.00 / 386.15 = 0.7898
```

**CORP-A1:**
```
target_sale_mv = 240.50 * 0.7898 = $189.95
face_to_sell   = 189.95 / (96.20/100) = $197.45
gross_proceeds = 197.45 * 96.20 / 100 = $189.95
txn_cost       = 197.45 * 0.0013      = $0.26
net_proceeds   = 189.95 - 0.26        = $189.69
```

**CORP-A2:**
```
target_sale_mv = 145.65 * 0.7898 = $115.04
face_to_sell   = 115.04 / (97.10/100) = $118.47
gross_proceeds = 118.47 * 97.10 / 100 = $115.03
txn_cost       = 118.47 * 0.0013      = $0.15
net_proceeds   = 115.03 - 0.15        = $114.88
```

Total net from Priority 3: $189.69 + $114.88 = **$304.57**

| Sale | Face Sold | Gross | Txn Cost | Net |
|---|---|---|---|---|
| CORP-A1 | $197.45 | $189.95 | $0.26 | $189.69 |
| CORP-A2 | $118.47 | $115.03 | $0.15 | $114.88 |

**Remaining shortfall: $304.60 - $304.57 = $0.03**

---

**Step 5: Residual.**

The remaining $0.03 is a rounding residual from the sell-fraction estimate. In practice, the implementation uses an iterative adjustment: if after the pro-rata pass there is a small residual (< $1.00), sell an additional sliver from the largest remaining position. Alternatively, the tolerance threshold for "fully covered" is configurable (default: $0.50).

**Final result: shortfall fully covered (within tolerance).**

### 10.3 Summary

| Priority | Tier | Assets Sold | Net Proceeds | Cumulative |
|---|---|---|---|---|
| 1 | Cash | CASH-001 | $200.00 | $200.00 |
| 2 | Govt Bond | UST-5Y | $295.40 | $495.40 |
| 3 | IG Corp | CORP-A1, CORP-A2 | $304.57 | $799.97 |
| **Total** | | | **$799.97** | |

Total transaction costs incurred: $0.10 + $0.26 + $0.15 = **$0.51**

Shortfall requested: $800.00. Residual: $0.03 (within tolerance).

### 10.4 Portfolio After Disinvestment

| Asset ID | Type | Face Before | Face After | Status |
|---|---|---|---|---|
| CASH-001 | Cash | $200.00 | $0.00 | Fully liquidated |
| UST-5Y | Govt Bond | $300.00 | $0.00 | Fully liquidated |
| CORP-A1 | IG Corp | $250.00 | $52.55 | Partial -- continues in portfolio |
| CORP-A2 | IG Corp | $150.00 | $31.53 | Partial -- continues in portfolio |
| HY-001 | HY Corp (Tier 3) | $100.00 | $100.00 | Untouched (cannot sell) |

---

## 11. Edge Cases

### 11.1 Single Asset in a Tier

If a tier has only one asset, the pro-rata logic degenerates to selling from that single asset. No special handling needed -- weight = 1.0.

### 11.2 Bond Trading at Par or Above

A bond with clean price >= 100 generates gross proceeds >= face amount. The algorithm works identically; no adjustment needed. Transaction costs still apply.

### 11.3 Zero or Near-Zero Market Value

An asset with MV near zero (e.g., deeply distressed or near-maturity with minimal remaining cash flows) should still be included in the waterfall but will contribute negligible proceeds. If `clean_price < 0.01` (per 100 face), skip the asset and log a warning -- selling it would produce effectively zero proceeds at a real transaction cost.

```python
if cp < 0.01:
    logger.warning(f"Asset {asset.asset_id} has near-zero price ({cp}); skipping in waterfall")
    continue
```

### 11.4 Negative Net Proceeds After Transaction Costs

For very small sales or very illiquid assets, transaction costs could theoretically exceed gross proceeds. The algorithm skips such sales:

```python
if net_proceeds <= 0:
    continue  # Selling this asset would lose money; skip it
```

This is logged but does not halt the run. The shortfall will either be covered by other assets or recorded as a deficit.

### 11.5 Asset Matures in the Current Period

If a bond matures at period `t`, its maturity cash flow has already been collected in Step 1 of the projection loop (before disinvestment is called). It should not appear in the active portfolio at disinvestment time. If it does appear (implementation bug), its clean price would be approximately par, and selling it is equivalent to collecting the maturity -- a defensive safeguard but not expected behavior.

### 11.6 Callable Bond Called Away

If a callable bond has been called in period `t`, it should have been removed from the portfolio before disinvestment. Same as 11.5 -- defensive check only.

### 11.7 Entire Portfolio Is Tier 3

If every asset in the portfolio is Tier 3 (an extreme and unlikely edge case), the eligible set is empty, and the full shortfall becomes a deficit immediately. This would indicate a fundamentally unsuitable portfolio for SBA purposes.

---

## 12. Implementation Notes

### 12.1 Interface with the Projection Engine

The projection engine calls `execute_disinvestment()` and uses the returned `DisinvestmentResult` to:

1. **Update portfolio state:** Remove fully liquidated assets; reduce face amounts of partially sold assets.
2. **Update cash balance:** `cash_balance[t] += total_net_proceeds - shortfall` (if proceeds > shortfall, the excess stays as cash; this should not happen since we sell only enough to cover the shortfall).
3. **Record audit trail:** Append all `SaleRecord` entries and any `DeficitRecord` to the period's audit log.
4. **Signal root-finder:** If `residual_deficit > 0`, the projection returns a positive-signed result to `brentq`, indicating C0 must increase.

### 12.2 Audit Trail

Every sale generates a `SaleRecord` dataclass (Section 6.2). These are collected into a `pd.DataFrame` for the run:

```python
columns = [
    "period", "asset_id", "asset_type", "waterfall_priority",
    "face_sold", "clean_price", "gross_proceeds", "txn_cost",
    "net_proceeds", "remaining_face", "rule_ref"
]
```

This DataFrame is:
- Persisted to Parquet as part of the run output
- Included in the multi-tab Excel workbook (output module)
- Available for challenge-mode comparison against company submissions

### 12.3 Performance Considerations

The disinvestment waterfall is called at most once per period per scenario (only when `net_cf < 0`). With 100 projection periods and 9 scenarios, worst case is 900 calls per C0 trial. Each call iterates over at most ~100-500 assets (typical SBA portfolio size). This is well within performance bounds for pure Python -- no vectorization or optimization needed.

**Bottleneck note:** The `pricer.clean_price()` call crosses into QuantLib (via `curves/bond_pricing.py`). If pricing is slow, the optimization is to pre-compute a price grid for all assets at all periods before the projection loop starts, then use dictionary lookup in the waterfall. This is a `curves/` module concern, not a `projection/` concern.

### 12.4 Testing Strategy

| Test | Description |
|---|---|
| `test_cash_only_shortfall` | Shortfall < cash balance; only cash drawn, no bonds sold |
| `test_single_tier_exhaustion` | Shortfall requires full liquidation of one tier |
| `test_multi_tier_waterfall` | Shortfall cascades through 3+ tiers (like the worked example) |
| `test_tier3_excluded` | Verify Tier 3 assets are never sold regardless of shortfall size |
| `test_pro_rata_within_tier` | Two assets in same tier; verify proportional sale |
| `test_partial_sale_residual` | Verify face amount correctly reduced, not zeroed |
| `test_deficit_recorded` | Shortfall exceeds all eligible MV; deficit properly recorded |
| `test_txn_cost_gradein` | Verify grade-in formula at t=0, t=5, t=10, t=15 |
| `test_negative_net_proceeds` | Asset where txn cost > gross; verify it is skipped |
| `test_golden_integration` | Full projection with known shortfall periods; match expected waterfall actions |

### 12.5 Configuration Points

| Parameter | Default | Source | Description |
|---|---|---|---|
| `WATERFALL_PRIORITY` | See Section 3.2 | `rules/v20XX/disinvestment.py` | Asset type to priority mapping |
| `MIN_RESIDUAL_FACE` | $1,000 | `rules/v20XX/disinvestment.py` | Minimum face to keep; below this, sell entire position |
| `SHORTFALL_TOLERANCE` | $0.50 | `rules/v20XX/disinvestment.py` | Residual below this is treated as zero |
| Bid-ask spread table | See Section 5.3 | `assumptions/tables/transaction_costs.yaml` | Per-asset-type current and LT spreads |
| Grade-in period | 10 years | `assumptions/tables/transaction_costs.yaml` | Linear grade-in horizon |

---

## Appendix A: Data Structures Summary

```python
@dataclass(frozen=True)
class DisinvestmentResult:
    """Complete result of a disinvestment waterfall execution."""
    sales: List[SaleRecord]
    total_net_proceeds: float
    residual_deficit: float          # 0.0 if shortfall fully covered
    deficit_record: Optional[DeficitRecord]

    @property
    def shortfall_covered(self) -> bool:
        return self.residual_deficit <= SHORTFALL_TOLERANCE
```

## Appendix B: Relationship to Other Specs

| Spec | Relationship |
|---|---|
| ALGO-01 (Yield Curve Construction) | Provides the scenario curves used by `curves/bond_pricing.py` to price assets for forced sale |
| ALGO-02 (Projection Engine) | Caller of this module; provides the `shortfall` trigger and consumes the `DisinvestmentResult` |
| ALGO-02a (Reinvestment Strategy) | Mirror operation: deploys excess cash. The two modules are mutually exclusive per period (either reinvest or disinvest, never both) |
| ALGO-03 (Credit Costs D&D) | Credit cost deductions are applied *before* the net CF calculation; they can contribute to triggering disinvestment |
| ALGO-04 (Spread Cap Enforcement) | Spread caps affect asset pricing, which in turn affects forced sale proceeds |
