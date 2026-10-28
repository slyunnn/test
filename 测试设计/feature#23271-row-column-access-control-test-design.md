# MatrixOne 行列访问控制测试设计（必要项评审版）

> 本设计覆盖 Feature [#23271](https://github.com/matrixorigin/matrixone/issues/23271)：内核按角色保存并加载查询改写规则。当前交付入口为 `ALTER ROLE ... ADD/DROP RULE`，元数据为 `mo_catalog.mo_role_rule(role_id, rule_name, rule)`，其中 `rule_name` 为 `db.table` 名称。执行时必须锁定实际 MatrixOne `main` SHA；本文把 2026-09-20 的 Feature review 作为当前 Oracle 来源，不把旧 BVT `.result` 当作多角色语义的唯一依据。

> 2026-10-28 必要项评审：原 60 条精简为 33 个执行单元；保留原编号以追溯 review，编号间隙不表示漏测。合并项的不同安全入口仍须分别给出结果，不把一条通过泛化到全部入口。保留安全目标与已交付行为的差距，不将已知限制当通过。

## 1. 测试原则

| 项目 | 说明 |
| --- | --- |
| RBAC 与 rule 分离 | rule 是查询改写，不替代对象权限。无 SELECT 一律拒绝；有 SELECT 但没有 rule 时保留原 SQL/RBAC 行为；rule 解析、加载或改写失败必须 fail closed。 |
| 原 SQL 语义保留 | 在授权可见范围内，行列权限改写不得改变 JOIN、聚合、排序、LIMIT、子查询等 SQL 原有语义。 |
| 策略不可绕过 | 本期验收范围内的直接表扫描、view、CTE、子查询、JOIN、UNION 与 prepared plan 均须应用策略。函数、导出和 DML 见其独立的支持边界，不得把未覆盖入口写成已受保护。 |
| 多角色规则 | 仅完全兼容的 SELECT 形态、投影表达式列表及来源才合并。兼容时在基础行过滤层以 OR 扩大可见范围；同一基础行最多一次，不同基础行的相同投影不额外去重。当前不兼容规则采用后加载的有效规则覆盖，测试必须记录加载顺序；不得擅自把它写成“拒绝冲突”或“列集并集”。 |
| 最小暴露 | 列权限必须阻断投影、谓词、排序、分组、表达式、通配符和元数据等可推断敏感列的路径。具体允许的谓词/错误策略须冻结。 |
| 元数据与缓存一致 | ADD/DROP 当前只清理执行管理语句的 session rule cache；其他已热缓存 session 需以 `SET ROLE` 或重连刷新。多 CN 全局立即失效尚非已交付承诺，必须单列风险而不能按“已通过”验收。 |
| 证据可追溯 | 保存 SQL、user/role/object、规则及错误证据，避免敏感值；不把尚未交付的审计字段或 policy version 当本期功能门禁。 |

### 1.1 发布前必须冻结的安全契约

| 项目 | 需冻结内容 |
| --- | --- |
| 管理入口 | `ALTER ROLE r ADD RULE 'select ...' ON TABLE db.t`、`ALTER ROLE r DROP RULE ON TABLE db.t`、`SHOW RULES ON ROLE r`；记录权限错误码与同表 ADD 替换语义。 |
| hint/rule 语法 | 区分 SELECT-like 语法校验、非法/多语句注入、合法 SELECT/子查询、合法语法但对象或列绑定失败；后者必须明确记录是在创建时还是访问时失败。 |
| 行策略语义 | 规则是查询改写 predicate；无 rule 时保留原 SQL。本期 rule 不声明 DML 行策略，写操作回归普通 RBAC，不能作为 row/column rule 通过证据。 |
| 列策略语义 | 允许列列表/掩码/拒绝方式；对 `SELECT *`、WHERE、ORDER BY、GROUP BY、表达式与 metadata 的行为。 |
| 多角色 | 完全相同的投影表达式列表和来源才兼容；`SET ROLE`、`SET SECONDARY ROLE ALL/NONE` 的生效集、兼容规则 OR、不同基础行重复投影保留，以及不兼容时覆盖的加载顺序。 |
| 生命周期 | 当前 session ADD/DROP、已热缓存 session 的 `SET ROLE`/重连、prepared 再执行的生效点；多 CN/重启/恢复需标明专项环境和未交付限制。 |

### 1.2 基本信息

| 项目 | 内容 |
| --- | --- |
| MatrixOne 基线 | Feature #23271 合入后的精确 commit、镜像、策略语法和部署拓扑（执行前冻结）。 |
| 元数据 | `mo_catalog.mo_role_rule(role_id, rule_name, rule)`；按 `db.table` 名称标识，DDL rename/drop 与旧名复用是直接风险面。 |
| 管理对象 | role、user、account、database/table、rule 与 session rule cache；对象 SELECT/CONNECT 授权必须在每个 case 明示。 |
| 启用前置 | 所有角色规则 E2E 均显式执行 `SET enable_remap_hint = 1`；记录 primary role、secondary role 状态和 session/inline remap。 |
| 多角色规则 | 相同来源及投影表达式列表时，行 predicate 在基础行层 OR；不同基础行相同投影保留。不可合并时当前实现按后加载有效规则覆盖，且不可假定不同角色顺序等价。 |
| 主要风险 | 越权数据泄露、策略绕过、规则 SQL 注入、plan/cache 陈旧、角色组合漏判、行重复/漏行、DML 不一致。 |

### 1.3 执行范围与分层

| 模块 | 必要执行单元 | 执行方式 |
| --- | ---: | --- |
| 策略管理与元数据 | 9 | SQL/BVT；两个 account 的同名对象隔离必须直接验证 |
| 单角色行列控制 | 7 | 小数据与管理员安全 Oracle 对照 |
| SQL 改写与绕过防护 | 8 | 每个真实扫描/间接读取入口独立断言 |
| 多角色合并 | 6 | 最小重复投影 fixture，记录活动角色及加载顺序 |
| 缓存、错误与对象生命周期 | 3 | 确定时序、多 CN；加载错误优先复用 UT |
| **合计** | **33** | 不全量重复单/多角色、单/多 CN 组合 |

### 1.4 精简说明

合并001/003/009、012/013/014/024、019/021、026/027、037/039、043/045及022/044/046/047/048；015承接035的元数据边界，011承接049的完整规则交错断言。023/034/050的通用写事务不属本期SELECT规则能力，只保留范围声明：业务DML遵循普通RBAC，不宣称行列写控制。保留032的可达间接读取风险于033。

删除独立重启、备份/PITR/CDC、审计能力、性能/长稳、全量既有RBAC重复回归；无规则与对象权限回归已由004覆盖。这些不是本Feature已通过能力，也不在本文留作隐藏的必跑附录。

### 1.5 参考资料

| 资料 | 地址 |
| --- | --- |
| Feature | <https://github.com/matrixorigin/matrixone/issues/23271> |
| 多角色合并约定 | <https://github.com/matrixorigin/matrixone/issues/23271#issuecomment-4447258547> |
| 现有权限回归 | MatrixOne 用户/角色、对象权限、审计、视图、备份恢复和多 CN 测试。 |

## 2. 测试用例

通用模型：`sales(order_id bigint primary key, tenant_id int, region varchar(20), owner varchar(30), amount decimal(12,2), cost decimal(12,2), note varchar(200))`。每个 SQL case 必须写明 `CONNECT`、`SELECT`、默认/primary role、secondary role、`enable_remap_hint` 以及清理语句。多角色最小专用表为 `t(a int, marker int)`，seed 为 `(1,1),(1,2),(2,3)`；它避免 `order_id` 唯一而掩盖“基础行重复投影”语义。所有 case 另保留管理员手工 SQL Oracle。

### 2.1 策略管理与元数据（9 个执行单元）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| RCAC-001 | 创建/metadata | ADD/替换/DROP/SHOW | 显式授权与启用 remap；ADD 后 SHOW/catalog 对照，同表第二次 ADD 改 predicate/投影，DROP 后再 DROP；对照管理员/无管理权用户读取 metadata | role_id、db.table 名称与文本持久化正确；同表 ADD 替换；DROP 不存在规则报错。管理/metadata 可见性符合冻结权限，失败不改变原 rule。 |
| RCAC-002 | 创建/隔离 | 同角色多表、双 account 同名对象 | 一角色两表各配置不同 rule；另在 account A/B 建同名 role/db/table、互斥 marker，分别 SHOW、查询、改 rule、重连 | 每表仅应用自身 rule；A 的读取、修改、缓存绝不影响 B。 |
| RCAC-004 | rule 与对象权限 | 四象限与删除 rule | 分别执行：有 SELECT 无 rule；有 rule 无 SELECT；有 SELECT 后 DROP RULE，B 已热缓存 session 在刷新前、`SET ROLE` 后、重连后查询；保留 rule 后撤销 SELECT | 无 rule 保持 RBAC 原 SQL；无 SELECT 始终拒绝；删除 rule 不等同撤销 SELECT；跨 session 刷新点按实测记录。 |
| RCAC-005 | 权限 | 无管理权限修改策略 | 普通用户、无对象权限用户分别尝试 ADD/DROP/替换规则，查询前后有效策略 | 未授权操作拒绝且原规则保留，不能越权写策略；只读 metadata 与 SHOW 另由001覆盖。 |
| RCAC-006 | 参数 | role/对象不存在 | 分别执行不存在 role、合法语法但不存在 db/table 的 ADD/DROP；检查 SHOW 与访问时行为 | 不存在 role 明确拒绝；对象绑定若创建时未做，记录残留 rule 和访问时安全失败，不能把“创建时拒绝”预设为当前 Oracle。 |
| RCAC-007 | 语法 | rule 合法性与注入 | 覆盖非法 SQL、非 SELECT-like、分号多语句、注释/转义文本、合法 SELECT 与合法子查询 | 非法/多语句拒绝且旧 rule 保持；合法子查询不因“像注入”而误拒绝。 |
| RCAC-008 | 绑定 | 不存在列与投影形态 | 空/重复投影、缺失列、函数/别名；分别检查 ADD 与受控访问 | 明确记录创建期或访问期绑定结果；访问期失败必须 fail closed、无越权结果。 |
| RCAC-010 | DDL 依赖 | rename/drop/旧名复用 | 有 rule 的表 RENAME、DROP、以旧名同 schema/异 schema 重建；每步 SHOW、受控查询和管理员 Oracle 对照 | rule 随对象安全更新、删除或 DDL 被阻止；重命名/旧名复用不得形成未过滤访问路径。 |
| RCAC-011 | 原子性 | 失败更新与查询交错 | 在确定时序中交错一次 ADD/替换/DROP 与查询；加入失败语句；事务仅测已确认支持的入口 | 无半提交或残缺 rule，每次编译使用完整规则；跨 session 旧缓存按046登记，不把陈旧结果算作新策略已生效。 |

### 2.2 单角色行列控制（7 个执行单元）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| RCAC-012 | 行控制 | predicate/NULL/空结果与无规则对照 | sales 上配置 tenant + region/owner/NULL 组合；另配零匹配规则，与管理员手工 SQL 及无规则 RBAC 查询比较 | SQL 三值逻辑正确；零匹配为空、不回退全表；完整可见行/列与 Oracle 一致。无规则/豁免身份不与规则加载失败混淆。 |
| RCAC-015 | 列控制 | 允许/禁止列与暴露边界 | 显式投影允许列、cost/note；检查返回 metadata，并对照 SHOW CREATE/DESC 的冻结支持边界 | 允许列值及类型正确；禁止列拒绝或按明确掩码契约处理。元数据是否隐藏与值访问分开记录，不能把未定义 metadata 隐藏能力写成通过。 |
| RCAC-016 | 列控制 | SELECT * | 同角色执行 `SELECT *`、`table.*` | 仅暴露允许列且列顺序/metadata 符合契约；不得包含敏感列。 |
| RCAC-017 | 列控制 | 表达式/别名 | 使用 `cost+amount`、CASE、函数、JSON/字符串函数引用禁止列 | 不得通过表达式、别名或隐式 cast 推断禁止列。 |
| RCAC-018 | 列控制 | WHERE/HAVING/ORDER/GROUP | 在各子句引用禁止列 | 依冻结规则拒绝或安全处理；不得以错误信息/结果泄露值。 |
| RCAC-019 | 聚合/DISTINCT | 授权行上聚合及去重 | 可见行执行 COUNT/SUM/MIN/MAX、DISTINCT；另尝试聚合禁止列 | 聚合/去重只针对已授权输入；隐藏行不进入结果，禁止列不经聚合泄露。多角色重复值由037/042作确定性判定。 |
| RCAC-020 | 分页 | LIMIT/OFFSET/ORDER | 策略行过滤后排序分页 | 先授权过滤再排序/分页，不能用 offset 绕过或推断隐藏行。 |

### 2.3 SQL 改写与绕过防护（8 个执行单元）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| RCAC-025 | 启用/优先级/Explain | role、session、inline、remapdb | 受控用户分别切换 `enable_remap_hint`、`remap_rewrites`、inline `/*+ {\"rewrites\": ...} */`、跨库 remapdb；同一请求多语句逐句执行，并记录 EXPLAIN | 明确 role → session → inline 的实际优先级。任何普通用户可关闭角色 rule 且读到扩展行的结果为安全失败；EXPLAIN 仅要求稳定的安全摘要，不要求泄露完整 rule。 |
| RCAC-026 | JOIN | 外表 JOIN 与自连接 alias | sales JOIN 维表；受控表以两个 alias 自连接，各自加谓词，与安全 Oracle 对照 | 每个 scan/alias 都受自身策略约束；无一侧漏改写、行数放大或受限列泄露。 |
| RCAC-028 | 子查询 | IN/EXISTS/scalar subquery | 受控表出现在内外层 | 每个查询块正确改写，无跨层丢失条件。 |
| RCAC-029 | CTE/derived | CTE、多层派生表 | CTE 内/外引用受控表，重复引用 | 策略在原扫描点生效；物化/内联结果相同。 |
| RCAC-030 | UNION | UNION/UNION ALL/INTERSECT | 受控表出现在各分支 | 每个分支受控；集合运算后不出现未授权行。 |
| RCAC-031 | View | 普通/嵌套视图 | 通过普通及嵌套 view 查询受控表，记录 definer/invoker 实际语义 | 策略不可被 view 绕过；实际安全语义明确。 |
| RCAC-033 | 间接读取边界 | 导出/物化/函数可达入口 | 仅列举目标版本实际可用、受控用户有权限的 SELECT INTO/CTAS/stage 或函数间接读取入口；分别记录执行身份、受限结果或拒绝 | 不把这些入口宣称为已交付正向能力；可达入口若返回/导出隐藏行列，记安全失败。不存在的接口不造测试，不展开通用函数与导出全矩阵。 |
| RCAC-036 | 优化回归 | predicate pushdown/index | 策略行 filter 与用户谓词、索引、JOIN reorder 组合 | 优化后等价于安全基线；不得因下推/重排丢策略。 |

### 2.4 多角色合并（6 个执行单元）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| RCAC-037 | 合并 | 两/三角色不同基础行相同投影 | t(a,marker)={(1,1),(1,2),(2,3)}；R1命中marker=1，R2命中marker=2，均投影a；显式激活。再加第三个相同来源/投影、含重叠与独有行的角色 | 两角色 SELECT a 返回两个1，COUNT(*)=2；DISTINCT才返回一个1；三兼容角色结果等于基础行predicate OR。不能沿用旧result的count=1；仅兼容组验证顺序不改变结果。 |
| RCAC-038 | 合并 | 两角色命中同一基础行 | R1/R2 同时命中 `(1,1)`，查询主键、普通投影及 count | 同一基础行只一次，来自 OR 过滤而非对结果做 DISTINCT。 |
| RCAC-040 | 兼容判定 | 表达式/来源精确匹配 | 分别配置不同列、同名列但表达式交换、列顺序/别名差异、相同逐行表达式、不同来源 | 仅实现定义的完全兼容形态合并；不能以“列名集合规范化”替代表达式/来源比较。 |
| RCAC-041 | 不兼容 | 不可合并形态覆盖 | 覆盖不同投影、ORDER BY/LIMIT、聚合、窗口等；改变 role 加载/启用顺序并记录 SHOW、活动角色和最终查询 | 当前实现按后加载的有效 rule 覆盖前者；不得取列集并集。若实现改为拒绝，才改为安全错误 Oracle。 |
| RCAC-042 | 混合语义 | 兼容合并后的 JOIN/聚合/分页 | 以 RCAC-037 的两条 `a=1` 基础行为 seed，对 JOIN、COUNT、LIMIT/OFFSET 和 DISTINCT 分别执行 | 原 SQL 语义在基础行 OR 后执行；普通投影/COUNT 保留两行，DISTINCT 才去重。 |
| RCAC-043 | 角色生效集 | primary/secondary与继承 | 分别 SET ROLE、SECONDARY ROLE ALL/NONE；以一次角色继承/默认登录作控制，记录每步活动角色和结果 | 拥有角色不等于激活；ALL不改变primary role；规则来自真实生效集，不隐式提权。 |

### 2.5 缓存、错误与对象生命周期（3 个执行单元）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| RCAC-046 | 缓存/撤销 | A改规则，B热缓存与prepared | A/B置于不同CN（单CN先跑控制）；B prepare/execute热缓存。A分别替换/DROP规则及撤销R2；记录刷新前、SET ROLE后先EXECUTE旧prepared并变参数、重新prepare、重连新建prepared各结果 | 新连接/明确刷新后使用新规则及生效角色；旧prepared及多次变参数执行不得绕过策略。当前仅清理管理session缓存，刷新前陈旧结果单列安全交付风险，不能声称跨session/CN即时撤销；不靠sleep掩盖生效点。 |
| RCAC-051 | 错误状态 | rule 加载/编译失败 | 注入 metadata/解析/caching 错误 | 查询拒绝并可诊断，不回退为无策略。 |
| RCAC-053 | 对象变更 | DROP role/user 后访问 | 删除 role/user/策略引用对象 | 关联策略清理或失效，不能由 ID 复用继承权限。 |

## 3. 安全基线与判定

### 3.1 结果 Oracle

每条受控查询都用管理员在独立基线中手工套用已冻结的行 predicate 和允许列集，生成安全 Oracle。比较完整行集合、列集合、聚合、排序和分页，而不只比较返回行数。

多角色 Oracle 为：先按当前实现判断每个有效角色 rule 的 SELECT 形态、投影表达式列表和来源是否完全兼容。兼容时，管理员基线以 `WHERE (P1) OR (P2) ... OR (Pn)` 计算基础行可见集合；同一基础行最多一次。随后在该基础行集合上执行原用户 SQL 的投影、JOIN、聚合、排序、LIMIT/OFFSET 与集合运算。不同基础行产生的相同投影值必须保留，除非原用户 SQL 自身包含 `DISTINCT`、`UNION` 等去重语义。不兼容时，以当前“后加载有效 rule 覆盖”计算 Oracle，记录角色加载顺序；绝不采用列集并集。接口未来若改为拒绝冲突，需在变更提交中同步更新本 Oracle。

### 3.2 权限与错误路径判定

| 情形 | 预期 |
| --- | --- |
| 无对象权限 | 由 RBAC 拒绝，与 rule 是否存在无关。 |
| 有对象 SELECT 但没有 rule / 删除 rule 后已刷新 session | 保留原 SQL，返回 RBAC 允许的数据；这不是 fail-closed 场景。 |
| 有 rule 但无对象 SELECT | RBAC 拒绝，rule 不授予对象访问。 |
| 列集/规则不可合并 | 按当前后加载规则覆盖；不得采用列集并集。 |
| 策略 metadata/解析/加载/缓存失败 | 拒绝该查询，提供安全错误。 |
| 不可安全改写的 SQL/函数/动态规则 | 拒绝或使用已验证的保守执行路径。 |
| 旧 plan、跨 CN cache、角色撤销未同步 | 当前只以 `SET ROLE` 或重连后的结果为生效 Oracle；刷新前的旧结果必须单独记录为交付风险，不能声称全局即时失效。 |

## 4. 结果记录要求

- 记录精确 MatrixOne commit、实际 rule SQL、`mo_role_rule` 元数据、用户/角色/account 拓扑、primary/secondary role 状态、`enable_remap_hint`/session/inline remap、CN/TN 数、对象权限、表数据 seed 和每个 role 的预期行/列集合。
- 每条绕过防护用例附原 SQL、EXPLAIN、改写证据、管理员安全 Oracle 与受控实际结果；错误日志不得包含未授权数据。
- 多角色用例记录每条 rule 的 predicate、基础行主键集合、投影表达式列表与来源、OR 基线、用户 SQL 结果及相同投影值对应的不同基础行数。不兼容时，额外记录有效角色顺序、覆盖行为和最终生效 rule。
- 缓存/撤销用例记录 A/B session、变更时间、session/CN、是否热缓存、`SET ROLE`/重连/prepare 再执行的每一步结果及实际生效时间；全局跨 session/CN 未承诺的状态不得标为通过。
- 安全缺陷附最小复现、user/role/object、策略版本、SQL、预期/实际泄露行或列，并按最高优先级处理。

## 5. 已有测试复用与验收边界

| 入口 | 本设计映射 | 使用约束 |
| --- | --- | --- |
| `test/distributed/cases/zz_accesscontrol/role_rule.sql` | 001/004/007/037/038/040/041/043/046 | 复用管理语法、授权、角色开关与数据搭建；按目标SHA重跑，不能照抄陈旧.result的多角色count=1。 |
| `pkg/frontend/rewrite_rule.go` 的规则验证、合并和缓存路径 | 006/008/011/025/040/041/046/051 | 为错误/交错/加载边界补直接证据；源码位置不是执行通过证明。 |

以[研发review](https://github.com/matrixorigin/matrixone/issues/23271#issuecomment-5747326189)及目标版本冻结契约判定。开关/覆盖规则可绕过、名称复用泄露、跨session即时撤销缺失须登记差距；不能仅因当前实现如此就宣布安全验收通过。本文只完成设计评审，未执行SQL/BVT。
