# MatrixOne WAND/BMW Retrieval FullText 索引测试设计

> 本设计覆盖 Feature [#25541](https://github.com/matrixorigin/matrixone/issues/25541)：为 `retrieval` FullText 索引实现基于 Block-Max WAND（BMW）的 BM25 精确 Top-K 检索。索引在 posting/docmap 及 block-local maximum score 的基础上，以 WAND 阈值剪枝跳过不可能进入 Top-K 的文档，直接返回精确 BM25 Top-K，避免对全部命中文档评分、SQL Sort 和全量扫描。

## 1. 测试原则

| 项目 | 说明 |
| --- | --- |
| 精确性优先 | WAND/BMW 是精确剪枝，不是 ANN。任意语料、K、过滤、更新/删除与 segment 形态下，返回主键、BM25 分数和稳定排序必须与 exhaustive BM25 Oracle 一致。 |
| 计划下推 | 仅当查询满足 retrieval 的适用条件时，`LIMIT k` 作为 heap Top-K 下推至索引；期望索引直接有序输出，最终计划不得保留不必要 SQL Sort。 |
| 安全剪枝 | 任何被跳过 posting/block 都必须由 upper bound 证明不可能超过当前阈值；bound 不确定、统计缺失或配置不兼容时必须保守评分，不能漏掉 Top-K。 |
| 过滤与存活性 | WHERE prefilter membership bitmap 与多 segment liveness bitmap 必须在 WAND walk 中生效；删除/更新/过滤掉的文档不能参与候选或污染 corpus statistics。 |
| 段一致性 | tag=0 base、tag=1 tail delta、compaction/rebuild/恢复后的 posting、docmap、block max、N/avgdl 必须语义一致。 |
| 性能证据 | 性能用例同时记录完全评分候选数、实际评分数、跳块数、heap 阈值演进、CPU/内存与 p50/p95；性能提升不能以准确率换取。 |

### 1.1 基本信息

| 项目 | 内容 |
| --- | --- |
| MatrixOne 基线 | PR #27663 的 merge commit `03ffd507597ded14abbce44c31616e5979463096` 及验收环境冻结的后续精确 commit/immutable image。 |
| 索引类型 | `retrieval` FullText 索引；底层包含 term postings `(docID, tf)`、docmap `(pk, doc length)`、per-block max score 与 segment metadata。 |
| 查询算法 | DAAT WAND + BMW block-local upper bound，使用 size-K min-heap 阈值 `θ`。 |
| 评分 | BM25，依赖文档数 `N`、平均文档长度 `avgdl`、term/document frequency、TF 与 query terms。 |
| 适用查询 | 多 term OR 式 BM25 Top-K，LIMIT 可下推；过滤 bitmap 与 liveness bitmap 可并入检索。 |
| 关键输出 | 精确 Top-K 主键、BM25 score、稳定 tie-break 规则、候选/跳过/评分诊断（若暴露）。 |

### 1.2 执行矩阵

| 测试类型 | 穷举 BM25 基线 | WAND/BMW retrieval | 通过标准 |
| --- | --- | --- | --- |
| 评分/Top-K | 执行 | 执行 | Top-K 主键、score 与 tie 顺序一致。 |
| 过滤与 liveness | SQL 精确过滤后评分 | bitmap 参与 WAND | 结果等价，过滤/删除文档不可见。 |
| segment 生命周期 | 全量重建基线 | base + tail + compaction | 结果与单 segment 基线一致。 |
| 计划 | 普通 FullText/Sort 基线 | pushed Top-K | Top-K 下推且无多余 Sort，结果不变。 |
| 性能 | 记录全量评分成本 | 记录剪枝收益 | 精确性为 100%，WAND 在目标语料/K 下减少评分工作。 |

### 1.3 覆盖范围

| 一级模块 | 用例数 | 主要覆盖内容 |
| --- | ---: | --- |
| DDL、索引结构与统计 | 10 | retrieval 创建、postings/docmap/block max、BM25 统计、metadata |
| WAND/BMW 正确性 | 15 | Top-K、pivot、threshold、block skip、tie、K 边界、Oracle |
| 计划、过滤与 SQL 组合 | 12 | LIMIT 下推、WHERE bitmap、JOIN/CTE、Boolean/自然语言、回退 |
| DML、segment 与恢复 | 11 | insert/update/delete、tail、liveness、compaction、rebuild、backup |
| 异常、并发与可观测性 | 6 | 损坏 bound、取消、并发、explain/metrics、兼容 |
| 性能与长稳 | 6 | common-term、不同 K/块、过滤选择性、多 CN、continuous run |
| **合计** | **60** |  |

### 1.4 参考资料

| 资料 | 地址 |
| --- | --- |
| Feature | <https://github.com/matrixorigin/matrixone/issues/25541> |
| Implementation | <https://github.com/matrixorigin/matrixone/pull/27663> |
| WAND | Broder et al., *Efficient Query Evaluation using a Two-Level Retrieval Process*, CIKM 2003. |
| BMW | Ding & Suel, *Faster Top-k Document Retrieval Using Block-Max Indexes*, SIGIR 2011. |

## 2. 测试用例

通用表：`docs(id bigint primary key, txt text, category varchar(20), status int)`；数据集同时准备小型可手工计算语料、固定随机语料、common-term 大语料和真实/公开基准语料。Oracle 独立计算 `Σ BM25(term, doc)`，不得复用 production posting cursor、bound、rank 或 BM25 实现。

### 2.1 DDL、索引结构与 BM25 统计（10 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| WAND-001 | DDL | 创建 retrieval 索引 | 建合法主键/文本表，创建 retrieval FullText 索引，SHOW | 创建成功；metadata 明确 index/parser/算法。 |
| WAND-002 | DDL | 建表内/ALTER 建索引 | 分别在 CREATE TABLE、ALTER 有存量数据表创建 | 历史/新增文档均进入索引，可检索。 |
| WAND-003 | DDL | 非法 schema | 无主键、非文本列、非法 retrieval 参数 | 明确错误，无隐藏表/元数据残留。 |
| WAND-004 | postings | term tf/docmap | 小语料含重复 term、不同长度文本 | posting 的 docID/tf 与 docmap 的 PK/doclen 语义一致。 |
| WAND-005 | block max | 多 block posting | 构造各 block 的不同 tf/score 上界 | 每 block max ≥ 该 block 任意真实 term score；不得低估。 |
| WAND-006 | stats | N/avgdl | 空、单文档、不同长度、多批写入后检查/查询 | N、avgdl 与 live 文档集一致，查询 score 符合 Oracle。 |
| WAND-007 | metadata | base/tag0 与 tail/tag1 | 全量建索引后增量写入，检查 segment/查询 | base 与 tail 均被读取，顺序/统计一致。 |
| WAND-008 | DDL | DROP/RECREATE | 删除后确认不可检索/对象清理，再重建 | 不残留旧 postings/stats；重建结果一致。 |
| WAND-009 | 配置 | BM25 参数/会话设置 | 变更支持的 BM25 配置并重复查询 | 配置作用域/缓存语义明确，score 与相应 Oracle 一致。 |
| WAND-010 | 兼容 | 普通 FullText 回归 | 同数据使用既有 FullText parser/index 查询 | 普通 FullText DDL/查询/评分无回归。 |

### 2.2 WAND/BMW Top-K 正确性（15 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| WAND-011 | 基础 | 单 term Top-K | 不同 tf/doclen 文档，K=1/多 K | 主键、score、排序等于 exhaustive BM25。 |
| WAND-012 | 基础 | 多 term OR | rare/common term 组合，K=1/10/全部 | 精确 Top-K；任一命中 term 的文档按 BM25 总分比较。 |
| WAND-013 | 阈值 | heap 未满/已满 | K 大于/小于命中数，采集 trace/metrics | 未满前 θ 语义正确；满后 θ 单调不降。 |
| WAND-014 | pivot | cursor 对齐评分 | 手工构造所有 cursor 位于同 pivot doc | 该 doc 被完整评分并按真实分更新 heap。 |
| WAND-015 | pivot | lagging cursor 跳跃 | 低 docID cursor 与更高 pivot 混排 | 只能跳过 UB 不足的文档；最终 Top-K 无遗漏。 |
| WAND-016 | BMW | block-local bound 剪枝 | 全局 max 很松、当前 block max 很低的语料 | 可跳整个 block；结果等于 Oracle。 |
| WAND-017 | BMW | bound 恰等于阈值 | 构造 UB = θ、真实 score = θ/略高/略低 | 比较符号严格遵循实现定义；tie 不漏。 |
| WAND-018 | tie | 同分文档 | 多文档同 BM25 score，K 截断于 tie | 使用冻结稳定 tie-break（如 PK/docID）；重复执行一致。 |
| WAND-019 | K 边界 | LIMIT 0/1/N/N+1/无 LIMIT | 对命中数 M 各边界执行 | LIMIT 0 空；K≥M 返回全部；无下推时正确回退。 |
| WAND-020 | query | 重复 term/空 query | `red red apple`、空/仅停用词（按 parser） | term 去重/权重契约正确；空 query 稳定处理。 |
| WAND-021 | query | 不存在 term | 混合存在/不存在 term 与全不存在 | 不存在 term 不破坏 cursor；全不存在返回空。 |
| WAND-022 | doclen | 短/长文档 BM25 | 相同 tf、显著不同 doclen | score/排序符合 BM25 length normalization。 |
| WAND-023 | tf/df | 高频/罕见词 | 改变 tf 和 corpus df，比较 score | IDF/TF 影响与独立 Oracle 一致。 |
| WAND-024 | 属性 | 固定 seed 随机语料 | 多轮生成 term 分布、长度、K、query | 每轮 diff WAND 与 exhaustive；失败输出 seed/语料/query/K。 |
| WAND-025 | 保守回退 | bound/stat 缺失 | 故障注入或不完整 block metadata（单测） | 禁止不安全剪枝；保守评分或明确错误，绝不返回错误 Top-K。 |

### 2.3 计划、过滤与 SQL 组合（12 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| WAND-026 | 计划 | LIMIT Top-K 下推 | `MATCH ... ORDER BY score LIMIT k`，EXPLAIN/ANALYZE | k 下推为 heap size；retrieval 直接有序输出，无冗余 SQL Sort。 |
| WAND-027 | 计划 | LIMIT/OFFSET | 不同 OFFSET/LIMIT 与 score 排序 | 仅安全情形下推；最终分页结果等价 exhaustive SQL。 |
| WAND-028 | 过滤 | membership bitmap equality | `WHERE category='A'` + Top-K | bitmap 在 walk 中过滤；结果等于先过滤后 exhaustive BM25。 |
| WAND-029 | 过滤 | 低/中/高选择性 | 1%/10%/50%/无过滤矩阵 | 每格结果精确；记录评分数/跳过数/延迟。 |
| WAND-030 | 过滤 | 多谓词/NULL | AND、OR、IN、status NULL 与不可下推谓词 | 可下推项安全应用；其余保留 residual，不少返回。 |
| WAND-031 | liveness | delete bitmap | 删除 Top-K 候选文档后重复查询 | 删除文档不可返回/参与 θ，后续最佳 live 文档补位。 |
| WAND-032 | SQL | WHERE/投影/别名 | MATCH 在 WHERE、SELECT、ORDER BY 组合 | score 与行集一致，不重复/错算。 |
| WAND-033 | SQL | GROUP BY/HAVING | Top-K retrieval 与分组/聚合组合 | 仅符合语义时用索引；否则正确回退。 |
| WAND-034 | SQL | JOIN/CTE/derived | retrieval 表与维表 JOIN、CTE/派生表重写 | 计划改写不丢 FullText/普通谓词，结果与展开 SQL 相同。 |
| WAND-035 | 模式 | Natural/Boolean/Phrase | 各查询模式分别执行 | 仅受支持 BM25 OR Top-K 走 WAND；其他模式正确走原路径。 |
| WAND-036 | 兼容 | 无 LIMIT/非 score 排序 | MATCH 不按 score 排序、按普通列排序 | 不错误 Top-K 剪枝；全结果语义正确。 |
| WAND-037 | prepared | 计划缓存与 query 参数 | 多次参数化查询、改变 k/term/filter | 每次 heap/filter 正确更新，不串 query 状态。 |

### 2.4 DML、segment 与恢复（11 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| WAND-038 | INSERT | base 后增量写入 | 全量建立 base，插入可成为 Top-K 的文档 | tail tag=1 被检索；新文档正确进入 Top-K。 |
| WAND-039 | UPDATE | 更新文本/长度 | 将词频、term 集合、doclen 改为高/低评分 | 旧 posting/liveness 不污染；新 score 等于 Oracle。 |
| WAND-040 | UPDATE | 更新非索引列 | 仅 category/status 更新后带过滤查询 | FullText score 不变，membership bitmap 结果更新。 |
| WAND-041 | DELETE | 多 segment 删除 | base/tail 各删除高分文档 | liveness 在全部 segment 一致生效，无已删文档。 |
| WAND-042 | 事务 | commit/rollback | 事务内 insert/update/delete 后提交/回滚 | 可见性与 postings/docmap/liveness 原子一致。 |
| WAND-043 | compaction | tail 合并/base rebuild | 多 tail 后 compact/rebuild，前后查询 | 同一 Oracle 结果、score、tie；无文档丢失/重复。 |
| WAND-044 | cache | 索引对象/统计刷新 | DML、compact 后重复查询/多 session | N/avgdl/bounds 不陈旧，不串 segment 状态。 |
| WAND-045 | Snapshot | 快照恢复 | 多 segment/删除/更新后 snapshot restore | retrieval metadata、live docs、Top-K 完整恢复。 |
| WAND-046 | Backup/PITR | 备份恢复、不同 PITR 点 | 恢复后对每点运行 Oracle query set | 结果匹配该时点数据/索引状态。 |
| WAND-047 | 升级 | 旧 FullText/retrieval 元数据 | 升级加载/重建后查询 | 向后兼容或明确重建要求；不静默错误评分。 |
| WAND-048 | 异步 | async/CDC（支持时） | 创建异步索引，写入/更新后轮询 | 最终可见结果与同步索引一致，无 backlog 后错误。 |

### 2.5 异常、并发与可观测性（6 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| WAND-049 | 错误输入 | 非法 MATCH/parser/index 配置 | DDL/SQL 使用非法字段、参数、query | 清晰错误，无资源/隐藏表残留。 |
| WAND-050 | 损坏数据 | posting/docmap/block metadata 不一致（单测/故障注入） | 加载/查询异常索引状态 | 检测并安全失败/重建；不得越界、panic 或错误 Top-K。 |
| WAND-051 | 取消 | common-term 大语料检索取消 | 查询中 context deadline/cancel，随后健康查询 | 及时取消，无 heap/cursor/内存泄漏或后续污染。 |
| WAND-052 | 并发 | 多 query + DML/compact | 多 session Top-K/过滤查询，同时写入/compact | 无死锁、竞态、错误 score 或删除泄露。 |
| WAND-053 | Explain | WAND/BMW 诊断 | EXPLAIN ANALYZE/verbose retrieval query | 展示 index、pushed K、filter、segments；verbose 统计受控可读。 |
| WAND-054 | metrics | 评分/跳过/heap 观测 | 固定 query 采集 metrics/trace | 可关联 scored docs、skipped blocks/postings、θ、segments；无敏感文档文本泄露。 |

### 2.6 性能与长稳（6 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果/记录项 |
| --- | --- | --- | --- | --- |
| WAND-055 | common-term | 常见词大语料 | 与 exhaustive scorer 对照 K=10/100 | Top-K 100% 相同；记录评分减少率、跳块数、p50/p95、CPU。 |
| WAND-056 | K 矩阵 | K=1/10/100/1000/大 K | 固定语料/query set 对照 | 精确性不变；K 小时剪枝收益更明显，记录趋势。 |
| WAND-057 | block 参数 | 不同 block size/segment 大小 | 相同语料运行 BMW | 结果一致；记录 bound 紧度、block skip、索引大小与延迟。 |
| WAND-058 | 过滤矩阵 | membership bitmap 选择性 | 1/10/50/100% 过滤并对照 exhaustive | 过滤后 Top-K 精确；记录 bitmap 对评分/延迟影响。 |
| WAND-059 | 多 CN | 分布式查询吞吐 | 固定 query set、不同 CN/并发 | 结果一致；记录 QPS/p95、segment/cache 行为，无重复/漏结果。 |
| WAND-060 | continuous run | 官方 immutable image，唯一 DB 小语料 shakeout 后低负载 2h | 读写/查询混合，完整观测 | 无 correctness ledger 错误、backlog、内存增长或环境不稳；观测缺失记 FAIL。 |

## 3. Oracle 与结果判定

### 3.1 Exhaustive BM25 Oracle

1. 按 parser 产生 query/doc term 集，排除过滤和已删除文档后，对每个 OR-matching live doc 独立计算 BM25。
2. Oracle 使用冻结的 BM25 参数、`N`、`avgdl`、tf/df 与 stable tie-break；不得调用 retrieval index 代码、posting cursor、block bound 或 production scorer。
3. 逐 query 比较 Top-K 主键序列、score（浮点按声明 epsilon/ULP）和返回行数；任何差异均为正确性失败。

### 3.2 计划与剪枝判定

| 场景 | 必须观察到 | 禁止观察到 |
| --- | --- | --- |
| 支持的 score Top-K | K/过滤下推、retrieval 有序输出 | 额外全局 Sort、无理由评分所有匹配文档。 |
| BMW block skip | block upper bound 可证明跳过，最终等于 Oracle | UB 低估导致漏 Top-K。 |
| filter/liveness | 在 candidate 评分前排除不可见文档 | 已删除/过滤文档抬高 θ 或进入结果。 |
| 不支持 SQL/异常 stats | 安全回退或明确错误 | 仍用不完整数据剪枝并返回错误结果。 |

## 4. 结果记录要求

- 记录精确 commit/镜像、索引/Parser/BM25 参数、block size、CN/TN 拓扑、语料与 query set checksum、随机 seed 和 ground-truth 版本。
- 正确性报告逐 query 保存 query terms、K、过滤、live 文档数、Oracle/实际 PK 和 score、tie-break、segment 列表。
- 性能至少 3 轮，区分 warm-up/测量；记录 p50/p95、QPS、总命中数、实际评分数、跳 posting/block 数、heap θ、CPU/内存/IO、索引大小。
- 长稳前先完成唯一 DB、小语料、15 分钟 FULLTEXT2 shakeout；任何 correctness 错误、读写 backlog、环境不稳定或观测缺失均为 FAIL。
