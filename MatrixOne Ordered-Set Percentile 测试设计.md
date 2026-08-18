# MatrixOne Ordered-Set Percentile 测试设计

PERCENTILE_CONT / PERCENTILE_DISC 已完整实现，本质上是在 MatrixOne 聚合框架中新增的两种精确 ordered-set percentile 聚
合。测试以现有聚合能力为基线，在单 CN/多 CN、内存/spill、全量聚合/partial merge 等执行形态下分别执行 CONT 与 DISC，并
补充百分位数学语义、NULL 与边界值、数值及 DECIMAL 精度、非法参数、执行器复用和资源回收专项验证。

## 1. 测试原则

| 项目 | 说明 |
| --- | --- |
| 功能定位 | PERCENTILE_CONT / PERCENTILE_DISC 是 SQL ordered-set 精确百分位标量聚合；`WITHIN GROUP (ORDER BY ...)` 决定聚合内部排序，与最终结果集的 `ORDER BY` 无关。 |
| 公共能力 | 两函数应共用 parser、binder、聚合配置、分组执行、分布式 partial merge、spill 和资源回收等公共链路。 |
| 语义差异 | CONT 使用线性插值；DISC 返回排序序列中的实际离散值。两者不能共用相同的结果 Oracle。 |
| 确定性断言 | 精确百分位、NULL、类型、错误和单/多 CN 结果必须完全一致。 |
| 对照要求 | 单次聚合、partial merge、序列化/反序列化、内存执行、spill 执行、单 CN 与多 CN 必须与独立参考结果一致。 |
| 精度要求 | 不得把大整数或 DECIMAL 隐式转为 float64 后排序或计算，须覆盖 2^53 以上数值与 DECIMAL 边界。 |
| MVP 边界 | 窗口形式、动态 percentile 参数、非数值排序列及 DECIMAL256 当前不支持；必须稳定报错。 |
| 性能判定 | 固定环境及数据快照，记录耗时、峰值内存、spill 数量和结果；要求无 OOM、panic、资源泄漏或无界增长。 |

### 1.1 基本信息

| 项目 | 内容 |
| --- | --- |
| Feature | #25144 (https://github.com/matrixorigin/matrixone/issues/25144) |
| 功能入口 | `PERCENTILE_CONT(p) WITHIN GROUP (ORDER BY expr)`；`PERCENTILE_DISC(p) WITHIN GROUP (ORDER BY expr)` |
| 实现 PR | #26781 (https://github.com/matrixorigin/matrixone/pull/26781) |
| MatrixOne 基线 | `main`，合入提交 `f6889dd2a` 及后续修复 |
| 功能类型 | 精确 ordered-set 标量聚合 |
| 支持数据类型 | signed/unsigned 整数、float、DECIMAL64、受精度限制的 DECIMAL128 |
| 支持执行形态 | 无分组/GROUP BY、ASC/DESC、单 CN/多 CN、partial merge、spill |
| 不支持范围 | percentile window function、非常量/NULL percentile 参数、非数值排序列、DECIMAL256；部分最大精度 DECIMAL128 的 CONT 插值 |
| 数学语义 | CONT：`r = 1 + p × (N - 1)`，按相邻秩线性插值；DISC：返回累计分布首次达到 p 的排序值 |
| NULL 规则 | NULL 不参与排序、基数及结果计算；无有效值时返回 NULL |
| 资源风险 | 每个 group 可能保留排序状态并触发 spill；重点关注 merge 正确性、临时文件和 mpool 回收。 |

### 1.2 执行矩阵

| 测试类型 | 单 CN | 多 CN | 通过标准 |
| --- | --- | --- | --- |
| Parser / AST | 执行 | 不适用 | 合法 `WITHIN GROUP` 被正确解析、反序列化；非法语法稳定拒绝。 |
| Binder / Plan | 执行 | 执行 | 函数、常量 p、排序表达式和 ASC/DESC 正确编码为聚合配置。 |
| 精确数学语义 | 执行 | 执行 | SQL 结果等于测试侧独立排序与数学 Oracle。 |
| 类型与精度 | 执行 | 执行 | 不发生 uint/DECIMAL 精度丢失；返回类型和数值符合契约。 |
| 分组与 partial merge | 执行 | 执行 | 分片合并结果等于单次全量聚合结果。 |
| Spill | 执行 | 执行 | spill 与内存结果相同；无临时文件和 mpool 泄漏。 |
| 错误处理 | 执行 | 执行 | 无效输入返回用户可理解的错误，不 panic。 |
| 执行器复用 | 执行 | 不适用 | Reset/Free 后不残留上次 query/group/config 状态。 |
| 大数据与性能 | 执行 | 执行 | 结果正确、过程稳定；记录耗时、p50/p95、内存、spill 和 CPU。 |

### 1.3 覆盖范围

| 一级模块 | 用例数 | 主要覆盖内容 | 验证范围 |
| --- | ---: | --- | --- |
| 语法与计划 | 12 | parser、AST round-trip、binder、EXPLAIN、排序方向、窗口拒绝 | 单 CN 单元测试 |
| 数学语义 | 14 | CONT 插值、DISC 离散秩、NULL、空集、重复值、边界 percentile | 单 CN + 多 CN |
| 类型与精度 | 10 | int、uint、float、DECIMAL64/128、2^53、NaN、拒绝类型 | 单 CN + 多 CN |
| 分组与合并 | 8 | 多 group、分片、四路 merge、状态序列化/反序列化 | 单元测试 + 多 CN |
| spill 与生命周期 | 7 | spill、多 run compaction、取消、错误、Reset、Free、资源回收 | 单 CN 单元测试 |
| 异常 | 9 | 非常量/NULL/越界 p、多排序列、缺失语法、配置不匹配 | 单 CN + 多 CN |
| SQL 组合 | 5 | WHERE、HAVING、JOIN、CTE、最终结果排序 | SQL BVT |
| 性能与稳定性 | 6 | 大 group、高基数 group、倾斜、并发、长稳、单/多 CN 对照 | 性能环境 |
| **合计** | **71** | | |

### 1.4 参考资料

| 资料 | 地址或位置 |
| --- | --- |
| Feature Request | #25144 (https://github.com/matrixorigin/matrixone/issues/25144) |
| Exact ordered-set implementation | PR #26781 (https://github.com/matrixorigin/matrixone/pull/26781) |
| Approximate percentile 相关实现 | PR #25881 (https://github.com/matrixorigin/matrixone/pull/25881) |
| SQL BVT | `test/distributed/cases/function/func_aggr_ordered_set.test` |
| Parser 测试 | `pkg/sql/parsers/dialect/mysql/mysql_sql_test.go` |
| Plan 测试 | `pkg/sql/plan/ordered_set_aggregate_test.go` |
| 聚合函数类型检查 | `pkg/sql/plan/function/ordered_set_test.go` |
| 执行器 / spill / merge 测试 | `pkg/sql/colexec/aggexec/ordered_percentile_test.go` |

## 2. 测试用例

### 2.1 功能（12 条）

| 编号 | 二级模块 | 测试场景 | 前置条件/数据 | 测试步骤 | 预期结果 | 验证范围 |
| --- | --- | --- | --- | --- | --- | --- |
| PCT-001 | 基本语法 | CONT 无分组聚合 | 表含 1,2,100,101 | 执行 `percentile_cont(0.5) within group (order by v)` | 返回 51；SQL 可 parse、bind、执行 | 单 CN/多 CN |
| PCT-002 | 基本语法 | DISC 无分组聚合 | 同 PCT-001 | 执行 `percentile_disc(0.5) within group (order by v)` | 返回 2，为源列实际值 | 单 CN/多 CN |
| PCT-003 | 排序 | 默认 ASC 与显式 ASC | 表含乱序整数 | 分别执行默认排序与 `ORDER BY v ASC` | 两次结果完全相同 | 单 CN |
| PCT-004 | 排序 | DESC | 表含 1,2,100,101 | 对 CONT/DISC 执行 `ORDER BY v DESC` | 与测试侧 DESC 参考序列结果一致 | 单 CN/多 CN |
| PCT-005 | 分组 | 多 group 聚合 | group 具有不同值、基数与 NULL 比例 | GROUP BY g 查询 CONT/DISC | 每个 group 独立计算，无串组或漏组 | 单 CN/多 CN |
| PCT-006 | 聚合组合 | WHERE 后 percentile | 含有效值、NULL 与被过滤值 | 使用 WHERE 过滤后执行 percentile | 仅过滤后行参与排序和基数计算 | 单 CN/多 CN |
| PCT-007 | 聚合组合 | HAVING | 多 group，部分 group 的 percentile 不满足条件 | GROUP BY 后以 percentile 作为 HAVING 条件 | 先正确聚合再正确过滤 group | 单 CN |
| PCT-008 | 输出 | 结果别名与最终 ORDER BY | 多 group | percentile 使用别名，按别名排序 | 最终排序只影响输出行序，不影响 group 内 percentile 排序 | 单 CN |
| PCT-009 | EXPLAIN | Explain 语义 | 合法 CONT/DISC SQL | 执行 EXPLAIN / EXPLAIN ANALYZE | 显示函数名、WITHIN GROUP 排序表达式和方向；不 panic | 单 CN |
| PCT-010 | GROUP_CONCAT 回归 | 共用 WITHIN GROUP 语法 | 含 v、sort_key 的表 | 执行 `GROUP_CONCAT(v) WITHIN GROUP (ORDER BY sort_key)` | 现有 GROUP_CONCAT 两种排序写法不回归 | 单 CN BVT |
| PCT-011 | 关键字兼容 | within 作为列名 | 建表字段名为 `within` | 查询该列，并查询 `information_schema.KEYWORDS` | 普通标识符场景可用；关键字属性符合既定定义 | 单 CN BVT |
| PCT-012 | 事务可见性 | 同事务内聚合 | 事务内插入、删除、更新排序列 | 提交前后执行 percentile；回滚后重查 | 结果遵循事务隔离；回滚后不残留未提交值 | 单 CN/多 CN |

### 2.2 数学语义（14 条）

| 编号 | 二级模块 | 测试场景 | 前置条件/数据 | 测试步骤 | 预期结果 | 验证范围 |
| --- | --- | --- | --- | --- | --- | --- |
| PCT-013 | CONT | 偶数行中位数插值 | 升序值 1,2,100,101 | CONT，p=0.5 | 返回 51；证明不是离散取值 | 单 CN/多 CN |
| PCT-014 | DISC | 偶数行中位数离散取值 | 升序值 1,2,100,101 | DISC，p=0.5 | 返回 2；结果必须属于输入值集合 | 单 CN/多 CN |
| PCT-015 | CONT | 非整数位置插值 | 升序值 1,2,3,4,5 | CONT，p=0.25、0.75 | 分别返回 2、4 | 单 CN |
| PCT-016 | DISC | 非中位 percentile | 升序值 1,2,3,4,5 | DISC，p=0.25、0.75 | 返回正确的累计分布位置值 | 单 CN |
| PCT-017 | 边界 | p=0 | 乱序且含 NULL 的数据 | CONT/DISC，ASC 与 DESC 各执行一次 | 返回各自排序顺序的第一项 | 单 CN/多 CN |
| PCT-018 | 边界 | p=1 | 同 PCT-017 | CONT/DISC，ASC 与 DESC 各执行一次 | 返回各自排序顺序的最后一项 | 单 CN/多 CN |
| PCT-019 | NULL | NULL 混入 | 1,NULL,2,NULL,100,101 | CONT/DISC p=0.5 | 结果等同只输入 1,2,100,101 | 单 CN/多 CN |
| PCT-020 | 空集 | 无输入行 | WHERE 1=0 | 执行 CONT/DISC | 返回 NULL，不报错、不 panic | 单 CN/多 CN |
| PCT-021 | 全 NULL | group 无有效值 | group 内全部为 NULL | GROUP BY 执行 CONT/DISC | 该 group 返回 NULL；其他 group 不受影响 | 单 CN/多 CN |
| PCT-022 | 单值 | 单行 / 单有效值 | 一个 group 只有一个非 NULL 值 | 对 p=0,.25,.5,.95,1 分别执行 | 两函数均返回该唯一值 | 单 CN |
| PCT-023 | 重复值 | 重复值计入秩 | 1,1,1,100,100 | 多个 p 下执行 CONT/DISC | 结果按全部行而非 DISTINCT 值计算 | 单 CN/多 CN |
| PCT-024 | 负值 | 负数、零、正数 | -100,-1,0,1,100 | CONT/DISC 的 .25,.5,.75 | 与独立数学 Oracle 相同 | 单 CN |
| PCT-025 | ASC/DESC 对照 | 同输入双方向 | 固定乱序、重复、NULL 数据 | 分别对 ASC/DESC 计算多个 p | 各自等于独立参考排序结果，不复用生产排序 | 单 CN/多 CN |
| PCT-026 | 属性测试 | 确定性随机样本 | 固定 seed；生成不同 N、重复率、group、NULL 率和 p | SQL 结果与测试侧 big.Rat Oracle 比较 | 每一轮精确相等；失败日志含 seed、输入和 p | 单元测试 |

### 2.3 类型与精度（10 条）

| 编号 | 二级模块 | 测试场景 | 前置条件/数据 | 测试步骤 | 预期结果 | 验证范围 |
| --- | --- | --- | --- | --- | --- | --- |
| PCT-027 | 有符号整数 | INT8/16/32/64 | 每种类型包含边界、重复、负数 | CONT/DISC 多 percentile | 排序、插值和返回类型正确 | 单元测试 + SQL |
| PCT-028 | 无符号整数 | UINT8/16/32/64 | 包含 0、最大邻近值 | CONT/DISC 多 percentile | 按 native unsigned 顺序，不按 signed 或 float 顺序 | 单元测试 + SQL |
| PCT-029 | 大 uint | 超过 2^53 | 9007199254740992,9007199254740993 | DISC p=0/1，CONT 相邻值测试 | 结果无 float64 精度塌缩或顺序颠倒 | 单元测试 + SQL |
| PCT-030 | 浮点 | FLOAT32/FLOAT64 有限值 | 正负值、小数与重复 | CONT/DISC 多 percentile | 数值符合参考公式和精度容忍度 | 单元测试 + SQL |
| PCT-031 | NaN | float 含 NaN | 同时构造内存与强制 spill 数据 | ASC/DESC、CONT/DISC 对比 | 行为明确，且内存/spill/merge 完全一致 | 单元测试 |
| PCT-032 | DECIMAL64 | 固定 scale decimal | 1.0000,2.0000,100.0000,101.0000 | CONT/DISC .5,.25,.75 | CONT 精确插值；不经 float64；scale 符合契约 | 单元测试 + SQL |
| PCT-033 | DECIMAL128 | 支持精度范围 | 接近精度上限但合法的 DECIMAL128 | CONT/DISC 多 percentile | 数值、scale 和类型正确 | 单元测试 + SQL |
| PCT-034 | 最大精度 DECIMAL128 | CONT 受限边界 | DECIMAL(38,0) / DECIMAL(38,38) | 执行 CONT | 返回稳定的“不支持/参数非法”错误，不返回截断值 | SQL BVT |
| PCT-035 | 最大精度 DECIMAL128 | DISC 离散结果 | DECIMAL(38,0) | 执行 DISC | 若支持，返回输入的精确离散 decimal；无精度丢失 | 单元测试 + SQL |
| PCT-036 | 不支持类型 | varchar、JSON、日期、DECIMAL256 | 各自建表插入数据 | 作为 ORDER BY 表达式执行 percentile | 明确拒绝，不隐式 cast、不 panic | 单元测试 + SQL |

### 2.4 分布式一致性（8 条）

| 编号 | 二级模块 | 测试场景 | 前置条件/数据 | 测试步骤 | 预期结果 | 验证范围 |
| --- | --- | --- | --- | --- | --- | --- |
| PCT-037 | 批处理 | 单批与多批填充 | 相同数据切成不同 batch 大小 | 分别 BulkFill、多次 Fill | 最终结果一致 | aggexec 单元测试 |
| PCT-038 | 两路合并 | 两份 partial state | 数据按奇偶行切分 | 各自聚合后 merge | merge 结果等于全量聚合 | aggexec 单元测试 |
| PCT-039 | 四路合并 | 四份交错 partial state | 每个分片都含低/中/高值及 NULL | 四路 merge 后 flush | 等于全量参考结果，无分片偏置 | aggexec 单元测试 |
| PCT-040 | 分组 merge | 多 group partial state | 各分片含重叠 group 与独有 group | partial 聚合后 merge | group 对应关系正确，无跨 group 混合 | aggexec 单元测试 |
| PCT-041 | 序列化 | partial state round-trip | 包含重复、NULL、decimal | 序列化、反序列化、merge、flush | 结果等于未序列化 merge | aggexec 单元测试 |
| PCT-042 | 多 CN 无分组 | 分布式全局聚合 | 足够触发多 CN partial aggregation 的数据 | 单 CN 与多 CN 分别执行 | CONT/DISC 结果完全相同 | 多 CN E2E |
| PCT-043 | 多 CN 分组 | 分布式 group merge | 多 group、倾斜 group、NULL | 单 CN 与多 CN 分别执行 GROUP BY | 每一 group 的结果完全相同 | 多 CN E2E |
| PCT-044 | 物理计划变化 | 不同并行度/分片 | 固定数据与 SQL | 改变 CN 数、并行度或数据分布后运行 | 逻辑结果不变；无卡死或错误 | 多 CN E2E |

### 2.5 Spill 与资源生命周期（7 条）

| 编号 | 二级模块 | 测试场景 | 前置条件/数据 | 测试步骤 | 预期结果 | 验证范围 |
| --- | --- | --- | --- | --- | --- | --- |
| PCT-045 | spill | 强制单 run spill | 超过内存阈值的倒序数据 | 配置低 spill 阈值后执行 CONT/DISC | 结果等于纯内存结果；确实发生 spill | aggexec 单元测试 |
| PCT-046 | spill | 多 run 与 compaction | 大量数据，阈值足以产生多个 run | 强制 spill 后 flush | 无丢行/重行；结果等于 Oracle | aggexec 单元测试 |
| PCT-047 | spill | 特殊数据 | NULL、重复、大 uint、DECIMAL、NaN | 内存与 spill 分别执行 | 两种路径结果一致 | aggexec 单元测试 |
| PCT-048 | reuse | 正常 Reset | 同一 executor 连续执行不同 group、不同 p | 每轮 Reset 后重新填充 | 结果等同新 executor；不残留上一轮数据/config | aggexec 单元测试 |
| PCT-049 | cleanup | 正常 Free | 已分配内存并发生 spill | flush 后调用 Free | mpool 使用量归零；spill 文件清理 | aggexec 单元测试 |
| PCT-050 | cleanup | 错误路径 Free | 配置错误、merge 错误或 spill I/O 注入失败 | 在部分初始化后释放 | 不 panic；mpool 与临时文件均回收 | aggexec 单元测试 |
| PCT-051 | cancel | 查询取消 | 大数据、强制 spill、带 context deadline | 执行中取消 query，随后运行健康查询 | 取消及时返回；无 CN 卡死、文件残留或后续查询污染 | 单 CN/多 CN E2E |

### 2.6 异常（9 条）

| 编号 | 二级模块 | 测试场景 | 前置条件/数据 | 测试步骤 | 预期结果 | 验证范围 |
| --- | --- | --- | --- | --- | --- | --- |
| PCT-052 | 语法 | 缺少 WITHIN GROUP | 普通数值表 | percentile\_cont(0.5) / percentile\_disc(0.5) | 报“requires WITHIN GROUP”类错误 | Parser/Plan/SQL |
| PCT-053 | 语法 | 多排序表达式 | 普通数值表 | WITHIN GROUP (ORDER BY a,b) | 报“requires exactly one … expression”类错误 | Parser/Plan/SQL |
| PCT-054 | 参数 | p 为列引用 | 表含 p 列 | percentile\_cont(p) within group ... | 报“must be a non-null constant”类错误 | Plan/SQL |
| PCT-055 | 参数 | p=NULL | 普通数值表 | percentile\_disc(NULL) ... | 明确拒绝 | Plan/SQL |
| PCT-056 | 参数 | 范围外 p | 普通数值表 | 分别传 -0.01、1.01 | 明确的范围错误 | Plan/SQL |
| PCT-057 | 参数 | 非有限 p | float 常量或可表达 NaN/Inf | 分别传 NaN、正/负 Inf | 明确拒绝 | Plan/SQL |
| PCT-058 | 窗口 | OVER() | 普通数值表 | ... WITHIN GROUP (...) OVER () | 明确“不支持 window function”错误 | Plan/SQL |
| PCT-059 | merge 配置 | percentile 或方向不一致 | 两个不同配置 partial state | 尝试 merge | 拒绝合并；原 state 保持可释放 | aggexec 单元测试 |
| PCT-060 | 状态损坏 | 无效序列化 state | 截断、未知版本、非法长度 state | 反序列化或 merge | 返回错误，无 panic、越界或泄漏 | aggexec 单元测试 |

### 2.7 SQL 组合（5 条）

| 编号 | 二级模块 | 测试场景 | 前置条件/数据 | 测试步骤 | 预期结果 | 验证范围 |
| --- | --- | --- | --- | --- | --- | --- |
| PCT-061 | JOIN | percentile 作用于 join 结果 | 事实表与维表关联 | JOIN 后按维度 GROUP BY percentile | join 过滤和重复语义正确；无错误下推导致结果变化 | SQL BVT |
| PCT-062 | CTE | CTE 内部 percentile | CTE 先过滤或投影数值列 | 在 CTE 内及外层分别聚合 | 结果符合等价展开 SQL | SQL BVT |
| PCT-063 | 子查询 | scalar 子查询 | 外层引用 percentile scalar 子查询 | 执行过滤、投影组合查询 | scalar 结果正确，无重复计算导致语义变化 | SQL BVT |
| PCT-064 | UNION ALL | 多分支输入 | 两个分支各含不同值范围 | 在 UNION ALL 外执行 percentile | 结果等同对合并后的完整集合计算 | SQL BVT |
| PCT-065 | DISTINCT / ORDER BY | 输出去重与排序 | 多 group 产生相同 percentile 值 | SELECT DISTINCT、最终 ORDER BY | 只影响输出，不改变 group 内 ordered-set 输入 | SQL BVT |

### 2.8 性能与稳定性（6 条）

| 编号 | 二级模块 | 测试场景 | 前置条件/数据 | 测试步骤 | 预期结果 | 验证范围 |
| --- | --- | --- | --- | --- | --- | --- |
| PCT-066 | 大单组 | 单个超大 group | 10 万、100 万、1000 万行固定快照 | 分别执行 p50/p95/p99 CONT/DISC | 结果正确；记录耗时、峰值内存、spill 数、CPU | 单 CN/多 CN |
| PCT-067 | 高基数分组 | 大量小 group | 固定行数、不同 group cardinality | 执行 grouped percentile | 无 OOM；结果正确；记录 group state 资源 | 单 CN/多 CN |
| PCT-068 | 数据倾斜 | 一个大 group + 多小 group | 固定倾斜比例 | 执行 grouped percentile | 小 group 不受大 group spill/资源压力影响；无失败 | 多 CN |
| PCT-069 | 并发 | 并发读 percentile | 固定大表和查询集 | 10/50/100 并发执行 | 无错误、panic、OOM；记录 QPS、p50/p95、RSS | 多 CN |
| PCT-070 | 长稳 | 长时间混合负载 | 多 CN；CONT/DISC、grouped/ungrouped、不同 p | 持续运行 2 小时并采集指标 | 无持续内存/临时文件增长、CN 重启、panic 或结果漂移 | 多 CN |
| PCT-071 | 精确/近似对照 | approx_percentile 对照 | 同一固定大数据集 | 同时执行 exact 与 approximate p95/p99 | exact 结果作为基准；记录 approximate 误差和双方资源，不要求值相等 | 单 CN/多 CN |

## 3. 单 CN / 多 CN / 内存与 Spill 对照基线

| 对比项 | 单 CN、内存聚合 | 单 CN、spill 聚合 | 多 CN、partial merge | 验证方法 | 数据规模 | 记录指标 | 通过标准 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| CONT 基础语义 | 全量输入由一个聚合器处理 | 强制排序 run 落盘并在 flush 时合并 | 各 CN 局部聚合后汇总合并 | 对同一输入执行 p0/p25/p50/p75/p95/p99 | 5 行、100 行、10 万行 | 结果值、返回类型、执行耗时 | 三种执行形态的结果均等于独立 CONT Oracle。 |
| DISC 基础语义 | 同左 | 同左 | 同左 | 对同一输入执行多个 p | 同左 | 结果值、源值归属 | 三种执行形态一致；结果必须来自输入有效值集合。 |
| ASC/DESC | 按 SQL 排序方向计算 | 使用 spill 排序方向计算 | 分布式合并后按统一方向计算 | 分别运行 ASC、DESC | 乱序、重复、NULL 数据 | 结果、排序方向配置 | 每种路径均等于对应方向的参考排序结果。 |
| NULL | 聚合器跳过 NULL | spill 前后均跳过 NULL | partial state 不包含 NULL 对秩的影响 | 插入 NULL、全 NULL、空输入 | 小表与 10 万行 | 结果、有效行数 | NULL 不参与秩计算；无有效值返回 NULL。 |
| 分组 | 每 group 独立维护状态 | 不同 group 可独立产生 spill run | 同一 group 的多 CN state 正确 merge | GROUP BY 后逐行比对 | 10、1 万、10 万 group | group 数、结果行、内存 | 不串组、不漏组；结果与单 CN 参考结果一致。 |
| 重复与数据倾斜 | 全量重复值和高频 group | 大 group 可触发 spill | 大 group 跨 CN 分片 merge | 输入重复值、一个超大 group 加多个小 group | 100 万～1000 万行 | spill 次数、峰值内存、耗时 | 结果正确，无 OOM、无限 state 增长或小 group 饥饿。 |
| 大整数 | 原生整数比较 | spill 序列化和排序保持原值 | partial state 传输保持原值 | 使用 2^53 以上 uint64 | 小表 + 大表 | 结果、类型 | 不发生 float64 精度丢失或排序颠倒。 |
| DECIMAL | 内存中精确计算 | spill 后精度不变 | CN 间序列化、merge 后精度不变 | DECIMAL64/128 执行 CONT/DISC | 小表 + 分片表 | 结果、scale、错误 | 支持范围内精确一致；不支持边界稳定报错。 |
| partial merge | 不适用 | 不适用 | 2/4/多路 partial state 合并 | 人工切分数据并与全量路径比较 | 5 行、100 行、10 万行 | 结果、state 大小 | merge 结果等于对完整数据一次聚合。 |
| executor reuse | 新实例基线 | spill 后 reset 基线 | 多 query/CN 复用基线 | 同一实例连续执行不同数据、p、方向 | 100 次循环 | mpool、临时文件、结果 | 每轮等同新实例，无上轮残留。 |
| 取消与错误 | 取消前无 partial state 泄漏 | 取消时回收临时 run | 一 CN 失败或取消不导致其他 CN 卡死 | deadline/cancel、错误注入 | 强制 spill 大数据 | 取消耗时、文件数、mpool、CN 状态 | 返回预期错误；后续查询正常；无资源残留。 |
| 性能 | 单节点基线 | 记录落盘成本 | 记录分布式吞吐与 merge 成本 | 同配置、同快照、预热后重复执行 | 10 万/100 万/1000 万行 | p50/p95、CPU、RSS、mpool、spill 数 | 结果正确、过程稳定；保留对照证据，不预设未经评审的硬阈值。 |

### 3.1 精确结果参考数据

| 输入（升序，有效值） | percentile | CONT 预期 | DISC 预期 | 用途 |
| --- | --- | --- | --- | --- |
| 1 | 0, .5, 1 | 1 | 1 | 单值边界。 |
| 1, 2, 3, 4, 5 | .25 | 2 | 2 | 基础 percentile。 |
| 1, 2, 3, 4, 5 | .5 | 3 | 3 | 奇数行中位数。 |
| 1, 2, 3, 4, 5 | .95 | 4.8 | 5 | CONT 与 DISC 差异。 |
| 1, 2, 100, 101 | .5 | 51 | 2 | 偶数行、非相邻值插值反例。 |
| 1, 1, 1, 100, 100 | .5 | 1 | 1 | 重复值参与秩计算。 |
| NULL, 1, 2, 100, 101, NULL | .5 | 51 | 2 | NULL 忽略。 |
| NULL, NULL | .5 | NULL | NULL | 全 NULL group。 |
| 1, 2, 100, 101（DESC） | .5 | 51 | 100 | DESC 以降序序列定义离散秩。 |
| 9007199254740992, 9007199254740993 | 0, 1 | 最小/最大值 | 最小/最大值 | uint64 超过 2^53 的精度保护。 |

## 4. 结果判定与记录要求

| 结果类型 | 记录内容 | 判定要求 |
| --- | --- | --- |
| 确定性 SQL 结果 | SQL、建表/插入数据、预期结果、实际结果、返回数据类型、执行节点数 | 实际值、NULL 状态和类型必须与独立 Oracle 完全一致；浮点仅可使用已声明的数值容差。 |
| CONT 数学结果 | 有效输入排序序列、N、p、秩 r、插值两端值、测试侧计算结果 | 必须符合 r = 1 + p × (N - 1)；不得依据生产代码计算预期值。 |
| DISC 数学结果 | 有效输入排序序列、N、p、选中秩、实际返回值 | 返回值必须是有效输入中的实际值，且满足累计分布语义。 |
| 分组结果 | 每个 group 的输入集合、有效行数、预期和实际 percentile | 每个 group 独立正确；不得有串组、漏 group 或 NULL group 错误。 |
| 类型与精度 | 输入类型、精度/scale、超 2^53 数值、结果类型和字面值 | 不得出现隐式 float64 精度损失、排序变化、decimal scale 异常或截断。 |
| SQL 错误 | SQL、错误码/错误类别、关键错误信息、是否有残留对象或资源 | 不支持语法、类型或参数必须稳定失败；不得 panic、内部错误或静默返回错误值。 |
| 单 CN / 多 CN 对照 | SQL、数据 seed、CN 数、分片方式、每个结果集 | 同一快照下，逐 group、逐函数、逐 percentile 结果必须相等。 |
| merge 对照 | 完整输入结果、各 partial 输入、状态序列化信息、merge 后结果 | partial merge 必须等价于全量单次聚合；配置不一致必须拒绝 merge。 |
| spill 对照 | 内存阈值、spill run 数、临时文件路径/数量、内存和 spill 结果 | spill 与内存结果必须相同；结束、错误或取消后临时文件应被清理。 |
| 生命周期结果 | Reset/Free 前后 mpool、临时文件数、重复执行结果 | mpool 回到基线；无临时文件残留；复用结果等同新实例。 |
| 取消与故障 | 取消时间、context 错误、CN 日志、后续健康查询结果 | 查询应在 deadline 内退出；无 CN 卡死、goroutine 泄漏、资源泄漏或后续查询污染。 |
| 性能结果 | commit、镜像版本、硬件/部署配置、数据生成 seed、规模、并发、预热次数、p50/p95、CPU、RSS、mpool、spill 数 | 相同环境与数据快照下可复现；必须正确且稳定，无 OOM、panic、CN 重启或持续资源增长。 |
| 回归证据 | 测试文件、命令、完整退出码、失败时最小复现 SQL/seed | 单元、BVT、多 CN、性能测试分别留存结果；不得以“无 panic”替代结果正确性验证。 |

