# 14: Regression Testing Framework

Automated testing procedures to detect implementation changes across test runs and deployments.

---

## Regression Testing Overview

**Purpose**: Catch unintended changes to SBA implementation over time

**Mechanism**: 
1. Run tests once → Capture baseline results
2. Run tests periodically (weekly, before release) → Compare to baseline
3. Deviation from baseline → Flag as potential regression
4. Investigate → Determine if intentional optimization or unintended bug

---

## Baseline Capture Procedure

### Step 1: Initial Baseline Run

**When**: After Phase 2 & 3 validations pass  
**What**: Execute all 26 unit tests + 8 integration tests  
**Output**: `baseline_results.xlsx` spreadsheet

### Step 2: Baseline Data Structure

**Spreadsheet Tabs**:

```
Tab 1: "UnitTest_Results"
  ├─ Test ID (e.g., 1.1, 2.5, 3.2)
  ├─ Component (D&D, Goal-Seek, KRD, Rebalancing, Z-Spread)
  ├─ Description
  ├─ Baseline_Expected_Value (e.g., cumulative_default = 0.05)
  ├─ Baseline_Actual_Value (e.g., 0.0500012)
  ├─ Tolerance (e.g., ±0.0001)
  ├─ Test_Pass_Date (when baseline captured)
  ├─ Python_Version_Tested (e.g., 3.9.7)
  ├─ QuantLib_Version (e.g., 1.25)
  ├─ Git_Commit_ID (SBA model version)
  └─ Notes

Tab 2: "IntegrationTest_Results"
  ├─ Test ID (e.g., 1, 2, 3, ... 8)
  ├─ Description
  ├─ Baseline_Duration_Seconds
  ├─ Is_6Month_Dryrun (Flag)
  ├─ Key_Metrics (goal_seek_iterations, rebalancing_iterations, P&L)
  ├─ Test_Pass_Date
  ├─ Environment_Details
  └─ Notes

Tab 3: "EdgeCase_Results"
  ├─ Test ID (E1.1, E2.1, ... E6.3)
  ├─ Scenario
  ├─ Expected_Behavior
  ├─ Baseline_Result_Status (PASS / CAVEAT)
  ├─ Special_Handling (if any)
  ├─ Test_Pass_Date
  └─ Notes

Tab 4: "Environment"
  ├─ Operating System (Windows 10 / Linux / Mac)
  ├─ Python Version (3.9.7)
  ├─ QuantLib Version (1.25)
  ├─ NumPy Version (1.X.X)
  ├─ Pandas Version (1.X.X)
  ├─ SBA Model Version (v0.1.8)
  ├─ Spec Version (v1.0.0)
  ├─ Baseline_Run_Date
  ├─ Baseline_Run_By (person name)
  └─ Notes
```

### Step 3: Manual Baseline Validation

Before committing baseline:
1. [ ] Run all 26 unit tests → Verify all PASS
2. [ ] Run all 8 integration tests → Verify all PASS
3. [ ] Run 20 edge case tests → Verify no crashes
4. [ ] Document environment (Python, QuantLib, OS versions)
5. [ ] Sign off: "This baseline represents correct SBA behavior"

---

## Regression Detection Procedure

### Step 1: Periodic Test Runs (Auto-triggered)

**Frequency**: 
- Morning smoke test (daily)
- Before each deployment
- After major code changes
- Quarterly audit

**Execution**: Run test suite (dryrun mode for speed)

**Output**: Current results file (`current_results.xlsx`)

### Step 2: Comparison Algorithm

**For each test**:
```
current_value = latest_test_output
baseline_value = baseline_results[test_id]
tolerance = tolerance_set_for_test

IF abs(current_value - baseline_value) > tolerance:
    REGRESSION_FLAG = True
    delta = current_value - baseline_value
    delta_pct = delta / baseline_value × 100%
    
    REPORT: "Test [ID]: REGRESSION DETECTED"
            "Baseline: [baseline_value]"
            "Current:  [current_value]"
            "Delta:    [delta] ([delta_pct]%)"
            "Expected: [tolerance]"
ELSE:
    REGRESSION_FLAG = False
    REPORT: "Test [ID]: ✓ PASS (within tolerance)"
```

### Step 3: Regression Report

**Output**: `regression_report_[DATE].csv`

```
Test_ID,Component,Description,Status,Baseline_Value,Current_Value,Delta,Delta_Pct,Tolerance,Exceeds_Tolerance
1.1,D&D,0% Default,PASS,0.00000,0.00000,0.00000,0.0%,±1E-10,No
1.2,D&D,100% Default,PASS,1.00000,1.00000,0.00000,0.0%,±1E-10,No
2.1,Goal-Seek,Linear Convergence,REGRESSION,8.50,8.53,0.03,0.4%,±0.01,Yes
3.1,KRD,Risk-Free Baseline,PASS,-2.5M,-2.50001M,-0.00001M,0.0%,±1bp,No
...
```

### Step 4: Regression Investigation

**If REGRESSION found**:

1. **Severity Assessment**
   - Small deviation (< 5% delta): Likely numerical rounding, acceptable
   - Medium deviation (5-20% delta): Investigate cause
   - Large deviation (> 20% delta): Major change detected, root-cause analysis required

2. **Root Cause Analysis**
   - Check recent code changes (git log)
   - Check parameter changes (Excel config)
   - Check environment changes (Python, QuantLib versions)
   - Check data changes (test data corrupted?)

3. **Decision Matrix**
   - **Intentional Change**: Update baseline, document change reason
   - **Unintended Bug**: Fix code, re-run regression report
   - **Environment Diff**: Re-run on target environment, accept if reproducible

4. **Action & Reporting**
   - Document finding: "Regression in test 2.1 (goal-seek iterations)"
   - Root cause: "Recent refactor of discontinuity detection alogrithm"
   - Resolution: "Code review + re-baseline authorized"
   - Sign-off: "Approved by [person]"

---

## Baseline Update Procedure

**When baseline needs updating**:
1. Intentional algorithm improvement → New baseline more efficient
2. Configuration change → New baseline reflects new settings
3. Specification update → New baseline per new spec

**Update Process**:

```
CURRENT BASELINE:
  └─ baseline_v1.0_[DATE].xlsx

NEW TEST RESULTS (after changes):
  └─ new_results.xlsx

VERIFICATION:
  1. Review all differences
  2. Confirm intentional (not accidental)
  3. Testing passed: ✅ All tests PASS
  4. Document change reason

NEW BASELINE:
  └─ baseline_v1.1_[DATE].xlsx
     (Version incremented)

ARCHIVE:
  └─ baseline_v1.0_[DATE].xlsx (kept for history)

GIT COMMIT:
  "Update test baseline: [reason]"
  "Performance improvement in goal-seek: 8.5 → 8.2 iterations"
  "New baseline v1.1 capturing optimization"
```

---

## Continuous Integration (CI) Integration

### GitHub Actions / GitLab CI Setup

**File**: `.github/workflows/regression_tests.yaml`

```yaml
name: SBA Regression Tests

on:
  push:
    branches: [main, dev]
  schedule:
    - cron: '0 8 * * *'  # Daily 8am

jobs:
  regression_test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v2
      
      - name: Setup Python
        uses: actions/setup-python@v2
        with:
          python-version: 3.9
      
      - name: Install Dependencies
        run: |
          pip install -r requirements.txt
          pip install quantlib
      
      - name: Run Regression Tests
        run: |
          python regression_test_suite.py \
            --baseline baseline_results.xlsx \
            --output current_results.xlsx \
            --mode dryrun
      
      - name: Check for Regressions
        run: |
          python check_regressions.py \
            --baseline baseline_results.xlsx \
            --current current_results.xlsx \
            --tolerance_factor 1.05  # Allow 5% margin
      
      - name: Upload Artifacts
        if: always()
        uses: actions/upload-artifact@v2
        with:
          name: regression-results
          path: |
            current_results.xlsx
            regression_report_*.csv
      
      - name: Notify on Regression
        if: failure()
        uses: actions/github-script@v6
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '⚠️ Regression Test Failed!\n\nSee artifacts for details.'
            })
```

---

## Test Automation Script

**File**: `regression_test_suite.py`

```python
#!/usr/bin/env python3
"""
SBA Regression Test Suite
Runs all unit, integration, and edge case tests
Compares to baseline and generates regression report
"""

import sys
import pandas as pd
from datetime import datetime

class RegressionTestSuite:
    def __init__(self, baseline_file, dryrun=True):
        self.baseline = pd.read_excel(baseline_file)
        self.dryrun = dryrun
        self.results = []
        
    def run_unit_tests(self):
        """Run all 26 component unit tests"""
        tests = [
            ('1.1', self.test_d2d_boundary_0pct),
            ('1.2', self.test_d2d_boundary_100pct),
            # ... all 26 tests
        ]
        
        for test_id, test_func in tests:
            expected, actual, passed = test_func()
            self.results.append({
                'test_id': test_id,
                'expected': expected,
                'actual': actual,
                'passed': passed
            })
    
    def run_integration_tests(self):
        """Run all 8 integration scenarios"""
        scenarios = [
            ('I1', '6-month dryrun', self.test_6month_projection),
            # ... all 8 scenarios
        ]
        
        for test_id, name, test_func in scenarios:
            status, metrics = test_func()
            self.results.append({
                'test_id': test_id,
                'description': name,
                'status': status,
                'metrics': metrics
            })
    
    def compare_to_baseline(self):
        """Compare current results to baseline"""
        regressions = []
        
        for result in self.results:
            baseline_row = self.baseline[
                self.baseline['Test_ID'] == result['test_id']
            ]
            
            if baseline_row.empty:
                continue
                
            baseline_value = baseline_row['Baseline_Actual_Value'].values[0]
            tolerance = baseline_row['Tolerance'].values[0]
            
            current_value = result['actual']
            delta = abs(current_value - baseline_value)
            
            if delta > tolerance:
                regressions.append({
                    'test_id': result['test_id'],
                    'baseline': baseline_value,
                    'current': current_value,
                    'delta': delta,
                    'tolerance': tolerance,
                    'severity': 'HIGH' if delta > tolerance*2 else 'MEDIUM'
                })
        
        return regressions
    
    def generate_report(self, output_file):
        """Generate regression report"""
        report_df = pd.DataFrame(self.results)
        report_df.to_excel(output_file, index=False)
        
        regressions = self.compare_to_baseline()
        if regressions:
            print(f"⚠️  REGRESSIONS DETECTED: {len(regressions)}")
            for r in regressions:
                print(f"  {r['test_id']}: {r['delta']:.4f} "
                      f"(tolerance: {r['tolerance']:.4f}) "
                      f"[{r['severity']}]")
            return False
        else:
            print("✅ All regression tests PASSED")
            return True

if __name__ == '__main__':
    suite = RegressionTestSuite('baseline_results.xlsx', dryrun=True)
    suite.run_unit_tests()
    suite.run_integration_tests()
    suite.generate_report('current_results.xlsx')
```

---

## Regression Types & Responses

| Regression Type | Delta | Cause | Response |
|-----------------|-------|-------|----------|
| **Floating-point rounding** | < 0.001% | Numerical precision | Accept, no action |
| **Configuration drift** | 0.01-1% | Parameter changed in Excel | Investigate parameter change |
| **Environment change** | 1-5% | Python/QuantLib version | Test on baseline environment |
| **Algorithm optimization** | 5-20% | Intentional code refactor | Review change, re-baseline if approved |
| **Unintended bug** | > 20% | Code error introduced | Root-cause analysis, fix, re-test |

---

## Quarterly Regression Audit

**Schedule**: Every 3 months  
**Duration**: ~2 hours

**Procedure**:
1. Run full test suite (no dryrun, 101-year projections)
2. Compare all results to baseline
3. Generate comprehensive audit report
4. Present findings to team
5. Approve baseline updates (if any)
6. Archive audit results

**Deliverable**: `SBA_Regression_Audit_Q[N]_[YEAR].pdf`

---

## Baseline Maintenance

### Version Control

**Location**: Git repository
```
/tests/
  ├─ baseline_v1.0_20240101.xlsx (Initial baseline)
  ├─ baseline_v1.1_20240315.xlsx (After goal-seek optimization)
  ├─ baseline_v1.2_20240630.xlsx (After spec update)
  └─ baseline_current.xlsx (Symlink to latest)
```

### Versioning Scheme

```
baseline_v[MAJOR].[MINOR]_[DATE].xlsx

MAJOR: Changed when major algorithm changes (e.g., goal-seek refactor)
MINOR: Changed when minor optimization (e.g., tolerance adjustment)
DATE: YYYYMMDD when baseline established
```

### Retention Policy

- Keep all baseline versions (history for audits)
- Current + 2 previous as "active" (for comparison)
- Archive older versions (offline storage)
- Never delete (audit trail)

---

## Regression Acceptance Criteria

**Test passes regression if**:
- ✅ Within baseline tolerance
- ✅ OR intentional optimization (documented, approved)
- ✅ OR environment difference explained (Python version, etc.)

**Test fails if**:
- ❌ Outside tolerance without explanation
- ❌ Unintended code change not approved
- ❌ Data corruption suspected

---

## Example Regression Report Output

```
═══════════════════════════════════════════════════════════════════
SBA REGRESSION TESTING REPORT
Generated: 2024-03-24 15:32:00
Baseline Version: v1.1 (2024-03-15)
Current Run Date: 2024-03-24
Environment: Python 3.9.7, QuantLib 1.25, Ubuntu 20.04
═══════════════════════════════════════════════════════════════════

SUMMARY:
  Total Tests Run: 54 (26 unit + 8 integration + 20 edge cases)
  Passed: 52
  Failed/Regression: 2
  Caveat/Warning: 1 (callable bond validation incomplete)
  
REGRESSION DETAILS:

  ⚠️  Test 2.1 (Goal-Seek Linear Convergence)
      Baseline: 8.5 iterations
      Current:  8.7 iterations
      Delta:    +0.2 iterations (+2.4%)
      Tolerance: ±0.5 iterations
      Status:    Within tolerance ✓
      Likely cause: Minor numerical variation

  🔴 Test 4.3 (Rebalancing DV01 Convergence)
      Baseline: DV01 mismatch = $150k
      Current:  DV01 mismatch = $245k
      Delta:    +$95k (+63%)
      Tolerance: ±$50k
      Status:    EXCEEDS TOLERANCE ❌
      Action required: Investigate rebalancing algorithm
      
INVESTIGATION NOTES:
  - Code change detected: swap_rebalancer.py lines 170-200 refactored
  - Review found: Loop exit condition changed
  - Resolution: Revert change, return to v1.1 logic
  - Re-test: Scheduled for 2024-03-24 16:00

CAVEAT:
  - Test E3.3 (Callable bond): Marked as caveat (spec validation incomplete)
  - No regression action required (known limitation)

═══════════════════════════════════════════════════════════════════
CONCLUSION: 1 active regression (Test 4.3) requires investigation
ACTION: Do not merge changes until resolved
═══════════════════════════════════════════════════════════════════
```

---

## Success Metrics for Regression Framework

- ✅ Baseline successfully captured (all 54 tests passing)
- ✅ Regression detection functioning (catches intentional + accidental changes)
- ✅ CI/CD integration working (auto-runs on commits)
- ✅ Reports generated accurately (no false positives/negatives)
- ✅ Investigation guide used (issues resolved efficiently)
- ✅ Zero production regressions (if framework working)

