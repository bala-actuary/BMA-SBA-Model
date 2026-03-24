# Algorithm Specification 05: Lapse Cost (LapC) and Liquidity Coverage Ratio (LCR)

**Target Modules:** `calculations/lapse_cost.py`, `calculations/lcr.py`
**Status:** Implementation-Ready
**Version:** 1.0
**Date:** 2026-03-24

---

## 1. Overview

### 1.1 Purpose

This specification defines two related but distinct calculations required for SBA eligibility and ongoing compliance:

1. **Lapse Cost (LapC)** -- a capital-like adjustment held within the BEL to reflect residual lapse risk for portfolios containing policyholder options (surrenders, partial withdrawals, guaranteed annuity options). LapC ensures that even well-matched portfolios cannot ignore the possibility that lapses deviate from expectation.

2. **Liquidity Coverage Ratio (LCR)** -- a ratio test ensuring the insurer maintains sufficient liquid assets to cover potential surrender outflows under stress. The 105% minimum threshold acts as a hard gate: failure means the block is not SBA-eligible.

Both calculations sit downstream of the projection engine. LapC feeds into the BEL (increasing it); LCR is a pass/fail eligibility test. Together with the three application stress tests (Section 4), they form the lapse and liquidity governance layer of the SBA framework.

### 1.2 Target Modules

| Module | Responsibility |
|--------|---------------|
| `calculations/lapse_cost.py` | Compute LapC from BSCR inputs; run lapse up/down stress tests |
| `calculations/lcr.py` | Compute LCR ratio from portfolio and surrender assumptions; evaluate pass/fail |

Both modules are **pure Python** (no QuantLib). They consume outputs from:
- `projection/` -- projected cash flows and BEL per scenario
- `model_points/` -- asset portfolio with market values, ratings, tiers
- `assumptions/` -- lapse rate assumptions, BSCR parameters, LCR haircut tables

### 1.3 Regulatory References

| Reference | Description |
|-----------|-------------|
| Rules Sched XXV Para 29(1)(i)D, p. 175 | Lapse Cost formula definition |
| Rules Sched XXV Para 29(2) | Lapse risk eligibility criteria |
| Rules Sched XXV Para 29(2)(iii), p. 175 | LCR requirement (105% minimum) |
| Rules Sched XXV Para 31, pp. 176-177 | Liquidity risk management program |
| Handbook D37, pp. 310-313 | Lapse Cost detailed guidance |
| Handbook D22, p. 253 | LCR detailed guidance |
| Handbook E5.6h, p. 320 | Application stress tests |

### 1.4 How These Fit in the Calculation Pipeline

```
Projection Engine (C0 per scenario)
        |
        v
BEL Calculator (biting scenario) ---> BEL_base
        |                                  |
        v                                  v
  Lapse Cost (LapC)              Risk Margin (RM)
        |                                  |
        v                                  v
  BEL_adjusted = BEL_base + LapC    TP = BEL_adjusted + RM
        |
        v
  LCR Check (pass/fail gate)
        |
        v
  Stress Tests (pass/fail gate)
```

---

## 2. Lapse Cost Calculation

### 2.1 Formula

**Rule Reference:** Rules Subpara. 29(1)(i)D; Handbook D37

```
LapC = (Lapse_Rate_Sigma / BSCR_Lapse_Shock) * Lapse_Capital_Requirement
```

Where:
- **Lapse_Rate_Sigma** -- standard deviation of the company's historical lapse experience (annualized)
- **BSCR_Lapse_Shock** -- the lapse shock factor prescribed by the BMA BSCR framework (the percentage shock applied to lapse rates in the BSCR calculation)
- **Lapse_Capital_Requirement** -- the capital charge for lapse risk as computed under the BSCR standard formula

### 2.2 Term Definitions

#### 2.2.1 Lapse_Rate_Sigma

The standard deviation of observed annual lapse rates over a credible historical period (typically 5-10 years). This measures volatility of actual lapse experience around the expected (mean) lapse rate.

```
Given annual lapse rates: [q_1, q_2, ..., q_n]
Mean lapse rate:  q_bar = (1/n) * SUM(q_i)
Sigma:            Lapse_Rate_Sigma = sqrt( (1/(n-1)) * SUM( (q_i - q_bar)^2 ) )
```

**Validation rules:**
- `n >= 5` (minimum 5 years of history required; fail loudly if fewer)
- `0 <= Lapse_Rate_Sigma <= 1.0` (sanity bound)
- If company-provided sigma is not available, use a conservative default (to be specified by BMA or set as a model parameter)

#### 2.2.2 BSCR_Lapse_Shock

The lapse shock factor from the BSCR standard formula. This is the magnitude of the lapse stress applied in the BSCR calculation (e.g., if the BSCR tests a 50% increase in lapse rates, the shock factor is 0.50).

**Input source:** BSCR filing or BMA-prescribed tables. Stored in `assumptions/bscr_parameters.yaml`.

**Validation rules:**
- `BSCR_Lapse_Shock > 0` (strictly positive; zero would cause division by zero)
- Typical range: 0.20 to 0.60

#### 2.2.3 Lapse_Capital_Requirement

The capital charge for lapse risk as calculated in the BSCR framework. This is the dollar amount of capital required to absorb the impact of the BSCR lapse shock on the liability portfolio.

**Input source:** BSCR filing (Long-Term Risk module, lapse sub-risk).

**Validation rules:**
- `Lapse_Capital_Requirement >= 0`
- For portfolios with no surrender options, this should be zero (and therefore LapC = 0)

### 2.3 Economic Interpretation

The ratio `Lapse_Rate_Sigma / BSCR_Lapse_Shock` is a scaling factor that converts the full BSCR lapse capital charge into a "best estimate variability" adjustment. Intuitively:

- The BSCR shock represents a tail-risk scenario (e.g., 1-in-200 year event).
- The sigma represents ordinary variability around expected experience.
- The ratio captures what fraction of the tail-risk capital is needed to cover normal lapse variability within the BEL.

If lapse experience is very stable (low sigma), LapC is small. If lapse experience is volatile relative to the BSCR shock, LapC is larger, reflecting the insurer's difficulty in predicting lapses.

### 2.4 How LapC Adjusts the BEL

When policyholder options exist in the liability block:

```
BEL_adjusted = BEL_base + LapC
```

Where `BEL_base` is the BEL from the biting scenario (C0 of the most onerous interest rate scenario). LapC is an additive increase to the BEL, making the liability valuation more conservative.

**When LapC = 0:** If the block has no policyholder options (e.g., payout annuities with no surrender value), lapse risk is not applicable and LapC = 0. The BEL is unchanged.

### 2.5 Lapse Stress Tests

**Rule Reference:** Rules Sched XXV Para 29(2); Handbook D37

Beyond computing LapC, the insurer must demonstrate that the SBA block maintains at least 100% ECR coverage under two lapse stress scenarios. These are pass/fail tests -- failure disqualifies the block from SBA treatment.

#### 2.5.1 Lapse Up Stress

```
stressed_lapse_rate = base_lapse_rate * 1.50
```

- Apply 150% of the expected (base) lapse rate to all periods in the projection.
- Re-run the full projection engine with the stressed lapse rates.
- Recalculate BEL under the stressed assumption.
- Check: ECR coverage ratio >= 100%.

#### 2.5.2 Lapse Down Stress

```
stressed_lapse_rate = max(base_lapse_rate * 0.50, 0.0)
```

- Apply 50% of the expected lapse rate (floored at zero -- no negative lapses).
- Re-run the full projection engine with the stressed lapse rates.
- Recalculate BEL under the stressed assumption.
- Check: ECR coverage ratio >= 100%.

**Why Lapse Down is a stress:** Lower lapses mean the insurer retains more liabilities for longer, requiring more assets to back them. For duration-mismatched portfolios, this can increase the BEL significantly.

#### 2.5.3 Stress Test Result Structure

```python
@dataclass(frozen=True)
class LapseStressResult:
    scenario: str              # "lapse_up" or "lapse_down"
    base_lapse_rate: float     # Original expected lapse rate
    stressed_lapse_rate: float # After applying multiplier
    bel_base: float            # BEL under base lapse assumption
    bel_stressed: float        # BEL under stressed lapse assumption
    ecr_coverage: float        # ECR coverage ratio under stress
    passes: bool               # ecr_coverage >= 1.0
```

### 2.6 Pseudocode

```python
@rule_ref("Rules Para 29(1)(i)D", "Handbook D37")
def compute_lapse_cost(
    lapse_rate_sigma: float,
    bscr_lapse_shock: float,
    lapse_capital_requirement: float,
    has_policyholder_options: bool,
) -> LapseCostResult:
    """
    Compute Lapse Cost (LapC) adjustment to BEL.

    Parameters
    ----------
    lapse_rate_sigma : float
        Standard deviation of historical annual lapse rates.
    bscr_lapse_shock : float
        BSCR lapse shock factor (e.g., 0.40 for a 40% shock).
    lapse_capital_requirement : float
        Dollar capital charge for lapse risk from BSCR.
    has_policyholder_options : bool
        Whether the liability block contains surrender/option features.

    Returns
    -------
    LapseCostResult with lapc, scaling_factor, and metadata.
    """
    # Gate: no policyholder options => no lapse cost
    if not has_policyholder_options:
        return LapseCostResult(
            lapc=0.0,
            scaling_factor=0.0,
            lapse_rate_sigma=lapse_rate_sigma,
            bscr_lapse_shock=bscr_lapse_shock,
            lapse_capital_requirement=lapse_capital_requirement,
            has_policyholder_options=False,
        )

    # Validate inputs
    if bscr_lapse_shock <= 0:
        raise ValueError(
            f"BSCR lapse shock must be strictly positive, got {bscr_lapse_shock}"
        )
    if lapse_rate_sigma < 0:
        raise ValueError(
            f"Lapse rate sigma must be non-negative, got {lapse_rate_sigma}"
        )
    if lapse_capital_requirement < 0:
        raise ValueError(
            f"Lapse capital requirement must be non-negative, got {lapse_capital_requirement}"
        )

    # Core formula
    scaling_factor = lapse_rate_sigma / bscr_lapse_shock
    lapc = scaling_factor * lapse_capital_requirement

    return LapseCostResult(
        lapc=lapc,
        scaling_factor=scaling_factor,
        lapse_rate_sigma=lapse_rate_sigma,
        bscr_lapse_shock=bscr_lapse_shock,
        lapse_capital_requirement=lapse_capital_requirement,
        has_policyholder_options=True,
    )


@rule_ref("Rules Para 29(2)", "Handbook D37")
def run_lapse_stress_tests(
    base_lapse_rate: float,
    projection_runner: Callable,  # function that re-runs projection with modified lapse
    ecr_calculator: Callable,     # function that computes ECR coverage
) -> list[LapseStressResult]:
    """
    Run Lapse Up (150%) and Lapse Down (50%) stress tests.
    Returns pass/fail for each.
    """
    results = []

    for scenario, multiplier in [("lapse_up", 1.50), ("lapse_down", 0.50)]:
        stressed_rate = max(base_lapse_rate * multiplier, 0.0)

        # Re-run projection with stressed lapse rate
        bel_stressed = projection_runner(lapse_rate=stressed_rate)
        bel_base = projection_runner(lapse_rate=base_lapse_rate)

        # Calculate ECR coverage under stress
        ecr_coverage = ecr_calculator(bel=bel_stressed)

        results.append(LapseStressResult(
            scenario=scenario,
            base_lapse_rate=base_lapse_rate,
            stressed_lapse_rate=stressed_rate,
            bel_base=bel_base,
            bel_stressed=bel_stressed,
            ecr_coverage=ecr_coverage,
            passes=(ecr_coverage >= 1.0),
        ))

    return results
```

---

## 3. LCR Calculation

### 3.1 Formula

**Rule Reference:** Rules Sched XXV Para 29(2)(iii); Handbook D22

```
LCR = Available_Liquidity_Sources / Potential_Surrender_Outflows
```

**Threshold:** `LCR >= 1.05` (105%)

If LCR < 105%, the liability block **fails** the LCR eligibility test and cannot use SBA treatment.

### 3.2 Available Liquidity Sources

Available liquidity is the sum of market values of eligible liquid assets, after applying regulatory haircuts by liquidity tier.

```
Available_Liquidity = SUM( asset_mv_i * (1 - haircut_i) )
    for each asset i where is_lcr_eligible(asset_i) = True
```

#### 3.2.1 LCR Haircut Table

| Liquidity Level | Asset Types | Haircut Range | Typical Haircut |
|----------------|-------------|---------------|-----------------|
| Level 1 | Cash, central bank reserves, sovereign bonds (AAA-AA) | 0% - 5% | 0% for cash; 5% for govt bonds |
| Level 2A | Investment-grade corporate bonds (A- or better), agency MBS, covered bonds | 10% - 15% | 15% |
| Level 2B | Below-IG corporate bonds, other marketable securities, RMBS | 25% - 50% | 50% |

**Level composition caps** (from Basel III / BMA adaptation):
- Level 2 total (2A + 2B) must not exceed 40% of total available liquidity
- Level 2B alone must not exceed 15% of total available liquidity

#### 3.2.2 Asset Eligibility

An asset is LCR-eligible if it is:
1. Freely tradeable (not encumbered, pledged, or restricted)
2. Not a Tier 3 / limited-basis asset (these are illiquid by definition)
3. Readily valued (active market, observable prices)

#### 3.2.3 Haircut Assignment Logic

```python
def assign_lcr_haircut(asset: Asset, haircut_table: LCRHaircutTable) -> float:
    """
    Determine the LCR haircut for a single asset based on its
    classification, rating, and liquidity tier.
    """
    if asset.asset_class == "cash":
        return 0.0
    elif asset.asset_class == "government_bond" and asset.rating in ("AAA", "AA+", "AA", "AA-"):
        return haircut_table.level_1_govt  # typically 0.05
    elif asset.asset_class in ("corporate_bond", "agency_mbs") and asset.rating >= "A-":
        return haircut_table.level_2a  # typically 0.15
    elif asset.asset_class in ("corporate_bond", "other_security"):
        return haircut_table.level_2b  # typically 0.50
    else:
        # Not LCR-eligible
        raise AssetNotEligibleError(f"Asset {asset.id} not eligible for LCR")
```

### 3.3 Potential Surrender Outflows

Two stress scenarios are evaluated; the binding (higher) outflow is used for the LCR denominator.

#### 3.3.1 Fast-Moving Deterioration

An immediate, severe surrender shock applied at time zero:

```
outflow_fast = surrender_value_total * mass_lapse_rate_fast
```

Where:
- `surrender_value_total` = aggregate cash surrender value of the in-force liability block
- `mass_lapse_rate_fast` = the instantaneous mass lapse rate under fast-moving stress (typically 20% or the BSCR lapse shock, whichever is higher)

This represents a scenario where negative publicity, rating downgrade, or market panic triggers an immediate wave of surrenders.

#### 3.3.2 Sustained Deterioration

A gradual surrender stress over 12 months:

```
outflow_sustained = surrender_value_total * sustained_lapse_rate * duration_factor
```

Where:
- `sustained_lapse_rate` = elevated (but lower than fast-moving) lapse rate sustained over 12 months (e.g., 10% annual rate)
- `duration_factor` = 1.0 (12-month horizon, so the annual rate applies directly)

This represents a prolonged period of elevated surrenders driven by sustained competitive or economic pressure.

#### 3.3.3 Binding Outflow

```
Potential_Surrender_Outflows = max(outflow_fast, outflow_sustained)
```

### 3.4 LCR Ratio Computation

```
LCR = Available_Liquidity / Potential_Surrender_Outflows
```

**Threshold check:**
```
passes = (LCR >= 1.05)
```

**Edge case:** If `Potential_Surrender_Outflows == 0` (no surrender optionality), the LCR test is not applicable. The function should return `LCR = inf` with `passes = True` and a flag indicating the test was not required.

### 3.5 Level Composition Cap Enforcement

After computing available liquidity, enforce the Level 2 caps:

```python
def enforce_level_caps(level_1: float, level_2a: float, level_2b: float) -> tuple[float, float, float]:
    """
    Enforce Basel III-style level composition caps.
    Level 2 total <= 40% of available.
    Level 2B <= 15% of available.
    """
    total_uncapped = level_1 + level_2a + level_2b

    # Cap Level 2B at 15% of total
    max_2b = 0.15 * total_uncapped
    level_2b_capped = min(level_2b, max_2b)

    # Cap total Level 2 at 40% of total
    max_level_2 = 0.40 * total_uncapped
    total_level_2 = level_2a + level_2b_capped
    if total_level_2 > max_level_2:
        # Scale down Level 2A proportionally, Level 2B already capped
        excess = total_level_2 - max_level_2
        level_2a_capped = level_2a - excess
        level_2a_capped = max(level_2a_capped, 0.0)
    else:
        level_2a_capped = level_2a

    return level_1, level_2a_capped, level_2b_capped
```

### 3.6 Pseudocode

```python
@rule_ref("Rules Para 29(2)(iii)", "Handbook D22")
def calculate_lcr(
    portfolio: list[Asset],
    surrender_value_total: float,
    mass_lapse_rate_fast: float,
    sustained_lapse_rate: float,
    haircut_table: LCRHaircutTable,
    has_policyholder_options: bool,
) -> LCRResult:
    """
    Compute Liquidity Coverage Ratio for an SBA liability block.

    Parameters
    ----------
    portfolio : list[Asset]
        Assets backing the SBA block, with market values and classifications.
    surrender_value_total : float
        Aggregate cash surrender value of the liability block.
    mass_lapse_rate_fast : float
        Mass lapse rate for fast-moving deterioration (e.g., 0.20).
    sustained_lapse_rate : float
        Elevated lapse rate for sustained deterioration (e.g., 0.10).
    haircut_table : LCRHaircutTable
        Regulatory haircuts by liquidity level.
    has_policyholder_options : bool
        Whether the block has surrender features.

    Returns
    -------
    LCRResult with ratio, available, outflows, passes flag.
    """
    # Gate: if no policyholder options, LCR test not applicable
    if not has_policyholder_options:
        return LCRResult(
            lcr=float("inf"),
            available_liquidity=0.0,
            potential_outflows=0.0,
            outflow_fast=0.0,
            outflow_sustained=0.0,
            passes=True,
            not_applicable=True,
        )

    # --- Compute available liquidity by level ---
    level_1_total = 0.0
    level_2a_total = 0.0
    level_2b_total = 0.0

    asset_details = []
    for asset in portfolio:
        if not is_lcr_eligible(asset):
            continue

        haircut = assign_lcr_haircut(asset, haircut_table)
        contribution = asset.market_value * (1 - haircut)
        level = classify_lcr_level(asset)

        if level == "1":
            level_1_total += contribution
        elif level == "2A":
            level_2a_total += contribution
        elif level == "2B":
            level_2b_total += contribution

        asset_details.append({
            "asset_id": asset.id,
            "market_value": asset.market_value,
            "haircut": haircut,
            "contribution": contribution,
            "level": level,
        })

    # Enforce level composition caps
    level_1_final, level_2a_final, level_2b_final = enforce_level_caps(
        level_1_total, level_2a_total, level_2b_total
    )
    available_liquidity = level_1_final + level_2a_final + level_2b_final

    # --- Compute potential surrender outflows ---
    outflow_fast = surrender_value_total * mass_lapse_rate_fast
    outflow_sustained = surrender_value_total * sustained_lapse_rate
    potential_outflows = max(outflow_fast, outflow_sustained)

    # --- Compute ratio ---
    if potential_outflows == 0:
        return LCRResult(
            lcr=float("inf"),
            available_liquidity=available_liquidity,
            potential_outflows=0.0,
            outflow_fast=outflow_fast,
            outflow_sustained=outflow_sustained,
            passes=True,
            not_applicable=True,
        )

    lcr = available_liquidity / potential_outflows
    passes = lcr >= 1.05

    return LCRResult(
        lcr=lcr,
        available_liquidity=available_liquidity,
        potential_outflows=potential_outflows,
        outflow_fast=outflow_fast,
        outflow_sustained=outflow_sustained,
        passes=passes,
        not_applicable=False,
        level_1=level_1_final,
        level_2a=level_2a_final,
        level_2b=level_2b_final,
        asset_details=pd.DataFrame(asset_details),
    )
```

---

## 4. Application Stress Tests

**Rule Reference:** Handbook E5.6h

These three stress tests are required as part of the SBA application and ongoing validation. They are pass/fail: failure at application means denial of SBA approval; failure during ongoing monitoring triggers remediation.

### 4.1 Combined Mass Lapse + Credit Spread Widening

**Rule Reference:** Handbook E5.6h(i)

Apply both shocks simultaneously:

**Mass Lapse Shock:**
```
mass_lapse_rate = max(0.20, bscr_lapse_shock)
surrendered_policies = in_force_count * mass_lapse_rate
```

**Credit Spread Widening (instantaneous):**

| Rating | Spread Widening (bps) |
|--------|----------------------|
| AAA | +277 |
| AA | +350 |
| A | +450 |
| BBB | +600 |
| BB | +900 |
| B+ and below | +1200 |

**Procedure:**
1. Apply mass lapse: reduce liability cash flows by the lapsed portion.
2. Apply credit spread widening: reduce asset market values by the spread-induced price decline.
3. Re-run projection engine with the shocked portfolio and liabilities.
4. Recalculate BEL.
5. Check: does the insurer maintain positive surplus (available capital > 0)?

```python
@rule_ref("Handbook E5.6h(i)")
def stress_combined_lapse_credit(
    portfolio: list[Asset],
    liabilities: LiabilityBlock,
    bscr_lapse_shock: float,
    spread_widening_table: dict[str, float],
    projection_runner: Callable,
) -> StressTestResult:
    # Mass lapse
    mass_lapse_rate = max(0.20, bscr_lapse_shock)
    stressed_liabilities = apply_mass_lapse(liabilities, mass_lapse_rate)

    # Credit spread widening -- reduce asset MVs
    stressed_portfolio = []
    for asset in portfolio:
        widening_bps = spread_widening_table[asset.rating]
        # Price impact: approximate using modified duration
        price_shock = asset.modified_duration * (widening_bps / 10_000)
        stressed_mv = asset.market_value * (1 - price_shock)
        stressed_portfolio.append(asset.with_market_value(stressed_mv))

    # Re-run projection
    bel_stressed = projection_runner(
        portfolio=stressed_portfolio,
        liabilities=stressed_liabilities,
    )

    # Check surplus
    total_assets_stressed = sum(a.market_value for a in stressed_portfolio)
    surplus = total_assets_stressed - bel_stressed
    passes = surplus > 0

    return StressTestResult(
        test_name="combined_mass_lapse_credit_spread",
        bel_stressed=bel_stressed,
        total_assets_stressed=total_assets_stressed,
        surplus=surplus,
        passes=passes,
        mass_lapse_rate=mass_lapse_rate,
        spread_widening_table=spread_widening_table,
    )
```

### 4.2 One-Notch Downgrade

**Rule Reference:** Handbook E5.6h(ii)

Apply a one-notch rating downgrade to **every** asset simultaneously:

```
AAA -> AA+, AA+ -> AA, AA -> AA-, ..., BBB- -> BB+, ..., B -> B-, B- -> CCC
```

**Procedure:**
1. Downgrade every asset's rating by one notch.
2. Recalculate credit costs (D&D) using the new lower ratings (higher default/downgrade rates).
3. Re-run the projection engine with the increased credit costs.
4. Recalculate BEL.
5. Check: impact is manageable (positive surplus maintained).

```python
@rule_ref("Handbook E5.6h(ii)")
def stress_one_notch_downgrade(
    portfolio: list[Asset],
    liabilities: LiabilityBlock,
    rating_notch_map: dict[str, str],
    projection_runner: Callable,
) -> StressTestResult:
    # Downgrade all assets by one notch
    stressed_portfolio = []
    for asset in portfolio:
        new_rating = rating_notch_map[asset.rating]  # e.g., "AA" -> "AA-"
        stressed_portfolio.append(asset.with_rating(new_rating))

    # Re-run projection (credit costs will be higher with lower ratings)
    bel_stressed = projection_runner(
        portfolio=stressed_portfolio,
        liabilities=liabilities,
    )

    total_assets = sum(a.market_value for a in stressed_portfolio)
    surplus = total_assets - bel_stressed
    passes = surplus > 0

    return StressTestResult(
        test_name="one_notch_downgrade",
        bel_stressed=bel_stressed,
        total_assets_stressed=total_assets,
        surplus=surplus,
        passes=passes,
    )
```

### 4.3 No Reinvestment into Limited-Basis Assets

**Rule Reference:** Handbook E5.6h(iii)

**Procedure:**
1. Configure the projection engine so that reinvestment into Tier 3 (limited-basis) assets is prohibited.
2. All reinvestment must go into Tier 1 or approved Tier 2 assets only.
3. Re-run the projection engine under this constraint.
4. Recalculate BEL.
5. Check: portfolio remains adequate (positive surplus maintained).

```python
@rule_ref("Handbook E5.6h(iii)")
def stress_no_limited_reinvestment(
    portfolio: list[Asset],
    liabilities: LiabilityBlock,
    projection_runner: Callable,
) -> StressTestResult:
    # Run projection with reinvestment restricted to Tier 1 and Tier 2
    bel_stressed = projection_runner(
        portfolio=portfolio,
        liabilities=liabilities,
        reinvestment_tiers_allowed=["tier_1", "tier_2"],  # Tier 3 excluded
    )

    total_assets = sum(a.market_value for a in portfolio)
    surplus = total_assets - bel_stressed
    passes = surplus > 0

    return StressTestResult(
        test_name="no_limited_reinvestment",
        bel_stressed=bel_stressed,
        total_assets_stressed=total_assets,
        surplus=surplus,
        passes=passes,
    )
```

---

## 5. Worked Example: Lapse Cost

### 5.1 Inputs

| Parameter | Value | Source |
|-----------|-------|--------|
| Historical lapse rates (5 years) | [3.2%, 2.8%, 4.1%, 3.5%, 3.0%] | Company experience study |
| BSCR Lapse Shock | 40% (0.40) | BSCR framework |
| Lapse Capital Requirement | $12,500,000 | BSCR filing |
| Policyholder options exist? | Yes | Product features |

### 5.2 Step 1: Compute Lapse_Rate_Sigma

```
Rates: [0.032, 0.028, 0.041, 0.035, 0.030]
Mean:  q_bar = (0.032 + 0.028 + 0.041 + 0.035 + 0.030) / 5 = 0.0332

Deviations from mean:
  0.032 - 0.0332 = -0.0012
  0.028 - 0.0332 = -0.0052
  0.041 - 0.0332 =  0.0078
  0.035 - 0.0332 =  0.0018
  0.030 - 0.0332 = -0.0032

Squared deviations:
  0.00000144
  0.00002704
  0.00006084
  0.00000324
  0.00001024

Sum of squared deviations: 0.00010280
Variance (n-1): 0.00010280 / 4 = 0.00002570
Sigma: sqrt(0.00002570) = 0.005070 (approximately 0.51%)
```

### 5.3 Step 2: Compute Scaling Factor

```
Scaling Factor = Lapse_Rate_Sigma / BSCR_Lapse_Shock
               = 0.005070 / 0.40
               = 0.012675
```

### 5.4 Step 3: Compute LapC

```
LapC = Scaling_Factor * Lapse_Capital_Requirement
     = 0.012675 * 12,500,000
     = $158,438
```

### 5.5 Step 4: Adjust BEL

Assuming `BEL_base = $950,000,000` (from biting scenario):

```
BEL_adjusted = BEL_base + LapC
             = 950,000,000 + 158,438
             = $950,158,438
```

### 5.6 Interpretation

The LapC of $158,438 is small relative to the $950M BEL (about 0.017%), reflecting the low volatility of the company's lapse experience (sigma of 0.51%) relative to the 40% BSCR shock. This is typical for well-understood books with stable surrender behavior.

### 5.7 Lapse Stress Test Results

Using `base_lapse_rate = 0.0332`:

| Scenario | Stressed Rate | BEL Impact | ECR Coverage | Pass? |
|----------|--------------|------------|--------------|-------|
| Lapse Up (150%) | 4.98% | BEL increases by $2.1M (more surrenders, forced disinvestment) | 112% | Yes |
| Lapse Down (50%) | 1.66% | BEL increases by $3.8M (liabilities persist longer, more reinvestment risk) | 108% | Yes |

Note: In this example, Lapse Down has a larger BEL impact than Lapse Up, reflecting a duration-mismatched portfolio where retaining liabilities longer is more costly than paying them out early.

---

## 6. Worked Example: LCR

### 6.1 Portfolio for LCR

| Asset ID | Class | Rating | Market Value ($M) | LCR Level | Haircut | Contribution ($M) |
|----------|-------|--------|-------------------|-----------|---------|-------------------|
| A1 | Cash | N/A | 50.0 | 1 | 0% | 50.00 |
| A2 | Govt Bond | AAA | 200.0 | 1 | 5% | 190.00 |
| A3 | Corp Bond | AA | 150.0 | 2A | 15% | 127.50 |
| A4 | Corp Bond | A | 100.0 | 2A | 15% | 85.00 |
| A5 | Corp Bond | BBB | 80.0 | 2B | 50% | 40.00 |
| A6 | Private Placement | BB | 30.0 | N/A | -- | 0.00 (not eligible) |
| A7 | Tier 3 Asset | A | 40.0 | N/A | -- | 0.00 (not eligible) |
| **Total** | | | **650.0** | | | |

### 6.2 Step 1: Compute Available Liquidity by Level

```
Level 1 (uncapped):  50.00 + 190.00 = 240.00
Level 2A (uncapped): 127.50 + 85.00 = 212.50
Level 2B (uncapped): 40.00
Total uncapped:      492.50
```

### 6.3 Step 2: Apply Level Composition Caps

```
Level 2B cap: 15% of 492.50 = 73.88   --> 40.00 <= 73.88, no cap binds
Level 2 total: 212.50 + 40.00 = 252.50
Level 2 cap: 40% of 492.50 = 197.00   --> 252.50 > 197.00, cap BINDS

Excess Level 2: 252.50 - 197.00 = 55.50
Reduce Level 2A by 55.50: 212.50 - 55.50 = 157.00

Final:
  Level 1:  240.00
  Level 2A: 157.00
  Level 2B:  40.00
  Available Liquidity = 437.00
```

### 6.4 Step 3: Compute Potential Surrender Outflows

```
Surrender value total: $500,000,000

Fast-moving: 500.0 * 0.20 = 100.0
Sustained:   500.0 * 0.10 =  50.0

Binding outflow: max(100.0, 50.0) = 100.0
```

### 6.5 Step 4: Compute LCR

```
LCR = 437.00 / 100.00 = 4.37 (437%)
```

### 6.6 Step 5: Threshold Check

```
4.37 >= 1.05  -->  PASSES
```

### 6.7 Interpretation

The portfolio has very strong liquidity coverage (437% vs. the 105% minimum). The Level 2 composition cap reduced available liquidity from $492.5M to $437.0M (a $55.5M reduction), illustrating how the caps can constrain portfolios heavy in corporate bonds. Even after the cap, the ratio is well above the threshold.

---

## 7. Edge Cases

### 7.1 Lapse Cost Edge Cases

| Condition | Behavior |
|-----------|----------|
| No policyholder options | LapC = 0; skip calculation entirely |
| `Lapse_Rate_Sigma = 0` | LapC = 0 (no variability => no lapse cost). Log a warning -- zero sigma with policyholder options is unusual |
| `BSCR_Lapse_Shock = 0` | **FAIL LOUDLY.** Division by zero. This should never happen for a block with policyholder options. Raise `ValueError` |
| `Lapse_Capital_Requirement = 0` | LapC = 0. Valid for blocks where BSCR determines zero lapse capital |
| Very high sigma (> 0.10) | Calculate normally but flag for review. Sigma above 10% absolute is extreme and may indicate data quality issues |
| Fewer than 5 years of history | **FAIL LOUDLY.** Insufficient data for credible sigma. Raise `InsufficientDataError` |
| Negative lapse rates in history | **FAIL LOUDLY.** Lapse rates must be in [0, 1]. Raise `ValidationError` |
| Lapse Down produces rate < 0 | Floor at 0.0 (already handled by `max(rate * 0.50, 0.0)`) |

### 7.2 LCR Edge Cases

| Condition | Behavior |
|-----------|----------|
| No surrender optionality | LCR not applicable. Return `passes = True`, `not_applicable = True` |
| No eligible liquid assets | `Available_Liquidity = 0`. If outflows > 0, LCR = 0 and **FAILS** |
| Zero surrender value | `Potential_Outflows = 0`. LCR = infinity, passes. Flag `not_applicable` |
| All assets are Tier 3 | No LCR-eligible assets. Available = 0. Likely fails |
| Level 2 caps reduce available below outflows | LCR may fail despite raw liquidity being adequate. Report both uncapped and capped figures |
| Negative market values | **FAIL LOUDLY.** `Raise ValidationError`. Market values must be >= 0 |
| Haircut >= 100% | **FAIL LOUDLY.** `Raise ValidationError`. Haircuts must be in [0, 1) |
| LCR exactly 105% | **PASSES** (threshold is >=, not >) |

### 7.3 Stress Test Edge Cases

| Condition | Behavior |
|-----------|----------|
| BSCR lapse shock < 20% | Combined stress uses 20% mass lapse (the floor) |
| Asset has no rating | Cannot apply spread widening or downgrade. **FAIL LOUDLY** -- all assets must have ratings |
| Asset already at lowest rating (CCC or below) | One-notch downgrade leaves it at CCC (cannot downgrade further) or moves to default treatment |
| Portfolio has no Tier 3 assets | No-reinvestment stress is trivially satisfied (no change from base). Still run and report |

---

## 8. Implementation Notes

### 8.1 Pydantic Models

```python
from pydantic import BaseModel, Field, field_validator

class LapseCostInputs(BaseModel):
    """Validated inputs for Lapse Cost calculation."""
    lapse_rate_history: list[float] = Field(
        ..., min_length=5,
        description="Annual lapse rates for credible historical period"
    )
    bscr_lapse_shock: float = Field(
        ..., gt=0, le=1.0,
        description="BSCR lapse shock factor"
    )
    lapse_capital_requirement: float = Field(
        ..., ge=0,
        description="BSCR lapse capital charge in dollars"
    )
    has_policyholder_options: bool = Field(
        ..., description="Whether the block has surrender/option features"
    )

    @field_validator("lapse_rate_history")
    @classmethod
    def validate_lapse_rates(cls, v):
        for rate in v:
            if not 0 <= rate <= 1:
                raise ValueError(f"Lapse rate {rate} not in [0, 1]")
        return v


class LCRHaircutTable(BaseModel):
    """Regulatory haircuts by liquidity level."""
    level_1_cash: float = Field(default=0.00, ge=0, lt=1)
    level_1_govt: float = Field(default=0.05, ge=0, lt=1)
    level_2a: float = Field(default=0.15, ge=0, lt=1)
    level_2b: float = Field(default=0.50, ge=0, lt=1)


class LapseCostResult(BaseModel):
    """Output of Lapse Cost calculation."""
    lapc: float = Field(..., ge=0, description="Lapse Cost dollar amount")
    scaling_factor: float = Field(..., ge=0)
    lapse_rate_sigma: float = Field(..., ge=0)
    bscr_lapse_shock: float = Field(..., gt=0)
    lapse_capital_requirement: float = Field(..., ge=0)
    has_policyholder_options: bool


class LCRResult(BaseModel):
    """Output of LCR calculation."""
    lcr: float = Field(..., description="Liquidity Coverage Ratio")
    available_liquidity: float = Field(..., ge=0)
    potential_outflows: float = Field(..., ge=0)
    outflow_fast: float = Field(..., ge=0)
    outflow_sustained: float = Field(..., ge=0)
    passes: bool = Field(..., description="LCR >= 105%")
    not_applicable: bool = Field(default=False)
    level_1: float = Field(default=0.0, ge=0)
    level_2a: float = Field(default=0.0, ge=0)
    level_2b: float = Field(default=0.0, ge=0)


class StressTestResult(BaseModel):
    """Output of an application stress test."""
    test_name: str
    bel_stressed: float
    total_assets_stressed: float
    surplus: float
    passes: bool
```

### 8.2 Rule References

Every public function in both modules must use the `@rule_ref` decorator:

```python
@rule_ref("Rules Para 29(1)(i)D", "Handbook D37, pp. 310-313")
def compute_lapse_cost(...): ...

@rule_ref("Rules Para 29(2)", "Handbook D37")
def run_lapse_stress_tests(...): ...

@rule_ref("Rules Para 29(2)(iii)", "Handbook D22, p. 253")
def calculate_lcr(...): ...

@rule_ref("Handbook E5.6h(i)")
def stress_combined_lapse_credit(...): ...

@rule_ref("Handbook E5.6h(ii)")
def stress_one_notch_downgrade(...): ...

@rule_ref("Handbook E5.6h(iii)")
def stress_no_limited_reinvestment(...): ...
```

### 8.3 Assumptions Files

LCR haircuts and stress parameters should be stored in versioned YAML:

```yaml
# assumptions/v2024/lcr_haircuts.yaml
level_1_cash: 0.00
level_1_govt: 0.05
level_2a: 0.15
level_2b: 0.50
level_2_cap: 0.40
level_2b_cap: 0.15

# assumptions/v2024/stress_parameters.yaml
mass_lapse_floor: 0.20
lapse_up_multiplier: 1.50
lapse_down_multiplier: 0.50
sustained_lapse_rate: 0.10
credit_spread_widening:
  AAA: 277
  AA: 350
  A: 450
  BBB: 600
  BB: 900
  B_plus_and_below: 1200
```

### 8.4 Integration with BEL and Risk Margin

LapC feeds into the BEL, which feeds into the Risk Margin:

```
BEL_adjusted = BEL_base + LapC
TP = BEL_adjusted + Risk_Margin

Where Risk_Margin = CoC * SUM(Modified_ECR(t) / (1 + r(t+1))^(t+1))
```

The Modified ECR at each future time step must include the lapse component. If the LapC is material, the projected ECR must reflect ongoing lapse risk.

### 8.5 Audit Trail

Both modules must log their inputs, intermediate values, and outputs to the run's audit trail (SQLite + JSON Lines). Key audit points:

- **LapC:** log sigma computation (all rates, mean, variance), scaling factor, final LapC
- **LCR:** log each asset's level classification, haircut, contribution; level cap adjustments; both outflow scenarios; final ratio
- **Stress tests:** log all shocked values, re-projected BEL, surplus

### 8.6 Numerical Precision

Per the project's precision standards (Tech Spec 12.2):
- Monetary values: 2 decimal places (dollars and cents)
- Rates: 6 decimal places
- Basis points: 2 decimal places
- LCR ratio: 4 decimal places (e.g., 1.0523)
- Internal calculations: 64-bit floating point (Python default)

### 8.7 Testing Strategy

| Test | Purpose |
|------|---------|
| Golden test: LapC with known inputs | Verify formula produces exact expected result from Section 5 |
| Golden test: LCR with known portfolio | Verify level caps and ratio from Section 6 |
| Property test: LapC >= 0 always | Hypothesis: for any valid inputs, LapC is non-negative |
| Property test: LCR monotonic in asset MVs | Adding liquid assets should not decrease LCR (excluding cap effects) |
| Edge: zero sigma | LapC = 0 |
| Edge: BSCR shock = 0 | Raises ValueError |
| Edge: no policyholder options | Both LapC and LCR return trivial/not-applicable results |
| Edge: Level 2 cap binding | Verify cap enforcement reduces available liquidity correctly |
| Stress: combined lapse + credit | Verify both shocks applied simultaneously |
| Stress: one-notch downgrade | Verify rating map applied to all assets |
| Stress: no-reinvestment | Verify Tier 3 excluded from reinvestment |
| Regression: illustrative calculation | Cross-check against BMA illustrative example when lapse/LCR data available |
