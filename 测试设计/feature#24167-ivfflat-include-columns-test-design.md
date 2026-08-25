# MatrixOne IVF Index INCLUDE Columns 测试设计

> 本设计覆盖 Feature [#24167](https://github.com/matrixorigin/matrixone/issues/24167)：为 IVFFLAT 索引支持 `INCLUDE` 列。该能力在 entries 隐藏索引表中冗余存储指定的原表列，以支持 Index-Only Scan 和 include 条件部分下推；不带 `INCLUDE` 的既有 IVF 索引行为必须保持不变。

## 1. 测试原则

| 项目 | 说明 |
| --- | --- |
| 功能定位 | `INCLUDE (col...)` 只扩展 IVFFLAT 索引的 entries 表和优化路径；向量索引的距离计算、聚类及普通 IVF 查询语义保持兼容。 |
| 正确性优先 | 无论是否命中 index-only / 下推优化，查询结果（行集合、距离排序、LIMIT/OFFSET 语义）必须与不使用索引的基线一致。 |
| 计划与结果双校验 | 每个优化场景同时检查 `EXPLAIN`/`EXPLAIN ANALYZE` 和查询结果；仅计划变化而结果错误、或结果正确但错误绕过目标路径，均不通过。 |
| 模式隔离 | 默认/`auto` 可使用安全的 include 优化；`post`、`pre`、`force` 保持既有语义；`include` 显式启用特化路径和多轮切桶。 |
| 数据一致性 | entries 表中的 include 值必须随 INSERT、UPDATE、DELETE、REINDEX、恢复同步，不能产生过期值或残留行。 |
| 可观测性 | 多轮搜索必须通过 `EXPLAIN ANALYZE` 摘要展示轮次、bucket window、round limit 和输出行数，verbose 输出受限且可读。 |

### 1.1 基本信息

| 项目 | 内容 |
| --- | --- |
| MatrixOne 基线 | `main`（Feature #24167） |
| 功能入口 | `CREATE INDEX ... USING IVFFLAT ... INCLUDE (col1, col2)` |
| 适用索引 | IVFFLAT 向量索引；包含/不包含 include 列的索引均须覆盖。 |
| 主要优化 | 覆盖查询的 Index-Only Scan、include 谓词下推、`mode=include` 的分轮切 bucket 补候选。 |
| 关键元数据 | `IndexDef.IndexAlgoParams.include_columns`；entries 列名以 `__mo_index_include_` 为前缀。 |
| 关键模式 | 未指定、`auto`、`include`、`post`、`pre`、`force`。 |

### 1.2 执行矩阵

| 测试类型 | 无 INCLUDE IVF | INCLUDE IVF | 通过标准 |
| --- | --- | --- | --- |
| 基础 IVF 回归 | 执行 | 执行 | 距离、排序、Top-K 和 DML 后结果正确。 |
| DDL/元数据 | 基线兼容 | 执行 | 语法、校验、SHOW、entries schema 和元数据准确。 |
| 覆盖/下推优化 | 不适用 | 执行 | 计划符合门控，且结果等价于基线。 |
| 查询模式 | 执行 | 执行 | 模式边界不改变既有语义；`include` 具备预期特化行为。 |
| 生命周期与恢复 | 执行 | 执行 | 索引及 include 值全程一致。 |
| 性能 | `mode=pre` / `mode=post` 基线 | `mode=include`（过滤列必须 INCLUDE） | 使用同一 ANN-Benchmark 数据和过滤选择性，对比 Recall、延迟、扫描/候选行数、JOIN 与索引空间。 |

### 1.3 覆盖范围

| 一级模块 | 用例数 | 主要覆盖内容 |
| --- | ---: | --- |
| 功能与 DDL | 12 | 语法、列约束、元数据、隐藏表、SHOW |
| 查询与优化 | 17 | index-only、部分下推、表达式、排序、模式门控 |
| 一致性与并发 | 10 | INSERT/UPDATE/DELETE、事务、并发、REINDEX |
| ALTER/异常 | 8 | 列变更、非法输入、序列化、回退与安全性 |
| 恢复与兼容 | 6 | snapshot、备份恢复、升级和旧索引 |
| 性能与可观测性 | 12 | ANN-Benchmark 的 pre/post/include 对比、规模、Top-K、过滤选择性、轮次 explain |
| **合计** | **65** |  |

### 1.4 参考资料

| 资料 | 地址或位置 |
| --- | --- |
| Feature | <https://github.com/matrixorigin/matrixone/issues/24167> |
| 技术设计 | Issue 附件 `ivf-index-include-columns-design_.md` |
| 用户指南 | Issue 评论附件 `ivfflat-include-columns-user-guide.md` |
| 模板 | <https://github.com/Ariznawlll/mo-test-reports/blob/main/fulltext-gojieba-test/gojieba%E5%85%A8%E6%96%87%E7%B4%A2%E5%BC%95%E6%B5%8B%E8%AF%95%E8%AE%BE%E8%AE%A1.md> |

## 2. 测试用例

### 2.1 功能与 DDL（12 条）

通用数据：创建 `t(id bigint primary key, vec vecf32(3), name varchar(100), category varchar(20), price decimal(10,2), created_at datetime, description text)`；向量使用可精确排序的小数据集和可扩展的随机数据集两套数据。

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| IVFIC-001 | DDL | 单 include 列建索引 | `CREATE INDEX ... ON t(vec) USING IVFFLAT ... INCLUDE(name)` | 创建成功；entries 含 `__mo_index_include_name`，类型与源列一致。 |
| IVFIC-002 | DDL | 多 include 列及顺序 | 使用 `INCLUDE(name, category, price)` 创建并检查 entries、元数据 | 列完整保存且顺序稳定；`IndexAlgoParams.include_columns` 为合法 JSON 数组字符串。 |
| IVFIC-003 | DDL | 无 INCLUDE 回归 | 用现有语法创建 IVFFLAT 并执行查询 | 创建、entries schema、查询计划和结果与改动前兼容。 |
| IVFIC-004 | DDL | 建表内定义索引 | 在 `CREATE TABLE` 中定义 IVFFLAT + INCLUDE，插入数据后搜索 | 表/索引均创建成功，存量与新增数据均有 include 值。 |
| IVFIC-005 | DDL | 非存在列 | `INCLUDE(not_exists)` | 明确报错，不创建索引或隐藏表。 |
| IVFIC-006 | DDL | 向量列不可 include | `INCLUDE(vec)` | 明确拒绝；元数据和隐藏表无残留。 |
| IVFIC-007 | DDL | 主键列不可 include | `INCLUDE(id)`（单列及复合主键） | 明确拒绝；已存的 pk 不被重复作为 include 列。 |
| IVFIC-008 | DDL | 重复 include 列 | `INCLUDE(name, name)` | 明确拒绝，不能静默去重。 |
| IVFIC-009 | DDL | 不同类型 | 逐个 include 整数、decimal、varchar、datetime、bool、JSON、TEXT/BLOB | 合法类型可建索引并 DML 同步；大类型出现设计约定的 warning/提示（若实现）。 |
| IVFIC-010 | 元数据 | SHOW CREATE TABLE | 对多列 include 执行 `SHOW CREATE TABLE` | 以 `INCLUDE (name, category, price)` 回显，可直接执行恢复。 |
| IVFIC-011 | 元数据 | SHOW INDEX | 执行 `SHOW INDEX` | `Column_name` 只显示向量 key；`Index_params` 含 include 元数据，不产生伪造 include 行。 |
| IVFIC-012 | 隐藏表 | 系统列冲突与引用 | 源表含接近系统前缀的列名，创建索引并查 entries 定义 | entries 使用标准前缀且无命名冲突；隐藏复合主键仍为最后列。 |

### 2.2 查询与优化（17 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| IVFIC-013 | Index-Only | 全覆盖投影 | `SELECT id,name,l2_distance(...) d ... ORDER BY d LIMIT 10`；include `name` | 结果等价基线；计划无原表 JOIN，TVF 输出 include 值。 |
| IVFIC-014 | Index-Only | 覆盖 WHERE | include `name,category`，查询 `WHERE category='A'` | category 条件下推 entries；无原表 JOIN；结果/排序正确。 |
| IVFIC-015 | Index-Only | SELECT、WHERE、ORDER BY 全覆盖 | 组合 include 列、距离别名、普通投影/排序 | 覆盖判定正确，投影/列 tag 重写无错列或空值。 |
| IVFIC-016 | 非覆盖 | 投影含非 include 列 | 选择 `description` 且 include `name,category` | 必须 JOIN 原表；结果完整且不错误走 index-only。 |
| IVFIC-017 | 部分下推 | include 与非 include AND 条件 | `category='A' AND created_at > ...` | category 下推；created_at 保留原表；结果等价全表精确基线。 |
| IVFIC-018 | 下推表达式 | 比较、IN、BETWEEN、IS NULL、LIKE | 分别用 include 列构造各谓词 | 支持的表达式语义正确；NULL 与 LIKE 边界一致。 |
| IVFIC-019 | 下推表达式 | AND 拆分与 OR 保守性 | 混合 include/非 include 的 AND、OR | AND 仅下推纯 include 子树；跨列 OR 整体保留 residual。 |
| IVFIC-020 | 下推表达式 | 距离过滤保守回退 | `l2_distance(vec,q)<x` 与 include 条件组合 | 距离谓词不被错误通用下推；最终集合正确。 |
| IVFIC-021 | 排序 | 距离升序、降序、等距离 | 构造相同/不同距离，含 LIMIT/OFFSET | 排序稳定且 OFFSET/LIMIT 语义与基线相同。 |
| IVFIC-022 | 模式 | 默认模式 | 不指定 rank option，执行覆盖和部分下推查询 | 安全覆盖/下推可启用；不触发 include 专属多轮行为。 |
| IVFIC-023 | 模式 | `mode=auto` | 同 IVFIC-022 | 可利用安全 include 优化，查询结果和默认一致。 |
| IVFIC-024 | 模式 | `mode=post` | 覆盖查询及 include 谓词查询 | 保持既有 post 路径，不自动切换 index-only/partial pushdown。 |
| IVFIC-025 | 模式 | `mode=pre`/BloomFilter | 含 include 与非 include 条件 | 保持 pre-filter 语义；BloomFilter、include 输出和结果可共存。 |
| IVFIC-026 | 模式 | `mode=force` | 小表精确查询，含 INCLUDE 索引 | 仍走既有 force 语义，结果完全正确。 |
| IVFIC-027 | 模式 | `mode=include` 无 include 元数据 | 对旧 IVF 索引指定 include 模式 | 保守回退 post 路径，无 panic/错误 SQL。 |
| IVFIC-028 | 模式 | `mode=include` 无可下推条件 | 查询引用未覆盖列且无 include 谓词 | 回退 post 路径，结果与基线一致。 |
| IVFIC-029 | 复合主键 | 覆盖判定 | 复合 PK 表分别查询 pk 分量、显式 include 分量、pk 等价表达式 | 不将原始分量误判为天然覆盖；仅安全情形可 index-only。 |

### 2.3 一致性与并发（10 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| IVFIC-030 | INSERT | 索引创建后写入 | 插入不同 include 值的向量行，再执行覆盖查询 | 行立即按事务语义可见，返回的 include 值正确。 |
| IVFIC-031 | UPDATE | 仅 include 列更新 | 更新 name/category，查新旧值和 entries | 旧值不再返回，新值可过滤/投影；entries 无重复旧行。 |
| IVFIC-032 | UPDATE | 向量与 include 同时更新 | 更新 vec、category、name | 旧向量近邻/旧值消失，新向量/新值同时生效。 |
| IVFIC-033 | UPDATE | 非 key 但 include affected 判定 | 更新 include 列，检查 DML 索引维护计划及结果 | 判定为受影响索引，entries 被重写。 |
| IVFIC-034 | DELETE/TRUNCATE | 删除与清表 | 删除命中行、TRUNCATE 后重插 | 删除值无残留；重插数据可正确检索。 |
| IVFIC-035 | 事务 | 提交/回滚 | INSERT、UPDATE、DELETE 后分别 COMMIT/ROLLBACK，多 session 查询 | 源表与 entries 原子一致；回滚不污染索引。 |
| IVFIC-036 | 并发 | 读写并发 | 多 writer 更新 include/vec，多 reader 覆盖查询 | 无 panic/死锁/错行；提交后最终一致。 |
| IVFIC-037 | 缓存 | 查询级状态隔离 | 交替并发执行不同 RequestedIncludeColumns/过滤条件 | 不串列、不串过滤条件、不泄漏上一查询结果。 |
| IVFIC-038 | REINDEX | 全量重建 | 写入数据后 `REINDEX`，比较前后 entries 和查询 | include 列随重建保留，结果/排序一致。 |
| IVFIC-039 | 多索引 | 同表多个 IVF INCLUDE | 不同向量列或不同 include 集合建索引并查询 | 各索引 entries/元数据独立，优化器选用正确索引。 |

### 2.4 ALTER、异常与安全（8 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| IVFIC-040 | ALTER | DROP include 列 | `ALTER TABLE DROP COLUMN category` | 整个关联 IVF 索引及三张隐藏表按约定自动删除，无悬挂元数据。 |
| IVFIC-041 | ALTER | 修改 include 类型 | 更改 include 列类型/属性 | 索引被识别为受影响并重建或重注册；查询正确。 |
| IVFIC-042 | ALTER | RENAME include 列 | 重命名 category 后执行 SHOW、DML、查询 | `include_columns` 元数据同步更新为新名，索引可用。 |
| IVFIC-043 | ALTER | 无关列变更 | 修改未被索引/未 include 的列 | IVF 索引不被误删除或无谓重建。 |
| IVFIC-044 | 序列化 | 特殊列名 | 反引号、逗号、引号等合法特殊标识符 | JSON 编解码、SHOW 和恢复 DDL 不丢失/错拆列名。 |
| IVFIC-045 | SQL 安全 | 字面量转义 | include 字符串含单引号、反斜杠、空串、换行 | 下推 SQL 正确转义；不会语法失败、注入或扩大结果。 |
| IVFIC-046 | 防御 | include 数据长度错配 | 单测/mock Search 返回 keys 与 include 数据长度不一致 | 报明确内部错误，不能错位返回数据。 |
| IVFIC-047 | 防御 | 空轮与异常过滤 | 构造每轮全被去重/过滤的候选 | 连续空轮受上限保护并安全结束，无死循环。 |

### 2.5 恢复与兼容（6 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| IVFIC-048 | Snapshot | 快照恢复 | 建 INCLUDE 索引、写入多版本数据、snapshot/restore | 元数据、entries include 值和查询结果均恢复。 |
| IVFIC-049 | 备份恢复 | 逻辑备份/恢复 | 导出后恢复至新库，执行 `SHOW CREATE` 和查询 | 恢复 DDL 含 INCLUDE，索引可用。 |
| IVFIC-050 | PITR | 时间点恢复 | 在 include 更新前后设置恢复点 | 恢复后 include 值与目标时间点一致。 |
| IVFIC-051 | 兼容 | 旧索引升级 | 构造无 `include_columns` 的旧元数据并升级/加载 | 视为无 include；无需重建且普通 IVF 正常。 |
| IVFIC-052 | 兼容 | 旧逗号格式元数据 | 注入历史逗号分隔 include 值并加载 | 解析回退兼容，SHOW/查询正确。 |
| IVFIC-053 | 失败恢复 | 建索引/恢复中断 | DDL 失败、节点重启或任务中断后重试 | 无半成品隐藏表/脏元数据；重试可成功。 |

### 2.6 性能与可观测性（12 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果/记录项 |
| --- | --- | --- | --- | --- |
| IVFIC-054 | 性能 | 覆盖 Top-K 对照 | 大表上比较无 include、include index-only、全表基线 | 记录延迟、扫描行数、原表 JOIN；index-only 应消除原表 JOIN。 |
| IVFIC-055 | 性能 | 部分下推选择性 | 低/中/高选择性 include 条件 + residual 条件 | 记录 entries 候选数、JOIN 输入、延迟；结果均与基线一致。 |
| IVFIC-056 | 性能 | 索引构建与空间 | 不同 include 列数量/类型建索引 | 记录建索引耗时、entries 行数、存储体积，确认开销随 include 合理增长。 |
| IVFIC-057 | 多轮 | 第一轮不足补候选 | 首轮候选均被 residual filter 淘汰，后续 bucket 才命中 | `mode=include` 继续切桶直至满足 LIMIT 或耗尽；不会错误少返回。 |
| IVFIC-058 | 多轮 | 不重叠 bucket window | 强制至少三轮，采集内部 SQL/trace | bucket 窗口连续且不重叠，pk 去重后无重复输出。 |
| IVFIC-059 | Explain | 默认 EXPLAIN ANALYZE 摘要 | 运行多轮 include 查询 | 展示 round_count、bucket_windows、round_limits、empty_rounds、输出统计；不无限展开。 |
| IVFIC-060 | Explain | VERBOSE 展开上限 | 对大量轮次执行 verbose analyze | 前 N/后 M 轮可查看，中间轮折叠；输出规模受控且含摘要。 |
| IVFIC-061 | ANN-Benchmark | `mode=pre` 基线 | 载入固定 ANN-Benchmark 数据集和查询集；使用不含 INCLUDE 的 `idx_pre`，以过滤列谓词执行 pre 模式 | 记录 Recall@K、p50/p95/p99、QPS、entries 候选数、原表扫描/Join 输入及资源；结果满足精确过滤基线。 |
| IVFIC-062 | ANN-Benchmark | `mode=post` 基线 | 在同一数据快照上使用不含 INCLUDE 的 `idx_post`，执行相同过滤谓词和 post 模式 | 使用与 IVFIC-061 相同 K、probe、并发、warm-up 和查询序列；记录同一指标。 |
| IVFIC-063 | ANN-Benchmark | `mode=include` 对照 | 在隔离表/索引上创建 `idx_include ... INCLUDE (filter_col)`，以同一谓词和 include 模式执行 | EXPLAIN 显示过滤列从 entries 读取并下推；按覆盖情况避免或缩小原表 JOIN；结果满足精确过滤基线。 |
| IVFIC-064 | ANN-Benchmark | 过滤选择性矩阵 | 对 1%、10%、50%（及无过滤）选择性重复 pre/post/include | 每个单元均报告 Recall/延迟/候选数；include 的优势只在过滤列已 INCLUDE 时判定。 |
| IVFIC-065 | ANN-Benchmark | 索引代价与结果稳定性 | 对 `idx_pre`、`idx_post`、`idx_include` 记录建索引时间、entries 大小；多次随机化查询顺序重复 | include 的额外空间/构建代价可量化；三模式不出现过滤错误、少返回或结果漂移。 |

#### 2.6.1 ANN-Benchmark 模式对比执行规范

性能结论必须基于 ANN-Benchmark（`ann-benchmarks`）格式的固定 base/query/ground-truth 数据集；建议至少覆盖一个中等规模欧氏距离数据集和一个接近目标业务维度/规模的数据集。若使用公开数据集，记录数据集名称、原始文件校验和、维度、base/query 数、距离度量和 ground-truth 版本；若使用内部转换数据，转换脚本、随机 seed 和产物校验和同样必须归档。

| 项目 | `mode=pre` | `mode=post` | `mode=include` |
| --- | --- | --- | --- |
| 索引定义 | `idx_pre`：仅向量 key，不含过滤列 | `idx_post`：仅向量 key，不含过滤列 | `idx_include`：向量 key，**`INCLUDE (filter_col)`**；多个过滤列须全部列入 INCLUDE。 |
| 查询数据 | 与其他列完全相同 | 与其他列完全相同 | 与其他列完全相同。 |
| 查询谓词 | `WHERE filter_col = :value` | `WHERE filter_col = :value` | `WHERE filter_col = :value`，谓词列必须正是 INCLUDE 列。 |
| 模式 | 显式 `mode=pre` | 显式 `mode=post` | 显式 `mode=include`。 |
| 公平性约束 | 同一 lists/probe、K、LIMIT/OFFSET、会话变量、并发和查询顺序。禁止复用 include 索引。 | 同左。 | 同左；只允许因 INCLUDE entries 结构产生的计划差异。 |
| 正确性基线 | 对每个查询先按 filter 精确过滤，再全表计算 exact Top-K。 | 同左。 | 同左。 |

推荐建模方法：为每条 base vector 以固定 seed 生成低/中/高选择性的 `filter_col`（例如 `category`）；每一条 benchmark query 使用同一 filter value 运行三种模式。为避免同一表上多个 IVF 索引导致优化器选错，三模式应在内容完全相同的独立表上执行，或在每次运行前仅保留目标索引。`mode=include` 的索引创建 DDL 必须显式包含过滤列，例如：

```sql
CREATE INDEX idx_include ON ann_include (vec)
  USING IVFFLAT LISTS <lists> OP_TYPE 'vector_l2_ops'
  INCLUDE (category);
```

每种模式在 warm-up 后至少运行 3 轮；轮次之间随机化 query 顺序，并报告每轮及汇总结果。查询使用产品实际支持的 rank option 语法显式指定相应 mode；记录原始 SQL 和 EXPLAIN ANALYZE，避免以客户端默认 mode 代替被测模式。

| 必录指标 | 说明 |
| --- | --- |
| Recall@1/@10/@100、返回行数 | 与“先过滤、再 exact Top-K”的 ground truth 比较；返回不足必须标为正确性失败。 |
| 延迟与吞吐 | 单查询 p50/p95/p99、QPS、warm-up 与测量轮次分别记录。 |
| 执行代价 | entries 扫描/候选数、原表扫描/Join 输入、round count、bucket windows、CPU/内存/磁盘 I/O。 |
| 索引代价 | 建索引时间、entries 表大小、总索引大小、include 相对无 include 的增量。 |
| 计划证据 | `mode=pre/post/include`、过滤下推、是否 Index-Only/Join、residual filter 必须由 EXPLAIN/ANALYZE 证明。 |

## 3. 基线对照与判定

### 3.1 结果基线

每个查询场景均建立以下其中一种可信基线：

1. 小数据集：穷举计算 `l2_distance` 后按距离、主键稳定排序，比较完整行集合及距离。
2. 大数据集：禁用 IVF 优化或使用全表精确查询，比较 Top-K 的主键、投影值和距离（浮点数按约定 epsilon 比较）。
3. 含过滤：先按 SQL WHERE 语义过滤，再执行向量 Top-K；不得把 IVF 候选截断误当作最终 LIMIT。

### 3.2 计划判定

| 场景 | 必须观察到 | 禁止观察到 |
| --- | --- | --- |
| 覆盖查询 | `ivf_search` 输出所需 include 列；无原表 JOIN | 仍扫描原表取已覆盖列。 |
| 部分下推 | include 条件位于 entries 内部 SQL；非 include 条件保留 residual | 过滤条件丢失、跨列 OR 被错误拆下推。 |
| `post/pre/force` | 对应既有路径 | 未经模式许可的 include 特化改写。 |
| `include` 多轮 | 后续不重叠 bucket、去重、可终止 | 首轮不足即返回，或空轮无限循环。 |

## 4. 结果记录要求

- 每条用例记录 MatrixOne 版本、集群规模、参数（`lists`、probe limit、rank mode）、建表/建索引 SQL 和数据集种子。
- 功能用例记录 SQL、实际/预期结果、`SHOW CREATE TABLE`、`SHOW INDEX` 和必要的 entries schema。
- 优化用例附 `EXPLAIN`/`EXPLAIN ANALYZE`，标明是否 index-only、下推条件、是否原表 JOIN 及轮次摘要。
- 性能数据至少运行 3 次，记录 p50/p95、扫描/候选行数、索引大小；仅在相同硬件和数据快照下比较。
- 缺陷须附最小复现 SQL、期望/实际结果、计划、错误日志及是否影响正确性、性能或可用性。
