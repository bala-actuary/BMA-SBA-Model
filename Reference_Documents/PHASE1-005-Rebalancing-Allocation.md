# 04-Rebalancing-Allocation.md

# REBALANCING & ASSET ALLOCATION MODULE

**Files**: 
- [alm_rebalancing.py](../../Athora_BEL_Model_v0.1.8/site-packages/alm/model/projection/alm_rebalancing.py)
- [asset_allocator.py](../../Athora_BEL_Model_v0.1.8/site-packages/alm/model/projection/asset_allocator.py)

**Purpose**: Maintain asset/liability balance through portfolio rebalancing constrained by allocations (SAA/CAA/Most Onerous).

---

## Core Concept

**Goal**: If actual allocations drift from targets, trade assets to rebalance back to target.

```
Current Asset Allocation (CAA, market-value weighted):
  Corporate: 45% (drifted up)
  Sovereign: 25% (drifted down)
  Cash:      30% (target)

Strategic Asset Allocation (SAA, target):
  Corporate: 40%
  Sovereign: 30%
  Cash:      30%

Action: SELL €2.5M corporate (45%→40%), BUY €2.5M sovereign (25%→30%)
```

---

## Allocation Strategies

### SAA (Strategic Asset Allocation)
- **Source**: Lookup table in asset_parameter_provider
- **Nature**: Static targets (set annually or less frequently)
- **Example**: {Corporate: 0.40, Sovereign: 0.30, Cash: 0.30}

###CAA (Current Asset Allocation)
- **Source**: Calculated from actual balance sheet market values
- **Nature**: Dynamic (changes with market moves)
- **Example**: {Corporate: 0.45, Sovereign: 0.25, Cash: 0.30}

### Most Onerous
- **Selection**: min(SAA, CAA) in terms of weighted average spread
- **Rationale**: Use more conservative allocation (tighter spreads)
- **Example**: If SAA_spread=1.5%, CAA_spread=1.8% → use CAA (Most Onerous)

---

## Two-Phase Rebalancing Algorithm

```
PHASE 1: SELL (from over-weighted positions)
  
  ├─ Calculate drift: actual% - target%
  ├─ Identify over-weighted classes (drift > 0)
  ├─ SELL assets from over-weighted classes
  ├─ Proceeds → cash account
  └─ PnL realized (profit/loss on sale)

PHASE 2: BUY (reinvest in under-weighted positions)
  
  ├─ Identify under-weighted classes (drift < 0)
  ├─ Use stock assets (reinvestment templates)
  ├─ BUY amounts to restore target allocation
  ├─ Cash ← proceeds
  └─ Record costof trading (bid/offer spreads)
```

### Optimization Problem (scipy.optimize.minimize)

```
Minimize:  Σ((target_rate_i - desired_rate_i)^2)

Subject To:  Σ(trade_i) = 0  (net proceeds = 0)
             bounds[i].lb ≤ trade[i] ≤ bounds[i].ub
             trade[i] ∈ {-portfolio_i, ..., +cash_available}

Solver: SLSQP (Sequential Least Squares Programming)

Output: Trades for each asset class (buy/sell amounts)
```

---

## Asset Scalar Strategies

**ReduceAllAssetsEqually**: All assets in class × (1 - rebalance_demand)

**ReduceCashToSAAFirst**: Reduce cash first, maintaining SAA for non-cash

---

## Rebalancing Events

```
@dataclass
class RebalancingEvent:
    t: int                          # Time period
    event_type: ['MATURE', 'SELL', 'BUY']
    model_type: str                 # Bond, Swap, etc.
    id: int                         # Asset ID
    description: str                # Asset name
    allocation_name: str            # Asset class
    notional: float                 # Amount traded
    market_value: float             # MV at trade
    dv01: float                     # DV01 impact
```

---

## Derivative Rebalancing: IR Swaps (KRD Matching)

### ⚠️ CRITICAL: Liability KRD Calculation Uses Risk-Free Rate ONLY

**SPECIFICATION REQUIREMENT** (Spec Section 2.3.5.3 & 3.3.8.6):
- Liability KRDs are calculated by discounting remaining cashflows at the **risk-free rate ONLY**
- **NO SBA spread** is applied when calculating liability KRDs or sensitivities
- This is distinct from the final SBA BEL calculation which uses (RF + SBA spread)
- Asset KRDs may reflect spread changes (asset class spreads evolve during projection)
- Rebalancing ensures asset KRDs track liability KRDs as measured at risk-free rates

**Why This Matters**:
- Sets the baseline for derivative rebalancing targets
- Impacts swap notionals and hedge effectiveness
- **MUST VERIFY IN CODE**: Confirm liability KRD calculation excludes SBA spread

### Core Strategy

Maintain the **Key Rate Duration (KRD)** sensitivity match between assets and liabilities across 5 key tenors.

### KRD Calculation (Both Assets & Liabilities)

**Approach**: Shock yield curve at specific tenors, measure sensitivity change:

$KRD_k = \Delta MV(tenor\_shock\_at\_k) / 0.0001$

Where:
- k = tenor: 5Y, 10Y, 15Y, 20Y, 30Y (5 key rate durations)
- tenor_shock_at_k = +1bp shock at tenor k (shape-preserving)
- ΔMV = Change in market value due to shock
- Result: DV01 sensitivity per key rate duration

**For Liabilities** (⚠️ CRITICAL):
- Use risk-free rate ONLY (no SBA spread)
- Calculate KRD per remaining liability CF schedule
- Per-period recalculation as CFs decay

**For Assets**:
- Calculate KRD per asset type (Bonds, Swaps, etc.)
- Per-period recalculation as portfolio changes

### Rebalancing Entry (Derivative Rebalancing Phase)

```
TARGET KRD SETTING:
  asset_KRD_ratio[k] = asset_KRD[k] / liability_KRD[k]
  
REBALANCING LOOP (for each tenor k: 30Y, 20Y, 15Y, 10Y, 5Y):
  
  current_delta[k] = asset_KRD[k] - target_ratio[k] * liability_KRD[k]
  
  IF current_delta[k] > 0 (assets more sensitive):
    → Buy RECEIVER swap (pay fixed, receive floating)
    → Swap notional = current_delta[k] / swap_DV01[k]
  
  ELSE IF current_delta[k] < 0 (assets less sensitive):
    → Buy PAYER swap (pay floating, receive fixed)
    → Swap notional = |current_delta[k]| / swap_DV01[k]
  
  ELSE:
    → No swap needed (already matched)

COLLATERALIZATION:
  → All swaps fully cash-collateralized
  → Margin call = |swap_MV| → deducted from cash
```

### Key Properties

1. **Swaps never sold**: Only purchased (can offset via new swaps in opposite direction)
2. **Zero market value at issue**: Swaps created at par
3. **Bid-offer spread applied**: Capitalized as t0 cash outflow
4. **Interactive with physical rebalancing**: Steps 5-6 in projection iterate up to 3 times

---

## Duration Constraints

```
max_reinvest_duration = min(
  liability_duration * 1.30,
  ap.reinvestment_asset_max_duration_relative
)

IF asset.duration > max_reinvest_duration:
  PREVENT from being purchased (even if underweight)
```

---

## ConfigurationParameters

| Parameter | Type | Default | Purpose |
|-----------|------|---------|---------|
| rebalance_enable | bool | False | Enable rebalancing? |
| rebalance_taa_source | enum | SAA | Which allocation to target |
| rebalancing_allowance_to_meet_taa | float | 0.02 | Tolerance (2%) |
| reinvestment...max_duration | float | 1.30 | DurationMultiplier |

---

## Verification Checklist

- [ ] SAA/CAA correctly loaded
- [ ] Rebalancing triggered at configured periods
- [ ] Optimization converges (scipy success)
- [ ] Cash positive after buy phase
- [ ] Duration limits enforced
- [ ] Rebalancing events logged

---

## References

**Code**: [alm_rebalancing.py](../../Athora_BEL_Model_v0.1.8/site-packages/alm/model/projection/alm_rebalancing.py)
**User Guide**: Section 2.2 ("Rebalancing Strategy")
