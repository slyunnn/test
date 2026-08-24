# MatrixOne Ordered-Set Percentile 测试设计（Review 修订版）

> 本文依据 `MatrixOne_Ordered-Set-Percentile_测试方案分析.md` 修订，覆盖 Issue [#25144](https://github.com/matrixorigin/matrixone/issues/25144) 当前的标量 MVP：`PERCENTILE_CONT` 与 `PERCENTILE_DISC`。窗口形式、动态 percentile 参数、非数值排序表达式及 DECIMAL256 不属于本期支持范围。

## 1. 测试原则

| 项目 | 说明 |
| --- | --- |
| 功能定位 | 两函数是精确 ordered-set 标量聚合；`WITHIN GROUP (ORDER BY ...)` 决定聚合内部顺序，与最终输出行排序独立。 |
| 数学语义 | CONT 使用 `r = 1 + p × (N - 1)` 后对相邻秩线性插值；DISC 返回排序后累计分布首次达到 p 的实际输入值。 |
| MVP 边界 | 仅常量且非 NULL 的有限 p，范围 `[0,1]`；窗口形式、动态/NULL p、非数值排序列、DECIMAL256 须稳定拒绝。 |
| 路径隔离 | 内存 partial merge、独立 spill、远程多 CN merge 分别验证；带 spill runs 的序列化/partial merge 是当前明确不支持组合。 |
| 类型契约 | DISC 保留输入类型；CONT 对非 DECIMAL 返回 float64。因此大整数 CONT 按 IEEE-754 float64 舍入判定，不要求任意大整数十进制精确。 |
| 独立 Oracle | 测试侧排序、rank、插值与转换不得复用生产排序器、rank 计算器或类型转换实现。 |
| 分层门禁 | 核心功能进入 PR/CI；实现可靠性独立专项；大规模、并发和长稳仅进入 nightly。 |

### 1.1 发布基线冻结（执行前置条件）

测试执行前必须在测试记录与 CI 配置中一次性冻结下列信息；任何一项为空时，仅允许做预检，不得出具发布验收结论。

| 项目 | 冻结要求 |
| --- | --- |
| MatrixOne 源码 | 精确 commit SHA（不接受 `main`、分支名或“后续修复”描述）。 |
| 构建产物 | 镜像 digest 或二进制 SHA256、编译参数、Go 版本。 |
| 部署 | CN/TN 数量、MORPC 协议版本、CPU/内存、存储和临时目录。 |
| 资源配置 | 实际生效的内存限制、spill 阈值/目录及相关 session/system variables。 |
| 测试数据 | 数据生成脚本版本、随机 seed、行数、分片方式与预期结果版本。 |
| Oracle | Oracle 程序/脚本 commit、浮点 ULP 或 epsilon、DECIMAL 比较规则。 |

### 1.2 基本信息

| 项目 | 内容 |
| --- | --- |
| 功能入口 | `PERCENTILE_CONT(p) WITHIN GROUP (ORDER BY expr [ASC|DESC])`；`PERCENTILE_DISC(p) WITHIN GROUP (ORDER BY expr [ASC|DESC])` |
| 实现参考 | PR [#26781](https://github.com/matrixorigin/matrixone/pull/26781)；实际验收以 1.1 冻结的 commit 为准。 |
| 支持类型 | signed/unsigned integer、float、DECIMAL64、支持范围内的 DECIMAL128。 |
| NULL 输入值 | 不参与排序、有效行数和计算；空集合或全 NULL 返回 NULL。 |
| NULL percentile 参数 | MatrixOne 当前 MVP 明确拒绝（`p` 必须为非 NULL 常量）；此点与 PostgreSQL 的 NULL 返回行为不同，作为兼容性差异记录。 |
| NaN/Infinity | 本期作为合法浮点输入冻结：ASC 下 NaN 位于普通数值前，DESC 下位于普通数值后；选中/插值端点为 NaN 时 CONT 返回 NaN；`-Inf < 有限值 < +Inf`。 |
| 分布式限制 | 仅内存状态支持序列化和 partial merge；含 spill runs 的状态序列化/partial merge 必须拒绝。 |

### 1.3 执行分层与覆盖范围

| 层级 | 用例数 | 主要范围 | 触发方式 |
| --- | ---: | --- | --- |
| 核心功能门禁 | 32 | 语法、数学、NULL、类型、分组、单/多 CN、独立 Oracle、负例 | PR CI / 发布门禁 |
| 实现可靠性 | 20 | 内存 partial merge、独立 spill、compaction、Free、取消、MORPC、故障恢复 | 专项回归 |
| Nightly 性能 | 8 | 大数据、高基数、倾斜、并发、长稳、exact/approx 对照 | Nightly |
| **合计** | **60** |  |  |

### 1.4 现有覆盖映射

| 已有测试 | 已覆盖内容 | 本设计补充重点 |
| --- | --- | --- |
| `func_aggr_ordered_set.test` | 基础 SQL、分组、ASC/DESC、NULL、DECIMAL、WITHIN 标识符 | 冻结语义、边界、Oracle、多 CN 与负例。 |
| `ordered_percentile_test.go` | 多数数值类型、基础数学、NaN、内存/spill 一致性 | 大整数 CONT 返回类型、路径隔离、资源与错误契约。 |
| `remote_expr_test.go` | MORPC v16/v17 远程执行门禁 | release 版本矩阵和多 CN E2E。 |

## 2. 核心功能门禁（32 条）

### 2.1 语法、计划与 SQL 组合（8 条）

| 编号 | 场景 | 测试步骤 | 预期结果 |
| --- | --- | --- |
| PCT-R001 | CONT 基本语法 | 对 `1,2,100,101` 执行 CONT p=.5 | 返回 51；parse/bind/execute 成功。 |
| PCT-R002 | DISC 基本语法 | 对相同数据执行 DISC p=.5 | 返回 2，且属于输入集合。 |
| PCT-R003 | ASC 默认与显式 | 比较默认和 `ORDER BY v ASC` | 结果相同。 |
| PCT-R004 | DESC | CONT/DISC 对乱序、重复、NULL 数据执行 DESC | 等于独立 DESC Oracle。 |
| PCT-R005 | GROUP BY/WHERE/HAVING | 多 group、过滤与 aggregate HAVING 组合 | 每 group 独立正确；过滤后才计算。 |
| PCT-R006 | 最终 ORDER BY/别名 | 按 percentile 别名排序 | 仅改变输出行序，不改变 WITHIN GROUP 结果。 |
| PCT-R007 | JOIN/CTE/UNION ALL | 在 join、CTE 和 UNION ALL 的最终输入上聚合 | 等于对应展开 SQL，不丢行/重行。 |
| PCT-R008 | EXPLAIN | EXPLAIN/ANALYZE 合法 SQL | 显示 ordered-set 聚合及方向，不 panic。 |

### 2.2 数学语义与 NULL（10 条）

| 编号 | 场景 | 数据与步骤 | 预期结果 |
| --- | --- | --- | --- |
| PCT-R009 | CONT 偶数中位数 | `1,2,100,101`，p=.5 | 51。 |
| PCT-R010 | DISC 偶数中位数 | `1,2,100,101`，p=.5 | 2。 |
| PCT-R011 | 非整数 rank | `1..5`，CONT/DISC p=.25/.75/.95 | 分别对照公式与累计分布；CONT .95=4.8、DISC .95=5。 |
| PCT-R012 | p 边界 | 乱序数据，p=0/1，ASC 和 DESC | 返回各排序方向的首/尾值。 |
| PCT-R013 | 单值与重复 | 单有效值；`1,1,1,100,100` 多个 p | 单值始终返回本身；重复参与 rank，不按 DISTINCT。 |
| PCT-R014 | 正负与零 | `-100,-1,0,1,100` 多个 p | 与独立 Oracle 一致。 |
| PCT-R015 | NULL 输入行 | `1,NULL,2,NULL,100,101` p=.5 | 等同 `1,2,100,101`；结果 CONT=51、DISC=2。 |
| PCT-R016 | 空集/全 NULL | `WHERE 1=0`、单 group 全 NULL、多 group 混合 | 无有效值 group 返回 NULL，其他 group 不受影响。 |
| PCT-R017 | NULL p | `percentile_*(NULL)` | 明确报“non-null constant”类错误；记录为与 PostgreSQL 的预期差异。 |
| PCT-R018 | 属性测试 | 固定 seed 生成 N、重复率、NULL 率、group 与 p | 每轮与独立 Oracle 比较；失败输出 seed、输入、方向和 p。 |

### 2.3 类型、精度与特殊浮点（9 条）

| 编号 | 场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- |
| PCT-R019 | 有符号整数 | INT8/16/32/64 边界、负值、重复 | 原生整数排序正确；DISC 保留输入类型。 |
| PCT-R020 | 无符号整数 | UINT8/16/32/64 的 0、最大邻近值 | 按 unsigned 原生顺序；DISC 精确返回原值。 |
| PCT-R021 | 超 2^53 的 DISC | `2^53`、`2^53+1`，p=0/1 | 精确返回 uint64；不发生排序颠倒或塌缩。 |
| PCT-R022 | 超 2^53 的 CONT | 相邻超大整数及非端点 p | 用 `float64` 标准舍入作为预期；检查返回 float64，不要求原十进制精确。 |
| PCT-R023 | FLOAT32/FLOAT64 有限值 | 正负、小数、重复，多 p | 使用 IEEE bit pattern 或声明 ULP 容差比较。 |
| PCT-R024 | NaN 排序与结果 | ASC/DESC 下含 NaN；使 NaN 落在选中秩/插值端点 | ASC NaN 在普通值前、DESC 在后；选中/参与插值时 CONT 返回 NaN；内存与 spill 一致。 |
| PCT-R025 | Infinity | 含 `-Inf`、有限值、`+Inf` 的 ASC/DESC | 排序为 `-Inf < 有限值 < +Inf`；端点和 DISC 返回契约一致。 |
| PCT-R026 | DECIMAL64 | 固定 scale 的插值/离散 p | Oracle 比较 unscaled value、scale、结果类型；不经 float64。 |
| PCT-R027 | DECIMAL128/限制 | 合法精度与最大精度、DECIMAL256 | 支持范围正确；不支持 CONT/DECIMAL256 稳定报错。 |

### 2.4 分布式与负例（5 条）

| 编号 | 场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- |
| PCT-R028 | 单 CN/多 CN 无分组 | 同一冻结数据快照、多个 p、ASC/DESC | CONT/DISC 完全一致。 |
| PCT-R029 | 单 CN/多 CN 分组 | 倾斜 group、NULL、重复和大整数 | 逐 group 与单 CN/Oracle 一致，无串组。 |
| PCT-R030 | percentile 形态 | p 为列/表达式、p<0、p>1、NaN/Inf | 明确拒绝，错误类别稳定。 |
| PCT-R031 | 不支持 SQL | 缺 WITHIN GROUP、多排序键、`OVER()` | 分别报 requires WITHIN、exactly one expression、window unsupported 类错误。 |
| PCT-R032 | 不支持排序类型 | varchar、JSON、日期、DECIMAL256 | 明确拒绝，不隐式转换、无 panic。 |

## 3. 实现可靠性专项（20 条）

### 3.1 内存 partial merge 与远程合并（7 条）

| 编号 | 场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- |
| PCT-R033 | Fill 批次不变性 | 同一输入以单 batch、多 batch 及不同 batch size 聚合 | 结果一致。 |
| PCT-R034 | 两/四路 merge | 将数据交错切分后局部聚合并 merge | 等于全量一次聚合。 |
| PCT-R035 | 分组 merge | 重叠/独有 group 在多 partial state merge | group 映射正确、无跨组值。 |
| PCT-R036 | 内存状态 round-trip | 无 spill 状态序列化/反序列化后 merge/flush | 等于未序列化状态和 Oracle。 |
| PCT-R037 | merge 配置不一致 | p、方向、类型不一致的 state merge | 拒绝 merge；状态仍可 Free。 |
| PCT-R038 | 损坏状态 | 截断、未知版本、非法长度序列化 state | 返回错误，无越界/panic/泄漏。 |
| PCT-R039 | MORPC 版本 | v16/v17 的远程表达式/聚合执行 | 兼容门禁通过，结果与本地一致。 |

### 3.2 Spill、compaction 与限制组合（6 条）

| 编号 | 场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- |
| PCT-R040 | 独立 spill | 使用实际生效阈值（不写“设置为 1”），以足够输入字节触发 spill | 记录阈值、行数、估计字节、spill runs；结果等于内存 Oracle。 |
| PCT-R041 | 多 run/compaction | 构造足够数据产生多个 run 和可观测 compaction | 记录 run 数、compaction、spill rows、peak memory；无丢/重行。 |
| PCT-R042 | spill 特殊值 | NULL、重复、NaN、超 2^53、DECIMAL 分别执行 | 与内存路径及类型契约一致。 |
| PCT-R043 | spill + 序列化 | 试图序列化含 spill runs 的状态 | 明确“不支持”错误，而非错误成功。 |
| PCT-R044 | spill + partial merge | 试图 merge 含 spill runs 的 partial state | 明确“不支持”错误；不将其列为成功组合验收。 |
| PCT-R045 | spill I/O 故障 | 运行期间注入写/读/目录错误 | 错误可诊断；后续 Free 清理资源。 |

### 3.3 生命周期、取消与资源（7 条）

| 编号 | 场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- |
| PCT-R046 | Free 后重建 | 执行内存聚合 → Free → 新建 executor，再执行不同数据/p/方向 | 新实例无残留，结果等于独立新实例；不假设不存在的 Reset API。 |
| PCT-R047 | 正常释放 | 内存与 spill 成功后调用真实 `Free` | mpool 回基线，临时文件清理。 |
| PCT-R048 | 错误后释放 | binder/merge/spill 失败后的部分初始化状态 Free | 不 panic；内存与文件回收。 |
| PCT-R049 | 单 CN 取消 | 强制 spill 的大查询使用 deadline/cancel | 在 deadline 内退出，后续健康查询正常。 |
| PCT-R050 | 多 CN 取消 | 一个或多个远程 fragment 被取消 | 无 CN 卡死、goroutine/临时文件泄漏、后续污染。 |
| PCT-R051 | 重复执行稳定性 | 循环创建、聚合、Free，混合类型与方向 | mpool/RSS 不持续增长，结果稳定。 |
| PCT-R052 | 事务可见性 | 事务内 INSERT/UPDATE/DELETE 后聚合，提交/回滚对照 | 遵循隔离，回滚不残留数据。 |

## 4. Nightly 性能（8 条）

| 编号 | 场景 | 数据/步骤 | 记录与通过标准 |
| --- | --- | --- | --- |
| PCT-R053 | 大单 group | 100k/1m/10m，p50/p95/p99，CONT/DISC | 正确；记录 p50/p95、CPU、RSS、mpool、spill、I/O。 |
| PCT-R054 | 高基数 group | 固定总行数，提高 group cardinality | 无 OOM；逐 group 抽样 Oracle 正确。 |
| PCT-R055 | 数据倾斜 | 一个超大 group + 多小 group | 无饥饿/失败，记录长尾与 spill。 |
| PCT-R056 | 并发 | 10/50/100 并发查询集 | 无错误/panic/OOM；记录 QPS、p50/p95、资源。 |
| PCT-R057 | 长稳 | 多 CN 混合查询持续 2 小时 | 无持续内存/临时文件增长、CN 重启或结果漂移。 |
| PCT-R058 | exact/approx 对照 | 同一固定数据执行 exact 和 approximate p95/p99 | exact 作真值；记录 approx 误差和双方资源，不要求相等。 |
| PCT-R059 | spill 成本 | 内存可容与强制 spill 的同一快照 | 记录耗时/资源差异；结果完全一致。 |
| PCT-R060 | 并行度 | 固定数据改变 CN 数与并行度 | 逻辑结果不变；记录吞吐、merge 成本。 |

## 5. Oracle 与结果判定

| 数据类型/场景 | Oracle 与比较规则 |
| --- | --- |
| DISC 整数（含 uint64） | 用原生整数/大整数排序，比较原始值及 SQL 返回类型，要求精确相等。 |
| CONT 非 DECIMAL | 使用独立 rank/插值实现，最终显式转换为 float64；比较 IEEE-754 bit pattern 或预先声明 ULP。 |
| FLOAT | 不以 `big.Rat` 强求文本精确相等；采用 bit pattern/ULP，NaN 单独用 `isNaN` 判断。 |
| DECIMAL | 比较 unscaled value、scale 与最终 SQL 类型；Oracle 不调用生产 decimal/rank/sorter。 |
| NULL/空集合 | 判断 SQL NULL 状态，不将 NULL 作为普通排序值。 |
| 单/多 CN、memory/spill | 使用同一冻结数据和 Oracle；允许执行指标不同，不允许逻辑结果不同。 |

## 6. 结果记录要求

- 所有门禁结果必须包含第 1.1 节全部冻结项、SQL、数据 seed、预期/实际、返回类型、CN 数与执行计划。
- spill 结果额外记录实际阈值、输入行数/估计字节、spill run 数、compaction、spill rows、peak memory、临时文件清理状态。
- 浮点/大整数失败必须给出输入排序序列、p、rank、插值端点、期望 float64 位模式或 ULP、实际值和类型。
- 标记“不支持”的 PCT-R043/PCT-R044 只有稳定拒绝才通过；不得因组合未执行而视为成功。
- 性能用例只在固定环境的 nightly 运行；不能以“未 panic”替代数值正确性或资源泄漏判断。
