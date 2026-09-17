# MatrixOne Enhanced AUTO_INCREMENT 测试设计

> 本设计覆盖 Feature [#23371](https://github.com/matrixorigin/matrixone/issues/23371)：增强 AUTO_INCREMENT，支持修改起始值、`AUTO_ID_CACHE` 表选项，以及 `auto_increment_increment` / `auto_increment_offset` 会话变量。核心目标是在分布式环境中保证 ID 唯一、序列语义明确、缓存可控且与既有表兼容。

## 1. 测试原则

| 项目 | 说明 |
| --- | --- |
| 唯一性优先 | 无论单/多 CN、缓存大小、并发、事务或显式插入，自动生成 ID 都不得重复；允许的间隙必须可由缓存或回滚语义解释。 |
| DDL 原子性 | ALTER 起始值、表选项变更、建表/恢复失败不得留下部分元数据或破坏现有计数器。 |
| 会话隔离 | increment/offset 必须严格按 session 生效；不同 session 可并发使用不同设置，不能相互泄漏。 |
| 兼容性 | 不显式使用新语法的既有 AUTO_INCREMENT 表保持原有默认行为；MySQL 兼容语义的差异须冻结并记录。 |
| 分布式验证 | ID 分配依赖分布式 counter 和本地 cache，所有核心用例须覆盖单 CN 与多 CN，且以全局唯一性和最终序列规则判定。 |
| 结果与状态双校验 | 除 INSERT 结果外，检查 SHOW CREATE、系统元数据、重连/重启后的下一 ID、执行计划/日志中的 cache 分配证据（可获得时）。 |

### 1.1 基本信息

| 项目 | 内容 |
| --- | --- |
| MatrixOne 基线 | Feature #23371 合入后的精确 commit、镜像和 CN/TN 拓扑（执行前冻结）。 |
| DDL 入口 | `ALTER TABLE t AUTO_INCREMENT = n`；`CREATE TABLE ... AUTO_ID_CACHE = n`（最终语法以实现为准）。 |
| 会话变量 | `auto_increment_increment`、`auto_increment_offset`。 |
| 默认行为 | 不设置新选项时继承现有 AUTO_INCREMENT 的起始值、步长与 cache 行为。 |
| 关键对象 | AUTO_INCREMENT 列、表级 counter 元数据、分布式 ID cache、session 变量。 |
| 主要风险 | 并发重复/乱序、手工插入冲突、counter 回退、cache 失效、DDL/DML 原子性、备份恢复后序列漂移。 |

### 1.2 执行矩阵

| 测试类型 | 单 CN | 多 CN | 通过标准 |
| --- | --- | --- | --- |
| DDL/元数据 | 执行 | 执行 | 定义、SHOW、错误与计数器状态一致。 |
| 默认生成/显式插入 | 执行 | 执行 | 唯一、步长/offset 正确，无不可解释冲突。 |
| Session 变量 | 执行 | 执行 | 每 session 独立且序列满足公式。 |
| Cache/并发 | 执行 | 必须执行 | 全局无重复；cache 间隙、性能与失效可解释。 |
| 事务/恢复/升级 | 执行 | 执行 | 状态持久、恢复正确，旧表兼容。 |
| 性能/长稳 | 基线 | 必须执行 | 吞吐稳定，无计数器热点、泄漏或 CN 异常。 |

### 1.3 覆盖范围

| 一级模块 | 用例数 | 主要覆盖内容 |
| --- | ---: | --- |
| DDL 与元数据 | 12 | ALTER 起始值、建表 option、SHOW、非法参数、schema 变更 |
| 默认 ID 与显式插入 | 11 | 起始/边界、批量、NULL/0、手工值、类型 |
| Session increment/offset | 10 | 序列公式、隔离、切换、非法变量、复制式分配 |
| Cache、并发与分布式 | 11 | cache 大小、分配、CN 重启、并发、热点 |
| 事务、恢复与兼容 | 9 | commit/rollback、backup/PITR、升级、rename/truncate |
| 异常、性能与可观测性 | 7 | DDL/DML 故障、性能、长稳、诊断 |
| **合计** | **60** |  |

### 1.4 参考资料

| 资料 | 地址 |
| --- | --- |
| Feature | <https://github.com/matrixorigin/matrixone/issues/23371> |
| 兼容性参考 | <https://docs.pingcap.com/tidb/stable/auto-increment> |
| 现有 AUTO_INCREMENT 回归 | MatrixOne 当前 DDL/DML、事务、备份恢复和多 CN 测试。 |

## 2. 测试用例

### 2.1 DDL 与元数据（12 条）

通用表：`t(id BIGINT AUTO_INCREMENT PRIMARY KEY, v VARCHAR(50))`；需同时覆盖有/无历史数据、单列/复合约束兼容表和不同整数宽度的 AUTO_INCREMENT 列。

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| AINC-001 | CREATE | 默认 AUTO_INCREMENT 回归 | 不带新选项建表并连续插入 | 默认起始、连续性和 SHOW 与变更前一致。 |
| AINC-002 | CREATE | `AUTO_ID_CACHE` 合法值 | 分别以小/中/大合法 cache 创建表，SHOW CREATE | 建表成功，选项持久化并按定义生效。 |
| AINC-003 | CREATE | 建表内起始值 + cache | 在 CREATE TABLE 同时定义 AUTO_INCREMENT 起始值及 cache | 首个/后续 ID 与配置一致，metadata 完整。 |
| AINC-004 | ALTER | 空表提高起始值 | `ALTER TABLE t AUTO_INCREMENT = 1000` 后插入 | 下一自动 ID 按冻结规则从目标值开始。 |
| AINC-005 | ALTER | 有数据提高起始值 | 已生成 ID 后改为更大值，插入多行 | 后续 ID 不低于目标值且无重复。 |
| AINC-006 | ALTER | 低于当前 counter | 已生成高 ID 后设置较低起始值 | 行为按产品契约（拒绝或不回退）稳定；绝不复用已有 ID。 |
| AINC-007 | ALTER | 等于当前/重复 ALTER | 多次设置相同或临界值 | 幂等/错误语义明确，counter 不意外跳变。 |
| AINC-008 | 元数据 | SHOW CREATE/SHOW TABLE STATUS | 创建/ALTER 后查看表定义与 next-id 信息（若暴露） | 起始值、cache 与相关 metadata 正确回显。 |
| AINC-009 | 约束 | 非 AUTO_INCREMENT 表/列 | 对无 auto 列、普通列或多个 auto 列执行 ALTER | 明确拒绝，无元数据残留。 |
| AINC-010 | 参数 | 非法起始值/cache | 0、负数、非整数、超范围及极大值 | 明确范围/类型错误；不改变原 counter。 |
| AINC-011 | DDL 原子性 | ALTER 与并发 DML | 一边 ALTER、一边 INSERT/SELECT | DDL/DML 遵循锁与原子性；无重复/半更新状态。 |
| AINC-012 | Schema 演进 | ADD/DROP/RENAME auto 列 | 受支持的 ALTER 组合后插入/SHOW | counter 依赖正确迁移或安全拒绝。 |

### 2.2 默认 ID 与显式插入（11 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| AINC-013 | 基础 | 单行自动生成 | 多次 `INSERT(v)` | ID 唯一，默认递增规则正确。 |
| AINC-014 | 批量 | 多行 VALUES/INSERT SELECT | 单语句插入多行、空结果 INSERT SELECT | 每个实际插入行获得唯一 ID；顺序/间隙符合分配契约。 |
| AINC-015 | 默认值 | NULL/DEFAULT/省略 id | 用三种写法插入 | 所有合法写法触发自动生成，语义一致。 |
| AINC-016 | 零值 | 显式 0 | 插入 `id=0`（受 SQL mode 影响时覆盖配置） | 行为符合 MatrixOne 冻结规则，不与自动 ID 混淆。 |
| AINC-017 | 手工值 | 低于当前 ID | 插入未占用低值后自动插入 | 手工行成功/失败按 PK 规则；counter 不错误回退。 |
| AINC-018 | 手工值 | 高于当前 ID | 插入大于 current 的显式 ID 后自动插入 | 后续 next-id 是否前移按产品契约验证；不得产生 duplicate key。 |
| AINC-019 | 冲突 | 已存在显式 ID | 插入重复主键、随后自动插入 | 正确报重复键；失败不消耗/污染 counter（按契约记录）。 |
| AINC-020 | 类型 | TINYINT/INT/BIGINT 有符号/无符号 | 各类型接近上限插入 | 边界生成与溢出错误正确；无回绕。 |
| AINC-021 | 删除 | DELETE/TRUNCATE 后生成 | 删除最高值、DELETE 全表、TRUNCATE 后重插 | DELETE 不复用 ID；TRUNCATE 的重置/不重置语义冻结并验证。 |
| AINC-022 | 多表 | 多 auto 表独立性 | 并发写入多个不同配置表 | counter/cache 完全隔离。 |
| AINC-023 | 查询 | LAST_INSERT_ID/返回值（支持时） | 单行/批量/失败插入后读取 | 返回本 session 正确生成 ID，不受其他 session 干扰。 |

### 2.3 Session increment 与 offset（10 条）

令 `inc = auto_increment_increment`、`off = auto_increment_offset`；对每个 session，自动生成值应满足产品定义的同余序列（常见兼容规则为 `id ≡ off (mod inc)`，起点与表 counter 共同决定首项）。

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| AINC-024 | 默认 | 默认变量值 | 新 session 查询变量并插入 | 默认值与兼容定义一致。 |
| AINC-025 | increment | inc=2/3/大值 | 设置 inc，连续/批量插入 | 序列步长与同余关系正确，ID 唯一。 |
| AINC-026 | offset | offset=1..inc | 固定 inc，切换合法 offset 插入 | 每个 session 返回满足其 offset 的序列。 |
| AINC-027 | 组合 | 不同 session 不同 inc/off | 两 session 交替/并发插入同表 | 所有 ID 全局唯一；各 session 的生成值符合各自设置。 |
| AINC-028 | 切换 | session 内动态 SET | 先默认插入，再 SET inc/off 并继续插入 | 切换仅影响后续分配；无重复或错误回退。 |
| AINC-029 | 隔离 | 连接池/重连 | 设置变量后重连、新开 session、复用连接 | session 生命周期和 reset 行为符合产品契约，无变量泄漏。 |
| AINC-030 | 参数 | 0/负数/非整数/offset>inc | 设置非法变量 | 明确错误，原 session 值保持不变。 |
| AINC-031 | DDL 交互 | ALTER 起始值 + inc/off | 多种 session 设置下 ALTER 后插入 | 每 session 从合法且不小于 counter 的下一个同余值分配。 |
| AINC-032 | 手工值交互 | 显式高 ID + inc/off | 手工插入后由多个 session 自动插入 | 不违反各 session 序列或全局唯一约束。 |
| AINC-033 | 多 CN | session 变量跨 CN | 同 session/不同 session 经不同 CN 写入 | 会话设置正确传递；不因路由改变失效。 |

### 2.4 Cache、并发与分布式（11 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- |
| AINC-034 | cache | cache=1 基线 | 高并发插入，记录分配与吞吐 | 全局唯一；作为最小 cache 性能/间隙基线。 |
| AINC-035 | cache | cache 较大 | 相同负载下使用大 cache | 全局唯一；缓存分配减少协调开销，允许未使用预留 ID 间隙。 |
| AINC-036 | cache | 不同表 cache 隔离 | 同时写入 cache 不同的多表 | 每表配置独立，不能借用/污染其他计数器。 |
| AINC-037 | cache | cache 耗尽/续租 | 单 CN 插入超过多个 cache 区间 | 新 cache 无重叠、无重复；边界无性能异常。 |
| AINC-038 | 多 CN | 多 CN 并发 INSERT | 多 CN、多 session 同表插入，收集所有 ID | 无重复；总行数=唯一 ID 数。 |
| AINC-039 | 多 CN | cache 分片 | 各 CN 获取 cache 后交替插入 | 分配区间不重叠；允许全局物理顺序交错。 |
| AINC-040 | 故障 | CN 重启/切换 | cache 已预留但未用尽时重启 CN，再写入 | 已发 ID 不复用；未用 cache 的 gap 符合持久化策略。 |
| AINC-041 | 故障 | TN/元数据短暂不可用 | 在分配/续租时故障注入后恢复 | 明确重试/错误；恢复后无重复或 counter 损坏。 |
| AINC-042 | 并发 | INSERT 与显式高值竞争 | 高并发自动插入期间插入高显式 ID | 无 duplicate key 风暴、死锁或不可恢复的 counter 跳变。 |
| AINC-043 | 热点 | 单表热点与多表分散 | 比较同总 QPS 下单表/多表写入 | 记录吞吐/延迟；热点不造成 CN 异常或无限重试。 |
| AINC-044 | 可观测性 | cache/counter 诊断 | 查看系统表、日志、metrics（实现提供时） | 可关联表、next-id、cache 区间、CN；敏感内部信息不泄露。 |

### 2.5 事务、恢复与兼容（9 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- |
| AINC-045 | 事务 | commit | 事务内插入多行后 COMMIT | 数据和已分配 ID 可见，next-id 正确前进。 |
| AINC-046 | 事务 | rollback | 插入后 ROLLBACK，再插入 | 回滚行不可见；ID 是否允许 gap 按冻结语义验证，绝不重复。 |
| AINC-047 | 事务 | savepoint/失败语句 | 部分失败、savepoint 回滚后继续写 | counter 和数据状态一致，无重复/卡死。 |
| AINC-048 | 备份恢复 | 逻辑备份/恢复 | 建带 cache/起始值表并写入，导出恢复 | schema 选项与 next-id 语义恢复，后续插入无冲突。 |
| AINC-049 | Snapshot/PITR | 多次 ALTER/DML 后恢复到不同时间点 | 每恢复点后插入并检查 ID | counter 与恢复点数据一致；不与现存行冲突。 |
| AINC-050 | 升级 | 旧 AUTO_INCREMENT 表 | 升级前创建的无新 option 表，升级后写入/SHOW | 默认行为兼容；metadata 能安全读取。 |
| AINC-051 | 复制/订阅 | CDC/订阅（支持时） | 写入 auto 表、恢复/消费目标端 | 主从/订阅语义不产生 ID 重写或冲突。 |
| AINC-052 | Rename/clone | RENAME、LIKE/CTAS | 重命名表，创建 LIKE/CTAS 后插入 | counter 归属与复制/重置规则明确且正确。 |
| AINC-053 | 权限 | SET/ALTER 权限 | 低权限用户执行 session SET、ALTER、写入 | 遵循权限模型；不能越权改变表级 counter。 |

### 2.6 异常、性能与稳定性（7 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果/记录项 |
| --- | --- | --- | --- | --- |
| AINC-054 | 错误恢复 | DDL/DML 中断 | 建表/ALTER/批量插入中注入故障并重试 | 无半成品 metadata、重复 ID 或资源泄漏。 |
| AINC-055 | 性能 | cache 大小矩阵 | cache=1/小/中/大，在同数据和并发下写入 | 记录 QPS、p50/p95、counter RPC、CPU/内存与 ID gap。 |
| AINC-056 | 性能 | inc/off 影响 | 默认与多个 inc/off 组合的多 CN 压测 | 正确性不变，记录协调/吞吐差异。 |
| AINC-057 | 长稳 | 单表热点 2 小时 | 多 CN 高并发自动/显式混合插入 | 无重复、CN 重启、计数器耗尽、内存增长或性能持续退化。 |
| AINC-058 | 长稳 | 多表混合 | 不同类型/cache/session 设置的表混合写 | 配置隔离；无跨表污染。 |
| AINC-059 | 边界 | 接近类型上限 | 多 CN 并发逼近整数上限并尝试续租 | 溢出前不重复，溢出后稳定报错且服务健康。 |
| AINC-060 | 回归 | 非 auto DML/既有表 | 执行普通 PK、sequence（如有）、旧 auto BVT | 无功能/性能回归。 |

## 3. 基线与判定

### 3.1 ID 正确性基线

每个并发/分布式测试保留完整写入记录，按以下规则判定：

1. `COUNT(*) = COUNT(DISTINCT id)`；失败即最高优先级正确性缺陷。
2. 所有自动生成 ID 都在对应表类型范围内；显式插入失败行不应以成功写入计数。
3. 设置 inc/off 的 session，其自动 ID 满足被冻结的序列公式；跨 session 不要求物理连续，但必须全局唯一。
4. cache、rollback、重启导致的 gap 必须在测试报告中关联到实际 cache 分配/事务事件；不能把重复或回退解释为 gap。

### 3.2 计划与元数据判定

| 场景 | 必须观察到 | 禁止观察到 |
| --- | --- | --- |
| ALTER 起始值 | 新定义/next-id 元数据原子生效 | 旧/新值交错、counter 回退或半更新。 |
| AUTO_ID_CACHE | 表级配置持久、分配区间不重叠 | 跨表共享 cache、无解释的重复 ID。 |
| Session 变量 | session 局部生效、跨 CN 传递正确 | 其他 session/连接池泄漏。 |
| 重启/恢复 | 已发 ID 不复用，恢复点与 counter 协调 | 与现有行冲突、counter 丢失或无界跳跃。 |

## 4. 结果记录要求

- 记录精确 MatrixOne commit/镜像、CN/TN 数量、协议版本、AUTO_ID_CACHE、session inc/off、表定义、数据规模、并发和随机 seed。
- 并发用例记录每个 writer 的 session、CN、事务结果、自动/显式 ID 集合，并提供 `COUNT` 与 `COUNT DISTINCT` 证据。
- cache/故障用例记录分配区间、实际 gap、重启/故障时间、系统日志或 metrics；未观察到内部指标时须明确说明。
- 性能至少 3 轮，记录 QPS、p50/p95、CPU/内存、counter RPC/等待（可得时）；仅同环境、同数据快照可比较。
- 缺陷附最小建表 SQL、变量设置、并发脚本/seed、完整 ID 集合、拓扑及错误日志。
