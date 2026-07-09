# Regime 1–4 约束贝叶斯网络模拟实验技术设计与执行计划

**版本：V1.0**  
**仓库：** `wrqkkk/BMC-MRM-simulation`  
**参考代码：** `simulation0618_regime1.ipynb`  
**文档状态：** 正式代码实现前的技术规格基线

---

## 1. 文档目的

本文档用于定义新的 **Regime 1–4 约束贝叶斯网络模拟实验**的完整技术方案。

后续代码实现必须以当前已有的 `simulation0618_regime1.ipynb` 为工程基线，保留其已有的参数命名、job table、随机种子、配置哈希、断点续跑、原子写入、批量状态扫描、结果重建和去重机制；主要替换以下内容：

1. 原有 true DAG 的层级结构；
2. 原有基于旧层级关系生成 constraint 的逻辑；
3. 原有部分结果指标，补充 Zhao 和 Jia（2025）文章中使用的 DAG 结构恢复指标；
4. 新增 Regime 1–4；
5. 新增每次实际使用的 blacklist / whitelist constraint 边的完整保存与追溯功能。

本项目的核心目标不是比较多个新算法谁更强，而是研究：

> **在同一个经典离散贝叶斯网络结构学习框架下，none、weak、strong 和 misspecified constraints 会带来什么收益、什么成本，以及错误先验会造成什么后果。**

---

## 2. 项目目标与研究问题

### 2.1 核心研究问题

本项目研究 **domain-informed structural constraints** 对 Bayesian network 结构学习的影响。

重点回答以下问题：

1. 正确的 constraint 是否能够提高结构恢复精度；
2. strong constraint 是否主要通过减少不合理反向边而改善 directed structure；
3. constraint 是否会提高 precision，但同时牺牲 recall；
4. 错误设定的 constraint 会损伤哪些真实边；
5. misspecified constraint 是否会迫使模型通过错误替代路径解释数据；
6. constraint 的收益和成本是否随图密度、相关程度、bootstrap threshold 和 true DAG regime 改变。

### 2.2 比较对象

主实验延续现有代码中的 benchmark 逻辑，比较：

```text
none
weak
strong
current_misspecified
graphical_lasso
```

其中：

- `none`：无结构先验；
- `weak`：只加入最弱、最可靠的方向约束；
- `strong`：根据当前 Regime 的 block-level 结构加入完整方向先验；
- `current_misspecified`：在 strong constraint 基础上故意错误禁止部分 true edges；
- `graphical_lasso`：作为无向 skeleton benchmark，在适用时保留。

---

## 3. 现有代码基线

### 3.1 保留现有离散 BN 学习流程

当前 `simulation0618_regime1.ipynb` 已经使用离散 BN 学习流程。

每个 job 的基本顺序是：

```text
make_true_dag()
    ↓
generate_data_from_dag()
    ↓
discretize_for_bn()
    ↓
dat_disc
    ↓
bnlearn::boot.strength()
```

真正进入 Bayesian network 结构学习的是：

```text
dat_disc
```

学习方法继续使用：

```text
algorithm = "hc"
score = "bde"
iss = 10
```

除非后续另行做出正式设计变更，否则这部分不改变。

### 3.2 必须保留的工程机制

新 Regime 1–4 代码必须继续保留以下已有功能：

1. experiment grid；
2. job table；
3. `config_json`；
4. `config_hash`；
5. deterministic `job_id`；
6. job-specific random seeds；
7. 每个 job 只生成一次 true DAG；
8. 每个 job 只生成一次数据；
9. 每个 constraint level 只跑一次 bootstrap strength；
10. 同一 strength table 复用于多个 `tau`；
11. atomic write；
12. `RUNNING` / `COMPLETED` / `FAILED` 状态文件；
13. 断点续跑；
14. current-job-only result rebuild；
15. 结果去重与 stale-result 隔离；
16. batch progress 持续保存。

---

## 4. 变量分组与节点规模

### 4.1 Block 定义

新 true DAG 使用 role-based blocks：

| Block | 节点数 | 含义 |
|---|---:|---|
| `A` | 5 | 环境暴露组学变量，即主要 exposure block |
| `B` | 5 | 领域内已知对结局有强影响的竞争性解释变量 |
| `C` | 5 | 其他中间机制变量或辅助变量 |
| `D` | 5 | 可选额外直接风险变量 |
| `Y` | 1 | 研究结局 |

节点命名建议为：

```text
A1, A2, A3, A4, A5
B1, B2, B3, B4, B5
C1, C2, C3, C4, C5
D1, D2, D3, D4, D5
Y
```

### 4.2 节点总数

按照当前 Regime 定义：

```text
Regime 1: A + B + C + Y = 16 nodes
Regime 2: A + B + C + D + Y = 21 nodes
Regime 3: A + B + C + Y = 16 nodes
Regime 4: A + B + C + D + Y = 21 nodes
```

现有代码中的字段名 `p` 必须保留。

现有可编辑参数名 `P_NODES` 也不随意改名。对于多 Regime job table，最终每一行 job 中保存的 `p` 应由当前 `regime_type` 对应的节点系统决定。

---

## 5. Regime 1–4 的 true DAG 设计

### 5.1 Regime 1：主设定，A 与 B 并列进入 C

Regime 1 是当前最自然的主设定：

```text
A, B -> C -> Y
A -> Y
B -> Y
C -> Y
```

对应合法核心 block-level relations：

```text
A -> C
B -> C
A -> Y
B -> Y
C -> Y
```

设计含义：

- `A` 是主要环境暴露 block；
- `B` 是已知强竞争变量 block；
- `A` 与 `B` 作为并列上游来源；
- `A` 和 `B` 均可通过 `C` 影响 `Y`；
- `A`、`B`、`C` 均允许直接影响 `Y`；
- Regime 1 不默认存在 `A -> B` 或 `B -> A`。

### 5.2 Regime 2：Regime 1 + D 直接风险层

结构为：

```text
A, B -> C -> Y
A -> Y
B -> Y
C -> Y
D -> Y
```

Regime 2 在 Regime 1 基础上加入：

```text
D -> Y
```

`D` 代表额外直接风险因素，不属于主要的 `A/B -> C -> Y` 机制链。

### 5.3 Regime 3：链式替代设定

结构为：

```text
A -> B -> C -> Y
A -> Y
B -> Y
C -> Y
```

对应核心 block-level relations：

```text
A -> B
B -> C
A -> Y
B -> Y
C -> Y
```

该 Regime 用于模拟逐级传导结构。

### 5.4 Regime 4：Regime 3 + D 直接风险层

结构为：

```text
A -> B -> C -> Y
A -> Y
B -> Y
C -> Y
D -> Y
```

Regime 4 是 Regime 3 的扩展版。

---

## 6. 层内边设计

### 6.1 所有 Regime 都允许层内边

四个 Regime 均允许 block 内部存在 true edges，例如：

```text
A2 -> A4
B1 -> B5
C3 -> C4
D1 -> D3
```

层内边用于模拟：

- 暴露变量之间的复杂相关结构；
- 强竞争变量之间的依赖；
- 中间变量之间的内部机制；
- D block 内部的风险结构。

### 6.2 DAG 无环保证

层内边不能任意双向生成。

建议每个 block 先生成局部 topological order，例如：

```text
A3 < A1 < A5 < A2 < A4
```

只允许从顺序靠前节点指向顺序靠后节点。

这样既允许层内结构，又保证不会形成 cycle。

### 6.3 保留 `WITHIN_TRUE_RATE`

现有参数名：

```text
WITHIN_TRUE_RATE
```

必须保留。

新代码中，它表示 true DAG 目标边中 within-block edges 的目标比例。

不能因为旧代码是 tier、新代码是 block，就随意改参数名。

---

## 7. True DAG 生成逻辑

### 7.1 总体原则

新的 `make_true_dag()` 应继续保留原有思路：

```text
固定 core edges
+
随机 extra edges
+
目标边数
+
无环约束
+
确定性随机种子
```

### 7.2 目标边数

继续沿用现有定义：

```text
target_edges = p * edge_multiplier
```

参数名保持：

```text
EDGE_MULTS
edge_multiplier
target_edges
```

不改名。

### 7.3 生成步骤

每个 job 的 true DAG 建议按以下顺序生成：

```text
1. 根据 regime_type 创建节点与 block annotation
2. 读取当前 Regime 的合法 block-level direction
3. 插入当前 Regime 的固定 core edges
4. 生成 within-block candidate edges
5. 生成合法 between-block candidate edges
6. 根据 WITHIN_TRUE_RATE 控制层内边比例
7. 按 target_edges 补充 extra edges
8. 每加一条边检查 DAG acyclicity
9. 达到目标边数后保存 true DAG
```

### 7.4 固定 core edges

正式实验仍应保留“不同 rep 共享一组 core edges”的设计。

例如同一个 Regime 的 100 张 true DAG 可以共享一套固定骨架，同时其他 extra edges 随 `rep` 变化。

这样可以支持：

- representative DAG 分析；
- consensus edge 分析；
- 检查哪些 core edges 能够在大量 runs 中稳定恢复。

### 7.5 True edge annotation

每条 true edge 至少保存：

```text
from
to
from_block
to_block
edge_role
edge_origin
is_within_layer
is_between_layer
```

其中：

- `edge_role`：例如 `A_to_C`、`B_to_Y`、`within_A`；
- `edge_origin`：例如 `core` 或 `extra`。

---

## 8. 参数命名规则

### 8.1 原有参数名必须保留

不得为了“更好看”而重新命名已有参数。

| 原参数名 | 新实验中的含义 |
|---|---|
| `N_VALUES` | 样本量集合 |
| `P_NODES` | 固定 Regime 运行时的节点数参数 |
| `EDGE_MULTS` | `target_edges = p * edge_multiplier` 的 multiplier |
| `RHO_VALUES` | 保留原代码中的 `rho` 参数及其数据生成含义 |
| `REP_VALUES` | 独立 simulation replicate / true DAG job |
| `B_BOOT_VALUE` | `boot.strength()` 的 bootstrap 次数 |
| `THRESHOLDS` | bootstrap strength threshold 集合 |
| `MISSPEC_RATE` | `current_misspecified` 中错误禁止 true edges 的比例 |
| `WITHIN_TRUE_RATE` | true DAG 中 within-block edges 的目标比例 |
| `WL_MISSING_N` | whitelist misspecification 参数 |
| `WL_EXTRA_N` | whitelist misspecification 参数 |
| `WL_REVERSED_N` | whitelist misspecification 参数 |
| `DIRECTION_THRESHOLD` | bootstrap average 后方向判定阈值 |

### 8.2 新增参数

真正需要新增的核心字段只有：

```text
regime_type
```

用于区分：

```text
regime1
regime2
regime3
regime4
```

---

## 9. Job table 与可复现性

### 9.1 每个 job 的身份

继续使用 `make_job_config()` 构建完整配置。

当前代码中的种子体系保留：

```text
seed_dag
seed_data
seed_constraint
seed_boot
```

当前逻辑为：

```text
seed_dag = SEED_BASE + rep + 1000*n + 100*edge_multiplier + 10*round(rho*10)
seed_data = seed_dag + 1
seed_constraint = seed_dag + 2
seed_boot = seed_dag + 3
```

新代码需要确保 `regime_type` 进入完整配置和 hash 计算，否则不同 Regime 在其他参数相同时可能发生配置冲突。

### 9.2 `config_hash`

继续使用：

```text
json_for_hash()
make_config_hash()
SHA-256
```

完整链路为：

```text
job config
    ↓
config_json
    ↓
SHA-256
    ↓
config_hash
    ↓
job_id
```

作用是防止：

> 参数已经改变，但旧结果文件仍然被错误识别为当前实验已完成结果。

### 9.3 `job_id`

`job_id` 应继续保持 deterministic。

它应能够唯一标识：

- experiment；
- regime；
- `n`；
- `p`；
- `edge_multiplier`；
- `rho`；
- `rep`；
- misspecification 设置；
- `B_BOOT`；
- `DIRECTION_THRESHOLD`；
- `config_hash`。

---

## 10. 一次生成，多次复用

这是整个实验设计中最重要的效率与公平性机制之一。

### 10.1 每个 job 只生成一次 true DAG

```text
one job
    ↓
one true DAG
```

不同 constraint level 不重新生成 true DAG。

### 10.2 每个 job 只生成一次数据

当前 `run_one_job()` 的逻辑应继续保持：

```text
make_true_dag()
    ↓
generate_data_from_dag()
    ↓
dat_num
    ↓
discretize_for_bn()
    ↓
dat_disc
```

之后同一份 `dat_disc` 用于：

```text
none
weak
strong
current_misspecified
```

不能在 constraint loop 内重新生成数据。

### 10.3 每个 constraint level 只跑一次 bootstrap

对于一个固定 job：

```text
constraint_level
    ↓
make_constraint_set()
    ↓
run_bn_boot_strength()
    ↓
one strength table
```

这张 strength table 复用于所有 `tau`。

例如：

```text
THRESHOLDS = 0.4;0.8
```

应执行：

```text
one bootstrap strength table
    ├── tau = 0.4 -> average_edges()
    └── tau = 0.8 -> average_edges()
```

不能因为 `tau` 改变就重新跑 bootstrap。

### 10.4 公平比较结构

最终层级必须是：

```text
one job
    ├── one true DAG
    ├── one generated dataset
    └── one dat_disc

        ├── none
        │   └── one bootstrap strength table
        │       ├── tau = 0.4
        │       └── tau = 0.8
        │
        ├── weak
        │   └── one bootstrap strength table
        │       ├── tau = 0.4
        │       └── tau = 0.8
        │
        ├── strong
        │   └── one bootstrap strength table
        │       ├── tau = 0.4
        │       └── tau = 0.8
        │
        └── current_misspecified
            └── one fixed misspecified constraint realization
                └── one bootstrap strength table
                    ├── tau = 0.4
                    └── tau = 0.8
```

---

## 11. Constraint 设计

### 11.1 `none`

```text
blacklist = empty
whitelist = empty
```

不加入任何结构先验。

### 11.2 `weak`

保留最弱方向知识：

```text
Y -> non-Y nodes is forbidden
```

即禁止：

```text
Y -> A
Y -> B
Y -> C
Y -> D
```

### 11.3 `strong`

`strong` 改为 regime-aware constraint。

核心原则：

> **禁止不符合当前 Regime 的跨 block 反向边，但允许 within-block edges。**

#### Regime 1

合法主方向：

```text
A -> C
B -> C
A -> Y
B -> Y
C -> Y
```

#### Regime 2

在 Regime 1 基础上增加：

```text
D -> Y
```

#### Regime 3

合法主方向：

```text
A -> B
B -> C
A -> Y
B -> Y
C -> Y
```

#### Regime 4

在 Regime 3 基础上增加：

```text
D -> Y
```

### 11.4 `current_misspecified`

继续沿用现有主 benchmark 的逻辑：

```text
strong blacklist
    +
从当前 job 的 true DAG 中抽取部分 true edges
    +
错误加入 blacklist
```

抽取比例由：

```text
MISSPEC_RATE
```

控制。

因此：

- misspecified constraint 是 job-specific；
- 每个 true DAG 对应自己的一次 misspecified 抽样；
- 不同 rep 的 misspecified 边可以不同；
- 同一个 job 内，misspecified 边固定；
- 不能在每个 bootstrap replicate 重新抽；
- 不能在不同 `tau` 下重新抽。

---

## 12. Constraint 边审计与完整溯源

### 12.1 必须保存每次实际使用的 constraint

这是新版本的硬性要求。

执行顺序必须是：

```text
cons <- make_constraint_set(...)
    ↓
保存 exact blacklist / whitelist
    ↓
run_bn_boot_strength(...)
```

也就是说，constraint 必须在真正进入 BN 学习前被完整保存。

### 12.2 输出目录

继续使用：

```text
constraint_audits/
wl_references/
```

建议文件名：

```text
constraint_audits/<job_id>__<constraint_level>__constraint_audit.csv
wl_references/<job_id>__<constraint_level>__whitelist_reference.csv
```

### 12.3 每条 constraint edge 的字段

至少保存：

| 字段 | 含义 |
|---|---|
| `job_id` | 对应哪个 job |
| `config_hash` | 配置身份 |
| `experiment_id` | 实验线 |
| `regime_type` | 当前 Regime |
| `constraint_level` | none / weak / strong / current_misspecified |
| `constraint_type` | blacklist / whitelist / none |
| `from` | 起点 |
| `to` | 终点 |
| `from_block` | 起点所属 block |
| `to_block` | 终点所属 block |
| `edge_role` | 如 A_to_C、B_to_Y、within_A |
| `constraint_detail` | 约束说明 |
| `seed_constraint` | constraint 随机种子 |
| `is_true_edge` | 是否命中当前 true DAG 的真实边 |
| `is_reversed_true_edge` | 是否是真实边的反向 |
| `is_within_layer` | 是否为层内边 |
| `is_between_layer` | 是否为层间边 |
| `is_misspecified_edge` | 是否为故意错误加入的 constraint edge |

### 12.4 `none` 也必须留下审计记录

对于 `none`，仍然应生成明确记录：

```text
n_blacklist_constraints = 0
n_whitelist_constraints = 0
```

这样可以区分：

```text
确实没有 constraint
```

和：

```text
程序失败，所以 constraint 文件没有生成
```

### 12.5 结果溯源链

最终应能够实现：

```text
某张图或某个结果
    ↓
job_id + constraint_level
    ↓
row_results
    ↓
averaged_edges / estimated_edges
    ↓
constraint_audits
    ↓
该次实际使用的 exact blacklist / whitelist
```

---

## 13. Bootstrap 学习与 tau 复用

### 13.1 学习核心

继续使用：

```r
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

默认保持：

```text
algorithm = "hc"
score = "bde"
iss = 10
```

### 13.2 `average_edges()` 去重逻辑

每个无序节点对最多保留一个方向。

例如：

```text
A1 -> C2
C2 -> A1
```

先统一成 pair key：

```text
A1__C2
```

然后在该 pair 内选择一个最终方向。

不能让最终 averaged graph 同时出现两个相反方向。

### 13.3 `DIRECTION_THRESHOLD`

保留现有参数名：

```text
DIRECTION_THRESHOLD
```

继续用于 bootstrap average 后的方向多数判定。

---

## 14. 评价指标

### 14.1 参考 Zhao 和 Jia（2025）

正式报告指标参考：

> Zhao Y, Jia J. DAGSLAM: causal Bayesian network structure learning of mixed type data and its application in identifying disease risk factors. BMC Medical Research Methodology. 2025;25:154.

主指标为：

```text
FDR
TPR
FPR
SHD
F1
```

同时报告：

```text
Directed structure
Skeleton structure
```

### 14.2 Directed metrics

定义：

```text
TP = 估计出的边方向与 true DAG 完全一致
R  = skeleton 存在，但方向与 true DAG 相反
FP = true skeleton 中不存在该边
P  = 估计出的 directed edges 数
T  = true directed edges 数
F  = false/non-edge possibilities
E  = estimated graph 中多出的 skeleton edges
M  = estimated graph 中缺失的 true skeleton edges
```

公式：

```text
FDR_directed = (R + FP) / P
TPR_directed = TP / T
FPR_directed = (R + FP) / F
SHD_directed = E + M + R
F1_directed = 2 * (1 - FDR_directed) * TPR_directed /
              ((1 - FDR_directed) + TPR_directed)
```

### 14.3 与 precision / recall 的关系

```text
precision = 1 - FDR
recall = TPR
```

所以：

- FDR 越低越好；
- TPR 越高越好；
- FPR 越低越好；
- SHD 越低越好；
- F1 越高越好。

### 14.4 Skeleton metrics

同时计算：

```text
FDR_skeleton
TPR_skeleton
FPR_skeleton
SHD_skeleton
F1_skeleton
```

### 14.5 保留旧指标

现有字段可以继续保留，例如：

```text
directed_precision
directed_recall
directed_f1
skeleton_precision
skeleton_recall
skeleton_f1
reversal_rate
backward_edge_rate
ordered_true_recall
reversible_true_recall
n_est_edges
```

这些指标用于补充解释 constraint 的收益和成本。

### 14.6 后续可扩展的 role-aware metrics

后续可以增加：

```text
between-layer directed F1
between-layer skeleton F1
within-layer skeleton F1
A-related edge recovery
B-related edge recovery
C-to-Y recovery
D-to-Y recovery
```

这些作为扩展指标，不替代标准 overall FDR / TPR / FPR / SHD / F1。

---

## 15. 输出目录结构

建议继续保持按 artifact 类型分目录：

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

不同 experiment line 必须使用不同 `OUT_DIR`。

不能因为文件都位于同一个 parent directory，就把不同实验版本结果混在一起。

---

## 16. Atomic write：避免半写文件被当作正式结果

### 16.1 当前机制

继续使用：

```text
atomic_write_csv()
atomic_write_json()
```

基本流程：

```text
写入 <target>.tmp
    ↓
写入完成
    ↓
删除旧 target（如有必要）
    ↓
rename .tmp -> 正式 target
```

### 16.2 为什么需要 atomic write

它用于防止：

- Colab 中途断开；
- runtime crash；
- Google Drive 写入中断；
- CSV 只写了一半却被后续程序认为是完整结果。

### 16.3 新 artifact 也必须使用 atomic write

包括：

```text
constraint_audits
wl_references
row_results
metadata
progress logs
summary results
```

后处理文件搜索必须继续排除：

```text
*.tmp
```

---

## 17. 断点续跑机制

### 17.1 Job 状态机

继续使用 `inspect_job_status()`。

| `run_status` | 判断条件 | `recommended_action` |
|---|---|---|
| `completed_current` | result、metadata、completed flag 均存在，且 `config_hash` 匹配 | `skip` |
| `config_mismatch` | 文件存在，但 metadata hash 与当前配置不同 | `manual_check_or_rerun` |
| `result_without_metadata` | result 存在，但 metadata 缺失 | `manual_check_or_rerun` |
| `interrupted_or_incomplete` | RUNNING flag 存在，但 COMPLETED flag 不存在 | `rerun` |
| `pending` | 没有完整结果 | `run` |

### 17.2 实际执行条件

主循环只选择：

```text
RUN_ACTIONS <- c("run", "rerun")
```

因此：

- 已完成且配置匹配的 job 自动跳过；
- 中断 job 自动重跑；
- pending job 正常运行；
- 配置冲突 job 不自动静默覆盖。

### 17.3 Job 开始与完成标记

Job 开始时写：

```text
<job_id>_RUNNING.json
```

成功完成后写：

```text
<job_id>.csv
<job_id>_COMPLETED.json
```

然后删除 running flag。

### 17.4 为什么不能只看 result 文件是否存在

一个 job 只有同时满足以下条件才算真正完成：

```text
result exists
+
metadata exists
+
COMPLETED flag exists
+
metadata config_hash == current config_hash
```

仅有 CSV 文件不能证明它是当前配置下的完整结果。

---

## 18. 单个 job 失败时不中断整个 batch

### 18.1 `tryCatch()` 隔离错误

主循环继续用 `tryCatch()` 包裹：

```text
run_one_job()
```

如果一个 job 失败：

```text
写 <job_id>_FAILED.json
记录 error
继续下一个 job
```

不能因为一个 job 报错导致整个几百个 job 的 batch 终止。

### 18.2 持续保存 batch progress

每处理完一个 job，继续写：

```text
logs/batch_progress_<timestamp>.csv
```

这样即使 Colab 之后断开，也可以知道已经处理到哪里。

---

## 19. Fresh status rescan

### 19.1 为什么需要重新扫描

Notebook 内存中的 `run_status` 可能已经过时。

因此完成或中断一轮运行后，应重新读取文件系统状态。

### 19.2 重新扫描步骤

先删除旧内存列：

```text
run_status
recommended_action
status_detail
```

然后重新对所有当前 job 执行：

```text
inspect_job_status()
```

得到新的 status table。

这样显示的是当前 `OUT_DIR` 真正存在的文件状态，而不是旧的 notebook 内存状态。

---

## 20. 去重与 stale-result 隔离

这里包含多种不同的“去重”，必须分别处理。

### 20.1 配置级去重

使用：

```text
config_json
    ↓
config_hash
    ↓
job_id
```

不同配置不会静默共用同一个 job 身份。

### 20.2 Edge-level 去重

true edges 与 constraint edges 使用：

```text
distinct(from, to)
```

同时删除：

```text
self-loop
```

### 20.3 Averaged edge 去重

对每个无序节点对建立 pair key：

```text
paste(pmin(from, to), pmax(from, to), sep = "__")
```

例如：

```text
A1 -> C2
C2 -> A1
```

统一映射为：

```text
A1__C2
```

最终最多保留一个方向。

### 20.4 每个 job 只有一个 row result 文件

固定路径：

```text
row_results/<job_id>.csv
```

当：

```text
OVERWRITE_EXISTING <- FALSE
```

且 job 状态为：

```text
completed_current
```

则自动跳过，不重新计算。

### 20.5 只从当前 job table 重建结果

继续使用：

```text
rebuild_results_from_rows(
    current_only = TRUE,
    job_table_ref = job_table
)
```

该函数不能无脑读取 `row_results/` 里的所有历史 CSV。

应执行：

```text
当前 job_table
    ↓
生成当前应有的 result paths
    ↓
只读取真实存在的当前 result files
    ↓
排除 .tmp
    ↓
再次过滤到当前 job_id
```

这样避免旧实验残留结果污染当前 summary。

### 20.6 增加显式唯一键检查

建议新代码增加 uniqueness audit。

主 BN 结果逻辑唯一键至少包括：

```text
job_id
+
method
+
constraint_level
+
tau
```

如果出现意外重复，应报错或生成 audit，而不是简单使用：

```text
distinct()
```

把问题静默删除。

---

## 21. 结果重建与汇总

### 21.1 Raw results

继续生成：

```text
all_results_raw.csv
```

它保存当前 job table 对应的所有有效结果行。

### 21.2 Summary results

继续生成：

```text
summary_results.csv
```

分组字段至少保留：

```text
regime_type
edge_multiplier
rho
tau
within_true_rate
B_BOOT
constraint_level
misspecification settings
```

### 21.3 汇总指标

至少汇总：

```text
Directed:
FDR
TPR
FPR
SHD
F1

Skeleton:
FDR
TPR
FPR
SHD
F1
```

建议同时保存：

```text
mean
sd
n_replicates
```

---

## 22. Constraint realization 的固定与复用

### 22.1 每个 job × constraint level 只有一个 constraint realization

对于一个 job：

```text
none -> one constraint set
weak -> one constraint set
strong -> one constraint set
current_misspecified -> one sampled misspecified constraint set
```

### 22.2 Misspecified edges 不可反复重抽

`current_misspecified` 中抽到的错误 blacklist edges：

- 由当前 true DAG 决定候选 true edges；
- 由 `seed_constraint` 决定抽样；
- 在该 job 内固定；
- 所有 bootstrap replicate 共享；
- 所有 `tau` 共享。

### 22.3 完整层级

```text
one job
    -> one true DAG
    -> one generated dataset
    -> one dat_disc

    -> none: one constraint set
    -> weak: one constraint set
    -> strong: one constraint set
    -> current_misspecified: one misspecified constraint set

        -> one bootstrap strength table per constraint level
            -> multiple tau-specific averaged DAGs
```

---

## 23. Pilot 执行计划

### 23.1 Pilot 目的

第一版代码完成后不能直接运行正式 `1:100` replicates。

Pilot 用于验证：

1. 四个 Regime 是否生成正确结构；
2. p 是否正确；
3. DAG 是否无环；
4. `target_edges` 是否达到目标；
5. within-block edges 是否真正允许；
6. strong constraint 是否没有误禁所有层内边；
7. 同一 job 是否只生成一次数据；
8. 同一 constraint level 是否只跑一次 bootstrap；
9. `tau` 是否复用同一 strength table；
10. misspecified 是否从当前 true DAG 抽 true edges；
11. 每次实际使用的 constraint 是否保存；
12. 断点续跑是否正常；
13. current-only result rebuild 是否正常；
14. Zhao–Jia 指标是否计算正确。

### 23.2 Pilot 参数

Pilot 应使用和正式实验完全相同的代码路径，只缩小参数规模。

例如：

```text
REP_VALUES = small subset
B_BOOT_VALUE = small value
```

不应另写一套 pilot-only pipeline。

---

## 24. 正式实验计划

继续沿用已经建立的主参数口径：

```text
N_VALUES <- "5000"
EDGE_MULTS <- "1;2"
RHO_VALUES <- "0.2;0.8"
REP_VALUES <- "1:100"
B_BOOT_VALUE <- 200
THRESHOLDS <- "0.4;0.8"
MISSPEC_RATE <- 0.10
WITHIN_TRUE_RATE <- 0.25
DIRECTION_THRESHOLD <- 0.5
```

新增：

```text
regime_type = regime1 / regime2 / regime3 / regime4
```

最终正式结果应支持按以下维度汇总：

```text
regime_type
× edge_multiplier
× rho
× tau
× constraint_level
× rep
```

---

## 25. Notebook 使用方法

### 25.1 第一次运行：只检查配置，不跑 job

先设置：

```text
RUN_JOBS <- FALSE
```

运行：

```text
setup
utilities
experiment grid
job table
status scan
```

检查：

```text
experiment_grid_current.csv
job_table_current.csv
status summary
```

确认参数组合正确后再正式运行。

### 25.2 正式运行

设置：

```text
RUN_JOBS <- TRUE
OVERWRITE_EXISTING <- FALSE
```

这样：

- 当前已完成 job 自动跳过；
- 中断 job 自动重跑；
- pending job 正常运行。

### 25.3 Colab 中断后恢复

重新运行：

```text
setup
utilities
experiment grid
job table
status scan
```

系统重新判断：

```text
completed_current -> skip
interrupted_or_incomplete -> rerun
pending -> run
```

无需手动记住跑到第几个 job。

### 25.4 运行完成后

执行 fresh status rescan，然后运行：

```text
rebuild_results_from_rows(
    current_only = TRUE,
    job_table_ref = job_table
)
```

输出：

```text
all_results_raw.csv
summary_results.csv
```

---

## 26. 实现阶段

| 阶段 | 主要工作 | 完成标准 |
|---|---|---|
| 1 | 核对现有 notebook 架构 | 参数名、路径、状态机制全部梳理清楚 |
| 2 | 新增 A/B/C/D/Y 节点系统 | 节点数与 block annotation 正确 |
| 3 | 实现 Regime 1–4 true DAG | 四个 Regime 均无环、结构正确 |
| 4 | 重写 `make_true_dag()` | 保留 core + extra + `EDGE_MULTS` + `WITHIN_TRUE_RATE` |
| 5 | 重写 regime-aware `make_constraint_set()` | none / weak / strong / current_misspecified 正确 |
| 6 | 新增 constraint audit | 每次实际 blacklist / whitelist 都可追溯 |
| 7 | 新增 Zhao–Jia metrics | directed 和 skeleton 的 FDR / TPR / FPR / SHD / F1 正确 |
| 8 | 验证断点续跑 | completed job 不重复跑，中断 job 可恢复 |
| 9 | 验证去重和 stale-result 隔离 | 旧文件不污染当前结果 |
| 10 | Pilot | 四个 Regime 小规模完整跑通 |
| 11 | Full run | 正式参数运行 |
| 12 | Post-processing | 生成主表、结果图、代表 DAG、constraint audit summary |

---

## 27. V1 验收标准

V1 代码只有同时满足以下条件才算完成。

### 27.1 True DAG

- A = 5；
- B = 5；
- C = 5；
- D = 5；
- Y = 1；
- Regime 1 和 3 使用 16 个节点；
- Regime 2 和 4 使用 21 个节点；
- 四个 Regime 结构正确；
- 所有 true DAG 无环；
- within-block edges 被允许；
- strong constraint 不自动禁止所有 within-block edges。

### 27.2 数据与 bootstrap 复用

- 每个 job 只生成一次 true DAG；
- 每个 job 只生成一次 `dat_num`；
- 每个 job 只生成一次 `dat_disc`；
- 同一个 `dat_disc` 用于所有 constraint levels；
- 每个 constraint level 只生成一次 constraint set；
- 每个 constraint level 只跑一次 bootstrap strength；
- 多个 `tau` 复用同一 strength table。

### 27.3 Constraint audit

- 每个 job × constraint level 都有明确审计记录；
- exact blacklist 被保存；
- exact whitelist 被保存；
- misspecified 抽到的 true edges 被明确标记；
- `none` 也留下零 constraint 记录；
- row result 能追溯到 constraint audit 文件。

### 27.4 断点续跑

- 已完成且 hash 匹配的 job 自动 skip；
- 中断 job 自动 rerun；
- config mismatch 不静默覆盖；
- result-only 文件不被误判为正式完成；
- critical outputs 使用 atomic write。

### 27.5 去重

- true edges 无重复；
- constraint edges 无重复；
- self-loop 被删除；
- averaged graph 每个 unordered pair 最多保留一个方向；
- current-only rebuild 不读取旧实验无关结果；
- 逻辑唯一键出现异常重复时产生 audit 或错误。

### 27.6 指标

正式输出至少包含：

```text
Directed:
FDR
TPR
FPR
SHD
F1

Skeleton:
FDR
TPR
FPR
SHD
F1
```

---

## 28. V1 边界

当前文档冻结的是：

- 工程架构；
- 四个 Regime 的 block-level 结构；
- A/B/C/D/Y 节点数；
- 参数命名；
- 断点续跑；
- 一次生成、多次复用；
- constraint audit；
- 结果去重；
- Zhao–Jia 指标体系。

V1 暂不最终冻结每一条具体 fixed core arc，例如：

```text
A1 -> C1
A2 -> C2
B1 -> C3
```

具体 core edge template 应在正式改代码阶段单独确认，然后再开始 full run。

V1 首先实现主 benchmark 和 constraint provenance。现有更详细的 8-misspecification 体系可以保留代码接口，但应在四个 Regime 主 pipeline 通过 pilot 后再继续扩展。

---

## 29. 总结

新的 Regime 1–4 实验继续复用 `simulation0618_regime1.ipynb` 已经建立的工程框架：

```text
editable experiment parameters
    ↓
experiment grid
    ↓
job table
    ↓
deterministic seeds
    ↓
config hash
    ↓
job-specific true DAG and data
    ↓
constraint-specific bootstrap learning
    ↓
tau-specific averaged DAG
    ↓
per-job results
    ↓
current-only result rebuild
    ↓
summary results
```

新的科学设计集中在四部分：

1. 用 A/B/C/D/Y role-based Regime 1–4 替代旧 true DAG 层级结构；
2. 允许 within-block edges；
3. 使用 regime-aware strong constraint；
4. 保存每次实际使用的 exact blacklist / whitelist，并用 Zhao 和 Jia（2025）的 FDR、TPR、FPR、SHD、F1 指标评价 directed 和 skeleton structure。

最终目标是保证：

> **每一张图、每一行结果、每一个 constraint、每一次 misspecified 抽样，都能够从 summary 追溯到 job、true DAG、数据配置、bootstrap 输出和实际使用的 blacklist / whitelist。**
