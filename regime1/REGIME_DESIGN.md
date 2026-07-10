# Regime 1 设计说明

## 1. 设计定位

**主设定：A 与 B 并列进入 C，再共同解释 Y。**

本文件只说明 Regime 1 的 true DAG、合法边空间、固定 core edges、strong constraint 和数据生成角色。5benchmark 与 7misspec 的运行和结果解释分别见同目录下 `README_5benchmark.md` 与 `README_7misspec.md`。

固定工程基线：

```r
BASE_DIR <- "/content/drive/MyDrive/CBN_simulation_2"
SEED_BASE <- 2026
```

## 2. 节点系统

| Block | 节点数 | 角色 |
|---|---:|---|
| A | 5 | 主要 exposure block |
| B | 5 | 已知对结局有强影响的 competing predictors |
| C | 5 | 中间机制或辅助变量 |
| Y | 1 | 研究结局 |

总节点数：**p = 16**。

## 3. Block-level true DAG

```text
A, B -> C -> Y
A -> Y
B -> Y
C -> Y
```

合法跨 block 方向：

- `A -> C`
- `B -> C`
- `A -> Y`
- `B -> Y`
- `C -> Y`

A 与 B 是并列上游 block；不允许 A -> B 或 B -> A。

所有 block 都允许 within-block true edges，但必须保持 DAG 无环。代码通过逐条加边并检查 acyclicity 实现。

## 4. 固定 core edges

Regime 1 每个 replicate 都共享下面的固定 core backbone，其余边根据 `EDGE_MULTS`、`WITHIN_TRUE_RATE` 和 `seed_dag` 随机补足。

| Core edge | 类型 |
|---|---|
| `A1 -> C1` | between-block core |
| `A2 -> C2` | between-block core |
| `B1 -> C3` | between-block core |
| `B2 -> C4` | between-block core |
| `A3 -> Y` | between-block core |
| `B3 -> Y` | between-block core |
| `B4 -> Y` | between-block core |
| `C1 -> Y` | between-block core |
| `C2 -> Y` | between-block core |
| `A1 -> A2` | within-block core |
| `B1 -> B2` | within-block core |
| `C3 -> C4` | within-block core |

固定 core edge 数：**12**。

## 5. True DAG 生成

目标边数沿用原代码命名：

```text
target_edges = p * edge_multiplier
```

正式场景：

| `edge_multiplier` | 目标边数 |
|---:|---:|
| 1 | 16 |
| 2 | 32 |

`WITHIN_TRUE_RATE <- 0.25` 控制 within-block true edges 的目标比例。生成顺序为：fixed core → within-block extra edges → legal between-block extra edges → 必要时从剩余合法候选补足，同时始终检查无环。

## 6. Strong constraint

`strong` 的含义不是 oracle DAG，也不是把 true edges 全部告诉算法。它只提供正确的 **block-level role-based blacklist**：

- within-block edge 保持可搜索；
- 上述合法跨 block 方向保持可搜索；
- 其他不符合 Regime 1 角色结构的跨 block 方向进入 blacklist。

因此，算法仍需自己判断哪些具体节点对之间真正存在边。

## 7. 数据生成与离散 BN 学习

每个 job 只生成一次 true DAG、一次数值数据和一次 `dat_disc`。同一份 `dat_disc` 被所有 constraint conditions 复用。结构学习继续使用：

```text
algorithm = hc
score = bde
iss = 10
```

`rho` 沿用原参数名，用于控制共同因子造成的共线性强弱。Regime 1 中 A/B 共享既有 common-factor 结构。

## 8. 四个 Regime 的位置

| Regime | 节点数 | 主结构 | D block | 主要比较意义 |
|---|---:|---|---|---|
| Regime 1 | 16 | A、B 并列进入 C | 无 | 主设定 |
| Regime 2 | 21 | Regime 1 + D -> Y | 有 | 额外直接风险压力 |
| Regime 3 | 16 | A -> B -> C 链式 | 无 | 链式替代机制 |
| Regime 4 | 21 | Regime 3 + D -> Y | 有 | 链式复杂压力测试 |

## 9. 对比 Regime 时的结论组织

优先做成配对比较：

- **Regime 1 vs Regime 2**：并列上游结构固定，只考察增加 D -> Y 后 constraint 收益是否变化。
- **Regime 3 vs Regime 4**：链式结构固定，只考察增加 D -> Y 后的变化。
- **Regime 1 vs Regime 3**：比较并列上游与链式机制。
- **Regime 2 vs Regime 4**：在都有 D block 时比较并列与链式机制。

由于 Regime 1/3 的 `p=16`，Regime 2/4 的 `p=21`，跨不同节点规模比较时应明确说明维度差异；最干净的主比较仍是 1 vs 3 和 2 vs 4。
