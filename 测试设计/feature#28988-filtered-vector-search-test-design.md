# MatrixOne 多索引过滤向量检索测试设计（必要项二次评审版）

> Feature [#28988](https://github.com/matrixorigin/matrixone/issues/28988)：HNSW、IVF-FLAT、CAGRA、IVFPQ 的过滤向量检索。
>
> 修订日期：2026-10-28。实现基线为已合入 [PR #30127](https://github.com/matrixorigin/matrixone/pull/30127)，merge commit `60ad9a14fc6eda33b10ce74226078de3cffd24b2`。
>
> 本轮将上版表内52个执行行（涉及62个原编号）精简为 **38个执行单元**。每行使用单一主编号，合并项成为必须断言；修正上版参数章节的编号/语义错位，保留模式、算法特有终止、C/GPU资源和协议门槛。本文件是测试设计，不代表已执行结果。

## 1. 验收范围与裁剪原则

### 1.1 必要测试的判定

仅保留满足下列至少一项的验证：

1. 直接对应 Issue 的验收条件：模式矩阵、不静默短页、`EXPLAIN` 可观测、literal/prepared 一致、零/少于/恰好/多于 K、双 CN。
2. 防御本 PR 引入的高危缺陷：过滤泄漏、无界或无进展迭代、错误的终止上限、协议 fence、C 内存或 GPU mask 生命周期、索引 generation 失效、panic/overflow。
3. 证明四种索引家族的特有实现路径；不以某一算法的通过替代另一算法的终止或过滤实现。

不为同一机制重复展开所有距离、K 值、硬件和谓词排列。别名、优先级、去重/满页终止、协议适用范围等改为所属主用例的**必须断言**。性能、长稳和跨产品比较不是本 Feature 的合入门槛。

### 1.2 需求与交付差异必须单列

| 原 Issue 目标 | 合入交付行为 | 验收记录要求 |
| --- | --- | --- |
| post 过滤后补候选直至满足或耗尽 | `post` 单次搜索；扩搜由 `iterative` 负责 | 分别记录交付矩阵结果与原需求差距；短页不可被误标为“需求完成”。 |
| auto/default 自动选路 | 显式 `auto` 为 ADAPTIVE_TOP 或安全 force；未设 mode 保持兼容行为 | `default` 与 `auto` 分别验证。 |
| 不能保证完整页时回退或等价完成 | iterative 受算法 cap、索引行数和执行形态约束 | 记录 cap、终止原因和短页事实；不得把“提高参数”写成无条件保证。 |
| 计划可观测 | SQL knob 与 Index Filter 可由 EXPLAIN 观察；多数运行期轮次不能仅由 EXPLAIN 证明 | 用 EXPLAIN 加 request/UT/运行证据验证；不得编造字段。 |

任何最终报告必须同时给出“交付实现是否通过”和“原 Issue 是否仍存在未接受的范围差异”。用例裁剪不改变这项义务。

### 1.3 模式与算法契约

| 模式 | 必要验证结论 |
| --- | --- |
| default | 保持既有兼容行为；可记录短页，不能假定等同 auto。 |
| post | 单轮索引搜索，结果必须满足过滤；不承诺迭代补满。 |
| pre | 可下推条件在索引或 membership 中执行；一次搜索不等于完整召回。 |
| iterative / include | `include` 是 `iterative` 别名；按算法扩搜并在 cap、索引耗尽、页目标、取消等条件终止。 |
| auto | 验证 post/pre/force 的安全分支和短页后续分支；满页 ANN 不等于精确排名。 |
| force | 绕过向量索引、CPU 精确排序；用于同快照 oracle，不冒充 GPU 索引验证。 |

HNSW、IVF-FLAT、CAGRA、IVFPQ 都须覆盖可用模式的解析和关键语义。算法特有的迭代终止分别验证：HNSW 扩 `itopk`、CAGRA 的 1024 上限、IVFPQ 扩 `nprobe` 与 built NLists、IVF-FLAT 连续 centroid cursor。

## 2. 数据、oracle 与通用断言

### 2.1 最小共享 fixture

使用独占数据库及无向量索引的等价表作为精确控制。核心数据为 1,000 行单调 L2 向量，前 500 行不满足 `category=1`、后 500 行满足；精确首页为 501..510。另建：

- cosine 与 inner-product 的单调 HNSW 控制各一条，验证历史三种距离算子均不再被遗漏；
- `id % 7 = 0` 的周期选择率、`IN`、NULL、组合 AND/OR、零命中、少于 K、恰好 K；
- IVF-FLAT 的有/无 INCLUDE 列副本，列表数 4 或 8；
- GPU 的 CAGRA/IVFPQ 固定小数据与真实建成索引；IVFPQ 显式 lists；
- HNSW 多子索引、GPU built + CDC overflow、shard 仅在对应算法路径执行。

GPU 结论必须来自真 GPU 与匹配 libmo/cuVS 构建；CPU 回退或 force 不能标为 GPU 通过。GPU 环境不可用时状态为“阻塞”，不是“通过”。

### 2.2 Oracle 与判定

令同快照、满足完整 WHERE 且有有效距离的行数为 Q，目标页为 `P=min(K,max(Q-O,0))`。每项记录 Q、P、实际返回数、非法/重复 PK、模式、有效 knob、终止或回退原因。

1. **过滤与可见性**：每个返回行必须满足原 WHERE、membership、删除状态和租户/快照约束。
2. **页完成度**：`actual/P` 只描述事实；再按 mode、算法和 cap 判定它是缺陷、已声明边界，或原需求差距。
3. **近邻质量**：以同快照精确窗口计算 recall。仅 force、无索引控制或明确可穷举 fixture 要求逐 ID 等价；IVFPQ/量化索引不要求等于原始 f32 精确排序。
4. **迭代证据**：最终满 K 不是扩搜的证据。HNSW/CAGRA/IVFPQ 记录轮次 knob；IVF-FLAT 记录不重复 list cursor；auto 记录实际分支。
5. **排序与分页**：精确控制在过滤、合并后应用 offset/limit；同距离按距离组判断，不能通过添加第二排序键改变被测语义。

### 2.3 本轮统计

| 模块 | 执行单元 |
| --- | ---: |
| 模式与策略 | 5 |
| 过滤、数量与距离 | 6 |
| 扩搜、终止与分页 | 8 |
| 参数、PREPARE与协议 | 5 |
| C membership/GPU mask | 6 |
| 生命周期与分支 | 8 |
| **合计** | **38** |

## 3. 必要执行项

每行一个主编号/执行单元，复用同一fixture完成相邻控制和合并断言；不再为模式别名、零值、session隔离等重复装载数据。保留各算法独立终止和直接C/GPU路径。

### 3.1 模式与策略（5项）

| 编号 | 场景与必要断言 |
| --- | --- |
| FVS-001 | 四算法default/post及force控制：省略mode不等于auto；post只搜一次，PK/INCLUDE过滤位置按矩阵，force精确CPU oracle。合并002/007，不另跑全套相同数据。 |
| FVS-003 | pre的INCLUDE/非INCLUDE过滤；仅真正服务查询的索引校验能力，未服务索引不误拒绝。吸收009，不扩全部多索引排列。 |
| FVS-004 | iterative的资格内过滤、扩搜与停止；include作为iterative别名在同fixture对照，不误作独立策略。 |
| FVS-006 | auto三种必要分支（post满、post短/pre满、前两支短）；另用多输出/外层聚合代表性安全force控制。无用pre支可省略；volatile复用060，不为每种SQL包裹单建矩阵。 |
| FVS-010 | IVF-FLAT post+INCLUDE的空页force兜底、非空短页、满页控制；不能把空页行为推广成全部短页自动补齐。 |

### 3.2 过滤、数量与距离（6项）

| 编号 | 场景与必要断言 |
| --- | --- |
| FVS-011 | 原近端全不合格红例与id%7周期分布，双CN抽测；同fixture保留K9/10、49/50历史边界。pre/iterative按能力和cap验收，default/post短页仍单列需求差异，不能统一要求“不再短页”。 |
| FVS-012 | L2覆盖全部适用算法，cosine/IP各一个HNSW单调控制；能力不支持明确记矩阵，不用错误op_type伪造覆盖。 |
| FVS-014 | 同值列一个进INCLUDE、一个不进；证明索引内过滤与membership/residual区别。INCLUDE机制必须建含列索引，不能只写mode=include。 |
| FVS-015 | 同fixture保留PK范围/IN、组合AND/OR、NULL谓词及IN/SEMI；负/稀疏key编码复用045直接UT。现有hybrid-fulltext BVT必跑，filter-only MATCH在producer单次执行；不扩完整谓词排列或任意INNER JOIN能力。 |
| FVS-018 | 一组距离范围含/不含边界，source条件不丢；距离谓词不被误建为普通PRE域，PQ/量化排名与原WHERE合法性分开。 |
| FVS-019 | Q=0、少于K、恰好K、多于K及空表/空域，共用数据控制；P计算正确，空域不当PASS，不能无限凑K。满页与精确召回是两件事，吸收020/034；NULL/未索引成员数不混成可搜索Q。 |

### 3.3 扩搜、终止与分页（8项）

| 编号 | 场景与必要断言 |
| --- | --- |
| FVS-023 | HNSW多子索引上小/大itopk及cap，确认全局合并/去重、预测起点、逐轮推进和有界结束；不污染共享ef_search。合并024及该算法033无进展断言。 |
| FVS-025 | CAGRA表>1024行且合格<K；itopk/max_itopk SQL1024接受、1025拒绝，session超限截到1024；无历史cuVS越界和无界扩搜，恢复原038的CAGRA语义到此主项。 |
| FVS-026 | IVFPQ以实际built NLists为上限，probe扩大/饱和、cap和行数终止；加入033无进展/重复候选控制，不只看catalog lists。 |
| FVS-027 | IVFPQ overflow-only首轮搜索完即终止；inline无lists与CREATE INDEX同样拒绝，不能建成后无限迭代。 |
| FVS-028 | IVF-FLAT cursor推进、list不重扫、零向量/同分质心完整permutation；合并033重复/无进展控制。 |
| FVS-029 | IVF-FLAT exact membership 100/101边界，纯INCLUDE同样少量匹配作控制；不能误套小精确域全list保证。 |
| FVS-030 | local DOP/多reader拆分形态；保留真实physical plan，按交付单轮/短页边界记录，不假设单reader结果证明分布式补满。 |
| FVS-031 | K/O=10/10与10/500，追加K0及提前满页控制；offset仅对过滤合并结果生效，零页不发无效搜索、不挂起。算法预算溢出通过现有UT断言覆盖，不另建大OFFSET压力数据。 |

### 3.4 参数、PREPARE与协议（5项）

参数编号恢复原设计含义：039/040对应执行期session/优先级，原037的范围校验并入035，原038的CAGRA上限并入025。

| 编号 | 场景与必要断言 |
| --- | --- |
| FVS-035 | 四SQL knob的解析/适用族，unknown key、非正整数、cap<start、nprobe1/80000/80001；graph不接受IVF probe knob，实际适用族中mode不生效的合法选项可接受。合并036/037。 |
| FVS-039 | PREPARE后SET三个vector_*、执行、reset；SQL>session>index/default优先级及两个session隔离一并验证。probe_limit保持原计划期行为，不错当三个执行期变量；不写回共享plan。 |
| FVS-041 | 最大uint64 itopk的实际行数/分配边界与storage Top-K列，HNSW/IVFPQ/IVF-FLAT不panic，CAGRA按1024上限拒绝。 |
| FVS-042 | EXPLAIN显示SQL参数/Index Filter，session有效值用request/UT佐证；实际轮次、分支和终止不能从静态EXPLAIN臆断。 |
| FVS-043 | 合入基线V127新增knob scope在placement、发送re-probe、handshake/decode拒绝prior版本；无knob按V111基础协议控制。远端副本只stamp适用参数，非向量扫描不抬版本；不能据此推断旧节点会正确执行无knob的iterative。 |

### 3.5 C membership与GPU mask（6项）

| 编号 | 场景与必要断言 |
| --- | --- |
| FVS-045 | cbitmap/CRoaring/C sorted64表示选择、负/稀疏key编码、非法元素宽度；GPU拒绝Go-owned/non-exact或缺host_ids，空exact表示拒绝全部。合并046直接错误控制。 |
| FVS-047 | membership与mask/delete copy准入门槛/拒绝、共享引用及最后Free；失败不部分执行或静默换CPU，合并052内存断言。 |
| FVS-048 | membership AND INCLUDE/PK AND删除位，shard切片正确；prepare前删除排除，轮间删除按已声明快照契约，吸收051。 |
| FVS-049 | CAGRA/IVFPQ每index/shard/overflow准备一次bitset、多轮复用；prepare/host pass/upload次数与释放有直接证据。 |
| FVS-050 | prepare后追加行不能因mask尾部默认全1通过；清尾位及copy限制shard范围，复用直接C++/GPU测试。 |
| FVS-053 | IVFPQ所探list无合格项的padding/sentinel正确剔除，非法ID/距离不泄漏，不出现vector magnitude错误；首轮空不是全索引耗尽。 |

### 3.6 生命周期与分支（8项）

| 编号 | 场景与必要断言 |
| --- | --- |
| FVS-055 | INSERT/DELETE/UPDATE普通/INCLUDE/向量列及一条rollback，在既有同步屏障后资格/存活性正确；不重复全部事务排列。 |
| FVS-056 | CAGRA/IVFPQ built+CDC overflow，PK/subquery/membership按交集过滤后全局合并，无重复/遗漏。 |
| FVS-057 | async REINDEX空pipeline与shard切换，无rows不panic，generation/host_ids/mask一致。 |
| FVS-058 | 缺required域、构建/设备失败、空域及跨轮取消，各路径复用生命周期fixture；停止及时、错误不伪空成功、资源归还，后续请求正常。合并059，锁持有仅附代码核对，不另建054项。 |
| FVS-060 | sequence/volatile filter及membership join危险重放走安全exact，按语义单次求值；filter-only MATCH复用015证据。 |
| FVS-061 | 一条producer/回表并发更新的语句快照控制；保留同快照，不新增权限矩阵。 |
| FVS-063 | UNION ALL双分支向量搜索，runtime filter/knob/state各自独立，合法重复不被去重。 |
| FVS-064 | prepare后一次drop/rebuild依赖控制，重规划或明确失效，不复用旧generation/domain/mask。服务重启、全量冷启动不在必要项。 |

## 4. 执行批次与门槛

1. **CPU批次**：第3节所有适用HNSW/IVF-FLAT模式、过滤、参数、两种迭代终止与protocol/C资源UT；双CN红例及PREPARE/session抽测并入011/039/043，不另设062全矩阵。030 split-reader边界不得漏执行。
2. **GPU批次**：第3节所有适用CAGRA/IVFPQ模式和参数、各自终止、prepared mask、built+overflow、shard及取消；GPU在交付范围内即为功能门槛，没有硬件记阻塞，CPU/force不能替代。
3. **复用证据**：hybrid-fulltext、snapshot/volatile、UNION ALL及索引依赖回归按015/060/061/063/064记录，不因合并而变成可选项。

不安排性能冒烟、ANN benchmark、跨产品比较、长稳或服务重启。轮次与内存只作为功能执行证据记录，不扩成收益测量。

## 5. 裁剪及可复用证据

### 5.1 编号处置

保留38个主编号，不再把“涉及的原编号数”“表格行数”和“约执行次数”混算。合并关系：
002/007→001；005→004；008→006；009→003；013/017/021/022→015；016/062→011（prepared双入口另附039）；020/034→019；024→023；032→031；033→023/026/028；036/037→035；038→025；040→039；044→043；046→045；051→048；052→047；059→058/019。

原054锁细节仍只做代码核对；065..070及服务重启/权限扩展不列必要执行项。保留原需求与交付差异的验收义务。

### 5.2 直接复用入口

路径相对MatrixOne，CPU位于`test/distributed/cases/`，GPU位于`test/distributed/gpu_cases/`。

| 保留项 | 入口/证据 |
| --- | --- |
| 001/003/004/006/010/011/012/014/015/018/019 | CPU `vector/vector_filtered_search_{hnsw,ivfflat,modes}.sql`，GPU `vector/vector_filtered_search_gpu.sql`及相应result；花括号表示三个实际文件，不是单一文件名。 |
| 015/060 | CPU/GPU `vector/vector_hybrid_fulltext*.sql`；`TestVectorPreFulltextMembershipSearchesOnce`、`TestVectorPreVolatileFilterRunsOnce`。 |
| 023/025/026/027/028/029/030/031 | 四算法iterative UT；IVFPQ `TestIterativeSearchTerminates`；`TestCompleteCentroidRanking`；`vector_cagra_itopk_limit.sql`及`vector_ivfpq_iterative_overflow.sql`。多reader物理形态另留SQL证据。 |
| 035/039/041/042/043 | CPU/GPU `vector_filtered_search_knobs*.sql`；`TestParseRankOptionSearchKnobs`、`TestApplySearchKnobs`、`TestStampSessionKnobs`、`TestSearchKnobScopeRefusesPriorEpochDestination`；scope协议基线按实现SHA冻结。 |
| 045/047/048 | `TestNewForCPicksByEstimate`、`TestNewForCAdmissionDenied`、`TestFilteredSearchMembershipRejectsGoSorted64`、`TestAdmitFilterMask`；C++ `EvalFilterMembershipTest`及`FilteredSearchAppliesMembership`。 |
| 049/050/053 | `TestGpuCagraPreparedBitset`、`TestGpuIvfPqPreparedBitset`及multi GPU对应测试；C++ `PreparedBitsetExcludesRowsAddedAfterPrepare`、`FilterEmptyInProbedListsPadsWithSentinels`。 |
| 055/056/057/058/061/063/064 | GPU overflow/sharded BVT、`TestCreateEndWithoutRows`及现有generation/cancellation/依赖回归；缺新路径直接snapshot或取消覆盖时补场景或报缺口，不以历史通过替代。 |

每行映射只作入口，实际报告记录准确文件/测试名和触发路径；相同名字在不同算法包须区分。不能只运行一个算法就填满其它算法状态。

## 6. 证据与结果出口

38个执行单元逐项记录：MatrixOne SHA、运行环境与 GPU/原生库版本、数据与索引实况、完整 SQL/参数、Q/P/actual、过滤/重复检查、有效请求 knob、EXPLAIN 与实际执行证据、轮次/分支/终止原因、错误和资源清理。

验收输出必须分两层：

1. **交付实现验收**：承诺的 mode/算法/参数组合无过滤泄漏、无无界迭代、协议或内存失败不静默降级，确定性 fixture 有直接证据。
2. **原需求完成度**：显式说明 post/default 短页、cap、split-reader 以及可观测性限制是否仍偏离 Issue；未获范围接受时，不因全部保留测试通过而关闭需求。

已有测试映射仅作为可复用证据入口；每轮报告必须写入实际执行 ref、命令、平台、通过/失败/阻塞和未覆盖项。
