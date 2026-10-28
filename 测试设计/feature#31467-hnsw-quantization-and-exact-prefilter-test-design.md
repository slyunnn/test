# MatrixOne HNSW 量化、窄类型、精确预过滤与内存定容测试设计（必要项评审版）

> Feature [#31467](https://github.com/matrixorigin/matrixone/issues/31467)，评审日期：2026-10-28。实现基线为已合入 [PR #31468](https://github.com/matrixorigin/matrixone/pull/31468)，merge commit `f847803802289b2b2d0df4f0a2f842c2804b823a`。
>
> 本次将原50项精简为 **32个执行单元**，沿用保留项原编号；被合并项作为主项必须断言，不再独立装载/执行。删除通用或非本次引入的扩展场景、任意损坏注入、重复稳定性循环和性能评估。
>
> 本文是测试设计，未重新运行数据库/原生库测试。[QA评论](https://github.com/matrixorigin/matrixone/issues/31467#issuecomment-6458910073)只确认依赖构建；[#31493](https://github.com/matrixorigin/matrixone/issues/31493)的异常后持续缺行仍需直接验收，不能由本次文档评审解除。

## 1. 评审结论与范围

### 1.1 保留标准与处置

保留直接证明四项需求（量化、窄列、精确预过滤、内存定容）的用例，以及新dict/native转换/异步同步引入的高风险路径和#31493。已有UT能直接证明的边界复用UT，不再另设全量SQL/硬件排列。

| 处置 | 原编号及原因 |
| --- | --- |
| 合并为必须断言 | 005→003；011/015→010/013；014→009；017/035→018/033；021/024→020；029→027；034→036/033；043→010/033；046→044/045；048→047。检查仍在，取消独立执行项。 |
| 删除独立测试 | 007通用DROP重建、008普通索引共存、026通用分页/会话矩阵、049任意文件/库损坏、050重复稳定性循环；兼容性保留042，threshold读取时点并入019，错误清理并入033/047。 |
| 删除非必要专项 | 量化收益benchmark、阈值性能交叉点、大模型重写收益评估及长稳，不列本文件待执行项。 |

### 1.2 不可裁掉的契约

| 维度 | 验收要求 |
| --- | --- |
| 类型 | 六种quantization、大小写、省略按原列类型、禁止widening；四窄列可直接build/sync/query，匹配类型不额外转换。 |
| 整数量化 | 显式int8/uint8仿射量化仅L2/L2sq；原生整数省略量化不能一概套此限制，保留native uint8 IP控制。 |
| 范围 | 每索引单一训练范围；仅int8/uint8有dict；普通sync沿用范围，空表首批训练，REINDEX/TRUNCATE重训。dict/model发布一致。 |
| 越界 | 整数范围外有限值可饱和；float16无法表示值拒绝，不存Inf；已提交异步错误后的合法行仍须可见或明确失败/安全回退。 |
| 精确 | threshold默认5000、0关闭、范围0..1e9；按真实membership数量决定。精确只针对存储表示，不恢复量化前精度；post无域不启用。 |
| 容量 | option→session→可用内存估计；default0；存储width/dim/M参与估算，显式容量也受内存约束，reserve只覆盖实际接收行。 |
| 兼容与边界 | 旧两隐藏表、无dict索引可读，shutdown-only升级。CDC tail+idxcron merge是后续范围；本期整模型重写不作为收益测试扩展。 |

原native构建成功、PR作者历史UT/BVT/benchmark、当前功能验收必须分列。资源不足无法link/build时记阻塞，不换旧服务或不匹配库代替。

### 1.3 执行数量

| 模块 | 执行单元 |
| --- | ---: |
| DDL、类型与距离 | 5 |
| 转换、训练与范围 | 6 |
| 精确预过滤 | 5 |
| 内存定容与构建 | 6 |
| 同步、持久化与兼容 | 7 |
| 异常写入与恢复 | 3 |
| **合计** | **32** |

## 2. 最小数据与oracle

### 2.1 共用SQL数据

```sql
CREATE TABLE t_ref(id BIGINT PRIMARY KEY, v VECF32(4));
INSERT INTO t_ref VALUES
 (1,'[1,1,1,1]'),(2,'[3,3,3,3]'),(3,'[5,5,5,5]'),
 (4,'[50,50,50,50]'),(5,'[52,52,52,52]'),(6,'[54,54,54,54]');

CREATE TABLE t_q(id BIGINT PRIMARY KEY, v VECF32(4));
INSERT INTO t_q SELECT * FROM t_ref;
CREATE INDEX i_q USING hnsw ON t_q(v)
 OP_TYPE 'vector_l2_ops' QUANTIZATION 'int8';

SELECT id FROM t_q ORDER BY l2_distance(v,'[2,2,2,3]') LIMIT 3;
SELECT id FROM t_ref ORDER BY l2_distance(v,'[2,2,2,3]') LIMIT 3;
```

独占数据库内保存/恢复实验开关和session参数。另做vecf64、四种窄类型副本，按实际编码后的值生成精确控制；cosine使用归一化且方向分离的数据，query也满足既有HNSW规范。单个量化参数/错误路径独立装载，避免一种失败污染后续正常结果。

| fixture | 可直接核对的预期 |
| --- | --- |
| 原表q=[2,2,2,3]、K=3 | id顺序2,1,3；L2sq分别3,7,31，L2为其平方根。量化查询按存储oracle验误差；不把原始精确距离直接硬写成量化值。 |
| 原表q=[52,52,52,51]、K=2 | id顺序5,4；L2sq分别1,13。 |
| UT固定训练范围[-1,1] | int8(-1)=-128、int8(1)=127；uint8(-1)=0、uint8(1)=255；mul=127.5。端点间L2sq=255²/127.5²=4，L2=2。 |
| 显式容量2，6行 | 在注入/确认内存可容至少2行后，3个模型，每个2行；后续添加行后重新核对分布。 |
| 稀疏图exact fixture | 5000行，`v=[id%50,floor(id/50)]`，`c=1`当id%500=0，共10个合格；q=[0,0]、K10预期500,1000,...,5000；q=[24,51]、K3预期2500,3000,2000。 |

精确预过滤阈值边界用可控T（如10）验证S=T-1/T/T+1，默认5000与参数范围单独检查。无需为了每个边界都构建多个5000行图。稀疏图设小M/ef_search，用已有固定seed UT证明graph会漏而exact找全；SQL随机图若恰好也找全，不足以证明exact分支已运行。

### 2.2 两层数值oracle

- **原列oracle**：同快照无索引/force，按实际列值和SQL距离计算，测原始准确率及业务排名。
- **存储oracle**：独立按当前dict训练范围、对应类型转换、clamp和query转换枚举所有合格行；测索引转换、距离单位和exact选择是否正确。避免复用被测ToStorage作为唯一oracle。
- 无量化的native整数或可精确表示小样本可强比较；浮点以类型/量化误差确定容差，不能用统一大误差掩盖错误缩放。
- 对量化collision、饱和端点及同分组按合法tie配额验收；不能添加第二排序键改变索引适用性后宣称覆盖原查询。

## 3. 必要测试项

每行是一个执行单元，表中合并断言必须完成；相同fixture复用，类型差异不做无依据的全组合。

### 3.1 DDL、类型与距离（5项）

| 编号 | 场景与必要断言 |
| --- | --- |
| HQ-001 | 六种quantization、大小写、省略参数；CREATE/SHOW与实际model类型一致，float64只在合法列上验证。 |
| HQ-002 | 四窄列省略quantization各覆盖build/query/sync；包含native uint8 IP控制。原生类型与query分派正确。 |
| HQ-003 | widening反例及一条非法量化值（如int4），合法同宽/更窄控制；拒绝不留半发布索引，原表可用。非向量列/普通参数不另扩通用DDL矩阵。 |
| HQ-004 | 显式int8/uint8量化的L2/L2sq接受、cosine/IP拒绝，不静默换距离。 |
| HQ-006 | REINDEX保留quantization、拒绝传入quantization，至少一个其它合法选项可生效；范围重训复用039。 |

### 3.2 转换、训练与范围（6项）

| 编号 | 场景与必要断言 |
| --- | --- |
| HQ-009 | ToStorage匹配类型原样指针/值、转换按数值而非重解释bits；native int8/uint8各用自身范围，另一整数目标映射正确，caller keepalive覆盖C调用。 |
| HQ-010 | float16 65504接受/100000拒绝、f64窄化溢出、NaN/Inf及不可映射range复用直接UT；非法query后合法query正常，不存Inf。空向量/维度错误作为现有UT断言，不另装SQL数据。 |
| HQ-012 | 固定[-1,1]的codes、query转换、rescale与L2/L2sq单位；防漏除mul²及错误开平方。 |
| HQ-013 | 小表全样本、大表采样元素预算与NULL排除；退化全零/常量样本有可用range，count/sample/stream同快照，不要求随机采样重复相同。 |
| HQ-016 | int8/uint8范围外新值饱和且仍被索引，普通sync不扩range；端点碰撞按tie验收，不按原值强排序。 |
| HQ-018 | 仅整数存储有dict，同索引多model/后续batch共享range；load使用正确bounds，缺失/非法bounds的直接UT不能自造range；发布失败清理复用033。 |

### 3.3 精确预过滤（5项）

| 编号 | 场景与必要断言 |
| --- | --- |
| HQ-019 | 默认5000、0、1e9及越界参数；EXPLAIN VERBOSE正数显示、0省略，post无域不启用。threshold按合入plan algo_options读取，prepare只保留一条SET/重prepare时点控制，不扩通用prepared矩阵。 |
| HQ-020 | 固定稀疏图S=T-1/T/T+1和0控制，以native exact标志/固定漏例证明按真实S而非选择率决定；native及一条量化exact分别对原列/存储oracle，collision不要求恢复原精度。 |
| HQ-022 | 同fixture覆盖S=0、S<K、S=K、K=0及少量未索引域成员；无泄漏/挂起/伪造结果，数量按有效索引项判。 |
| HQ-023 | 多model跨域合并，pre/iterative均保留exact选择；全局Top-K正确且无重复。模型拆分数据复用030。 |
| HQ-025 | query按存储类型转换，窄化后cosine非零检查；L2/L2sq距离单位复用012，f64距离仍受usearch f32域约束。 |

### 3.4 内存定容与构建（6项）

| 编号 | 场景与必要断言 |
| --- | --- |
| HQ-027 | 固定hostRowsFitting注入，width/dim/M和graph overhead参与容量；unknown/0默认回退1M或显式值，测量error传播，不能把资源不足当unknown。 |
| HQ-028 | option/session/default0优先级，显式大容量受内存截限，build/sync一致；不做动态宿主内存排列或硬写必一模型。 |
| HQ-030 | 6行capacity=2得到3个model，追加后不超容量、全局检索正确；检查实际metadata/model行数。 |
| HQ-031 | 小表大目标及多批sync只reserve接收行，后续正确增长；不按容量上限一次预分配。 |
| HQ-032 | hnsw_create(params,cfg)内部count/sample/stream，同statement事务，无旧CROSS APPLY逐行调用。 |
| HQ-033 | source读取/worker/取消各用现有直接UT；SQL用float16越界数据导致CREATE失败，检查dict/model无半发布、无资源残留且原表可用，保留C14建索引拒绝证据；不扩任意磁盘损坏或库不匹配注入。 |

### 3.5 同步、持久化与兼容（7项）

| 编号 | 场景与必要断言 |
| --- | --- |
| HQ-036 | 空表/全NULL建立整数索引，首个有效batch训练；delete-only后再insert不遗留无range模型；普通后续batch不重训。 |
| HQ-037 | 四窄列/int量化的insert/update/delete及一条rollback控制；源类型、固定range、移除/添加及存活性正确，不把同机制重复扩全事务组合。 |
| HQ-038 | model保存/卸载/冷加载以及一条同版本重启；存储类型、dict、距离单位保持，不必新建索引才可读。 |
| HQ-039 | REINDEX重训、TRUNCATE后新分布，quantization保留，旧range清理，饱和行重训后可分开。 |
| HQ-040 | COPY ALTER与CLONE两个实际路径各一条；保留类型及正确dict/model绑定、源目标隔离；DROP清理作为末尾断言。 |
| HQ-041 | 一个CN缓存、另一个同步/重建后双入口读取；持久model进度与cache刷新分别核对，区分#31456路径，不全矩阵重复跑两CN。 |
| HQ-042 | 真实旧未量化vecf32/vecf64索引停机升级，无dict可读、旧模型不强拆，新建model按新容量规则。 |

### 3.6 异常写入与恢复（3项）

| 编号 | 场景与必要断言 |
| --- | --- |
| HQ-044 | #31493 INSERT独立库：float16索引后提交越界行再提交合法行；同环境合法控制和rollback控制并入。对比base/force/index/maintenance，不能静默永久漏合法行。 |
| HQ-045 | 独立UPDATE库：已有行改为越界值提交，再写合法行；保留旧/新值及正常控制，不能复用被INSERT污染的索引。 |
| HQ-047 | 对044/045使用修复方案正式支持的一条恢复路径，处理坏行后再写新合法行验证持续同步、错误可见性和清理；修复要求reindex/restart时才验证对应步骤，不强制穷举修正/删除/重启/重建所有组合。 |

## 4. 已知阻塞的独立验收

### 4.1 #31493最小INSERT场景

```sql
CREATE TABLE t_err(id BIGINT PRIMARY KEY, v VECF32(2));
INSERT INTO t_err VALUES (1,'[1,1]');
CREATE INDEX i_err USING hnsw ON t_err(v)
 OP_TYPE 'vector_l2_ops' QUANTIZATION 'float16';

INSERT INTO t_err VALUES (2,'[100000,100000]');
INSERT INTO t_err VALUES (3,'[2,2]');

SELECT id,v FROM t_err ORDER BY id;
EXPLAIN SELECT id FROM t_err ORDER BY l2_distance(v,'[2,2]')
 LIMIT 3 BY RANK WITH OPTION 'mode=pre';
SELECT id FROM t_err ORDER BY l2_distance(v,'[2,2]')
 LIMIT 3 BY RANK WITH OPTION 'mode=pre';
SELECT id FROM t_err ORDER BY l2_distance(v,'[2,2]')
 LIMIT 3 BY RANK WITH OPTION 'mode=force';
```

按每条写入的真实结果分支判定：

- 若修复方案同步拒绝不支持的写入：校验原子性、base中无坏行且合法id3可见，不能假设原合入版本已同步拒绝。
- 若base写入成功：不能仅以ToStorage返回错误判通过。验收索引/查询的明确错误、受支持的安全回退或有证据的恢复；不得持续静默返回只有id1而遗漏合法id3。
- 若三行仍在base，FORCE精确顺序应为3,1,2；若bad insert被拒绝，FORCE为3,1。旧报告的indexed只含1是失败现象，不是新预期。
- UPDATE另用新库，把已存在行改坏，不能复用已被INSERT故障污染的索引。补回滚控制，区分已提交坏向量与根本未提交变更。

同步使用实际maintenance进度、generation/build_ts和缓存freshness证据。允许异步收敛的有界轮询，但保存首次结果、全部错误及超时前样本，不能靠不断retry抹掉停滞。历史0/5/30秒只是报告观测点，不构成“所有环境30秒内必须完成”的新SLA；执行前记录配置与等待预算。原始问题首轮失败和修复后再次执行的结果分别存档。

### 4.2 故障归因与恢复出口

至少区分：

1. base写入是否提交，原SQL对坏行和后续合法行各返回什么。
2. ISCP consumer是否报错/重试，持久化model/dict及metadata是否前进。
3. 查询使用哪个generation，另一CN缓存是否刷新；清cache后正常只能证明缓存路径，不证明consumer恢复。
4. 支持的修正/删除/reindex/restart步骤是否恢复后续同步，以及恢复过程中有没有显式失败/回退。

水位卡住是待定位原因，不能从行缺失直接确认为retry/watermark实现缺陷。修复可改变故障处理方式，但验收出口必须包含数据完整性、错误可见性及后续合法更新不被永久搁置；不要求本设计指定尚未实现的恢复机制。

## 5. 直接证据与已有测试映射

### 5.1 每层证据各自证明什么

| 目标 | 直接证据 | 不足以替代的检查 |
| --- | --- | --- |
| 存储类型/无多余转换 | usearch config、ToStorage返回类型/同值及同型指针UT | SHOW中的quantization字符串或查询值正确。 |
| 模型与范围一致 | index身份、dict min/max、每model bound、发布事务/加载错误 | 只查询创建会话、只看dict存在。 |
| exact选择 | runtime threshold/member count及native filtered exact参数，阈值边界固定UT | graph偶然返回相同Top-K；EXPLAIN显示阈值不代表实际启用。 |
| 精度与单位 | 原列oracle+存储oracle、raw L2sq/rescale/SQL transform | 只比recall或只比返回PK，没有距离值。 |
| 容量/内存 | 注入可用行数、实际reserve/model数、分配失败 | 真实空闲内存不断变化时硬写“量化必一模型”。 |
| 正常与异常同步 | base/force/索引结果、consumer与generation、cache证据、恢复后新DML | native构建通过、普通create通过、单次清缓存后看到新行。 |

### 5.2 可复用入口

SQL主入口为 `test/distributed/cases/pessimistic_transaction/vector/vector_hnsw_quantization.sql` 及对应result；C1..C16分段按保留项选用。已有通过报告不能替代当前ref的直接证据。

| 保留项 | UT/BVT入口 |
| --- | --- |
| 001/002/003/004/006/009 | `TestHnswValidQuantization`、`TestHnswCheckQuantizationBase`、`TestHnswParamsFromTreeQuantization`、`TestToStorageNarrow`、`TestNarrowModelSearch`、`TestHnswSqlWriterNarrowColumns`；BVT C1..C4/C9。 |
| 010/012/013/016/018 | `TestToStorage`、`TestToStorageQuantizeEdges`、`TestToStorageNonFinite`、`TestRescaleDistancesTinyRange`、`TestSampleQuantizeBounds`、`TestDictBounds`、`TestLoadModelsAppliesDictBounds`；C5/C6/C8/C13/C14。 |
| 019/020/022/023/025 | `TestFilteredSearchMembershipExact`、`TestHnswSearchMembershipExactThreshold`、`TestHnswSearchCosineNarrowedQuery`；planner threshold/EXPLAIN测试及C11/C12。 |
| 027/028/030/031/032/033 | `TestPlanCapacity`、`TestBuildModelCapacity`、`TestHnswCreateCapacity`、`TestSyncBatchesGrowModelRoom`、`TestHnswCreateBuildsFromSource`、`TestHnswCreateStreamError`、`TestHnswCreateStreamStopsOnError`；C15。 |
| 036/037/038/039/040/041/042 | `TestHnswCreateEmptySource`、`TestSyncQuantizeBounds`及save/load/generation入口；C7/C9/C10，补实际旧数据升级、重启及双CN证据。 |
| 044/045/047 | #31493最小SQL及insert/update恢复回归；C14只证明build/query越界，不能替代已提交坏行之后的恢复。 |

### 5.3 执行顺序

冻结ref/匹配native及资源准入 → 定向UT和小型SQL矩阵 → #31493两条隔离回归及支持的恢复步骤 → 持久化/双CN/旧数据兼容。只有新预期需生成result并人工核对，随后正常comparison；不能录制当前错误当修复。

沿用仓库Go/CGo及native provenance流程。测试link或服务build资源不足记阻塞，不换旧二进制。复用已有测试中的错误清理/资源断言，不额外安排长稳、吞吐、ANN benchmark或收益专项。

## 6. 记录与完成标准

每项记录：实际ref/镜像、库与native provenance、平台/内存测量、index类型/quantization/M/ef/capacity、列类型/dim、dict range及generation、threshold与membership count、SQL/UT入口、oracle、实际plan/输出、错误和状态（未执行/通过/失败/阻塞）。原生内存/模型数需要区分reserve、有效行、文件、mmap和可回收cache，不用一个RSS差值替代全部。

同步场景追加commit/维护进度、freshness配置及两CN缓存状态；保存有界轮询的首次与超时结果。若修改全CNfreshness间隔，记录全部节点响应及旧值/默认恢复，测试库/prepare/session/native句柄和临时文件均清理。不会删除共享业务对象。

发布验收应满足：32个执行单元及其合并断言有直接证据或明确阻塞、正常量化和存储oracle正确、精确预过滤无越权/漏行、容量失败与取消无半代/泄漏、异常后合法行可见性问题已解决或明确阻塞。本轮验收不包含性能或收益结论。

当前#31493仍为待验产品阻塞；错误后的具体处置策略须随修复确认，不能由本设计宣称已修复。本次文档检查通过不改变该状态。

## 7. 资料

| 资料 | 链接 |
| --- | --- |
| Feature/QA状态 | [#31467](https://github.com/matrixorigin/matrixone/issues/31467)、[验收状态评论](https://github.com/matrixorigin/matrixone/issues/31467#issuecomment-6458910073) |
| 实现及设计 | [#31468 files](https://github.com/matrixorigin/matrixone/pull/31468/files)、[合入版本设计](https://github.com/matrixorigin/matrixone/blob/f847803802289b2b2d0df4f0a2f842c2804b823a/docs/design/20261026-hnsw-quantization.md) |
| 转换/容量/搜索 | [types.go](https://github.com/matrixorigin/matrixone/blob/f847803802289b2b2d0df4f0a2f842c2804b823a/pkg/vectorindex/hnsw/types.go)、[capacity.go](https://github.com/matrixorigin/matrixone/blob/f847803802289b2b2d0df4f0a2f842c2804b823a/pkg/vectorindex/hnsw/capacity.go)、[search.go](https://github.com/matrixorigin/matrixone/blob/f847803802289b2b2d0df4f0a2f842c2804b823a/pkg/vectorindex/hnsw/search.go) |
| BVT及结果 | [vector_hnsw_quantization.sql](https://github.com/matrixorigin/matrixone/blob/f847803802289b2b2d0df4f0a2f842c2804b823a/test/distributed/cases/pessimistic_transaction/vector/vector_hnsw_quantization.sql)、[对应result](https://github.com/matrixorigin/matrixone/blob/f847803802289b2b2d0df4f0a2f842c2804b823a/test/distributed/cases/pessimistic_transaction/vector/vector_hnsw_quantization.result) |
| 异常后缺行阻塞 | [#31493](https://github.com/matrixorigin/matrixone/issues/31493)，以reporter实测和修复后新证据分别记录，不自行确认watermark根因。 |
