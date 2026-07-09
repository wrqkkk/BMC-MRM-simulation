# Regime 1–4 Constraint-Based Bayesian Network Simulation

**Technical Design and Execution Plan — Version 1**

Repository: `wrqkkk/BMC-MRM-simulation`

Status: Design specification before code implementation

Reference implementation: `simulation0618_regime1.ipynb`

---

## Document purpose

This document defines the first technical specification for the new Regime 1–4 simulation experiments. The implementation will preserve the engineering structure and parameter naming conventions of the existing `simulation0618_regime1.ipynb`, while replacing the old five-tier true-DAG and tier-based constraint logic with four new role-based DAG regimes.

The project is a **constraint-centered Bayesian network simulation study**. The primary research question is not whether one newly proposed DAG-learning algorithm outperforms another algorithm. Instead, the study asks how different levels and correctness of prior structural knowledge affect structure recovery in a classical score-based Bayesian network framework. The central comparison is therefore among `none`, `weak`, `strong`, and `current_misspecified` constraints, with graphical lasso retained as the undirected skeleton benchmark where appropriate.

The new experiments continue to use the existing discrete-BN learning pipeline. In the reference notebook, each job generates one true DAG, generates one numerical dataset from that DAG, converts it once to `dat_disc` through `discretize_for_bn()`, and then passes the same `dat_disc` to `bnlearn::boot.strength()` with `algorithm = "hc"`, `score = "bde"`, and `iss = 10`. The new implementation must preserve this computational structure unless a later design decision explicitly changes the data-generation model.

---

## Scientific objective

The study evaluates the **benefit, cost, and misspecification risk of domain-informed structural constraints** in Bayesian network learning.

A correct constraint may reduce the search space, suppress biologically or temporally implausible reverse edges, increase precision, improve directed structure recovery, and produce a more interpretable network. The same restriction may also reduce recall if it is too strong. A misspecified constraint may wrongly exclude true edges and force the algorithm toward alternative incorrect paths. Therefore, the project must not only report which method has the highest F1 score; it must quantify where the benefit comes from, what is sacrificed, and which types of edges are most vulnerable to incorrect prior knowledge.

The central output is a paired comparison. For a fixed job, all constraint levels must share the same true DAG, the same generated dataset, the same discretized `dat_disc`, and the same job-level simulation configuration. The intended difference between the conditions is the constraint passed to the BN learner.

---

## Variable blocks and node counts

The new DAG system uses role-based blocks rather than the previous fixed five-tier structure.

| Block | Number of nodes | Scientific role |
|---|---:|---|
| `A` | 5 | Exposure block; the environmental exposure variables of primary interest |
| `B` | 5 | Known strong outcome-related competing predictors |
| `C` | 5 | Other intermediate or auxiliary variables |
| `D` | 5 | Optional additional direct-risk block |
| `Y` | 1 | Outcome node |

The node names will be machine-readable, for example `A1`–`A5`, `B1`–`B5`, `C1`–`C5`, `D1`–`D5`, and `Y`.

Under the current four-regime definition, Regime 1 and Regime 3 contain `A`, `B`, `C`, and `Y`, giving `p = 16`. Regime 2 and Regime 4 additionally contain `D`, giving `p = 21`.

The implementation must keep the existing parameter name `p`; it must not invent a replacement such as `n_nodes` or `node_count`. The old editable parameter name `P_NODES` should also be preserved where a single-regime run uses a fixed node count. In a multi-regime experiment grid, the `p` stored in each experiment row and job row must reflect the active regime.

---

## Four true-DAG regimes

### Regime 1: primary parallel-upstream setting

The main scientific setting is:

```text
A, B -> C -> Y
A -> Y
B -> Y
C -> Y
```

At block level, the legal core relations are `A -> C`, `B -> C`, `A -> Y`, `B -> Y`, and `C -> Y`.

`A` and `B` are parallel upstream blocks. The true-DAG template does not assume `A -> B` or `B -> A`. This is the primary setting because `B` represents strong competing predictors of the outcome rather than variables that must be caused by the exposure block `A`.

### Regime 2: primary setting with an additional direct-risk block

The structure is:

```text
A, B -> C -> Y
A -> Y
B -> Y
C -> Y
D -> Y
```

Regime 2 extends Regime 1 by adding the direct-risk block `D`. The defining role of `D` is to contribute additional direct effects on `Y` without being part of the main `A/B -> C -> Y` mechanism.

### Regime 3: chain alternative

The structure is:

```text
A -> B -> C -> Y
A -> Y
B -> Y
C -> Y
```

This regime represents a sequential mechanism in which the exposure block `A` can influence `B`, which can influence `C`, before reaching `Y`. Direct effects from `A`, `B`, and `C` to `Y` are also permitted.

### Regime 4: chain alternative with an additional direct-risk block

The structure is:

```text
A -> B -> C -> Y
A -> Y
B -> Y
C -> Y
D -> Y
```

This is the chain setting with the same additional direct-risk block used in Regime 2.

---

## Within-block edges

All four regimes allow edges inside the same block. This requirement is fundamental and must be preserved in both the true-DAG generator and the strong-constraint definition.

Examples include `A2 -> A4`, `B1 -> B5`, `C3 -> C4`, or `D1 -> D3`. Such edges model dependence structure within exposure variables, competing predictors, intermediate variables, or direct-risk variables.

The true-DAG generator must guarantee acyclicity. A safe implementation is to generate or assign a local topological order within each block and only add within-block edges that respect that order. The current helper logic for acyclicity checking should be retained or adapted rather than removed.

The existing parameter name `WITHIN_TRUE_RATE` must be preserved. In the current notebook it controls the target proportion of within-tier true edges. In the new experiments, it will control the target proportion of within-block true edges. The name must not be changed merely because the underlying blocks have changed.

The strong constraint must **not** prohibit all within-block edges. The point of the strong condition is to forbid scientifically incompatible between-block directions while leaving valid within-block structure searchable.

---

## True-DAG generation strategy

The new `make_true_dag()` should preserve the reference notebook's general strategy: fixed core structure plus random extra edges, target edge count controlled by `edge_multiplier`, deterministic random seeds, explicit acyclicity validation, and one saved true-edge table per job.

The new generator should proceed conceptually as follows. It first creates the active nodes and block annotations for the selected regime. It then loads the regime-specific legal between-block directions. A shared set of regime-specific core edges is inserted so that all replicates of the same regime have a common interpretable backbone. Random within-block and legal between-block extra edges are then added until the target edge count is reached, subject to acyclicity.

The existing target-edge convention must be preserved:

```text
target_edges = p * edge_multiplier
```

The editable parameter remains `EDGE_MULTS`, the job-table column remains `edge_multiplier`, and the computed field remains `target_edges`.

The old experiment used fixed core edges shared across replicates and random extra edges around the core. The new Regime 1–4 implementation should retain this property. It is important for consensus analysis because it creates a meaningful set of true edges that are common across `rep = 1:100` within a regime, while still allowing replicate-specific graph variation.

Every saved true edge should carry enough annotation for later stratified metrics and provenance. At minimum, the edge table should include `from`, `to`, `from_block`, `to_block`, `edge_role`, `edge_origin`, `is_within_layer`, and `is_between_layer`. Existing fields should be retained where useful for backward compatibility.

---

## Parameter naming policy

Existing parameters must retain their current names. The implementation should not introduce renamed aliases for concepts that already exist.

| Existing editable parameter | Meaning in the new experiments |
|---|---|
| `N_VALUES` | Sample sizes included in the experiment grid |
| `P_NODES` | Node count for a fixed-regime run; the corresponding job field remains `p` |
| `EDGE_MULTS` | Multipliers controlling `target_edges = p * edge_multiplier` |
| `RHO_VALUES` | Existing `rho` scenarios; retain the same name and current interpretation in data generation |
| `REP_VALUES` | Independent simulation replicates / true-DAG jobs |
| `B_BOOT_VALUE` | Number of bootstrap resamples used by `boot.strength()` |
| `THRESHOLDS` | Bootstrap edge-strength thresholds; output rows continue to use the column `tau` |
| `MISSPEC_RATE` | Proportion of true edges wrongly blacklisted in `current_misspecified` |
| `WITHIN_TRUE_RATE` | Target proportion of within-block true edges |
| `WL_MISSING_N` | Existing whitelist-misspecification control |
| `WL_EXTRA_N` | Existing whitelist-misspecification control |
| `WL_REVERSED_N` | Existing whitelist-misspecification control |
| `DIRECTION_THRESHOLD` | Minimum majority direction probability required after bootstrap averaging |

The only genuinely new concept needed by the four-regime framework is a regime identifier such as `regime_type`. Existing parameters must not be renamed to match new prose terminology.

---

## Job-table architecture and reproducibility

The existing job-table architecture should be retained.

`make_job_config()` constructs a complete job configuration from one experiment-grid row and the values of `n`, `edge_multiplier`, `rho`, and `rep`. The current deterministic seed formula is:

```text
seed_dag = SEED_BASE + rep + 1000*n + 100*edge_multiplier + 10*round(rho*10)
seed_data = seed_dag + 1
seed_constraint = seed_dag + 2
seed_boot = seed_dag + 3
```

The exact naming of `seed_dag`, `seed_data`, `seed_constraint`, and `seed_boot` must be retained. Regime information must also become part of the configuration that is serialized and hashed; otherwise two regimes with otherwise identical parameter values could collide.

The configuration is serialized with `json_for_hash()` and hashed with SHA-256 through `make_config_hash()`. The resulting `config_hash` is embedded in `job_id`. This prevents a changed configuration from silently masquerading as an old completed job.

A job ID remains deterministic. It should identify the experiment, major scenario parameters, replicate, misspecification settings, bootstrap settings, direction threshold, and a truncated configuration hash. The exact textual format may be extended to include regime information, but existing parameter names should remain intact.

---

## Generate once, reuse many times

A critical efficiency and fairness feature of the reference implementation is that the true DAG and data are created once per job, outside the constraint loop.

Inside `run_one_job()`, the current sequence is:

```text
make_true_dag()
    -> save true DAG

generate_data_from_dag()
    -> dat_num

discretize_for_bn()
    -> dat_disc

for each constraint_level:
    make_constraint_set()
    run_bn_boot_strength(dat_disc, ...)

    for each tau in THRESHOLDS:
        average_edges()
        calc_metrics()
```

This architecture must be preserved.

The same true DAG and the same generated data are therefore shared by `none`, `weak`, `strong`, and `current_misspecified` within one job. This creates a paired comparison rather than four unrelated simulations.

The data must not be regenerated inside the `constraint_level` loop. Likewise, the bootstrap strength table for a given constraint level must be calculated once and then reused across every value in `THRESHOLDS`. Changing `tau` is a post-processing operation on the same bootstrap strength table; it must not trigger another bootstrap run.

This is the exact meaning of the "load/generate once" requirement in this simulation workflow.

---

## Constraint definitions for the new regimes

### `none`

No blacklist and no whitelist are supplied.

### `weak`

The weak prior continues to encode only the minimal outcome-direction rule: `Y` cannot point to other variables.

The existing conceptual definition is retained:

```text
Y -> non-Y nodes is forbidden
```

### `strong`

The strong constraint becomes regime-aware. It must forbid scientifically invalid **between-block reverse directions** while leaving within-block edges allowed.

For Regime 1, the allowed core between-block relations are `A -> C`, `B -> C`, `A -> Y`, `B -> Y`, and `C -> Y`. The strong blacklist should prohibit directions that violate the regime's block-level ordering or role definition.

Regime 2 adds `D -> Y` and must preserve the direct-risk role of `D` according to the final approved template.

Regime 3 allows the chain `A -> B -> C -> Y`, as well as direct `A/B/C -> Y` edges.

Regime 4 adds `D -> Y` to the Regime 3 structure.

Within-block edges must remain searchable under `strong`.

### `current_misspecified`

The existing main-benchmark misspecification logic should be preserved conceptually. Starting from the strong blacklist, a proportion of true edges from the current job's true DAG is sampled according to `MISSPEC_RATE` and wrongly added to the blacklist.

Therefore, the misspecified constraint is job-specific and true-DAG-specific. It is not one global randomly sampled constraint reused across all 100 replicates.

Within a fixed job and fixed `current_misspecified` condition, the sampled misspecified edges must remain fixed for all bootstrap resamples and all values of `tau`.

---

## Exact constraint-edge audit and provenance

The new implementation must save the exact constraint edges used in every job and every constraint condition. This is a hard requirement.

The reference notebook already reserves the directories:

```text
constraint_audits/
wl_references/
```

The new implementation must actively write to these directories.

Immediately after:

```text
cons <- make_constraint_set(...)
```

and before:

```text
run_bn_boot_strength(...)
```

the exact `cons$blacklist` and `cons$whitelist` must be serialized.

A deterministic file naming convention should be used, for example:

```text
constraint_audits/<job_id>__<constraint_level>__constraint_audit.csv
wl_references/<job_id>__<constraint_level>__whitelist_reference.csv
```

Each constraint-audit row should contain the following provenance fields whenever applicable:

| Field | Purpose |
|---|---|
| `job_id` | Links the constraint to one true DAG and one generated dataset |
| `config_hash` | Verifies configuration identity |
| `experiment_id` | Experiment line |
| `regime_type` | Regime 1–4 |
| `constraint_level` | `none`, `weak`, `strong`, `current_misspecified`, or later misspecification variants |
| `constraint_type` | `blacklist`, `whitelist`, or `none` |
| `from`, `to` | Exact constrained arc |
| `from_block`, `to_block` | A/B/C/D/Y role annotation |
| `edge_role` | For example `A_to_C`, `B_to_Y`, `within_A` |
| `constraint_detail` | Human-readable explanation |
| `seed_constraint` | Seed used to construct or sample the constraint |
| `is_true_edge` | Whether the constrained arc is a true edge in the current DAG |
| `is_reversed_true_edge` | Whether the arc is the reverse of a true edge |
| `is_within_layer` | Whether both nodes belong to the same block |
| `is_between_layer` | Whether the edge crosses blocks |
| `is_misspecified_edge` | Whether this specific edge was deliberately added by misspecification |

For `none`, the implementation should still produce an auditable record indicating that zero blacklist and zero whitelist edges were used. This avoids ambiguity between "no constraint" and "constraint file missing because the pipeline failed."

The job-level result rows should also store `constraint_audit_file`, `whitelist_reference_file`, `n_blacklist_constraints`, and `n_whitelist_constraints` so that a summary row can be followed back to the exact constraint artifact.

The intended provenance chain is:

```text
figure or summary result
    -> job_id + constraint_level
    -> row_results
    -> averaged_edges / estimated_edges
    -> constraint_audits
    -> exact blacklist / whitelist used for learning
```

---

## Bootstrap learning and threshold reuse

The learning core remains:

```text
bnlearn::boot.strength(
    data = dat_disc,
    R = B_BOOT,
    algorithm = algorithm,
    algorithm.args = list(
        score = score,
        iss = iss,
        blacklist = ...,
        whitelist = ...
    ),
    cpdag = FALSE
)
```

The default algorithm parameters continue to be `algorithm = "hc"`, `score = "bde"`, and `iss = 10` unless explicitly changed in the experiment table.

For each constraint level, `run_bn_boot_strength()` is executed once. The resulting strength table is reused for all `tau` values from `THRESHOLDS`.

`average_edges()` must continue to collapse the bootstrap output to at most one directed edge per unordered node pair. The current logic constructs a pair key from the alphabetically ordered node names, groups by the pair, keeps the strongest/most directionally supported representation, and applies `DIRECTION_THRESHOLD`. This prevents both directions of the same pair from entering the final averaged graph as duplicated edges.

---

## Metrics and reporting criteria

The main reporting criteria should follow the DAGSLAM paper by Zhao and Jia (2025), which reports five standard criteria for both directed structures and skeleton structures:

```text
FDR
TPR
FPR
SHD
F1
```

The reference is:

> Zhao Y, Jia J. DAGSLAM: causal Bayesian network structure learning of mixed type data and its application in identifying disease risk factors. BMC Medical Research Methodology. 2025;25:154. DOI: 10.1186/s12874-025-02582-6.

For the directed structure, the notation is:

```text
TP = estimated edges with the correct true direction
R  = estimated edges whose direction is reversed relative to the truth
FP = estimated edges absent from the true skeleton
P  = number of estimated directed edges
T  = number of true directed edges
F  = number of false/non-edge possibilities under the paper's definition
E  = extra skeleton edges
M  = missing skeleton edges
```

The directed criteria are:

```text
FDR_directed = (R + FP) / P
TPR_directed = TP / T
FPR_directed = (R + FP) / F
SHD_directed = E + M + R
F1_directed = 2 * (1 - FDR_directed) * TPR_directed /
              ((1 - FDR_directed) + TPR_directed)
```

The key relationship with the current notebook's familiar metrics is:

```text
precision = 1 - FDR
recall = TPR
```

The new code should report the Zhao–Jia criteria for both directed and skeleton structures. Existing columns such as `directed_precision`, `directed_recall`, `directed_f1`, `skeleton_precision`, `skeleton_recall`, `skeleton_f1`, and existing diagnostic metrics may be retained for backward compatibility and richer interpretation, but the principal published summary must include directed and skeleton FDR, TPR, FPR, SHD, and F1.

Later role-aware extensions may additionally report between-block and within-block recovery. These extensions should supplement rather than replace the standard overall criteria.

---

## Output-directory contract

The implementation should preserve the current output philosophy: one deterministic artifact family per job, atomic writes, and separate directories by artifact type.

The intended structure is:

```text
OUT_DIR/
├── experiment_grid_current.csv
├── job_table_current.csv
├── run_manifest_current.json
├── row_results/
├── metadata/
├── true_dags/
├── strength_tables/
├── estimated_edges/
├── averaged_edges/
├── constraint_audits/
├── wl_references/
├── logs/
├── all_results_raw.csv
└── summary_results.csv
```

Different experiment lines must continue to use separate `OUT_DIR` locations. Results from different experiment versions must not be merged merely because they exist under a common parent directory.

---

## Atomic writes and protection from partial files

The current notebook uses `atomic_write_csv()` and `atomic_write_json()`.

The write protocol is:

```text
write to <target>.tmp
remove old target if necessary
rename .tmp to final target
```

This behavior must be retained for new artifact types, including constraint audits.

The purpose is to avoid treating a partially written CSV or JSON file as a valid completed output after a Colab disconnect, runtime crash, or interrupted Drive write.

Downstream file discovery must continue to exclude `.tmp` files.

---

## Breakpoint resume and job-state machine

The current notebook already implements job-level breakpoint resume through `inspect_job_status()` and deterministic output artifacts. This logic should be preserved.

The status machine is:

| `run_status` | Detection rule | `recommended_action` |
|---|---|---|
| `completed_current` | Result file, metadata file, and completed flag all exist, and metadata `config_hash` matches the current job | `skip` |
| `config_mismatch` | Existing files are present but metadata hash differs from current configuration | `manual_check_or_rerun` |
| `result_without_metadata` | Result exists but metadata is missing | `manual_check_or_rerun` |
| `interrupted_or_incomplete` | Running flag exists but completed flag does not | `rerun` |
| `pending` | No completed result is found | `run` |

The main job loop selects only jobs whose `recommended_action` is contained in:

```text
RUN_ACTIONS <- c("run", "rerun")
```

Completed current jobs are not rerun when `OVERWRITE_EXISTING <- FALSE`.

At job start, the pipeline writes `<job_id>_RUNNING.json` and the job metadata. At successful completion, it writes the job result CSV and `<job_id>_COMPLETED.json`, then removes the running flag.

Therefore, after an interrupted Colab session, the user can rebuild the job table and status table. Completed matching jobs are skipped, interrupted jobs are classified as rerunnable, and pending jobs are run normally.

A completed job must never be considered valid solely because a result filename exists. The current design correctly requires result + metadata + completed flag + matching `config_hash`.

---

## Failure isolation and batch progress

The main loop wraps each call to `run_one_job()` in `tryCatch()`.

If one job fails, the pipeline writes a `<job_id>_FAILED.json` artifact containing the error information and continues to the next job instead of terminating the whole batch.

After every processed job, accumulated batch status is atomically written to:

```text
logs/batch_progress_<timestamp>.csv
```

This feature must be preserved. It allows a long run to retain a continuously updated external progress record even if the notebook session later disconnects.

---

## Fresh status rescan

The existing notebook provides a forced status-rescan step. Before rescanning, old in-memory columns `run_status`, `recommended_action`, and `status_detail` are removed from `job_table`. The code then applies `inspect_job_status()` to every current job and rebuilds the status columns from the files currently visible in `OUT_DIR`.

This step should remain available after the main run. It prevents stale in-memory status labels from being mistaken for the real state of Drive outputs.

---

## Deduplication and stale-result isolation

The project uses several different forms of deduplication. They should be documented separately because they solve different problems.

### Configuration identity

`config_json` and SHA-256 `config_hash` uniquely identify the complete job configuration. The hash becomes part of `job_id`, preventing silent collision between changed configurations.

### Edge-level deduplication

True-edge and constraint helpers use `distinct(from, to)` to remove duplicate arcs. Constraint cleaning also removes self-loops.

The averaged-edge logic collapses both representations of an unordered pair into one pair key, then retains one selected direction after comparing strength and directional support. This prevents duplicate opposite-direction records for the same pair in the final averaged graph.

### One result file per job

Each `job_id` has one deterministic row-result path:

```text
row_results/<job_id>.csv
```

With `OVERWRITE_EXISTING <- FALSE`, a valid `completed_current` job is skipped. This prevents accidental duplicate recomputation from creating multiple result files for the same job.

### Rebuild only from the current job table

`rebuild_results_from_rows(current_only = TRUE, job_table_ref = job_table)` does not blindly read every CSV in `row_results/`. It constructs the expected result paths from the current `job_table`, retains only files that actually exist, excludes `.tmp` files, and filters the combined results back to current `job_id` values.

This is essential for preventing stale outputs from previous configurations from contaminating the current summary.

### Planned uniqueness validation

The new implementation should add an explicit validation check before summary aggregation. Under the main BN experiment, the logical uniqueness key is expected to include at least:

```text
job_id + method + constraint_level + tau
```

Any unexpected duplicate under the appropriate key should trigger an audit or error rather than being silently removed with a blanket `distinct()` call. Legitimate rows across different `tau` values or constraint levels must not be discarded.

---

## Result rebuilding and aggregation

The current `rebuild_results_from_rows()` should be retained and extended.

With `current_only = TRUE`, it reads only current-job result files, writes `all_results_raw.csv`, groups by the experiment/scenario/method configuration, and writes `summary_results.csv`.

The new grouping keys must include the regime identifier and continue to preserve existing names such as `edge_multiplier`, `rho`, `tau`, `within_true_rate`, `B_BOOT`, `constraint_level`, and the misspecification fields.

The summary should add means and, where appropriate, standard deviations for the Zhao–Jia directed and skeleton criteria. Existing metrics can remain in the raw and summary outputs for backward compatibility.

---

## Constraint reuse and fairness within a job

A critical design requirement is that a given job should have a single deterministic constraint realization for each `constraint_level`.

For `current_misspecified`, the exact true edges sampled into the wrong blacklist are determined by the current true DAG and `seed_constraint`. They must not be resampled for each bootstrap replicate. They must not be resampled for different `tau` values.

The intended hierarchy is:

```text
one job
    -> one true DAG
    -> one generated dataset
    -> one dat_disc

    -> none: one constraint set
    -> weak: one constraint set
    -> strong: one constraint set
    -> current_misspecified: one sampled misspecified constraint set

        -> one bootstrap strength table per constraint level
            -> multiple tau-specific averaged DAGs
```

Saving the exact constraint audit makes this hierarchy verifiable rather than merely assumed.

---

## Pilot execution plan

The first implementation should not immediately launch the full `REP_VALUES <- "1:100"`, `B_BOOT_VALUE <- 200` experiment.

A pilot should use the same parameter names and code path as the formal experiment but with small values, for example a small subset of `REP_VALUES` and a small `B_BOOT_VALUE`. The pilot must verify that all four regimes generate acyclic DAGs; the expected node counts are correct; within-block edges are possible; `target_edges` is reached when feasible; the same `dat_disc` is reused across constraints; strong constraints do not forbid within-block edges; misspecified edges are sampled from the current true DAG; constraint-audit files are written; completed jobs are skipped; interrupted jobs are classified correctly; and the Zhao–Jia metrics can be calculated without ambiguity.

The pilot is a structural validation stage, not a source of paper results.

---

## Formal experiment plan

The formal experiment should continue to use the existing parameter names and the previously established scenario dimensions unless the research design is explicitly changed.

The reference full-run values are:

```text
N_VALUES      <- "5000"
EDGE_MULTS    <- "1;2"
RHO_VALUES    <- "0.2;0.8"
REP_VALUES    <- "1:100"
B_BOOT_VALUE  <- 200
THRESHOLDS    <- "0.4;0.8"
MISSPEC_RATE  <- 0.10
WITHIN_TRUE_RATE <- 0.25
DIRECTION_THRESHOLD <- 0.5
```

The four-regime design adds `regime_type` as the mechanism dimension. Existing parameter names and meanings should remain unchanged.

The main 5-benchmark line is expected to compare:

```text
none
weak
strong
current_misspecified
graphical_lasso
```

The existing 8-misspecification machinery can be preserved for later extension, but the first implementation priority is the four-regime main benchmark with complete constraint provenance.

---

## Recommended notebook execution workflow

The notebook should be used in a safe two-stage manner.

First, run the setup, utility, experiment-grid, and job-table cells with `RUN_JOBS <- FALSE`. Inspect `job_table_current.csv`, the status summary, parameter combinations, and recommended actions. This step should not perform any simulation jobs.

After confirming the grid, set `RUN_JOBS <- TRUE` and keep `OVERWRITE_EXISTING <- FALSE` for a normal resumable run.

If Colab disconnects or the process stops, rerun the setup and job-table cells. The status scanner will classify jobs from the actual output files. Jobs with valid completed artifacts and matching hashes will be skipped; interrupted jobs will be selected for rerun; pending jobs will be run normally.

After execution, run the fresh status-rescan cell and then rebuild results with:

```text
rebuild_results_from_rows(current_only = TRUE, job_table_ref = job_table)
```

This ensures that `all_results_raw.csv` and `summary_results.csv` are built only from the current experiment grid.

---

## Implementation phases

| Phase | Main work | Acceptance condition |
|---|---|---|
| Architecture audit | Map the existing notebook functions, outputs, parameters, resume logic, and metrics | No existing parameter is unintentionally renamed |
| Regime definitions | Add A/B/C/D/Y node blocks and Regime 1–4 legal relations | Node counts and legal directions match the specification |
| True-DAG generator | Replace old five-tier graph logic while preserving core + random-extra + acyclicity architecture | Every saved DAG is acyclic and reaches the intended edge target when feasible |
| Constraint generator | Make `strong` regime-aware and preserve within-block search space | Strong blacklist contains only intended forbidden arcs |
| Constraint provenance | Save exact blacklist/whitelist artifacts before BN learning | Every job × constraint level has an auditable constraint record |
| Metric extension | Add directed and skeleton FDR, TPR, FPR, SHD, F1 following Zhao–Jia definitions | Metrics pass unit tests on hand-constructed toy graphs |
| Resume and deduplication validation | Verify hashes, flags, current-only rebuild, and uniqueness keys | Restarting the notebook does not rerun completed-current jobs or mix stale results |
| Pilot | Small replicate/bootstrap run across all four regimes | Outputs, constraints, DAGs, and metrics are internally consistent |
| Full run | Formal `REP_VALUES` and `B_BOOT_VALUE` | Complete resumable experiment outputs |
| Post-processing | Build comparison tables, representative DAGs, consensus DAGs, and provenance summaries | Every reported figure/table can be traced back to jobs and constraints |

---

## Acceptance criteria for Version 1 implementation

Version 1 code should not be considered complete until the following are true.

The four regimes generate the intended role-based DAG structures with `A = 5`, `B = 5`, `C = 5`, `D = 5`, and `Y = 1`, with Regime 1 and Regime 3 using 16 nodes and Regime 2 and Regime 4 using 21 nodes under the current regime definition.

Within-block true edges are allowed and are not automatically forbidden by the strong constraint.

One job generates its true DAG, `dat_num`, and `dat_disc` only once, outside the constraint loop.

One constraint level generates one constraint set and one bootstrap strength table, reused across all `tau` values.

Every actual blacklist and whitelist is saved before BN learning, including the exact misspecified edges sampled for `current_misspecified`.

Every completed job has result, metadata, completed flag, and matching configuration hash.

Interrupted jobs can be resumed without rerunning valid completed-current jobs.

Atomic writes are used for all critical CSV and JSON artifacts.

Summary rebuilding uses the current job table by default and excludes stale and `.tmp` files.

Directed and skeleton FDR, TPR, FPR, SHD, and F1 are reported using the Zhao–Jia criteria.

Existing parameter names such as `EDGE_MULTS`, `RHO_VALUES`, `REP_VALUES`, `B_BOOT_VALUE`, `THRESHOLDS`, `MISSPEC_RATE`, `WITHIN_TRUE_RATE`, and `DIRECTION_THRESHOLD` are preserved.

---

## Version 1 boundary

This document intentionally freezes the engineering and scientific framework before code generation. It does not yet define every individual fixed core arc such as whether a specific common core contains `A1 -> C1` or `A2 -> C2`; that exact edge template should be frozen during implementation review before the full run. The important Version 1 requirement is that every regime has a reproducible shared core backbone across replicates, random extra edges around that backbone, explicit edge-role annotation, and complete provenance.

Likewise, the first implementation should prioritize the main 5-benchmark comparison and reliable constraint auditing. The more detailed 8-misspecification experiments can be reconnected after the four-regime main pipeline has passed pilot validation.

---

## Summary

The new Regime 1–4 experiment will reuse the reliable engineering skeleton of `simulation0618_regime1.ipynb`: editable experiment tables, deterministic job configurations, configuration hashes, job-specific seeds, atomic writes, explicit running/completed flags, status scanning, resumable execution, per-job result files, current-job-only result rebuilding, bootstrap strength reuse across thresholds, and deterministic output organization.

The scientific changes are concentrated in the true-DAG generator, block annotations, regime-aware constraints, and reporting criteria. The new framework uses A/B/C/D/Y role-based DAGs, allows within-block edges, treats Regime 1 as the primary parallel-upstream setting, records the exact constraint edges used in every job, and reports the standard directed and skeleton FDR, TPR, FPR, SHD, and F1 criteria used by Zhao and Jia, while preserving the existing parameter names and workflow conventions.
