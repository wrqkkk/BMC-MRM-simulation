# Regime 4 设计说明

## 1. 设计定位

**Regime 3 + D 直接风险 block，是链式结构的复杂压力测试。**

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
A -> B -> C -> Y
A -> Y
B -> Y
C -> Y
D -> Y
```

合法跨 block 方向：`A -> B`、`B -> C`、`A -> Y`、`B -> Y`、`C -> Y`、`D -> Y`。

当前版本不允许 `A -> C` skip-layer edge；D 只作为 direct-risk block，通过 `D -> Y` 进入结构。所有 block 都允许 within-block true edges，并通过 acyclicity check 保证无环。

## 4. 固定 core edges

| Core edge | 类型 |
|---|---|
| `A1 -> B1` | between-block core |
| `A2 -> B2` | between-block core |
| `B1 -> C1` | between-block core |
| `B2 -> C2` | between-block core |
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

`target_edges = p * edge_multiplier`。`edge_multiplier=1` 时目标 21 条边，`edge_multiplier=2` 时目标 42 条边。`WITHIN_TRUE_RATE <- 0.25` 控制 within-block true edges 的目标比例。

## 6. Strong constraint

`strong` 不是 oracle DAG。它保留 within-block 搜索空间，只允许上述合法跨 block 方向，其他跨 block 方向进入 blacklist。算法仍需判断具体节点对是否存在边。

## 7. 数据生成与离散 BN 学习

每个 job 只生成一次 true DAG、一次数值数据和一次 `dat_disc`；同一份 `dat_disc` 复用于所有 constraint conditions。学习继续使用 `hc + BDe`，`iss=10`。

`rho` 沿用原参数名。Regime 4 中 D block 使用独立 common factor `U_D`，避免被人为并入 A/B 主机制。

## 8. 四个 Regime 的位置

| Regime | 节点数 | 主结构 | D block | 主要比较意义 |
|---|---:|---|---|---|
| Regime 1 | 16 | A、B 并列进入 C | 无 | 主设定 |
| Regime 2 | 21 | Regime 1 + D -> Y | 有 | 额外直接风险压力 |
| Regime 3 | 16 | A -> B -> C 链式 | 无 | 链式替代机制 |
| Regime 4 | 21 | Regime 3 + D -> Y | 有 | 链式复杂压力测试 |

## 9. 对比 Regime 时的结论组织

Regime 2 vs 4 是节点数相同、有 D block 条件下的并列上游 vs 链式机制主比较；Regime 3 vs 4 用于考察增加 D -> Y 的影响。跨 `p=16` 与 `p=21` 比较时必须明确节点规模差异。
