# MatrixOne 向量索引窄基类型与量化测试设计

> 本设计覆盖 Feature [#25026](https://github.com/matrixorigin/matrixone/issues/25026)：IVFFLAT/IVFPQ/CAGRA 支持窄向量基类型及 `QUANTIZATION` 索引量化，并扩展基础数组/向量 SQL 函数对新类型的支持。

## 1. 测试原则

| 项目 | 说明 |
| --- | --- |
| 功能边界 | 基类型是原表向量列的声明类型；`QUANTIZATION` 仅控制索引 entries 内的副本类型，不能改变原表列的类型或原始值。 |
| 正确性优先 | 所有索引查询均以精确全表距离计算为基线；近似索引允许召回率差异，但不得返回不满足 WHERE、排序或 LIMIT 语义的行。 |
| 支持矩阵驱动 | 按索引算法、基类型、量化类型、距离算子、主键限制、CPU/GPU 构建分别覆盖合法和拒绝组合。 |
| 数值判定 | 结果为浮点时使用 epsilon；f16/bf16/整数量化允许精度下降，但需满足约定误差与 Recall@K 阈值。 |
| 双路径对照 | `gpu_mode=0/1` 的相同数据和 SQL 应返回相同结果（浮点按 epsilon）；无 GPU 环境仅执行 CPU 可用项并标记跳过。 |
| 兼容性 | 无 `QUANTIZATION` 的既有索引、VECF32/VECF64 既有函数、旧表和旧索引必须保持行为兼容。 |

### 1.1 基本信息

| 项目 | 内容 |
| --- | --- |
| MatrixOne 基线 | `main`（Feature #25026） |
| 基础类型 | `VECF32`、`VECF64`、`VECF16`、`VECBF16`、`VECINT8`、`VECUINT8` |
| IVFFLAT | CPU；支持 f32/f64/f16/bf16/int8/uint8 基类型及 f32/f16/bf16/int8/uint8 量化。 |
| IVFPQ/CAGRA | GPU/cuVS；仅 f32/f16 基类型，量化支持 f16/int8/uint8，需实验开关与 bigint PK。 |
| 距离算子 | L2、L2 squared、inner product、cosine；IVFFLAT 额外支持 L1。GPU 索引的 int8/uint8 量化仅限 L2。 |
| 函数范围 | `vector_dims`、norm、normalize、subvector、inner/cosine/L2 距离、base64 转换函数。 |

### 1.2 执行矩阵

| 测试类型 | CPU-only | GPU noarchsimd | GPU archsimd | 通过标准 |
| --- | --- | --- | --- | --- |
| IVFFLAT 与基础函数 | 执行 | 执行 | 执行 | 结果正确，类型与数值契约符合预期。 |
| IVFPQ/CAGRA | 跳过（环境限制） | 执行 | 执行 | 实验开关、GPU 构建及算法结果正确。 |
| GPU/CPU 对照 | 不适用 | 执行 | 执行 | 结果一致（浮点 epsilon）。 |
| SIMD 对照 | noarchsimd 基线 | noarchsimd 基线 | 执行 | 结果一致；性能单独记录。 |

### 1.3 覆盖范围

| 一级模块 | 用例数 | 主要覆盖内容 |
| --- | ---: | --- |
| 类型与 DDL | 11 | 列类型、维度、索引语法、实验开关、元数据 |
| IVFFLAT 与量化 | 12 | 全部基类型、量化、算子、DML、精度 |
| IVFPQ/CAGRA | 12 | GPU 前置、参数、量化、include、容量与回退 |
| 异常与兼容 | 9 | 非法组合、主键/类型限制、旧索引、错误信息 |
| 基础数组函数 | 12 | 单目/双目函数、边界、返回类型、base64 |
| 运行时与性能 | 8 | gpu_mode、构建变体、召回、空间、稳定性 |
| **合计** | **64** |  |

### 1.4 参考资料

| 资料 | 地址 |
| --- | --- |
| Feature 与用户验收示例 | <https://github.com/matrixorigin/matrixone/issues/25026> |
| IVF INCLUDE 测试设计模板 | <https://github.com/slyunnn/test/blob/main/%E6%B5%8B%E8%AF%95%E8%AE%BE%E8%AE%A1/feature%2325144-ivfflat-include-columns-test-design.md> |

## 2. 测试用例

### 2.1 类型与 DDL（11 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| VQ-001 | 列类型 | 全部窄类型建表 | 分别建 `VECF16/VECBF16/VECINT8/VECUINT8(d)` 表，插入、查询 | 建表和合法字面量读写成功；SELECT 保持声明类型。 |
| VQ-002 | 维度 | 固定维度校验 | 对各类型插入维度不足/超出/空向量、维度 1 和大维度 | 不匹配维度明确报错；边界合法维度稳定。 |
| VQ-003 | 范围 | int8/uint8 值范围 | 插入 -128/127、0/255 和越界整数 | 边界可用；越界值拒绝或按既定转换规则处理且无截断静默错误。 |
| VQ-004 | DDL | IVFFLAT 全基类型 | 各基类型创建无量化 IVFFLAT | 创建成功；entries 类型等于原基类型；最近邻结果可用。 |
| VQ-005 | DDL | IVFPQ/CAGRA f32/f16 | 打开实验开关后，分别对 VECF32/VECF16 + bigint PK 建索引 | 创建成功，SHOW 元数据准确。 |
| VQ-006 | 开关 | GPU 索引实验开关 | 未设置/设置 `experimental_ivfpq_index`、`experimental_cagra_index` 分别创建 | 未开启明确拒绝；开启后可创建对应索引。 |
| VQ-007 | 语法 | QUANTIZATION 格式与大小写 | 使用合法名称、大小写、空值、重复子句 | 合法值规范化；非法/重复子句明确报错。 |
| VQ-008 | 元数据 | SHOW CREATE/SHOW INDEX | 创建含量化索引后执行 SHOW | 量化类型、算法参数、基类型均可正确回显。 |
| VQ-009 | DDL | ASYNC 与 AUTO_UPDATE | 各算法分别创建 ASYNC、计划重建索引 | 异步完成后可用；计划参数保存且不丢失量化配置。 |
| VQ-010 | 参数 | 算法专有可选参数 | 覆盖 lists、m、bits、graph degree、itopk、训练百分比、容量 | 合法参数生效；默认值正确；无效算法参数被拒绝。 |
| VQ-011 | 生命周期 | DROP/重建 | DROP 含量化索引，再用不同量化重建 | 旧隐藏对象清理；新索引无陈旧配置。 |

### 2.2 IVFFLAT 与量化（12 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| VQ-012 | 基类型 | f16/bf16 最近邻 | 小数据精确构造，分别查询 L2 Top-K | 命中顺序与全表基线一致或满足可解释的 epsilon。 |
| VQ-013 | 基类型 | int8/uint8 最近邻 | 包含正负边界（int8）与 0/255（uint8）的数据 | 距离与 Top-K 正确，无符号/符号混淆。 |
| VQ-014 | 基类型 | VECF64 回归 | VECF64 建 IVFFLAT，执行距离查询 | 既有 f64 能力无回归。 |
| VQ-015 | 量化 | f32→f16/bf16 | 分别创建 `QUANTIZATION 'float16'/'bf16'` | 原表仍 VECF32；entries 按目标类型存储；查询可用。 |
| VQ-016 | 量化 | f32→int8/uint8 | 分别创建整数量化索引并检索 | 索引创建/查询成功；记录 Recall@K 和距离误差。 |
| VQ-017 | 量化 | 无量化基线 | 同数据比较无量化与各量化索引 | 无量化 entries 保留基类型；量化只影响索引路径。 |
| VQ-018 | 算子 | L2/L2SQ | 各基类型/量化执行 L2 与 L2 squared | 两者排序一致，数值满足 sq = l2²（epsilon）。 |
| VQ-019 | 算子 | IP/cosine | float 系列量化/非量化使用 IP、cosine | SQL 正确执行，排序/结果与精确计算基线一致。 |
| VQ-020 | 算子 | L1 | IVFFLAT 全基类型使用 L1 | 支持并正确；GPU 索引不应误接受 L1。 |
| VQ-021 | DML | INSERT/UPDATE/DELETE | 索引后写入、更新向量、删除行 | 索引条目同步；旧向量不再命中，无残留。 |
| VQ-022 | 事务 | 提交与回滚 | 在事务内进行各类 DML 并跨 session 查询 | 遵循隔离语义；回滚不污染 entries。 |
| VQ-023 | 训练 | 小样本/训练参数边界 | lists 与训练比例接近行数边界，执行建索引/查询 | 失败时错误清晰且无残留；成功时索引稳定。 |

### 2.3 IVFPQ/CAGRA（12 条，GPU）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| VQ-024 | IVFPQ | f32/f16 + 无量化 | 创建 IVFPQ 并 Top-K 查询 | GPU 索引建立、使用并返回有效近邻。 |
| VQ-025 | IVFPQ | f16/int8/uint8 量化 | 三种量化分别创建、查询、比对 Recall@K | 配置生效，结果满足设定召回阈值。 |
| VQ-026 | IVFPQ | PQ 参数 | 覆盖 `m`、`bits_per_code`、lists、训练参数 | 合法组合成功；不合法维度/参数组合报错。 |
| VQ-027 | CAGRA | f32/f16 + 无量化 | 创建 CAGRA、执行 Top-K | 索引可用，结果正确且稳定。 |
| VQ-028 | CAGRA | f16/int8/uint8 量化 | 分别检索，比较精确基线 | 配置生效，整数量化只在 L2 下使用。 |
| VQ-029 | CAGRA | 图参数 | 覆盖 intermediate_graph_degree、graph_degree、itopk_size | 参数下传正确；非法范围拒绝。 |
| VQ-030 | 容量 | max_index_capacity | 0 自动、恰好容量、超容量 DML | 自动容量正常；超过限制的行为、错误和恢复符合设计。 |
| VQ-031 | INCLUDE | GPU 索引 include 标量列 | IVFPQ/CAGRA include int/float 列并带过滤查询 | include 条件与向量结果正确；不影响量化路径。 |
| VQ-032 | 更新 | AUTO_UPDATE/REINDEX | 数据变更后自动/手工重建 | 新索引保持量化类型、参数和查询正确。 |
| VQ-033 | 溢出路径 | cuVS overflow | 构造超出主 GPU 索引候选的查询/容量情形 | 回退路径无崩溃，结果和 GPU/CPU 契约一致。 |
| VQ-034 | 并发 | GPU 读写并发 | 并发查询、插入、更新、索引重建 | 无死锁、显存错误或交叉结果。 |
| VQ-035 | 可观测性 | EXPLAIN/ANALYZE | 三算法各执行 explain analyze | 显示算法/索引路径；配置不被错误隐藏。 |

### 2.4 异常与兼容（9 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| VQ-036 | 约束 | GPU 索引不支持 f64/bf16/int8/uint8 基类型 | 对 IVFPQ/CAGRA 使用这些基类型建索引 | 明确拒绝，提示仅允许 f32/f16。 |
| VQ-037 | 约束 | GPU 索引非 bigint PK | 使用 int、varchar、复合 PK 建 IVFPQ/CAGRA | 明确拒绝；IVFFLAT 对原有 PK 能力无回归。 |
| VQ-038 | 量化 | GPU bf16 量化 | IVFPQ/CAGRA 指定 `bf16` | 明确拒绝。 |
| VQ-039 | 量化 | int8/uint8 + 非 L2 | 在 GPU 索引使用 IP、cosine（及 L2SQ 如不支持） | 拒绝并说明整数量化仅 L2，不能产生错误近邻。 |
| VQ-040 | 算法 | IVFPQ/CAGRA L1 | 指定 L1 op_type | 明确拒绝。 |
| VQ-041 | 输入 | 不可解析向量/NaN/Inf | 对全部新类型插入非法文本、NaN、Inf | 安全失败或按既定规则处理，无 panic/脏索引。 |
| VQ-042 | 兼容 | 旧无量化索引 | 升级/加载旧 IVFFLAT、IVFPQ、CAGRA 元数据 | 无量化配置默认等价原基类型，索引可查询。 |
| VQ-043 | 兼容 | 已有函数与 f32/f64 | 执行旧函数 SQL、prepared statement、视图 | 返回值和错误行为保持兼容。 |
| VQ-044 | 恢复 | 备份/快照/PITR | 量化索引备份恢复、时间点恢复 | 基类型、量化参数和数据结果完整恢复。 |

### 2.5 基础数组/向量函数（12 条）

对 f16、bf16、int8、uint8 分别执行本节函数；结果数值与等价 f32 输入比对。窄类型计算按 f32 进行，标量结果为 `float64`。

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| VQ-045 | 单目 | `vector_dims` | 各维度/类型调用 | 返回 int64 且等于声明维度。 |
| VQ-046 | 单目 | `summation`、`l1_norm`、`l2_norm` | 负数、零、正数、整数边界输入 | 数值符合 f32 基线与 epsilon。 |
| VQ-047 | 单目 | `normalize_l2` | 非零向量调用并再算 norm | 返回同类型数组，L2 norm 约为 1。 |
| VQ-048 | 单目 | 零向量 normalize | 对全零向量调用 | 返回/错误符合既定契约，无 NaN、panic。 |
| VQ-049 | 单目 | `subvector` | start=1、末尾、负 start、有/无 len | 1 基索引、负索引和长度边界正确，返回同类型。 |
| VQ-050 | 双目 | `inner_product` | 同/异类型可支持组合与向量字面量 | 数值正确，结果为 float64。 |
| VQ-051 | 双目 | cosine similarity/distance | 正交、相同、反向和零向量 | 数值范围/互补关系正确；零向量契约一致。 |
| VQ-052 | 双目 | `l2_distance`/`l2_distance_sq` | 多类型、负值、整型边界 | 满足平方关系，结果为 float64。 |
| VQ-053 | 维度 | 二元函数维度不匹配 | 对所有二元函数传不同维度 | 明确报错，无隐式截断。 |
| VQ-054 | SQL 组合 | SELECT/WHERE/GROUP BY/视图 | 在过滤、聚合、prepared statement 中调用 | 类型推导、执行与结果正确。 |
| VQ-055 | 编码 | `vec*_from_base64` | 六种类型合法/截断/非法 base64 输入 | 合法输入得到正确类型和维度；非法输入明确失败。 |
| VQ-056 | NULL | NULL 传播 | 各函数传 NULL 向量/NULL 参数 | 行为遵循 SQL NULL 规则，无 panic。 |

### 2.6 运行时、性能与稳定性（8 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果/记录项 |
| --- | --- | --- | --- | --- |
| VQ-057 | gpu_mode | CPU/GPU 对照 | 同一 GPU build 分别 `SET gpu_mode=0/1`，跑距离函数和 IVF 查询 | 主键结果一致；浮点误差不超 epsilon。 |
| VQ-058 | 默认值 | gpu_mode 默认 | CPU/GPU build 各新 session 查询变量 | CPU 默认 0，GPU build 默认 1。 |
| VQ-059 | 构建 | CPU noarchsimd vs archsimd | 同种子完整函数/IVFFLAT 套件 | 结果一致；记录耗时。 |
| VQ-060 | 构建 | GPU noarchsimd vs archsimd | IVFPQ/CAGRA、函数和 gpu_mode 套件 | 结果一致；记录吞吐、延迟、GPU 内存。 |
| VQ-061 | 召回 | 量化 Recall@K | 固定随机种子和查询集，以全表精确 Top-K 为真值 | 记录各算法/量化/算子的 Recall@1、@10、@100。 |
| VQ-062 | 精度 | 数值误差分布 | 记录函数/距离的最大、平均、p99 绝对/相对误差 | 符合为各类型预设的阈值；超阈值保留样本。 |
| VQ-063 | 存储与构建 | 量化收益 | 比较无量化与量化索引的 entries 大小、建索引耗时、查询延迟 | 记录压缩收益与性能，不以牺牲正确性换指标。 |
| VQ-064 | 长稳 | 多轮并发与重建 | 持续查询、DML、REINDEX/自动更新 | 无内存/显存泄漏、崩溃、结果漂移或资源耗尽。 |

## 3. 基线与结果判定

### 3.1 精确基线

1. 小数据：在 SQL 中对全表计算同一距离函数，按 `distance ASC, pk ASC` 排序，核对完整结果。
2. 大数据：离线或禁用索引计算精确 Top-K；索引查询比较 Recall@K、过滤后行集合和距离误差。
3. 量化：必须同时验证原列值未改变、entries 量化配置已生效、查询误差在阈值内。

### 3.2 浮点与近似判定

| 项目 | 判定 |
| --- | --- |
| CPU/GPU、SIMD/标量 | 标量函数和精确路径以绝对/相对 epsilon 对比；排序并列时使用 PK 稳定排序。 |
| f16/bf16 | 误差阈值按数据幅度与类型精度设定，并保留原始向量、实际值、基线值。 |
| int8/uint8 量化 | 只判定支持的 L2 路径；按固定数据集和参数记录 Recall@K，不接受错误过滤或越界值。 |
| 近似索引 | 预先声明数据规模、lists/probe、图/PQ 参数和最低 Recall@K；阈值变动需说明。 |

## 4. 结果记录要求

- 记录版本、commit、CPU/GPU 型号、CUDA/cuVS、构建变体、`gpu_mode`、实验开关及完整索引参数。
- 每个 DDL/负例保留 SQL、错误码/错误文本、SHOW CREATE/SHOW INDEX 与隐藏 entries schema。
- 每个检索用例保留查询向量、精确 Top-K、索引 Top-K、Recall@K、距离误差及 EXPLAIN。
- 性能至少三次运行，记录 p50/p95、QPS、索引大小、构建时间、CPU/GPU 内存；仅在相同环境与数据快照下横向比较。
- GPU 或 archsimd 不可用时，显式标记为环境跳过，不能误判为功能失败。
