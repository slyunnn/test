# MatrixOne 向量索引窄基类型与量化测试设计（必要项评审版）

> 本设计覆盖 Feature [#25026](https://github.com/matrixorigin/matrixone/issues/25026)：IVFFLAT/IVFPQ/CAGRA 支持窄向量基类型及 `QUANTIZATION` 索引量化，并扩展基础数组/向量 SQL 函数对新类型的支持。

> 2026-10-28 必要项评审：64 条收敛为 34 个执行单元；沿用原编号。按[Issue验收说明](https://github.com/matrixorigin/matrixone/issues/25026)保留类型/量化矩阵、基础函数及CPU/GPU分派证据。执行时冻结源码SHA、cuVS与构建产物；本文不声明已运行GPU或数值测试。HNSW扩展由#31467负责，不混入本Feature。

> 2026-10-29 设计优化：以研发的 Feature 设计为测试依据，源码和已有用例用于定位入口、冻结实际版本及识别设计冲突，不能用当前实现行为倒推或弱化 Feature 预期。研发已确认重复 `quantization` 子句属于 MySQL parser 行为，不属于索引功能，本设计移除该子场景；其他不一致保留测试和实际结果，待研发确认或修复。

## 1. 测试原则

| 项目 | 说明 |
| --- | --- |
| 功能边界 | 基类型是原表向量列的声明类型；`QUANTIZATION` 仅控制索引 entries 内的副本类型，不能改变原表列的类型或原始值。 |
| 正确性优先 | 所有索引查询均以精确全表距离计算为基线；近似索引允许召回率差异，但不得返回不满足 WHERE、排序或 LIMIT 语义的行。 |
| 支持矩阵驱动 | 每种新增基类型、量化类型、算法与拒绝边界至少有直接证据；不把所有参数与拓扑做笛卡尔积。三算法仍分别执行，不能用IVFFLAT通过代替GPU通过。 |
| 数值判定 | 结果为浮点时使用 epsilon；f16/bf16/整数量化允许精度下降，但需满足约定误差与 Recall@K 阈值。 |
| 双路径对照 | 对受gpu_mode影响的相同候选计算路径比较CPU/GPU数值与无并列fixture结果；不将独立构建的ANN索引任意Top-K差异误判为数值错误。无GPU标记环境跳过。 |
| 兼容性 | 无 `QUANTIZATION` 的既有索引、VECF32/VECF64 既有函数、旧表和旧索引必须保持行为兼容。 |

### 1.1 基本信息

| 项目 | 内容 |
| --- | --- |
| MatrixOne 基线 | 执行前冻结精确SHA、镜像、GPU/cuVS及构建标签；Issue是需求参考，不以浮动main作为执行证据。 |
| 基础类型 | `VECF32`、`VECF64`、`VECF16`、`VECBF16`、`VECINT8`、`VECUINT8` |
| IVFFLAT | CPU；支持 f32/f64/f16/bf16/int8/uint8 基类型及 f32/f16/bf16/int8/uint8 量化。 |
| IVFPQ/CAGRA | GPU/cuVS；仅 f32/f16 基类型，量化支持 f16/int8/uint8，需实验开关与 bigint PK。 |
| 距离算子 | L2、L2 squared、inner product、cosine；IVFFLAT 额外支持 L1。GPU 索引的 int8/uint8 量化仅限 L2。 |
| 函数范围 | `vector_dims`、norm、normalize、subvector、inner/cosine/L2 距离、base64 转换函数。 |

### 1.2 执行矩阵

| 环境 | 必要范围 | 通过标准 |
| --- | --- | --- |
| CPU noarchsimd | IVFFLAT、新类型及基础函数 | 类型、值及Oracle正确 |
| GPU noarchsimd | IVFPQ/CAGRA、gpu_mode=0/1 | GPU索引与受影响分派路径直接对照 |
| CPU/GPU archsimd | 仅059的新增窄类型内核代表fixture | 与各自noarchsimd数值/错误一致，不重复完整DDL/生命周期 |
| 不具备的环境 | 明确标记跳过及影响 | 不以CPU成功推断GPU/SIMD通过 |

### 1.3 必要覆盖与精简说明

| 一级模块 | 执行单元 | 归并 |
| --- | ---: | --- |
| 类型与语法 | 4 | 非法文本/非有限输入并入001；范围归003 |
| IVFFLAT与量化 | 8 | 012/013/014/017归004/015/016；事务并入021 |
| GPU算法与拒绝边界 | 10 | 006/025/028/029/038/040归对应算法或负例owner |
| 基础函数 | 8 | 零归一化归047；维度/NULL为所有函数共用断言 |
| 兼容、分派与质量 | 4 | 默认变量归057；构建差异归059；数值误差随函数与061核对 |
| **合计** | **34** | SHOW/entries/EXPLAIN随每个索引owner采集，不另跑重复case |

删除独立ASYNC调度、全算法专有参数扫表、DROP循环、通用备份/PITR、并发重建长稳与规模化压测；本次直接相关的持久量化配置重建及overflow仍保留。空间正确性由entries类型验证，不额外要求三轮压缩收益/吞吐基准。

### 1.4 参考资料

| 资料 | 地址 |
| --- | --- |
| Feature 与用户验收示例 | <https://github.com/matrixorigin/matrixone/issues/25026> |
| IVF INCLUDE 测试设计模板 | <https://github.com/slyunnn/test/blob/main/%E6%B5%8B%E8%AF%95%E8%AE%BE%E8%AE%A1/feature%2324167-ivfflat-include-columns-test-design.md> |

## 2. 必要测试用例

### 2.1 类型与语法（4 个执行单元）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| VQ-001 | 列类型/输入 | 四种窄类型读写与非法值 | VECF16/VECBF16/VECINT8/VECUINT8建表，合法字面量读写；分别传不可解析文本、NaN/Inf（若SQL入口无法构造则用转换UT） | 声明类型与原值按转换契约保留；非法输入依冻结规则返回值或拒绝，不接受panic/脏索引。非有限值若允许进入原列，仍需单独验证索引转换安全，不能自动认定可建索引。 |
| VQ-002 | 维度 | 固定维度校验 | 对各类型插入维度不足/超出/空向量、维度 1 和大维度 | 不匹配维度明确报错；边界合法维度稳定。 |
| VQ-003 | 数值范围 | 窄浮点与整数边界 | int8 -128/127、uint8 0/255及越界；f16最大有限值附近、bf16代表舍入值；记录原列转换与entries转换两个阶段 | 合法边界正确；拒绝/饱和/舍入以目标SHA契约分别判定，无符号混淆及静默错误。不能将量化范围行为套到原列类型转换。 |
| VQ-007 | 语法 | QUANTIZATION 格式与大小写 | 使用合法名称、大小写、空值及非法量化值；重复子句属 MySQL parser 行为，按研发确认不作为索引功能用例 | 合法值规范化；非法量化值明确报错。 |

### 2.2 IVFFLAT与量化（8 个执行单元）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| VQ-004 | IVFFLAT基类型 | 全部基类型无量化 | f32/f64/f16/bf16/int8/uint8分别建无量化IVFFLAT，含整型边界小fixture；采SHOW、entries和EXPLAIN，检索 | entries保持基类型；原值不变。用lists=1且覆盖全部候选的可判定fixture核对距离/Top-K；符号、f64及新增窄类型各有证据。 |
| VQ-015 | IVFFLAT量化 | 浮点entries及无量化对照 | 固定f32原表，分别不量化及QUANTIZATION float32/float16/bf16；至少一组窄基类型转换控制，检查SHOW/entries与检索 | 原列类型和值不变；entries按目标类型存储，省略保持基类型；用转换后Oracle解释距离差异，不强求与原始f32完全相同排序。 |
| VQ-016 | IVFFLAT量化 | int8/uint8及度量范围 | f32分别整数量化；对两种量化分别覆盖 L2/L2SQ 合法控制及 IP/cosine/L1 建索引负例；关联018..020 | L2/L2SQ 可用，配置、entries及查询正确；IP/cosine/L1 拒绝并报 20105、无残留。以独立转换/距离Oracle和061质量集判定，不仅要求SQL成功。 |
| VQ-018 | 算子 | L2/L2SQ数值 | 在相同已存表示与候选集上比较L2和L2SQ，含窄浮点和整数 | 数值满足sq=l2²容差；无并列确定fixture排序一致。不同训练/候选的ANN检索不要求任意结果完全相同。 |
| VQ-019 | 算子 | IP/cosine | IVFFLAT对浮点量化及无量化整数基列执行IP/cosine，含零向量控制；int8/uint8 量化负例关联016；GPU合法浮点组合关联024/027 | 数值及排序方向按算子契约与独立Oracle核对；整数基列无量化不等于整数 QUANTIZATION。 |
| VQ-020 | 算子 | L1 | IVFFLAT 全基类型使用 L1 | 支持并正确；GPU 索引不应误接受 L1。 |
| VQ-021 | DML/事务 | 量化条目写入与回滚 | 新增、更新向量和删除；分别commit/rollback后查询，含一个窄基类型与一个f32→窄entries代表组合 | 提交可见、回滚不可见；旧向量/删除行不作为有效结果；原表及索引不互相污染。各算法更新链路与032关联，无需每个类型重复全事务矩阵。 |
| VQ-023 | 训练 | 小样本/训练参数边界 | lists 与训练比例接近行数边界，执行建索引/查询 | 失败时错误清晰且无残留；成功时索引稳定。 |

### 2.3 GPU算法与拒绝边界（10 个执行单元）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| VQ-005 | GPU入口 | 基类型/实验开关 | IVFPQ/CAGRA分别以f32/f16+bigint PK测试开关未开/开启建索引，SHOW/EXPLAIN确认真实路径 | 关闭明确拒绝且无残留；开启在GPU环境创建并实际使用索引。CPU不可用标记跳过，不把EXPLAIN当运行证据。 |
| VQ-024 | IVFPQ | f32/f16及三种量化 | 无量化与float16/int8/uint8分别创建/检索，整数用L2；一个合法m/bits/lists训练组合并测不整除维度负例 | PQ实际构建并查询；配置/原列/返回ID合法；数值和Recall按061，不把PQ近似输出当精确全表Top-K。 |
| VQ-027 | CAGRA | f32/f16及三种量化 | 无量化与float16/int8/uint8分别建图/检索，整数用L2；选一个合法graph_degree及不合法大小关系控制 | CAGRA实际使用；参数/量化传递正确，非法组合拒绝且可清理；质量按061核对。 |
| VQ-030 | 容量 | 窄entries容量与边界 | GPU索引保留0自动容量与一个显式小容量，使用相同量化数据覆盖容量内/边界外写入 | 类型宽度与容量计算一致；按冻结契约报错/进入overflow，不混同两个路径；失败无脏配置，恢复后健康查询正常。不展开全部容量参数矩阵。 |
| VQ-031 | INCLUDE | GPU 索引 include 标量列 | IVFPQ/CAGRA include int/float 列并带过滤查询 | include 条件与向量结果正确；不影响量化路径。 |
| VQ-032 | 重建/加载 | REINDEX保留量化 | 各算法以代表窄entries配置写入后REINDEX并冷加载；SHOW/entries/查询核对 | 重建及加载保持目标类型/参数，查询不沿用旧entries；异步调度和AUTO_UPDATE全矩阵不在本期重复。 |
| VQ-033 | overflow分派 | 主索引+新增尾部候选 | GPU主索引建好后保留尚未归入主索引的可观测新增行（按目标实现同步边界构造），保证最近邻在尾部；gpu_mode=0/1查询 | 证明实际走overflow而非仅超容量；尾部行无丢失、去重/过滤正确，CPU/GPU距离相符，无显存或类型错误。 |
| VQ-036 | GPU类型拒绝 | 非支持基类型与bf16量化 | IVFPQ/CAGRA分别对f64/bf16/int8/uint8基类型、f32+QUANTIZATION bf16建索引 | 按矩阵拒绝并说明限制，无残留；合法f32/f16控制见005，不能只用一个算法证明另一算法。 |
| VQ-037 | 约束 | GPU 索引非 bigint PK | 使用 int、varchar、复合 PK 建 IVFPQ/CAGRA | 明确拒绝；IVFFLAT 对原有 PK 能力无回归。 |
| VQ-039 | GPU算子拒绝 | 整数量化非L2及L1 | IVFPQ/CAGRA的int8/uint8分别尝试IP/cosine；GPU尝试L1；L2SQ整数量化按目标版本精确支持定义单列结果 | 非支持组合明确拒绝；L2-only不得含糊外推为L2SQ可用。IVFFLAT支持矩阵另由016/020判定。 |

### 2.4 基础数组/向量函数（8 个执行单元）

对f16/bf16/int8/uint8逐类执行；每组同时检查NULL传播，二元组检查维度不匹配。窄输入按已存值转f32独立计算；vector_dims为int64，距离/范数标量为float64，数组函数保持类型。删除原053/056独立编号不删除这些断言。

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| VQ-045 | 单目 | `vector_dims` | 各维度/类型调用 | 返回 int64 且等于声明维度。 |
| VQ-046 | 单目 | `summation`、`l1_norm`、`l2_norm` | 负数、零、正数、整数边界输入 | 数值符合 f32 基线与 epsilon。 |
| VQ-047 | 单目 | normalize_l2含零向量 | 四种窄类型输入非零/零向量，对比独立f32计算后按返回类型转换的Oracle，核对返回类型 | f16/bf16 返回原类型，int8/uint8 返回 vecf32；非零结果按对应精度验证，零向量行为冻结，无未解释NaN/panic。 |
| VQ-049 | 单目 | `subvector` | start=1、末尾、负 start、有/无 len | 1 基索引、负索引和长度边界正确，返回同类型。 |
| VQ-050 | 双目 | `inner_product` | 同/异类型可支持组合与向量字面量 | 返回负点积、结果为 float64；数值和距离排序方向正确。 |
| VQ-051 | 双目 | cosine similarity/distance | 正交、相同、反向和零向量 | 数值范围/互补关系正确；零向量契约一致。 |
| VQ-052 | 双目 | `l2_distance`/`l2_distance_sq` | 多类型、负值、整型边界 | 满足平方关系，结果为 float64。 |
| VQ-055 | 编码 | `vec*_from_base64` | 六种类型合法/截断/非法 base64 输入 | 合法输入得到正确类型和维度；非法输入明确失败。 |

### 2.5 兼容、分派与量化质量（4 个执行单元）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| VQ-042 | 兼容 | 旧无量化索引及f32/f64函数 | 复用旧索引元数据fixture加载/查询，记录实际格式与版本；代表旧函数SQL/视图/prepared对照 | 省略量化默认原基类型，已有行为保持。没有旧版本产物/fixture则标缺证据，不用新建无量化索引冒充升级兼容。 |
| VQ-057 | 分派 | 默认值及gpu_mode=0/1 | CPU/GPU build新session记录默认；GPU build相同输入切0/1，运行窄类型函数、IVFFLAT候选距离和033overflow | CPU默认0、GPU默认1；证明受影响分派真实执行，数值在epsilon内，无并列fixture结果相同。不是把独立ANN建图结果强行要求相同。 |
| VQ-059 | SIMD构建 | 代表内核差异 | CPU及GPU各自noarchsimd/archsimd对照，复用四窄类型距离/范数、一个IVFFLAT和GPU代表fixture | 结果类型、数值容差和错误一致；只跑改动内核代表集，不重复所有DDL/重建。无工具链/硬件明确跳过。 |
| VQ-061 | 量化质量 | 有界精度/Recall对照 | 固定seed、规模和查询集，比较三算法无量化/支持的窄量化；冻结候选参数和最低Recall@K（至少K=10），记录距离误差 | 原列精确Top-K作质量真值；满足预先约定阈值且过滤、ID、LIMIT合法。未约定阈值只报告缺口，不自行判通过；不跑ANN-Benchmarks规模化吞吐/长稳。 |

## 3. 基线与结果判定

### 3.1 精确基线

1. 小数据：禁用索引或独立计算距离生成精确结果；仅在覆盖全部候选且无量化改变排序的fixture要求完整Top-K一致。量化后确定性Oracle按相同存储表示计算；大数据ANN不承诺全表精确。
2. 大数据：离线或禁用索引计算精确 Top-K；索引查询比较 Recall@K、过滤后行集合和距离误差。
3. 量化：必须同时验证原列值未改变、entries 量化配置已生效、查询误差在阈值内。

### 3.2 浮点与近似判定

| 项目 | 判定 |
| --- | --- |
| CPU/GPU、SIMD/标量 | 标量函数和精确路径以绝对/相对 epsilon 对比；排序并列时使用 PK 稳定排序。 |
| f16/bf16 | 误差阈值按数据幅度与类型精度设定，并保留原始向量、实际值、基线值。 |
| int8/uint8 量化 | IVFFLAT按其支持的各度量；GPU仅按冻结的L2限制。固定数据集核对Recall，不接受错误过滤、重复ID或错误类型。 |
| 近似索引 | 预先声明数据规模、lists/probe、图/PQ 参数和最低 Recall@K；阈值变动需说明。 |

## 4. 结果记录要求

- 记录版本、commit、CPU/GPU 型号、CUDA/cuVS、构建变体、`gpu_mode`、实验开关及完整索引参数。
- 每个 DDL/负例保留 SQL、错误码/错误文本、SHOW CREATE/SHOW INDEX 与隐藏 entries schema。
- 每个检索用例保留查询向量、精确 Top-K、索引 Top-K、Recall@K、距离误差及 EXPLAIN。
- GPU 或 archsimd 不可用时，显式标记为环境跳过，不能误判为功能失败。

### 4.1 研发 Feature 与实现的差异处理

以下按 Feature 设计保留测试，不因代码已有拒绝逻辑或测试工具忽略元数据而改写为通过。记录基线 SHA、具体类型/SQL、实际错误和证据；若研发确认 Feature 文档需要调整，再同步修改本设计。

| 覆盖点 | Feature 设计预期 | 已观察到的实现行为（`main@7597f979`） | 判定与后续 |
| --- | --- | --- | --- |
| VQ-016/019 | 原 Feature 写整数量化支持全部度量 | L2/L2SQ 成功，IP/cosine/L1 拒绝 | [#31664](https://github.com/matrixorigin/matrixone/issues/31664) 研发确认实现为预期，原 Feature 表格待研发修正；整数基列无量化保留全部度量。 |
| VQ-046/049 | 四种窄类型支持求和、范数及 subvector | 旧版本四种窄类型被拒绝 | [#31666](https://github.com/matrixorigin/matrixone/issues/31666) 确认产品缺陷，修复 PR #31775 已合入；需在新 main 直接回归，不能以合入代替通过。 |
| VQ-047 | 原 Feature 写返回与输入相同的向量类型 | int8/uint8 返回 vecf32；f16/bf16 返回同类型 | [#31665](https://github.com/matrixorigin/matrixone/issues/31665) 研发确认实际行为为预期，按此分别断言。 |
| VQ-050 | 原 Feature 将 inner_product 描述为 dot product | 返回负点积，作为距离用于排序 | [#31667](https://github.com/matrixorigin/matrixone/issues/31667) 研发确认负点积为预期，按数值及排序方向断言。 |
| VQ-042 | 旧索引升级后仍可访问 | 旧数据升级当前 main 时 catalog admission 失败，升级后断言未执行，见 [#31649](https://github.com/matrixorigin/matrixone/issues/31649) | 测试已执行并发现阻塞 Issue；记录为“已执行、被 Issue 阻塞”，不是升级断言通过。修复后针对阻塞断言回归。 |
| BVT 元数据 | 严格比较结果及元数据 | 部分失败与 JDBC 类型或测试器空结果解析有关；关闭元数据比对仅是诊断 | 分别记录严格结果与数值对照；定位工具/产品/预期原因后再修复或提 Issue，不把忽略元数据的结果记为严格通过。 |

### 4.2 执行完成与验收

每条用例单独区分“执行完成”和“验证通过”：完整运行并通过断言才可标记通过；已执行但因已提 Issue 阻塞的用例，在本轮执行进度中记为“已执行、被 Issue 阻塞”，关联 Issue 和未执行断言，可计入本轮执行处理完成，不能计入通过或替代修复回归。未运行的 GPU/其他环境用例仍按实际状态记录；业务验收是否接受未解决 Issue 由验收负责人决定。

## 5. 复用与验收边界

直接复用Issue的Part 1A..1E、Part 2窄类型函数及端到端smoke，分别映射004/015/016、024、027、036/037/039及045..055。先锁定目标源码中实际存在的BVT/UT入口；本文不虚构测试函数名，也不把smoke“有返回行”作为精度/召回通过。三算法的SHOW、entries类型、原列未变与EXPLAIN均随对应owner留证。
