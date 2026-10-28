# MatrixOne HNSW 量化、窄类型、精确预过滤与内存定容测试设计

> Feature [#31467](https://github.com/matrixorigin/matrixone/issues/31467)，编写日期：2026-10-28。实现基线为已合入 [PR #31468](https://github.com/matrixorigin/matrixone/pull/31468)，merge commit `f847803802289b2b2d0df4f0a2f842c2804b823a`。
>
> 规模判断：较大feature，包含存储类型/量化范围持久化、查询精确路径、模型容量与异步索引维护。沿用目录最近的必要验证原则：以需求契约、高风险实现及已知回归为主，共 **50 个必要验证项**；大规模收益与长稳列为独立扩展，不用笛卡尔积增加重复执行。
>
> 本文为测试设计。本轮只核对需求、源码和已有测试，不代表已执行数据库或原生库测试。当前验收还受 [#31493](https://github.com/matrixorigin/matrixone/issues/31493) 报告的异常写入后索引持续缺行问题影响。

## 1. 验收范围与关键契约

### 1.1 当前证据与验收出口

| 证据 | 可支持的结论 | 不能据此推出 |
| --- | --- | --- |
| PR已合入，作者报告UT/BVT及1M向量benchmark通过 | 已有实现和历史验证入口，可复用 | 当前验收环境/后续ref已通过；所有量化均更快或无精度损失。 |
| [QA状态评论](https://github.com/matrixorigin/matrixone/issues/31467#issuecomment-6458910073)确认原生依赖构建/provenance通过 | 依赖来源与该构建环节已验证 | HNSW SQL、量化、训练、精确预过滤、容量、同步、恢复或性能已验证。该评论明确功能执行受资源限制。 |
| #31493在合入SHA上的独立报告 | float16存储越界行提交后，后续合法行仍未进入索引，3个新库观测至30秒 | 当前环境已复现、根因已确认为watermark/retry，或该问题已解决。 |

最终报告分别记录：每项实际结果、环境阻塞和已知产品阻塞。**PR合入、native构建通过、普通正常用例通过，都不能替代异常后的可见性与恢复验收。** 不以本设计代替对#31493的修复确认。

### 1.2 必要验证契约

| 维度 | 合入版本契约及测试边界 |
| --- | --- |
| QUANTIZATION | `float32/float16/bf16/float64/int8/uint8`，大小写不敏感；省略时存储列自身元素类型。固定在CREATE，REINDEX保留且拒绝传入新quantization，改变类型用DROP+CREATE。 |
| 不允许widening | 按实际列元素宽度验证，例如vecf32→float64、vecf16→float32、vecint8→float16拒绝。内部ToStorage可有通用转换能力，不等于SQL层允许widening。 |
| 窄列原生索引 | vecf16/vecbf16/vecint8/vecuint8可直接建索引，build/sync/query在匹配存储类型时按原值交给usearch；不能仅从SQL值正确证明无中间扩宽/复制。 |
| 整数量化与距离 | 显式int8/uint8仿射量化仅支持L2/L2sq，拒绝cosine/IP；不能把该限制推广为原生vecuint8省略quantization时一律不允许IP，已有BVT包含原生整数IP控制。 |
| 训练与dict | 一个索引、所有维度/模型共用同一范围。浮点数据采样并按0.1/99.9百分位训练；仅int8/uint8存储有hnsw_dict，保存quantize_min/max。原生整数用自身类型范围，匹配范围可原样传递。 |
| 范围生命周期 | 普通sync沿用已训练范围，不按新批次重训；空表首次有向量时训练；REINDEX/TRUNCATE清理后按新数据重训。模型和dict在一致事务边界发布。 |
| 两种越界 | 整数仿射范围外的有限值可饱和到端点；float16不可表示的有限值（如100000）应报错，不能变Inf。二者不能混写成“所有越界都饱和/拒绝”。NaN/Inf在ToStorage拒绝。 |
| 距离与精度 | usearch内部L2sq经仿射比例还原，SQL L2/L2sq分别核对。量化后可能碰撞/饱和；精确枚举也只是对存储表示和转换后的query精确，不自动等于原向量精确排名。 |
| 精确预过滤 | 有精确membership及其数量，且`0<count<=hnsw_prefilter_exact_threshold`时扫描合格索引项计算距离；默认5000，范围0..1e9，0关闭。空域另验。依据合格数量，不是选择率估计。 |
| 精确路径成本 | 仍检查所有索引key的membership；阈值限制的是距离计算规模，不代表整索引扫描成本O(合格数)。post无域不因阈值大而变精确扫描。 |
| 模型容量 | `max_index_capacity`默认0；索引显式值→session→可用host内存估计。按存储宽度、维度、M估算可用内存75%所容行数；显式值也受可测内存上限约束。只reserve实际接收行。 |
| 不能测内存 | 合入PlanCapacity中：测量返回未知/0时默认回退1M行，显式正值沿用；测量报错则传播，不能把未知和资源不足错误混为成功。 |
| 构建入口 | `hnsw_create(params,cfg)`内部count/sample/stream，同语句事务；不靠CROSS APPLY逐行调用。 |
| 兼容/范围排除 | 旧索引两张隐藏表无dict仍可读；shutdown-only升级，不宣称混合版本/热升级；无新增协议gate。CDC tail+idxcron merge是后续范围，本期大模型sync仍可能整模型重写。 |

### 1.3 执行矩阵

| 层级 | 必要覆盖 | 使用目的 |
| --- | --- | --- |
| UT/native | 类型转换/指针、量化码、距离还原、exact标志、内存容量注入、错误清理 | 提供SQL无法证明的实现路径证据；确认fork usearch及libmo来源与实际ref匹配。 |
| SQL/BVT | 小样本类型/量化矩阵、SHOW/计划/结果、生命周期及异常恢复 | 使用现有C1..C16分段，补明确缺失的insert/update错误后恢复。 |
| 单/双CN | 正常同步先单CN隔离consumer问题，再双入口验证freshness/generation与dict加载 | 区分未持久同步和另一CN缓存旧模型；不能反复清缓存后只报告成功。 |
| 构建资源 | 按实际resource guard准入；无法link/build时保留阻塞 | 不使用旧二进制或不匹配原生库替代目标ref，不提高限额强行跑大测试。 |
| 扩展专项 | 同数据量化内存/模型数/recall及精确阈值性能 | 单独约定规模和指标，未执行不影响文档完成，也不冒充性能通过。 |

### 1.4 覆盖数量

| 模块 | 必要项 | 编号 |
| --- | ---: | --- |
| DDL、类型与距离支持 | 8 | HQ-001..008 |
| 转换、训练与量化范围 | 10 | HQ-009..018 |
| 精确预过滤与查询 | 8 | HQ-019..026 |
| 内存定容与构建 | 8 | HQ-027..034 |
| 持久化、同步与兼容 | 8 | HQ-035..042 |
| 异常写入、一致性与恢复 | 8 | HQ-043..050 |
| **合计** | **50** | 同fixture合并断言，不重复装载全套数据；类型差异和高危路径不能互相替代。 |

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

### 3.1 DDL、类型与距离支持（8项）

| 编号 | 场景/步骤 | 必须断言 |
| --- | --- | --- |
| HQ-001 | 显式六种quantization、大小写及省略参数，CREATE/SHOW/EXPLAIN | 合法值稳定持久化；省略按列元素类型存储；float64仅合法列上验证。不能只看SHOW证明实际模型类型。 |
| HQ-002 | 四种窄列省略quantization，build/查询/后续sync各一组 | 原生类型正确，模型配置与query类型一致；vecuint8原生IP作控制，不套显式仿射量化L2限制。 |
| HQ-003 | widening反例及同宽/更窄控制 | vecf32→float64、vecf16→float32、整数→浮点宽类型拒绝；合法更窄/同宽按支持矩阵接受，无半发布索引。 |
| HQ-004 | int8/uint8量化的L2、L2sq、cosine、IP | 前两者支持、后两者明确拒绝，不能静默更换距离；已有普通未量化cosine/IP不回归。 |
| HQ-005 | 非法值如int4、错误参数/维度及非向量列 | 明确报错，元数据/模型/dict无部分成功，原表和其它索引可用。 |
| HQ-006 | REINDEX省略、新旧quantization参数及其它合法选项 | 省略保留类型；显式传入quantization按既有规则拒绝，不能原地换码；其它选项可生效且类型不变。 |
| HQ-007 | DROP+CREATE改变quantization，原表同数据 | 新索引配置、dict和缓存引用属于新对象，不能加载旧模型/范围；旧隐藏对象按生命周期清理。 |
| HQ-008 | 旧vecf32/vecf64索引及普通索引共存 | 保持既有索引选择/结果/列绑定；无新dict的历史对象无需补空dict即可读取。 |

### 3.2 转换、训练与量化范围（10项）

| 编号 | 场景/步骤 | 必须断言 |
| --- | --- | --- |
| HQ-009 | ToStorage同类型指针/所有权UT及窄列SQL | 匹配类型且native整数范围一致时原样传递；转换按值而非重解释bits，caller keepalive覆盖C调用；SQL正确不替代此证据。 |
| HQ-010 | f64→f32/f16/bf16，float16=65504边界、近零/负零 | 合法窄化遵循编码值，有限转Inf拒绝；支持最大值不误拒绝，符号零/下溢按类型实现，不新增“保留所有精度”承诺。 |
| HQ-011 | NaN/±Inf、空向量和错误维度直接UT | 各存储类型包括原样路径都校验，明确错误，不进入usearch产生坏距离。未支持SQL特殊值字面量不强求SQL可表达。 |
| HQ-012 | 固定范围[-1,1]，正负端点、query转换和L2/L2sq | codes、mul/add、rescale及SQL距离transform正确；反例能捕获漏除mul²、L2少/多开平方。 |
| HQ-013 | 训练采样、NULL排除、小表全量与大表元素预算 | 一次index-wide范围，样本目标按元素规模约2M而非2M向量；count/sample/stream的snapshot一致；不硬编码大样本每次训练值相同。 |
| HQ-014 | 原生int8/uint8，另一整型目标及matching range | 同型使用[-128,127]/[0,255]；跨类型映射依实际BaseBounds/目标码，不能自动按每批observed min/max重训。 |
| HQ-015 | 常量/全零/负零样本及极窄/极宽range | 正常退化样本得到可用一致范围；无法在f32映射的mul/add明确报错，不能全码静默饱和或除零。 |
| HQ-016 | int8/uint8超训练范围的有限新值及query | 端点饱和仍可被检索，dict不随sync扩张；饱和碰撞不要求区分原向量距离，不能拿float16越界拒绝契约替代。 |
| HQ-017 | 同索引多个模型及新sync批次 | 所有模型共用dict的同一范围，新增模型不自行训练；各自随机范围/按维度范围是失败。 |
| HQ-018 | 非量化、float量化、int量化分别查dict角色 | 仅int8/uint8存储创建dict；quantize_min/max可加载，错误/缺失/不完整bounds按既有校验失败或明确未训练状态处理，不能自造范围返回结果。 |

### 3.3 精确预过滤与查询（8项）

| 编号 | 场景/步骤 | 必须断言 |
| --- | --- | --- |
| HQ-019 | 阈值默认5000、0/1/1e9、-1和1e9+1；EXPLAIN VERBOSE | 合法范围及0禁用，正数预过滤显示PrefilterExactThreshold，0省略；post无membership不带误导的exact配置。 |
| HQ-020 | 稀疏图精确membership S=T-1/T/T+1，阈值0对照 | S<=T的有效精确域启用exact；S>T或0走graph；以native调用exact标志/固定UT漏例证明分支，不能只凭结果相等。 |
| HQ-021 | 同S、不同总行数/选择率；相同估计不同精确S | 根据精确合格数作决定，不用selectivity代替；距离次数与合格索引项对应，仍扫描key过滤的成本另记。 |
| HQ-022 | S=0、S<K、S=K、K=0，NULL/未索引域成员 | 空域0行，少于K正常结束，无无过滤泄漏/挂起；域成员数、真实索引数分开，未索引项不伪造距离。 |
| HQ-023 | 2个以上模型，合格行跨模型，pre/iterative | 每模型exact并合并全局正确Top-K，模型数量不是精确次数阈值；不重复PK，不因iterative绕过已选择exact。 |
| HQ-024 | native与量化exact各一组；近邻差距大与量化collision | exact对当前存储空间枚举正确；无量化/宽间隔fixture可对原表一致；有损数据不得宣称恢复原始精度。 |
| HQ-025 | L2/L2sq及受支持cosine/IP查询、窄query和高精度距离边界 | 查询先按存储类型转换，cosine在窄化后仍满足既有非零/规范检查；usearch距离仍为f32域，f64存储不能被理解成无限距离精度。 |
| HQ-026 | K/O、字面/prepare、EXPLAIN及session隔离 | 分页对合格全局结果生效；threshold在合入版本编码进plan algo_options，记录prepare/SET/重prepare实际时点，不擅自要求和#28988执行期knob相同；无跨session污染。 |

### 3.4 内存定容与构建（8项）

| 编号 | 场景/步骤 | 必须断言 |
| --- | --- | --- |
| HQ-027 | 注入固定hostRowsFitting，不同width/dim/M | 每行估计包含元素宽度、graph links、overhead；较窄类型增加可容行数但非简单8倍，不能只算vector bytes。 |
| HQ-028 | option显式/session/默认0及超内存显式值 | 按优先级确定目标，内存可测时显式值也截限；build和sync同规则，动态环境不硬编码一模型。 |
| HQ-029 | host测量返回0/unknown与返回error分别注入 | unknown默认1M/显式正值沿用；error传播，不把“无空间”当unknown成功，也不把历史1M当新默认固定值。 |
| HQ-030 | 6行显式capacity=2、随后批量插入 | 初始3个2行模型，追加按容量分组，全局搜索正确；检查实际metadata/model数，不只看DDL设置。 |
| HQ-031 | 小表、大目标容量及多批sync | reserve按实际接收行，后续批次可增长而不丢数据/越界；不能一次按可用内存75%预分配整个上限。 |
| HQ-032 | build SQL/计划及内部count/sample/stream | 单次hnsw_create(params,cfg)，无旧CROSS APPLY逐行输入；source/dict引用正确，完整操作在statement事务内。 |
| HQ-033 | count/sample/stream各阶段错误、取消及worker错误 | 停止生产/消费，清理原生模型、buffer及临时文件，不部分发布dict/models、不死锁；只检查create报错不够。 |
| HQ-034 | 空表/全NULL/单行及资源不足时create | 无向量时不发布假训练range；首次sync可继续；准入失败明确报错且原表可用，不能用巨大预分配掩盖问题。 |

### 3.5 持久化、同步与兼容（8项）

| 编号 | 场景/步骤 | 必须断言 |
| --- | --- | --- |
| HQ-035 | CREATE发布与重载，故障注入dict/model写中断 | dict与model事务一致，失败无可查询的半代；load给全部model绑定正确range，不能只校验建索引会话。 |
| HQ-036 | 空表int量化索引→首次正常batch；先delete-only再insert | 首次有效向量训练一次，先建立的空model接收后续range；已有range后续批次不改写。 |
| HQ-037 | 四窄列/int量化的insert/update/delete，事务commit/rollback | sync使用正确源类型和固定range；更新向量/PK按现有规则正确移除/添加，删除不复活，rollback不索引。 |
| HQ-038 | 正常保存/卸载/重新加载及同版本服务重启 | model元素类型、dict、距离单位保持；加载后取与原相同存储oracle，不需要建新索引才能查询。 |
| HQ-039 | REINDEX重训与TRUNCATE后新分布 | quantization保留，旧dict清理并按新数据重训，旧饱和值可重新分开；TRUNCATE空表不残留旧range，后续数据正确。 |
| HQ-040 | COPY ALTER、CREATE TABLE CLONE及DROP清理 | 按实际重建/复制路径保留quantization、数据及正确dict/模型关联；源/目标互不借用range，隐藏对象按生命周期清理。 |
| HQ-041 | 两CN交替读取、一个CN已缓存、另一CN同步/重建 | 正确generation/freshness后两入口均见新数据和dict；记录持久模型已更新与cache刷新两类证据，区分#31456缓存问题。 |
| HQ-042 | 升级前未量化两隐藏表索引，停机升级 | 历史模型可读，无dict视作旧无range含义；新建模型按新capacity规则，老模型不强拆。shutdown-only，不宣称混合版本有新gate保护。 |

### 3.6 异常写入、一致性与恢复（8项）

| 编号 | 场景/步骤 | 必须断言 |
| --- | --- | --- |
| HQ-043 | CREATE含float16不可表示数据；非法query随后合法query | build/query清楚报错不存Inf，失败后正常查询/其它索引仍可用；“报错正确”不替代后续状态检查。 |
| HQ-044 | #31493 INSERT：float16索引建好→提交100000向量→提交合法[2,2] | 比较base、force、索引和maintenance状态。若已commit且后续合法行持续不可见又无明确错误/回退，作为阻塞；不能用量化容差或允许ANN误差解释。 |
| HQ-045 | 独立UPDATE路径：已有正常向量更新为100000→提交后再写合法行 | 与INSERT分开验证，保留原行旧/新值及index状态；异步失败不应静默永久阻断后续有效变化。具体拒绝/回退机制随修复契约确认，不能猜定writer必同步拒绝。 |
| HQ-046 | 对比控制：float16仅合法、异常事务rollback、int量化范围外可饱和 | 正常链可收敛，rollback不污染，int饱和仍索引；三个控制区分consumer停滞、训练饱和和缓存旧代。 |
| HQ-047 | 错误后修正/删除坏行、再写新合法行 | 正式支持的修复操作后索引恢复到对应快照；记录是否需要retry/reindex，未定义修复路径列待确认，不能仅删坏行后不再查后续。 |
| HQ-048 | 错误期间重启或REINDEX恢复、恢复后继续DML | 不丢base数据，不把含坏行REINDEX理所当然视为成功；先按契约处理坏行，再验重建/重启后持续同步，不能只确认服务能启动。 |
| HQ-049 | 缺/坏dict、损坏model或native依赖不匹配的隔离注入 | 明确拒绝/可诊断失败，清理句柄与资源；不能自造range、静默漏模型或用旧库替代；不在共享业务库直接改隐藏对象。 |
| HQ-050 | 小周期重复构建/取消/失败/重载与资源检查 | 无持续C内存、文件、cache准入或worker泄漏，无重复/失效PK；保持resource guard，资源不足记阻塞而非旧二进制通过。 |

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

全部名称已核对PR实际代码或合入设计，路径相对MatrixOne。入口可复用不代表已执行或覆盖整行所有参数。SQL主要为 `test/distributed/cases/pessimistic_transaction/vector/vector_hnsw_quantization.sql` 及对应`.result`，分段C1..C16。

| 必要项 | 已有入口 | 使用范围/缺口 |
| --- | --- | --- |
| 001..008 | `TestHnswValidQuantization`、`TestHnswParamsFromTreeQuantization`、`TestHnswCheckQuantizationBase`、`TestBuildSecondaryIndexDefsQuantizationBase`；BVT C1..C4/C9 | DDL支持矩阵及原生整数控制；旧索引兼容须真实旧数据验。 |
| 002、009、025、037 | `TestToStorageNarrow`、`TestNarrowColumns`、`TestNarrowModelSearch`、`TestHnswSqlWriterNarrowColumns`、`TestNewAlgoQueryType`、`TestNarrowQuerySearch` | 原生类型/指针、query分派、sync SQL；不能用内部允许的通用转换替代DDL限制。 |
| 010..012、015、016 | `TestToStorage`、`TestStoredFloat32`、`TestToStorageQuantizeEdges`、`TestToStorageNonFinite`、`TestQuantizedModelSearch`、`TestRescaleDistancesTinyRange`；BVT C5/C6/C8/C14 | 类型边界、固定bounds、非有限值与距离单位；已提交坏行之后的异步恢复不在这些UT证明范围内。 |
| 013、014、017、018、035、036 | `TestSampleQuantizeBounds`、`TestQuantizedBuildUsesBounds`、`TestDictBounds`、`TestLoadModelsAppliesDictBounds`、`TestSyncQuantizeBounds`、`TestHnswCreateDict`、`TestHnswCreateEmptySource`；BVT C7/C13 | sample/dict/sync共用range；还需失败发布/重启及跨CN载入SQL证据。 |
| 019..024 | `TestFilteredSearchMembershipExact`、`TestHnswSearchMembershipExactThreshold`；planner现有`TestHnswIterativeWidens`等，BVT C11/C12 | sparse graph漏例、阈值边界、多model与pre/iterative；量化exact使用存储oracle补验证。 |
| 025、026 | `TestHnswSearchCosineNarrowedQuery`、`TestNewSearcherQuantization`、`TestExplainSettings`；BVT查询/threshold片段 | 窄化后cosine和explain；prepare/session读取时点不从#28988类推。 |
| 027..031 | `TestPlanCapacity`、`TestBuildModelCapacity`、`TestHnswCreateCapacity`、`TestSyncBatchesGrowModelRoom`、`TestSyncQuantizeBounds`；BVT C15 | mock可用内存、reserve增长、模型拆分；真实mem估计不能只靠一轮RSS判定。 |
| 032..034 | `TestHnswCreateBuildsFromSource`、`TestHnswCreateStreamError`、`TestHnswCreateStreamStopsOnError`、`TestHnswCreateBadInput`、`TestBuildHnswCreate_WrongArgCount` | count/sample/stream、取消/错误清理；配合create失败后无半发布SQL。 |
| 037..042 | BVT C7/C9/C10/C15及现有HNSW模型save/load、generation测试 | 正常sync、REINDEX/COPY/CLONE/TRUNCATE；历史case不自动证明异常恢复、两CN与shutdown升级。 |
| 043..050 | BVT C14仅覆盖建索引/查询越界；#31493最小SQL及新insert/update恢复回归 | **重点覆盖缺口**：已提交坏行、后续合法行、修正/删除/重启与持续同步必须有新直接证据，不能引用C14替代。 |

### 5.3 执行顺序

先冻结验收ref/原生依赖及资源准入 → 跑类型/量化/dict/容量/threshold定向UT与native exact分支 → 小SQL/BVT正常矩阵 → #31493 INSERT和UPDATE隔离回归及恢复 → 两CN缓存/持久化/重启 → 必要静态/owning检查。BVT首次生成结果需人工核对，再进行正常comparison；不能把重新录制当前错误输出当修复。

沿用实际ref的Go/CGo与native provenance流程。已有 `make -j1 cgo NATIVE_BUILD_JOBS=1` 成功记录只能记依赖环节；target test无法link或服务无法构建时保留blocked，不能换旧服务验证新feature。

## 6. 非合入门槛的扩展评估

以下作为独立收益/容量专项，不与50项必要验证相乘；若产品验收另约定指标，再按约定执行。

| 专项 | 比较方法 | 结论约束 |
| --- | --- | --- |
| 量化收益 | 固定同一数据、dim、M/ef、query/seed及硬件，比较f32/f16/bf16/int8/uint8的文件大小、构建峰值、模型数、recall、QPS/p50/p95 | graph links/overhead也占内存，不能宣称整个索引必按元素宽度同比缩小。PR的1M×768数据只是历史证据；f16在该arm64构建可更慢。 |
| 精确阈值交叉点 | 相同总索引N、不同S/维度/ef_search，T=0与覆盖S分别测，另增N观察filter检查成本 | 不规定默认5000在所有机器最优；exact正确与exact更快分别证明。 |
| 大模型sync代价 | 对固定显式capacity及自动mem容量比较单次正常增量的重写字节/耗时，控制资源 | 本期整模型重写是已知限制；不把后续CDC tail+merge设成本期通过条件。大规模长稳另行评估，不阻塞小规模必要回归执行。 |

## 7. 记录与完成标准

每项记录：实际ref/镜像、库与native provenance、平台/内存测量、index类型/quantization/M/ef/capacity、列类型/dim、dict range及generation、threshold与membership count、SQL/UT入口、oracle、实际plan/输出、错误和状态（未执行/通过/失败/阻塞）。原生内存/模型数需要区分reserve、有效行、文件、mmap和可回收cache，不用一个RSS差值替代全部。

同步场景追加commit/维护进度、freshness配置及两CN缓存状态；保存有界轮询的首次与超时结果。若修改全CNfreshness间隔，记录全部节点响应及旧值/默认恢复，测试库/prepare/session/native句柄和临时文件均清理。不会删除共享业务对象。

发布验收应满足：50项必要验证有直接证据或明确阻塞、正常量化和存储oracle正确、精确预过滤无越权/漏行、容量失败与取消无半代/泄漏、异常后合法行可见性问题已解决或明确阻塞。性能与收益结论只覆盖实测范围。

当前#31493仍为待验产品阻塞；错误后的具体处置策略须随修复确认，不能由本设计宣称已修复。本次文档检查通过不改变该状态。

## 8. 资料

| 资料 | 链接 |
| --- | --- |
| Feature/QA状态 | [#31467](https://github.com/matrixorigin/matrixone/issues/31467)、[验收状态评论](https://github.com/matrixorigin/matrixone/issues/31467#issuecomment-6458910073) |
| 实现及设计 | [#31468 files](https://github.com/matrixorigin/matrixone/pull/31468/files)、[合入版本设计](https://github.com/matrixorigin/matrixone/blob/f847803802289b2b2d0df4f0a2f842c2804b823a/docs/design/20261026-hnsw-quantization.md) |
| 转换/容量/搜索 | [types.go](https://github.com/matrixorigin/matrixone/blob/f847803802289b2b2d0df4f0a2f842c2804b823a/pkg/vectorindex/hnsw/types.go)、[capacity.go](https://github.com/matrixorigin/matrixone/blob/f847803802289b2b2d0df4f0a2f842c2804b823a/pkg/vectorindex/hnsw/capacity.go)、[search.go](https://github.com/matrixorigin/matrixone/blob/f847803802289b2b2d0df4f0a2f842c2804b823a/pkg/vectorindex/hnsw/search.go) |
| BVT及结果 | [vector_hnsw_quantization.sql](https://github.com/matrixorigin/matrixone/blob/f847803802289b2b2d0df4f0a2f842c2804b823a/test/distributed/cases/pessimistic_transaction/vector/vector_hnsw_quantization.sql)、[对应result](https://github.com/matrixorigin/matrixone/blob/f847803802289b2b2d0df4f0a2f842c2804b823a/test/distributed/cases/pessimistic_transaction/vector/vector_hnsw_quantization.result) |
| 异常后缺行阻塞 | [#31493](https://github.com/matrixorigin/matrixone/issues/31493)，以reporter实测和修复后新证据分别记录，不自行确认watermark根因。 |
