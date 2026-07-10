# Regime 4 · 7misspec / 7-condition Constraint Sensitivity 说明

对应代码：`simulation0701_regime4_7misspec.ipynb`

## 1. 实验目的

最终固定 7 个 conditions：

```text
strong
BL_missing
BL_extra
WL_correct
WL_missing
WL_extra
WL_reversed
```

**不做任何 BL reverse 条件。** Regime 4 结构为：

```text
A -> B -> C -> Y
A -> Y
B -> Y
C -> Y
D -> Y
```

节点数 `p=21`。固定：`BASE_DIR <- "/content/drive/MyDrive/CBN_simulation_2"`，`SEED_BASE <- 2026`。

## 2. 七个 conditions

| 条件 | 定义 |
|---|---|
| `strong` | Regime 4 正确 strong blacklist baseline |
| `BL_missing` | 从 strong blacklist 中删除 10% 本应禁止的 arcs |
| `BL_extra` | 从当前 true DAG 抽取 10% true edges，并把每条被抽中边的两个方向都加入 blacklist |
| `WL_correct` | strong + 正确 WL，覆盖 true DAG 的 25% edges |
| `WL_missing` | 从 WL_correct 中移除 25% correct WL edges |
| `WL_extra` | 用 false within-block edges 替换 25% correct WL edges，保持 WL 总规模 |
| `WL_reversed` | 将 25% correct within-block WL edges 反向强制加入 |

明确不做 `BL_reversed`、`BL_within_reversed`、`BL_between_reversed`。

## 3. WL 比例设计

```r
WL_REFERENCE_RATE <- 0.25
WL_PERTURB_RATE   <- 0.25
```

| `edge_multiplier` | true edges | `WL_correct` | WL perturbation |
|---:|---:|---:|---:|
| 1 | 21 | 6 | 2 |
| 2 | 42 | 11 | 3 |

## 4. 正式参数与结果规模

400 jobs；每个 job 产生 `7 constraints × 2 tau = 14` 行，因此完整 raw results 理论上约 **5600 行**，summary 为 **56 行**。

## 5. 输出文件夹结构

```text
/content/drive/MyDrive/CBN_simulation_2/outputs/simulation0701_regime4_7misspec/
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

先确认 **400 / 400 jobs**。主结果先按 8 个场景分别看：`edge_multiplier ∈ {1,2}`、`rho ∈ {0.2,0.8}`、`tau ∈ {0.4,0.8}`。

主指标是 directed / skeleton 的 `FDR`、`TPR`、`FPR`、`SHD`、`F1`。重点使用 `directed_F1_mean`、`directed_FDR_mean`、`directed_TPR_mean`、`directed_SHD_mean`；`skeleton_F1_mean` 用于区分 adjacency 与 orientation；`n_est_edges_mean` 用于解释 tau；`backward_edge_rate_mean` 只作诊断。

需要具体 replicate 时看 `all_results_raw.csv`；需要解释错误先验实际改了什么边时，必须看 `constraint_changes/` 和 `constraint_audits/`，并按 `edge_role` 汇总，例如 `A_to_B`、`B_to_C`、`A_to_Y`、`B_to_Y`、`C_to_Y`、`D_to_Y`、`within_A/B/C/D`。

## 6. 怎样对比和组织结论

### Blacklist sensitivity

`strong vs BL_missing vs BL_extra`：比较缺失正确 blacklist 与错误禁止 true edges 哪一种损伤更大。

### Correct whitelist 增益

`strong vs WL_correct`：判断少量正确高置信边是否进一步改善 directed recovery 与 bootstrap stability。

### Whitelist 错误类型

`WL_correct vs WL_missing vs WL_extra vs WL_reversed`：分别识别漏掉正确 WL、加入 false WL、强制错误方向造成的损伤。

BL 扰动率是 10%，WL perturbation 是 WL_correct 的 25%，因此不能直接把二者损伤大小解释成同等错误强度下的横向比较。

结论应按：**overall directed recovery → skeleton → constraint damage type → chain/direct-risk edge role → rho / graph density / tau** 的顺序组织。
