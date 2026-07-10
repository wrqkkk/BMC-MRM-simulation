# Regime 1 · 5benchmark 说明

对应代码：`simulation0701_regime1_5benchmark.ipynb`

## 1. 实验目的

Regime 1 的 5benchmark 比较不同先验强度对 BN 结构恢复的影响：

```text
none
weak
strong
current_misspecified
graphical_lasso
```

Regime 1 true DAG：

```text
A, B -> C -> Y
A -> Y
B -> Y
C -> Y
```

固定：`BASE_DIR <- "/content/drive/MyDrive/CBN_simulation_2"`，`SEED_BASE <- 2026`。

## 2. 五个 benchmark 的含义

| 条件 | 约束逻辑 | 主要问题 |
|---|---|---|
| `none` | 无 blacklist / whitelist | 完全无约束搜索的基线 |
| `weak` | 只禁止 `Y -> non-Y` | 最小可靠方向知识能否改善恢复 |
| `strong` | Regime 1 的正确 role-based strong blacklist | 完整 block-level 先验收益 |
| `current_misspecified` | strong + 抽取当前 true DAG 中 10% true edges，并将其两个方向都加入 blacklist | 错误先验的漏边代价 |
| `graphical_lasso` | 无向 skeleton benchmark | 只比较 skeleton，不比较 directed metrics |

`current_misspecified` 与 7misspec 中的 `BL_extra` 对应。

## 3. 正式参数与结果规模

```text
EDGE_MULTS = 1;2
RHO_VALUES = 0.2;0.8
REP_VALUES = 1:100
B_BOOT_VALUE = 200
THRESHOLDS = 0.4;0.8
```

共 400 jobs。每个 job 产生 8 行 BN 结果（4 constraint × 2 tau）和 1 行 graphical lasso 结果，因此完整 raw results 理论上约 **3600 行**；summary 为 **36 行**。

## 4. 输出文件夹结构

```text
/content/drive/MyDrive/CBN_simulation_2/outputs/simulation0701_regime1_5benchmark/
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

先检查 notebook 输出：`Current job_table row result files found: X / X`。正式参数下应有 **400 / 400 jobs**。若不足，先看 `logs/`、`*_FAILED.json` 与 fresh status rescan，不要直接用不完整 summary 下结论。

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
| `backward_edge_rate_mean` | 越低越好 | 机制上不允许的 backward edges；strong 下为 0 很大程度是结构性结果 |
| `n_est_edges_mean` | 解释性指标 | 用于理解 tau 的 precision–recall trade-off |

其中 `precision = 1 - FDR`，`recall = TPR`。正式报告优先使用 Zhao–Jia 口径的 FDR、TPR、FPR、SHD、F1。

#### 第三步：需要定位具体 job 时看 `all_results_raw.csv`

它用于计算 standard deviation / confidence interval、找 representative job、检查异常 replicate，并从 `job_id` 追溯 true DAG、constraint 和 averaged network。

#### 第四步：用 edge 文件解释为什么指标变化

- `true_dags/`：每个 job 的 true DAG；
- `strength_tables/`：bootstrap strength 与 direction；
- `averaged_edges/` / `estimated_edges/`：tau 后的最终边；
- `constraint_audits/`：真正传给 `bnlearn` 的 exact blacklist / whitelist；
- `scenario_consensus_edges.csv`：跨 replicates 的 consensus edge 频率。

结论组织应遵循：**先 overall directed recovery，再 skeleton，再错误类型/边类型，再解释 tau、rho、graph density。**

## 5. 5benchmark 应怎样比较

1. **none vs weak vs strong**：正确先验逐步增强后，directed F1、FDR、TPR、SHD 如何变化。
2. **strong vs current_misspecified**：错误禁止 10% true edges 的代价。
3. **current_misspecified vs none**：部分错误先验是否仍保留总体收益。
4. **BN vs graphical lasso**：只在 skeleton 指标层面对比，不能把 graphical lasso 当 directed 方法。

## 6. 结论组织模板

> 在 `edge_multiplier=...`, `rho=...`, `tau=...` 下，strong 相比 none 的 directed F1 从 ... 变为 ...，FDR 从 ... 变为 ...，TPR 从 ... 变为 ...。这说明正确的 role-based constraint 主要改善了……。与此同时 skeleton F1 ……，提示 constraint 的主要收益集中在方向恢复/或 adjacency 恢复。current_misspecified 相比 strong ……，说明错误禁止真实边主要造成……。

不要把 `strong` 的 `backward_edge_rate=0` 单独当成算法学习能力证据，因为这些 backward directions 本身被 blacklist 排除；更关键的是看 **在排除不合理方向的同时，TPR、F1 和 SHD 是否改善**。
