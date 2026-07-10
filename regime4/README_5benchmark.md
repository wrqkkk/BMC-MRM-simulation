# Regime 4 · 5benchmark 说明

对应代码：`simulation0701_regime4_5benchmark.ipynb`

## 1. 实验目的

Regime 4 结构：

```text
A -> B -> C -> Y
A -> Y
B -> Y
C -> Y
D -> Y
```

节点数 `p=21`。固定：`BASE_DIR <- "/content/drive/MyDrive/CBN_simulation_2"`，`SEED_BASE <- 2026`。

5benchmark：`none / weak / strong / current_misspecified / graphical_lasso`。

`current_misspecified` 统一实现为：在 strong 基础上，从当前 true DAG 抽取 10% true edges，并将每条被抽中边的两个方向都加入 blacklist。

## 2. 正式参数与结果规模

`EDGE_MULTS=1;2`，`RHO_VALUES=0.2;0.8`，`REP_VALUES=1:100`，`B_BOOT_VALUE=200`，`THRESHOLDS=0.4;0.8`。共 400 jobs；完整 raw results 理论上约 **3600 行**，summary 为 **36 行**。

## 3. 输出文件夹结构

```text
/content/drive/MyDrive/CBN_simulation_2/outputs/simulation0701_regime4_5benchmark/
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

先确认正式运行达到 **400 / 400 jobs**。主结果先看 `summary_results.csv`，按 8 个场景分别分析：`edge_multiplier ∈ {1,2}`、`rho ∈ {0.2,0.8}`、`tau ∈ {0.4,0.8}`。

重点指标：`directed_FDR_mean`、`directed_TPR_mean`、`directed_FPR_mean`、`directed_SHD_mean`、`directed_F1_mean`；辅助看 `skeleton_F1_mean`、`backward_edge_rate_mean` 和 `n_est_edges_mean`。`precision = 1 - FDR`，`recall = TPR`。

需要具体 replicate 时看 `all_results_raw.csv`；解释边级机制时看 `true_dags/`、`strength_tables/`、`averaged_edges/`、`constraint_audits/` 和 `scenario_consensus_edges.csv`。

## 4. 怎样对比

1. `none vs weak vs strong`：正确先验逐步增强的收益。
2. `strong vs current_misspecified`：错误禁止 10% true edges 的代价。
3. `current_misspecified vs none`：部分错误先验是否仍保留总体收益。
4. graphical lasso 只做 skeleton benchmark。

Regime 4 还应重点判断：在链式 A -> B -> C 结构加 D -> Y 的复杂场景中，strong 是否仍稳定改善 directed recovery，以及错误 constraint 是否更容易破坏 chain edges 或 direct-to-Y edges。

## 5. 结论组织

按 **overall directed recovery → skeleton → chain/direct-risk edge recovery → rho / graph density / tau** 组织。不要把 strong 的 `backward_edge_rate=0` 单独当作算法学习能力证据，应结合 TPR、F1 和 SHD 判断约束收益。
