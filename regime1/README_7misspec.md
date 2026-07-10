# Regime 1 · 7misspec / 7-condition Constraint Sensitivity 说明

对应代码：`simulation0701_regime1_7misspec.ipynb`

## 1. 实验目的

该实验不再包含任何 BL reverse 条件。最终固定为 7 个 conditions：

```text
strong
BL_missing
BL_extra
WL_correct
WL_missing
WL_extra
WL_reversed
```

设计参考 0620 组会中确定的 BL + WL sensitivity 主线，并结合 0701 的 Regime 1–4 role-based DAG 结构与比例化 whitelist 设计。

Regime 1 true DAG：

```text
A, B -> C -> Y
A -> Y
B -> Y
C -> Y
```

固定：`BASE_DIR <- "/content/drive/MyDrive/CBN_simulation_2"`，`SEED_BASE <- 2026`。

## 2. 七个 conditions 的定义

| 条件 | 定义 |
|---|---|
| `strong` | 当前 Regime 的正确 strong blacklist baseline |
| `BL_missing` | 从 strong blacklist 中删除 10% 本应禁止的 arcs |
| `BL_extra` | 从当前 true DAG 抽取 10% true edges，并把每条被抽中边的两个方向都加入 blacklist |
| `WL_correct` | strong + 正确 whitelist；覆盖当前 true DAG 的 25% edges |
| `WL_missing` | 从 `WL_correct` 中移除 25% 的 correct WL edges |
| `WL_extra` | 用 false within-block edges 替换 25% 的 correct WL edges，保持 whitelist 总规模不变 |
| `WL_reversed` | 将 25% 的 correct within-block WL edges 反向强制加入 |

**明确不做：`BL_reversed`、`BL_within_reversed`、`BL_between_reversed`。**

## 3. WL 比例设计

```r
WL_REFERENCE_RATE <- 0.25
WL_PERTURB_RATE   <- 0.25
```

| `edge_multiplier` | true edges | `WL_correct` | WL perturbation |
|---:|---:|---:|---:|
| 1 | 16 | 4 | 1 |
| 2 | 32 | 8 | 2 |

这样 sparse 与 dense DAG 下的 whitelist 相对覆盖率保持一致。

## 4. 正式参数与结果规模

```text
EDGE_MULTS = 1;2
RHO_VALUES = 0.2;0.8
REP_VALUES = 1:100
B_BOOT_VALUE = 200
THRESHOLDS = 0.4;0.8
```

共 400 jobs。每个 job 产生 `7 constraints × 2 tau = 14` 行，因此完整 `all_results_raw.csv` 理论上约 **5600 行**，summary 为 **56 行**。

## 5. 输出文件夹结构

```text
/content/drive/MyDrive/CBN_simulation_2/outputs/simulation0701_regime1_7misspec/
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
├── constraint_changes/
├── wl_references/
├── logs/
├── tables/
├── figures/
├── all_results_raw.csv
├── summary_results.csv
├── true_dag_edges.csv
├── averaged_network_edges.csv
└── scenario_consensus_edges.csv
```

### 结果使用方法

#### 第一步：确认任务完整性

正式参数下应有 **400 / 400 jobs**。若不足，先看 `logs/`、`*_FAILED.json` 与 fresh status rescan，不要直接用不完整 summary 下结论。

#### 第二步：先看 `summary_results.csv`

主结果先按 8 个场景分别看，不要一上来跨场景简单平均：`edge_multiplier ∈ {1,2}`、`rho ∈ {0.2,0.8}`、`tau ∈ {0.4,0.8}`。

| 指标 | 方向 | 解释 |
|---|---|---|
| `directed_FDR_mean` | 越低越好 | 估计方向边中的错误发现比例 |
| `directed_TPR_mean` | 越高越好 | true directed edges 的恢复率 |
| `directed_FPR_mean` | 越低越好 | 非真实可能边中的错误发现率 |
| `directed_SHD_mean` | 越低越好 | 估计 DAG 与 true DAG 的结构距离 |
| `directed_F1_mean` | 越高越好 | directed precision 与 recall 的折中，主指标之一 |
| `skeleton_F1_mean` | 越高越好 | 只看 adjacency，不看方向 |
| `backward_edge_rate_mean` | 越低越好 | backward edges；strong 下为 0 很大程度是结构性结果 |
| `n_est_edges_mean` | 解释性指标 | 用于理解 tau 的 precision–recall trade-off |

其中 `precision = 1 - FDR`，`recall = TPR`。正式报告优先使用 Zhao–Jia 口径的 FDR、TPR、FPR、SHD、F1。

#### 第三步：需要定位具体 job 时看 `all_results_raw.csv`

用于计算 standard deviation / confidence interval、找 representative job、检查异常 replicate，并从 `job_id` 追溯 true DAG、constraint 和 averaged network。

#### 第四步：用 edge 文件解释为什么指标变化

- `true_dags/`：每个 job 的 true DAG；
- `strength_tables/`：bootstrap strength 与 direction；
- `averaged_edges/` / `estimated_edges/`：tau 后的最终边；
- `constraint_audits/`：真正传给 `bnlearn` 的 exact blacklist / whitelist；
- `constraint_changes/`：实际删除、增加、替换或反转了哪些边；
- `scenario_consensus_edges.csv`：跨 replicates 的 consensus edge 频率。

结论组织应遵循：**先 overall directed recovery，再 skeleton，再错误类型/边类型，再解释 tau、rho、graph density。**

## 6. 7misspec 应怎样比较

### 6.1 Blacklist sensitivity

```text
strong vs BL_missing vs BL_extra
```

回答：正确 blacklist 缺失一部分，与错误额外禁止 true edges，哪一种损伤更大？

### 6.2 Correct whitelist 的增益

```text
strong vs WL_correct
```

回答：在已有正确 strong blacklist 的基础上，再提供少量高置信 true edges，是否进一步改善 directed recovery 与 bootstrap stability？

### 6.3 Whitelist 错误类型

```text
WL_correct vs WL_missing vs WL_extra vs WL_reversed
```

回答：漏掉正确 WL、加入 false WL、强制错误方向，三种错误知识各自主要造成 recall loss、false positives、reversal 还是 SHD 上升？

### 6.4 BL 与 WL 横向比较的注意事项

BL 的扰动率是 10%；WL 的 perturbation 是 `WL_correct` 的 25%。因此不要直接把 BL 与 WL 的损伤绝对值解释成“同等错误强度下谁更严重”，除非后续另做 matched-intensity sensitivity。

## 7. `constraint_changes/` 是最关键的解释文件

建议按 `edge_role` 汇总每次随机改变的边：`A_to_C`、`B_to_C`、`A_to_Y`、`B_to_Y`、`C_to_Y`、`within_A/B/C`。这样才能判断错误先验的损伤是否取决于被破坏的是 mechanism-chain edge、direct-to-Y edge，还是 within-block edge。

## 8. 结论组织模板

> 相比 strong，BL_missing 在……场景主要导致……；BL_extra 因为同时禁止被抽中 true edge 的两个方向，主要表现为……。WL_correct 相比 strong ……，说明少量正确高置信边能够/不能够进一步提高……。在 WL 错误中，WL_missing 主要影响……，WL_extra 主要影响……，WL_reversed 主要影响……。这些差异在高 `rho`、高图密度或高 `tau` 下是否放大，应按 8 个场景分别检查。
