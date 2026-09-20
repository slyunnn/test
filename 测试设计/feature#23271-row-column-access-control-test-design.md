# MatrixOne 行列访问控制测试设计

> 本设计覆盖 Feature [#23271](https://github.com/matrixorigin/matrixone/issues/23271)：内核按角色保存并加载查询改写规则。当前交付入口为 `ALTER ROLE ... ADD/DROP RULE`，元数据为 `mo_catalog.mo_role_rule(role_id, rule_name, rule)`，其中 `rule_name` 为 `db.table` 名称。执行时必须锁定实际 MatrixOne `main` SHA；本文把 2026-09-20 的 Feature review 作为当前 Oracle 来源，不把旧 BVT `.result` 当作多角色语义的唯一依据。

## 1. 测试原则

| 项目 | 说明 |
| --- | --- |
| RBAC 与 rule 分离 | rule 是查询改写，不替代对象权限。无 SELECT 一律拒绝；有 SELECT 但没有 rule 时保留原 SQL/RBAC 行为；rule 解析、加载或改写失败必须 fail closed。 |
| 原 SQL 语义保留 | 在授权可见范围内，行列权限改写不得改变 JOIN、聚合、排序、LIMIT、子查询等 SQL 原有语义。 |
| 策略不可绕过 | 本期验收范围内的直接表扫描、view、CTE、子查询、JOIN、UNION 与 prepared plan 均须应用策略。函数、导出和 DML 见其独立的支持边界，不得把未覆盖入口写成已受保护。 |
| 多角色规则 | 仅完全兼容的 SELECT 形态、投影表达式列表及来源才合并。兼容时在基础行过滤层以 OR 扩大可见范围；同一基础行最多一次，不同基础行的相同投影不额外去重。当前不兼容规则采用后加载的有效规则覆盖，测试必须记录加载顺序；不得擅自把它写成“拒绝冲突”或“列集并集”。 |
| 最小暴露 | 列权限必须阻断投影、谓词、排序、分组、表达式、通配符和元数据等可推断敏感列的路径。具体允许的谓词/错误策略须冻结。 |
| 元数据与缓存一致 | ADD/DROP 当前只清理执行管理语句的 session rule cache；其他已热缓存 session 需以 `SET ROLE` 或重连刷新。多 CN 全局立即失效尚非已交付承诺，必须单列风险而不能按“已通过”验收。 |
| 审计可追溯 | 拒绝与成功改写都应产生足以定位 user、role、object、policy version 与拒绝原因的审计/诊断证据（避免记录敏感值）。 |

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

### 1.3 执行矩阵

| 测试类型 | 单角色 | 多角色 | 通过标准 |
| --- | --- | --- | --- |
| 策略 DDL/元数据 | 执行 | 执行 | 定义、授权、撤销和系统 metadata 正确且安全。 |
| 行/列改写 | 执行 | 执行 | 仅返回授权行/列，结果与手工安全视图基线一致。 |
| SQL 绕过防护 | 执行 | 执行 | 所有表访问路径均施加同一有效策略。 |
| 多角色合并 | 不适用 | 执行 | 相同来源和投影表达式列表的 rule 才按基础行 OR 合并；不同基础行的相同投影值保留。不兼容时记录后加载覆盖，绝不扩大列权限。 |
| 缓存/生命周期 | 执行 | 执行 | `SET ROLE` 或重连后的 session 不保留旧 rule；未刷新跨 session 结果单列风险。 |
| 恢复/并发 | 专项 | 专项 | 仅有对应环境直接证据时，才能声明策略、审计和授权在故障/多 CN 下保持一致。 |

### 1.4 覆盖范围

| 一级模块 | 用例数 | 主要覆盖内容 |
| --- | ---: | --- |
| 策略管理与元数据 | 11 | grant/revoke、语法、管理员权限、metadata、注入防护 |
| 单角色行列控制 | 13 | 行谓词、列集、SELECT、表达式、聚合、NULL、边界 |
| SQL 改写与绕过防护 | 12 | view、JOIN、CTE、子查询、UNION、prepared、导出/函数 |
| 多角色合并 | 9 | 基础行 OR 合并、重叠、列一致性、切换、冲突 |
| DML、缓存与生命周期 | 8 | DML 支持边界、改 rule 失效、登录、重启、事务 |
| 恢复、审计与性能 | 7 | backup/PITR、复制、多 CN、诊断、性能/长稳 |
| **合计** | **60** |  |

### 1.5 参考资料

| 资料 | 地址 |
| --- | --- |
| Feature | <https://github.com/matrixorigin/matrixone/issues/23271> |
| 多角色合并约定 | <https://github.com/matrixorigin/matrixone/issues/23271#issuecomment-4447258547> |
| 现有权限回归 | MatrixOne 用户/角色、对象权限、审计、视图、备份恢复和多 CN 测试。 |

## 2. 测试用例

通用模型：`sales(order_id bigint primary key, tenant_id int, region varchar(20), owner varchar(30), amount decimal(12,2), cost decimal(12,2), note varchar(200))`。每个 SQL case 必须写明 `CONNECT`、`SELECT`、默认/primary role、secondary role、`enable_remap_hint` 以及清理语句。多角色最小专用表为 `t(a int, marker int)`，seed 为 `(1,1),(1,2),(2,3)`；它避免 `order_id` 唯一而掩盖“基础行重复投影”语义。所有 case 另保留管理员手工 SQL Oracle。

### 2.1 策略管理与元数据（11 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| RCAC-001 | 创建/metadata | 管理入口与 SHOW | `ADD RULE` 后执行 `SHOW RULES ON ROLE` 及 `mo_role_rule` 对照；写明 SELECT/CONNECT 与 remap 开关 | rule_name 为 `db.table`，rule 文本和 role 正确持久化。 |
| RCAC-002 | 创建/隔离 | 同角色多表、双 account 同名对象 | 一角色两表各配置不同 rule；另在 account A/B 建同名 role/db/table、互斥 marker，分别 SHOW、查询、改 rule、重连 | 每表仅应用自身 rule；A 的读取、修改、缓存绝不影响 B。 |
| RCAC-003 | 更新 | 同表重复 ADD | 对同一 `db.table` 两次 ADD，第二次改变 predicate/投影，SHOW 与受控查询均核对 | 当前语义为替换；仅保留后规则。 |
| RCAC-004 | rule 与对象权限 | 四象限与删除 rule | 分别执行：有 SELECT 无 rule；有 rule 无 SELECT；有 SELECT 后 DROP RULE，B 已热缓存 session 在刷新前、`SET ROLE` 后、重连后查询；保留 rule 后撤销 SELECT | 无 rule 保持 RBAC 原 SQL；无 SELECT 始终拒绝；删除 rule 不等同撤销 SELECT；跨 session 刷新点按实测记录。 |
| RCAC-005 | 权限 | 非管理员管理策略 | 普通用户/无对象权限用户 grant/revoke/read metadata | 严格拒绝，无越权写入或敏感规则泄漏。 |
| RCAC-006 | 参数 | role/对象不存在 | 分别执行不存在 role、合法语法但不存在 db/table 的 ADD/DROP；检查 SHOW 与访问时行为 | 不存在 role 明确拒绝；对象绑定若创建时未做，记录残留 rule 和访问时安全失败，不能把“创建时拒绝”预设为当前 Oracle。 |
| RCAC-007 | 语法 | rule 合法性与注入 | 覆盖非法 SQL、非 SELECT-like、分号多语句、注释/转义文本、合法 SELECT 与合法子查询 | 非法/多语句拒绝且旧 rule 保持；合法子查询不因“像注入”而误拒绝。 |
| RCAC-008 | 绑定 | 不存在列与投影形态 | 空/重复投影、缺失列、函数/别名；分别检查 ADD 与受控访问 | 明确记录创建期或访问期绑定结果；访问期失败必须 fail closed、无越权结果。 |
| RCAC-009 | 元数据 | SHOW 与 catalog | 管理员、对象 owner、普通用户查询 `SHOW RULES`、`mo_role_rule` 与 information_schema | 仅授权主体看到必要 rule metadata；列/对象 metadata 不得绕过有效 rule。 |
| RCAC-010 | DDL 依赖 | rename/drop/旧名复用 | 有 rule 的表 RENAME、DROP、以旧名同 schema/异 schema 重建；每步 SHOW、受控查询和管理员 Oracle 对照 | rule 随对象安全更新、删除或 DDL 被阻止；重命名/旧名复用不得形成未过滤访问路径。 |
| RCAC-011 | 原子性 | 策略 DDL 失败/并发 | 执行失败 ADD/DROP、并发 ADD/DROP 与可用的 rollback；记录引擎是否允许 rule DDL 置于事务 | 不出现半提交 rule；查询只见完整旧或完整新策略。事务语义不由本 Feature 假设。 |

### 2.2 单角色行列控制（13 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| RCAC-012 | 行控制 | 基本 tenant predicate | role 查询 sales，比较管理员手工 `WHERE tenant_id=x` 基线 | 仅可见授权 tenant 行，列值无变化。 |
| RCAC-013 | 行控制 | 多条件/NULL predicate | region/owner/NULL 组合规则 | 遵循 SQL 三值逻辑；无 NULL 越权或漏行。 |
| RCAC-014 | 行控制 | 无匹配行 | 规则匹配零行 | 返回空结果，不能退化为全表。 |
| RCAC-015 | 列控制 | 显式受允许列 | 投影允许列和禁止的 `cost/note` | 允许列正确；禁止列明确拒绝或按冻结掩码规则处理。 |
| RCAC-016 | 列控制 | SELECT * | 同角色执行 `SELECT *`、`table.*` | 仅暴露允许列且列顺序/metadata 符合契约；不得包含敏感列。 |
| RCAC-017 | 列控制 | 表达式/别名 | 使用 `cost+amount`、CASE、函数、JSON/字符串函数引用禁止列 | 不得通过表达式、别名或隐式 cast 推断禁止列。 |
| RCAC-018 | 列控制 | WHERE/HAVING/ORDER/GROUP | 在各子句引用禁止列 | 依冻结规则拒绝或安全处理；不得以错误信息/结果泄露值。 |
| RCAC-019 | 聚合 | COUNT/SUM/MIN/MAX | 在可见行/允许列上聚合；试聚合禁止列 | 聚合只基于可见行；禁止列不能经聚合泄露。 |
| RCAC-020 | 分页 | LIMIT/OFFSET/ORDER | 策略行过滤后排序分页 | 先授权过滤再排序/分页，不能用 offset 绕过或推断隐藏行。 |
| RCAC-021 | DISTINCT | 重复/相同投影 | 在不同隐藏行投影相同允许值 | DISTINCT 只作用于授权行，计数不泄露隐藏行。 |
| RCAC-022 | 参数化 | prepared SQL | 参数化 predicate/投影，多次改值 | 所有执行均附策略，不因计划复用漏改写。 |
| RCAC-023 | 事务可见性 | SELECT rule 与事务读取 | 管理员在事务内 INSERT/UPDATE 并 commit/rollback，受控 session 只验证提交可见性与 SELECT rule；业务 DML 权限另按 RBAC 测 | 不将写入行策略列为本 Feature 能力；受控 SELECT 不出现跨 tenant 读。 |
| RCAC-024 | 对照 | 无策略/管理员 | 管理员或明确豁免角色查询 | 仅授权豁免主体按契约可全表；普通 role 绝不继承。 |

### 2.3 SQL 改写与绕过防护（12 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- |
| RCAC-025 | 启用/优先级/Explain | role、session、inline、remapdb | 受控用户分别切换 `enable_remap_hint`、`remap_rewrites`、inline `/*+ {\"rewrites\": ...} */`、跨库 remapdb；同一请求多语句逐句执行，并记录 EXPLAIN | 明确 role → session → inline 的实际优先级。任何普通用户可关闭角色 rule 且读到扩展行的结果为安全失败；EXPLAIN 仅要求稳定的安全摘要，不要求泄露完整 rule。 |
| RCAC-026 | JOIN | 受控表 join 未受控表 | sales 与维表 join，过滤/投影两侧列 | sales 策略在 join 前/等价安全位置生效，无 join 放大泄露。 |
| RCAC-027 | JOIN | 同一表自连接 | 受控表两个 alias 使用不同 join 条件 | 每个 scan/alias 都附策略，不能一侧绕过。 |
| RCAC-028 | 子查询 | IN/EXISTS/scalar subquery | 受控表出现在内外层 | 每个查询块正确改写，无跨层丢失条件。 |
| RCAC-029 | CTE/derived | CTE、多层派生表 | CTE 内/外引用受控表，重复引用 | 策略在原扫描点生效；物化/内联结果相同。 |
| RCAC-030 | UNION | UNION/UNION ALL/INTERSECT | 受控表出现在各分支 | 每个分支受控；集合运算后不出现未授权行。 |
| RCAC-031 | View | 普通/嵌套视图 | 通过普通及嵌套 view 查询受控表，记录 definer/invoker 实际语义 | 策略不可被 view 绕过；实际安全语义明确。 |
| RCAC-032 | Function | 函数间接访问的范围探测 | 本期不把 UDF/存储过程/table function 计为已交付 rule 入口；若接口可访问受控表，记录执行身份、拒绝或受限结果 | 不计入本 Feature 正向通过；任何可返回未过滤数据的路径为安全失败。 |
| RCAC-033 | 导出 | 导出/物化入口的范围探测 | 本期不把 COPY/SELECT INTO/stage/CTAS 计为已交付 rule 入口；对可用入口记录拒绝或受限结果 | 不计入本 Feature 正向通过；任何可导出隐藏行/列的路径为安全失败。 |
| RCAC-034 | DML | SELECT-rule 的写支持边界 | `ADD RULE` 仅接受 SELECT-like rule；对 INSERT/UPDATE/DELETE/MERGE 验证不能以 rule 定义写 predicate，业务 DML 仍按普通 RBAC 回归 | 本 Feature 不声明 DML 行策略；禁止把普通 RBAC 通过误报为 row/column write control。 |
| RCAC-035 | 元路径 | SHOW/describe/system catalog | SHOW CREATE、DESC、统计/索引访问 | 不通过元数据或索引扫描暴露受限列/行信息。 |
| RCAC-036 | 优化回归 | predicate pushdown/index | 策略行 filter 与用户谓词、索引、JOIN reorder 组合 | 优化后等价于安全基线；不得因下推/重排丢策略。 |

### 2.4 多角色合并（9 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| RCAC-037 | 合并 | 不同基础行的相同投影 | `t(a,marker)={(1,1),(1,2),(2,3)}`；R1=`marker=1`、R2=`marker=2`，均投影 `a`；显式激活两角色 | `SELECT a` 返回两个 `1`，`COUNT(*)=2`；只有 `SELECT DISTINCT a` 返回一个 `1`。先运行最小 case，不能沿用旧 `role_rule.result count=1`。 |
| RCAC-038 | 合并 | 两角色命中同一基础行 | R1/R2 同时命中 `(1,1)`，查询主键、普通投影及 count | 同一基础行只一次，来自 OR 过滤而非对结果做 DISTINCT。 |
| RCAC-039 | 合并 | 三个兼容角色 | 三个角色均使用相同来源和逐项相同投影；predicate 含重叠/独有行 | 基础行集合等于三个 predicate 的 OR；对同一明确的有效角色集结果确定。不可把不兼容规则的角色顺序等价性放入本 case。 |
| RCAC-040 | 兼容判定 | 表达式/来源精确匹配 | 分别配置不同列、同名列但表达式交换、列顺序/别名差异、相同逐行表达式、不同来源 | 仅实现定义的完全兼容形态合并；不能以“列名集合规范化”替代表达式/来源比较。 |
| RCAC-041 | 不兼容 | 不可合并形态覆盖 | 覆盖不同投影、ORDER BY/LIMIT、聚合、窗口等；改变 role 加载/启用顺序并记录 SHOW、活动角色和最终查询 | 当前实现按后加载的有效 rule 覆盖前者；不得取列集并集。若实现改为拒绝，才改为安全错误 Oracle。 |
| RCAC-042 | 混合语义 | 兼容合并后的 JOIN/聚合/分页 | 以 RCAC-037 的两条 `a=1` 基础行为 seed，对 JOIN、COUNT、LIMIT/OFFSET 和 DISTINCT 分别执行 | 原 SQL 语义在基础行 OR 后执行；普通投影/COUNT 保留两行，DISTINCT 才去重。 |
| RCAC-043 | 角色切换 | primary 与 secondary 生效集 | 分别断言 `SET ROLE r`、`SET SECONDARY ROLE ALL`、`SET SECONDARY ROLE NONE`，并断言 ALL 不改变 primary role | “拥有两个角色”不等于两者都生效；记录每个切换后的有效 rule 及结果。 |
| RCAC-044 | 角色撤销 | 活跃 session 移除一个 role | 并发 revoke R2，记录缓存前、`SET ROLE` 后和重连后的重复查询 | 按实测 session 刷新时序收缩；刷新后无旧 R2 行残留。 |
| RCAC-045 | 角色层级 | 继承/默认角色 | grant role、登录、启停角色 | 有效角色集与策略合并规则一致，不能隐式提权。 |

### 2.5 DML、缓存与生命周期（8 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| RCAC-046 | prepared/session cache | A 改 rule、B 热缓存 | B prepare/execute 并热缓存；A ADD/DROP/替换；依次验证 B 刷新前、`SET ROLE` 后、重连后、prepared 再执行 | 记录当前实现的 session cache 边界；只有刷新后的结果可作即时生效 Oracle。刷新前仍有旧结果须列为安全交付风险，不可误报全局已失效。 |
| RCAC-047 | 登录 cache | 新旧登录 session | 改 rule 后比较已有 session、`SET ROLE` 刷新的旧 session 与新连接 | 新连接及明确刷新后的 session 使用新 rule；记录旧 session 的实测边界。 |
| RCAC-048 | 多 CN cache | 跨 CN 已知限制 | 不同 CN 编译/执行；一 CN 改 rule，其他 CN 依次刷新/重连查询 | 当前标记为专项环境/待交付风险，不将“全 CN 立即失效”计为已支持。 |
| RCAC-049 | 并发 | 改 rule 与查询并发 | 高频查询期间 ADD/DROP/替换 rule，记录 session 是否刷新 | 每次查询见完整旧或新 rule；跨 session 未刷新旧结果单独标为已知限制，不能混入成功统计。 |
| RCAC-050 | DML 事务 | 普通 RBAC 写与 SELECT rule | 对跨 tenant INSERT/UPDATE 的允许/拒绝执行既有 RBAC Oracle；随后通过受控 SELECT 检查提交/回滚行可见性 | 不把普通 RBAC 写入误报为 rule 写控制；所有受控读取仍受 SELECT rule 限制。 |
| RCAC-051 | 错误状态 | rule 加载/编译失败 | 注入 metadata/解析/caching 错误 | 查询拒绝并可诊断，不回退为无策略。 |
| RCAC-052 | 重启 | CN/TN 重启 | 策略存在、缓存已热后重启并查询 | 从持久 metadata 正确恢复，无临时豁免窗口。 |
| RCAC-053 | 对象变更 | DROP role/user 后访问 | 删除 role/user/策略引用对象 | 关联策略清理或失效，不能由 ID 复用继承权限。 |

### 2.6 恢复、审计与性能（7 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果/记录项 |
| --- | --- | --- | --- | --- |
| RCAC-054 | Backup/restore | 专项恢复回归 | 不纳入本 Feature 的小数据验收；若恢复功能声明包含 role/rule metadata，则在 MOTR recovery 专项验证新 account/db 的 rule、角色与结果 | 不以快照/clone 代替逻辑恢复验证；未执行专项不能计为 Feature 通过。 |
| RCAC-055 | PITR/复制 | 专项 PITR/CDC 回归 | 不纳入本 Feature 的小数据验收；仅在 PITR/CDC 交付范围内按多个时间点/目标验证 metadata 与数据 | 策略版本与时点数据一致；无专项直接证据不得标通过。 |
| RCAC-056 | 审计 | 审计专项 | 不纳入本 Feature 的正向通过条件；仅在审计接口已交付时核对允许、拒绝、改 rule 的可用字段 | 记录 actor、role、object、结果与可用策略版本，不记录敏感完整数据。 |
| RCAC-057 | 性能 | 单角色行过滤 | 高/中/低选择性规则对照管理员安全视图 | 结果正确；记录 p50/p95、扫描行、CPU、内存、策略编译/cache。 |
| RCAC-058 | 性能 | 多角色基础行 OR 合并 | 1/2/5/10 角色、重叠率矩阵，含相同投影值的不同基础行。 | 基础行结果无重复；相同投影值按原 SQL 语义保留。记录谓词合并、内存/延迟，不可无界放大。 |
| RCAC-059 | 长稳 | 多 CN 查询+改 rule | 多用户/角色持续读取和 rule 更新 | 在专项环境记录泄露、cache 增长、CN 重启或审计丢失；普通单节点结果不能替代。 |
| RCAC-060 | 回归 | 既有 RBAC/无策略表 | 执行对象权限、管理员、普通无策略表 BVT | 原有权限与查询性能不回归。 |

## 3. 安全基线与判定

### 3.1 结果 Oracle

每条受控查询都用管理员在独立基线中手工套用已冻结的行 predicate 和允许列集，生成安全 Oracle。比较完整行集合、列集合、聚合、排序和分页，而不只比较返回行数。

多角色 Oracle 为：先按当前实现判断每个有效角色 rule 的 SELECT 形态、投影表达式列表和来源是否完全兼容。兼容时，管理员基线以 `WHERE (P1) OR (P2) ... OR (Pn)` 计算基础行可见集合；同一基础行最多一次。随后在该基础行集合上执行原用户 SQL 的投影、JOIN、聚合、排序、LIMIT/OFFSET 与集合运算。不同基础行产生的相同投影值必须保留，除非原用户 SQL 自身包含 `DISTINCT`、`UNION` 等去重语义。不兼容时，以当前“后加载有效 rule 覆盖”计算 Oracle，记录角色加载顺序；绝不采用列集并集。接口未来若改为拒绝冲突，需在变更提交中同步更新本 Oracle。

### 3.2 必须 fail closed 的情形

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
