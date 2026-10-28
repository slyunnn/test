# MatrixOne Enhanced AUTO_INCREMENT 测试设计（必要项评审版）

> 本设计覆盖 Feature [#23371](https://github.com/matrixorigin/matrixone/issues/23371)：增强 AUTO_INCREMENT，支持修改起始值、`AUTO_ID_CACHE` 表选项，以及 `auto_increment_increment` / `auto_increment_offset` 会话变量。核心目标是在分布式环境中保证 ID 唯一、序列语义明确、缓存可控且与既有表兼容。

> 2026-10-09 研发 review 修订：以已合入 [PR #28528](https://github.com/matrixorigin/matrixone/pull/28528)、合入提交 `2b56f0d5e11b7af0b29cf91bd1d0f03430da3f75` 为契约基线。本 PR 主要新增表级 AUTO_ID_CACHE；ALTER 起点和 session 序列复用上游能力并验证组合行为。本文为测试设计与已有测试映射，不代表本轮已执行 UT/BVT 或性能测试。

> 2026-10-28 必要项评审：75 条收敛为 41 个执行单元。保留研发 review 的全部确定语义与发布边界；原编号不重排。以改动路径为单位复用 UT/BVT，不要求每条用例交叉跑完整单/多 CN 矩阵。

## 1. 测试原则

| 项目 | 说明 |
| --- | --- |
| 唯一性优先 | 无论单/多 CN、缓存大小、并发、事务或显式插入，自动生成 ID 都不得重复；允许的间隙必须可由缓存或回滚语义解释。 |
| DDL 原子性 | ALTER 起始值、schema 演进、建表/恢复失败不得留下部分元数据；本期不支持 ALTER AUTO_ID_CACHE。 |
| 会话隔离 | increment/offset 必须严格按 session 生效；不同 session 可并发使用不同设置，不能相互泄漏。 |
| 兼容性 | 不显式使用新语法的既有 AUTO_INCREMENT 表保持原有默认行为；MySQL 兼容语义的差异须冻结并记录。 |
| 定向分布式验证 | 分配唯一性、session隔离/序列和已提交观察必须有多CN直接证据；DDL参数矩阵不重复全拓扑。不得仅凭主键去重后行数推断分配安全。 |
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

### 1.2 执行分层

| 层级 | 必须验证 | 不扩大为 |
| --- | --- | --- |
| SQL/BVT | 参数、精确ID、SHOW、事务与生命周期 | 所有普通DML/权限/CDC/PITR回归 |
| allocator/store UT | 请求跨度、按需分配、屏障、门禁、策略重绑 | 用吞吐或SHOW结果推断内部正确 |
| 多CN集成 | 027/038/042写入对账、069即时观察、冷加载 | 所有DDL与类型参数的全拓扑笛卡尔积 |
| 独占重启集群 | 040及061..066开关/升级边界 | 混合版本或带非零策略表原地降级支持 |

### 1.3 必要覆盖与精简说明

| 一级模块 | 执行单元 | 合并与范围 |
| --- | ---: | --- |
| DDL与元数据 | 7 | 起点提高/重复设置并入006；SHOW随DDL/观察检查 |
| 默认ID与显式输入 | 5 | 自动写法/批量归014；显式低/高值归018；类型上限归020 |
| Session序列 | 5 | 步长/offset/default归025；隔离/多CN归027 |
| Cache与并发 | 6 | 续租归035；跨表归036；跨CN分片归038 |
| 事务与复制生命周期 | 3 | commit/rollback/savepoint归046；逻辑导出048；各复制路径052 |
| 开关与发布边界 | 6 | 保留061..066全部确定预期 |
| 观察、语法及策略传递 | 9 | 保留067..075，074只保留共享frontier正确性UT |
| **合计** | **41** | 保留编号间隙；一个owner内的独立入口仍逐项记录 |

删除独立性能/热点/2小时长稳、PITR/CDC、通用权限、LAST_INSERT_ID及泛化故障矩阵，不在附录追加必跑项。失败数据原子性、CN重启安全和完整写入对账保留；038/042用有界并发，不做压力验收。旧表默认兼容由002/061/066覆盖。

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

### 2.1 DDL 与元数据（7 个执行单元）

通用表：`t(id BIGINT AUTO_INCREMENT PRIMARY KEY, v VARCHAR(50))`；需同时覆盖有/无历史数据、单列/复合约束兼容表和不同整数宽度的 AUTO_INCREMENT 列。

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| AINC-002 | CREATE | CACHE 合法值 | 省略、0、1、2、8、1000000 分别建表，冷 CN 加载后 SHOW/写入 | 省略/0 继承 CN 默认，SHOW 省略零选项；正数持久且 SHOW 如实呈现，冷加载仍遵循表策略。 |
| AINC-003 | CREATE | 起始值 + cache | 默认 session、CACHE=1，分别以 AUTO_INCREMENT=0、10 建新表并插入；inc=3/off=2 时以起点10建表 | 起点0合法，默认 session 首值1；起点10首值10；inc=3/off=2 生成11、14、17。session 参数不持久化为表策略。 |
| AINC-006 | ALTER | 起点提高、重复与低于M/H | 默认session、CACHE=1/8：空表ALTER1000→首值1000；有数据提高；重复ALTER；独立表构造M=200、H>200，再ALTER100 | 默认session低设100成功且下一值201；提高后不低于目标、无重复；重复设置状态一致。记录M/H/N，允许显式重置H；保留CACHE，不以旧H作为下界。 |
| AINC-009 | 约束 | 无自增列/多个自增列 | 普通/临时表无自增列时分别指定CACHE=0/1；建立含两个可见自增列的表并混合显式/自动写入 | 无自增列CACHE=0成功、非零拒绝且无残表；多个自增列合法，各列独立分配且遵循同一表级策略。 |
| AINC-010 | 参数 | CACHE非法值与起点边界分离 | CACHE=-1、1.5、1000001、重复选项及uint64溢出；另测起点0/合法上界/超列类型上界 | 非法CACHE拒绝、不截断、不留残表；CACHE=0和建表起点0是正向控制。起点测试分别记录DDL接受与发号溢出，不把两阶段错误混同。 |
| AINC-011 | DDL 原子性 | ALTER 与并发 DML | 一边 ALTER、一边 INSERT/SELECT | DDL/DML 遵循锁与原子性；无重复/半更新状态。 |
| AINC-012 | Schema 演进 | 删除/保留/转移自增属性 | CACHE=0/1/8，DROP最后一个自增列、MODIFY去掉最后一个自增属性；对照保留、转移属性及多自增列仅删除一列 | 最终无可见自增列时成功且策略归0，SHOW无CACHE，普通数据保留；最终仍有自增列则保留策略、后续各列安全发号。 |

### 2.2 默认 ID 与显式插入（5 个执行单元）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| AINC-014 | 自动输入 | 单/多行、空输入和默认写法 | CACHE=0/1/8执行单行、多行VALUES、INSERT SELECT及空结果；对照省略id、NULL、DEFAULT及显式0的实际SQL mode契约 | 每个实际自动行唯一；空SQL输入不申请自动号段；合法默认写法一致。0按已冻结产品规则判定，不擅自与NULL等价。CACHE=1精确跨度由034证明。 |
| AINC-018 | 显式输入 | 低值、高值与负值 | 独立CACHE=1表，插入显式100再自动；另从有数据表插入未占用低值、负值再自动；与非默认session组合见031/042 | 默认新表100后生成101；低/负显式值不使水位错误回退或推进；显式键按约束校验。自动发号不与现存值冲突。 |
| AINC-019 | 冲突 | 已存在显式 ID | 插入重复主键、随后自动插入；记录错误、数据和分配事件 | 重复键明确报错，失败语句数据原子性正确；后续自动发号安全。允许已申请共享号段产生gap，不要求失败退还或回滚counter。 |
| AINC-020 | 类型边界 | 有/无符号整数及耗尽续租 | 复用类型BVT，在合法整数类型上验证边界/溢出；以一个窄类型在CACHE>1下跨续租逼近上限 | 合法值不回绕/重复；超过类型范围稳定报错，服务可继续处理健康表；不再为各类型重复多CN压力。 |
| AINC-021 | 删除 | DELETE/TRUNCATE 后生成 | 默认session、CACHE=1/8，删除最高值、DELETE全表、TRUNCATE后重插；检查SHOW | DELETE不重置发号；TRUNCATE后首值1且保留CACHE。TRUNCATE按隐式提交验证，不要求外层ROLLBACK撤销它。 |

### 2.3 Session increment 与 offset（5 个执行单元）

令 `inc = auto_increment_increment`、`off = auto_increment_offset`；变量各自合法时，`effective_off = off > inc ? 1 : off`。仅自动生成行应满足 `id ≡ effective_off (mod inc)`，首项还受起点、现存数据及可用号段约束。显式ID不参与同余检查；并发/续租允许跳项，不能要求所有相邻结果差固定为inc。

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| AINC-025 | 序列 | 默认与合法inc/off矩阵 | 新session记录默认值；独立CACHE=1表分别设置inc=2/3/合法大值及off=1/中间/inc，连续/批量写入 | 默认1/1；自动行满足有效offset同余，单writer精确fixture符合序列。不同session/续租可跳项，不要求全局相邻差固定。 |
| AINC-027 | session隔离/多CN | 不同序列及连接生命周期 | CN-A/B各自建立连接、设置不同inc/off并交替写同表；切换/重连新session，若连接池有显式reset则验证该入口 | 自动ID全局唯一且符合各writer变量；新连接恢复默认，现有连接不泄漏设置。记录真实连接路由，不假定同一TCP session可跨CN迁移。 |
| AINC-028 | 切换 | session 内动态 SET | 先默认插入，再 SET inc/off 并继续插入 | 切换仅影响后续分配；无重复或错误回退。 |
| AINC-030 | 参数 | 非法范围与offset归一化 | 分别测试变量自身非法范围/类型；独立新表设置inc=3/off=5后插入三个NULL | 各自合法范围内off>inc可设置，生成时有效offset=1，得到1、4、7；真正非法输入按变量定义报错，不能把off>inc归入拒绝组。 |
| AINC-031 | 组合 | ALTER与显式高值后的inc/off | 独立CACHE=1空表ALTER10，分别inc=3/off=2和inc=3/off=5；另插显式高ID再自动 | ALTER10后首值分别11和10；高值后自动候选高于现存边界且满足effective_off；不沿用ALTER前H作下界。 |

### 2.4 Cache、并发与分布式（6 个执行单元）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| AINC-034 | cache | CACHE=1真实需求分配 | UT记录分配调用/原始跨度，覆盖构造、空输入、全显式、估算、低水位、并发等待者；SQL混合(NULL,100,NULL) | 前述无自动需求路径不申请自动号段；显式高值必要水位推进另计。混合结果1、100、101；不因估算/低水位/等待者扩大预留；精确断言见§3.3。 |
| AINC-035 | cache | CACHE>1跨度及耗尽续租 | CACHE=2/8，默认及inc=3/off=2，构造批量需求大于基础跨度并跨多个号段，UT记录请求/预取事件 | CACHE是基础原始数值跨度而非固定生成行数；需求可扩大，覆盖CN默认跨度；非默认session即使CACHE>1仍无后台预取。续租区间不重叠、无重复。 |
| AINC-036 | cache | 不同表 cache 隔离 | 同时写入 cache 不同的多表 | 每表配置独立，不能借用/污染其他计数器。 |
| AINC-038 | 多CN分配 | 并发与交替消费预留号段 | A/B各预留cache、交替及有界并发写同表，跨续租；记录writer_id/request_id、计划行数、提交结果和ID | 全部计划写入成功且实际行数=预期；自动duplicate-key为0，逐请求不丢不重，各CN号段不重叠。允许全局顺序交错，唯一性等式仅附加检查。 |
| AINC-040 | 故障 | CN 重启/切换 | cache 已预留但未用尽时重启 CN，再写入 | 已发 ID 不复用；未用 cache 的 gap 符合持久化策略。 |
| AINC-042 | 并发交互 | 自动写与显式高值/ALTER竞争 | 有界并发自动写期间，一次插入未占用显式高ID；独立阶段按011交错ALTER；按每条语句inc/off核对 | 无自动分配duplicate-key、死锁或旧cache跨epoch使用；计划/成功/失败/实际行逐项对账。显式行不检查同余，故障未知提交不得盲重试。 |

### 2.5 事务与复制生命周期（3 个执行单元）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| AINC-046 | 事务 | commit、rollback及savepoint/失败语句 | 独立CACHE=1/8表写多行后分别COMMIT/ROLLBACK；已支持的savepoint部分回滚或失败后继续写入 | 数据遵循事务边界；共享号段不随业务事务回滚，允许gap；后续成功且无冲突，不要求counter与数据同步回退。 |
| AINC-048 | 备份恢复 | 逻辑导出/恢复 | 默认session、CACHE=1表写入1，导出DDL和数据后恢复；物理DUMP/LOAD另按072执行 | 正数CACHE被导出并恢复，数据1保留，下一自动值2；0策略保持默认。不以逻辑导出替代DUMP TABLE/LOAD TABLE。 |
| AINC-052 | 复制/重命名 | RENAME、LIKE、CLONE、COPY独立路径 | 默认session、CACHE=1、最大ID=200；独立副本执行RENAME、CREATE TABLE dst LIKE src、CREATE TABLE dst CLONE src、ALTER TABLE src ADD COLUMN extra INT, ALGORITHM=COPY | 各路径保留CACHE；RENAME/COPY保留数据、下一值201；LIKE为空表、首值1；CLONE复制数据、下一值201。每条路径独立验收，CTAS不能代替CLONE。 |

### 2.6 发布开关与协议边界（6 个执行单元）

本节重启用例使用独占测试集群及同一持久数据目录；停止服务后修改配置并重启，不通过修改运行中对象模拟生产配置切换。

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| AINC-061 | 默认关闭 | 缺省配置/显式false | 分别省略开关、设false，普通/临时表测试CACHE省略/0/非零CREATE，检查目录与分配调用 | 省略/0成功；非零明确拒绝、无残表或分配副作用。无自增列CACHE=0仍合法；与仓库显式开启配置作对照。 |
| AINC-062 | 禁用重启 | 已有非零策略表 | 开启时CACHE=1表写入ID=1，停全集群，关闭开关后重启；执行INSERT、allocator观察、TRUNCATE和LIKE建表 | 使用非零策略的分配/观察路径明确拒绝；TRUNCATE/LIKE拒绝且无残表、原数据保留；SHOW CREATE仍含CACHE=1。涉及internal_auto_increment的SHOW STATUS/info-schema可整句失败，不能要求仅跳过该表。 |
| AINC-063 | 再次开启 | 开→关→开 | 接062且不执行起点维护，重新停服、全CN开启并重启；检查原数据/SHOW并插入 | 原ID=1和CACHE=1保留，下一ID=2，无静默策略降级或禁用期间隐式取号；与064用独立数据分支。 |
| AINC-064 | 禁用维护例外 | ALTER起点仍可用 | 独立表ID=1、CACHE=1，禁用重启后ALTER AUTO_INCREMENT=100；检查SHOW，开启重启再写入 | 起点维护成功并保留策略；禁用时发号仍拒绝；默认session重新开启后首个自动值100。不能把开关理解为禁止所有维护操作。 |
| AINC-065 | 协议门禁 | V80/V81与开关组合 | UT分别控制本地版本、PRE_INSERT发送端/接收端版本及开关；有效非零策略做拒绝/接受对照 | 合入契约下V80拒绝、V81且开启接受；关闭仍拒绝。失败不发布metadata、不构造可静默丢策略的执行路径；不是混合版本支持证明。 |
| AINC-066 | 发布流程 | 全角色升级与降级边界 | 排空事务、停全部旧CN/TN、升级全部角色、确认节点清单/版本后全CN开启；验证新旧表 | 同版本集群读写与策略正确；记录旧进程已停止的证据。不把旧节点重入、混合运行或含非零策略的原地降级列为支持场景；局部门禁不能证明集群准入。 |

### 2.7 观察、生命周期与语法补充（7 个执行单元）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| AINC-067 | 只读观察 | 空CACHE=1重复观察 | 起点10空表，重复SHOW TABLE STATUS、查询information_schema.tables.auto_increment，再插入；UT同时记录分配次数 | 每次观察值10，观察阶段零号段预留，首个插入ID=10；仅比较SHOW结果不足以证明零分配。 |
| AINC-068 | 事务所有者 | 未提交CREATE后观察 | 普通表BEGIN→CREATE(起点10,CACHE=0/1)→SHOW/info-schema；分别COMMIT/ROLLBACK | 事务内读到自己的表及值10；commit后仍可观察，rollback后表不存在。CACHE=1不预留号段，私有观察使用owner事务且不获取外部屏障。 |
| AINC-069 | 跨CN可见性 | 观察过空缓存再读新值 | CACHE=1，CN-A观察空表，CN-B插入并提交，CN-A立即重新观察；交换writer/reader | 默认session起点1时，A先读1，B提交ID=1后A读2；不能一直读旧快照。通过实现的TN有序屏障和MinCommittedTS保证allocator读取，不用sleep/retry掩盖问题；DDL元数据传播单独用提交因果屏障同步。 |
| AINC-070 | 观察错误 | 取消/缺失屏障/屏障失败 | UT阻塞屏障，确认SQL尚未开始；分别释放、取消、缺能力、返回错误，并在同scope重复观察 | 成功释放后点查携带准确frontier；失败时不启动点查、不返回伪成功值、不耗号，释放资源；同scope一致传播屏障错误。 |
| AINC-071 | 临时表 | 普通/临时CREATE事务差异 | 对照068：BEGIN→CREATE TEMPORARY TABLE(CACHE=1)→ROLLBACK→插入/SHOW；同连接执行 | 顶层临时CREATE不被外层rollback撤销，表和CACHE=1仍在、首次ID=1；普通表rollback后不存在。内部COPY按实际事务所有者验证，不能套用顶层临时DDL规则。 |
| AINC-072 | 物理导出恢复 | DUMP TABLE/LOAD TABLE | 默认session、CACHE=1源表写ID=1，flush后经stage/object执行DUMP TABLE；目标LIKE源表后LOAD TABLE，冷CN观察并写入 | 目标CACHE=1与数据1恢复，随后ID=2、总行数2；明确覆盖公开DUMP/LOAD入口、对象路径和Reset，不用逻辑导出结果代替。 |
| AINC-073 | 语法所有权 | 表级/分区级/ALTER | ALTER AUTO_ID_CACHE；PARTITION与SUBPARTITION选项放CACHE=0/1；表级合法选项和已有分区选项作控制 | ALTER CACHE不支持；分区/子分区位置即使0也在解析边界拒绝且不留表；合法表级选项不能误拒绝。 |

### 2.8 观察scope与策略传递（2 个执行单元）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果/记录项 |
| --- | --- | --- | --- | --- |
| AINC-074 | 观察scope | 共享frontier但不缓存offset | UT覆盖一次internal_auto_increment向量求值的多表/并发观察，以及第二次独立求值；记录屏障、点查和请求scope | 同求值共享一次屏障，全部点查携带该frontier且不缓存offset；不同求值获取新frontier。不是整条SQL永远一次屏障；无自动号段分配。 |
| AINC-075 | 策略绑定 | 冷加载及Reset物理ID | UT覆盖TableDef/relation已知策略、旧→替换ID、已绑定新ID、无关ID及无hint；关联021/052/072的SQL证据 | 仅匹配策略可复用，Reset正确重绑；无关/缺失hint回退持久发现，不跨表借用；开关/版本检查仍有效，不要求每条路径零目录查询。 |

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
- 下列映射为已存在测试的复用入口。执行报告逐用例填写实际commit、命令、证据、结果及未覆盖项；历史PR中的“通过”不等于本轮通过。

## 4. 结果记录要求

- 记录精确MatrixOne commit/镜像、全角色版本/节点清单、各CN开关、协议版本、CACHE、session inc/off/effective_off、表定义、规模、并发和seed。
- 并发记录每行writer_id/request_id、session/CN、自动/显式标记、提交结果、重试次数；提供计划/成功/失败/未知结果数、实际行数及逐请求对账，单列自动分配duplicate-key。
- cache/故障用例记录分配区间、实际 gap、重启/故障时间、系统日志或 metrics；未观察到内部指标时须明确说明。
- 缺陷附最小建表 SQL、变量设置、并发脚本/seed、完整 ID 集合、拓扑及错误日志。
- 观察用例记录owner事务、提交时间戳、frontier、屏障/点查/分配次数；取消与错误路径保留资源清理证据，不以sleep/retry后的成功覆盖首次失败。
- 生命周期记录操作前后物理表ID、持久CACHE、M/H/N、数据与下一个生成值；重启记录停服/配置/启动步骤。每个参数组合独立给结果。

## 5. 已有测试复用映射

以下名称已核对PR #28528的合入内容，定位入口为[PR文件变更](https://github.com/matrixorigin/matrixone/pull/28528/files)。映射表示相关覆盖，不表示单个函数覆盖整行用例全部参数；未覆盖的本表必要SQL组合与多CN时序仍须补证据；不追加通用压力/性能范围。

| 设计用例 | 已有测试入口 | 复用证据/还需执行 |
| --- | --- | --- |
| 002、003、006、009、010、012 | `TestAutoIDCachePlanAndPersistence`、`TestAutoIDCacheZeroWithoutAutoColumn`、`TestAutoIDCachePlanRejectsInvalidOptions`、`TestAutoIDCacheAlterFinalDefinition` | 合法/非法选项、无auto列零策略、最终schema归一化；补多自增列SQL及起点边界。 |
| 034、035 | `TestAutoIDCacheDemandOnly`、`TestAutoIDCacheExplicitSpanAndConcurrentDemand` | 复用§3.3精确请求列表；补空SQL输入、实际大batch与默认session组合。 |
| 040、061..065 | `TestAutoIDCacheOptInConfig`、`TestAutoIDCacheDisabledHasNoAllocatorSideEffects`、`TestAutoIDCacheDDLAndProtocolGate`、`TestAutoIDCacheDDLRejectsBeforeMetadataWork`、`TestAutoIDCacheRemoteWireGate`、`TestAutoIDCacheRestartAndDisabledNode` | 开关、副作用、协议、独占重启；064维护例外和066部署流程须单独记录。 |
| 067..070 | `TestAutoIDCacheUncommittedObservation`、`TestAutoIDCacheColdServicesAndProbe`、`TestAutoIDCachePointObservation`、`TestAutoIDCachePointObservationWaitsForBarrierBeforeRead`、`TestAutoIDCachePointObservationBarrierErrors`、`TestAutoIDCachePointObservationErrors` | 事务所有者、无预留、读前屏障及错误；配合公开SQL/多CN即时观察。 |
| 012、021、048、052、068、069、071、072 | `TestAutoIDCachePublicLifecycle`；现有`auto_id_cache.sql`及对应`.result` | 复用普通/临时事务差异、COPY/RENAME/LIKE、TRUNCATE、SHOW、DUMP/LOAD；CLONE精确结果映射BVT，保留各路径独立验收。 |
| 073 | `TestAutoIDCacheRejectsPartitionOwnership`、`TestAutoIDCacheASTOwnership`；公开生命周期及BVT | 分区/子分区位置拒绝与表级正向控制；补ALTER CACHE拒绝入口。 |
| 074、075 | `TestAutoIDCacheObservationScopeConcurrent`、`TestAutoIDCacheKnownPolicyObservationSQL`、`TestAutoIDCacheResetPolicyRebinding`、`TestAutoIDCacheResetCarriesReplacementPolicy` | 共享屏障/不缓存offset、已知策略/Reset重绑和回退；SQL证据关联021/052/072，不追加大目录性能。 |

本次必要项评审保留研发review六类要求：合法参数与M/H/N、默认关闭发布边界、确定性需求分配、不耗号及事务观察、精确生命周期、完整并发对账。重复总括项008/044分别归DDL和067..075，不能漏掉其底层断言。数据库执行状态另行记录，不将文档检查标记为UT/BVT通过。
