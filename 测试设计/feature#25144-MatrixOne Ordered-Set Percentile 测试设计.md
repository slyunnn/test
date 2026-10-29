# MatrixOne Ordered-Set Percentile 测试设计（必要项评审版）

> 本文依据 `MatrixOne_Ordered-Set-Percentile_测试方案分析.md` 修订，覆盖 Issue [#25144](https://github.com/matrixorigin/matrixone/issues/25144) 的 `PERCENTILE_CONT` 与 `PERCENTILE_DISC`，并纳入后续已合并的 prepared 参数、ordered-set window 与 DISC 可排序类型扩展。按输入行或分组变化的 percentile 参数仍不支持。

> 2026-10-28 必要项评审：60 条收敛为 36 个执行单元；保留原review修正的MVP、类型、独立Oracle及不支持组合，原编号不重排。合入PR #26781的参考提交为`f6889dd2a90077aa4fbe46e66ad91e02b36966cf`；实际执行仍按1.1冻结目标SHA并核对后续变更。本文未执行数据库/UT。

> 2026-10-29 合同修订：结合已合并的 [#28540](https://github.com/matrixorigin/matrixone/pull/28540)、[#29106](https://github.com/matrixorigin/matrixone/pull/29106)、[#29130](https://github.com/matrixorigin/matrixone/pull/29130) 和 [#29035](https://github.com/matrixorigin/matrixone/pull/29035)，调整 R027/R030/R031/R032/R043。R024 按 [#26956](https://github.com/matrixorigin/matrixone/pull/26956) 的 SQL 排序及 [#28854](https://github.com/matrixorigin/matrixone/issues/28854) 修正：ASC/DESC 下 NaN 均在普通数值之后。设计修订不表示相关测试已通过。

## 1. 测试原则

| 项目 | 说明 |
| --- | --- |
| 功能定位 | 两函数是精确 ordered-set 聚合，支持标量/分组与 `OVER` 窗口；`WITHIN GROUP (ORDER BY ...)` 决定聚合内部顺序，与最终输出行排序独立。 |
| 数学语义 | CONT 使用 `r = 1 + p × (N - 1)` 后对相邻秩线性插值；DISC 返回排序后累计分布首次达到 p 的实际输入值。 |
| 参数边界 | p 可为常量或单次 EXECUTE 内固定的 prepared 参数及支持的表达式；必须非 NULL、有限且位于 `[0,1]`。列引用及逐行/逐组变化的 p 须拒绝。 |
| 路径隔离 | 内存 partial merge、独立 spill、远程多 CN merge 分别验证；含 spill runs 的 partial 源状态不得序列化或作为来源 merge，终态目标可接收内存 partial 并在本地 spill。 |
| 类型契约 | DISC 保留输入类型；CONT 对非 DECIMAL 返回 float64。因此大整数 CONT 按 IEEE-754 float64 舍入判定，不要求任意大整数十进制精确。 |
| 独立 Oracle | 测试侧排序、rank、插值与转换不得复用生产排序器、rank 计算器或类型转换实现。 |
| 分层门禁 | SQL/BVT验证语义，UT验证状态/编码/spill生命周期，多CN验证真实远程merge与取消。只保留直接改动风险，不追加通用性能/长稳。 |

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
| 功能入口 | `PERCENTILE_CONT(p) WITHIN GROUP (ORDER BY expr [ASC\|DESC])`；`PERCENTILE_DISC(p) WITHIN GROUP (ORDER BY expr [ASC\|DESC])` |
| 实现参考 | PR [#26781](https://github.com/matrixorigin/matrixone/pull/26781)、[#28540](https://github.com/matrixorigin/matrixone/pull/28540)、[#29106](https://github.com/matrixorigin/matrixone/pull/29106)、[#29130](https://github.com/matrixorigin/matrixone/pull/29130)、[#29035](https://github.com/matrixorigin/matrixone/pull/29035)；实际验收以 1.1 冻结的 commit 为准。 |
| 支持类型 | CONT 支持数值类型及支持范围内的 DECIMAL64/128；DISC 另支持 VARCHAR、DATE、DECIMAL256 等可排序标量，仍拒绝 JSON、array/vector。 |
| NULL 输入值 | 不参与排序、有效行数和计算；空集合或全 NULL 返回 NULL。 |
| NULL percentile 参数 | 拒绝 NULL 常量和执行期绑定的 NULL 参数；此点与 PostgreSQL 的 NULL 返回行为不同，作为兼容性差异记录。 |
| NaN/Infinity | 本期作为合法浮点输入冻结：ASC 和 DESC 下 NaN 均位于普通数值之后，与 SQL `ORDER BY` 一致；选中/插值端点为 NaN 时 CONT 返回 NaN；`-Inf < 有限值 < +Inf`。 |
| 分布式限制 | 仅内存 partial 源支持序列化和作为来源 merge；终态目标可本地 spill 并接收内存 partial，不因此推断跨 CN 传输 spilled partial 已支持。 |

### 1.3 必要覆盖与精简说明

| 层级 | 执行单元 | 必须覆盖 |
| --- | ---: | --- |
| 数学与SQL语义 | 10 | 两函数、方向、rank边界、分组、NULL及有界属性Oracle |
| 数值类型 | 8 | 原生整数/unsigned、大整数float64、特殊浮点、DECIMAL |
| MVP拒绝边界 | 4 | 非法p/NULL p、不支持SQL与类型 |
| 内存状态与分布式 | 6 | 分组远程merge、批次/切分、编码、错误与协议门禁 |
| spill路径 | 5 | 真实spill、compaction、特殊值、不支持组合、I/O失败 |
| 生命周期与取消 | 3 | 释放/重建、单CN与多CN取消 |
| **合计** | **36** | 合并fixture共享数据，不省略独立路径断言 |

001吸收002/003/009/010；005吸收006并随用例采008计划；020吸收021；028吸收029；034吸收033/035；043分别检查 spilled 来源拒绝与终态目标接收的正确性；046吸收047/048的释放断言。删除独立通用事务052、重复执行长稳051及053..060规模化性能/approx对照。保留有界随机Oracle，不以海量压测代替正确性。单CN强制spill与多CN内存merge必须分开，不把跨 CN 传输 spilled partial 当作已支持。

### 1.4 现有覆盖映射

| 已有测试 | 已覆盖内容 | 本设计补充重点 |
| --- | --- | --- |
| `func_aggr_ordered_set.test` | 基础 SQL、分组、ASC/DESC、NULL、DECIMAL、WITHIN 标识符 | 冻结语义、边界、Oracle、多 CN 与负例。 |
| `ordered_percentile_test.go` | 多数数值类型、基础数学、NaN、内存/spill 一致性 | 大整数 CONT 返回类型、路径隔离、资源与错误契约。 |
| `remote_expr_test.go` | MORPC v16/v17 远程执行门禁 | release 版本矩阵和多 CN E2E。 |

## 2. 核心功能门禁（23 个执行单元）

### 2.1 语法、计划与SQL组合（4 个执行单元）

| 编号 | 场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- |
| PCT-R001 | 两函数基本语法/默认方向 | 对1,2,100,101分别CONT/DISC p=.5；比较默认与显式ASC；随例采EXPLAIN | CONT=51、DISC=2且DISC来自输入；parse/bind/execute正确；默认ASC。不另重复同一偶数中位数fixture。 |
| PCT-R004 | DESC | CONT/DISC 对乱序、重复、NULL 数据执行 DESC | 等于独立 DESC Oracle。 |
| PCT-R005 | 分组及输出排序 | 多group经WHERE过滤、aggregate HAVING；最终按percentile别名ORDER BY | 每group在过滤后独立计算；最终排序仅改输出顺序，不改WITHIN GROUP的内部方向/数值；计划绑定不panic。 |
| PCT-R007 | SQL输入与共享语法回归 | 一个JOIN/CTE展开输入和UNION ALL重复输入作对照；复用已存在的WITHIN标识符及GROUP_CONCAT两种排序语法小fixture | 聚合输入的行重数不变；共享WITHIN GROUP parser/binder新增分支不破坏既有标识符/GROUP_CONCAT，不展开全部SQL组合矩阵。 |

### 2.2 数学语义与NULL（7 个执行单元）

| 编号 | 场景 | 数据与步骤 | 预期结果 |
| --- | --- | --- | --- |
| PCT-R011 | 秩公式与累计分布边界 | 1..5，CONT/DISC p=.25/.75/.95，并选N*p恰为整数及其两侧有限p | 独立rank/ceil Oracle一致，CONT .95=4.8、DISC .95=5；DISC边界不偏一。浮点p按实际常量位模式计算，不用文本近似掩盖边界。 |
| PCT-R012 | p 边界 | 乱序数据，p=0/1，ASC 和 DESC | 返回各排序方向的首/尾值。 |
| PCT-R013 | 单值与重复 | 单有效值；`1,1,1,100,100` 多个 p | 单值始终返回本身；重复参与 rank，不按 DISTINCT。 |
| PCT-R015 | NULL 输入行 | `1,NULL,2,NULL,100,101` p=.5 | 等同 `1,2,100,101`；结果 CONT=51、DISC=2。 |
| PCT-R016 | 空集/全 NULL | `WHERE 1=0`、单 group 全 NULL、多 group 混合 | 无有效值 group 返回 NULL，其他 group 不受影响。 |
| PCT-R017 | NULL p | `percentile_*(NULL)` | 明确报“non-null constant”类错误；记录为与 PostgreSQL 的预期差异。 |
| PCT-R018 | 属性测试 | 固定 seed 生成 N、重复率、NULL 率、group 与 p | 每轮与独立 Oracle 比较；失败输出 seed、输入、方向和 p。 |

### 2.3 类型、精度与特殊浮点（8 个执行单元）

| 编号 | 场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- |
| PCT-R019 | 有符号整数 | INT8/16/32/64 边界、负值、重复 | 原生整数排序正确；DISC 保留输入类型。 |
| PCT-R020 | 无符号与超2^53 DISC | UINT8/16/32/64含0/上界邻值；独立2^53、2^53+1，p=0/1及中间秩 | unsigned原生顺序正确，DISC精确返回输入值及同类型；不得经float64导致相邻大整数排序塌缩。 |
| PCT-R022 | 超 2^53 的 CONT | 相邻超大整数及非端点 p | 用 `float64` 标准舍入作为预期；检查返回 float64，不要求原十进制精确。 |
| PCT-R023 | FLOAT32/FLOAT64 有限值 | 正负、小数、重复，多 p | 使用 IEEE bit pattern 或声明 ULP 容差比较。 |
| PCT-R024 | NaN 排序与结果 | ASC/DESC 下含 NaN、有限值及 ±Inf；使 NaN 落在选中秩/插值端点，分别验证内存及 spill | ASC/DESC 下 NaN 均在普通值后，与 SQL `ORDER BY` 一致；选中/参与插值时 CONT 返回 NaN；内存与 spill 一致。 |
| PCT-R025 | Infinity | 含 `-Inf`、有限值、`+Inf` 的 ASC/DESC | 排序为 `-Inf < 有限值 < +Inf`；端点和 DISC 返回契约一致。 |
| PCT-R026 | DECIMAL64 | 固定 scale 的插值/离散 p | Oracle 比较 unscaled value、scale、结果类型；不经 float64。 |
| PCT-R027 | DECIMAL128/限制 | 合法精度与最大精度，分别验证 CONT 与 DISC 的 DECIMAL256 | 支持范围正确；CONT 的 DECIMAL256/不支持宽度稳定报错；DISC 的 DECIMAL256 精确返回输入值及类型。 |

### 2.4 分布式与负例（4 个执行单元）

| 编号 | 场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- |
| PCT-R028 | 单/多CN内存merge | 同一快照分别无group/有group，含倾斜、NULL、重复、大整数，CONT/DISC及ASC/DESC；证明实际远程partial merge | 逐group与单CN/独立Oracle一致，无串组；必须保持内存状态，不把spill失败解释成已支持远程spill。一个CN数控制与一个真实多CN拓扑足够。 |
| PCT-R030 | percentile形态 | 同一 prepared 语句使用多个有效参数、算术/CAST；分别测试 NULL、越界、NaN/Inf、列及依赖行表达式，并验证有效—无效—有效复用 | 执行期固定参数及支持的表达式按各次绑定值正确计算且不复用旧值；非法参数和行依赖表达式稳定拒绝，后续有效执行不受污染。 |
| PCT-R031 | SQL 形态与窗口 | 缺 WITHIN GROUP、多排序键；CONT/DISC 的 `OVER()`、分区及 ASC/DESC，与独立逐分区 Oracle 比较 | 缺 WITHIN 和多排序键分别稳定报错；支持的窗口形态返回正确分区结果、行数和类型，无 panic。 |
| PCT-R032 | 排序类型边界 | DISC 的 VARCHAR、DATE、DECIMAL256 和 JSON/array；CONT 的非数值与 DECIMAL256 | DISC 对支持类型精确返回输入值并保留类型，JSON/array 稳定拒绝；CONT 对不支持类型稳定拒绝，不隐式转换、无 panic。 |

## 3. 实现可靠性（13 个执行单元）

### 3.1 内存partial merge与远程门禁（5 个执行单元）

| 编号 | 场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- |
| PCT-R034 | Fill/merge不变性 | 同一输入单/多batch、两/四路交错切分，在重叠/独有group上局部聚合再merge，含空partial | 结果与全量一次聚合及Oracle一致，group映射无串扰；各批次/切分控制有直接结果，不要求任意并行度压力矩阵。 |
| PCT-R036 | 内存状态 round-trip | 无 spill 状态序列化/反序列化后 merge/flush | 等于未序列化状态和 Oracle。 |
| PCT-R037 | merge 配置不一致 | p、方向、类型不一致的 state merge | 拒绝 merge；状态仍可 Free。 |
| PCT-R038 | 损坏状态与配置版本 | 对实际序列化 state 构造截断、负长度及超出剩余数据长度；对带版本标记的 percentile 配置单独构造未知版本 | 不合法输入返回错误，无越界/panic/泄漏，解码失败后的原 state 仍可使用和 Free。当前 state 编码无独立版本字段，不能构造真正的 state 未知版本；若后续引入版本字段，再增加 state 未知版本拒绝验证。配置版本检查不得冒充 state 版本检查。 |
| PCT-R039 | MORPC门禁 | 复用remote_expr_test.go的v16/v17发送/接收能力边界，分别构造可拒绝及可执行路径 | 不支持版本明确拒绝而非丢失ordered-set配置；支持版本结果与本地一致。记录精确本地/远端版本，不由门禁UT推导混合版本部署已支持。 |

### 3.2 Spill、compaction与限制组合（5 个执行单元）

| 编号 | 场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- |
| PCT-R040 | 独立 spill | 使用实际生效阈值（不写“设置为 1”），以足够输入字节触发 spill | 记录阈值、行数、估计字节、spill runs；结果等于内存 Oracle。 |
| PCT-R041 | 多 run/compaction | 构造足够数据产生多个 run 和可观测 compaction | 记录 run 数、compaction、spill rows、peak memory；无丢/重行。 |
| PCT-R042 | spill 特殊值 | NULL、重复、NaN、超 2^53、DECIMAL 分别执行 | 与内存路径及类型契约一致。 |
| PCT-R043 | spill 源拒绝与终态目标 | 独立构造含真实 spill runs 的来源 state，验证序列化和作为 merge 来源；另构造已有本地 runs 的终态目标，多次接收内存 partial，与无 spill 基线和独立 Oracle 比较，检查 runs 与 Free | spilled 来源的序列化/merge 稳定拒绝、runs 不丢失；终态目标正确接收内存 partial，结果与基线/Oracle 一致且释放资源。两条路径分别留证；不能以 merge 返回成功替代结果和清理验证，也不能推断远程 spilled partial 支持。 |
| PCT-R045 | spill I/O错误 | 在运行期注入写/读或目录错误，选择能覆盖创建与已创建run的失败点 | 错误明确传播；部分初始化状态可Free，mpool/临时文件回收，后续健康聚合正常；不做无关系统故障全矩阵。 |

### 3.3 生命周期、取消与资源（3 个执行单元）

| 编号 | 场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- |
| PCT-R046 | 成功/失败Free及新建 | 内存/spill成功后，以及merge/解码/spill失败的部分状态调用真实Free；再新建executor执行不同类型/p/方向 | mpool回基线，临时文件删除；新实例无旧状态、结果正确。不假设不存在的Reset API，不用2小时循环证明释放。 |
| PCT-R049 | 单CN取消 | 有证据的运行/spill查询触发deadline或cancel，释放executor后再健康查询 | 取消及时终止、错误不伪成功，内存/文件回收，后续结果不污染；记录取消点和完成时间，不以极短未进入spill的查询冒充。 |
| PCT-R050 | 多CN取消 | 内存远程merge的一个fragment取消或超时，记录远端退出及清理 | 无卡死、悬挂goroutine或残留状态，后续健康查询正确；不强行构造本期不支持的远程spill组合。 |

## 4. Oracle与结果判定

| 数据类型/场景 | Oracle 与比较规则 |
| --- | --- |
| DISC 整数（含 uint64） | 用原生整数/大整数排序，比较原始值及 SQL 返回类型，要求精确相等。 |
| CONT 非 DECIMAL | 使用独立 rank/插值实现，最终显式转换为 float64；比较 IEEE-754 bit pattern 或预先声明 ULP。 |
| FLOAT | 不以 `big.Rat` 强求文本精确相等；采用 bit pattern/ULP，NaN 单独用 `isNaN` 判断。 |
| DECIMAL | 比较 unscaled value、scale 与最终 SQL 类型；Oracle 不调用生产 decimal/rank/sorter。 |
| NULL/空集合 | 判断 SQL NULL 状态，不将 NULL 作为普通排序值。 |
| 单/多 CN、memory/spill | 使用同一冻结数据和 Oracle；允许执行指标不同，不允许逻辑结果不同。 |

## 5. 结果记录要求

- 所有门禁结果必须包含第 1.1 节全部冻结项、SQL、数据 seed、预期/实际、返回类型、CN 数与执行计划。
- spill 结果额外记录实际阈值、输入行数/估计字节、spill run 数、compaction、spill rows、peak memory、临时文件清理状态。
- 浮点/大整数失败必须给出输入排序序列、p、rank、插值端点、期望 float64 位模式或 ULP、实际值和类型。
- PCT-R043 分别记录 spilled 来源序列化/merge 的拒绝，以及终态目标接收内存 partial 的结果、runs 和清理；未执行的入口不得算通过。
- 取消/释放用例保留mpool、临时文件及远端退出证据；不能以“未panic”替代数值与资源正确性。
