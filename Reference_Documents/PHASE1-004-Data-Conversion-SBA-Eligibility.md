# 03-Data-Conversion-SBA-Eligibility.md

# DATA CONVERSION & SBA ELIGIBILITY MODULE

**Files**: 
- [data_2_finalise_data.py](../../Athora_BEL_Model_v0.1.8/site-packages/alm/data_conversion/data_2_finalise_data.py) - Lines 613-635 (SBA eligibility)
- [data_1_convert_UBS_Delta.py](../../Athora_BEL_Model_v0.1.8/site-packages/alm/data_conversion/data_1_convert_UBS_Delta.py)

**Purpose**: Transform raw data (UBS Delta format) → standardized format → final SBA-eligible asset file.

## Pipeline

INPUT → convert1 → INTERMEDIATE → convert2 → FINAL ASSET FILE

## SBA Eligibility Decision Tree

EXCLUDE IF:
- Business unit doesn't match
- Asset class in [Derivatives, Collateral-VM, Currency Derivatives, Equity]
- Rating < BBB- (minimum)
- Model type not supported
- Data quality issues (NULL notional, maturity, rates)

INCLUDE IF:
- All filters passed
- Bond, cash, or supported derivative
- Complete market data

## Verification Checklist

- [ ] Asset type classification correct
- [ ] Rating filter enforced (BBB- minimum)
- [ ] Z-spread weighting excludes derivatives
- [ ] Output files reconcile source → final
