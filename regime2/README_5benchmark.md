# Regime 2 · 5benchmark 说明

对应代码：`simulation0701_regime2_5benchmark.ipynb`

## 1. 实验目的

Regime 2 结构：

```text
A, B -> C -> Y
A -> Y
B -> Y
C -> Y
D -> Y
```

节点数 `p=21`。固定：`BASE_DIR <- "/content/drive/MyDrive/CBN_simulation_2"`，`SEED_BASE <- 2026`。

5benchmark：

```text
none
weak
strong
current_misspecified
graphical_lasso
```

| 条件 | 约束逻辑 |
|---|---|
| `none` | 无 blacklist / whitelist |
| `weak` | 只禁止 `Y -> non-Y` |
| `strong` | Regime 2 正确 role-based strong blacklist |
| `current_misspecified` | strong + 抽取当前 true DAG 中 10% true edges，将其两个方向都加入 blacklist |
| `graphical_lasso` | 无向 skeleton benchmark |

## 2. 正式参数与结果规模

`EDGE_MULTS=1;2`，`RHO_VALUES=0.2;0.8`，`REP_VALUES=1:100`，`B_BOOT_VALUE=200`，`THRESHOLDS=0.4;0.8`。共 400 jobs；完整 raw results 理论上约 **3600 行**，summary 为 **36 行**。

## 3. 输出文件夹结构

```text
/content/drive/MyDrive/CBN_simulation_2/outputs/simulation0701_regime2_5benchmark/
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

先确认正式运行达到 **400 / 400 jobs**。主结果先看 `summary_results.csv`，按 8 个场景分别分析：`edge_multiplier ∈ {1,2}`、`rho ∈ {0.2,0.8}`、`tau ∈ {0.4,0.8}`，不要一开始跨场景平均。

重点指标：

| 指标 | 方向 | 用途 |
|---|---|---|
| `directed_FDR_mean` | 越低越好 | 错误方向发现比例 |
| `directed_TPR_mean` | 越高越好 | true directed edges 恢复率 |
| `directed_FPR_mean` | 越低越好 | 非真实边错误发现率 |
| `directed_SHD_mean` | 越低越好 | 结构距离 |
| `directed_F1_mean` | 越高越好 | 主要综合指标 |
| `skeleton_F1_mean` | 越高越好 | adjacency 恢复 |
| `backward_edge_rate_mean` | 越低越好 | 方向诊断；strong=0 主要是结构性结果 |
| `n_est_edges_mean` | 解释性指标 | 解释 tau 的 precision–recall trade-off |

`precision = 1 - FDR`，`recall = TPR`。正式报告优先使用 Zhao–Jia 口径的 FDR、TPR、FPR、SHD、F1。

需要具体 job 时看 `all_results_raw.csv`；解释边级机制时看 `true_dags/`、`strength_tables/`、`averaged_edges/`、`constraint_audits/` 和 `scenario_consensus_edges.csv`。

## 4. 怎样对比

1. `none vs weak vs strong`：正确先验逐步增强的收益。
2. `strong vs current_misspecified`：错误禁止 10% true edges 的代价。
3. `current_misspecified vs none`：部分错误先验是否仍保留总体收益。
4. graphical lasso 只和 BN 做 skeleton 层面对比。

Regime 2 还应特别观察 D -> Y 的加入是否改变 strong 的 directed F1 收益，以及错误 constraint 是否更容易破坏 direct-to-Y edges。

## 5. 结论组织

先写 overall directed recovery，再写 skeleton，再解释 rho、edge density 和 tau。不要把 strong 的 `backward_edge_rate=0` 单独作为算法学习能力证据；应结合 TPR、F1 和 SHD 判断正确约束是否真正提高恢复质量。
