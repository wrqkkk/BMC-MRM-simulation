# Regime 1–4 代码修改与仓库整理报告

## 1. 本次修改范围

本次统一整理 Regime 1–4 共 8 个 notebook：每个 Regime 一份 5benchmark 和一份 7misspec / 7-condition constraint sensitivity 代码。

固定工程基线：

```r
BASE_DIR <- "/content/drive/MyDrive/CBN_simulation_2"
SEED_BASE <- 2026
```

随机数基准 2026 固定，不再改变。

## 2. 5benchmark 的统一设计

每个 Regime 的 5benchmark 运行：

```text
none
weak
strong
current_misspecified
graphical_lasso
```

其中 `current_misspecified` 现在统一按组会设计实现为：从当前 job 的 true DAG 中抽取 10% true edges，并将每条被抽中 true edge 的两个方向都加入 blacklist，再叠加当前 Regime 的 strong blacklist。

## 3. 7misspec / 7-condition sensitivity 的统一设计

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

明确删除所有 BL reverse 条件，包括 `BL_reversed`、`BL_within_reversed` 与 `BL_between_reversed`。

`BL_extra` 与 5benchmark 的 `current_misspecified` 对应：抽取 10% true edges，并将其两个方向都加入 blacklist。

Whitelist 采用比例化设计：

```r
WL_REFERENCE_RATE <- 0.25
WL_PERTURB_RATE   <- 0.25
```

因此不同图密度下保持相同的相对 whitelist coverage，避免固定 4 条 WL 导致稀疏图与稠密图先验信息量不一致。

## 4. 四个 Regime

| Regime | 节点数 | 主结构 | D block |
|---|---:|---|---|
| Regime 1 | 16 | A、B 并列进入 C，再到 Y | 无 |
| Regime 2 | 21 | Regime 1 + D -> Y | 有 |
| Regime 3 | 16 | A -> B -> C -> Y | 无 |
| Regime 4 | 21 | Regime 3 + D -> Y | 有 |

所有 Regime 都允许 within-block true edges，并通过 acyclicity check 保证无环。

## 5. 结果审计与复现

保留原 notebook 的：experiment grid、deterministic job table、`config_json` / `config_hash`、`SEED_BASE = 2026` 的 job-specific seed 派生、atomic write、RUNNING / COMPLETED / FAILED、断点续跑、current-only result rebuild、stale-result 隔离、one job one true DAG / one dataset / one `dat_disc`、one constraint realization per job × constraint level、one bootstrap strength table per constraint level，以及多个 tau 复用同一 strength table。

7misspec 额外保留 `constraint_changes/`，记录每次实际删除、增加或反转的 constraint edge。

## 6. 仓库结构

每个 Regime 文件夹包含 5 个文件：

```text
regimeX/
├── simulation0701_regimeX_5benchmark.ipynb
├── simulation0701_regimeX_7misspec.ipynb
├── README_5benchmark.md
├── README_7misspec.md
└── REGIME_DESIGN.md
```

总计：8 个 notebook + 12 个说明文档。

## 7. README 的结果使用方法

每个 5benchmark / 7misspec README 都包含“结果使用方法”，说明输出文件夹结构、如何确认 jobs 是否完整、`summary_results.csv` 与 `all_results_raw.csv` 分别做什么、哪些指标作为主要结果、如何按 `edge_multiplier × rho × tau` 的 8 个场景组织比较、如何从 `job_id` 追溯到 true DAG / constraint audit / constraint changes / averaged edges，以及如何组织最终论文结论。

## 8. 静态检查

8 个 notebook 已完成 JSON / nbformat 格式验证、工作目录检查、`SEED_BASE <- 2026` 检查、7misspec active levels 检查、确认 active design 不含 BL reverse，并清空 notebook 旧运行输出，避免历史 8misspec 文本混入当前版本。

当前环境没有 R / bnlearn，因此未声称完成真实 R runtime pilot。正式运行前仍建议每个新 notebook 先用小 pilot 验证。
