# MatrixOne 行列访问控制测试设计

> 本设计覆盖 Feature [#23271](https://github.com/matrixorigin/matrixone/issues/23271)：在内核支持基于角色的行/列访问控制。策略以 `(roleid, dbid, tableid) → query rewrite hint` 持久化；用户访问表时，优化器/编译器附加策略改写。多角色对同一表的规则仅在受控列集合一致时支持，并按 `rule1 UNION DISTINCT rule2` 合并可见行。

## 1. 测试原则

| 项目 | 说明 |
| --- | --- |
| 默认拒绝 | 无权限、无有效策略、策略加载失败或规则不兼容时必须 fail closed：不得返回未授权行/列。 |
| 原 SQL 语义保留 | 在授权可见范围内，行列权限改写不得改变 JOIN、聚合、排序、LIMIT、子查询等 SQL 原有语义。 |
| 策略不可绕过 | 所有访问同一物理表的路径均须应用策略：直接表扫描、视图、CTE、子查询、JOIN、UNION、prepared plan、存储过程/函数及导出入口（若支持）。 |
| 多角色规则 | 多角色规则只有列集合一致才允许合并；每个角色的行规则以 `UNION DISTINCT` 扩大可见行集合，且不因重叠角色规则产生重复行。 |
| 最小暴露 | 列权限必须阻断投影、谓词、排序、分组、表达式、通配符和元数据等可推断敏感列的路径。具体允许的谓词/错误策略须冻结。 |
| 元数据与缓存一致 | GRANT/REVOKE、角色切换、登录重连、策略 cache 失效后的授权结果必须及时且一致；旧 plan 不得继续使用已撤销权限。 |
| 审计可追溯 | 拒绝与成功改写都应产生足以定位 user、role、object、policy version 与拒绝原因的审计/诊断证据（避免记录敏感值）。 |

### 1.1 发布前必须冻结的安全契约

| 项目 | 需冻结内容 |
| --- | --- |
| 管理入口 | `GRANT ROWCOL PRIVILEGE ...`、REVOKE 或管理函数的最终 SQL 语法、权限要求和错误码。 |
| hint/rule 语法 | 可引用的列、允许表达式/函数、参数、子查询、常量与转义规则；禁止原始 SQL 注入式拼接。 |
| 行策略语义 | 规则是 allow-list、deny-list 还是 predicate；无规则/无匹配规则的结果；DML 是否支持。 |
| 列策略语义 | 允许列列表/掩码/拒绝方式；对 `SELECT *`、WHERE、ORDER BY、GROUP BY、表达式与 metadata 的行为。 |
| 多角色 | 何时判定“列相同”、角色启用集、`UNION DISTINCT` 的去重键与不兼容规则的错误。 |
| 生命周期 | 策略在登录、SET ROLE/ALTER ROLE、GRANT/REVOKE、plan cache、重启、备份恢复、复制中的生效点。 |

### 1.2 基本信息

| 项目 | 内容 |
| --- | --- |
| MatrixOne 基线 | Feature #23271 合入后的精确 commit、镜像、策略语法和部署拓扑（执行前冻结）。 |
| 元数据 | `mo_role_rowcol_priv`（或最终等价系统表）：保存 role、database、table 与 rewrite rule/hint。 |
| 管理对象 | role、user、database/table、row policy、column set、policy version。 |
| 规则加载 | 登录/角色改变时预加载，或 query compilation 时加载；实现可选，但可见结果和失效时序必须一致。 |
| 多角色规则 | 列集合一致时，行规则以 `UNION DISTINCT` 合并；列不一致为受控不支持路径。 |
| 主要风险 | 越权数据泄露、策略绕过、规则 SQL 注入、plan/cache 陈旧、角色组合漏判、行重复/漏行、DML 不一致。 |

### 1.3 执行矩阵

| 测试类型 | 单角色 | 多角色 | 通过标准 |
| --- | --- | --- | --- |
| 策略 DDL/元数据 | 执行 | 执行 | 定义、授权、撤销和系统 metadata 正确且安全。 |
| 行/列改写 | 执行 | 执行 | 仅返回授权行/列，结果与手工安全视图基线一致。 |
| SQL 绕过防护 | 执行 | 执行 | 所有表访问路径均施加同一有效策略。 |
| 多角色合并 | 不适用 | 执行 | 相同列集做 UNION DISTINCT；不兼容策略 fail closed。 |
| 缓存/生命周期 | 执行 | 执行 | 改权/切角色后旧权限不残留。 |
| 恢复/并发 | 执行 | 执行 | 策略、审计和授权在故障/多 CN 下保持一致。 |

### 1.4 覆盖范围

| 一级模块 | 用例数 | 主要覆盖内容 |
| --- | ---: | --- |
| 策略管理与元数据 | 11 | grant/revoke、语法、管理员权限、metadata、注入防护 |
| 单角色行列控制 | 13 | 行谓词、列集、SELECT、表达式、聚合、NULL、边界 |
| SQL 改写与绕过防护 | 12 | view、JOIN、CTE、子查询、UNION、prepared、导出/函数 |
| 多角色合并 | 9 | union distinct、重叠、列一致性、切换、冲突 |
| DML、缓存与生命周期 | 8 | write 语义、改权失效、登录、重启、事务 |
| 恢复、审计与性能 | 7 | backup/PITR、复制、多 CN、诊断、性能/长稳 |
| **合计** | **60** |  |

### 1.5 参考资料

| 资料 | 地址 |
| --- | --- |
| Feature | <https://github.com/matrixorigin/matrixone/issues/23271> |
| 多角色合并约定 | <https://github.com/matrixorigin/matrixone/issues/23271#issuecomment-4447258547> |
| 现有权限回归 | MatrixOne 用户/角色、对象权限、审计、视图、备份恢复和多 CN 测试。 |

## 2. 测试用例

通用模型：`sales(order_id bigint primary key, tenant_id int, region varchar(20), owner varchar(30), amount decimal(12,2), cost decimal(12,2), note varchar(200))`。基线表含多个 tenant/region/owner，含重叠行、NULL、相同金额和足够数据量。示例行策略为 `tenant_id = <allowed>`，列集合示例为 `{order_id, tenant_id, region, owner, amount}`；具体 hint 以最终冻结语法写入。

### 2.1 策略管理与元数据（11 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| RCAC-001 | 创建 | 为 role/table 创建行列策略 | 管理员用正式入口授予策略，检查系统表/SHOW | 策略持久化，role/db/table/rule/列集准确。 |
| RCAC-002 | 创建 | 同角色多表策略 | 对多个库表授予不同规则 | 表级隔离，访问每表只应用其自身策略。 |
| RCAC-003 | 更新 | 替换/重复授予 | 相同对象重复 GRANT，修改规则/列集 | 幂等或替换语义冻结；不得遗留多个冲突规则。 |
| RCAC-004 | 撤销 | REVOKE 单表/全部规则 | 授权后撤销，重连和旧 session 查询 | 立即/定义时点失效，不能继续访问旧范围。 |
| RCAC-005 | 权限 | 非管理员管理策略 | 普通用户/无对象权限用户 grant/revoke/read metadata | 严格拒绝，无越权写入或敏感规则泄漏。 |
| RCAC-006 | 参数 | 不存在 role/db/table | 对错误对象创建/撤销策略 | 清晰错误，系统表不留残留。 |
| RCAC-007 | 参数 | 非法 rule/hint | 不合法语法、未知列、非确定函数、子查询/注释/分号 | bind/validation 阶段拒绝，不能保存可注入内容。 |
| RCAC-008 | 参数 | 非法列集合 | 空集、重复列、敏感/不存在列、非法标识符 | 按契约拒绝或规范化；不扩大权限。 |
| RCAC-009 | 元数据 | SHOW/information_schema | 管理员、对象 owner、普通用户分别查询 | 可见性按权限控制；普通用户不可借 metadata 推断策略。 |
| RCAC-010 | DDL 依赖 | RENAME/DROP table/column | 策略存在时改名、删除对象 | 依赖同步更新或安全阻止/级联删除；不能保留指向错误对象的策略。 |
| RCAC-011 | 原子性 | 策略 DDL 失败/事务 | 并发 grant/revoke、失败注入、rollback（支持时） | 无半提交规则；查询只见完整旧或完整新策略。 |

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
| RCAC-023 | 事务可见性 | 事务内查询 | 策略下 INSERT/UPDATE 后本/他 session 查询 | 行可见性遵循隔离与策略，不出现跨 tenant 读。 |
| RCAC-024 | 对照 | 无策略/管理员 | 管理员或明确豁免角色查询 | 仅授权豁免主体按契约可全表；普通 role 绝不继承。 |

### 2.3 SQL 改写与绕过防护（12 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- |
| RCAC-025 | Explain | 改写证据 | EXPLAIN/ANALYZE 受控表查询 | 显示策略 filter/受限列或安全摘要；不泄露完整敏感 rule。 |
| RCAC-026 | JOIN | 受控表 join 未受控表 | sales 与维表 join，过滤/投影两侧列 | sales 策略在 join 前/等价安全位置生效，无 join 放大泄露。 |
| RCAC-027 | JOIN | 同一表自连接 | 受控表两个 alias 使用不同 join 条件 | 每个 scan/alias 都附策略，不能一侧绕过。 |
| RCAC-028 | 子查询 | IN/EXISTS/scalar subquery | 受控表出现在内外层 | 每个查询块正确改写，无跨层丢失条件。 |
| RCAC-029 | CTE/derived | CTE、多层派生表 | CTE 内/外引用受控表，重复引用 | 策略在原扫描点生效；物化/内联结果相同。 |
| RCAC-030 | UNION | UNION/UNION ALL/INTERSECT | 受控表出现在各分支 | 每个分支受控；集合运算后不出现未授权行。 |
| RCAC-031 | View | 普通/嵌套视图 | 通过 view 查询受控表，含 definer/invoker 模式（支持时） | 策略不可被 view 绕过；两种安全语义明确。 |
| RCAC-032 | Function | UDF/存储过程/table function | 通过可访问的函数间接读取表（支持时） | 执行身份与策略契约一致，不能提权读取。 |
| RCAC-033 | 导出 | COPY/SELECT INTO/stage | 对受控表导出或 CTAS（支持时） | 只导出可见行/列；目标表不会含隐藏数据。 |
| RCAC-034 | DML | INSERT/UPDATE/DELETE/MERGE | 对受控表执行写操作 | 若本期支持，写入/修改/删除仅允许策略行且检查新值；若不支持，明确拒绝而非绕过。 |
| RCAC-035 | 元路径 | SHOW/describe/system catalog | SHOW CREATE、DESC、统计/索引访问 | 不通过元数据或索引扫描暴露受限列/行信息。 |
| RCAC-036 | 优化回归 | predicate pushdown/index | 策略行 filter 与用户谓词、索引、JOIN reorder 组合 | 优化后等价于安全基线；不得因下推/重排丢策略。 |

### 2.4 多角色合并（9 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| RCAC-037 | 合并 | 两角色不重叠行 | R1 tenant=1、R2 tenant=2，列集合相同 | 用户同时拥有 R1/R2，结果为两集合并集。 |
| RCAC-038 | 合并 | 两角色重叠行 | 两规则重叠部分行 | 按 `UNION DISTINCT` 去重；无重复行/聚合双计。 |
| RCAC-039 | 合并 | 三个及以上角色 | 交叠/独有行规则组合 | 任意顺序结果相同，等于所有规则并集。 |
| RCAC-040 | 列一致 | 相同列集合不同顺序/别名 | 分别配置等价列集 | 按规范化集合判定一致并允许合并（若定义如此）。 |
| RCAC-041 | 列冲突 | 不同列集合 | R1/R2 对同表列集不同，查询/激活角色 | 明确拒绝或只允许单角色激活；绝不取并集扩大列权限。 |
| RCAC-042 | 行/列混合 | 相同列集、不同行规则 | 对 JOIN、聚合、分页重复测试 | 使用 UNION DISTINCT 的逻辑行集，再执行用户 SQL。 |
| RCAC-043 | 角色切换 | SET ROLE/ALTER ROLE | 同连接切换 R1、R2、R1+R2（支持时） | 立即生效；旧角色结果不缓存泄漏。 |
| RCAC-044 | 角色撤销 | 活跃 session 移除一个 role | 并发 revoke R2，重复查询 | 可见行按失效时序收缩，无旧 R2 行残留。 |
| RCAC-045 | 角色层级 | 继承/默认角色（支持时） | grant role、登录、启停角色 | 有效角色集与策略合并规则一致，不能隐式提权。 |

### 2.5 DML、缓存与生命周期（8 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| RCAC-046 | plan cache | 改策略后复用 prepared plan | 先执行/缓存，再 GRANT/REVOKE/更新 rule，重复执行 | 必须重新加载或使 plan 失效；撤销后不能继续读。 |
| RCAC-047 | 登录 cache | 登录时预加载策略 | 策略更新后已有/新登录 session 查询 | 按冻结刷新机制同步；不得无限陈旧。 |
| RCAC-048 | 多 CN cache | 不同 CN 编译/执行同 SQL | 一 CN 改策略/撤销，其他 CN 查询 | 所有 CN 在承诺时限内失效，期间 fail closed。 |
| RCAC-049 | 并发 | 改策略与查询并发 | 高频查询期间 grant/revoke/alter role | 每次查询见完整旧或新策略，无混合泄露。 |
| RCAC-050 | DML 事务 | 行策略与写入事务 | 跨 tenant INSERT/UPDATE，commit/rollback | 仅允许的写入最终生效；回滚无侧漏。 |
| RCAC-051 | 错误状态 | rule 加载/编译失败 | 注入 metadata/解析/caching 错误 | 查询拒绝并可诊断，不回退为无策略。 |
| RCAC-052 | 重启 | CN/TN 重启 | 策略存在、缓存已热后重启并查询 | 从持久 metadata 正确恢复，无临时豁免窗口。 |
| RCAC-053 | 对象变更 | DROP role/user 后访问 | 删除 role/user/策略引用对象 | 关联策略清理或失效，不能由 ID 复用继承权限。 |

### 2.6 恢复、审计与性能（7 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果/记录项 |
| --- | --- | --- | --- | --- |
| RCAC-054 | Backup/restore | 策略与受控数据备份恢复 | 恢复至新 account/db 后用各角色查询 | rule、角色映射、结果和默认拒绝状态完整恢复。 |
| RCAC-055 | PITR/复制 | 策略变更前后恢复/CDC（支持时） | 恢复多个时间点/复制目标查询 | 策略版本与该时点数据一致，不错配。 |
| RCAC-056 | 审计 | 允许/拒绝/改策略事件 | 执行代表性操作并检查日志 | 记录 actor、role、object、操作/结果、policy version；不记录敏感完整数据。 |
| RCAC-057 | 性能 | 单角色行过滤 | 高/中/低选择性规则对照管理员安全视图 | 结果正确；记录 p50/p95、扫描行、CPU、内存、策略编译/cache。 |
| RCAC-058 | 性能 | 多角色 UNION DISTINCT | 1/2/5/10 角色、重叠率矩阵 | 结果无重复；记录去重/内存/延迟，不可无界放大。 |
| RCAC-059 | 长稳 | 多 CN 查询+改权 | 多用户/角色持续读写及策略更新 | 无泄露、cache 增长、CN 重启或审计丢失。 |
| RCAC-060 | 回归 | 既有 RBAC/无策略表 | 执行对象权限、管理员、普通无策略表 BVT | 原有权限与查询性能不回归。 |

## 3. 安全基线与判定

### 3.1 结果 Oracle

每条受控查询都用管理员在独立基线中手工套用已冻结的行 predicate 和允许列集，生成安全 Oracle。比较完整行集合、列集合、聚合、排序和分页，而不只比较返回行数。

多角色 Oracle 为：先分别计算每条角色规则得到相同列集的行集合，进行 `UNION DISTINCT`，再执行用户 SQL 的其余逻辑。列集合不一致时，预期是冻结的拒绝结果，不能自行采用列集并集或交集。

### 3.2 必须 fail closed 的情形

| 情形 | 预期 |
| --- | --- |
| 无对象权限、无有效行列策略或列集不兼容 | 拒绝访问或只按最小权限返回；不得返回全表。 |
| 策略 metadata/解析/加载/缓存失败 | 拒绝该查询，提供安全错误。 |
| 不可安全改写的 SQL/函数/动态规则 | 拒绝或使用已验证的保守执行路径。 |
| 旧 plan、跨 CN cache、角色撤销未同步 | 在承诺失效点后不允许任何旧权限结果。 |

## 4. 结果记录要求

- 记录精确 MatrixOne commit、策略语法/版本、用户/角色拓扑、CN/TN 数、对象权限、表数据 seed 和每个 role 的预期行/列集合。
- 每条绕过防护用例附原 SQL、EXPLAIN、改写证据、管理员安全 Oracle 与受控实际结果；错误日志不得包含未授权数据。
- 多角色用例记录每条 rule 的行集合、列规范化结果、UNION DISTINCT 后 Oracle 和重复行计数。
- 缓存/撤销用例记录变更时间、session/CN、plan cache 状态、实际生效时间及是否出现任何越权结果。
- 安全缺陷附最小复现、user/role/object、策略版本、SQL、预期/实际泄露行或列，并按最高优先级处理。
