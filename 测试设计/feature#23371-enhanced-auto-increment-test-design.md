# MatrixOne Enhanced AUTO_INCREMENT 测试设计

> 本设计覆盖 Feature [#23371](https://github.com/matrixorigin/matrixone/issues/23371)：增强 AUTO_INCREMENT，支持修改起始值、`AUTO_ID_CACHE` 表选项，以及 `auto_increment_increment` / `auto_increment_offset` 会话变量。核心目标是在分布式环境中保证 ID 唯一、序列语义明确、缓存可控且与既有表兼容。

> 2026-10-09 研发 review 修订：以已合入 [PR #28528](https://github.com/matrixorigin/matrixone/pull/28528)、合入提交 `2b56f0d5e11b7af0b29cf91bd1d0f03430da3f75` 为契约基线。本 PR 主要新增表级 AUTO_ID_CACHE；ALTER 起点和 session 序列复用上游能力并验证组合行为。本文为测试设计与已有测试映射，不代表本轮已执行 UT/BVT 或性能测试。

## 1. 测试原则

| 项目 | 说明 |
| --- | --- |
| 唯一性优先 | 无论单/多 CN、缓存大小、并发、事务或显式插入，自动生成 ID 都不得重复；允许的间隙必须可由缓存或回滚语义解释。 |
| DDL 原子性 | ALTER 起始值、schema 演进、建表/恢复失败不得留下部分元数据；本期不支持 ALTER AUTO_ID_CACHE。 |
| 会话隔离 | increment/offset 必须严格按 session 生效；不同 session 可并发使用不同设置，不能相互泄漏。 |
| 兼容性 | 不显式使用新语法的既有 AUTO_INCREMENT 表保持原有默认行为；MySQL 兼容语义的差异须冻结并记录。 |
| 分布式验证 | ID 分配依赖分布式 counter 和本地 cache，所有核心用例须覆盖单 CN 与多 CN，且以全局唯一性和最终序列规则判定。 |
| 结果与状态双校验 | 除 INSERT 结果外，检查 SHOW CREATE、系统元数据、重连/重启后的下一 ID、执行计划/日志中的 cache 分配证据（可获得时）。 |

### 1.1 基本信息

| 项目 | 内容 |
| --- | --- |
| MatrixOne 基线 | 契约基线为 PR #28528 合入提交；执行时另记实际 commit、镜像和全部角色版本，检查后续变更。 |
| DDL 入口 | `ALTER TABLE t AUTO_INCREMENT = n`；`CREATE TABLE ... AUTO_ID_CACHE [=] n`。不支持 ALTER AUTO_ID_CACHE；CACHE 只能属于表。 |
| 会话变量 | `auto_increment_increment`、`auto_increment_offset`。 |
| 默认行为 | CACHE 省略/0 继承 CN 的 CountPerAllocate/LowCapacity；生产开关默认 false，仓库 feature 测试配置显式开启。 |
| 参数契约 | CACHE 合法范围 0..1000000；1 为真实需求分配，2..1000000 为基础原始数值跨度，可因实际需求扩大。非默认 session 即使 CACHE>1 也不后台预取。 |
| 发布边界 | 合入版本 CACHE 门槛 V81；本地/发送端/接收端门禁仅证明拒绝能力，不证明支持混合版本。全角色升级后才开启。 |
| 关键对象 | AUTO_INCREMENT 列、表级 counter 元数据、分布式 ID cache、session 变量。 |
| 主要风险 | 自动发号冲突/写入丢失、显式值推进错误、水位与候选值混淆、cache策略失效、观察陈旧、DDL/DML原子性、恢复后序列漂移。 |

### 1.2 执行矩阵

| 测试类型 | 单 CN | 多 CN | 通过标准 |
| --- | --- | --- | --- |
| DDL/元数据 | 执行 | 执行 | 定义、SHOW、错误与计数器状态一致。 |
| 默认生成/显式插入 | 执行 | 执行 | 唯一、步长/offset 正确，无不可解释冲突。 |
| Session 变量 | 执行 | 执行 | 每 session 独立且序列满足公式。 |
| Cache/并发 | 执行 | 必须执行 | 全局无重复；cache 间隙、性能与失效可解释。 |
| 事务/恢复/升级 | 执行 | 执行 | 状态持久、恢复正确，旧表兼容。 |
| 性能/长稳 | 基线 | 必须执行 | 吞吐稳定，无计数器热点、泄漏或 CN 异常。 |
| 开关/重启 | 独占集群执行 | 全 CN 配置一致 | 默认关闭、开启→关闭→开启；保留数据/策略，拒绝路径无残表。 |
| 观察/事务可见性 | UT + SQL | 必须执行 | 不耗号，读到事务自身状态和跨 CN 已提交新状态；取消/屏障错误不伪成功。 |
| 协议门禁 | 本地/发送端/接收端 UT | 同版本功能集成 | V80 拒绝、V81 接受（开关开启）；禁用仍拒绝非零策略。不得据此声明混合版本可运行。 |

### 1.3 覆盖范围

| 一级模块 | 用例数 | 主要覆盖内容 |
| --- | ---: | --- |
| DDL 与元数据 | 12 | ALTER 起始值、建表 option、SHOW、非法参数、schema 变更 |
| 默认 ID 与显式插入 | 11 | 起始/边界、批量、NULL/0、手工值、类型 |
| Session increment/offset | 10 | 序列公式、隔离、切换、非法变量、复制式分配 |
| Cache、并发与分布式 | 11 | cache 大小、分配、CN 重启、并发、热点 |
| 事务、恢复与兼容 | 9 | commit/rollback、backup/PITR、升级、rename/truncate |
| 异常、性能与可观测性 | 7 | DDL/DML 故障、性能、长稳、诊断 |
| 发布开关与协议边界 | 6 | 默认关闭、重启、维护例外、V81、全量升级 |
| 观察、生命周期与语法补充 | 7 | 不耗号、事务/跨 CN、临时表、DUMP/LOAD、语法拒绝 |
| 元数据性能与策略传递 | 2 | 共享观察屏障、物理表 ID 绑定与回退 |
| **合计** | **75** | 保留原 AINC-001..060 编号，新增 061..075；一条用例的参数组合分别执行。 |

### 1.4 参考资料

| 资料 | 地址 |
| --- | --- |
| Feature | <https://github.com/matrixorigin/matrixone/issues/23371> |
| 实现与已有测试 | [PR #28528](https://github.com/matrixorigin/matrixone/pull/28528/files)；测试映射见 §5。 |
| SQL/allocator 契约 | [合入版本设计](https://github.com/matrixorigin/matrixone/blob/2b56f0d5e11b7af0b29cf91bd1d0f03430da3f75/docs/design/CLAUDE_AUTO_INCREMENT_23371.md) |
| 发布边界 | [合入版本 rollout 设计](https://github.com/matrixorigin/matrixone/blob/2b56f0d5e11b7af0b29cf91bd1d0f03430da3f75/docs/design/CLAUDE_AUTO_ID_CACHE_ROLLOUT_23371.md) |
| 兼容性参考 | <https://docs.pingcap.com/tidb/stable/auto-increment> |
| 现有 AUTO_INCREMENT 回归 | MatrixOne 当前 DDL/DML、事务、备份恢复和多 CN 测试。 |

### 1.5 执行前提与判定边界

非零 CACHE 正向用例需在全角色升级完成、全部 CN 显式开启以下配置后执行；禁用用例单独使用关闭配置。仓库测试默认启用不能替代生产默认关闭验证。

```toml
[cn.auto-increment]
enable-auto-id-cache = true
```

- 默认 session 指 `increment=1, offset=1`；精确 ID 示例使用独立新表、无其他 writer，且显式指定 CACHE=1（另有说明除外）。
- 区分表中现存最大值 M、allocator 已预留高水位 H、下一自动候选 N。H 不等于 M 或 N；显式 ALTER 可重置预留水位，测试不得仅因 H 下降就判错。默认 session、M=200、ALTER 起点为100时，下一值应为201。
- CACHE=1 不承诺全局单调或无间隙；失败、回滚、显式高值推进可产生 gap。唯一性按单表/单自增列检查，不要求不同表或不同自增列的 ID 互不相同。
- 本期不独立修复 REPLACE/IGNORE 的既有问题；其拒绝行显式高值提前推进等行为不能据此追加“失败必须退还共享号段”的要求。
- 全角色升级、停止旧 CN/TN、排空工作负载后才能启用；不支持混合版本运行、旧节点重入或带非零策略表原地降级。协议拒绝 UT 与可支持的部署流程分开记录。

## 2. 测试用例

### 2.1 DDL 与元数据（12 条）

通用表：`t(id BIGINT AUTO_INCREMENT PRIMARY KEY, v VARCHAR(50))`；需同时覆盖有/无历史数据、单列/复合约束兼容表和不同整数宽度的 AUTO_INCREMENT 列。

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| AINC-001 | CREATE | 默认 AUTO_INCREMENT 回归 | 不带新选项建表并连续插入 | 默认起始、连续性和 SHOW 与变更前一致。 |
| AINC-002 | CREATE | CACHE 合法值 | 省略、0、1、2、8、1000000 分别建表，冷 CN 加载后 SHOW/写入 | 省略/0 继承 CN 默认，SHOW 省略零选项；正数持久且 SHOW 如实呈现，冷加载仍遵循表策略。 |
| AINC-003 | CREATE | 起始值 + cache | 默认 session、CACHE=1，分别以 AUTO_INCREMENT=0、10 建新表并插入；inc=3/off=2 时以起点10建表 | 起点0合法，默认 session 首值1；起点10首值10；inc=3/off=2 生成11、14、17。session 参数不持久化为表策略。 |
| AINC-004 | ALTER | 空表提高起始值 | 默认 session、CACHE=1，`ALTER TABLE t AUTO_INCREMENT = 1000` 后插入 | 首值1000，后续1001；策略保留。 |
| AINC-005 | ALTER | 有数据提高起始值 | 已生成 ID 后改为更大值，插入多行 | 后续 ID 不低于目标值且无重复。 |
| AINC-006 | ALTER | 低于现存最大值/预留水位 | 默认 session，构造现存最大ID=200及高于200的预留水位；ALTER起点100后插入 | ALTER成功，下一值201；分别记录M/H/N，允许显式ALTER重置预留水位，不以旧H作为下一值下界。 |
| AINC-007 | ALTER | 等于当前/重复 ALTER | 多次设置相同或临界值 | 幂等/错误语义明确，counter 不意外跳变。 |
| AINC-008 | 元数据 | 定义与下一值分开检查 | SHOW CREATE检查持久策略；SHOW TABLE STATUS和information_schema.tables检查下一值；执行067..070 | 正数策略保真、0省略；CACHE=1空表重复观察不预留号段；不能只验证插入后SHOW。 |
| AINC-009 | 约束 | 无自增列/多个自增列 | 普通/临时表无自增列时分别指定CACHE=0/1；建立含两个可见自增列的表并混合显式/自动写入 | 无自增列CACHE=0成功、非零拒绝且无残表；多个自增列合法，各列独立分配且遵循同一表级策略。 |
| AINC-010 | 参数 | CACHE非法值与起点边界分离 | CACHE=-1、1.5、1000001、重复选项及uint64溢出；另测起点0/合法上界/超列类型上界 | 非法CACHE拒绝、不截断、不留残表；CACHE=0和建表起点0是正向控制。起点测试分别记录DDL接受与发号溢出，不把两阶段错误混同。 |
| AINC-011 | DDL 原子性 | ALTER 与并发 DML | 一边 ALTER、一边 INSERT/SELECT | DDL/DML 遵循锁与原子性；无重复/半更新状态。 |
| AINC-012 | Schema 演进 | 删除/保留/转移自增属性 | CACHE=0/1/8，DROP最后一个自增列、MODIFY去掉最后一个自增属性；对照保留、转移属性及多自增列仅删除一列 | 最终无可见自增列时成功且策略归0，SHOW无CACHE，普通数据保留；最终仍有自增列则保留策略、后续各列安全发号。 |

### 2.2 默认 ID 与显式插入（11 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| AINC-013 | 基础 | 单行自动生成 | 多次 `INSERT(v)` | ID 唯一，默认递增规则正确。 |
| AINC-014 | 批量 | 多行 VALUES/INSERT SELECT | 单语句插入多行、空结果 INSERT SELECT | 每个实际插入行获得唯一 ID；顺序/间隙符合分配契约。 |
| AINC-015 | 默认值 | NULL/DEFAULT/省略 id | 用三种写法插入 | 所有合法写法触发自动生成，语义一致。 |
| AINC-016 | 零值 | 显式 0 | 插入 `id=0`（受 SQL mode 影响时覆盖配置） | 行为符合 MatrixOne 冻结规则，不与自动 ID 混淆。 |
| AINC-017 | 手工值 | 低于当前 ID | 插入未占用低值后自动插入 | 手工行成功/失败按 PK 规则；counter 不错误回退。 |
| AINC-018 | 手工值 | 显式高值推进 | 默认session、CACHE=1新表先插入显式100，再自动插入；另测负数控制 | 后续自动值101；负数不推进水位；自动生成不与已存在显式ID冲突。 |
| AINC-019 | 冲突 | 已存在显式 ID | 插入重复主键、随后自动插入；记录错误、数据和分配事件 | 重复键明确报错，失败语句数据原子性正确；后续自动发号安全。允许已申请共享号段产生gap，不要求失败退还或回滚counter。 |
| AINC-020 | 类型 | TINYINT/INT/BIGINT 有符号/无符号 | 各类型接近上限插入 | 边界生成与溢出错误正确；无回绕。 |
| AINC-021 | 删除 | DELETE/TRUNCATE 后生成 | 默认session、CACHE=1/8，删除最高值、DELETE全表、TRUNCATE后重插；检查SHOW | DELETE不重置发号；TRUNCATE后首值1且保留CACHE。TRUNCATE按隐式提交验证，不要求外层ROLLBACK撤销它。 |
| AINC-022 | 多表 | 多 auto 表独立性 | 并发写入多个不同配置表 | counter/cache 完全隔离。 |
| AINC-023 | 查询 | LAST_INSERT_ID/返回值（支持时） | 单行/批量/失败插入后读取 | 返回本 session 正确生成 ID，不受其他 session 干扰。 |

### 2.3 Session increment 与 offset（10 条）

令 `inc = auto_increment_increment`、`off = auto_increment_offset`；变量各自合法时，`effective_off = off > inc ? 1 : off`。仅自动生成行应满足 `id ≡ effective_off (mod inc)`，首项还受起点、现存数据及可用号段约束。显式ID不参与同余检查；并发/续租允许跳项，不能要求所有相邻结果差固定为inc。

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| AINC-024 | 默认 | 默认变量值 | 新 session 查询变量并插入 | 默认值与兼容定义一致。 |
| AINC-025 | increment | inc=2/3/大值 | 设置 inc，连续/批量插入 | 序列步长与同余关系正确，ID 唯一。 |
| AINC-026 | offset | offset=1..inc | 固定 inc，切换合法 offset 插入 | 每个 session 返回满足其 offset 的序列。 |
| AINC-027 | 组合 | 不同 session 不同 inc/off | 两 session 交替/并发插入同表 | 所有 ID 全局唯一；各 session 的生成值符合各自设置。 |
| AINC-028 | 切换 | session 内动态 SET | 先默认插入，再 SET inc/off 并继续插入 | 切换仅影响后续分配；无重复或错误回退。 |
| AINC-029 | 隔离 | 连接池/重连 | 设置变量后重连、新开 session、复用连接 | session 生命周期和 reset 行为符合产品契约，无变量泄漏。 |
| AINC-030 | 参数 | 非法范围与offset归一化 | 分别测试变量自身非法范围/类型；独立新表设置inc=3/off=5后插入三个NULL | 各自合法范围内off>inc可设置，生成时有效offset=1，得到1、4、7；真正非法输入按变量定义报错，不能把off>inc归入拒绝组。 |
| AINC-031 | DDL 交互 | ALTER起始值 + inc/off | 独立空表、CACHE=1，ALTER起点10后，分别在inc=3/off=2和inc=3/off=5下写入 | 首值分别11和10；候选满足有效offset与ALTER后的数据/分配边界，不使用ALTER前预留高水位作为下界。 |
| AINC-032 | 手工值交互 | 显式高 ID + inc/off | 手工插入后由多个 session 自动插入 | 不违反各 session 序列或全局唯一约束。 |
| AINC-033 | 多 CN | session 变量跨 CN | 同 session/不同 session 经不同 CN 写入 | 会话设置正确传递；不因路由改变失效。 |

### 2.4 Cache、并发与分布式（11 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| AINC-034 | cache | CACHE=1真实需求分配 | UT记录分配调用/原始跨度，覆盖构造、空输入、全显式、估算、低水位、并发等待者；SQL混合(NULL,100,NULL) | 前述无自动需求路径不申请自动号段；显式高值必要水位推进另计。混合结果1、100、101；不因估算/低水位/等待者扩大预留；精确断言见§3.3。 |
| AINC-035 | cache | CACHE>1原始跨度 | CACHE=2/8，大于基础跨度的实际批量需求；默认及inc=3/off=2分别检查分配记录 | CACHE为基础原始数值跨度，不是固定生成行数；需求大可扩大；覆盖CN默认跨度；非默认session无后台预取。性能单列055，不替代分配断言。 |
| AINC-036 | cache | 不同表 cache 隔离 | 同时写入 cache 不同的多表 | 每表配置独立，不能借用/污染其他计数器。 |
| AINC-037 | cache | cache 耗尽/续租 | 单 CN 插入超过多个 cache 区间 | 新 cache 无重叠、无重复；边界无性能异常。 |
| AINC-038 | 多 CN | 多 CN 并发 INSERT | 无预期冲突场景，记录每行writer_id/request_id、计划行数、提交结果和ID；核对§3.1 | 全部计划写入成功且实际行数=预期行数；自动分配duplicate-key为0；逐请求无丢失/重复，唯一性等式仅为附加检查。 |
| AINC-039 | 多 CN | cache 分片 | 各 CN 获取 cache 后交替插入 | 分配区间不重叠；允许全局物理顺序交错。 |
| AINC-040 | 故障 | CN 重启/切换 | cache 已预留但未用尽时重启 CN，再写入 | 已发 ID 不复用；未用 cache 的 gap 符合持久化策略。 |
| AINC-041 | 故障 | TN/元数据短暂不可用 | 在分配/续租时故障注入后恢复 | 明确重试/错误；恢复后无重复或 counter 损坏。 |
| AINC-042 | 并发 | INSERT 与显式高值竞争 | 高并发自动插入期间插入高显式 ID | 无 duplicate key 风暴、死锁或不可恢复的 counter 跳变。 |
| AINC-043 | 热点 | 单表热点与多表分散 | 比较同总 QPS 下单表/多表写入 | 记录吞吐/延迟；热点不造成 CN 异常或无限重试。 |
| AINC-044 | 可观测性 | 观察副作用与诊断 | 执行067..070及074，关联物理表ID、列、M/H/N、CN、事务、分配事件 | 观察不发号，私有/已提交事务路径正确；跨CN值及时可见，取消/屏障失败退出；保留可定位证据。 |

### 2.5 事务、恢复与兼容（9 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| AINC-045 | 事务 | commit | 事务内插入多行后 COMMIT | 数据和已分配 ID 可见，next-id 正确前进。 |
| AINC-046 | 事务 | rollback | 插入后ROLLBACK，再插入并记录号段 | 回滚行不可见；共享号段不随业务事务回滚，允许gap；后续成功且无冲突，不要求ID连续。 |
| AINC-047 | 事务 | savepoint/失败语句 | 部分失败、savepoint回滚后继续写 | 数据遵循回滚边界，允许已预留号段间隙；无自动发号冲突/卡死，不要求counter与数据同步回退。 |
| AINC-048 | 备份恢复 | 逻辑导出/恢复 | 默认session、CACHE=1表写入1，导出DDL和数据后恢复；另执行072的DUMP/LOAD | 正数CACHE被导出并恢复，数据1保留，下一自动值2；0策略保持默认。不以逻辑导出替代DUMP TABLE/LOAD TABLE。 |
| AINC-049 | Snapshot/PITR | 多次 ALTER/DML 后恢复到不同时间点 | 每恢复点后插入并检查 ID | counter 与恢复点数据一致；不与现存行冲突。 |
| AINC-050 | 升级 | 旧表兼容与全量升级 | 升级前创建无新option表，按066停旧进程并升级全部角色；关闭/开启时验证旧表 | 旧表等价CACHE=0，默认行为兼容；记录全部角色版本与配置。带非零策略表不做原地降级成功验收。 |
| AINC-051 | 复制/订阅 | CDC/订阅（支持时） | 写入 auto 表、恢复/消费目标端 | 主从/订阅语义不产生 ID 重写或冲突。 |
| AINC-052 | 复制/重命名 | RENAME、LIKE、CLONE、COPY独立路径 | 默认session、CACHE=1、最大ID=200；独立副本执行RENAME、CREATE TABLE dst LIKE src、CREATE TABLE dst CLONE src、ALTER TABLE src ADD COLUMN extra INT, ALGORITHM=COPY | 各路径保留CACHE；RENAME/COPY保留数据、下一值201；LIKE为空表、首值1；CLONE复制数据、下一值201。每条路径独立验收，CTAS不能代替CLONE。 |
| AINC-053 | 权限 | SET/ALTER 权限 | 低权限用户执行 session SET、ALTER、写入 | 遵循权限模型；不能越权改变表级 counter。 |

### 2.6 异常、性能与稳定性（7 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果/记录项 |
| --- | --- | --- | --- | --- |
| AINC-054 | 错误恢复 | DDL/DML 中断 | 建表/ALTER/批量插入中注入故障并重试 | 无半成品 metadata、重复 ID 或资源泄漏。 |
| AINC-055 | 性能 | cache大小与旧版默认对比 | 同硬件/数据/并发下比较合入前默认、合入后CACHE=0及1/8/大值；至少3轮 | 记录QPS、p50/p95/p99、分配RPC/跨度、CPU/内存及gap；旧版仅比默认策略。报告中位数、波动与相对变化，性能不能替代034正确性。 |
| AINC-056 | 性能 | inc/off 影响 | 默认与多个 inc/off 组合的多 CN 压测 | 正确性不变，记录协调/吞吐差异。 |
| AINC-057 | 长稳 | 单表热点 2 小时 | 多 CN 高并发自动/显式混合插入 | 无重复、CN 重启、计数器耗尽、内存增长或性能持续退化。 |
| AINC-058 | 长稳 | 多表混合 | 不同类型/cache/session 设置的表混合写 | 配置隔离；无跨表污染。 |
| AINC-059 | 边界 | 接近类型上限 | 多 CN 并发逼近整数上限并尝试续租 | 溢出前不重复，溢出后稳定报错且服务健康。 |
| AINC-060 | 回归 | 非 auto DML/既有表 | 执行普通 PK、sequence（如有）、旧 auto BVT | 无功能/性能回归。 |

### 2.7 发布开关与协议边界（6 条）

本节重启用例使用独占测试集群及同一持久数据目录；停止服务后修改配置并重启，不通过修改运行中对象模拟生产配置切换。

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| AINC-061 | 默认关闭 | 缺省配置/显式false | 分别省略开关、设false，普通/临时表测试CACHE省略/0/非零CREATE，检查目录与分配调用 | 省略/0成功；非零明确拒绝、无残表或分配副作用。无自增列CACHE=0仍合法；与仓库显式开启配置作对照。 |
| AINC-062 | 禁用重启 | 已有非零策略表 | 开启时CACHE=1表写入ID=1，停全集群，关闭开关后重启；执行INSERT、allocator观察、TRUNCATE和LIKE建表 | 使用非零策略的分配/观察路径明确拒绝；TRUNCATE/LIKE拒绝且无残表、原数据保留；SHOW CREATE仍含CACHE=1。涉及internal_auto_increment的SHOW STATUS/info-schema可整句失败，不能要求仅跳过该表。 |
| AINC-063 | 再次开启 | 开→关→开 | 接062且不执行起点维护，重新停服、全CN开启并重启；检查原数据/SHOW并插入 | 原ID=1和CACHE=1保留，下一ID=2，无静默策略降级或禁用期间隐式取号；与064用独立数据分支。 |
| AINC-064 | 禁用维护例外 | ALTER起点仍可用 | 独立表ID=1、CACHE=1，禁用重启后ALTER AUTO_INCREMENT=100；检查SHOW，开启重启再写入 | 起点维护成功并保留策略；禁用时发号仍拒绝；默认session重新开启后首个自动值100。不能把开关理解为禁止所有维护操作。 |
| AINC-065 | 协议门禁 | V80/V81与开关组合 | UT分别控制本地版本、PRE_INSERT发送端/接收端版本及开关；有效非零策略做拒绝/接受对照 | 合入契约下V80拒绝、V81且开启接受；关闭仍拒绝。失败不发布metadata、不构造可静默丢策略的执行路径；不是混合版本支持证明。 |
| AINC-066 | 发布流程 | 全角色升级与降级边界 | 排空事务、停全部旧CN/TN、升级全部角色、确认节点清单/版本后全CN开启；验证新旧表 | 同版本集群读写与策略正确；记录旧进程已停止的证据。不把旧节点重入、混合运行或含非零策略的原地降级列为支持场景；局部门禁不能证明集群准入。 |

### 2.8 观察、生命周期与语法补充（7 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| AINC-067 | 只读观察 | 空CACHE=1重复观察 | 起点10空表，重复SHOW TABLE STATUS、查询information_schema.tables.auto_increment，再插入；UT同时记录分配次数 | 每次观察值10，观察阶段零号段预留，首个插入ID=10；仅比较SHOW结果不足以证明零分配。 |
| AINC-068 | 事务所有者 | 未提交CREATE后观察 | 普通表BEGIN→CREATE(起点10,CACHE=0/1)→SHOW/info-schema；分别COMMIT/ROLLBACK | 事务内读到自己的表及值10；commit后仍可观察，rollback后表不存在。CACHE=1不预留号段，私有观察使用owner事务且不获取外部屏障。 |
| AINC-069 | 跨CN可见性 | 观察过空缓存再读新值 | CACHE=1，CN-A观察空表，CN-B插入并提交，CN-A立即重新观察；交换writer/reader | 默认session起点1时，A先读1，B提交ID=1后A读2；不能一直读旧快照。通过实现的TN有序屏障和MinCommittedTS保证allocator读取，不用sleep/retry掩盖问题；DDL元数据传播单独用提交因果屏障同步。 |
| AINC-070 | 观察错误 | 取消/缺失屏障/屏障失败 | UT阻塞屏障，确认SQL尚未开始；分别释放、取消、缺能力、返回错误，并在同scope重复观察 | 成功释放后点查携带准确frontier；失败时不启动点查、不返回伪成功值、不耗号，释放资源；同scope一致传播屏障错误。 |
| AINC-071 | 临时表 | 普通/临时CREATE事务差异 | 对照068：BEGIN→CREATE TEMPORARY TABLE(CACHE=1)→ROLLBACK→插入/SHOW；同连接执行 | 顶层临时CREATE不被外层rollback撤销，表和CACHE=1仍在、首次ID=1；普通表rollback后不存在。内部COPY按实际事务所有者验证，不能套用顶层临时DDL规则。 |
| AINC-072 | 物理导出恢复 | DUMP TABLE/LOAD TABLE | 默认session、CACHE=1源表写ID=1，flush后经stage/object执行DUMP TABLE；目标LIKE源表后LOAD TABLE，冷CN观察并写入 | 目标CACHE=1与数据1恢复，随后ID=2、总行数2；明确覆盖公开DUMP/LOAD入口、对象路径和Reset，不用逻辑导出结果代替。 |
| AINC-073 | 语法所有权 | 表级/分区级/ALTER | ALTER AUTO_ID_CACHE；PARTITION与SUBPARTITION选项放CACHE=0/1；表级合法选项和已有分区选项作控制 | ALTER CACHE不支持；分区/子分区位置即使0也在解析边界拒绝且不留表；合法表级选项不能误拒绝。 |

### 2.9 元数据性能与策略传递（2 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果/记录项 |
| --- | --- | --- | --- | --- |
| AINC-074 | 观察成本 | 多空表及共享frontier | 多个空CACHE=1表执行SHOW/info-schema；UT覆盖同一次internal_auto_increment向量求值的并发观察、不同求值；性能比较1/100/1024表 | 同求值共享一次屏障，点查均携带该frontier且不缓存offset；不同求值取新frontier。不是整条SQL永远一次屏障；记录求值次数、屏障/点查数与端到端p50/p95，不以旧的无屏障点查基准代替当前延迟。 |
| AINC-075 | 策略绑定 | 冷加载/Reset物理ID | UT覆盖已解析TableDef/relation策略、旧ID→替换ID、已绑定新ID、无关ID与无hint；SQL覆盖TRUNCATE/COPY/CLONE/LOAD | 已知且匹配策略复用、Reset正确重绑；无关/缺失hint回退持久发现，不能跨表借用策略。开关/版本检查仍有效；记录默认表大目录冷加载与Reset成本，不要求所有路径零目录查询。 |

## 3. 基线与判定

### 3.1 ID 正确性基线

每个并发/分布式测试保留完整写入记录，按以下规则判定：

1. 无预期冲突/故障的负载：所有计划写入均成功提交，新增实际行数等于计划行数；不得跳过错误后只检查结果表。预置数据与本轮新增数据分开计数。
2. 每行写入带唯一的 `(writer_id, request_id)`；批量请求为每行分配独立request_id或额外row_index。逐项对账计划集合、提交结果与实际数据，无丢行、重复执行或未解释结果。测试驱动记录重试次数，故障场景的未知提交结果必须对账后判断，不能盲目重试。
3. 单独统计自动分配导致的duplicate-key，无预期冲突场景必须为0。显式重复键负向测试独立标注；其他失败必须保留，不能计入成功或过滤。
4. `COUNT(*) = COUNT(DISTINCT id)`仅为补充条件：主键可拦截重复候选，甚至全部写失败时0=0仍成立。有多个自增列时逐列检查；可另设无唯一约束的自增列对照表捕获重复候选，但不替代成功数核对。
5. 自动ID在列类型范围内；按每行对应语句的inc/effective_off验证同余关系，不检查显式ID。同session并发/续租可跳项，跨session不要求连续或全局单调。
6. gap关联真实号段、显式高值、rollback、重启等事件；业务回滚不要求回收共享号段。显式ALTER可重置预留水位；禁止与现存数据冲突，不能用笼统“counter不回退”代替M/H/N判定。

### 3.2 计划与元数据判定

| 场景 | 必须观察到 | 禁止观察到 |
| --- | --- | --- |
| ALTER 起始值 | 区分M/H/N，显式重置与数据边界协调；默认session下M=200、ALTER100后下一值201 | 与现存行冲突、旧cache跨epoch错误使用、半更新；H下降本身不算失败。 |
| AUTO_ID_CACHE | 表级配置持久、分配区间不重叠 | 跨表共享 cache、无解释的重复 ID。 |
| Session 变量 | session 局部生效、跨 CN 传递正确 | 其他 session/连接池泄漏。 |
| 重启/恢复 | 已发 ID 不复用，恢复点与 counter 协调 | 与现有行冲突、counter 丢失或无界跳跃。 |
| 关闭开关 | SHOW CREATE保真、非零策略使用拒绝、独立ALTER起点维护仍可用 | 静默退回CN默认、拒绝后残表、把关闭当数据回滚。 |
| 只读观察 | 不分配号段；私有txn读自身，已提交观察按frontier读取 | 缓存旧offset、忽略取消/屏障错误、观察导致取号。 |

### 3.3 CACHE 分配的确定性证据

复用PR的allocator/store测试替身记录调用和原始跨度；SQL结果与性能只作补充。显式高值引起的水位更新与自动号段申请分别计数。以下来自已有UT的精确fixture，不能泛化为每个生产batch都必须相同次数。

| fixture | 自动结果 | 分配请求的原始跨度列表 |
| --- | --- | --- |
| CACHE=1，默认session，构造/估算预取/低水位路径 | 无自动输出 | 空列表 |
| CACHE=1，空输入/全显式输入 | 显式值按输入保留；高值水位另计 | 空列表；空输入还需SQL空结果INSERT SELECT控制 |
| CACHE=1，默认session，三个NULL | 1、2、3 | `[3]` |
| CACHE=1，默认session，NULL/100/NULL | 1、100、101 | `[2, 1]`；显式高值可使先前预留尾部失效，不能简单断言总跨度=自动行数 |
| CACHE=1，inc=3/off=2，三个NULL | 2、5、8 | `[9]`；原始数值跨度不是输出行数 |
| CACHE=1，inc=3/off=2，两个并发单行请求 | 集合为2、5 | `[3, 3]`；等待者不放大按需预留 |
| CACHE=8，同上非默认session fixture | 集合为2、5 | 单次请求，跨度在8..16；无后台预取，不采用CN默认10000 |

### 3.4 验收与证据分层

- SQL/BVT检查用户可见结果、SHOW、错误和持久化；UT证明分配次数、跨度、事务所有者、屏障时序、协议拒绝及策略绑定。
- 多CN集成验证冷加载、即时观察、真实DDL生命周期；独占重启集群验证关闭/开启配置及数据持久性。
- 性能比较以同环境、相同数据/并发和至少3轮结果为依据；报告绝对值、相对变化、波动及预先约定阈值。尚未约定阈值时报告测量结果，不自行宣称性能验收通过。
- 下列映射为已存在测试的复用入口。执行报告逐用例填写实际commit、命令、证据、结果及未覆盖项；历史PR中的“通过”不等于本轮通过。

## 4. 结果记录要求

- 记录精确MatrixOne commit/镜像、全角色版本/节点清单、各CN开关、协议版本、CACHE、session inc/off/effective_off、表定义、规模、并发和seed。
- 并发记录每行writer_id/request_id、session/CN、自动/显式标记、提交结果、重试次数；提供计划/成功/失败/未知结果数、实际行数及逐请求对账，单列自动分配duplicate-key。
- cache/故障用例记录分配区间、实际 gap、重启/故障时间、系统日志或 metrics；未观察到内部指标时须明确说明。
- 性能至少 3 轮，记录 QPS、p50/p95、CPU/内存、counter RPC/等待（可得时）；仅同环境、同数据快照可比较。
- 缺陷附最小建表 SQL、变量设置、并发脚本/seed、完整 ID 集合、拓扑及错误日志。
- 观察用例记录owner事务、提交时间戳、frontier、屏障/点查/分配次数；取消与错误路径保留资源清理证据，不以sleep/retry后的成功覆盖首次失败。
- 生命周期记录操作前后物理表ID、持久CACHE、M/H/N、数据与下一个生成值；重启记录停服/配置/启动步骤。每个参数组合独立给结果。

## 5. 已有测试复用映射

以下名称已核对PR #28528的合入内容，定位入口为[PR文件变更](https://github.com/matrixorigin/matrixone/pull/28528/files)。映射表示相关覆盖，不表示单个函数覆盖整行用例全部参数；未覆盖的SQL组合、多CN压力和性能仍须补执行。

| 设计用例 | 已有测试入口 | 复用证据/还需执行 |
| --- | --- | --- |
| 002、009、010、012 | `TestAutoIDCachePlanAndPersistence`、`TestAutoIDCacheZeroWithoutAutoColumn`、`TestAutoIDCachePlanRejectsInvalidOptions`、`TestAutoIDCacheAlterFinalDefinition` | 合法/非法选项、无auto列零策略、最终schema归一化；补多自增列SQL及起点边界。 |
| 034、035、037 | `TestAutoIDCacheDemandOnly`、`TestAutoIDCacheExplicitSpanAndConcurrentDemand` | 复用§3.3精确请求列表；补空SQL输入、实际大batch与默认session组合。 |
| 040、061..065 | `TestAutoIDCacheOptInConfig`、`TestAutoIDCacheDisabledHasNoAllocatorSideEffects`、`TestAutoIDCacheDDLAndProtocolGate`、`TestAutoIDCacheDDLRejectsBeforeMetadataWork`、`TestAutoIDCacheRemoteWireGate`、`TestAutoIDCacheRestartAndDisabledNode` | 开关、副作用、协议、独占重启；064维护例外和066部署流程须单独记录。 |
| 008、044、067..070 | `TestAutoIDCacheUncommittedObservation`、`TestAutoIDCacheColdServicesAndProbe`、`TestAutoIDCachePointObservation`、`TestAutoIDCachePointObservationWaitsForBarrierBeforeRead`、`TestAutoIDCachePointObservationBarrierErrors`、`TestAutoIDCachePointObservationErrors` | 事务所有者、无预留、读前屏障及错误；配合公开SQL/多CN即时观察。 |
| 012、021、048、052、068、069、071、072 | `TestAutoIDCachePublicLifecycle`；现有`auto_id_cache.sql`及对应`.result` | 复用普通/临时事务差异、COPY/RENAME/LIKE、TRUNCATE、SHOW、DUMP/LOAD；CLONE精确结果映射BVT，保留各路径独立验收。 |
| 073 | `TestAutoIDCacheRejectsPartitionOwnership`、`TestAutoIDCacheASTOwnership`；公开生命周期及BVT | 分区/子分区位置拒绝与表级正向控制；补ALTER CACHE拒绝入口。 |
| 074、075 | `TestAutoIDCacheObservationScopeConcurrent`、`TestAutoIDCacheKnownPolicyObservationSQL`、`TestAutoIDCacheResetPolicyRebinding`、`TestAutoIDCacheResetCarriesReplacementPolicy` | 共享屏障/不缓存offset、已知策略/Reset重绑和回退；大目录及带屏障端到端性能单独测量。 |

本次文档修订完成标准：研发review六类意见均有对应预期/用例、编号与数量一致、Markdown表格正常渲染。数据库执行状态另行记录，不将文档检查标记为UT/BVT通过。
