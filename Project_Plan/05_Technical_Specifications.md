# BMA SBA Benchmark Model - Technical Specifications

## Document Control

| Field | Value |
|---|---|
| **Project Name** | BMA SBA Benchmark Model |
| **Document Type** | Technical Specifications |
| **Version** | 1.0 (Draft for Discussion) |
| **Date** | March 2026 |

---

## 1. BMA Interest Rate Scenarios

### 1.1 The 9 Prescribed Scenarios

All scenarios modify the base risk-free yield curve. Shifts are in basis points (bps).

| ID | Name | Short End Shift | Long End Shift | Temporal Pattern |
|---|---|---|---|---|
| 1 | **Base** | 0 | 0 | No change |
| 2 | **Decrease** | -150 bps | -150 bps | Grade linearly to full shift by year 10, flat after |
| 3 | **Increase** | +150 bps | +150 bps | Grade linearly to full shift by year 10, flat after |
| 4 | **Down-Up** | -150 bps peak | -150 bps peak | Grade to -150 by year 5, return to base by year 10 |
| 5 | **Up-Down** | +150 bps peak | +150 bps peak | Grade to +150 by year 5, return to base by year 10 |
| 6 | **Decrease + Positive Twist** | -150 bps | -50 bps | Grade linearly by year 10, flat after |
| 7 | **Decrease + Negative Twist** | -50 bps | -150 bps | Grade linearly by year 10, flat after |
| 8 | **Increase + Positive Twist** | +50 bps | +150 bps | Grade linearly by year 10, flat after |
| 9 | **Increase + Negative Twist** | +150 bps | +50 bps | Grade linearly by year 10, flat after |

**Rule Reference:** Rules Schedule XXV, Paragraph 28(7)(a-i)

### 1.2 Scenario Shift Formulas

**Parallel Scenarios (2, 3):**
```
shift(year, scenario) =
    if year <= 10: max_shift * (year / 10)
    else:          max_shift
```

**Hump Scenarios (4, 5):**
```
shift(year, scenario) =
    if year <= 5:  max_shift * (year / 5)
    if year <= 10: max_shift * (10 - year) / 5
    else:          0
```

**Twist Scenarios (6, 7, 8, 9):**
```
shift(year, tenor, scenario) =
    short_shift + (long_shift - short_shift) * (tenor / max_tenor)
    Applied with same temporal grading as parallel scenarios
```

### 1.3 Scenario Implementation Notes

- "Short end" = tenors <= 1 year
- "Long end" = tenors >= 20 years
- Intermediate tenors: linear interpolation between short and long shifts
- Minimum rate floor: 0 bps (rates cannot go negative, per BMA guidance) - **CONFIRM with BMA**
- Projection frequency: **Annual** (minimum required by BMA)

---

## 2. Asset Types and Modeling

### 2.1 Asset Tier Classification

| Tier | Category | Examples | Restrictions | BMA Approval Required |
|---|---|---|---|---|
| **Tier 1** | Acceptable without restriction | Government bonds, municipal bonds, IG corporate bonds, cash | None | No |
| **Tier 2** | Acceptable with prior approval | Private assets, structured securities (MBS, ABS, CLO), residential/commercial mortgages, IG preferred stock | Must have BMA pre-approval | **Yes** |
| **Tier 3** | Acceptable on limited basis | Below-investment-grade assets, commercial real estate, credit funds | **10% cap** of total SBA portfolio; **Cannot be sold** to meet shortfalls | No |

**Rule Reference:** Rules Schedule XXV, Paragraphs 28(13-16)

### 2.2 Asset Model Point Schema

```python
class AssetModelPoint(BaseModel):
    """One record per asset in the portfolio."""
    asset_id: str                    # Unique identifier
    asset_type: AssetType            # Enum: see below
    issuer_name: str                 # For reporting
    par_value: float                 # Face/par value (> 0)
    book_value: float                # Carrying value
    market_value: float              # Mark-to-market value
    coupon_rate: float               # Annual coupon rate (decimal, e.g., 0.05 = 5%)
    coupon_frequency: int            # Payments per year (1, 2, 4)
    issue_date: date                 # Bond issue date
    maturity_date: date              # Final maturity
    currency: str                    # ISO 4217 (e.g., "USD", "BMD")
    rating: CreditRating             # Enum: AAA through D
    tier: Literal[1, 2, 3]           # BMA tier classification
    is_callable: bool = False        # Has call option
    call_schedule: Optional[List[CallDate]] = None  # If callable
    is_amortizing: bool = False      # Has amortization schedule
    amort_schedule: Optional[List[AmortPayment]] = None
    prepayment_speed: Optional[float] = None  # PSA speed for MBS (e.g., 150 = 150% PSA)
    sector: str = ""                 # For reporting grouping
    bma_approved: bool = True        # Tier 2 assets must have BMA approval flag
```

### 2.3 Asset Type Enumeration

```python
class AssetType(str, Enum):
    GOVT_BOND = "GOVT_BOND"          # Government/sovereign bonds
    MUNI_BOND = "MUNI_BOND"          # Municipal bonds
    CORP_BOND_IG = "CORP_BOND_IG"    # Investment-grade corporate bonds
    CORP_BOND_HY = "CORP_BOND_HY"    # High-yield corporate bonds (Tier 3)
    CALLABLE_BOND = "CALLABLE_BOND"  # Callable fixed-rate bonds
    FLOATING_RATE = "FLOATING_RATE"  # Floating-rate notes
    AMORTIZING = "AMORTIZING"        # Amortizing/sinking fund bonds
    MBS_AGENCY = "MBS_AGENCY"        # Agency MBS
    MBS_NON_AGENCY = "MBS_NON_AGENCY" # Non-agency MBS
    ABS = "ABS"                      # Asset-backed securities
    CLO = "CLO"                      # Collateralized loan obligations
    PREFERRED_STOCK = "PREFERRED_STOCK" # IG preferred stock
    MORTGAGE_LOAN = "MORTGAGE_LOAN"  # Residential/commercial mortgage loans
    COMMERCIAL_RE = "COMMERCIAL_RE"  # Commercial real estate (Tier 3)
    CASH = "CASH"                    # Cash and cash equivalents
```

### 2.4 QuantLib Instrument Mapping

| Asset Type | QuantLib Class | Pricing Engine | Notes |
|---|---|---|---|
| GOVT_BOND, MUNI_BOND, CORP_BOND_IG/HY | `ql.FixedRateBond` | `ql.DiscountingBondEngine` | Standard bullet bond |
| CALLABLE_BOND | `ql.CallableFixedRateBond` | `ql.TreeCallableFixedRateBondEngine` | Hull-White one-factor model |
| FLOATING_RATE | `ql.FloatingRateBond` | `ql.DiscountingBondEngine` | Requires forward rate estimation |
| AMORTIZING | `ql.AmortizingFixedRateBond` | `ql.DiscountingBondEngine` | Custom notional schedule |
| MBS_AGENCY/NON_AGENCY | Custom or AbsBox | Custom | PSA prepayment model |
| ABS, CLO | Custom | Custom | Simplified CF model |
| CASH | N/A (no QuantLib) | N/A | Earns risk-free rate |

---

## 3. Liability Cash Flow Input (EPL)

### 3.1 EPL Schema

```python
class EPLCashFlow(BaseModel):
    """External Projected Liabilities - one record per period."""
    period: int                      # Projection year (1, 2, ..., T_max)
    liability_cf: float              # Net liability cash outflow this period
    block_id: str = "DEFAULT"        # Liability block identifier
    currency: str = "USD"
    description: str = ""            # Optional description
```

### 3.2 EPL Format Requirements

| Field | Type | Constraints |
|---|---|---|
| period | int | Sequential from 1, no gaps |
| liability_cf | float | Positive = cash outflow (claims, benefits, expenses) |
| block_id | str | Groups liabilities into blocks (for per-block biting scenario) |
| currency | str | Must match asset portfolio currency |

### 3.3 EPL Validation Rules

- Periods must be contiguous (1, 2, 3, ... with no gaps)
- Total EPL must be positive (there must be liabilities to back)
- Currency must match the asset portfolio currency
- If multiple blocks, each block is analyzed independently

---

## 4. Projection Engine Specifications

### 4.1 Core Projection Algorithm

For each scenario S (1 to 9):

```
INPUTS:
  portfolio: List[AssetModelPoint]
  epl: List[EPLCashFlow]
  scenario_curve: YieldCurve (from curves/ module)
  rules: RuleSet (from rules/v20XX/)
  C0: float (initial cash buffer - being solved for)

STATE:
  cash_balance[0] = C0
  active_assets = copy(portfolio)

FOR t = 1 to T_max:

  STEP 1: Collect asset cash flows
    asset_cf[t] = SUM over all active_assets of:
      - Coupon payments due in period t
      - Principal repayments (maturity, amortization, prepayment)
    Remove matured assets from active_assets

  STEP 2: Apply credit adjustment                    [@rule_ref: Para 28(22-26)]
    default_cost[t] = SUM over active_assets of:
      BMA_default_rate(rating, t) * remaining_par_value
    downgrade_cost[t] = SUM over active_assets of:
      BMA_downgrade_rate(rating, t) * remaining_par_value * phase_in_factor(rule_year)
    adjusted_cf[t] = asset_cf[t] - default_cost[t] - downgrade_cost[t]

  STEP 3: Calculate net cash flow
    net_cf[t] = adjusted_cf[t] + cash_balance[t-1] * scenario_rate(t) - epl[t]
    (cash_balance earns the scenario's short-term rate)

  STEP 4: Reinvest or liquidate                      [@rule_ref: Para 28(9)]
    IF net_cf[t] > 0:
      new_assets = reinvest(net_cf[t], scenario_curve, t, rules)
      active_assets += new_assets
      txn_cost = transaction_cost(net_cf[t], "BUY", rules)
      cash_balance[t] = 0

    IF net_cf[t] < 0:
      shortfall = abs(net_cf[t])
      proceeds, sold_assets = liquidate(shortfall, active_assets, scenario_curve, t, rules)
      active_assets -= sold_assets
      txn_cost = transaction_cost(proceeds, "SELL", rules)
      cash_balance[t] = proceeds - txn_cost - shortfall
      IF cash_balance[t] < 0: RECORD_DEFICIT

  STEP 5: Update cash balance
    cash_balance[t] = net_cf[t] - txn_cost  (simplified; actual logic above)

  STEP 6: Log audit trail
    Record: t, asset_cf, credit_adj, epl, net_cf, reinvest/liquidate actions,
            txn_cost, cash_balance, portfolio_mv, rule_refs

RESULT:
  required_C0 = solve_for(C0 such that min(cash_balance[t] for all t) = 0)
  Using scipy.optimize.brentq
```

### 4.2 Reinvestment Strategy

**Rule Reference:** Rules Schedule XXV, Paragraph 28(9)(a-b); Handbook E2 (CIO attestation)

```python
class ReinvestmentStrategy(ABC):
    @abstractmethod
    def reinvest(self, amount, curve, period, rules) -> List[AssetModelPoint]:
        """Return new assets purchased with the excess amount."""

class ProRataReinvestment(ReinvestmentStrategy):
    """Reinvest proportionally to current asset allocation.
    New bonds have coupon = scenario yield at their maturity."""

class CashOnlyReinvestment(ReinvestmentStrategy):
    """Hold all excess as cash earning the short-term rate.
    Simplest strategy; conservative for benchmark purposes."""
```

### 4.3 Disinvestment Waterfall

**Rule Reference:** Rules Schedule XXV, Paragraph 28(9)(c-d)

Liquidation order (most liquid to least liquid):

```
Priority 1: Cash and cash equivalents          (zero cost)
Priority 2: Government bonds (Tier 1)          (lowest spread)
Priority 3: Investment-grade corporate (Tier 1) (moderate spread)
Priority 4: Municipal bonds (Tier 1)           (moderate spread)
Priority 5: Approved structured (Tier 2)       (higher spread)
Priority 6: Other Tier 2 assets                (highest spread)
EXCLUDED:   Tier 3 (limited-basis) assets      (CANNOT be sold)
```

Sale price = `ql.BondFunctions.cleanPrice(bond, scenario_curve)` - transaction costs

If liquidation of all eligible assets cannot cover the shortfall, a deficit is recorded. This means the scenario requires a higher initial C0.

### 4.4 Solving for C0 (Root-Finding)

```python
from scipy.optimize import brentq

def find_C0(portfolio, epl, scenario_curve, rules):
    """Find the initial cash buffer C0 such that cash never goes negative."""

    def objective(C0):
        result = run_projection(portfolio, epl, scenario_curve, rules, C0)
        return min(result.cash_balance)  # Want this to be >= 0

    C0 = brentq(objective, a=0, b=sum(epl.liability_cf) * 2, xtol=0.01)
    return C0
```

---

## 5. Credit Cost Specifications

### 5.1 Default Costs

**Rule Reference:** Rules Schedule XXV, Paragraphs 28(22-24)

Default cost per asset per period:
```
default_cost = BMA_cumulative_default_rate(rating, year) * par_value
```

BMA publishes a cumulative default rate table by rating and year. The model uses these as a **floor** - companies may use higher rates but not lower.

### 5.2 Downgrade Costs

**Rule Reference:** Rules Schedule XXV, Paragraphs 28(25-26)

Downgrade cost represents the loss from credit migration (spread widening):
```
downgrade_cost = BMA_downgrade_rate(rating, year) * par_value * phase_in_factor
```

### 5.3 Phase-In (Transitional Arrangement)

For business in force as of December 31, 2023:

| Year | Phase-In Factor |
|---|---|
| 2024 | 20% |
| 2025 | 40% |
| 2026 | 60% |
| 2027 | 80% |
| 2028+ | 100% |

**Rule Reference:** Rules Schedule XXV (transitional provision)

### 5.4 Credit Cost Table Structure

```yaml
# assumptions/tables/bma_credit_floors.csv
rating,year_1,year_2,year_3,year_4,year_5,year_10,year_20,year_30
AAA,0.0001,0.0003,0.0006,0.0010,0.0015,0.0050,0.0150,0.0250
AA+,0.0002,0.0005,0.0010,0.0016,0.0024,0.0075,0.0200,0.0350
AA,0.0003,0.0008,0.0015,0.0024,0.0035,0.0100,0.0280,0.0450
...
```

*Note: Exact values from BMA published tables to be inserted when available in machine-readable form.*

---

## 6. Transaction Cost Specifications

### 6.1 Bid-Ask Spread Model

**Rule Reference:** Rules Schedule XXV, Section 6.2

```
transaction_cost = notional * bid_ask_spread(asset_type, liquidity_tier) / 2
```

### 6.2 Grade-In Rule

If current market spreads are tighter than long-term average:
```
effective_spread = current_spread + (lt_average - current_spread) * grading_factor(year)

grading_factor(year) = min(year / 10, 1.0)  # Linear grade over 10 years
```

### 6.3 Spread Table

| Asset Category | Current Spread (bps) | Long-Term Average (bps) |
|---|---|---|
| Government bonds | 2-5 | 5-10 |
| IG Corporate | 10-25 | 25-50 |
| Municipal | 10-20 | 20-40 |
| Structured (IG) | 25-50 | 50-100 |
| Below IG | 50-100 | 100-200 |

*Note: Exact values are configurable in `assumptions/tables/` and should reflect market conditions at valuation date.*

---

## 7. Stress Test Specifications

### 7.1 Combined Mass Lapse + Credit Spread Widening

**Rule Reference:** Handbook Section E5.6h(i)

**Mass Lapse:**
- Instantaneous lapse of the higher of:
  - 20% of in-force policies, OR
  - The BSCR lapse shock

**Credit Spread Widening:**
- Instantaneous widening applied to all assets:

| Rating | Spread Widening (bps) |
|---|---|
| AAA | +277 |
| AA | +350 |
| A | +450 |
| BBB | +600 |
| BB | +900 |
| B+ and below | +1200 |

**Test:** After applying both shocks simultaneously, recalculate BEL. The model must still demonstrate positive surplus.

### 7.2 One-Notch Downgrade

**Rule Reference:** Handbook Section E5.6h(ii)

- Apply a one-notch rating downgrade to **every** asset in the portfolio simultaneously
- Recalculate credit costs using the new (lower) ratings
- Recalculate BEL
- The model must demonstrate the impact is manageable

### 7.3 No Reinvestment Stress

**Rule Reference:** Handbook Section E5.6h(iii)

- Run projection where reinvestment into "Assets acceptable on a limited basis" (Tier 3) is prohibited
- All reinvestment must go into Tier 1 or approved Tier 2 assets
- Demonstrate that the portfolio remains adequate

---

## 8. LCR (Liquidity Coverage Ratio) Specifications

### 8.1 Formula

**Rule Reference:** Rules Paragraph 29(2)(iii)

```
LCR = Available Liquidity Sources / Potential Surrender Outflows >= 105%
```

### 8.2 Available Liquidity Sources

Classified by liquidity tier:

| Tier | Assets | Haircut |
|---|---|---|
| Level 1 | Cash, government bonds | 0-5% |
| Level 2A | IG corporate bonds, agency MBS | 10-15% |
| Level 2B | Below-IG bonds, other securities | 25-50% |

```
Available Liquidity = SUM(asset_mv * (1 - haircut)) for eligible liquid assets
```

### 8.3 Potential Surrender Outflows

Two scenarios:

**Fast-Moving Deterioration:**
- Immediate surrender of X% of policyholder values
- X = based on product type and surrender penalty structure

**Sustained Deterioration:**
- Gradual surrender over 12 months
- Lower rate but sustained pressure on liquidity

### 8.4 LCR Implementation

```python
def calculate_lcr(portfolio, rules, stress_scenario):
    available = sum(
        asset.market_value * (1 - rules.lcr_haircut(asset))
        for asset in portfolio
        if rules.is_lcr_eligible(asset)
    )
    outflows = rules.lcr_potential_surrender(stress_scenario)
    lcr = available / outflows
    passes = lcr >= 1.05
    return LCRResult(lcr=lcr, available=available, outflows=outflows, passes=passes)
```

---

## 9. Risk Margin Specifications

### 9.1 Formula

**Rule Reference:** Rules Subparagraph 36(4)

```
Risk Margin = CoC * SUM(t=0 to T) [ Modified_ECR(t) / (1 + r(t+1))^(t+1) ]
```

Where:
- **CoC** = 6% (Cost of Capital, prescribed by BMA)
- **Modified_ECR(t)** = projected Enhanced Capital Requirement at future time t
- **r(t)** = risk-free rate at time t (from base curve)

### 9.2 Modified ECR Projection

The ECR at future times is projected by scaling the current ECR proportionally to the remaining liability duration/exposure:
```
Modified_ECR(t) = ECR(0) * remaining_exposure(t) / total_exposure(0)
```

*Note: Exact methodology for projecting Modified ECR to be confirmed with BMA. This is a common area of interpretation difference.*

---

## 10. BEL Calculation Specification

### 10.1 Formula

```
BEL = MV_assets(T=0) + C0_biting
```

Where:
- **MV_assets(T=0)** = mark-to-market value of the starting portfolio under the base curve
- **C0_biting** = the initial cash buffer required under the biting (worst) scenario

### 10.2 Biting Scenario

```
biting_scenario = argmax(C0[s] for s in scenarios 1..9)
```

The scenario that requires the highest initial cash buffer is the "biting scenario."

### 10.3 Technical Provision

```
TP = BEL + Risk Margin
   = (MV_assets + C0_biting) + (CoC * discounted_future_ECR)
```

---

## 11. Data Input File Specifications

### 11.1 Asset Portfolio File (CSV/Excel)

**Required Columns:**

| Column | Type | Example | Required |
|---|---|---|---|
| asset_id | string | "BOND_001" | Yes |
| asset_type | enum | "GOVT_BOND" | Yes |
| issuer_name | string | "US Treasury" | Yes |
| par_value | float | 1000000 | Yes |
| book_value | float | 995000 | Yes |
| market_value | float | 1010000 | Yes |
| coupon_rate | float | 0.05 | Yes |
| coupon_frequency | int | 2 | Yes |
| issue_date | date | "2020-01-15" | Yes |
| maturity_date | date | "2030-01-15" | Yes |
| currency | string | "USD" | Yes |
| rating | enum | "AA+" | Yes |
| tier | int | 1 | Computed |
| is_callable | bool | false | No (default: false) |
| call_date | date | null | If callable |
| call_price | float | null | If callable |
| is_amortizing | bool | false | No (default: false) |
| prepayment_speed | float | null | MBS only |

### 11.2 EPL File (CSV/Excel)

| Column | Type | Example |
|---|---|---|
| period | int | 1 |
| liability_cf | float | 5000000 |
| block_id | string | "ANNUITY_BLOCK" |
| currency | string | "USD" |

### 11.3 Market Data File (CSV)

| Column | Type | Example |
|---|---|---|
| tenor_years | float | 0.25 |
| rate | float | 0.0425 |
| instrument_type | string | "SWAP" |
| date | date | "2024-12-31" |

---

## 12. Output Specifications

### 12.1 Excel Workbook Structure

| Tab | Contents |
|---|---|
| **Summary** | BEL, RM, TP, biting scenario, key ALM metrics |
| **Scenario Results** | C0 required per scenario, scenario comparison chart |
| **Cash Flow Detail** | Period-by-period CFs for biting scenario (expandable to all 9) |
| **Asset Portfolio** | Starting portfolio with MV, BV, rating, tier, duration |
| **Credit Costs** | Default and downgrade costs by period and rating |
| **Stress Tests** | Results of 3 mandatory stress tests |
| **LCR** | Available liquidity, potential surrender, LCR ratio |
| **Audit Trail** | Key rule references applied, assumption versions used |
| **Comparison** | Benchmark vs company (if submission provided) |

### 12.2 Numerical Precision

- All monetary values: 2 decimal places (dollars and cents)
- Rates: 6 decimal places (e.g., 0.042513)
- Basis points: 2 decimal places (e.g., 150.00 bps)
- Ratios (LCR): 4 decimal places (e.g., 1.0523)
- Internal calculations: 64-bit floating point (Python default)
