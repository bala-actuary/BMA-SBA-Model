# Algorithm Specification: Projection Engine

## Document Control

| Field | Value |
|---|---|
| **Spec ID** | ALGO-002 |
| **Target Module** | `projection/cashflow_engine.py` |
| **Supporting Modules** | `projection/credit_costs.py`, `projection/reinvestment.py`, `projection/disinvestment.py`, `engine/run_orchestrator.py` |
| **Status** | Implementation-Ready Draft |
| **Date** | March 2026 |
| **BMA Rules** | Schedule XXV Para 28(9)-(11); Handbook E4, E11 |
| **Cross-References** | [ALGO-001](01_Yield_Curve_Construction.md) (curves), [ALGO-002a](02a_Reinvestment_Strategy.md) (reinvestment), [ALGO-002b](02b_Disinvestment_Waterfall.md) (disinvestment), [ALGO-003](03_Credit_Costs_DD.md) (D&D), [ALGO-004](04_Spread_Cap_Enforcement.md) (spread cap) |

---

## 1. Overview

### 1.1 Purpose

This is the **heart of the BMA SBA Benchmark Model**. The projection engine performs a period-by-period cash flow projection that determines how much initial cash (C0) an insurer must hold alongside its asset portfolio so that all liability payments are met as they fall due, under each of the 9 BMA interest rate scenarios.

The SBA is a **cash flow adequacy test**, not a liability revaluation. The liability cash flows are fixed hurdles. The interest rate scenarios stress the *asset side* -- affecting reinvestment income, market values on forced sales, and the overall ability of the portfolio to generate sufficient cash. The projection engine is where this test is executed.

### 1.2 What This Module Does

1. Accepts a portfolio of assets, a stream of liability cash flows, a scenario curve, a rule set, and a trial C0 value.
2. Projects cash flows period-by-period from `t = 1` to `T_max`, applying coupons, maturities, D&D credit adjustments, liability outflows, reinvestment of surpluses, and liquidation of assets on shortfalls.
3. Returns the full projection trace (a DataFrame of period snapshots) and the minimum cash balance achieved across all periods.
4. Is called repeatedly by the C0 root-finder (`scipy.optimize.brentq`) to find the exact C0 that drives the minimum cash balance to zero.

### 1.3 Architectural Boundary

**This module is pure Python.** It does not import QuantLib. All yield curve data arrives as plain Python objects (discount factor arrays, rate callables) from the `curves/` package. This is a non-negotiable architectural constraint that ensures full auditability of the projection logic.

```
curves/                          projection/
  curve_builder.py  ---------->    cashflow_engine.py    (THIS MODULE)
  scenario_curves.py               credit_costs.py       (ALGO-003)
  bond_pricing.py                  reinvestment.py        (ALGO-002a)
                                   disinvestment.py       (ALGO-002b)
  [QuantLib boundary]              [Pure Python only]
```

### 1.4 Key Rule References

| Rule | Description | Used In |
|---|---|---|
| Para 28(9) | Cash flow matching at each time step | Section 3, Steps 5-8 |
| Para 28(9)(a-b) | Excess cash flows reinvested per strategy | Section 3, Step 7 |
| Para 28(9)(c-d) | Shortfalls met by selling assets at scenario yields | Section 3, Step 8 |
| Para 28(10)(a-c) | BEL = highest asset requirement across 9 scenarios | Section 5 |
| Subpara. 11 | Biting scenario = scenario requiring most assets | Section 5 |
| Para 28(22)-(26) | D&D credit cost deductions | Section 3, Step 3 |
| Para 28(15)-(16) | Tier 3 assets cannot be sold (10% cap) | Section 3, Step 8 |
| Handbook E4 | Projection methodology guidance | Sections 2-3 |
| Handbook E11 | Reinvestment/disinvestment rules | Section 3, Steps 7-8 |

---

## 2. Projection Architecture

### 2.1 High-Level Flow

```
                    +-------------------+
                    | C0 Root-Finder    |
                    | (brentq)          |
                    +--------+----------+
                             |
                   trial C0  |
                             v
+--------+    +--------------+---------------+    +----------+
| Assets |    |     run_projection()         |    | Scenario |
| (port) +--->|  Period-by-period loop       |<---+ Curve    |
+--------+    |  t = 1 .. T_max             |    +----------+
              |                              |
+--------+    |  Steps 1-10 per period       |    +----------+
| Liab.  +--->|  (see Section 3)             |<---+ RuleSet  |
| CFs    |    +--------------+---------------+    +----------+
+--------+                   |
                             v
                  +----------+----------+
                  | ProjectionResult    |
                  | - trace: DataFrame  |
                  | - min_cash: float   |
                  | - audit_log: list   |
                  +---------------------+
```

### 2.2 Inputs

| Input | Type | Description |
|---|---|---|
| `portfolio` | `List[AssetModelPoint]` | Asset positions at T=0: par, coupon rate, maturity, rating, tier, MV |
| `liability_cf` | `np.ndarray` | Liability cash flows by period: `liability_cf[t]` = outflow due at end of period `t` |
| `scenario_curve` | `ScenarioCurve` | Plain Python object providing `rate(t) -> float` for the reinvestment rate at period `t`, and `discount_factor(t) -> float` for asset pricing |
| `rules` | `RuleSet` | Versioned BMA rule parameters (D&D tables, phase-in factors, tier caps, transaction costs) |
| `C0` | `float` | Trial initial cash buffer (the variable being solved for) |
| `config` | `RunConfig` | Frozen run configuration (valuation date, rule year, tolerances) |

### 2.3 Outputs

```python
@dataclass(frozen=True)
class ProjectionResult:
    trace: pd.DataFrame          # One row per period, all intermediate values
    min_cash_balance: float      # min(cash_balance[t]) across all t -- target for root-finder
    final_cash_balance: float    # cash_balance[T_max]
    deficit_periods: List[int]   # Periods where cash went negative (should be empty if C0 correct)
    audit_log: List[AuditEntry]  # Rule references triggered during projection
```

The `trace` DataFrame has one row per period with these columns:

| Column | Description |
|---|---|
| `period` | Time step (1 to T_max) |
| `opening_cash` | Cash balance at start of period |
| `coupon_income` | Total coupon payments received |
| `principal_income` | Redemptions from maturing/amortizing assets |
| `gross_asset_cf` | `coupon_income + principal_income` |
| `default_cost` | D&D default deduction (ALGO-003) |
| `downgrade_cost` | D&D downgrade deduction (ALGO-003) |
| `net_asset_cf` | `gross_asset_cf - default_cost - downgrade_cost` |
| `interest_on_cash` | Interest earned on opening cash at scenario rate |
| `liability_cf` | Liability outflow for this period |
| `net_cf` | `net_asset_cf + interest_on_cash - liability_cf` |
| `reinvestment_amount` | Amount reinvested (if net_cf > 0) |
| `disinvestment_amount` | Amount liquidated (if net_cf < 0) |
| `transaction_cost` | Costs on reinvestment or disinvestment |
| `closing_cash` | Cash balance at end of period |
| `portfolio_mv` | Total mark-to-market of remaining assets |
| `num_active_assets` | Count of non-matured assets |
| `rule_refs` | BMA rule paragraphs invoked this period |

### 2.4 State

The projection maintains mutable state that evolves across periods:

```python
# Initialized once at t=0
cash_balance: float = C0
active_assets: List[AssetModelPoint] = deepcopy(portfolio)

# Updated each period
# (all intermediate values captured in the trace DataFrame)
```

**Immutability principle:** The input `portfolio` is deep-copied at the start. The original is never mutated. The `RunConfig` is frozen. Only the working copies (`cash_balance`, `active_assets`) change during projection.

---

## 3. Per-Period Algorithm

This is the complete 10-step sequence executed for each period `t = 1` to `T_max`. Each step includes the pseudocode, rule references, and cross-references to other algorithm specs.

### Step 1: Begin Period -- Remove Matured Assets, Reset Accumulators

**Purpose:** Transition to the new period. Remove any assets that matured at the end of the previous period. Reset all per-period accumulators.

**Rule reference:** N/A (housekeeping)

```python
@rule_ref("N/A - housekeeping")
def step_1_begin_period(t, active_assets, cash_balance):
    # Remove assets that matured at or before period t
    matured = [a for a in active_assets if a.maturity_period <= t]
    active_assets = [a for a in active_assets if a.maturity_period > t]

    # Reset per-period accumulators
    period_data = PeriodSnapshot(period=t, opening_cash=cash_balance)

    return active_assets, matured, period_data
```

**Note:** Matured assets' final principal was collected in the *previous* period's Step 2. Here we simply remove them from the active list.

### Step 2: Collect Asset Cash Flows (Coupons, Principal, Amortization)

**Purpose:** Compute all cash flows generated by assets during this period. This includes coupon payments from all active assets and principal repayments from assets maturing *at the end of this period*.

**Rule reference:** Para 28(9) -- "projected asset cash flows"

```python
@rule_ref("Para 28(9)")
def step_2_collect_asset_cf(t, active_assets):
    coupon_income = 0.0
    principal_income = 0.0

    for asset in active_assets:
        # Coupon: paid each period the asset is alive
        coupon_income += asset.par_value * asset.coupon_rate

        # Principal: returned at maturity
        if asset.maturity_period == t:
            principal_income += asset.par_value

        # Amortizing assets: scheduled principal in this period
        if asset.amortization_schedule is not None:
            principal_income += asset.amortization_schedule.get(t, 0.0)

    gross_asset_cf = coupon_income + principal_income
    return coupon_income, principal_income, gross_asset_cf
```

**For the illustrative example (1 bond, $4,500 par, 4.4% coupon, 5-yr maturity):**
- Periods 1-4: `coupon_income = 4500 * 0.044 = $198`, `principal_income = $0`, `gross_asset_cf = $198`
- Period 5: `coupon_income = $198`, `principal_income = $4,500`, `gross_asset_cf = $4,698`

### Step 3: Apply Default & Downgrade (D&D) Credit Adjustment

**Purpose:** Reduce asset cash flows to account for expected credit losses. This is detailed in ALGO-003.

**Rule reference:** Para 28(22)-(26)

```python
@rule_ref("Para 28(22)-(26)")
def step_3_apply_dd(t, active_assets, gross_asset_cf, rules):
    default_cost = 0.0
    downgrade_cost = 0.0

    for asset in active_assets:
        # Default cost: cumulative default rate by rating and year
        default_cost += (
            rules.default_rate(asset.rating, t) * asset.par_value
        )

        # Downgrade cost: with transitional phase-in
        downgrade_cost += (
            rules.downgrade_rate(asset.rating, t)
            * asset.par_value
            * rules.phase_in_factor(t)
        )

    net_asset_cf = gross_asset_cf - default_cost - downgrade_cost

    # FAIL LOUDLY: net_asset_cf should not be negative from D&D alone
    if net_asset_cf < 0:
        raise ProjectionError(
            f"Period {t}: D&D costs ({default_cost + downgrade_cost:.2f}) "
            f"exceed gross asset CF ({gross_asset_cf:.2f}). "
            f"Check credit tables or asset data."
        )

    return default_cost, downgrade_cost, net_asset_cf
```

**Cross-reference:** See [ALGO-003: Credit Costs D&D](03_Credit_Costs_DD.md) for the full D&D algorithm, BMA floor tables, and phase-in schedule.

**For the illustrative example:** D&D costs are zero (simplified example with a single high-quality bond).

### Step 4: Apply Transaction Costs on Maturing Assets

**Purpose:** Deduct any transaction costs associated with assets that matured and returned principal this period. In many implementations, maturing bonds have zero transaction costs (they redeem at par), but the framework must support it for callable bonds or structured assets.

**Rule reference:** Handbook E11 (transaction cost guidance)

```python
@rule_ref("Handbook E11")
def step_4_maturity_txn_costs(t, matured_assets, principal_income, rules):
    maturity_txn_cost = 0.0
    for asset in matured_assets:
        if asset.maturity_period == t:
            maturity_txn_cost += rules.transaction_cost(
                asset.asset_class, "MATURITY"
            )

    adjusted_principal = principal_income - maturity_txn_cost
    return maturity_txn_cost, adjusted_principal
```

**For the illustrative example:** Transaction costs are zero (bond redeems at par with no friction).

### Step 5: Subtract Liability Outflows

**Purpose:** The core of the cash flow adequacy test. Subtract the liability payment due this period from available cash flows.

**Rule reference:** Para 28(9) -- "at each time step, liability cash flows are compared to asset cash flows"

```python
@rule_ref("Para 28(9)")
def step_5_subtract_liabilities(t, liability_cf):
    # liability_cf[t] is the outflow due at end of period t
    # This is a fixed, non-negotiable obligation
    return liability_cf[t]
```

**For the illustrative example:** `liability_cf[t] = $1,000` for `t = 1..5`.

### Step 6: Calculate Net Cash Flow

**Purpose:** Combine all inflows and outflows to determine the net position for this period. Interest earned on the opening cash balance (at the scenario's reinvestment rate) is included here.

**Rule reference:** Para 28(9)(b) -- "net cash flows"

```python
@rule_ref("Para 28(9)(b)")
def step_6_net_cash_flow(
    opening_cash, net_asset_cf, liability_outflow,
    scenario_curve, t, maturity_txn_cost
):
    # Interest earned on cash held at start of period
    interest_on_cash = opening_cash * scenario_curve.rate(t)

    # Net cash flow for this period
    net_cf = (
        net_asset_cf               # Coupons + principal - D&D
        + interest_on_cash         # Interest earned on cash buffer
        - liability_outflow        # Liability payment due
        - maturity_txn_cost        # Transaction costs on maturities
    )

    return interest_on_cash, net_cf
```

**For the illustrative example (Base scenario, period 1):**
- `net_asset_cf = $198` (coupon only, no D&D)
- `interest_on_cash = $2,981 * 0.03 = $89.43 ~ $89`
- `liability_outflow = $1,000`
- `net_cf = $198 + $89 - $1,000 = -$713`

But wait -- the illustrative calculation folds the net outflow differently. Let us reconcile. In the illustrative example, the "Net Outflow" column is `$1,000 - $198 = $802`, and interest is computed on opening cash separately. The closing cash is `opening + interest - net_outflow`. Our formulation produces the same result:

`closing_cash = opening_cash + interest_on_cash + net_asset_cf - liability_outflow`
`= $2,981 + $89 + $198 - $1,000 = $2,268` (matches the illustrative table).

### Step 7: Handle Surplus -- Reinvestment

**Purpose:** If net cash flow is positive after all outflows, the excess is reinvested according to the defined reinvestment strategy.

**Rule reference:** Para 28(9)(a-b); Handbook E11

```python
@rule_ref("Para 28(9)(a-b)")
def step_7_reinvest(net_cf, opening_cash, interest_on_cash,
                    active_assets, scenario_curve, t, rules):
    # Total available cash after this period's flows
    available_cash = opening_cash + interest_on_cash + net_cf
    # (net_cf already includes net_asset_cf - liability_outflow from Step 6)

    reinvestment_amount = 0.0
    txn_cost = 0.0

    if available_cash > 0 and net_cf > 0:
        # Determine how much to reinvest vs. hold as cash
        # Strategy defined in ALGO-002a
        reinvestment_amount, new_assets, txn_cost = rules.reinvestment_strategy.reinvest(
            excess=net_cf,
            curve=scenario_curve,
            period=t,
            rules=rules
        )
        active_assets.extend(new_assets)

    closing_cash = available_cash - reinvestment_amount - txn_cost
    return reinvestment_amount, txn_cost, closing_cash, active_assets
```

**Cross-reference:** See [ALGO-002a: Reinvestment Strategy](02a_Reinvestment_Strategy.md) for the full reinvestment algorithm, including pro-rata vs. cash-only strategies.

**For the illustrative example:** Excess cash earns interest but is not reinvested into new bonds. The example uses a **cash-only reinvestment strategy** where all surplus remains as cash earning the scenario's risk-free rate. This is the simplest and most conservative approach for benchmark purposes.

### Step 8: Handle Shortfall -- Disinvestment Waterfall

**Purpose:** If the net position is negative (liabilities exceed available cash + asset income), assets must be sold to cover the shortfall. The disinvestment follows a strict waterfall from most liquid to least liquid.

**Rule reference:** Para 28(9)(c-d); Para 28(15)-(16) (Tier 3 restriction)

```python
@rule_ref("Para 28(9)(c-d)")
def step_8_disinvest(net_cf, opening_cash, interest_on_cash,
                     active_assets, scenario_curve, t, rules):
    available_cash = opening_cash + interest_on_cash + net_cf

    disinvestment_amount = 0.0
    txn_cost = 0.0

    if available_cash < 0:
        shortfall = abs(available_cash)

        # Liquidation waterfall (ALGO-002b):
        # Priority 1: Cash equivalents (zero cost)
        # Priority 2: Government bonds, Tier 1
        # Priority 3: IG corporate, Tier 1
        # Priority 4: Municipal bonds, Tier 1
        # Priority 5: Approved structured, Tier 2
        # Priority 6: Other Tier 2
        # EXCLUDED: Tier 3 assets (CANNOT be sold) -- Para 28(15)-(16)
        proceeds, sold_assets, txn_cost = rules.disinvestment_strategy.liquidate(
            shortfall=shortfall,
            assets=active_assets,
            curve=scenario_curve,
            period=t,
            rules=rules
        )

        for sold in sold_assets:
            active_assets.remove(sold)

        available_cash = proceeds - txn_cost - shortfall + available_cash
        disinvestment_amount = proceeds

        if available_cash < 0:
            # Deficit: cannot cover shortfall even after selling all eligible assets
            # This signals that C0 is too low -- the root-finder will increase it
            pass  # Deficit recorded in Step 10

    closing_cash = max(available_cash, 0.0) if available_cash >= 0 else available_cash
    return disinvestment_amount, txn_cost, closing_cash, active_assets
```

**Cross-reference:** See [ALGO-002b: Disinvestment Waterfall](02b_Disinvestment_Waterfall.md) for the full liquidation algorithm, sale pricing, and Tier 3 exclusion logic.

**For the illustrative example:** No disinvestment occurs because the example uses a simplified single-bond portfolio where the shortfall is covered by the cash buffer (C0). The bond is held to maturity.

### Step 9: Apply Scenario Interest Rate to Remaining Cash Balance

**Purpose:** In cases where Steps 7/8 have already incorporated interest (as in our Step 6 formulation), this step serves as a validation checkpoint. In alternative formulations where interest is applied at period-end, this is where it would be computed.

**Rule reference:** Para 28(9) -- "prevailing market yields"

```python
@rule_ref("Para 28(9)")
def step_9_validate_cash(closing_cash, t):
    # NaN check -- fail loudly
    if math.isnan(closing_cash) or math.isinf(closing_cash):
        raise ProjectionError(
            f"Period {t}: cash balance is {closing_cash}. "
            f"Numerical instability detected."
        )

    return closing_cash
```

**Implementation note:** In our formulation, interest on the opening cash balance is computed in Step 6 (beginning-of-period convention). This is consistent with the illustrative example where interest is earned on the opening balance before outflows. Step 9 is retained as a validation checkpoint and to support alternative timing conventions (end-of-period interest) if needed.

### Step 10: Record Audit Snapshot

**Purpose:** Write the complete period snapshot to the trace DataFrame. Every value computed in Steps 1-9 is captured for inspectability and audit.

**Rule reference:** @rule_ref decorator captures all rule references triggered during the period.

```python
@rule_ref("Audit - all rules")
def step_10_record(period_data, trace_rows, audit_log):
    # period_data is a dict/namedtuple with all values from Steps 1-9
    trace_rows.append({
        'period':               period_data.period,
        'opening_cash':         period_data.opening_cash,
        'coupon_income':        period_data.coupon_income,
        'principal_income':     period_data.principal_income,
        'gross_asset_cf':       period_data.gross_asset_cf,
        'default_cost':         period_data.default_cost,
        'downgrade_cost':       period_data.downgrade_cost,
        'net_asset_cf':         period_data.net_asset_cf,
        'interest_on_cash':     period_data.interest_on_cash,
        'liability_cf':         period_data.liability_cf,
        'net_cf':               period_data.net_cf,
        'reinvestment_amount':  period_data.reinvestment_amount,
        'disinvestment_amount': period_data.disinvestment_amount,
        'transaction_cost':     period_data.transaction_cost,
        'closing_cash':         period_data.closing_cash,
        'portfolio_mv':         period_data.portfolio_mv,
        'num_active_assets':    period_data.num_active_assets,
        'rule_refs':            period_data.rule_refs,
    })

    # Audit log: append rule references triggered this period
    for ref in period_data.rule_refs:
        audit_log.append(AuditEntry(period=period_data.period, rule=ref))

    return trace_rows, audit_log
```

### 3.1 Complete Projection Loop

Assembling all 10 steps into the main projection function:

```python
def run_projection(
    portfolio: List[AssetModelPoint],
    liability_cf: np.ndarray,
    scenario_curve: ScenarioCurve,
    rules: RuleSet,
    C0: float,
    config: RunConfig,
) -> ProjectionResult:
    """
    Run a single period-by-period cash flow projection for one scenario
    with a given trial C0.

    Returns ProjectionResult with full trace and min cash balance.
    """
    # Initialize state
    active_assets = deepcopy(portfolio)
    cash_balance = C0
    trace_rows = []
    audit_log = []
    T_max = len(liability_cf) - 1  # liability_cf[0] is unused (T=0)

    for t in range(1, T_max + 1):
        # Step 1: Begin period
        active_assets, matured, period_data = step_1_begin_period(
            t, active_assets, cash_balance
        )

        # Step 2: Collect asset cash flows
        coupon_income, principal_income, gross_asset_cf = step_2_collect_asset_cf(
            t, active_assets
        )

        # Step 3: Apply D&D credit adjustment
        default_cost, downgrade_cost, net_asset_cf = step_3_apply_dd(
            t, active_assets, gross_asset_cf, rules
        )

        # Step 4: Transaction costs on maturities
        maturity_txn_cost, adjusted_principal = step_4_maturity_txn_costs(
            t, matured, principal_income, rules
        )
        net_asset_cf = net_asset_cf - maturity_txn_cost

        # Step 5: Get liability outflow
        liability_outflow = step_5_subtract_liabilities(t, liability_cf)

        # Step 6: Calculate net cash flow
        interest_on_cash, net_cf = step_6_net_cash_flow(
            cash_balance, net_asset_cf, liability_outflow,
            scenario_curve, t, maturity_txn_cost=0  # already deducted
        )

        # Steps 7 & 8: Reinvest or disinvest
        total_available = cash_balance + interest_on_cash + net_asset_cf - liability_outflow

        if total_available >= 0:
            # Step 7: Surplus -- reinvest
            reinvestment_amount, txn_cost, closing_cash, active_assets = step_7_reinvest(
                net_cf, cash_balance, interest_on_cash,
                active_assets, scenario_curve, t, rules
            )
            disinvestment_amount = 0.0
        else:
            # Step 8: Shortfall -- disinvest
            disinvestment_amount, txn_cost, closing_cash, active_assets = step_8_disinvest(
                net_cf, cash_balance, interest_on_cash,
                active_assets, scenario_curve, t, rules
            )
            reinvestment_amount = 0.0

        # Step 9: Validate
        closing_cash = step_9_validate_cash(closing_cash, t)

        # Step 10: Record
        period_data.update(
            coupon_income=coupon_income,
            principal_income=principal_income,
            gross_asset_cf=gross_asset_cf,
            default_cost=default_cost,
            downgrade_cost=downgrade_cost,
            net_asset_cf=net_asset_cf,
            interest_on_cash=interest_on_cash,
            liability_cf=liability_outflow,
            net_cf=net_cf,
            reinvestment_amount=reinvestment_amount,
            disinvestment_amount=disinvestment_amount,
            transaction_cost=txn_cost,
            closing_cash=closing_cash,
            portfolio_mv=sum(a.market_value(scenario_curve, t) for a in active_assets),
            num_active_assets=len(active_assets),
        )
        trace_rows, audit_log = step_10_record(period_data, trace_rows, audit_log)

        # Advance state
        cash_balance = closing_cash

    # Build result
    trace_df = pd.DataFrame(trace_rows)
    cash_balances = trace_df['closing_cash'].values

    return ProjectionResult(
        trace=trace_df,
        min_cash_balance=float(np.min(cash_balances)),
        final_cash_balance=float(cash_balances[-1]),
        deficit_periods=[int(t) for t in trace_df[trace_df['closing_cash'] < 0]['period']],
        audit_log=audit_log,
    )
```

---

## 4. C0 Root-Finding

### 4.1 Objective Function

The C0 root-finder wraps `run_projection` and searches for the C0 value that makes the minimum cash balance across all periods equal to zero. This is a classic root-finding problem.

**Rule reference:** Para 28(10)(a-c) -- "the amount of initial assets required"

```python
from scipy.optimize import brentq

@rule_ref("Para 28(10)(a-c)")
def find_C0(
    portfolio: List[AssetModelPoint],
    liability_cf: np.ndarray,
    scenario_curve: ScenarioCurve,
    rules: RuleSet,
    config: RunConfig,
) -> Tuple[float, ProjectionResult]:
    """
    Find the minimum initial cash buffer C0 such that cash never
    goes negative during the projection.

    Returns (C0, ProjectionResult at the solved C0).
    """

    def objective(C0: float) -> float:
        result = run_projection(portfolio, liability_cf, scenario_curve, rules, C0, config)
        return result.min_cash_balance  # Want this == 0

    # Bracket: C0 must be >= 0. Upper bound is conservative:
    # worst case, we need cash to cover all liabilities with no asset income.
    upper_bound = float(np.sum(liability_cf)) * 2.0

    # Validate bracket: objective(0) should be negative (not enough cash),
    # objective(upper_bound) should be positive (more than enough).
    obj_at_zero = objective(0.0)
    obj_at_upper = objective(upper_bound)

    if obj_at_zero >= 0:
        # Assets alone cover all liabilities -- C0 = 0
        result = run_projection(portfolio, liability_cf, scenario_curve, rules, 0.0, config)
        return 0.0, result

    if obj_at_upper < 0:
        raise ProjectionError(
            f"Cannot find valid C0: even with C0 = {upper_bound:.2f}, "
            f"min cash = {obj_at_upper:.2f}. Check inputs."
        )

    C0_solved = brentq(
        objective,
        a=0.0,
        b=upper_bound,
        xtol=config.c0_tolerance,    # Default: 0.01
        rtol=1e-12,
        maxiter=100,
    )

    # Final run at solved C0 to get the full trace
    final_result = run_projection(
        portfolio, liability_cf, scenario_curve, rules, C0_solved, config
    )

    return C0_solved, final_result
```

### 4.2 Brentq Behavior

| Parameter | Value | Rationale |
|---|---|---|
| `a` (lower bracket) | `0.0` | C0 cannot be negative |
| `b` (upper bracket) | `2 * sum(liability_cf)` | Conservative: assumes zero asset income |
| `xtol` | `0.01` (configurable) | $0.01 precision on C0 -- sufficient for regulatory purposes |
| `rtol` | `1e-12` | Tight relative tolerance for numerical stability |
| `maxiter` | `100` | More than sufficient; brentq converges in ~50 iterations typically |

### 4.3 Monotonicity Guarantee

The objective function `f(C0) = min(cash_balance[t])` is **strictly monotonically increasing** in C0. Increasing C0 adds cash at T=0, which earns interest and provides a larger buffer, so the minimum cash balance can only increase. This guarantees:

1. **Uniqueness:** There is exactly one root (if one exists).
2. **Brentq convergence:** Brentq is guaranteed to converge for continuous, monotonic functions with a valid bracket.
3. **No local minima traps:** Unlike Newton's method, brentq will not oscillate or diverge.

---

## 5. BEL Assembly

### 5.1 Per-Scenario C0

After solving C0 for each of the 9 scenarios, the results are collected:

```python
@rule_ref("Para 28(10)(a-c)")
def compute_bel(
    portfolio: List[AssetModelPoint],
    liability_cf: np.ndarray,
    scenario_curves: Dict[str, ScenarioCurve],  # 9 scenario curves
    rules: RuleSet,
    config: RunConfig,
) -> BELResult:
    """
    Run C0 solve for all 9 scenarios and determine BEL.
    """
    scenario_results = {}
    mv_assets_t0 = sum(a.market_value_t0 for a in portfolio)

    for scenario_name, curve in scenario_curves.items():
        C0, projection = find_C0(portfolio, liability_cf, curve, rules, config)
        scenario_results[scenario_name] = ScenarioResult(
            scenario=scenario_name,
            C0=C0,
            total_initial_assets=mv_assets_t0 + C0,
            projection=projection,
        )

    # Biting scenario: the one requiring the highest C0
    biting = max(scenario_results.values(), key=lambda r: r.C0)

    # BEL = MV_assets(T=0) + C0_biting
    bel = mv_assets_t0 + biting.C0

    return BELResult(
        bel=bel,
        mv_assets_t0=mv_assets_t0,
        biting_scenario=biting.scenario,
        biting_C0=biting.C0,
        all_scenarios=scenario_results,
    )
```

### 5.2 BEL Formula

```
BEL = MV_assets(T=0) + C0_biting

Where:
  MV_assets(T=0) = Market value of all assets at valuation date
  C0_biting      = max over all 9 scenarios of C0_scenario
```

### 5.3 Spread Cap Check

After BEL is determined, the implied SBA spread must be checked against the 35bps cap. See [ALGO-004: Spread Cap Enforcement](04_Spread_Cap_Enforcement.md). If the cap binds, BEL is recalculated at the capped spread. The biting scenario is re-evaluated based on **capped** BEL values.

---

## 6. Worked Example

This section reproduces the golden test values from the BMA Illustrative Calculation (BMA_doc/BMA_SBA_Illustrative_Calculation_Comprehensive.md). Every number must match exactly.

### 6.1 Setup

**Asset:**
- 1 corporate bond: par = $4,500, coupon = 4.4%, yield = 4.5%, maturity = 5 years
- Market value at T=0: $4,480 (PV of cash flows at 4.5%)

**Liability:**
- 5-year annuity: $1,000 at end of each year

**Annual shortfall:** $1,000 - $198 = $802 (for years 1-4 while the bond is alive but only paying coupons)

**Year 5 surplus:** Bond matures returning $4,500 + $198 coupon = $4,698, minus $1,000 liability = $3,698 surplus

### 6.2 Base Scenario (Reinvestment Rate = 3.0%)

**C0 solve target:** Find C0 such that minimum closing cash = 0.

The C0 is the present value of the $802 annual shortfall for years 1-4, discounted at 3.0%:

```
C0_base = 802/(1.03)^1 + 802/(1.03)^2 + 802/(1.03)^3 + 802/(1.03)^4
        = 778.64 + 755.96 + 733.95 + 712.57
        = $2,981.12
        ~ $2,981
```

**Period-by-period trace (matching illustrative Table A):**

| Period | Opening Cash | Interest (3.0%) | Coupon | Principal | Liability | Net Outflow | Closing Cash |
|--------|-------------|-----------------|--------|-----------|-----------|-------------|-------------|
| 1 | $2,981 | +$89 | +$198 | $0 | -$1,000 | -$802 | $2,268 |
| 2 | $2,268 | +$68 | +$198 | $0 | -$1,000 | -$802 | $1,534 |
| 3 | $1,534 | +$46 | +$198 | $0 | -$1,000 | -$802 | $778 |
| 4 | $778 | +$23 | +$198 | $0 | -$1,000 | -$802 | -$1 |
| 5 | $0 | +$0 | +$198 | +$4,500 | -$1,000 | +$3,698 | $3,698 |

**Verification of each period:**

- Period 1: `$2,981 + $89 + $198 - $1,000 = $2,268`
- Period 2: `$2,268 + $68 + $198 - $1,000 = $1,534`
- Period 3: `$1,534 + $46 + $198 - $1,000 = $778`
- Period 4: `$778 + $23 + $198 - $1,000 = -$1` (rounding; C0 = $2,981 is rounded from $2,981.12)
- Period 5: `$0 + $0 + $198 + $4,500 - $1,000 = $3,698` (surplus after bond matures)

**Result:** `C0_base = $2,981`

### 6.3 Rates Down Scenario (Reinvestment Rate = 1.5%)

Lower reinvestment rate means the cash buffer earns less interest, requiring a larger starting amount.

```
C0_down = 802/(1.015)^1 + 802/(1.015)^2 + 802/(1.015)^3 + 802/(1.015)^4
        = 790.15 + 778.47 + 766.97 + 755.63
        = $3,091.22
        ~ $3,091
```

**Period-by-period trace (matching illustrative Table B):**

| Period | Opening Cash | Interest (1.5%) | Coupon | Principal | Liability | Net Outflow | Closing Cash |
|--------|-------------|-----------------|--------|-----------|-----------|-------------|-------------|
| 1 | $3,091 | +$46 | +$198 | $0 | -$1,000 | -$802 | $2,335 |
| 2 | $2,335 | +$35 | +$198 | $0 | -$1,000 | -$802 | $1,568 |
| 3 | $1,568 | +$24 | +$198 | $0 | -$1,000 | -$802 | $790 |
| 4 | $790 | +$12 | +$198 | $0 | -$1,000 | -$802 | $0 |
| 5 | $0 | +$0 | +$198 | +$4,500 | -$1,000 | +$3,698 | $3,698 |

**Verification of each period:**

- Period 1: `$3,091 + $46 + $198 - $1,000 = $2,335`
- Period 2: `$2,335 + $35 + $198 - $1,000 = $1,568`
- Period 3: `$1,568 + $24 + $198 - $1,000 = $790`
- Period 4: `$790 + $12 + $198 - $1,000 = $0`
- Period 5: `$0 + $0 + $198 + $4,500 - $1,000 = $3,698`

**Result:** `C0_down = $3,091`

### 6.4 Scenario Summary and BEL

| Scenario | Reinvestment Rate | C0 | Bond MV (T=0) | Total Initial Assets |
|---|---|---|---|---|
| Base (A) | 3.0% flat | $2,981 | $4,480 | $7,461 |
| Rates Down (B) | 1.5% flat | $3,091 | $4,480 | $7,571 |
| Rates Up (C) | 5.0% flat | $2,844 | $4,480 | $7,324 |
| Steepener (D) | 3.5% -> 5.0% | $2,885 | $4,480 | $7,365 |
| Flattener (E) | 2.0% -> 1.5% | $3,076 | $4,480 | $7,556 |

**Biting scenario:** Rates Down (B) -- requires the highest C0 of $3,091.

**BEL Calculation:**

```
BEL = MV_assets(T=0) + C0_biting
    = $4,480 + $3,091
    = $7,571
```

### 6.5 Why Rates Down is Biting

The Rates Down scenario is the most demanding because:

1. The cash buffer earns only 1.5% per year instead of 3.0% (base) or 5.0% (rates up).
2. This means less interest income is available to offset the $802 annual shortfall.
3. Therefore, a larger initial buffer is needed to cover the same fixed obligations.

This demonstrates the SBA's sensitivity to **reinvestment risk** -- a key insight for portfolios with significant asset-liability cash flow mismatches.

---

## 7. Multi-Asset Extension

### 7.1 Generalizing from 1 Bond to N Assets

The worked example uses a single bond for clarity. The algorithm generalizes to N assets with these changes:

**Step 2 (Asset Cash Flows):** Sum across all active assets:

```python
for asset in active_assets:
    coupon_income += asset.par_value * asset.coupon_rate
    if asset.maturity_period == t:
        principal_income += asset.par_value
```

Different assets may have different coupon frequencies (semi-annual, quarterly), different maturity dates, and amortization schedules. The engine handles all of these uniformly.

**Step 3 (D&D):** Each asset has its own rating, and the D&D deduction is computed per asset based on its rating and remaining term:

```python
for asset in active_assets:
    default_cost += rules.default_rate(asset.rating, t) * asset.par_value
    downgrade_cost += rules.downgrade_rate(asset.rating, t) * asset.par_value * phase_in
```

**Step 8 (Disinvestment):** With multiple assets, the waterfall becomes critical. Assets are sold in priority order (Tier 1 government bonds first, Tier 3 excluded entirely). The sale price of each asset depends on the scenario curve at that period -- a rates-up scenario means selling bonds at a loss (lower prices), while a rates-down scenario means selling at a gain.

### 7.2 Asset Interactions

Multiple assets introduce interactions that do not exist in the single-bond case:

| Interaction | Description |
|---|---|
| **Maturity ladder** | Different maturity dates create varying cash flow patterns across periods |
| **Rating diversity** | D&D costs vary by asset; a downgrade of one asset does not affect others |
| **Tier mixing** | Tier 3 assets provide cash flows but cannot be sold; their proportion affects disinvestment capacity |
| **Reinvestment assets** | Newly purchased assets (from Step 7) enter the active portfolio and generate future cash flows |
| **Duration matching** | Well-matched portfolios have smaller net cash flows each period, reducing C0 |

### 7.3 Computational Complexity

| Component | Single Asset | N Assets |
|---|---|---|
| Step 2 (Cash flows) | O(1) | O(N) per period |
| Step 3 (D&D) | O(1) | O(N) per period |
| Step 8 (Disinvestment) | N/A | O(N log N) per period (sort by priority) |
| Total per projection | O(T) | O(T * N) |
| C0 solve (brentq) | ~50 iterations | ~50 iterations (same) |
| Total per scenario | O(50 * T) | O(50 * T * N) |

For typical portfolios (N ~ 500 assets, T ~ 50 years), a single scenario C0 solve takes roughly 50 * 50 * 500 = 1.25M operations -- well within the performance budget for a desktop application.

---

## 8. Edge Cases

### 8.1 Negative Cash Balance

**Condition:** `closing_cash < 0` at some period `t`.

**Handling:** This is *expected* during the C0 root-finding process. When brentq is searching, it will try C0 values that are too low, resulting in negative cash. The projection must not crash -- it must return the negative minimum so that brentq can adjust upward.

```python
# Do NOT raise an exception for negative cash during root-finding.
# Instead, record it and return to brentq.
if closing_cash < 0:
    deficit_periods.append(t)
    # Continue projection -- do not abort
```

**Post-solve validation:** After C0 is found, `deficit_periods` should be empty. If not, something is wrong with the solve.

### 8.2 NaN / Inf Detection

**Condition:** Any intermediate value becomes NaN or Inf (e.g., division by zero, overflow).

**Handling:** Fail loudly. Raise `ProjectionError` immediately with full context.

```python
def _check_finite(value, name, period):
    if math.isnan(value) or math.isinf(value):
        raise ProjectionError(
            f"Period {period}: {name} = {value}. Numerical instability detected."
        )
```

This check is applied to: `net_asset_cf`, `interest_on_cash`, `net_cf`, `closing_cash`, `portfolio_mv`.

### 8.3 Asset Explosion (Reinvestment Creates Too Many Assets)

**Condition:** Repeated reinvestment over many periods creates thousands of small synthetic assets, degrading performance.

**Handling:** Implement asset consolidation. After reinvestment, merge synthetic assets with identical characteristics (same rating, coupon rate, maturity) into a single position:

```python
def consolidate_assets(active_assets, threshold=1000):
    if len(active_assets) > threshold:
        # Group by (rating, coupon_rate, maturity_period, asset_class)
        # Merge groups into single positions with summed par values
        active_assets = merge_identical_assets(active_assets)
    return active_assets
```

### 8.4 Zero Liability Periods

**Condition:** `liability_cf[t] == 0` for some period (e.g., a gap in the liability schedule).

**Handling:** The algorithm handles this naturally. With zero outflow, the net cash flow is positive (assuming assets generate income), triggering Step 7 (reinvestment). No special case needed.

### 8.5 All Assets Matured Before Liabilities End

**Condition:** All assets have matured but liability outflows remain.

**Handling:** The projection continues with `active_assets = []`. Asset cash flows are zero. The cash buffer must cover all remaining liabilities through interest accumulation alone. If the buffer is insufficient, the C0 root-finder will increase C0.

### 8.6 Brentq Bracket Failure

**Condition:** `objective(upper_bound)` is still negative, meaning even `C0 = 2 * sum(liabilities)` is not enough.

**Handling:** This indicates a pathological input (e.g., liability cash flows that grow exponentially, or D&D costs that consume all asset income). Raise a descriptive error:

```python
if obj_at_upper < 0:
    raise ProjectionError(
        f"Cannot bracket C0: min_cash at C0={upper_bound:.2f} is {obj_at_upper:.2f}. "
        f"Total liabilities: {np.sum(liability_cf):.2f}. "
        f"Check D&D assumptions and liability schedule."
    )
```

### 8.7 C0 = 0 (Assets Sufficient Without Cash)

**Condition:** `objective(0.0) >= 0` -- the asset portfolio alone generates enough cash to cover all liabilities.

**Handling:** Return `C0 = 0.0` immediately without invoking brentq. This is a valid result (the portfolio is perfectly or over-matched).

---

## 9. Implementation Notes

### 9.1 Pure Python Boundary

The projection engine (`projection/`) **must not import QuantLib**. All curve data arrives as plain Python objects:

| From `curves/` | Used In `projection/` As |
|---|---|
| `ScenarioCurve.rate(t)` | `float` -- reinvestment rate at period `t` |
| `ScenarioCurve.discount_factor(t)` | `float` -- for present value calculations |
| `BondPricer.clean_price(asset, curve, t)` | `float` -- sale price for disinvestment |

The `curves/` package pre-computes these values and exposes them through simple callables. The projection engine never needs to know that QuantLib is involved upstream.

### 9.2 DataFrame Intermediates

Every period's snapshot is stored as a row in the trace DataFrame. This enables:

- **Debugging:** Inspect any period's values to trace calculation errors.
- **Audit:** Regulators can review the complete projection path.
- **Visualization:** Plot cash balance, asset MV, or any column over time.
- **Testing:** Compare trace DataFrames against golden test fixtures cell by cell.

```python
# Example: inspect the trace after a projection
result = run_projection(portfolio, liability_cf, curve, rules, C0, config)
print(result.trace[['period', 'opening_cash', 'interest_on_cash', 'net_cf', 'closing_cash']])
```

### 9.3 Audit Trail

The `@rule_ref` decorator tags every calculation function with its BMA rule paragraph. The audit log accumulates all rule references triggered during the projection, enabling:

```python
# Example audit log entry
AuditEntry(
    period=3,
    rule="Para 28(22)-(26)",
    function="step_3_apply_dd",
    inputs={"rating": "A", "par_value": 4500, "t": 3},
    output={"default_cost": 2.25, "downgrade_cost": 1.80},
)
```

This supports the BMA's requirement for full traceability from output values back to regulatory rule paragraphs.

### 9.4 Performance Considerations

| Concern | Mitigation |
|---|---|
| **Brentq iterations** | ~50 iterations per scenario. Each iteration runs the full projection. For 9 scenarios: ~450 projection runs total. |
| **Projection length** | T_max up to 100 years for long-duration liabilities. Each period is O(N) for N assets. |
| **Memory** | The trace DataFrame for a 100-period projection with 20 columns is ~2,000 floats -- negligible. |
| **Asset consolidation** | Prevents reinvestment from creating unbounded asset lists. Threshold: consolidate when N > 1,000. |
| **Parallelism** | The 9 scenarios are independent and can be run in parallel (`concurrent.futures.ProcessPoolExecutor`). This provides a near-linear speedup on multi-core machines. |
| **Caching** | Discount factors and rates from `ScenarioCurve` can be pre-computed into arrays before the projection loop, avoiding repeated function call overhead. |

### 9.5 Testing Strategy

| Test Type | Description |
|---|---|
| **Golden test** | Reproduce the illustrative calculation exactly: C0_base = $2,981, C0_down = $3,091, BEL = $7,571. This is the primary integration test. |
| **Unit tests** | Test each of the 10 steps in isolation with known inputs and expected outputs. |
| **Property tests** | Use Hypothesis to verify: (a) C0 is monotonically increasing as reinvestment rate decreases, (b) C0 >= 0 for all valid inputs, (c) adding an asset with positive cash flows never increases C0. |
| **Edge case tests** | Zero liabilities, zero assets, single-period projection, all-matured-assets, Tier-3-only portfolio. |
| **Regression tests** | Store trace DataFrames as Parquet fixtures; compare cell-by-cell on every test run. |

### 9.6 Error Handling Philosophy

The projection engine follows the project-wide "fail loudly" principle:

1. **Invalid inputs:** Pydantic validates all inputs before the projection starts. Missing fields, wrong types, or out-of-range values halt the run immediately.
2. **Numerical issues:** NaN/Inf checks at every step. No silent degradation.
3. **Logic errors:** Assertions on invariants (e.g., `sum(sold_proceeds) >= shortfall` after disinvestment, `closing_cash == opening + interest + net_asset_cf - liability - txn_cost`).
4. **Audit trail:** Every error includes the period, step, and all relevant values for diagnosis.

---

## Appendix A: Glossary

| Term | Definition |
|---|---|
| **C0** | Initial cash buffer held alongside the asset portfolio at T=0 |
| **Biting scenario** | The scenario (of 9) requiring the highest C0 |
| **BEL** | Best Estimate Liability = MV_assets(T=0) + C0_biting |
| **Net cash flow** | Asset income + interest on cash - liability outflow - transaction costs |
| **Disinvestment waterfall** | Priority order for selling assets to cover shortfalls |
| **D&D** | Default and Downgrade credit cost deductions |
| **Trace DataFrame** | Period-by-period record of all projection values |
| **Run projection** | Single execution of the projection loop for one scenario and one C0 |
| **T_max** | Final projection period (typically = last liability cash flow period) |

## Appendix B: Relationship to Other Specs

```
ALGO-001: Yield Curve Construction
    |
    | Provides: ScenarioCurve objects (rate, discount_factor callables)
    v
ALGO-002: Projection Engine  <---- THIS DOCUMENT
    |
    |--- calls ---> ALGO-002a: Reinvestment Strategy (Step 7)
    |--- calls ---> ALGO-002b: Disinvestment Waterfall (Step 8)
    |--- calls ---> ALGO-003: Credit Costs D&D (Step 3)
    |
    | Produces: C0 per scenario, ProjectionResult
    v
ALGO-004: Spread Cap Enforcement
    |
    | Checks: implied spread vs. 35bps cap
    | Produces: final BEL (possibly capped)
    v
Engine / Output
```
