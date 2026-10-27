# MatrixOne 多索引过滤向量检索测试设计（必要验证精简版）

> Feature [#28988](https://github.com/matrixorigin/matrixone/issues/28988)：HNSW、IVF-FLAT、CAGRA、IVFPQ 的过滤向量检索。
>
> 修订日期：2026-10-27。实现基线为已合入 [PR #30127](https://github.com/matrixorigin/matrixone/pull/30127)，merge commit `60ad9a14fc6eda33b10ce74226078de3cffd24b2`。
>
> 本版以“必要测试”为准则：保留需求验收、高危实现路径和已知回归；移出性能基准、收益评估及长稳专项。原设计的 70 个编号被裁剪为 **52 个保留编号**，并通过共用数据与同次执行的断言组织为约 **40 个有效执行单元**。本文件是测试设计，不代表已执行结果。

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

## 3. 必要执行项

下表中的编号沿用原设计，便于关联现有 BVT/UT/C++ 测试。若同一行列出多个编号，表示它们在同一 fixture 或执行单元内完成，而不是分别新增一套数据。

### 3.1 模式与策略（8 项）

| 保留编号 | 场景与必要断言 |
| --- | --- |
| FVS-001 | 四算法未设 mode 的兼容行为；INCLUDE 与非 INCLUDE 分组，记录短页而不把 default 当 auto。 |
| FVS-002 | `post` 的 PK、INCLUDE、普通列过滤；确认单次搜索与无过滤泄漏。 |
| FVS-003 + FVS-009 | `pre` 的 INCLUDE/非 INCLUDE 过滤和实际服务索引能力；距离不匹配或实际使用的不支持组合被正确拒绝，未选索引不误拒绝。 |
| FVS-004 + FVS-005 | `iterative` 的近端不合格 fixture；确认扩搜、终止及 `include=iterative` 别名等价。 |
| FVS-006 | `auto` 的 post 满页、post 短/pre 满页、前两支短三种分支；不要求满页 ANN 必然等于 force 的 ID。 |
| FVS-007 | `force` 在 CPU/GPU 环境均走精确 CPU oracle，而非索引/GPU 搜索。 |
| FVS-008 | 多输出、聚合包裹、特殊 membership/volatile 等不安全区域安全 force 或省略不必要分支；保持 SQL 语义。 |
| FVS-010 | IVF-FLAT 的 post + INCLUDE 例外：空页、非空短页、满页；不得把空页 force 兜底外推为所有短页补齐。 |

### 3.2 过滤、数量与距离（10 项）

| 保留编号 | 场景与必要断言 |
| --- | --- |
| FVS-011 | 原 Issue 红例在两 CN：HNSW category=1；比较 default/post/pre/iterative/auto/force，保留历史 0/10 与精确 10/10 的动机控制。 |
| FVS-012 | L2 覆盖全部适用算法；cosine、IP 各保留一个 HNSW 单调回归。能力不支持时记录矩阵，不用错误 op_type 伪造覆盖。 |
| FVS-013 | PK 范围、IN、负数/稀疏 BIGINT；防止映射或小 membership 丢行/泄漏。 |
| FVS-014 | 同值列一列进入 INCLUDE、一列不进入 INCLUDE；证明索引内过滤与 membership/residual 的真实差异。 |
| FVS-015 + FVS-022 | 混合 PK/INCLUDE/普通列的 AND/OR/NULL 与 IN/SEMI 资格；每个结果满足完整 WHERE，保留原 SQL multiplicity。 |
| FVS-016 | 周期选择率和近端不合格数据；在同一执行单元保留历史预算边界 `K=9/10` 与 `K=49/50`，检查不再出现固定倍数阈值短页。 |
| FVS-017 | NULL 谓词、NULL 向量/参数/键；遵循 SQL 三值逻辑与索引可见性。 |
| FVS-018 | 一组距离范围边界（含/不含边界）；距离谓词不可被误建为普通 PRE 域。 |
| FVS-019 | Q=0、Q=3、空表/空域；不为凑 K 无限取数，缺 required 域是失败而非空集。 |
| FVS-020 + FVS-034 | Q=K 与 Q≫K；符合契约时完成路径返回 P，并断言“满页”不被误判为精确召回。 |

现有 hybrid-fulltext BVT 不再新设 FVS-021 执行项，但必须作为本 PR 已改变 PRE/membership 路径的**复用回归证据**；其结果记录在覆盖映射中。

### 3.3 扩搜、终止和分页（10 项）

| 保留编号 | 场景与必要断言 |
| --- | --- |
| FVS-023 | HNSW 多子索引；确认实际子索引数、跨子索引全局合并、PK 去重和只读 membership 共享。 |
| FVS-024 + FVS-033 | HNSW 小/大 itopk、max_itopk 和 membership 预测起点；每轮推进、有界停止、无进展/重复候选不得循环，且不污染共享 ef_search。 |
| FVS-025 | CAGRA 超过 1024 行和合格数不足 K；不向 cuVS 发超上限 key 预算，合法短页在 cap 停止。 |
| FVS-026 + FVS-033 | IVFPQ 以实际 built NLists 而非 catalog 值上限；nprobe 饱和、索引行数、cap 和无进展均终止。 |
| FVS-027 | IVFPQ overflow-only 首轮即耗尽；inline index 缺 lists 与 CREATE INDEX 一致拒绝。 |
| FVS-028 + FVS-033 | IVF-FLAT 连续 centroid cursor；列表不重扫、无重复 PK、最终访问序列正确。 |
| FVS-029 | IVF-FLAT 小精确 membership 只保留 100/101 边界；前者全 list 精确路径，后者不误套小域保证。 |
| FVS-030 | 多 reader/分区拆分下的完成边界；按交付契约记录单轮短页，不能宣称无条件完成。 |
| FVS-031 | 分页仅保留 10/10 与 10/500；精确控制符合切片，ANN 预算变化不被误当分页拼接缺陷。 |
| FVS-032 | offset、limit=0 和页目标已达；limit=0 不发起无效搜索，提前满足即停止。 |

### 3.4 参数、PREPARE 与协议（8 项）

| 保留编号 | 场景与必要断言 |
| --- | --- |
| FVS-035 | 四种 SQL knob 的合法/非法解析、适用算法和边界错误。 |
| FVS-036 | 搜索 knob 对请求的生效值，SQL option 与算法能力相符。 |
| FVS-037 | `PREPARE` 后改变 session knob；每次执行解析，literal 与 prepared 语义一致。 |
| FVS-038 | 并发会话设置隔离，不把 session 值写回共享计划或另一会话。 |
| FVS-039 + FVS-040 | SQL、session、默认值的优先级及 session 内组合；仅在一项执行中验证优先级和隔离。 |
| FVS-041 | 最大 uint64 itopk；HNSW/IVFPQ/IVF-FLAT 有实际行数限制且不 panic，CAGRA 依上限拒绝。 |
| FVS-042 | EXPLAIN 显示 SQL knob 和 Index Filter；运行期 session 值以 request/UT 佐证，不能只从 EXPLAIN 推断。 |
| FVS-043 + FVS-044 | 有 knob 时对 prior protocol 版本在 placement、发送重探、handshake/receiver 拒绝；无 knob 控制仍按基础协议运行，非向量 scan 不抬高要求。 |

### 3.5 C membership 与 GPU mask（7 项，GPU 门槛）

| 保留编号 | 场景与必要断言 |
| --- | --- |
| FVS-045 | 紧凑、稀疏、跨 container、负/远距 BIGINT；cbitmap/CRoaring/sorted64 的选择、键编码和非法元素宽度。 |
| FVS-046 | GPU 收到 Go-memory 或非 exact membership 时明确拒绝；空 exact 集合表示拒绝全部，不能按无过滤继续。 |
| FVS-047 + FVS-052 | membership 和 mask/delete copy 的内存准入、失败传播、引用共享与最后释放；不部分执行或静默 CPU 替代。 |
| FVS-048 + FVS-051 | membership、INCLUDE/PK、删除位的 AND mask；prepare 时已有删除必须排除，分片/负稀疏 PK 正确。 |
| FVS-049 | CAGRA/IVFPQ 多轮与 multi-index/shard 每个对象只准备一次 mask 并复用；释放正确。 |
| FVS-050 | prepare 后新增行不因 mask 末尾默认全 1 通过；上传清尾位并限制 shard copy。 |
| FVS-053 | IVFPQ 探测列表无合格项的 sentinel；不泄漏非法 ID/距离，空首轮不等于全索引耗尽。 |

锁持有细节不单设 FVS-054；其代码核对结果附于 GPU 执行记录，取消与错误传播由 FVS-058 验证。

### 3.6 DML、生命周期与多 CN（9 项）

| 保留编号 | 场景与必要断言 |
| --- | --- |
| FVS-055 | INSERT/DELETE/UPDATE 向量、普通列和 INCLUDE 列在既有可见性屏障后重算资格；rollback 行不可见。 |
| FVS-056 | CAGRA/IVFPQ built index + CDC overflow；membership、PK、子查询过滤均全局合并且无重复/遗漏。 |
| FVS-057 | async REINDEX 空 pipeline 与 sharding；无 rows 不 panic，generation、host_ids/mask 一致。 |
| FVS-058 + FVS-059 | 跨轮取消、设备/构建/缺 required 域失败和空域；及时停止、无下一轮、资源归还，缺域不当作空集。 |
| FVS-060 | volatile/sequence 过滤或 membership join；危险重放走 exact，表达式按语义单次求值。 |
| FVS-061 | 一条 producer/回表并发更新的语句快照控制；不新增完整权限矩阵。 |
| FVS-062 | CN1/CN2 抽测：红例、周期选择率、prepared/cap；确定性 fixture 一致，ANN 以合法性/行数/recall 判定。 |
| FVS-063 | 仅保留 UNION ALL 多分支向量搜索；runtime filter、knob 与状态按分支隔离，合法重复保留。 |
| FVS-064 | prepared 后 drop/rebuild 索引；不得复用失效 generation/domain/mask，需重规划或明确失效。服务冷启动/重启移至稳定性专项。 |

## 4. 执行批次与门槛

### 第 1 批：CPU 功能验收（P0）

依次执行 FVS-011；FVS-001/002/003+009/004+005/006/007/008；FVS-013..020；FVS-023/024/028/029/031/032；FVS-035..043；FVS-062 抽测。

此批用 SQL 和现有 CPU UT 回放，覆盖 Issue 的模式、短页、可观测、prepared、数量分档和双 CN 五类验收。FVS-021 的现有 hybrid-fulltext BVT 作为必跑复用证据附入本批结果。

### 第 2 批：GPU 功能验收（P0，适用于 GPU 交付）

执行 FVS-025/026/027；FVS-045..050/053；FVS-010；FVS-055..058/060/061/063/064。

CAGRA/IVFPQ 若在发布范围内，本批为功能门槛；没有合格 GPU 环境只能报告阻塞，不能以 CPU 或 force 代替。

### 第 3 批：非门槛专项（按需或 nightly）

- 性能冒烟：一个固定数据集、七种 mode、100% 与 5% 两个选择率点，记录延迟、返回数和 recall；无预设性能阈值时只报告事实。
- GPU mask 收益、ANN benchmark、跨产品比较：移入独立 benchmark 计划。
- 两小时混合长稳、冷启动/服务重启：移入 nightly/stability 计划。
- FVS-068 的轮次、累计工作量、内存峰值作为 FVS-024/026/028 的记录项，不单设执行项。

## 5. 编号处置映射

| 处置 | 编号 |
| --- | --- |
| 保留为主项或合并主项 | 001-020（除 021）、022-032、035-043、045-050、053、055-058、060-064 |
| 合并为父用例断言 | 005→004，009→003，022→015，033→024/026/028，034→020，040→039，044→043，051→048，052→047，059→019/058 |
| 复用既有回归、不新设执行项 | 021 hybrid fulltext BVT |
| 移出本 Feature 合入门槛 | 065/066 压缩为性能冒烟；067、069 移 benchmark；070 移 nightly；064 的冷启动/重启部分移 stability |
| 删除的冗余变体 | 061 的权限矩阵；063 的多索引/JOIN 变体（保留 UNION ALL） |

## 6. 证据与结果出口

每个执行单元记录：MatrixOne SHA、运行环境与 GPU/原生库版本、数据与索引实况、完整 SQL/参数、Q/P/actual、过滤/重复检查、有效请求 knob、EXPLAIN 与实际执行证据、轮次/分支/终止原因、错误和资源清理。

验收输出必须分两层：

1. **交付实现验收**：承诺的 mode/算法/参数组合无过滤泄漏、无无界迭代、协议或内存失败不静默降级，确定性 fixture 有直接证据。
2. **原需求完成度**：显式说明 post/default 短页、cap、split-reader 以及可观测性限制是否仍偏离 Issue；未获范围接受时，不因全部保留测试通过而关闭需求。

已有测试映射仅作为可复用证据入口；每轮报告必须写入实际执行 ref、命令、平台、通过/失败/阻塞和未覆盖项。