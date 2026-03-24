# 07-CLI-Execution-Framework.md

# CLI & EXECUTION FRAMEWORK

**Files**:
- [alm/main.py](../../Athora_BEL_Model_v0.1.8/site-packages/alm/main.py) - CLI router
- [main_server.py](../../Athora_BEL_Model_v0.1.8/site-packages/alm/model/main_server.py) - Multi-worker orchestration
- [main_client.py](../../Athora_BEL_Model_v0.1.8/site-packages/alm/model/main_client.py) - Single-worker execution
- [server_state_machine.py](../../Athora_BEL_Model_v0.1.8/site-packages/alm/model/server_state_machine.py) - Job state tracking

**Purpose**: Provide CLI interface and coordinate parallel execution across scenarios/business units.

---

## CLI Commands

### convert1: Raw Data Standardization
```bash
python -m alm convert1 setup.xlsm

Purpose: UBS Delta CSV → Standardised Excel format

Input:
  - setup.xlsm (file paths, metadata)
  - CSV files in 03. Assets/02. Raw Data/

Output:
  - {bu}_Standardised_{date}.xlsx
  - All standard columns, asset types classified

Flow:
  read_setup_excel()
  FOR each CSV file:
    classify_asset_types()
    apply_loadingfunctions()  # FixedRateBond(), Swap(), etc.
    output to Excel
```

### convert2: SBA Eligibility & Finalization
```bash
python -m alm convert2 setup.xlsm

Purpose: Apply SBA eligibility rules, create z-spread weights

Input:
  - {bu}_Standardised_*.xlsx from convert1

Output:
  - {bu}_Final_{date}.xlsx with:
    ├─ SBA_eligible flag (boolean)
    ├─ ineligible_reason (if excluded)
    ├─ _zspread_weight_ column
    ├─ zspread_weighted column
    └─ Summary sheets (audit trail)

Flow:
  load_standardised_files()
  FOR each asset:
    apply_eligibility_rules()  # Rating, asset class, etc.
    calculate_zspread_weights()
  output_final_file_with_audit()
```

### run-server: Multi-Worker Orchestration
```bash
python -m alm run-server \
  --runs runs.txt \
  --log output.log \
  --override param1=val1 param2=val2 \
  --lower-priority \
  --queue \
  --dryrun

Purpose: Execute all BUs/scenarios in parallel

Input:
  - runs.txt: pipe-delimited run specifications
    setup.xlsm|run_name|BU1,BU2,BU3
  - --override: parameter value overrides

Options:
  --log FILE                Write to FILE (vs stdout)
  --queue                   Wait for lock (exclusive execution)
  --nogui                   Disable progress GUI
  --autoclose               Close GUI when done
  --quiet                   Suppress output
  --lower-priority          Run workers below normal priority
  --dryrun                  1 year term, no goal seek

Output:
  - Results_{run_name}_{scenario}_{bu}.xlsx
  - logs/{run_name}/execution.log

Architecture:
  Main thread:
    ├─ load_program_state(runs.txt)
    ├─ ProcessPool(max_workers=CPU_count-1)
    ├─ Spawn workers for each (BU, Scenario) pair
    ├─ Monitor state machine
    └─ Merge results when all complete
  
  Worker subprocesses (1..8):
    ├─ run_client(SingleInstanceVariables)
    ├─ Project single scenario
    ├─ Return ProjectionResults
    └─ Exit
```

### run-client: Single-Worker Execution
```bash
python -m alm run-client \
  --setup setup.xlsm \
  --run run_name \
  --bu ANL \
  --scenario SBA_0_Base \
  --output ./output/ \
  --override param=val

Purpose: Execute ONE scenario (typically called by run-server)

Flow:
  1. initialise_settings()
  2. initialise_model()
  3. calibrate_model()
  4. project_liability()  # Main ALM projection
  5. write_results()
  6. Exit

Time: ~10-20 seconds per scenario
Memory: ~500 MB per worker

Error Handling:
  ├─ Log all exceptions with stack trace
  ├─ Write partial results if possible
  ├─ Signal completion status to parent process
  └─ Parent retries or marks as failed
```

### merge: Consolidate Summary Files
```bash
python -m alm merge output_folder/ summary.xlsx

Purpose: Combine all results from recursively searched folder

Input:
  output_folder/
  ├─ 2024-01-15/
  │   ├─ run_00_Base/
  │   │   ├─ Results_00_Base_ANL.xlsx
  │   │   └─ Results_00_Base_ABN.xlsx
  │   └─ run_01_NoDerivatives/
  │       └─ Results_01_NoDerivatives_ANL.xlsx
  └─ summary.xlsx  (output)

Output:
  summary.xlsx with:
    ├─ Master sheet (all scenarios, all BUs, key metrics)
    ├─ Scenario comparison chart
    └─ Convergence metrics
```

### extract-asset-values: Asset Value Extraction
```bash
python -m alm extract-asset-values \
  --asset-data input_assets.csv \
  --model-outputs output_folder/ \
  --destination enriched_assets.xlsx

Purpose: Join model outputs back to original asset file

Input:
  - Asset identifiers (ISIN, internal ID)
  - Model results (market values, DV01, stresses)

Output:
  - Original asset data enriched with model outputs
```

---

## Server State Machine

### ScenarioStep Enum
```
Step_0_NotStarted (initial)
  ├─ Load configuration
  └─ Prepare balance sheet

Step_1_ValueAssets (optional)
  ├─ IF value_all_assets_on_balance_sheet = TRUE
  ├─ Project assets alone (liability at 0)
  └─ Used for valuation of fixed income portfolio

Step_2_ValuingLiability1 (main)
  ├─ ALM projection at 100% liability
  ├─ First convergence attempt
  └─ Goal seek starts here

Step_3_ValuingLiability2 (optional SBA minimisation)
  ├─ IF sba_minimise_liability_proportion_across_scenarios
  ├─ Re-run at found minimum proportion
  └─ Used for group-level optimization

Step_4_WritingResults
  ├─ Export to Excel
  ├─ Write summary files
  └─ Logging

Step_5_CompleteOk|Error|Idle (final)
  └─ Mark scenario done
```

### StateTransition Diagram
```
Step_0 → Step_1? (value_all_assets?)
             ├─ Yes → Step_1 → Step_2
             └─ No  → Step_2

Step_2 → Step_3? (sba_minimise & min scenario?)
             ├─ Yes → Step_3 → Step_4
             └─ No  → Step_4

Step_4 → Step_5 (Complete)
```

---

## ProcessPool Architecture

```python
class ProcessPool:
    max_workers: int = CPU_count - 1  # e.g., 7 on 8-core machine
    job_queue: PriorityQueue  # Queued jobs
    workers: list[Process]    # Active subprocess workers
    
    def submit_work_async(job: Job) -> None:
        # Queue job unless already at max_workers
        IF num_active < max_workers:
          spawn_subprocess(job)
        ELSE:
          job_queue.append(job)
    
    def tick(self) -> None:
        # Called from main loop (~1/sec)
        ├─ Check for completed workers
        ├─ If worker done: collect results
        ├─ If free slot: spawn next queued job
        └─ Return control to main thread
```

---

## Worker Execution Flow

```
run_client(siv: SingleInstanceVariables):
  
  ├─ STEP 1: Initialise Settings (once per process)
  │   IF not settings_cached:
  │     load_settings(siv)
  │     → Settings with all params from Excel
  │
  ├─ STEP 2: Initialise Model (once per process)  
  │   IF not model_cached:
  │     ├─ Create AssetParameterProvider
  │     ├─ Load assets from Excel
  │     ├─ Create MarketProjection
  │     ├─ Create AssetProjector (QuantLib)
  │     ├─ Create ZSpreadProvider
  │     ├─ Create RebalancingStrategy
  │     ├─ Create BalanceSheetProjection
  │     └─ Create SBAGoalSeeker
  │
  ├─ STEP 3: Calibrate Model (once per process)
  │   ├─ calibrate_all_asset_values()
  │   ├─ apply_t0_stresses()
  │   └─ alm_rebalancing.calibrate()
  │
  ├─ STEP 4: Run Projection (main workhorse)
  │   IF operation == 'project_liability':
  │     results = sba_goal_seeker.perform_alm()
  │     RETURN ProjectionResultsLiability
  │
  └─ STEP 5: Write Results
      write_results(results)
      →  Results_{scenario}_{bu}.xlsx
```

---

## Error Handling & Logging

```
TRY/CATCH HIERARCHY:

run_server (main process)
  └─ ProcessPool.tick()
      FOR each worker:
        IF worker.has_error():
          ├─ Read exception
          ├─ Log stack trace
          ├─ Mark job as failed
          └─ Continue with next job

run_client (worker subprocess)
  ├─ TRY initialise_settings()
  │   CATCH → JobState = ERROR
  ├─ TRY initialise_model()
  │   CATCH → JobState = ERROR
  ├─ TRY project_liability()
  │   CATCH → Log warning, continue
  └─ TRY write_results()
      CATCH → JobState = ERROR

CRITICAL ERRORS (Halt):
  ├─ Asset file not found
  ├─ Excel file corrupted
  ├─ Negative asset valuation (model bug)
  ├─ NaN detected (numerical error)
  └─ Liability positive (bad data)

RECOVERABLE (Log & Continue):
  ├─ Goal seek didn't converge (use best attempt)
  ├─ Rebalancing failed (skip this period)
  ├─ Swap rebalancer diverged (skip swaps)
  └─ Cache mismatch (re-run projection)
```

---

## CLI Workflow Example

```bash
# STEP 1: Prepare data
python -m alm convert1 setup.xlsm
  → Reads UBS Delta CSV files
  → Outputs {bu}_Standardised_2023Q4.xlsx

python -m alm convert2 setup.xlsm
  → Reads standardised files
  → Applies SBA eligibility rules
  → Outputs {bu}_Final_2023Q4.xlsx with z-spread columns

# STEP 2: Run model
python -m alm run-server --runs runs.txt --queue --autoclose
  → Queues all scenarios
  → Spawns 7 worker processes
  → Waits for all completion
  → Merges results
  → Closes GUI

# STEP 3: Aggregate
python -m alm merge ./output combined_results.xlsx
  → Reads all Result_*.xlsx files
  → Consolidates into master summary
```

---

## Performance Tuning

| Knob | Default | Fast | Slow |
|------|---------|------|------|
| projection_term_years | 100 | 10 | 150 |
| goal_seek_perform | TRUE | FALSE | (slow) |
| enable_cache | TRUE | TRUE | FALSE |
| rebalance_enable | FALSE | FALSE | TRUE |
| num_workers | CPU-1 | CPU-1 | 1 |
| dryrun | FALSE | TRUE | FALSE |

**Dryrun Example** (validation):
```
projection_term_years: 1
goal_seek_perform: FALSE
enable_cache: FALSE
→ Execution: 0.5 seconds (vs 40 sec normal)
Purpose: Verify setup without multi-minute wait
```

---

## Typical Run Times

| Configuration | Time |
|---------------|------|
| Single scenario (no goal seek, 1 year) | 0.5 sec |
| Single scenario (normal, 100 years) | 10 sec |
| Goal seek (8 iterations × 10 sec) | 80 sec |
| 8 scenarios sequential | 640 sec (11 min) |
| 8 scenarios parallel (8 CPUs) | 80 sec (1.3 min) |

---

## Verification Checklist

- [ ] CLI commands work (convert1, convert2, run-server, etc.)
- [ ] Worker processes spawn and run in parallel
- [ ] State machine transitions correctly
- [ ] Results written to expected folder structure
- [ ] Error messages logged with context
- [ ] Partial results recoverable if crash occurs
- [ ] Memory stays < 4GB (8 workers × 500MB)
- [ ] Execution log shows all steps completed

---

## References

**Code**: 
  - [alm/main.py](../../Athora_BEL_Model_v0.1.8/site-packages/alm/main.py) - CLI entry
  - [main_server.py](../../Athora_BEL_Model_v0.1.8/site-packages/alm/model/main_server.py) - Orchestration
  - [main_client.py](../../Athora_BEL_Model_v0.1.8/site-packages/alm/model/main_client.py) - Worker

**User Guide**: Section 1 ("Running the Model")
