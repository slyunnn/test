# MatrixOne JSON Word Breaker 全文索引测试设计

> 本设计覆盖 Feature [#27704](https://github.com/matrixorigin/matrixone/issues/27704)：新增 JSON word breaker，使 FullText 索引能为 JSON 的结构化键值与原子字符串建立可检索 token，并由优化器把 JSON 谓词改写为“FullText 候选过滤 + 原 JSON 谓词复核”。目标是加速 JSON 查询，同时不改变 SQL 的正确结果。

## 1. 测试原则

| 项目 | 说明 |
| --- | --- |
| 正确性优先 | `ft_contains` 仅用于缩小候选；最终结果必须重新计算原 `json_extract_*` 谓词。任何 token 冲突、路径碰撞或近似分词都不能导致漏行/错行。 |
| 双重索引 | 默认 JSON word breaker 同时产生结构化键值 token 和 NLP token：短字符串键值对（建议阈值 100 bytes）/数值按 tuple 编码，所有原子字符串（数组元素和对象 value）均送普通 NLP word breaker。 |
| 路径语义 | 默认仅用末级 key 的编码须覆盖同名 key 路径碰撞；支持 full-path 时，编码必须能区分完整路径，并在配置切换后保持可解释行为。 |
| 数值有序性 | 数值 tuple 编码须保持 SQL 数值比较的字节序/比较序，支持 `=、<、<=、>、>=` 候选检索；结果仍由 JSON 数值谓词复核。 |
| 可观测性 | 每个优化用例同时检查 EXPLAIN 与结果；记录改写的 `ft_contains`、残余 JSON 谓词、tokenizer 配置和候选/最终行数。 |
| 兼容性 | 既有 JSON 函数、普通 FullText、默认/ngram/gojieba parser 及不带 JSON word breaker 的索引不得回归。 |

### 1.1 基本信息

| 项目 | 内容 |
| --- | --- |
| MatrixOne 基线 | Feature #27704 合入后的精确 commit/镜像（执行前冻结）。 |
| 功能入口 | JSON 列上的 FullText 索引，使用新的 JSON word breaker/parser；优化器识别可改写的 `json_extract_*` 谓词。 |
| 结构化 token | 当前语义可表示为末级 key/value（例如 `{"a":{"b":"XXX"}}` 产生 `b=XXX`）；实现应使用 tuple 编码避免分隔符歧义。 |
| NLP token | 对全部原子字符串，使用配置的普通 word breaker（例如 GoJieba）；不受对象/数组位置及字符串长度限制。 |
| 短字符串阈值 | 结构化字符串 token 的默认建议为 `<100 bytes`；实际配置名、默认值和单位以实现为准并写入测试记录。 |
| path 配置 | 至少覆盖 default（末级 key）及 `with_full_path`（如实现提供）；路径模式影响 token 召回，不影响 JSON SQL 语义。 |

### 1.2 执行矩阵

| 测试类型 | 无 JSON word breaker | JSON word breaker | 通过标准 |
| --- | --- | --- | --- |
| JSON 函数回归 | 执行 | 执行 | JSON SQL 返回结果完全一致。 |
| 结构化 token | 不适用 | 执行 | key/value、路径、类型、阈值及转义按契约编码。 |
| NLP token | 普通 parser 基线 | 执行 | JSON 原子字符串能按普通 NLP 规则召回。 |
| 优化器改写 | 不适用 | 执行 | 可改写谓词出现候选过滤和 residual JSON 复核；不可改写谓词安全回退。 |
| 数值范围 | 全表精确基线 | 执行 | 结果等价基线，排序编码不产生漏/错候选。 |
| 生命周期/恢复 | 执行 | 执行 | DML、重建、备份恢复和配置切换一致。 |

### 1.3 覆盖范围

| 一级模块 | 用例数 | 主要覆盖内容 |
| --- | ---: | --- |
| DDL 与 token 结构 | 11 | parser、元数据、键值、类型、阈值、转义 |
| 路径、数组与 NLP | 11 | full path、同名 key、数组、嵌套、中文/英文、长文本 |
| 优化器与正确性 | 13 | equality/range 改写、残余复核、组合谓词、回退 |
| 数值与边界 | 9 | 整数/浮点、范围、NULL、缺失、特殊字符、深度 |
| 一致性、恢复与异常 | 9 | DML、事务、并发、reindex、升级、故障 |
| 性能与可观测性 | 7 | 候选削减、选择性、索引大小、Explain、稳定性 |
| **合计** | **60** |  |

### 1.4 参考资料

| 资料 | 地址或位置 |
| --- | --- |
| Feature | <https://github.com/matrixorigin/matrixone/issues/27704> |
| Feature 讨论 | <https://github.com/matrixorigin/matrixone/issues/27704#issuecomment-5443723228> |
| FullText GoJieba 测试模板 | <https://github.com/Ariznawlll/mo-test-reports/blob/main/fulltext-gojieba-test/gojieba%E5%85%A8%E6%96%87%E7%B4%A2%E5%BC%95%E6%B5%8B%E8%AF%95%E8%AE%BE%E8%AE%A1.md> |

## 2. 测试用例

### 2.1 DDL 与结构化 token（11 条）

通用表：`t(id bigint primary key, doc json, note varchar(200))`。基础数据至少包含：`{"a":{"b":"XXX","c":"YYY"},"tag":"北京清华大学","n":3.14159,"ok":true,"nil":null,"arr":["alpha","中文词",7]}`。

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| JWB-001 | DDL | 单 JSON 列创建索引 | 创建 JSON FullText + JSON word breaker 索引；SHOW/ DESC | 创建成功，parser 配置和隐藏表 metadata 可见且稳定。 |
| JWB-002 | DDL | 建表内/ALTER 创建 | 分别在 CREATE TABLE、ALTER TABLE 上定义索引，写入历史/新增数据 | 两种入口均可用；存量和增量 JSON 均完成索引。 |
| JWB-003 | DDL | 非 JSON 列/非法 parser | 对 varchar、未知 parser、缺主键创建 | 明确拒绝；无半成品隐藏对象。 |
| JWB-004 | 元数据 | SHOW CREATE/SHOW INDEX | 输出定义、parser 参数及 path/阈值配置 | 可恢复回放；不会丢失 JSON word breaker 选项。 |
| JWB-005 | key-value | 简单字符串对象 | `{"b":"XXX"}`，查询 tokenizer/内部 token | 产生结构化键值 token；tuple/转义实现不依赖不安全文本拼接。 |
| JWB-006 | key-value | 嵌套对象末级 key | `{"a":{"b":"XXX"}}` | default 模式按末级 key 生成候选；SQL 复核保留完整 path 语义。 |
| JWB-007 | 标量类型 | string/number/bool/null | 分别索引并检查可索引 token 和查询 | 数值按数值编码；字符串、布尔、null 的索引/跳过策略符合冻结契约。 |
| JWB-008 | 阈值 | 小于/等于/大于短文本阈值 | 在默认阈值 T 的 `T-1/T/T+1` byte 字符串插入 key:value | 结构化 token 边界符合定义；所有原子字符串仍产生 NLP token。 |
| JWB-009 | 转义 | `=、引号、反斜杠、UTF-8` | key/value 含分隔符、控制字符、emoji、中文 | token 无冲突、截断或非法 UTF-8；查询仍精确。 |
| JWB-010 | 多列 | JSON 与普通文本多列索引 | 创建 `(doc,note)` 全文索引，分别检索 | 两列 token 边界清晰；JSON breaker 不污染普通文本 token。 |
| JWB-011 | 重建 | DROP/REINDEX | 删除、重建同一 JSON 索引，比较 token/查询 | 旧 metadata/隐藏表清理；新定义正确生效。 |

### 2.2 路径、数组与 NLP token（11 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| JWB-012 | 路径 | 同名末级 key 碰撞 | `a.b='X'` 与 `c.b='X/Y'`，分别用完整 JSON path 查询 | 即使 default token 候选扩大，residual JSON 复核后结果无错行/漏行。 |
| JWB-013 | 路径 | with_full_path 配置 | 用 full path 创建索引，对 `a.b`/`c.b` 精确查询 | token 可区分完整路径；计划候选数不高于末级 key 的同数据基线。 |
| JWB-014 | 路径 | path 配置切换 | default/full-path 分别建索引、运行相同 SQL | SQL 结果相同；仅 token 与候选/计划允许不同。 |
| JWB-015 | 路径 | 深层及空 key | 多层对象、空对象、合法特殊 key | 深度遍历稳定；空对象不产出伪 token。 |
| JWB-016 | 数组 | 字符串/数字/对象数组 | 数组中混合原子值、对象、嵌套数组 | 每个原子字符串进入 NLP；数值按契约编码；无跨元素拼接。 |
| JWB-017 | 数组 | 相同值不同位置 | 同一字符串出现在对象 value 与数组元素 | 两处均可 NLP 召回；结构化 token 的 path/位置策略可解释。 |
| JWB-018 | NLP | 中文词 | JSON value 为“我来到北京清华大学”，按配置 NLP 查询 | token/召回符合选定 NLP parser（如 GoJieba），不退化为结构化 token。 |
| JWB-019 | NLP | 英文、大小写、标点 | JSON value 含 `Hello, WORLD!`、型号/数字 | 普通 word breaker 归一/切分规则与非 JSON 文本一致。 |
| JWB-020 | NLP | 长字符串 | 超过结构化阈值的中文/英文文本 | 不生成短键值 token，但仍可按 NLP 查询召回。 |
| JWB-021 | NLP | 空串/空白/仅标点 | JSON 原子字符串的边界值 | 无空 token、panic 或异常文档长度。 |
| JWB-022 | 对照 | JSON breaker 与普通 parser | 同 JSON 序列化文本分别用两个 parser 索引 | 记录并验证结构化召回差异；普通全文能力不回归。 |

### 2.3 优化器改写与正确性（13 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| JWB-023 | equality | `json_extract_string(path)='XXX'` | 比较禁用索引的精确结果与启用索引结果、EXPLAIN | 改写生成 FullText 候选过滤与原 JSON residual；结果完全相同。 |
| JWB-024 | equality | 缺失 path/不同类型 | JSON 中 path 缺失、值为 number/bool/null、字符串对照 | 不把候选命中视作最终命中；SQL 类型语义保持。 |
| JWB-025 | equality | 末级 key 假阳性 | 同名 key 位于不同嵌套路径 | FullText 可多候选，但 residual 精确消除假阳性。 |
| JWB-026 | full path | 完整路径改写 | full-path parser 上精确 path equality | 改写使用 path-aware token；结果与 default/全表基线一致。 |
| JWB-027 | range | `json_extract_float64(path)>x` | 正/负、小数、边界值、多行数据 | 数值 tuple 候选与 residual 联用；结果等于精确数值过滤。 |
| JWB-028 | range | `<、<=、>=、BETWEEN` | 对每个比较符及相等边界执行 | 不出现边界漏行/错行；候选编码保持数值排序。 |
| JWB-029 | range | 整数与浮点混合 | 0、负数、极值、科学记数法/可表示浮点 | JSON 数值转换与比较契约一致，无字典序比较错误。 |
| JWB-030 | 组合 | JSON AND 普通谓词 | JSON equality/range 与表列过滤、LIMIT/ORDER BY | 谓词均保留，LIMIT 发生在最终复核后。 |
| JWB-031 | 组合 | 多 JSON AND/OR | 可改写与不可改写谓词混合，特别是跨 path OR | AND 仅安全下推部分；不安全 OR 保守回退，逻辑结果正确。 |
| JWB-032 | SQL 组合 | JOIN/CTE/derived/子查询 | JSON predicate 出现在不同查询块 | 改写作用域正确，不丢谓词或错误跨块下推。 |
| JWB-033 | 回退 | 不支持 JSON 函数/动态 path | 使用非目标 extract、动态 JSON path、复杂表达式 | 不强行改写；走原路径且结果正确。 |
| JWB-034 | 参数化 | prepared statement | path/value 通过参数或常量多次执行 | 安全绑定、计划缓存不串 token/参数；无法安全改写时回退。 |
| JWB-035 | 安全 | 恶意字符串 | value 含引号、注释符、分号、反斜杠 | 内部 token/`ft_contains` 构造已转义；无 SQL 注入或语法破坏。 |

### 2.4 数值与边界（9 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| JWB-036 | 数值编码 | 负数、零、正数 | 数值按升序/降序插入并做范围查询 | token 比较序满足数值比较，最终结果精确。 |
| JWB-037 | 数值编码 | 整数边界 | 64 位安全边界、大小整数、负边界 | 不溢出、不按字符串字典序排序。 |
| JWB-038 | 数值编码 | 浮点边界 | 近似相等、小数、正/负零、极大/小有限值 | 结果与 JSON float 语义一致；明确 epsilon/精确比较策略。 |
| JWB-039 | 特殊值 | NaN/Infinity（JSON/函数支持时） | 解析、索引、比较及回退路径 | 行为冻结为接受/拒绝之一；各路径一致，无 panic。 |
| JWB-040 | NULL | SQL NULL、JSON null、path 缺失 | 对 equality/range 与 IS NULL 类表达式（支持时）分别测试 | 三者不混淆；不支持改写时保守回退。 |
| JWB-041 | 大 JSON | 大对象/深嵌套/多数组 | 建索引、查询、reindex | 无栈溢出、超时、无限 token 或内存异常。 |
| JWB-042 | token 限制 | 超长 key/value/大量字段 | 接近/超过 token 长度、字段数量限制 | 截断/跳过策略索引与查询侧一致，告警/错误可诊断。 |
| JWB-043 | 编码版本 | tuple 编码边界 | 相邻数值、相邻字符串、相同 key 不同 value | 编码一一映射或 collision 被 residual 安全消除。 |
| JWB-044 | 多语言 | 简繁、中英日韩及 emoji | 同时跑结构化 equality 和 NLP 查询 | 原文/分词契约稳定，不隐式破坏 UTF-8。 |

### 2.5 一致性、恢复与异常（9 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| JWB-045 | INSERT | 索引后写入 JSON | 插入新 key/value、数组文本、数值后立即查询 | 结构化/NLP token 均可见，结果正确。 |
| JWB-046 | UPDATE | 更新/删除嵌套值 | 替换 value、删除 path、改类型 | 旧 token 不再命中，新 token 正确；无残留。 |
| JWB-047 | 事务 | commit/rollback | 多 session 对 JSON DML 与查询 | 原表、FullText 和优化器结果原子一致。 |
| JWB-048 | 并发 | 读写并发 | 多 writer 更改 JSON，读者执行 equality/range/NLP | 无错行、死锁、panic 或 token 串行污染。 |
| JWB-049 | ASYNC | 异步 FullText/CDC（支持时） | 创建 async 索引并写入，轮询候选/最终结果 | 遵循最终可见性；token 与同步索引一致。 |
| JWB-050 | 备份恢复 | snapshot/逻辑备份/PITR | 建索引、更新 JSON 后恢复 | parser/path 配置、token 与查询结果恢复。 |
| JWB-051 | 升级 | 旧 JSON/FullText 索引 | 升级加载无 JSON breaker 的旧 metadata | 旧索引行为不变；新 breaker 只在显式建索引后生效。 |
| JWB-052 | 词典/配置故障 | NLP 词典或 breaker 配置缺失 | 建索引、查询、重启 | 明确失败或安全回退；不写半成品索引。 |
| JWB-053 | 错误输入 | 非法 JSON/索引创建失败 | 插入非法 JSON、DDL/DML 中断 | 错误定位明确，无隐藏表、临时 token 或资源泄漏。 |

### 2.6 性能与可观测性（7 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果/记录项 |
| --- | --- | --- | --- | --- |
| JWB-054 | 候选缩减 | 高选择性 key:value equality | JSON 大表中比较全表与 breaker 索引 | 结果相同；记录候选/最终行、延迟、扫描与索引命中。 |
| JWB-055 | 路径碰撞 | 低选择性同名 key | default 与 full-path 比较 | 结果相同；记录 path token 对候选数和延迟的影响。 |
| JWB-056 | 数值范围 | 低/中/高选择性 range | 各选择性下对照精确全表 | Recall/结果为 100%；记录候选数、p50/p95、CPU。 |
| JWB-057 | NLP | 长文本/中文词查询 | JSON breaker 与普通全文 parser 对照 | 记录召回、延迟、token/索引大小；结果语义合理。 |
| JWB-058 | 存储 | 阈值/路径模式矩阵 | 改变短文本阈值和 path 模式 | 记录 entries/token 数、建索引耗时、索引大小与查询收益。 |
| JWB-059 | Explain | 改写证据 | equality/range/回退各执行 EXPLAIN ANALYZE | 可见候选 `ft_contains`、residual JSON 谓词、候选/最终行；回退无伪改写。 |
| JWB-060 | 长稳 | 混合查询与 DML | 固定 seed、并发持续运行 | 无内存/索引文件增长、结果漂移或资源泄漏。 |

## 3. 基线与判定

### 3.1 正确性基线

1. 对每个可改写 SQL，强制禁用 FullText/优化改写或使用无索引副本执行原 `json_extract_*` 谓词，作为精确基线。
2. 比较完整主键集合、投影值、聚合/排序/分页结果；不能只比较 FullText 候选集。
3. 数值范围测试依据 JSON 函数的实际数值转换结果做比较；字节 tuple 排序仅是候选优化，不是最终语义来源。

### 3.2 计划判定

| 场景 | 必须观察到 | 禁止观察到 |
| --- | --- | --- |
| 可改写 equality/range | FullText 候选过滤、原 JSON residual 复核、可解释 token/path 配置 | 仅依赖 token 返回结果或丢失原 JSON 谓词。 |
| default 末级 key | 允许路径碰撞扩大候选 | 假阳性直接返回，或因碰撞漏行。 |
| full-path | path-aware token/过滤 | 用末级 key 冒充 full-path 且不复核。 |
| 不支持表达式 | 原 JSON 路径安全执行 | 生成不等价的 `ft_contains` 或内部 SQL 拼接错误。 |

## 4. 结果记录要求

- 记录 MatrixOne commit、JSON word breaker/NLP parser 配置、short-string 阈值、path 模式、词典版本、集群拓扑和数据 seed。
- 每条优化用例保留原 SQL、改写后计划、token/候选条件、residual JSON 条件、精确基线和最终结果。
- 性能数据至少运行 3 轮，记录 p50/p95、候选/最终行数、扫描、建索引时间、索引大小、CPU/内存；只在相同数据快照下比较。
- 缺陷必须提供最小 JSON 文档、path、SQL、parser 配置、预期/实际主键集合与 EXPLAIN，以区分 token、改写、JSON 函数或执行器问题。
