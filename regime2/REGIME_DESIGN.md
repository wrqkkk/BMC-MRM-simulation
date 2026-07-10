# Regime 2 设计说明

## 1. 设计定位

**Regime 1 + D 直接风险 block，用于检验额外强直接风险因素是否改变 constraint 收益。**

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
| D | 5 | 额外 direct-risk block |
| Y | 1 | 研究结局 |

总节点数：**p = 21**。

## 3. Block-level true DAG

```text
A, B -> C -> Y
A -> Y
B -> Y
C -> Y
D -> Y
```

合法跨 block 方向：`A -> C`、`B -> C`、`A -> Y`、`B -> Y`、`C -> Y`、`D -> Y`。

A 与 B 是并列上游；D 是 direct-risk block。除 `D -> Y` 外，不允许 D 与 A/B/C 发生跨 block 连接。所有 block 均允许 within-block true edges，并通过 acyclicity check 保证无环。

## 4. 固定 core edges

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
| `D1 -> Y` | between-block core |
| `A1 -> A2` | within-block core |
| `B1 -> B2` | within-block core |
| `C3 -> C4` | within-block core |
| `D1 -> D2` | within-block core |

固定 core edge 数：**14**。

## 5. True DAG 生成

```text
target_edges = p * edge_multiplier
```

| `edge_multiplier` | 目标边数 |
|---:|---:|
| 1 | 21 |
| 2 | 42 |

`WITHIN_TRUE_RATE <- 0.25` 控制 within-block true edges 的目标比例。生成顺序为 fixed core → within-block extras → legal between-block extras → 必要时补足，同时检查无环。

## 6. Strong constraint

`strong` 不是 oracle DAG。它只提供正确的 block-level role-based blacklist：within-block edges 保持可搜索；合法跨 block 方向保持可搜索；其他跨 block 方向进入 blacklist。算法仍需判断具体节点对是否存在边。

## 7. 数据生成与离散 BN 学习

每个 job 只生成一次 true DAG、一次数值数据和一次 `dat_disc`；同一份 `dat_disc` 复用于所有 constraint conditions。结构学习使用 `hc + BDe`，`iss = 10`。

`rho` 沿用原参数名。Regime 2 中 D block 使用独立 common factor `U_D`，避免被人为并入 A/B 主机制。

## 8. 四个 Regime 的位置

| Regime | 节点数 | 主结构 | D block | 主要比较意义 |
|---|---:|---|---|---|
| Regime 1 | 16 | A、B 并列进入 C | 无 | 主设定 |
| Regime 2 | 21 | Regime 1 + D -> Y | 有 | 额外直接风险压力 |
| Regime 3 | 16 | A -> B -> C 链式 | 无 | 链式替代机制 |
| Regime 4 | 21 | Regime 3 + D -> Y | 有 | 链式复杂压力测试 |

## 9. 对比 Regime 时的结论组织

优先比较 Regime 1 vs 2、Regime 3 vs 4 来考察增加 D -> Y 的影响；比较 Regime 1 vs 3、Regime 2 vs 4 来考察并列上游与链式机制。跨 `p=16` 与 `p=21` 比较时必须明确节点规模差异。
