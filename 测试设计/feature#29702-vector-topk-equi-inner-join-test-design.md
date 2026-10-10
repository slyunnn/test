# MatrixOne 等值 INNER JOIN 下向量 Top-K 索引测试设计

> Feature [#29702](https://github.com/matrixorigin/matrixone/issues/29702)：普通等值 INNER JOIN 过滤向量表时支持 IVF-FLAT Top-K，同时保留 JOIN 重复次数、两侧投影、距离条件及最终分页。
>
> 规模判断：属于较大 feature。改写跨越候选资格生产、向量索引、原 INNER JOIN、最终 Top-K、列绑定及执行位置，错误可能直接改变结果行数或泄露不合格行，需要完整设计。编写日期：2026-10-10。交付基线为已合入 [PR #29767](https://github.com/matrixorigin/matrixone/pull/29767)，merge commit `a04c6fbfc9c57c2856c942ae640d766c42aef58d`。本文是测试设计，未将历史 PR 验证或本次文档检查记为本轮数据库测试通过。

## 1. 测试原则

| 项目 | 说明 |
| --- | --- |
| 保留多重集语义 | 同一向量行匹配 B 侧 m 行，最终仍贡献 m 行；资格域可去重，最终 INNER JOIN 不能被直接替换成 IN/SEMI。 |
| 资格在候选预算前生效 | 最近但无合法匹配的向量行不得耗尽候选预算；空资格域拒绝全部，不能当作无过滤。 |
| 分页分层 | 候选预算为 K+OFFSET，候选侧不消费 OFFSET；原 INNER JOIN 展开重复后再应用最终排序、OFFSET 和 LIMIT。 |
| 计划与结果双验证 | EXPLAIN 命中索引不等于结果正确；同时检查资格域、原 JOIN、最终分页、输出元数据与 SQL 结果。 |
| 精确与近似分开 | 单列表可穷举小样本用 FORCE/无索引 JOIN 作精确 oracle；多列表 ANN 测量召回，不新增全量精确召回承诺。 |
| 同分按 SQL 契约 | 只有距离排序时不要求同距离行固定顺序；不能为稳定测试额外加排序键后仍宣称覆盖本索引路径。 |
| 失败与回退完整 | 资格或插件能力不满足时保留旧计划及语义；不残留被试探修改的过滤、分页、绑定或索引保护状态。 |
| 资源与快照 | 资格 producer 和最终 JOIN 读取同一语句快照；域先完整发布再供 reader 使用，取消/提前结束不遗留资源。 |

### 1.1 基本信息与交付范围

| 项目 | 内容 |
| --- | --- |
| 执行基线 | 记录实际 commit、镜像、索引参数、配置、工具链及 CN/TN 拓扑；合入契约固定到上述 SHA，执行后续版本须核对新增变更。 |
| 正向形态 | 单个普通等值 INNER JOIN；两侧可安全重复读取的普通非分区表扫描；跨两侧同类型列的等值合取；单个受支持距离升序与 LIMIT。 |
| 向量条件 | 索引向量列已证明非 NULL（NOT NULL 或可识别的 IS NOT NULL）；查询向量可折叠成非 NULL 常量；向量表可位于 JOIN 任一侧。 |
| 首版索引/模式 | IVF-FLAT 既有 required-membership 能力；普通 LIMIT 与显式 `mode=pre`。本文精确样例使用 vecf32、L2 与 `vector_l2_ops`。其它距离/类型依现有插件能力逐项验证，不扩大承诺。 |
| FORCE | `mode=force` 明确选择精确扫描；不得解释为强制命中索引。 |
| 显式 AUTO/POST | 合入版本对此新等值 JOIN 形态保留原关系计划；普通 LIMIT 不等同于显式 `mode=auto`。其它模式不擅自改成 PRE。 |
| 成本范围调整 | Issue 原要求 cost-based；已批准 v1.1 方案 A 改为正确性/能力检查通过且索引可用时优先使用，完整 JOIN 成本模型后续实现。不能把“小表较慢”直接判为违背成本选择契约，也不承诺加速倍数。 |
| 执行位置 | 新复合索引路径首版保守 one-CN；多 CN 集群两入口都要验证完整依赖图的位置，不是宣称资格域跨 CN 分布式执行。 |
| 持久化/协议 | 本特性不新增 operator、wire 字段或 catalog 状态，不编造新的协议版本门槛。 |
| 范围排除 | 不新增 OUTER/ANTI、多级 JOIN 泛化、双侧最近对、相关查询向量、额外排序键及聚合/window 后 Top-K 能力；语义合法的这些查询保持既有正确路径。 |

### 1.2 执行矩阵

| 层级 | 环境/数据 | 证据与判定 |
| --- | --- | --- |
| 计划 UT | 精确合入基线、early/late 改写入口 | INNER 保留、候选预算、MustApply、绑定隔离、拒绝无修改、结果列元数据。 |
| SQL/BVT | 小型二维数据、lists=1、普通/PRE/FORCE | 独立 oracle、完整重复次数、距离谓词、分页；SQL 行数与执行成功同时检查。 |
| 集成 | 两 CN、两个 SQL 入口、实际物理计划 | 两入口语义一致；producer/reader/最终 JOIN 完整 one-CN placement 证据。 |
| 生命周期 | DML、PREPARE、取消、索引重建、重启 | 资格域按执行重建；不复用过期 snapshot、index generation 或消息。 |
| 业务形态 | VARCHAR PK、1536 维、普通及 gojieba 索引共存 | 重现 issue 查询形态；索引选择、列裁剪和重复语义不回归。 |
| 性能与召回 | 固定数据/seed/查询、不同选择性/重数/K/O | 对比普通/PRE/FORCE、合入前关系计划；大样本报告 recall 与延迟，阈值执行前约定。 |

### 1.3 覆盖范围

| 一级模块 | 用例数 | 编号 |
| --- | ---: | --- |
| 计划选择、绑定与元数据 | 10 | VJOIN-001..010 |
| JOIN 重复语义与分页 | 12 | VJOIN-011..022 |
| 资格过滤、距离与空输入 | 10 | VJOIN-023..032 |
| 保守回退与不支持边界 | 10 | VJOIN-033..042 |
| 生命周期、快照与多 CN | 12 | VJOIN-043..054 |
| 性能、召回与稳定性 | 6 | VJOIN-055..060 |
| **合计** | **60** | 每个参数组合独立记录执行结果。 |

### 1.4 参考资料

| 资料 | 地址 |
| --- | --- |
| 需求与历史反例 | [Issue #29702](https://github.com/matrixorigin/matrixone/issues/29702) |
| 实现 | [PR #29767](https://github.com/matrixorigin/matrixone/pull/29767/files) |
| 交付设计 | [合入版本 v1](https://github.com/matrixorigin/matrixone/blob/a04c6fbfc9c57c2856c942ae640d766c42aef58d/docs/design/CLAUDE_20261008-vector-equi-join-topk-v1.md) |
| 成本与范围澄清 | [合入版本 v1.1 方案 A](https://github.com/matrixorigin/matrixone/blob/a04c6fbfc9c57c2856c942ae640d766c42aef58d/docs/design/CLAUDE_20261008-vector-equi-join-topk-v1-1-review.md) |
| Planner UT | [apply_indices_equi_vector_test.go](https://github.com/matrixorigin/matrixone/blob/a04c6fbfc9c57c2856c942ae640d766c42aef58d/pkg/sql/plan/apply_indices_equi_vector_test.go) |
| SQL 与结果 | [vector_ivf_equi_join.sql](https://github.com/matrixorigin/matrixone/blob/a04c6fbfc9c57c2856c942ae640d766c42aef58d/test/distributed/cases/vector/vector_ivf_equi_join.sql)、[对应 result](https://github.com/matrixorigin/matrixone/blob/a04c6fbfc9c57c2856c942ae640d766c42aef58d/test/distributed/cases/vector/vector_ivf_equi_join.result) |

## 2. 数据准备与独立基线

### 2.1 最小确定性数据

测试使用独占数据库并保存/恢复 `experimental_ivf_index` 等会话设置；以下数据每个变更用例重新装载。二维坐标取二进制可精确表示的小数，避免用浮点误差解释距离边界失败。业务 1536 维复核与大规模召回另设数据集。

```sql
SET @saved_experimental_ivf_index = @@experimental_ivf_index;
SET experimental_ivf_index = 1;
CREATE TABLE chunks (
    id VARCHAR(32) PRIMARY KEY,
    v VECF32(2) NOT NULL,
    document_id VARCHAR(32),
    category VARCHAR(32)
);
INSERT INTO chunks VALUES
('c0','[0,0]','u','x'),
('c1','[0.0625,0]','u','x'),
('c2','[0.125,0]','d1','x'),
('c3','[0.25,0]','d2','z'),
('c4','[0.375,0]','d1','y'),
('c5','[0.75,0]','d3','outside'),
('c6','[0.1875,0]',NULL,'null'),
('c7','[0.5,0]','d4','new');
CREATE INDEX chunks_ivf USING ivfflat ON chunks(v)
    lists=1 op_type 'vector_l2_ops';
CREATE TABLE documents (
    row_id INT PRIMARY KEY,
    document_id VARCHAR(32),
    label VARCHAR(32)
);
INSERT INTO documents VALUES
(1,'d1','x'),(2,'d1','y'),(3,'d2','z'),(4,'d3','outside'),(5,NULL,'null');
CREATE TABLE unique_documents (
    document_id VARCHAR(32) PRIMARY KEY,
    label VARCHAR(32)
);
INSERT INTO unique_documents VALUES ('d1','x'),('d2','z'),('d3','outside');
```

标准查询使用同一个非 NULL 常量向量：

```sql
SELECT a.id, b.row_id, b.label, l2_distance(a.v,'[0,0]') AS distance
FROM chunks a INNER JOIN documents b ON a.document_id=b.document_id
WHERE l2_distance(a.v,'[0,0]') <= 0.5
ORDER BY l2_distance(a.v,'[0,0]') ASC
LIMIT 3;
```

同一 SQL 分别采用普通 LIMIT、尾部添加 `BY RANK WITH OPTION 'mode=pre'`、`BY RANK WITH OPTION 'mode=force'`；OFFSET 放在 rank option 前。保留没有向量索引的数据副本作为第二关系 oracle，不能用 IN 查询作重复 JOIN 的 oracle。确认 lists=1 的数据和实际探测覆盖；不能只设置一个未被执行器使用的用户变量就宣称全探测。

### 2.2 手算结果

| 查询变体 | 精确预期 |
| --- | --- |
| 重复键 JOIN、阈值0.5、K=3 | id依次c2、c2、c3；c2对应row_id集合为1、2，同距离内部顺序不限。 |
| 同上K=10 | c2、c2、c3、c4、c4，共5行；c0/c1最近但无匹配，c6的NULL不相等，c7无匹配，c5超阈值。 |
| 同上K=3、OFFSET=1 | id依次c2、c3、c4；边界c2/c4各保留任一合法B匹配，不硬编码label选择。 |
| 同上K=10、距离<=0.25 | c2、c2、c3；改为<0.25只返回两行c2。 |
| 唯一键 JOIN、阈值0.5、K=3 | c2、c3、c4，各一行。 |
| IN/membership对照、阈值0.5、K=3 | c2、c3、c4；用于证明与重复JOIN不同，不能要求二者结果一致。 |
| B过滤label='x'、K=10 | (c2,1,x)、(c4,1,x)，只保留过滤后匹配。 |
| 多等值条件再加a.category=b.label、阈值0.5、K=10 | (c2,1,x)、(c3,3,z)、(c4,2,y)。 |

不同距离严格升序，同距离按多重集比较；只有窗口截断同分组时允许多个合法子集。新增数据、修改过滤条件后必须重新计算oracle，不复用此表旧结果。

## 3. 测试用例

### 3.1 计划选择、绑定与元数据（10 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| VJOIN-001 | 默认选择 | 普通LIMIT | 唯一/重复B键，二维fixture执行EXPLAIN和SELECT | 均有IVF向量路径；原INNER与最终分页保留，结果符合§2。不能因成本模型延后允许正向fixture任意不命中。 |
| VJOIN-002 | PRE | 显式资格预过滤 | 001改为mode=pre，检查required domain与物理计划 | 命中索引；MustApply资格域在候选预算前生效；不是全局ANN取K后才JOIN。 |
| VJOIN-003 | FORCE | 精确对照 | 同SQL使用mode=force | 无向量索引Top-K扫描，保持精确关系JOIN/排序，重复次数和最终窗口正确。 |
| VJOIN-004 | 模式区分 | 显式POST/AUTO | 同SQL分别mode=post/auto，保留输入选项检查 | 新等值JOIN规则不应用，保留原关系计划；结果等于精确oracle。不得偷偷改成PRE，也不得把普通LIMIT当显式AUTO。 |
| VJOIN-005 | JOIN方向 | 向量表写在右侧 | 交换FROM两侧及等号左右，仍投影两侧列 | 两方向均命中，结果与元数据等价；识别基于绑定而非表的文本位置。 |
| VJOIN-006 | 等值合取 | 多键且存在重复 | 加a.category=b.label及第二个同类型等值键变体 | 完整合取进入资格域和原JOIN，匹配输出符合全部键；不能只拿document_id过滤。 |
| VJOIN-007 | 投影 | 别名/两侧表达式 | 距离AS别名排序，投影A/B列及确定性组合表达式；UT增加透明PROJECT | 索引仍可用；B值来自对应匹配行，A所需JOIN键未裁掉；上层混合表达式不被错误提前执行。 |
| VJOIN-008 | 客户端元数据 | 列来源与标记 | 比较索引/FORCE结果列顺序、名称、原始列名、类型、nullable/primary标记；增加非向量JOIN与中间PROJECT控制 | A主键来源保留；B.label保持自身nullable；计算距离无伪造原始列名或PK标记。共享列来源解析修改不能使普通查询元数据回归；须取协议/plan元数据，不能只看文本值。 |
| VJOIN-009 | 两个改写入口 | early/late一致 | UT分别进入logical早期和late插件路径，K=2/O=3 | 候选预算5且无候选OFFSET，最终K=2/O=3；INNER/ON/原B节点不变，完整可达图无可变B子树共享。 |
| VJOIN-010 | 试探隔离 | 插件拒绝/普通索引保护 | 保存原树及tag映射，构造能力拒绝；成功/拒绝后检查普通索引路径 | 失败不改变原filter、rank、分页、绑定；保护正常释放，普通索引不被永久禁用；错误不被吞成成功。 |

### 3.2 JOIN 重复语义与分页（12 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| VJOIN-011 | 唯一键 | B主键/唯一约束 | unique_documents及等价唯一约束表，K=1/3/10 | 一A最多匹配一B，结果c2/c3/c4；不要求必须消除原JOIN，首版可统一保留INNER。 |
| VJOIN-012 | 重复键 | 两个B匹配 | documents，K=3/10，同时查询chunk-only和两侧列 | K=3是c2,c2,c3，K=10阈值内5行；行身份用(A.id,B.row_id)，不以COUNT DISTINCT A.id验收。 |
| VJOIN-013 | 高重数 | 单A重数大于K+O | 最近合格A匹配100条B，K=10/O=5 | 最终10行都可来自同A且各B匹配合法；不能把候选预算误当输出行预算，不能丢重复。 |
| VJOIN-014 | 多对多 | 同document多A多B | 多个不同PK的chunk共用document，B重数为1/2/5混合 | 各A贡献m(a)份，低距离A的重复先占最终窗口；按完整JOIN oracle对账。 |
| VJOIN-015 | OFFSET | 跨重复组切片 | fixture阈值0.5，K=3/O=1；再覆盖组内/组间offset | 首组id为c2,c3,c4；候选K+O，O只消费一次且在展开后。边界同分匹配按§4.1验收。 |
| VJOIN-016 | LIMIT | 小于/等于/大于总行数 | K=1/5/10，O=0，小样本全探测 | 行数min(K,精确合格JOIN行数)，K=1为任一合法c2匹配；不能补入无资格行。 |
| VJOIN-017 | LIMIT零 | 静态/参数零 | LIMIT0及PREPARE的0→3→0 | 返回0且正常结束，不能等待未启动producer；后续非零查询可完成。 |
| VJOIN-018 | 大OFFSET | 越界与预算算术 | O大于JOIN总行数；UT测试K+O溢出/未绑定O | 越界为空；无法安全构造预算时保守不改写，不回绕成小预算。合法SQL原结果/原错误保留。 |
| VJOIN-019 | PREPARE | 固定向量、参数LIMIT | 固定字面向量与LIMIT ?，执行0/1/4/0并修改B后再执行 | 参数预算按次更新，结果列元数据稳定，资格域重新生产；不复用旧结果。 |
| VJOIN-020 | 同分 | 不同A等距离及重复B | 造对称向量与窗口截断tie group | 距离顺序与可接受同分窗口正确；不要求固定PK顺序。校验器不能给被测SQL增加第二排序键。 |
| VJOIN-021 | 错误改写控制 | JOIN对比IN | fixture分别执行原INNER和IN形式 | JOIN K3为c2,c2,c3，IN为c2,c3,c4；明确捕捉错误去重，不将IN当正确性基线。 |
| VJOIN-022 | NULL/未匹配键 | 两侧NULL及孤立键 | 加NULL键、无匹配键，保持普通=连接 | NULL不与NULL匹配；c6/c7不输出，c0/c1最近但仍不占合法结果；不把NULL-safe equality混作普通=。 |

### 3.3 资格过滤、距离与空输入（10 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| VJOIN-023 | A过滤 | 最近行被本地条件排除 | 在A侧加category过滤，构造超过K个更近但被排除行 | 本地谓词进入资格生产与正确残余路径；完全探测小样本返回正确远端合格行，不被近端无效候选耗尽。 |
| VJOIN-024 | B过滤 | 匹配存在但不满足label | label='x'及全部被排除两组 | 前者为c2/c4各一行，后者为空；资格按过滤后B判断，原JOIN不能展开被排除的B行。 |
| VJOIN-025 | 组合资格 | A/B本地过滤+多键 | 组合023/024与等值合取，并加入多重数 | 候选资格等价完整JOIN匹配；任何遗漏谓词造成的多行/少行都判失败。 |
| VJOIN-026 | 距离边界 | <、<=、等号、范围 | 用精确坐标在0.25上下构造行，K大于结果总数 | <=含c3，<排除c3；所有返回行满足源向量距离谓词；DistRange只编码可安全表达部分，残余谓词保留。 |
| VJOIN-027 | 不同距离表达式 | WHERE向量与排序向量不同 | WHERE使用q1，ORDER使用q2；再测非可编码距离条件 | 不能把q1范围错误绑定到q2索引距离；安全保留残余或回退，结果与全探测oracle一致，不强求命中。 |
| VJOIN-028 | NULL索引向量 | 可空列与显式排除 | 可空向量表含NULL；先不加过滤，再加v IS NOT NULL | 未证明非NULL时保留旧计划及NULL排序语义；显式排除后正向fixture可用索引，不漏非NULL合法行。 |
| VJOIN-029 | 查询向量 | NULL/未绑定/可折叠常量 | 参数向量在NULL/非NULL间切换；字面/常量折叠对照 | 未证明非NULL常量时不改写，NULL距离排序不误变空域；不能承诺所有向量参数都命中。结果行数和SQL错误遵循旧路径。 |
| VJOIN-030 | 空输入 | A空/B空/交集空 | 三类独立fixture，普通/PRE/FORCE | 全部返回0且无挂起；空required domain必须拒绝全部，不当作PASS。 |
| VJOIN-031 | 稀疏资格 | 结果少于K及近邻全不合格 | lists=1，造大量近端不匹配、少量远端匹配，K超过合格展开行数 | 返回所有精确合格行及正确重数，不返回无资格行；该精确结论不直接扩展到多列表近似探测。 |
| VJOIN-032 | 索引控制 | 缺索引/错误距离算子 | 同数据无索引副本；仅不兼容距离索引的表 | 保留正确关系执行，无向量索引误用；返回同SQL精确结果或原有非法距离/维度错误。 |

### 3.4 保守回退与不支持边界（10 条）

回退检查区分“本新规则不应用”与“全查询不能出现任何索引”。其它既有合法优化仍可生效；对专用小fixture可明确断言没有 Vector Index Scan，不能把断言无条件推广到复杂查询全部子树。

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| VJOIN-033 | JOIN类型 | LEFT/RIGHT/OUTER/ANTI | 在原引擎支持语法内执行各类型控制 | 不强行套本规则；保留未匹配行/反连接语义，和无索引同查询一致。 |
| VJOIN-034 | 非等值 | <、范围与混合条件 | document比较及等值+跨侧不等式 | 无充分资格证明时不改写；最终过滤不得从原树丢失。 |
| VJOIN-035 | 键证明 | OR、表达式键、类型转换 | OR连接、函数键、不同类型隐式转换、NULL-safe条件 | 仅符合既有类型/等值证明才应用；不满足的typed plan保留，结果/错误与原计划一致。 |
| VJOIN-036 | 数据来源 | 多级JOIN/分页子查询/分区 | 构造不能归约成两普通扫描的来源、子计划LIMIT/OFFSET | 本规则保守回退，不移动内层分页；安全化简后若确实符合条件则按最终typed plan判断。 |
| VJOIN-037 | 排序限制 | DESC或额外排序键 | 距离DESC、距离加B.label/A.id二级排序 | 不套单键升序预算证明，保留旧计划；次级排序和窗口正确。 |
| VJOIN-038 | 无LIMIT/计数语义 | 无LIMIT、SQL_CALC_FOUND_ROWS | 原SQL去LIMIT，或加found-rows语义 | 不用候选截断替代全量结果/计数；返回行与计数符合精确基线。 |
| VJOIN-039 | 关系算子 | DISTINCT/GROUP/window | 构造算子位于JOIN与Top-K之间的查询 | 不绕过去重/聚合/窗口语义；原逻辑结果保持，不将m(a)>=1证明用于变化后的结果空间。 |
| VJOIN-040 | 不可复制表达式 | volatile/副作用 | 投影RAND、原引擎支持的序列副作用及不可安全重复来源 | 拒绝新子树复制；UT确认不额外求值/消耗副作用，随机值不按两次执行逐值相等验收。 |
| VJOIN-041 | 参数与非法输入 | 动态offset/维度错误 | UT未绑定offset；SQL非法K/O、查询向量维度不匹配 | 保留原校验/错误或安全回退，不产生伪成功/预算溢出；支持的参数LIMIT仍按019验收。 |
| VJOIN-042 | 插件/既有能力 | 无membership能力与旧路径 | HNSW等能力拒绝UT；无JOIN、IN/SEMI、ON1=1向量provider、既有标量查询向量各作控制 | 无等价能力不误接本路径；已有受支持路径不回归。本需求不扩大HNSW/GPU或provider能力。 |

### 3.5 生命周期、快照与多 CN（12 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果 |
| --- | --- | --- | --- | --- |
| VJOIN-043 | B变更 | 资格/重数动态变化 | PREPARE后新增/删除重复B、d1改u并提交、再改回 | 每次新执行按可见B重建资格与重数；不会保留旧域，FORCE在同数据版本作对照。 |
| VJOIN-044 | A变更 | 向量/键/存活性 | INSERT、DELETE、UPDATE向量和document_id，提交后重查 | 候选和最终JOIN遵循当前可见数据；删除行不复活，键变化同步影响资格，结果符合相同索引可见性契约。 |
| VJOIN-045 | 事务快照 | B双读间并发提交 | 用确定性执行屏障暂停资格发布前后，另一事务更改B并提交；同时覆盖回滚 | producer与最终B读取遵循同一语句快照，不能资格来自旧B而展开来自新B；下一语句按其快照可见。禁止sleep碰时序。 |
| VJOIN-046 | 权限/租户 | producer与最终读取一致 | 不同租户/账户同名表、撤销B SELECT、已有权限过滤控制 | 权限错误保留；不因内部副本绕过访问控制或混入其它租户PK。只验证现有权限机制，不扩建RLS功能。 |
| VJOIN-047 | 取消 | producer/reader/最终JOIN阶段 | 生命周期fixture在各阶段阻塞并取消，随后执行正常查询 | 上下游正常退出，取消不伪装空成功，无domain/reader/goroutine持续泄漏；后续请求不被旧消息污染。 |
| VJOIN-048 | 域发布失败 | 缺失/未seal/构建失败 | 复用required-domain故障注入，确认reader不能先读；另测空域 | 完整域发布后才读；构建错误明确传播，不能按无过滤继续扫描，空域为0结果。 |
| VJOIN-049 | 提前结束 | consumer LIMIT终止 | 高重数、K=1、重复执行并检查清理 | 最终达到LIMIT后释放原JOIN及资格依赖；不遗留发送者、阻塞或跨次域；正常完成与取消分开计数。 |
| VJOIN-050 | PREPARE依赖 | DROP/重建索引与generation | 固定向量prepare，drop索引后execute，再重建execute；变更表定义控制 | 依既有依赖规则重规划/明确失效，不能访问旧对象；无索引时正确回退，有新索引时使用有效generation，列元数据准确。 |
| VJOIN-051 | 重启/冷加载 | 同版本恢复 | 建表索引后停服重启，两入口首次查询与预热后查询 | 持久索引加载正确，资格域按语句重建，无需新增catalog状态；精确fixture与FORCE一致。 |
| VJOIN-052 | 两CN部署 | 双入口one-CN计划 | 从CN-A、CN-B各执行唯一/重复、OFFSET、反向JOIN，留逻辑与物理计划 | 两入口均命中预期索引且结果正确；完整资格producer/consumer在受支持one-CN依赖域内，不能只查看scan的local标记。 |
| VJOIN-053 | 索引共存 | 普通covering/gojieba | 1536维表保留(document_id,seq_prefix,seq)、(document_id,start_time)及gojieba索引，执行普通/PRE/FORCE | document_id及输出列未被过早裁剪；向量与普通索引保护正确释放，全文查询控制不回归。不要求未使用的全文索引参与向量查询。 |
| VJOIN-054 | 资源边界 | 大域/大O/大重数 | 逐级放大E、K+O和匹配重数，设置受控资源限额，记录溢写/错误 | 无按P*D盲目预分配导致失控；原有溢写/资源错误正常传播，释放资源，不靠少返回行掩盖耗尽。 |

### 3.6 性能、召回与稳定性（6 条）

| 编号 | 二级模块 | 测试场景 | 测试步骤 | 预期结果/记录项 |
| --- | --- | --- | --- | --- |
| VJOIN-055 | 版本/模式对比 | 同快照关系基线 | 合入前精确关系计划、合入后FORCE、普通、PRE，核对每种实际计划 | 精确fixture结果等价；报告QPS、p50/p95/p99、CPU/峰值内存、读行/字节与producer成本。POST/AUTO可作回退控制，不包装成索引加速。 |
| VJOIN-056 | 选择性 | 资格与距离相关性 | 合格比例100%/10%/1%/0.1%，随机分布及“最近全不合格”两种；规模递增 | 报告域大小、候选/探测、返回行数和recall；小样本全探测不得因不合格行耗尽预算。大样本ANN不足额按召回与预算证据分析。 |
| VJOIN-057 | 重复/O成本 | B双读和展开 | B重数1/2/10/100，K=1/10/100，O=0/100/10000；规模受控 | 记录producer、B再次读取、回表与展开成本及最终窗口；承认小表/大重数可能慢，不规定所有组合必须快于FORCE。 |
| VJOIN-058 | 业务复核 | 1536维VARCHAR主键 | 先复现120行、重复document_id和NULL的小fixture；再扩展业务规模、lists=10及不同探测配置 | 计划支持且重复语义正确；120行不作为生产性能证据。大样本报告召回，不以单次与FORCE相同宣称精确ANN。 |
| VJOIN-059 | ANN基准 | ANN-Benchmarks数据+关系过滤 | 固定L2数据集/版本/seed与查询，外加document映射、eligibility和重复B，执行join-aware驱动 | 原版无JOIN benchmark只作向量检索参考；关系扩展重新计算JOIN ground truth，报告多重集召回、资格泄漏率、满足率及延迟，定义见§4.3。 |
| VJOIN-060 | 长稳 | 查询/DML/取消混合 | 受控并发持续2小时，交替普通/PRE/FORCE和PREPARE，独立静止检查点核对结果 | 无崩溃/死锁/持续资源增长，取消无残留；报告错误分类与延迟趋势。并发不同快照不直接逐行比较为同一oracle。 |

## 4. 基线与判定

### 4.1 关系语义与同分窗口

独立校验器从同一固定快照导出A/B，按完整ON和WHERE生成匹配对，以 `(A.id,B.row_id)` 标识输出实例，计算源向量L2并排序后取最终窗口。只输出A.id时仍按频数比较，不能去重。

- 精确小样本：无同分边界时比较完整有序结果；同分组完整落入窗口时比较该组多重集。
- 窗口切过同分组时，按oracle距离分组计算区间 `[O,O+K)` 在各组应占行数，检查被测结果在每组的合法匹配实例、数量和非递减距离；不限定组内B.row_id或等距离A的选择。
- 浮点比较使用与列类型相符的容差，边界fixture优先精确坐标；不能用过大容差吞掉WHERE越界。非法维度/NULL依原SQL契约处理。
- 对不同快照的两次查询不作强等价比较。修改数据后重新生成oracle，或用明确相同事务快照/静止检查点。

### 4.2 计划与时序判定

| 检查对象 | 必须证明 | 不能替代的证据 |
| --- | --- | --- |
| 正向默认/PRE | Vector Index Scan/实际IVF访问、required eligibility、原INNER与最终分页 | 仅搜索EXPLAIN里出现“index”不够。 |
| 候选预算 | K=2/O=3时候选5、候选无OFFSET、外层仍K2/O3 | 最终恰好2行不能证明OFFSET没有提前消费。 |
| JOIN语义 | ON、B投影、匹配重数不丢，资格域只去重A主键 | IN结果不等于重复JOIN结果。 |
| 域发布 | seal→发布required domain→reader消费；MustApply，不允许空域PASS | 无错误返回不能证明完整资格已参与候选。 |
| 子树隔离 | 复制A/B重绑后注册，原B/tag保持，拒绝无部分修改 | 结果偶然相同不能证明绑定无共享污染。 |
| one-CN | 最终物理依赖图完整且域可见范围一致 | 仅ForceOneCN flag或“查询从某CN发起”不足。 |
| 元数据 | 源列、计算列、nullable/primary及输出顺序与FORCE一致 | 文本结果一致不能替代客户端元数据检查。 |

原INNER和最终Sort/Limit存在是设计需要，不套用“命中Top-K索引就必须消除所有Sort/JOIN”的验收规则。

### 4.3 ANN、性能与召回指标

大样本至少覆盖10万、100万向量（按资源与数据集可用性执行并记录实际规模），查询集、数据映射和随机seed固定。精确基线保留原JOIN多重集及距离谓词，不能用未过滤的全局ANN ground truth，也不能只用去重后的chunk集合。

1. 主召回指标用JOIN行实例多重集：`recall = sum(min(actual_count(x), exact_count(x))) / exact_window_size`，x为(A.id,B.row_id)。分母是精确窗口真实行数，少于K时不仍除K；精确窗口为空时记N/A并独立验收实际为空。
2. 同分切片使用§4.1允许的等距离组配额计算可接受交集，避免任意tie顺序造成假召回损失；另报去重chunk recall仅作辅助，不能掩盖重数丢失。
3. 资格/WHERE泄漏率必须为0，ANN近似不允许输出不合法JOIN行。记录返回满足率 `actual_rows/exact_window_size`（非空窗口）和实际行数；近似探测下不足额不单凭少行定性为JOIN错误，结合全探测控制、域和候选证据归因。
4. 普通/PRE与FORCE比较速度时同时给召回、探测参数及资源预算；更快但召回不同的结果不称为等质量加速。约定召回目标后，再比较达到同一目标的延迟/QPS。
5. 至少3轮独立测量，统一硬件、快照、索引构建参数、查询并发、投影宽度和缓存状态，分别记录冷/热查询。记录实际probe、lists、索引构建时间/大小及索引内存，不以一次耗时下结论。
6. ANN-Benchmarks扩展驱动须明确新增JOIN逻辑和自定义ground truth，不能把官方无JOIN得分直接当本feature性能。完整成本模型尚未交付；性能阈值在执行前约定，未约定时只交测量与回退建议，不自行宣称性能验收通过。

## 5. 已有测试复用与执行顺序

| 设计用例 | 已核对入口 | 使用方式与缺口 |
| --- | --- | --- |
| 001、002、005..007、052 | `TestEquiVectorTopKPublicPlan` | 唯一/重复键、左右两侧、别名、B过滤、多等值键、OFFSET、逻辑placement；052仍需真实双入口物理计划。 |
| 003、004、029、033..040 | `TestEquiVectorTopKConservativeBoundaries` | FORCE/POST/AUTO、DESC、多排序键、无LIMIT、LEFT、非等值、volatile、NULL、found_rows等拒绝控制；不宣称覆盖全部复杂组合。 |
| 009、015、052 | `TestEquiVectorTopKPreservesJoinAndPagination` | early/late、K+O=5、INNER/ON/原B/tag保留、MustApply及私有树隔离。 |
| 010、028、029、036、041、042 | `TestEquiVectorTopKEligibilityRejectsWithoutMutation` | 无PK/可空向量/未绑定向量、HNSW、分页、分区、runtime filter、溢出、类型不符等typed拒绝及原树无修改。 |
| 007、008、019 | `TestEquiVectorTopKMetadataMatchesForce`、`TestEquiVectorTopKPreparedLimit` | 列来源/标记与参数LIMIT填充；补真实客户端协议和重复执行结果。 |
| 011..032、043 | `vector_ivf_equi_join.sql`及`.result` | 复用重复与IN对照、阈值、OFFSET、空输入、NULL、PREPARE及B更新；本设计坐标调整后必须独立生成/审阅预期，不能直接复制旧result。 |
| 045、047..050、054 | 既有IVF required-domain/cancellation/prepared-generation及vectorscan生命周期/race测试 | 执行时定位实际文件/测试名，并记录与新复合JOIN场景的直接关联；缺少直接覆盖则补场景，不能引用PR历史“已通过”代替。 |
| 055..060 | 新的JOIN基准驱动和结果记录 | PR历史小样本/双CN比较不代表生产规模性能；这些是拟执行项，本设计未声称已测量。 |

建议顺序：先复现最小fixture与模式控制 → owning planner/compile及元数据验证 → SQL结果与BVT（首次生成后人工核对，再正常比较两轮并确认清理）→ 多CN/取消/快照/依赖生命周期 → 大规模召回与性能。执行测试需按仓库工具链与CGo规则，不能复用未验证版本的旧服务二进制。

PR还包含独立CDC清理同步修复；它不构成本feature验收范围，不因为同在一个PR就扩展为CDC测试设计。

## 6. 结果记录与完成标准

- 每个用例记录状态（未执行/通过/失败/阻塞）、实际SQL/脚本、commit/镜像、计划、结果和oracle来源；UT/BVT历史报告、本轮执行和仅静态核对分列。
- 保留A/B数据、唯一键约束、重复分布、NULL比例、过滤选择性、向量维度/类型、距离/op_type、lists/probe、K/O、模式、seed及两侧投影；没有真实plan不能声称走了索引。
- 正确性失败记录完整(A.id,B.row_id)频数及距离、边界同分组、资格域/候选/最终结果，区分非法行、少重数、分页错误和ANN召回损失。
- 多CN记录入口、producer/reader/最终JOIN的physical placement；事务测试记录快照/提交顺序与同步屏障，取消测试记录资源回收和后续请求。
- 性能记录各轮原始数据、p50/p95/p99、QPS、CPU/内存、域规模、扫描/回表/展开成本、错误/取消率、recall和满足率；未提供的内部指标标为不可观测，不编造数值。
- 发布验收要求正向SQL语义、保守回退、重复分页、权限/快照、one-CN及清理路径有直接证据；性能和召回按约定目标评估。未执行项保持未执行，不用“生成设计/表格渲染正常”替代数据库测试结论。
- 清理本次独占数据库/索引及测试连接，在保存设置的同一连接恢复 `SET experimental_ivf_index = @saved_experimental_ivf_index`；数据库测试连续运行时先确认上轮清理完成，不删除共享业务对象。
