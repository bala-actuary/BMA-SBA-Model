# Algorithm Specification: Risk Margin Calculation

## Document Control

| Field | Value |
|---|---|
| **Spec ID** | ALGO-006 |
| **Target Module** | `calculations/risk_margin.py` |
| **Supporting Modules** | `curves/curve_builder.py` (risk-free rates), `projection/cashflow_engine.py` (liability runoff), `rules/vYYYY/risk_margin_params.py` (CoC, ECR inputs) |
| **Status** | Implementation-Ready Draft |
| **Date** | March 2026 |
| **BMA Rules** | Rules Subpara 36(4), pp. 179-180; Handbook D38.5, p. 314 |
| **Cross-References** | [ALGO-001](01_Yield_Curve_Construction.md) (base risk-free curve), [ALGO-002](02_Projection_Engine.md) (projection engine / BEL), [ALGO-003](03_Credit_Costs_DD.md) (credit costs) |

---

## 1. Overview

### 1.1 Purpose

The Risk Margin (RM) represents the cost a hypothetical third party would require, over and above the Best Estimate Liability (BEL), to take on the insurance obligations. It is calculated using a Cost-of-Capital (CoC) method prescribed by the BMA. Together, BEL and RM form the Technical Provision:

```
Technical Provision = BEL + Risk Margin
```

This module computes the Risk Margin after the BEL has been determined by the projection engine (ALGO-002). It is the final step before the Technical Provision can be reported.

### 1.2 What This Module Does

1. Accepts the current ECR (or its components), the base risk-free curve, and a liability exposure runoff profile.
2. Projects the Modified ECR at each future time step using a proportional runoff of remaining liability exposure.
3. Discounts the projected Modified ECR stream back to T=0 using risk-free rates (no spread).
4. Multiplies the discounted sum by the prescribed Cost of Capital rate (6%).
5. Returns the Risk Margin, a detailed breakdown DataFrame, and the resulting Technical Provision.

### 1.3 Architectural Boundary

**This module is pure Python.** It lives in `calculations/`, not `curves/`, so it does not import QuantLib. Risk-free discount factors arrive as plain NumPy arrays or callables from the `curves/` package. This is consistent with the project's QuantLib firewall constraint.

```
curves/                          calculations/
  curve_builder.py  ---------->    risk_margin.py    (THIS MODULE)
  [QuantLib boundary]              [Pure Python only]

projection/
  cashflow_engine.py  ---------->  (provides liability runoff profile)
```

### 1.4 Key Rule References

| Rule | Description | Used In |
|---|---|---|
| Subpara 36(4), p. 179 | Risk Margin = CoC x sum of discounted future Modified ECR | Section 2 |
| Handbook D38.5, p. 314 | Risk Margin methodology and CoC rate | Section 2 |
| Rules Tables 1A-2B | BSCR risk charge factors | Section 3.2 |
| Handbook D20-D28 | BSCR component calculations | Section 3.2 |
| Handbook D29-D37 | Insurance risk charges (mortality, longevity, lapse, expense) | Section 3.2 |
| Rules Para 28(10) | BEL = MV_assets + C0_biting | Section 5 |

---

## 2. Risk Margin Formula

### 2.1 Core Formula

Per Rules Subpara 36(4) and Handbook D38.5:

```
                   T
RM = CoC * SUM   [ Modified_ECR(t) / (1 + r(t+1))^(t+1) ]
                  t=0
```

Where:

| Symbol | Definition | Source |
|---|---|---|
| **CoC** | Cost of Capital rate = 6% (0.06) | Prescribed by BMA, Handbook D38.5 |
| **Modified_ECR(t)** | Projected Enhanced Capital Requirement at future time t | Section 3 |
| **r(t+1)** | Risk-free spot rate for maturity (t+1) years, from base scenario curve | ALGO-001, base curve |
| **T** | Final projection year (last year with nonzero liability exposure) | From liability cash flow schedule |

### 2.2 Discount Factor Form

Equivalently, using discount factors from the base curve:

```
                   T
RM = CoC * SUM   [ Modified_ECR(t) * DF(t+1) ]
                  t=0
```

Where `DF(t+1) = 1 / (1 + r(t+1))^(t+1)` is the risk-free discount factor for maturity (t+1) from the base scenario curve (no spread adjustment, no scenario shift).

### 2.3 Why Base Curve Only

The discount rates used in the Risk Margin calculation are always from the **base scenario** (Para 28(7)(a) -- no interest rate adjustment). The Risk Margin is not recalculated under each of the 9 stress scenarios. This is because:

- The Risk Margin reflects the cost of holding capital against non-hedgeable risks (insurance risks), not interest rate risk.
- Interest rate risk is already captured in the BEL through the 9-scenario projection.
- The BMA formula specifies `r(t)` as the risk-free rate, implying the base (unadjusted) curve.

### 2.4 Summation Index Convention

The BMA formula uses `t = 0, 1, ..., T` as the time index for the ECR and `t+1` as the discount maturity. This means:

| t | Interpretation | Discounted at maturity |
|---|---|---|
| 0 | ECR at valuation date (now) | 1 year |
| 1 | ECR after 1 year of runoff | 2 years |
| ... | ... | ... |
| T | ECR at final projection year | T+1 years |

The ECR at t=0 is the **current** ECR before any runoff. It is discounted 1 year because the CoC represents the cost of holding capital for one year -- the first year's capital cost is payable at the end of year 1.

---

## 3. Modified ECR Projection

The core challenge in Risk Margin calculation is projecting ECR into the future. The BMA does not prescribe a single exact methodology, noting this is "a common area of interpretation difference" (Tech Specs, Section 9.2). This spec defines two methods and recommends a primary approach for the benchmark model.

### 3.1 Method 1: Proportional Runoff (Recommended for Benchmark)

**This is the recommended approach for the benchmark model.** It is simple, transparent, and conservative.

#### 3.1.1 Formula

```
Modified_ECR(t) = ECR(0) * remaining_exposure(t) / total_exposure(0)
```

Where:

| Term | Definition |
|---|---|
| **ECR(0)** | Enhanced Capital Requirement at valuation date (input from BSCR calculation) |
| **remaining_exposure(t)** | Sum of undiscounted future liability cash flows from time t onward |
| **total_exposure(0)** | Sum of all undiscounted liability cash flows = remaining_exposure(0) |

#### 3.1.2 Exposure Runoff Factor

Define the runoff factor:

```
alpha(t) = remaining_exposure(t) / total_exposure(0)
```

Properties:
- `alpha(0) = 1.0` (100% of exposure remains at valuation date)
- `alpha(t)` is monotonically non-increasing
- `alpha(T+1) = 0.0` (all liabilities have run off after the final payment)

Then:

```
Modified_ECR(t) = ECR(0) * alpha(t)
```

#### 3.1.3 Computing Remaining Exposure

Given a liability cash flow vector `L = [L(1), L(2), ..., L(T)]` where `L(t)` is the total liability payment at time t:

```
remaining_exposure(t) = SUM(s = t+1 to T) [ L(s) ]
```

Note: `remaining_exposure(t)` represents liabilities **not yet paid** as of time t. After the payment at time t, the remaining exposure drops by `L(t)`.

```python
# Pseudocode: compute runoff factors from liability cash flows
def compute_runoff_factors(liability_cfs: np.ndarray) -> np.ndarray:
    """
    Parameters
    ----------
    liability_cfs : array of shape (T,)
        liability_cfs[i] = payment at time i+1 (i.e., L(1)..L(T))

    Returns
    -------
    alpha : array of shape (T+1,)
        alpha[t] = runoff factor at time t, for t = 0..T
    """
    total_exposure = np.sum(liability_cfs)
    if total_exposure <= 0:
        raise ValueError("Total liability exposure must be positive")

    # remaining_exposure(t) = sum of L(t+1) through L(T)
    # This is the reverse cumulative sum
    reverse_cumsum = np.cumsum(liability_cfs[::-1])[::-1]
    # reverse_cumsum[i] = sum of liability_cfs[i:]  = sum of L(i+1) through L(T)

    # alpha(0) = total_exposure / total_exposure = 1.0
    # alpha(t) for t = 1..T-1: remaining_exposure(t) / total_exposure
    # alpha(T) = 0.0  (nothing remains after last payment)
    remaining = np.zeros(len(liability_cfs) + 1)
    remaining[0] = total_exposure
    remaining[1:len(liability_cfs)] = reverse_cumsum[1:]
    remaining[-1] = 0.0

    alpha = remaining / total_exposure
    return alpha
```

#### 3.1.4 Rationale for Proportional Runoff

- **Simplicity:** One input (ECR(0)) plus the liability cash flow schedule already available from ALGO-002.
- **Conservatism:** Assumes all BSCR risk components run off at the same rate as the aggregate liability. In practice, some risks (e.g., expense risk) may persist longer, making this a reasonable benchmark assumption.
- **Auditability:** The calculation is fully transparent -- no hidden model points or component-level assumptions.
- **Industry precedent:** Proportional runoff is the standard approach under Solvency II for the cost-of-capital Risk Margin (EIOPA guidelines).

### 3.2 Method 2: Component-Level ECR Projection (Advanced)

This method projects each BSCR risk charge separately, then reaggregates with the correlation matrix. It is more accurate but requires significantly more inputs and assumptions.

#### 3.2.1 BSCR Structure

The Enhanced Capital Requirement is:

```
ECR = max(MSM, BSCR)
```

Where MSM = Minimum Solvency Margin, and:

```
Basic BSCR = sqrt( SUM_i SUM_j [ Corr(i,j) * C(i) * C(j) ] )
```

The relevant risk charges for long-term insurance SBA business are:

| Risk Charge | Symbol | Runoff Driver | BMA Reference |
|---|---|---|---|
| Fixed Income Risk | C_fi | MV of fixed income assets | Handbook D20 |
| Equity Risk | C_eq | MV of equity assets | Handbook D21 |
| Credit Risk | C_cr | Debtor balances | Handbook D22 |
| Long-Term Mortality | C_mort | Sum at risk (SAR) | Handbook D29 |
| Long-Term Morbidity | C_morb | Morbidity exposure | Handbook D30 |
| Longevity Risk | C_long | PV of annuity liabilities | Handbook D31 |
| Lapse Risk | C_lapse | Surrender value exposure | Handbook D33-D37 |
| Expense Risk | C_exp | PV of future expenses | Handbook D32 |

#### 3.2.2 Component-Level Projection

Each risk charge is projected by its own runoff driver:

```
C_i(t) = C_i(0) * driver_i(t) / driver_i(0)
```

Then the future BSCR is reaggregated:

```
Modified_ECR(t) = sqrt( SUM_i SUM_j [ Corr(i,j) * C_i(t) * C_j(t) ] )
```

#### 3.2.3 Why This Is Not the Default

- Requires the full BSCR breakdown and correlation matrix as inputs.
- Each risk charge needs a separate runoff driver, which may not be available from the company's SBA submission.
- For a **regulator's benchmark model** that is verifying submissions, the proportional method is more appropriate because it does not require the regulator to replicate the company's internal BSCR model.
- This method is available as a **configurable option** for sensitivity analysis.

### 3.3 Configuration

```python
class RiskMarginMethod(str, Enum):
    PROPORTIONAL = "proportional"      # Method 1: recommended default
    COMPONENT    = "component_level"   # Method 2: advanced option

class RiskMarginConfig(BaseModel):
    """Configuration for Risk Margin calculation. Frozen after creation."""
    method: RiskMarginMethod = RiskMarginMethod.PROPORTIONAL
    coc_rate: float = 0.06  # BMA-prescribed, not user-configurable in production
    ecr_at_t0: float        # Current ECR — required input
    # Only needed for component method:
    bscr_components: Optional[dict[str, float]] = None
    correlation_matrix: Optional[np.ndarray] = None
    component_drivers: Optional[dict[str, np.ndarray]] = None

    model_config = ConfigDict(frozen=True, arbitrary_types_allowed=True)
```

---

## 4. Discount Rate

### 4.1 Source

The discount rates for Risk Margin are taken from the **base scenario risk-free curve** constructed in ALGO-001. No spread adjustment is applied -- the Risk Margin discounting uses pure risk-free rates.

### 4.2 Extracting Discount Factors

```python
def get_rm_discount_factors(
    base_curve: BaseCurve,
    max_tenor: int,
) -> np.ndarray:
    """
    Extract discount factors for Risk Margin calculation.

    Parameters
    ----------
    base_curve : BaseCurve
        The base scenario risk-free curve from ALGO-001.
    max_tenor : int
        Maximum projection year T. Need DF for maturities 1..T+1.

    Returns
    -------
    df : array of shape (max_tenor + 1,)
        df[t] = discount factor for maturity (t+1), for t = 0..T
        i.e., df[0] = DF(1 year), df[T] = DF(T+1 years)
    """
    maturities = np.arange(1, max_tenor + 2)  # 1, 2, ..., T+1
    df = np.array([base_curve.discount_factor(m) for m in maturities])
    return df
```

### 4.3 Rate vs. Discount Factor

The BMA formula is written with rates: `1 / (1 + r(t+1))^(t+1)`. In implementation, we use discount factors directly from the curve object, which avoids any compounding convention ambiguity. The curve object (ALGO-001) handles the conversion internally.

---

## 5. Integration with BEL and Technical Provision

### 5.1 Calculation Sequence

The Risk Margin is computed **after** the BEL. The full sequence is:

```
Step 1: Build base curve and 9 scenario curves          (ALGO-001)
Step 2: For each scenario, find C0 via root-finding      (ALGO-002)
Step 3: Identify biting scenario = argmax(C0)             (ALGO-002)
Step 4: BEL = MV_assets(T=0) + C0_biting                 (ALGO-002)
Step 5: Risk Margin = CoC * discounted_future_ECR         (THIS MODULE)
Step 6: Technical Provision = BEL + Risk Margin           (THIS MODULE)
```

### 5.2 Inputs Required

| Input | Source | Description |
|---|---|---|
| `ecr_at_t0` | Company submission or BSCR calculation | Current Enhanced Capital Requirement |
| `liability_cfs` | From projection engine (ALGO-002) | Undiscounted liability cash flow vector L(1)..L(T) |
| `base_curve` | From curve builder (ALGO-001) | Base scenario risk-free curve |
| `bel` | From projection engine (ALGO-002) | Best Estimate Liability |

### 5.3 Outputs

| Output | Type | Description |
|---|---|---|
| `risk_margin` | float | The computed Risk Margin |
| `technical_provision` | float | TP = BEL + RM |
| `rm_detail_df` | pd.DataFrame | Period-by-period breakdown (see Section 5.4) |

### 5.4 Detail DataFrame Schema

The `rm_detail_df` provides full auditability of the calculation:

| Column | Type | Description |
|---|---|---|
| `t` | int | Projection year (0 to T) |
| `remaining_exposure` | float | Sum of future liability payments from t onward |
| `alpha` | float | Runoff factor = remaining_exposure / total_exposure |
| `modified_ecr` | float | ECR(0) * alpha(t) |
| `discount_maturity` | int | t + 1 |
| `discount_factor` | float | DF(t+1) from base risk-free curve |
| `discounted_ecr` | float | modified_ecr * discount_factor |
| `cumulative_sum` | float | Running sum of discounted_ecr from t=0 to current t |

The final Risk Margin = CoC * cumulative_sum at t=T.

---

## 6. Worked Example

### 6.1 Setup

A simplified 5-period example to demonstrate the full calculation.

**Given:**

| Parameter | Value |
|---|---|
| ECR at T=0 | 500,000 |
| Cost of Capital (CoC) | 6% |
| Projection horizon | 5 years |

**Liability cash flows (undiscounted):**

| Year t | Liability Payment L(t) |
|---|---|
| 1 | 200,000 |
| 2 | 250,000 |
| 3 | 200,000 |
| 4 | 150,000 |
| 5 | 200,000 |
| **Total** | **1,000,000** |

**Base risk-free spot rates (annual compounding):**

| Maturity | Spot Rate r(m) | Discount Factor DF(m) |
|---|---|---|
| 1 | 4.00% | 1 / 1.04^1 = 0.96154 |
| 2 | 4.10% | 1 / 1.041^2 = 0.92248 |
| 3 | 4.15% | 1 / 1.0415^3 = 0.88517 |
| 4 | 4.20% | 1 / 1.042^4 = 0.84834 |
| 5 | 4.25% | 1 / 1.0425^5 = 0.81130 |
| 6 | 4.30% | 1 / 1.043^6 = 0.77413 |

### 6.2 Step 1: Compute Remaining Exposure and Runoff Factors

| t | Remaining Exposure (liabilities from t+1 onward) | alpha(t) |
|---|---|---|
| 0 | L(1)+L(2)+L(3)+L(4)+L(5) = 1,000,000 | 1.00000 |
| 1 | L(2)+L(3)+L(4)+L(5) = 800,000 | 0.80000 |
| 2 | L(3)+L(4)+L(5) = 550,000 | 0.55000 |
| 3 | L(4)+L(5) = 350,000 | 0.35000 |
| 4 | L(5) = 200,000 | 0.20000 |
| 5 | 0 | 0.00000 |

Note: At t=5, all liabilities have been paid, so the ECR drops to zero. The summation effectively runs from t=0 to t=4 (the last period with nonzero exposure).

### 6.3 Step 2: Compute Modified ECR

```
Modified_ECR(t) = ECR(0) * alpha(t) = 500,000 * alpha(t)
```

| t | alpha(t) | Modified_ECR(t) |
|---|---|---|
| 0 | 1.00000 | 500,000 |
| 1 | 0.80000 | 400,000 |
| 2 | 0.55000 | 275,000 |
| 3 | 0.35000 | 175,000 |
| 4 | 0.20000 | 100,000 |
| 5 | 0.00000 | 0 |

### 6.4 Step 3: Discount Modified ECR

Each Modified_ECR(t) is discounted at maturity (t+1):

| t | Modified_ECR(t) | Discount Maturity | DF(t+1) | Discounted ECR |
|---|---|---|---|---|
| 0 | 500,000 | 1 | 0.96154 | 480,770 |
| 1 | 400,000 | 2 | 0.92248 | 368,992 |
| 2 | 275,000 | 3 | 0.88517 | 243,422 |
| 3 | 175,000 | 4 | 0.84834 | 148,460 |
| 4 | 100,000 | 5 | 0.81130 | 81,130 |
| 5 | 0 | 6 | 0.77413 | 0 |
| | | | **Sum** | **1,322,774** |

### 6.5 Step 4: Apply Cost of Capital

```
Risk Margin = CoC * Sum of Discounted ECR
            = 0.06 * 1,322,774
            = 79,366
```

### 6.6 Step 5: Technical Provision

Assuming BEL = 10,500,000 (from ALGO-002):

```
Technical Provision = BEL + Risk Margin
                    = 10,500,000 + 79,366
                    = 10,579,366
```

### 6.7 Verification Checks

1. **RM is positive:** 79,366 > 0. Correct -- there are non-hedgeable risks remaining.
2. **RM << BEL:** RM / BEL = 0.76%. Typical range for long-duration life insurance is 0.5% to 3% of BEL. This is reasonable.
3. **RM decreases if rates rise:** Higher risk-free rates produce lower discount factors, reducing the discounted ECR sum and hence RM.
4. **RM = 0 if ECR(0) = 0:** If the company has zero capital requirement, RM = 0. Correct edge case.

---

## 7. Edge Cases and Validation

### 7.1 Zero or Negative Liability Cash Flows

```python
if total_exposure <= 0:
    raise ValueError(
        f"Total liability exposure must be positive for RM calculation, "
        f"got {total_exposure}. Check liability cash flow inputs."
    )
```

A zero total exposure would mean there are no liabilities to reserve against. This is an input error, not a valid scenario.

### 7.2 ECR at T=0 is Zero

```python
if ecr_at_t0 == 0.0:
    # Legitimate: company has zero capital requirement
    # RM = 0, TP = BEL + 0 = BEL
    return RiskMarginResult(risk_margin=0.0, technical_provision=bel, ...)
```

### 7.3 ECR at T=0 is Negative

```python
if ecr_at_t0 < 0:
    raise ValueError(
        f"ECR at T=0 must be non-negative, got {ecr_at_t0}. "
        f"A negative ECR is not valid under BMA rules."
    )
```

### 7.4 Very Long Projection Horizons

For long-duration liabilities (e.g., whole life, annuities), T can be 60-100 years. The implementation must handle this efficiently, but there is no computational concern -- the calculation is a simple vector dot product.

```python
# Efficient for any T: vectorized computation
discounted_ecr_sum = np.dot(modified_ecr_vector, discount_factor_vector)
risk_margin = coc_rate * discounted_ecr_sum
```

### 7.5 Missing Discount Factors Beyond Curve Tenor

If the base risk-free curve does not extend to maturity T+1:

```python
if max_required_maturity > base_curve.max_tenor:
    raise ValueError(
        f"Base curve max tenor ({base_curve.max_tenor}) is shorter than "
        f"required RM discount maturity ({max_required_maturity}). "
        f"Extend the base curve or reduce the projection horizon."
    )
```

The base curve must be extrapolated (flat-forward or Smith-Wilson, per ALGO-001) to cover the full projection horizon before calling this module.

### 7.6 Non-Monotonic Runoff

If liability cash flows are structured such that `remaining_exposure(t)` is not strictly decreasing (e.g., deferred annuities with a gap period), the alpha factors will still be non-increasing because they are cumulative sums from the right. However, they may have flat segments (alpha(t) = alpha(t-1) during periods with zero liability payments). This is valid and does not require special handling.

### 7.7 CoC Rate Override

The CoC rate is prescribed at 6% by the BMA. The implementation accepts it as a parameter for testing purposes but should log a warning if a non-standard rate is used:

```python
if abs(config.coc_rate - 0.06) > 1e-10:
    logger.warning(
        f"Non-standard CoC rate: {config.coc_rate:.4%}. "
        f"BMA prescribes 6%. Override used for sensitivity testing."
    )
```

---

## 8. Implementation Notes

### 8.1 Function Signature

```python
@rule_ref("Subpara 36(4), Handbook D38.5", "Risk Margin: CoC method")
def calculate_risk_margin(
    ecr_at_t0: float,
    liability_cfs: np.ndarray,
    base_curve: BaseCurve,
    bel: float,
    config: RiskMarginConfig | None = None,
) -> RiskMarginResult:
    """
    Calculate the Risk Margin using the BMA Cost-of-Capital method.

    Parameters
    ----------
    ecr_at_t0 : float
        Enhanced Capital Requirement at valuation date. Must be >= 0.
    liability_cfs : np.ndarray
        Undiscounted liability cash flows, shape (T,).
        liability_cfs[i] = payment at time i+1.
    base_curve : BaseCurve
        Base scenario risk-free curve (from ALGO-001).
    bel : float
        Best Estimate Liability (from ALGO-002).
    config : RiskMarginConfig, optional
        Calculation configuration. Defaults to proportional method with 6% CoC.

    Returns
    -------
    RiskMarginResult
        Contains risk_margin, technical_provision, detail_df, and audit metadata.

    Raises
    ------
    ValueError
        If inputs fail validation (negative ECR, zero exposure, curve too short).
    """
```

### 8.2 Result Object

```python
class RiskMarginResult(BaseModel):
    """Immutable result of Risk Margin calculation."""
    risk_margin: float
    technical_provision: float  # BEL + RM
    bel: float                  # Echo back for audit trail
    ecr_at_t0: float
    coc_rate: float
    method: RiskMarginMethod
    projection_years: int       # T
    discounted_ecr_sum: float   # Sum before applying CoC
    detail_df: pd.DataFrame     # Period-by-period breakdown (Section 5.4)

    model_config = ConfigDict(frozen=True, arbitrary_types_allowed=True)
```

### 8.3 Full Pseudocode (Proportional Method)

```python
@rule_ref("Subpara 36(4), Handbook D38.5", "Risk Margin: CoC method")
def calculate_risk_margin(ecr_at_t0, liability_cfs, base_curve, bel, config=None):
    # --- 0. Defaults and validation ---
    if config is None:
        config = RiskMarginConfig(ecr_at_t0=ecr_at_t0)

    coc = config.coc_rate
    validate_inputs(ecr_at_t0, liability_cfs, base_curve)

    if ecr_at_t0 == 0.0:
        return RiskMarginResult(risk_margin=0.0, technical_provision=bel, ...)

    T = len(liability_cfs)  # number of projection periods

    # --- 1. Compute runoff factors ---
    total_exposure = np.sum(liability_cfs)
    # reverse_cumsum[i] = sum of liability_cfs[i:]
    reverse_cumsum = np.cumsum(liability_cfs[::-1])[::-1]

    # alpha[t] for t = 0..T
    alpha = np.zeros(T + 1)
    alpha[0] = 1.0
    for t in range(1, T):
        alpha[t] = reverse_cumsum[t] / total_exposure
    alpha[T] = 0.0  # all liabilities paid

    # --- 2. Compute Modified ECR ---
    modified_ecr = ecr_at_t0 * alpha  # shape (T+1,)

    # --- 3. Get discount factors for maturities 1..T+1 ---
    discount_factors = np.array([
        base_curve.discount_factor(t + 1) for t in range(T + 1)
    ])

    # --- 4. Discount Modified ECR ---
    discounted_ecr = modified_ecr * discount_factors  # shape (T+1,)

    # --- 5. Sum and apply CoC ---
    discounted_ecr_sum = np.sum(discounted_ecr)
    risk_margin = coc * discounted_ecr_sum

    # --- 6. Technical Provision ---
    technical_provision = bel + risk_margin

    # --- 7. Build audit DataFrame ---
    detail_df = pd.DataFrame({
        "t": np.arange(T + 1),
        "remaining_exposure": np.concatenate([[total_exposure], reverse_cumsum[1:], [0.0]]),
        "alpha": alpha,
        "modified_ecr": modified_ecr,
        "discount_maturity": np.arange(1, T + 2),
        "discount_factor": discount_factors,
        "discounted_ecr": discounted_ecr,
        "cumulative_sum": np.cumsum(discounted_ecr),
    })

    # --- 8. Return immutable result ---
    return RiskMarginResult(
        risk_margin=risk_margin,
        technical_provision=technical_provision,
        bel=bel,
        ecr_at_t0=ecr_at_t0,
        coc_rate=coc,
        method=RiskMarginMethod.PROPORTIONAL,
        projection_years=T,
        discounted_ecr_sum=discounted_ecr_sum,
        detail_df=detail_df,
    )
```

### 8.4 Component-Level Method Pseudocode

```python
@rule_ref("Subpara 36(4), Handbook D20-D37", "Risk Margin: component-level ECR projection")
def calculate_risk_margin_component(
    bscr_components: dict[str, float],
    correlation_matrix: np.ndarray,
    component_drivers: dict[str, np.ndarray],
    base_curve: BaseCurve,
    bel: float,
    coc_rate: float = 0.06,
) -> RiskMarginResult:
    """
    Component-level ECR projection with correlation-adjusted reaggregation.

    Parameters
    ----------
    bscr_components : dict
        e.g., {"C_fi": 120000, "C_cr": 80000, "C_mort": 50000, ...}
    correlation_matrix : np.ndarray
        BSCR correlation matrix, shape (n_components, n_components)
    component_drivers : dict
        e.g., {"C_fi": array_of_runoff_factors, "C_cr": array_of_runoff_factors, ...}
        Each array has shape (T+1,) with values in [0, 1], starting at 1.0.
    """
    component_names = sorted(bscr_components.keys())
    n = len(component_names)
    T = len(next(iter(component_drivers.values()))) - 1

    modified_ecr = np.zeros(T + 1)
    for t in range(T + 1):
        # Project each component
        charges = np.array([
            bscr_components[name] * component_drivers[name][t]
            for name in component_names
        ])
        # Reaggregate with correlation: sqrt(C^T @ Corr @ C)
        modified_ecr[t] = np.sqrt(charges @ correlation_matrix @ charges)

    # Discount and sum as in proportional method
    discount_factors = np.array([
        base_curve.discount_factor(t + 1) for t in range(T + 1)
    ])
    discounted_ecr_sum = np.dot(modified_ecr, discount_factors)
    risk_margin = coc_rate * discounted_ecr_sum
    # ... build result ...
```

### 8.5 Testing Strategy

| Test Case | Description | Expected |
|---|---|---|
| `test_worked_example` | 5-period example from Section 6 | RM = 79,366 (within rounding tolerance) |
| `test_zero_ecr` | ECR(0) = 0 | RM = 0, TP = BEL |
| `test_negative_ecr` | ECR(0) < 0 | Raises ValueError |
| `test_zero_exposure` | All liability CFs = 0 | Raises ValueError |
| `test_single_period` | One liability payment at T=1 | alpha = [1.0, 0.0], RM = CoC * ECR * DF(1) |
| `test_flat_curve` | All rates = 5% | Verify against hand calculation |
| `test_long_horizon` | T = 100 | Completes in < 1 second, RM > 0 |
| `test_monotonic_alpha` | Arbitrary cash flows | alpha is non-increasing |
| `test_tp_equals_bel_plus_rm` | Any valid input | TP == BEL + RM exactly |
| `test_rm_decreases_with_higher_rates` | Compare flat 3% vs flat 6% | RM(3%) > RM(6%) |
| `test_component_vs_proportional` | Same ECR, uniform drivers | Both methods produce identical RM |
| `test_non_standard_coc` | CoC = 4% | Warning logged, RM = 4% * sum |
| `test_hypothesis_roundtrip` | Hypothesis-generated inputs | RM >= 0, TP >= BEL, alpha monotonic |

### 8.6 Audit Trail

Every call to `calculate_risk_margin` produces a JSON Lines audit entry:

```json
{
  "timestamp": "2024-12-31T14:30:00Z",
  "module": "calculations.risk_margin",
  "function": "calculate_risk_margin",
  "rule_ref": "Subpara 36(4), Handbook D38.5",
  "inputs": {
    "ecr_at_t0": 500000,
    "coc_rate": 0.06,
    "method": "proportional",
    "projection_years": 5,
    "total_liability_exposure": 1000000,
    "bel": 10500000
  },
  "outputs": {
    "risk_margin": 79366.42,
    "technical_provision": 10579366.42,
    "discounted_ecr_sum": 1322773.70
  },
  "run_id": "run_20241231_143000_abc123"
}
```

### 8.7 Performance Considerations

The Risk Margin calculation is computationally trivial compared to the C0 root-finding in ALGO-002. For T=100 periods, it involves:

- One reverse cumulative sum: O(T)
- One element-wise multiply: O(T)
- One dot product: O(T)

Total: O(T) with small constants. No iteration, no optimization, no QuantLib calls. Expected execution time is well under 1 millisecond for any realistic T.

### 8.8 Dependencies

```
calculations/risk_margin.py
    imports: numpy, pandas, pydantic, logging
    from curves: BaseCurve (type only, for discount_factor() interface)
    from rules: rule_ref decorator
    from engine: audit_logger (for JSON Lines output)
```

No QuantLib. No scipy. Pure Python + NumPy + pandas.
