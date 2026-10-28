# MatrixOne 等值 INNER JOIN 下向量 Top-K 索引测试设计（必要项评审版）

> Feature [#29702](https://github.com/matrixorigin/matrixone/issues/29702)：普通等值 INNER JOIN 过滤向量表时支持 IVF-FLAT Top-K，同时保留 JOIN 重复次数、两侧投影、距离条件及最终分页。
>
> 规模判断：属于较大 feature。改写跨越候选资格生产、向量索引、原 INNER JOIN、最终 Top-K、列绑定及执行位置，错误可能直接改变结果行数或泄露不合格行，需要完整设计。编写日期：2026-10-10。交付基线为已合入 [PR #29767](https://github.com/matrixorigin/matrixone/pull/29767)，merge commit `a04c6fbfc9c57c2856c942ae640d766c42aef58d`。本文是测试设计，未将历史 PR 验证或本次文档检查记为本轮数据库测试通过。

> 2026-10-28评审：原60条精简为 **25个执行单元**，保留原编号。按JOIN多重集、资格预算、分页/绑定、保守回退和直接执行生命周期筛选；相同fixture上的控制/断言合并。取消性能、ANN-Benchmarks、长稳、服务重启和完整权限矩阵。

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
| 生命周期 | PREPARE、B变更、语句快照、取消及一次索引依赖失效 | 资格域按次执行重建；保留直接依赖风险，取消通用DML/重启矩阵。 |
| 列裁剪 | 原VARCHAR PK fixture增加一个普通covering索引 | 验证JOIN所需列不被提前剪掉；不增加1536维/全文索引组合。 |

### 1.3 覆盖范围与裁剪

| 模块 | 执行单元 | 原编号 |
| --- | ---: | --- |
| 计划选择、投影与绑定 | 7 | 001、003、005、006、007、009、010 |
| 多重集与分页 | 5 | 012、013、015、018、019 |
| 资格、距离及空域 | 5 | 025、026、027、028、030 |
| 必要回退 | 3 | 033、037、040 |
| 执行生命周期与placement | 5 | 045、047、050、052、053 |
| **合计** | **25** | 每行一个执行单元，合并断言必须执行。 |

主要合并：002→001，004→003，008→007，011/014/021/022→012，016/020→015，017/043→019，023/024→025，029→028，031→030，032→001精确控制，034/035/036→033资格UT，038→037，042→010插件控制，048/049→047。041的动态参数安全性归018/028，通用非法维度不另测。

删除独立项：039完整DISTINCT/GROUP/window矩阵、044普通向量DML矩阵、046完整权限/租户矩阵、051重启、054资源压力、055..060性能/业务规模/ANN benchmark/长稳。053只保留直接相关covering列裁剪，不展开gojieba与1536维组合。

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

测试使用独占数据库并保存/恢复 `experimental_ivf_index` 等会话设置；以下数据每个变更用例重新装载。二维坐标取二进制可精确表示的小数，避免用浮点误差解释距离边界失败。必要验收使用此二维小数据；不新增业务维度或大规模召回数据集。

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

## 3. 必要执行项

### 3.1 计划选择、投影与绑定（7项）

| 编号 | 场景与必要断言 |
| --- | --- |
| VJOIN-001 | 普通LIMIT与显式PRE各跑唯一/重复B键；计划有IVF、完整资格域、原INNER和最终分页，配合同数据无索引精确控制。两种入口共用fixture，不另设002/032执行项。 |
| VJOIN-003 | FORCE无向量Top-K索引且精确；显式POST/AUTO按本feature合入基线保留原关系路径，不偷偷改PRE。普通LIMIT不得误当显式AUTO。 |
| VJOIN-005 | 向量表位于JOIN右侧、等号左右交换，仍输出两侧列；基于绑定识别，结果/元数据不随文本位置变化。 |
| VJOIN-006 | 多等值键且B重复，资格与原JOIN都保留完整合取；不能只拿一个document_id键筛选。 |
| VJOIN-007 | 距离别名、A/B确定性投影及中间PROJECT；结果值和列来源/nullable/PK/计算列标记与FORCE一致，必要列不误裁剪。共享列来源解析增加一条普通JOIN控制，合并原008。 |
| VJOIN-009 | early/late改写UT，K=2/O=3；候选5且无候选OFFSET，原K/O及INNER/ON/B保留，候选/producer私有树与tag不共享。 |
| VJOIN-010 | 插件能力拒绝及未服务索引控制，原filter/rank/分页/tag不被部分修改，普通索引保护释放；既有SEMI/provider路径复用回归，不扩新索引算法能力。 |

### 3.2 多重集与分页（5项）

| 编号 | 场景与必要断言 |
| --- | --- |
| VJOIN-012 | 同fixture比较唯一B与重复B、chunk-only与两侧投影；K3为c2,c2,c3、K10阈值内5行，包含同document多A匹配。IN仅作反例，不作oracle；NULL与未匹配键不输出。 |
| VJOIN-013 | 最近合格A匹配重数大于K+O，取K10/O5；输出可全来自同A，但每个(A.id,B.row_id)合法，不能提前去重或按候选行数截断重复。 |
| VJOIN-015 | K3/O1跨重复组得c2,c3,c4，少于/恰好/多于K共用数据；同分切片按配额验收，再加一组不同A同距离控制，不给SQL加二级排序。 |
| VJOIN-018 | O超过总行数为空；UT验证K+O溢出/未绑定O保守回退，不回绕预算，不扩通用非法参数矩阵。 |
| VJOIN-019 | 静态LIMIT0与固定向量LIMIT ?执行0→3→0共用fixture；再修改B资格/重复次数、改回后复执行。每次重建域/预算、元数据稳定，两种零页均不挂起；吸收017/043，无需另设全部DML组合。 |

### 3.3 资格、距离及空域（5项）

| 编号 | 场景与必要断言 |
| --- | --- |
| VJOIN-025 | A本地条件、B.label过滤分别控制后合并，与多键/重复一起验；近端不合格行不能耗尽预算，资格producer和最终JOIN不漏完整过滤。 |
| VJOIN-026 | 小fixture在0.25做<与<=，K大于合格数；含边界得c2,c2,c3，不含仅c2两行，正确保留source距离/residual。 |
| VJOIN-027 | WHERE距离用q1、排序用q2的一条反例；保留正确残余或回退，不能错把q1范围绑定到q2索引距离，不强求命中。 |
| VJOIN-028 | 可空索引向量未证明非NULL时不改写，IS NOT NULL后可用；查询向量NULL/未绑定及可折叠常量对照，NULL排序不被当空ANN域。 |
| VJOIN-030 | A空、B空、交集空及少于K，在同fixture变体中验证；空required域拒绝全部，远端少量合格行全探测返回正确重数，不误补无资格行。 |

### 3.4 必要回退（3项）

| 编号 | 场景与必要断言 |
| --- | --- |
| VJOIN-033 | 一条LEFT和一条非等值SQL控制；资格UT复用跨侧表达式/类型不一致、非普通扫描及子计划分页等拒绝分支。原关系语义与分页不变，不扩OUTER/ANTI/CTE全排列。 |
| VJOIN-037 | DESC、额外排序键、无LIMIT及found-rows语义采用已有定向UT/小SQL；不把单键升序候选证明套到其它结果空间。 |
| VJOIN-040 | 一条volatile/副作用控制，拒绝复制危险表达式；用求值次数/sequence状态证明，不逐值比较两次RAND结果。 |

本规则拒绝不等于全查询所有子树禁止索引；按最终typed plan判断。未修改的复杂聚合/window语法不新增独立矩阵。

### 3.5 执行生命周期与placement（5项）

| 编号 | 场景与必要断言 |
| --- | --- |
| VJOIN-045 | 用确定性屏障在资格producer与原B再次读取之间提交B更新；同语句快照一致，下一语句按其快照可见。不能用sleep碰时序。 |
| VJOIN-047 | 直接生命周期fixture合并缺域/构建失败、一个代表性阶段取消，以及K=1正常提前结束；不按无过滤继续、不伪成功，域/reader/worker清理，后续请求正常。各路径需分别触发，复用同fixture。 |
| VJOIN-050 | prepare后一次DROP/重建索引依赖控制；重规划或明确失效，不访问旧对象/generation，列元数据正确。 |
| VJOIN-052 | 两CN入口只抽测唯一/重复、OFFSET、反向JOIN；保留物理依赖图，证明producer/consumer/原JOIN在受支持one-CN域内，非仅scan local标记。 |
| VJOIN-053 | 小VARCHAR fixture增加普通covering索引，执行普通/PRE；JOIN键和输出列不被过早裁掉，保护成功/失败后释放。无需大维度/全文索引组合。 |

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

### 4.3 验收限度

必要SQL验收固定lists=1及确认全探测的小fixture，校验完整JOIN多重集、距离与最终窗口。多列表ANN仍不承诺全量精确召回，不能用ANN误差解释过滤泄漏或丢重复；本版不安排大样本召回测量、性能比较或ANN-Benchmarks。

完整JOIN成本模型在合入设计中已延后，性能收益不作为本次必要测试项。合入基线以后若模式/能力变化，先核对新契约再执行，不能把其它feature的模式矩阵直接套入本规则。

## 5. 已有测试复用与执行顺序

| 保留项 | 已核对入口与证据 |
| --- | --- |
| 001/005/006/007/052 | `TestEquiVectorTopKPublicPlan`；两入口physical placement另保留实机证据。 |
| 003/028/033/037/040 | `TestEquiVectorTopKConservativeBoundaries`，仅保留相关拒绝控制。 |
| 009/015 | `TestEquiVectorTopKPreservesJoinAndPagination`，early/late及K+O、私有树、原INNER。 |
| 010/018/028/033 | `TestEquiVectorTopKEligibilityRejectsWithoutMutation`，能力、NULL/动态参数、overflow及typed拒绝后原树完整。 |
| 007/019 | `TestEquiVectorTopKMetadataMatchesForce`、`TestEquiVectorTopKPreparedLimit`。 |
| 012/013/015/019/025/026/027/028/030/053 | `vector_ivf_equi_join.sql`及result复用相关fixture，新增断言独立算预期，不直接复制错误result。 |
| 045/047/050 | 定位当前ref已有required-domain/cancellation/prepared依赖测试，补新复合JOIN直接场景或明确覆盖缺口；不以历史“通过”替代。 |

顺序：计划UT与小fixture oracle → 必要SQL/BVT → 空域/失败/取消/快照/依赖 → 两CN及covering抽测。BVT新预期人工核对后正常comparison并确认teardown。执行遵循验收ref工具链/CGo规则。

PR内独立CDC清理修复不纳入本feature设计；新增权限矩阵、服务重启、全量DML、压力/长稳和性能收益均已移出必要项。

## 6. 结果记录与完成标准

- 每个用例记录状态（未执行/通过/失败/阻塞）、实际SQL/脚本、commit/镜像、计划、结果和oracle来源；UT/BVT历史报告、本轮执行和仅静态核对分列。
- 保留A/B数据、唯一键约束、重复分布、NULL比例、过滤选择性、向量维度/类型、距离/op_type、lists/probe、K/O、模式、seed及两侧投影；没有真实plan不能声称走了索引。
- 正确性失败记录完整(A.id,B.row_id)频数及距离、边界同分组、资格域/候选/最终结果，区分非法行、少重数、分页错误和ANN召回损失。
- 多CN记录入口、producer/reader/最终JOIN的physical placement；事务测试记录快照/提交顺序与同步屏障，取消测试记录资源回收和后续请求。
- 发布验收要求25个执行单元及其合并断言有直接证据：正向语义、保守回退、重复分页、快照、one-CN及清理均不可遗漏。未执行项保持未执行，不用“生成设计/表格渲染正常”替代数据库测试结论。
- 清理本次独占数据库/索引及测试连接，在保存设置的同一连接恢复 `SET experimental_ivf_index = @saved_experimental_ivf_index`；数据库测试连续运行时先确认上轮清理完成，不删除共享业务对象。
