# MatrixOne 多索引过滤向量检索测试设计

> Feature [#28988](https://github.com/matrixorigin/matrixone/issues/28988)：统一HNSW、IVFPQ、CAGRA的过滤向量检索，并覆盖IVF-FLAT兼容性。属于较大feature：涉及查询模式、候选预算、membership、GPU过滤、迭代终止、PREPARE会话参数与远端协议，需完整测试设计。
>
> 编写日期：2026-10-23。实现基线为已合入[PR #30127](https://github.com/matrixorigin/matrixone/pull/30127)，merge commit `60ad9a14fc6eda33b10ce74226078de3cffd24b2`。本文是拟执行的测试设计，历史issue现象、PR作者报告与本轮实际执行分开记录；本轮未运行数据库、GPU或性能测试。

## 1. 测试原则

| 项目 | 说明 |
| --- | --- |
| 需求与交付双重判定 | 原issue要求post自适应完成、default自动选路与不静默短页；交付版本选择显式iterative扩搜，post/default可短页。分别报告实现契约符合度与原需求缺口，不因PR已合入自动判原需求全部满足。 |
| 合格性强约束 | 任何模式不得返回不满足关系谓词、membership、删除状态或租户范围的行；ANN近似不能解释过滤泄漏。 |
| 行数与召回分开 | 满K不代表近邻召回正确；短页不一定是当前模式的实现错误。分别记录实际行数、合法候选总数、召回、预算、终止原因与原需求差距。 |
| 模式显式验证 | default、post、pre、iterative、include、auto、force各检查实际计划/调用，不能把名字相似当语义相同。 |
| 计划与运行双校验 | EXPLAIN证明静态模式/SQL参数，运行证据证明实际扩搜、会话参数、adaptive分支和资源回收。不能用最后返回10行倒推一定扩搜过。 |
| 数值与存储区别 | FORCE按原表精确CPU计算；HNSW/CAGRA/IVF-FLAT候选选择仍可能近似，IVFPQ即使全探测仍按PQ距离排序；量化存储不要求与原始f32排名逐项相等。 |
| 快照与生命周期 | 每次执行建立自己的域/过滤状态；取消、空域、提前LIMIT、PREPARE重用、shard及CDC overflow不得泄漏资源或复用过期状态。 |

### 1.1 基线及能力范围

| 项目 | 内容 |
| --- | --- |
| 算法 | HNSW（CPU/usearch）、IVF-FLAT（CPU搜索，可GPU质心）、CAGRA及IVFPQ（GPU/cuVS）；算法间比较使用等价数据副本和距离。 |
| CPU入口 | `experimental_hnsw_index`、`experimental_ivf_index`等按实际ref配置；两CN入口均验证。 |
| GPU入口 | 真GPU及配套libmo/cuVS构建，开启对应实验索引；记录gpu_mode、GPU/驱动/CUDA/cuVS、设备与索引参数，不能用CPU回退冒充GPU测试通过。 |
| 过滤机制 | PK过滤、INCLUDE列内索引过滤、其它关系谓词形成的membership，以及搜索后residual过滤；距离范围谓词单独核对。 |
| 配置 | SQL选项`itopk/max_itopk/nprobe/max_nprobe`；会话`vector_itopk/vector_max_itopk/vector_max_nprobe`按执行解析，`probe_limit`保留原有计划期读取。 |
| 参数协议 | 合入基线`SearchKnobsProtocolVersion=V127`；无三个新增非零knob字段的IndexSearchScan基线V111。执行后续版本时核对常量；不把版本号当全链路混合版本支持证明。 |
| 原需求红例 | 固定历史commit `01d60e1c4ded1b0f3fc1a4ecd75ce54e95e23b90`，近端500行均不合格，HNSW固定预算返回0而FORCE返回501..510。历史不是本轮重现结果。 |

### 1.2 需求与交付差异及验收记录

| 原issue要求 | PR合入版本的交付行为 | 测试处理 |
| --- | --- | --- |
| post过滤后自动补候选直到满足或耗尽 | post一次搜索，扩搜由独立iterative负责；IVF-FLAT某些INCLUDE post空页有force兜底 | 验证post单次及iterative补搜；post短页符合交付也要单列原需求差距，不能标成需求已完成。 |
| auto/default自动选择 | 显式auto为ADAPTIVE_TOP或安全force；default兼容旧行为；既有默认策略会话开关只作用于IVF-FLAT | default与显式auto分别验收。未设mode不保证自动补满。 |
| 索引/谓词不能保证页完整时精确回退或等价完成 | iterative有cap，IVF-FLAT多reader场景也可能单轮；pre单次可短；auto可依次尝试post/pre/force | 记录支持矩阵、cap和实际终止，不把“加大参数”当无条件完成证明。是否接受缩小范围由需求验收明确记录。 |
| 可观测策略、预算、补搜与回退 | EXPLAIN显示SQL参数及Index Filter；会话参数不进可复用plan，通常不打印；IVF-FLAT ANALYZE有round_count | 静态字段与实际运行证据组合；缺少其它算法轮次/终止可观测性时列缺口，不编造字段。 |

完整验收报告必须同时写“交付行为是否符合矩阵”和“原issue哪些要求尚未满足/被范围调整替代”。本设计记录差异，不代表替需求方批准这些调整。

### 1.3 模式矩阵

| 模式 | 合入基线行为 | 不足额时 |
| --- | --- | --- |
| default（省略mode） | 单次搜索，PK可内过滤；IVF-FLAT/CAGRA/IVFPQ的可下推INCLUDE过滤也内执行；其余通常搜索后过滤 | 保持既有默认行为，不承诺迭代完成；IVF-FLAT原有默认策略设置另作控制。 |
| post | 单次搜索，PK可内过滤，INCLUDE及其它普通过滤搜索后执行 | 不迭代；IVF-FLAT裸Top-K的INCLUDE post空页可走force兜底，非空短页不能据此假设也兜底。 |
| pre | 单次把能下推的过滤放入搜索，INCLUDE/PK走索引条件，其余经PRE/membership；HNSW通过membership | 可短页，不因一次搜索叫pre就宣称全探测/精确召回。 |
| iterative | pre策略配合逐轮扩搜，受算法cap、索引行数、实际list数及执行形态约束 | 达到页目标或终止条件停止，允许已声明边界的短页；不能无限循环。 |
| include | 所有四种算法均接受为iterative别名 | 按iterative，非独立算法；无INCLUDE列也可验证别名。要测INCLUDE下推本身必须建含INCLUDE列的索引。 |
| auto | 安全区域尝试ADAPTIVE_TOP `[post,pre,force]`，pre与post等价时可省略；不安全/不适配区域直接force | 第一个满页分支胜出，前面不足再走后续分支。满页ANN不等于精确召回，不能要求auto总等于FORCE的ID。 |
| force | 绕过向量索引，CPU精确关系排序 | 对同快照、合法输入、无执行错误，应返回精确窗口；不是“强制GPU/索引”。 |

### 1.4 算法扩搜与排名边界

| 算法 | 轮次推进 | 终止/短页边界 | 排名判定 |
| --- | --- | --- | --- |
| HNSW | 重启搜索，扩大itopk，membership支持选择率预测起点，不修改共享索引ef_search | max_itopk、索引行数、页目标或更小精确membership；默认iterative cap20000 | 对原存储向量计算距离；候选选择仍ANN。历史单调fixture可核对501..510，不外推全部精确召回。 |
| CAGRA | 重启搜索，扩大itopk；cuVS filtering_rate保持auto | SQL itopk/max_itopk上限1024；共享session超限按1024截限；默认cap1024；cuVS内部列表可能再受filtering_rate放大 | 满页可能仍遗漏更近合格行；显式更大搜索宽度可改善，不硬编码图构建产生的ANN ID。 |
| IVFPQ | 重启并扩大nprobe，实际built index的NLists为终态上限；每轮可能重搜已访问列表 | max_nprobe、实际lists、索引行数、membership、取消；overflow-only无list时首轮即终轮 | 全探测仍PQ近似，不能要求等于FORCE原向量排名。 |
| IVF-FLAT | 持续同一质心cursor，进一步扫描列表；不重复扫描已访问列表 | cap/所有list；split readers共享路由时按交付契约单轮。小精确membership<=100走全list精确PK路径；纯INCLUDE不等同小membership | native存储候选距离精确，量化存储按量化值；ANN列表范围仍可影响召回。 |

### 1.5 覆盖范围

| 一级模块 | 用例数 | 编号 |
| --- | ---: | --- |
| 模式与策略选择 | 10 | FVS-001..010 |
| 过滤、结果数量与距离 | 12 | FVS-011..022 |
| 扩搜、终止与分页 | 12 | FVS-023..034 |
| 参数、PREPARE与协议 | 10 | FVS-035..044 |
| C membership/GPU mask与资源 | 10 | FVS-045..054 |
| DML、执行生命周期与多CN | 10 | FVS-055..064 |
| 性能、召回与长稳 | 6 | FVS-065..070 |
| **合计** | **70** | 按适用算法、模式、硬件及参数组合展开，不把GPU缺失记为通过。 |

### 1.6 参考资料

| 资料 | 地址 |
| --- | --- |
| 需求及讨论 | [Issue #28988](https://github.com/matrixorigin/matrixone/issues/28988)；[研发能力边界说明](https://github.com/matrixorigin/matrixone/issues/28988#issuecomment-5711589347) |
| 历史分页/谓词矩阵 | [固定候选阈值与深分页](https://github.com/matrixorigin/matrixone/issues/28988#issuecomment-5712275136)；[多子索引](https://github.com/matrixorigin/matrixone/issues/28988#issuecomment-5708306159) |
| 实现与交付设计 | [PR #30127](https://github.com/matrixorigin/matrixone/pull/30127/files)；[合入版本设计](https://github.com/matrixorigin/matrixone/blob/60ad9a14fc6eda33b10ce74226078de3cffd24b2/docs/design/20261012-filtered-vector-search.md) |
| 协议 | [合入版本protocol.go](https://github.com/matrixorigin/matrixone/blob/60ad9a14fc6eda33b10ce74226078de3cffd24b2/pkg/sql/colexec/vectorscan/protocol.go) |
| 外部性能线索 | [100k×128维用户报告](https://github.com/matrixorigin/matrixone/issues/28988#issuecomment-6108068652)；脚本未随评论提供，不将该报告当可复现或本轮实测数据。 |

## 2. 数据准备与oracle

### 2.1 CPU最小fixture

使用独占数据库、保存/恢复会话设置，复制无向量索引的等价数据表作独立精确控制。HNSW示例来自需求与已交付BVT，不要求其它索引盲目接受同一DDL参数。

```sql
SET experimental_hnsw_index=1;
CREATE TABLE t_l2 (
  id BIGINT PRIMARY KEY,
  category INT NOT NULL,
  tag INT,
  v VECF32(2)
);
INSERT INTO t_l2
SELECT result, IF(result<=500,0,1),
       IF(result%3=0,NULL,result%10),
       CAST(CONCAT('[',result,',0]') AS VECF32(2))
FROM generate_series(1,1000,1) g;
CREATE INDEX i_l2 USING hnsw ON t_l2(v)
  OP_TYPE 'vector_l2_ops' max_index_capacity 200;

SELECT id FROM t_l2 WHERE category=1
ORDER BY l2_distance(v,'[0,0]')
LIMIT 10 BY RANK WITH OPTION 'mode=iterative';
```

逐项替换mode为post/pre/iterative/include/auto/force，并单列无mode控制。观测是否实际形成5个子索引，不因capacity设置就假定落盘子索引数正确。其它fixture包括：

- cosine：`normalize_l2([1,id/1000])`，查询`[1,0]`；inner product：`[1001-id,0]`，查询`[1,0]`，沿包/SQL负内积升序语义验收。L2/cosine/IP能力按索引实际支持逐项记录。
- IVF-FLAT：同数据副本，lists=4/8；有/无`include(category)`两张独立索引表。`grp`与category同值但不进入INCLUDE，`note=id%3`作为其它过滤列。
- GPU：8维`[id,0,...,0]`或BVT固定的缩放数据；CAGRA使用明确graph参数及`include(category)`，IVFPQ必须显式lists（如8）、m=8、bits_per_code=8；确认训练和实际建成索引，保留算法参数/训练seed。不能把索引未完成时的精确计划当GPU验证。
- 稀疏分布：prefix、periodic、随机、与距离强相关/反相关；NULL向量、NULL tag、无索引项membership及不同PK分布单设表，避免污染固定预期。

### 2.2 精确窗口与历史回归

| 数据/条件 | K/O | FORCE及无索引精确预期 |
| --- | --- | --- |
| 1..1000单调L2、category=1 | 10/0 | 501..510，共10行。 |
| 同上 | 10/10 | 511..520。 |
| 同上深分页 | 10/490 | 991..1000。 |
| 同上越界 | 10/500 | 0行；少页源于窗口耗尽。 |
| id%7=0，查询原点 | 10/0 | 7,14,...,70；总合格142。 |
| 同上 | 49/0、50/0 | 分别7..343步长7、7..350步长7。 |
| id IN(3,700,999) | 10/0 | 3,700,999，仅3行。 |
| id BETWEEN991 AND1000 | 10/0 | 恰好10行。 |
| category=9 | 10/0 | 0行。 |
| category=1 AND(tag=4 OR tag IS NULL) | 5/0 | 501,504,507,510,513。 |

这些是精确oracle，不表示每种ANN模式都必须返回同一ID。HNSW已交付的单调fixture、IVF-FLAT全探测native或小精确membership控制可采用强结果比较；CAGRA以合法性、行数及测量召回为主，IVFPQ单列PQ排序。K=9/10、49/50的历史预算断点均保留回归，但不硬编码新策略必须沿用历史20/75/98预算。

设同一快照上满足完整WHERE且向量距离有效的行数为Q，精确页目标 `P=min(K,max(Q-O,0))`。实际行数小于P叫短页，不能拿包含NULL/已删/未索引项的原始membership数直接充当可搜索Q。具体索引的可见性与CDC阶段需要单独核对。

## 3. 测试用例

### 3.1 模式与策略选择（10条）

| 编号 | 二级模块 | 场景与步骤 | 预期结果/证据 |
| --- | --- | --- | --- |
| FVS-001 | default兼容 | 四算法省略mode/knob，对比合入前固定基线；INCLUDE与非INCLUDE分组 | 保持声明的默认行为及原有会话策略，记录short page；不能强行要求default变auto。IVF-FLAT零向量重复修复等明确变化另验。 |
| FVS-002 | post | PK、INCLUDE、普通列各测mode=post | 一次索引搜索；仅可用PK条件内执行，其余residual；返回行须合法。短页记录交付允许及原需求缺口。 |
| FVS-003 | pre | category有/无INCLUDE，分别EXPLAIN及查询 | HNSW经membership；IVF-FLAT/GPU可下推列内过滤，其它经PRE域。一次搜索不宣称自动扩搜。 |
| FVS-004 | iterative | 近端不合格fixture，小起点、大cap | 内执行资格过滤并按算法推进，满目标或达到明确终止条件；检查调用/轮次，不以满页直接推断多轮。 |
| FVS-005 | include兼容 | 四算法mode=include对比iterative，含有/无INCLUDE列两组 | 是iterative别名，模式/预算等价；无列不应把别名误判非法。近似图结果按合法性/召回而非无条件逐ID一致比较。 |
| FVS-006 | auto分支 | 裸单列top-k构造post满、post短/pre满、前两支短三组 | 检查ADAPTIVE_TOP分支实际执行与结果；后支不应在前支已满时重复发布，最后force保证精确页数量；不要求已满ANN支召回100%。 |
| FVS-007 | force | 同SQL在CPU/GPU环境mode=force | 绕过向量索引并按原向量精确排序，得到P行；计划是CPU exact，不能作为GPU搜索速度。 |
| FVS-008 | auto安全回退 | 多输出列、外层聚合包裹top-k、provider/特殊membership区域、低估行数及volatile | 按交付适用规则直接force或省略不必要pre；保留SQL语义，别把无Adaptive Top一律当缺陷。 |
| FVS-009 | 索引能力选择 | 同表多个索引及距离不匹配控制，构造非服务索引不支持某knob/mode | 仅真正服务查询的索引执行能力校验；未选索引不能错误拒绝查询，非法实际使用组合仍拒绝。 |
| FVS-010 | IVF-FLAT兼容例外 | post+INCLUDE，分别0行、非空短页、满页；default/pre对照 | post空页的既有force分支正确，非空短页不误当全补齐保证；纯INCLUDE pre不必建membership JOIN。 |

### 3.2 过滤、数量与距离（12条）

| 编号 | 二级模块 | 场景与步骤 | 预期结果/证据 |
| --- | --- | --- | --- |
| FVS-011 | 原需求红例 | HNSW两CN，category=1；default/post/pre/iterative/auto/force | 对历史0/10动机保留控制；已交付HNSW pre/iterative单调fixture应为501..510。其它模式依§1矩阵，不能统一判全精确。 |
| FVS-012 | 距离种类 | 复用L2/cosine/IP单调数据，选正确op_type | 验证通用过滤/预算机制；HNSW固定fixture核对精确ID。其它算法仅对已支持距离展开，不能偷偷用不匹配索引。 |
| FVS-013 | PK过滤 | 范围、IN、负数/离散BIGINT主键 | PK可直接过滤时不丢行/泄漏，映射不受符号/稀疏值破坏；小membership控制独立检查。 |
| FVS-014 | INCLUDE位置 | category进INCLUDE，grp不进，值完全相同；分别过滤 | 验证索引内filter与membership/残余的真实区别；INCLUDE机制测试必须有含列索引，不能只写mode=include。 |
| FVS-015 | 组合谓词 | category=1 AND(note=2 OR tag IS NULL)，普通列与INCLUDE/PK混合 | producer保留全部必要关系过滤，GPU mask为AND，不放宽OR/NULL语义；每个结果满足原WHERE。 |
| FVS-016 | 周期与阈值 | id%7=0，K=9/10/49/50；近端全不合格/随机选择率对照 | 迭代适用且预算足够时不因历史倍数断点突然只返回少量行；post/default短页按交付记录，不硬编码旧候选预算。 |
| FVS-017 | NULL语义 | tag IS NULL/IS NOT NULL、NULL向量、参数NULL及NULL键控制 | WHERE三值逻辑正确；NULL向量/距离按原SQL和索引可见性契约，不把membership行数等同有效向量数。 |
| FVS-018 | 距离谓词 | 距离<、<=边界及不同query vector的关系过滤 | 范围按搜索输出/residual契约处理，GPU距离范围不错误制造普通PRE域；所有返回值满足源WHERE。边界精度与PQ排名分开。 |
| FVS-019 | 0/少于K | Q=0、3，K=10；索引空、域空及表空 | force目标0/3；精确空域不做无过滤搜索；iterative能识别更小目标/耗尽并结束，不为了凑K无限取数。 |
| FVS-020 | 恰好/大于K | Q=K及Q远大于K，改变合格行在向量空间的位置 | 符合交付矩阵的完成路径返回P；验证filter真值与去重；达到cap的ANN短页附证据，不自动宣称原需求通过。 |
| FVS-021 | fulltext混合 | MATCH过滤+向量top-k，score不投影与投影两组 | HNSW/GPU按支持路径同时使用全文与向量索引；filter-only MATCH在membership producer执行一次；读取score时不错误删必要求值。 |
| FVS-022 | IN/SEMI资格 | id IN子查询、空子查询、关联表资格及重复键控制 | membership表示资格集合，不能凭此声称支持任意INNER JOIN重复展开。与#29702交互按其已支持形态复核，保留原SQL multiplicity。 |

### 3.3 扩搜、终止与分页（12条）

| 编号 | 二级模块 | 场景与步骤 | 预期结果/证据 |
| --- | --- | --- | --- |
| FVS-023 | HNSW多子索引 | capacity=200的1000行及capacity=400的8000行，确认实际5/20子索引 | 扩搜跨子索引合并，候选按PK去重并全局排序；members/readers共享只读域，不能只补第一个子索引。 |
| FVS-024 | HNSW cap | 小/大itopk和max_itopk，含membership预测起点控制 | 按最终预算/索引行数饱和推进，cap耗尽正常终止；不改共享ExpansionSearch，后续另一session不受污染。 |
| FVS-025 | CAGRA cap | 表>1024行、合格数<K、SQL/会话上限 | 不向cuVS传超1024的返回key预算；短页可达cap，不能无限扩搜或抛历史topk>1024错误；内部filtering_rate放大另记。 |
| FVS-026 | IVFPQ list终态 | catalog lists与实际built NLists不一致控制，nprobe逐轮扩大 | 终态依实际索引NLists，达cap/list数/行数停止，doubling饱和；不以错误catalog参数维持无限重搜。 |
| FVS-027 | IVFPQ overflow-only | 仅CDC overflow、无built list，过滤合格<K；inline建索引遗漏lists控制 | overflow-only首轮已搜全部行即终轮；inline IVFPQ无lists与CREATE INDEX一致拒绝，不能建成后卡在iterative。 |
| FVS-028 | IVF-FLAT cursor | 多轮、质心同分/零cosine vector | 质心排名是完整permutation，同一list不重复扫描；无重复PK、无丢list，不用每轮重启模拟cursor继续。 |
| FVS-029 | IVF-FLAT小域 | exact membership大小99/100/101，纯INCLUDE同样命中数作对照 | <=100精确PK路径扫描全list，EXPLAIN窗口如实反映；纯INCLUDE无精确域不能误套保证，101按一般ANN预算验证。 |
| FVS-030 | IVF-FLAT多reader | local DOP、多CN读entries触发split及单reader对照 | 交付split形态按单轮执行，可短页；保留实际physical plan及缺口，不从单reader成功推断分布式也补满。 |
| FVS-031 | OFFSET | K/O=10/10、10/490、10/500，普通/参数LIMIT | 目标按P；OFFSET在过滤/全局排序后只消费一次，内部预算含所需窗口。近似结果另测召回，不拿全局未过滤ANN窗口当oracle。 |
| FVS-032 | LIMIT零/算术 | K=0/1，O很大、K+O溢出，static/prepare交替 | 零页不等待未启动producer；非法或不可表示预算保留明确错误/安全路径，不回绕、panic或巨额分配。 |
| FVS-033 | 不足/无进展 | 多轮候选重复、members无对应索引项、cap先到 | 去重后计数，未达目标也必须按上限结束，终止原因可定位；未索引NULL成员不能令迭代无限等待。 |
| FVS-034 | 满页非精确 | 图中最近节点被过滤、CAGRA默认width已有满K；IVFPQ全探测 | iterative按行数满页就停，不自动为提升recall继续；CAGRA更宽itopk及IVFPQ原向量oracle报告质量，不要求全探测PQ变精确。 |

### 3.4 参数、PREPARE与协议（10条）

| 编号 | 二级模块 | 场景与步骤 | 预期结果/证据 |
| --- | --- | --- | --- |
| FVS-035 | 选项解析 | 未知key、非key=value、0/负数/小数/非数字、重复冲突选项 | 按解析契约拒绝不合法输入，不静默忽略；重复项以已定义解析规则验收并记录，不能凭设计猜测覆盖顺序。 |
| FVS-036 | 算法族 | graph传nprobe/max_nprobe、IVF传合法四种knob | 不适用族拒绝；实际选择其它不使用该knob的扫描不被session值错误加门禁；某mode不使用但该族支持的选项可接受。 |
| FVS-037 | 起点/上限 | SQL itopk=100/max_itopk=50；nprobe=8/max_nprobe=4；nprobe=1/80000/80001 | 非法显式cap<start拒绝；SQL探测范围1..80000；具体actual list数再约束搜索，不把允许解析等同确实探了80000。 |
| FVS-038 | CAGRA参数差异 | SQL itopk/max_itopk=1024/1025；session相同值 | SQL>1024拒绝，共享session超值对CAGRA按1024截限；对HNSW不套CAGRA上限；还原会话后正常。 |
| FVS-039 | PREPARE逐次解析 | prepare固定SQL后SET三个vector_*变量，execute，再reset；更换LIMIT | 新执行读当前session值、无旧plan污染；probe_limit维持原计划期契约，不能强求SET后也同样动态变化。 |
| FVS-040 | 优先级与隔离 | SQL选项、session、索引ef_search/itopk_size、内建值逐级组合；双session并发 | 依据交付优先级选择并记录最终值；只设cap而未显式itopk的post不因此变迭代；无跨session共享knob。 |
| FVS-041 | 极大itopk | uint64最大值，HNSW/IVFPQ/IVF-FLAT INCLUDE，default/pre；CAGRA控制 | 前三按实际索引/列表行数限制分配，storage Top-K列不丢，不panic；CAGRA按上限拒绝。 |
| FVS-042 | EXPLAIN证据 | SQL四knob及Index Filter、预算/窗口；session-only值对照 | 静态字段按实际设置显示，SQL不是session快照；session值用运行request证据验证。IVF-FLAT ANALYZE round_count可检查，其它算法不编造轮数字段。 |
| FVS-043 | 协议拒绝 | 带internal_topk/max_internal_topk/max_nprobe的scope发往V126/V127；发送时目标版本变化 | V127基线以下在placement、发送re-probe、stream handshake或receiver拒绝，不能忽略字段；同scope无knob控制按其V111基础协议判定。 |
| FVS-044 | 协议适用范围 | session配置在coordinator与remote scan副本传递；fulltext等非适用扫描控制 | 每次消息副本带适用参数、不写回可复用plan；非向量扫描不被无关session变量抬高协议要求。iterative algo_options旧节点可忽略，不能由knob fence推断无knob迭代也有完整旧节点保护。 |

### 3.5 C membership、GPU mask与资源（10条）

| 编号 | 二级模块 | 场景与步骤 | 预期结果/证据 |
| --- | --- | --- | --- |
| FVS-045 | C表示选择 | 紧凑跨度、稀疏/跨container、少量远距BIGINT key，包含负值；C sorted64再测元素宽度1/2/4/8、非法宽度及截断项 | docfilter.NewForC按资格选cbitmap或较小估计的CRoaring/C sorted64；C/C++读写key编码一致，结果精确，内存估计可核对；非法宽度拒绝、截断项不越界读取。 |
| FVS-046 | 错误表示 | GPU收到Go内存membership、非exact表示或缺int64 host_ids；HNSW负向控制 | 不支持表示明确拒绝，不能按“无membership”继续；GPU只接受支持的C exact表示，空exact集合仍是拒绝全部。 |
| FVS-047 | 内存准入 | membership估计预留失败、引用计数共享及最后Free | build前准入，拒绝报错；最后owner释放后归还，失败/取消无C内存泄漏或重复Free。 |
| FVS-048 | GPU AND组合 | membership、INCLUDE/PK谓词、删除位各自及组合；shard切片 | fused mask为三者交集，非union；无额外membership复制/按key错误取行，shard边界和负/稀疏PK正确。 |
| FVS-049 | prepared bitset复用 | CAGRA/IVFPQ多轮及multi-index/sharded，记录prepare/upload调用 | 同搜索每个index/shard/overflow各准备一次mask并复用各轮；不能因扩搜重复全表mask pass，正常释放。 |
| FVS-050 | prepare后新增行 | mask建好后在直接测试fixture追加row，再SearchQuantizeWithBitset；分片边界控制 | 超mask范围新增行不因raft默认全1而通过；upload清尾位且copy限制shard。不把该C++结构测试当SQL跨语句快照证明。 |
| FVS-051 | 删除快照 | prepare时已有delete、轮间新delete及下一查询 | 旧delete必须排除；当前搜索按prepare时delete契约，最终SQL结果仍遵循引擎可见性；下一搜索重建。不得宣称每轮都重读delete。 |
| FVS-052 | mask准入 | mask及delete拷贝合计在1MiB前后，注入拒绝/分配失败 | 达门槛按实际双份内存准入并归还；OOM明确传播，无CPU偷偷替换GPU索引搜索，也无部分mask继续执行。 |
| FVS-053 | IVFPQ sentinel | probed lists没有合格行但全索引有，默认/pre/iterative分别运行 | host post-filter把padding/excluded候选变sentinel并剔除，无非法ID/距离泄漏或vector magnitude错误；空首轮不等于全索引耗尽。 |
| FVS-054 | GPU调用边界 | 正常/取消/设备错误，观察共享锁及CPU/GPU路径 | 索引搜索/扩搜实际在GPU，mask准备和关系扫描可CPU；GPU调用不持MatrixOne共享锁，错误正常传播。auto显式force分支另标CPU exact，不冒称GPU扩搜。 |

### 3.6 DML、执行生命周期与多CN（10条）

| 编号 | 二级模块 | 场景与步骤 | 预期结果/证据 |
| --- | --- | --- | --- |
| FVS-055 | DML可见性 | INSERT/DELETE/UPDATE category、INCLUDE/非INCLUDE、向量；事务提交/回滚 | 在既有索引同步/可见性屏障后新语句重算资格和候选，旧payload不泄漏，rollback行不可见；不以sleep替代同步证据。 |
| FVS-056 | built+overflow | CAGRA/IVFPQ built index加CDC overflow，PK/subquery/membership各模式 | built与overflow都用正确过滤并全局合并，PK不重复，不能只过滤built或漏新可见行。 |
| FVS-057 | REINDEX/sharding | 异步重建、空pipeline end、分片过滤及切换 | 无rows builder不panic，generation及host_ids/mask匹配，sharded pre/iterative与过滤真值一致。 |
| FVS-058 | 跨轮取消 | HNSW/CAGRA/IVFPQ轮间及IVF cursor读取阶段确定性取消 | 及时终止、无下一轮，返回取消而非成功空页；域、GPU mask、C owner及内存准入归还，后续请求正常。 |
| FVS-059 | 空域/缺域/失败 | MustApply域尚未发布、构建失败、空域、提前LIMIT0 | 缺required域不能当PASS，失败传播，空域0行，consumer不死等；保留执行顺序的测试屏障证据。 |
| FVS-060 | PRE单次求值 | nextval/volatile过滤及membership join，pre/iterative/auto | unsafe replay区域走安全exact，sequence按原契约每行一次；比较计数/序列状态，不要求两次RAND值逐个相等。 |
| FVS-061 | snapshot/权限 | producer与回表之间并发更新；多账户同名表、B权限变化控制 | 遵循语句快照和既有授权边界，域不能来自另一个独立事务/租户；不宣称新增跨语句一致快照。 |
| FVS-062 | 双CN入口 | CN1/CN2各跑原fixture、periodic、NULL、prepared及cap组合，重复并交替 | 相同快照精确/确定性fixture一致；ANN按合法性/召回/行数验证，不硬编码全部近似查询跨CN逐ID相同。 |
| FVS-063 | 复合SQL隔离 | UNION ALL相同/不同分支、同语句多个向量索引、ordinary JOIN控制 | alias/runtime filter/knob/state按分支独立；UNION ALL应保留合法重复，不把跨分支重复当单次搜索去重对象。 |
| FVS-064 | 冷启动/失败恢复 | 服务重启、索引冷加载、prepare后drop/rebuild、失败后再次execute | 选择有效generation，按旧依赖规则重规划/明确失效，不复用已释放mask/domain；查询与连接可继续正常工作。 |

### 3.7 性能、召回与长稳（6条）

| 编号 | 二级模块 | 场景与步骤 | 预期结果/记录项 |
| --- | --- | --- | --- |
| FVS-065 | 模式对比 | 四算法default/post/pre/iterative/include/auto/force，PK/INCLUDE/普通列分组 | 同snapshot/查询投影/资源测延迟、QPS、行数、recall；include机制组必须建含列索引；FORCE为CPU基线。只比较速度不能证明过滤完成。 |
| FVS-066 | 选择率/分布 | 100%/10%/5%/1%/0.1%及零；随机、clustered、最近全不合格；K/O矩阵 | 记录初始/最终knob、轮次/分支、P、short-page率、recall；重现100k×128维评论形态但不复用未取得脚本的数值。 |
| FVS-067 | GPU mask收益 | 大域、小域及各C表示；1M×768按硬件资源受控执行，单轮/多轮对照 | 拆分域构建、host mask、upload、GPU search、回表及最终排序，证实mask每搜索准备一次；PR已有ms只作历史线索。 |
| FVS-068 | 扩搜成本 | HNSW多子索引、IVFPQ重复探测、IVF cursor、DOP拆分、CAGRA宽度 | 报告总轮次/累计工作而非只最后一轮；峰值CPU/GPU/C内存及准入；split单轮短页作为能力边界呈现。 |
| FVS-069 | ANN基准与质量 | 固定公开L2/相应距离数据、查询与seed，补关系eligibility和精确ground truth | 使用ANN-Benchmarks数据/驱动扩展时明确过滤逻辑；原无过滤ground truth无效。满页率与recall分别报告，同质量/预算才比较加速。 |
| FVS-070 | 长稳 | 多session持续2小时，混合prepare、SET、DML、取消和双入口，静止点核对 | 无泄漏/死锁/崩溃/持续性能退化；记录错误分类及质量趋势，不比较不同并发快照为同一结果。 |

## 4. 判定方法与证据要求

### 4.1 三种不同的正确性结论

1. **过滤/可见性正确性**：返回行均满足原关系条件、membership和可见性，PK在单次实体检索结果内不重复，非UNION ALL输出不得因多轮/多子索引重复返回。任何模式泄漏不合格行都不接受。
2. **页完成度**：以同快照精确窗口P为分母，`fill_ratio=actual_rows/P`；P=0时记N/A并要求actual=0。`actual<P`是事实；再按算法、mode、cap、reader形态判断是实现缺陷、交付已声明边界，还是原需求未满足。不能把未探测完的索引称为“合格集合耗尽”。
3. **近邻质量**：独立计算过滤后原向量精确距离并排序，`recall=合法返回ID与精确窗口ID交集数/P`。不足额与选错近邻都可降低recall；另报返回数及precision/非法行数，避免混淆原因。IVFPQ/量化索引报告对原向量的质量损失，不把PQ全探测当exact。

只有精确FORCE/无索引控制或明确可穷举的确定性fixture要求完整精确窗口等价。原表count、向量有效行数、实际索引项数、membership成员数分别记录；NULL、删除、异步尚未索引成员不能混在同一Q里。

### 4.2 排序、同分与分页

核对距离方向和函数语义，`inner_product`按负点积升序取最近。只有距离排序时，不要求等距离PK固定顺序；校验器按距离组比较，窗口切过同分组时验证合法组配额，而不是给被测SQL添加第二排序键改变选路。

精确分页必须在过滤和全局合并之后应用O/K。ANN不同O可能触发不同预算而改变近邻集合，不强求所有独立ANN分页拼起来必然等于一次大K查询；同快照exact控制必须符合切片。多轮返回用实体PK去重，但UNION ALL或合法JOIN重复语义保持原SQL契约。

需要距离容差时按vecf32/相应类型固定容差并报告，精确边界优先二进制可表示输入；不能用过宽容差把WHERE泄漏或错误顺序当浮点误差。

### 4.3 扩搜与adaptive的确定性证据

| 机制 | 建议直接证据 | 不足以证明的现象 |
| --- | --- | --- |
| HNSW/CAGRA重启 | UT记录StartRound/NextRound、每次itopk、取消与终止；共享index参数不变 | 只看最终K行不能证明扩搜或无状态污染。 |
| IVFPQ扩probe | 实际built NLists、各轮nprobe、终轮标志，overflow-only首轮停止 | 只看catalog lists会漏掉generation差异。 |
| IVF-FLAT连续cursor | list访问序列为无重复子序列/最终permutation，round_count与cursor推进 | 单次调用成功不能证明没有重扫list。 |
| adaptive | 各支启动/完成/输出计数、短页触发下一支、full支胜出 | EXPLAIN列出三个分支不等于三支都实际执行。 |
| GPU prepared mask | prepare/host pass/upload次数与search轮次，C owner引用及内存归还 | 最终结果正确不能证明每轮没有重复构建mask。 |
| required membership | producer发布/reader使用的屏障，缺域报错，空域与无域区别 | 不能以sleep后查询成功代替发布顺序测试。 |

cap限制必须在实际request和轮次里核对；EXPLAIN通常只体现SQL静态参数。session逐执行变化通过request/UT、远端消息副本和SQL效果共同证明。

### 4.4 性能测量

- 至少3轮独立运行，记录冷/热缓存、建索引参数、训练、数据seed、CPU/GPU型号、内存、实际topology与统计信息；固定查询集、投影、并发及数据快照。
- 输出每种算法/mode的p50/p95/p99、QPS、实际P与返回数、short-page率、recall、错误率、轮次/终止、CPU/C/GPU内存、准入等待及可得的扫描/传输指标。
- 选择率要有不同空间分布，尤其“最近全部不合格”。仅随机分布的5%不能覆盖prefix/clustered失败形态。
- 按同recall/满足率目标比较速度，或给质量-延迟曲线；不能把固定post少返8行的1ms直接宣称优于完整页的5ms。
- 外部pgvector及issue评论只作线索。比较其它数据库须统一距离、向量数据、过滤、K/O、投影、硬件与参数，实际获取/审阅脚本后才可复现实验，不引用历史0.18/0.998作为新结论。
- GPU unavailable只标环境阻塞，不能把CPU构建或force运行填成GPU通过。性能阈值和召回目标执行前约定；没有阈值只报告测量事实。

## 5. 已有测试复用映射

以下入口按PR合入内容核对。路径默认相对MatrixOne仓库，CPU SQL在`test/distributed/cases/`、GPU在`test/distributed/gpu_cases/`。映射表示可复用的直接证据入口，不声称单个函数已覆盖设计行全部变体；执行报告需填实际测试ref、命令、平台和未覆盖项。

| 设计用例 | 已有入口 | 复用范围及待补内容 |
| --- | --- | --- |
| 001..010、011..020、031 | `cases/vector/vector_filtered_search_hnsw.sql`、`vector_filtered_search_ivfflat.sql`、`vector_filtered_search_modes.sql`及各自`.result` | 模式/缺页/NULL/分页/auto；两个CN和额外选择率组合需保留本轮结果。 |
| 013..018、021、022、025、034 | `gpu_cases/vector/vector_filtered_search_gpu.sql`、`vector_hybrid_fulltext_gpu.sql`、`vector_cagra_itopk_limit.sql` | GPU内过滤/membership、距离范围、PQ/图质量、cap；不可复制CAGRA不稳定ID为固定oracle。 |
| 021、060 | `cases/vector/vector_hybrid_fulltext.sql`；`pkg/sql/plan/vector_fulltext_membership_test.go`中的`TestVectorPreFulltextMembershipSearchesOnce`；`vector_pre_replay_test.go`中的`TestVectorPreVolatileFilterRunsOnce` | filter-only MATCH单次执行及volatile exact回退；补真实SQL状态/输出检查。 |
| 023、024、026..030、033 | 四算法`pkg/vectorindex/<algo>/iterative_test.go`；HNSW/IVF的StartRound/NextRound，CAGRA/IVFPQ的`TestIterativeSearchRounds`及IVFPQ `TestIterativeSearchTerminates`；`TestCompleteCentroidRanking`、`TestPlanReaderWidensMembership` | 轮次预算、终止、cursor；函数同名按所属包区分。真实split-reader与多子索引另留physical证据。 |
| 035..042 | `cases/vector/vector_filtered_search_knobs.sql`；`gpu_cases/vector/vector_filtered_search_knobs_gpu.sql`；`TestParseRankOptionSearchKnobs`、`TestApplySearchKnobs`、`TestApplySearchKnobsSessionFallback`、`TestApplySearchKnobsInternalTopKLimit` | 参数解析、优先级、适用范围、session与CAGRA上限；补并发session及实际远端执行。 |
| 027、041 | `TestInlineVectorIndexWithoutOptionsUsesParamsFromTree`；`TestIvfSearchIncludeModeOversizedRoundLimit`、`TestSortAndLimitExactResultsOversizedLimit`、`TestScanEntriesOversizedLimitKeepsStorageTopK`；`gpu_cases/pessimistic_transaction/vector/vector_ivfpq_iterative_overflow.sql` | inline无lists、uint64最大预算、overflow-only终止，确认真实GPU路径。 |
| 043、044 | `pkg/sql/compile/vector_scan_protocol_test.go`中的`TestSearchKnobScopeRefusesPriorEpochDestination`；`TestStampSessionKnobs`、`TestSourceNodeForRemoteRunStampsSessionKnobs`、`TestRequiredProtocolVersion` | placement/发送重探/handshake/decode及无knob控制；不扩大为整个混合版本拓扑支持。 |
| 045..048、052 | `pkg/common/docfilter/cmembership_test.go`中的`TestNewForCPicksByEstimate`、`TestNewForCAdmissionDenied`、`TestCMembershipOf`、`TestCExactMembershipOf`；`TestFilteredSearchMembershipRejectsGoSorted64`、`TestAdmitFilterMask` | C表示、准入、错误类型；C/C++端结果与内存生命周期需结合直接consumer证据。 |
| 048..053 | `cgo/cuvs/test/filter_test.cu`、`cagra_test.cu`、`ivf_pq_test.cu`中的`EvalFilterMembershipTest`、`FilteredSearchAppliesMembership`、`FilterEmptyInProbedListsPadsWithSentinels`及`PreparedBitsetExcludesRowsAddedAfterPrepare` | mask交集、sentinel、prepare后新增；名称含C++fixture的按实际test列表定位，不误当Go函数运行。 |
| 049、050、057 | `pkg/cuvs/prepared_bitset_gpu_test.go`中的`TestGpuCagraPreparedBitset`、`TestGpuIvfPqPreparedBitset`、`TestMultiGpuCagraPreparedBitset`、`TestMultiGpuIvfPqPreparedBitset`；`TestCreateEndWithoutRows` | GPU bitset准备/复用/shard与空pipeline；race与内存检查按硬件构建条件执行。 |
| 055..064 | `gpu_cases/pessimistic_transaction/vector/vector_filtered_search_overflow_gpu.sql`、`gpu_cases/vector/vector_sharded_filtered.sql`及现有vector依赖/取消测试 | DML、overflow、shard；缺少新路径直接覆盖的取消/授权/snapshot组合须新增或标未覆盖。 |
| 065..070 | 专项benchmark驱动、固定过滤ground truth和长稳脚本 | PR性能报告与156文件历史通过不替代本轮测量；公开数据源/驱动及自定义过滤映射需归档。 |

建议执行顺序：冻结需求/交付矩阵与环境 → CPU模式/历史fixture及独立oracle → 参数/终止/协议/C资源UT → 实机GPU与C++直接consumer → BVT结果审阅后正常比较两轮并确认清理 → 两CN/异步overflow/取消/snapshot → 大规模性能与长稳。测试工具链、CGo和GPU构建按验收ref仓库说明执行。

正向集成使用支持相应algo_options、membership及knob的兼容节点和匹配原生库；旧版本拒绝在隔离的协议/兼容测试中验证。尤其无knob的iterative可能被旧节点按单轮解释，不能只验证V127 knob fence就宣称完整feature可混合部署。

## 6. 结果记录与验收出口

每个用例/参数组合填写：未执行、通过、失败或阻塞；另列需求差异状态。应附如下证据：

| 类别 | 必填信息 |
| --- | --- |
| 基线/构建 | MatrixOne SHA、镜像digest、libmo/cuVS/CUDA、Go/工具链、CPU/GPU、配置及实验开关、CN/TN/Log拓扑和入口。 |
| 数据/索引 | 行数及有效向量数、向量类型/维度、op_type、量化/训练、lists/graph/capacity、实际子索引/shard/NLists、built/overflow行数、PK分布、INCLUDE列。 |
| 查询/预算 | 全SQL/参数、mode、K/O、谓词/选择率、SQL knob、session值和读取时点、最终effective request、静态/实际计划。 |
| 结果/质量 | 同快照Q/P、actual、非法/重复PK、排序/tie处理、fill_ratio、recall、终止/回退原因和首次不足额证据。 |
| 生命周期 | 轮次/分支、域发布/引用、mask prepare/upload、内存准入和归还、取消及错误日志、索引generation/快照同步点。 |
| 性能 | 各轮原始样本、p50/p95/p99、QPS、CPU/C/GPU峰值内存、构建/传输/搜索分项、质量与预算对照。 |
| 清理 | 独占库/索引/sequence、prepared statement、连接及临时资源清理，保存的session设置恢复；不删除共享业务对象。 |

验收分两层：

- **交付实现验收**：按已冻结的模式/算法/参数矩阵验证，无过滤泄漏、无无界迭代、协议与内存失败不静默降级，承诺的确定性fixture有直接证据。
- **原需求完成度**：明确post/default不自动完成、caps/split-reader短页与可观测性缺口是否满足或仍偏离原issue；未获明确范围接受时保留差异，不能仅以所有交付用例PASS关闭需求。

本次文档完成标准：70条编号连续、表格/链接结构检查通过、手算fixture经过独立核对、UT/BVT映射来源可定位。数据库/GPU/性能执行结果另报，历史材料不自动标为本轮通过。
