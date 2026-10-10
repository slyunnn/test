# MatrixOne ARM64 NEON SIMD 专项验证补充清单

> 对应 [Feature #29495](https://github.com/matrixorigin/matrixone/issues/29495)，补充于2026-10-10。采用内核、平台和构建链专项清单，不扩展为通用SQL、多CN或大数据测试设计。
>
> 实现：[PR #29496](https://github.com/matrixorigin/matrixone/pull/29496)，合入提交 `143a03b3060d6eff36f82b42d2e1f1b7566e897a`。已有[验证报告](https://github.com/matrixorigin/matrixone/issues/29495#issuecomment-6093508201)的源码基线是 `0ccc989731f9c2b40d31554a44ca505c656c0e62`，运行平台darwin/arm64、Apple M2 Pro、Go 1.27.1。本次只补充文档、核对已有材料；没有重新执行构建、UT、benchmark或平台验证。

## 1. 已有证据与结论边界

| 项目 | 已有材料记载 | 本清单如何使用 |
| --- | --- | --- |
| darwin/arm64 SIMD构建及包UT | 指定评论报告GOEXPERIMENT=simd构建、完整metric包测试通过 | 标为“已有报告通过”，不是本次重跑；保留平台、commit、工具链与原始日志入口。 |
| scalar构建 | 同评论报告GOEXPERIMENT=nosimd完整包测试通过 | 证明该平台的scalar构建/行为；不能替代SIMD分支已实际执行的证据。 |
| 窄类型性能 | bf16/f16/int8/uint8的l2sq约2.9x/2.1x/2.1x/2.2x，0 allocs/op | 对应已有`Benchmark_Narrow_NeonVsScalar`的1024维fixture；不外推到所有类型、距离、维度或数据库QPS。 |
| amd64/GPU构建及性能 | PR作者报告Linux amd64、AVX2/AVX-512、CPU/GPU构建和benchmark | 作为历史参考单列；其commit、硬件及工具链不等同于上述评论的验证基线，不冒充当前回归结果。 |
| Linux arm64服务器 | 指定评论明确未覆盖 | 待有真实目标硬件执行；交叉编译或Apple同架构通过不能标成Linux实机通过。 |
| Go/镜像/lint交付链 | PR包含Go 1.27、镜像及golangci-lint升级 | 包级UT不证明镜像已发布、可构建或完整静态检查通过；需要相应产物和日志。 |

当前可支持的结论是：已有报告在指定darwin/arm64基线上验证了NEON路径、scalar回退和已测窄类型l2sq性能。跨平台、构建链和更广泛性能的验收按下表独立记录。对feature宜写“已实现、已验证范围通过”，不把“Fixed”解释成所有平台全部通过。

## 2. 补充验证清单（18项）

“待执行”表示本清单未提供该验证项在统一验收基线上的实际执行证据；已有历史结果不自动改写成新结果。不可用硬件标为“未覆盖/环境阻塞”并注明原因，不能以SKIP计为该架构通过。

### 2.1 平台与路径选择

| 编号 | 项目 | 验证内容与通过标准 | 当前证据状态 |
| --- | --- | --- | --- |
| SIMD-001 | darwin/arm64 NEON | Go>=1.27、GOEXPERIMENT=simd；记录实际被选Go文件、hasNeon/禁用变量、NEON专属测试运行而非SKIP；完整metric包通过 | 已有评论报告通过；路径选取日志待归档/补齐。 |
| SIMD-002 | Linux arm64服务器 | 在真实目标CPU执行001及数值/性能项，记录CPU型号、OS/kernel、实际ISA路径；不能只用GOOS交叉编译代替运行 | 待执行。 |
| SIMD-003 | Linux amd64 AVX-512 | 支持该ISA的机器运行完整metric包，证明选中AVX-512且专项测试不跳过；重点检查Go API迁移及整数新kernel | PR历史报告可参考；统一基线待验证。 |
| SIMD-004 | Linux amd64 AVX2 | 在具备AVX2的硬件用MO_METRIC_NO_AVX512=1执行，确认AVX2路径仍启用；保留整数极值和数值恢复测试 | PR历史报告可参考；统一基线待验证。 |
| SIMD-005 | SIMD关闭与运行时回退 | 各目标架构GOEXPERIMENT=nosimd构建/完整包测试；另用ARCHSIMD=0检查Makefile实际命令。arm64可用MO_METRIC_NO_NEON=1做运行时回退控制 | darwin nosimd已有报告；Makefile及其它平台组合待验证。运行时禁用导致NEON专项SKIP是负向控制，不能算NEON覆盖。 |

### 2.2 工具链与构建交付

| 编号 | 项目 | 验证内容与通过标准 | 当前证据状态 |
| --- | --- | --- | --- |
| SIMD-006 | Go最低版本与自动切换 | Go<1.27且GOTOOLCHAIN=local应清楚报最低版本不足；受支持Go与GOWORK=off可构建；GOTOOLCHAIN=auto时记录最终实际版本 | 待执行；不能把自动下载新Go后的成功归功于旧Go。 |
| SIMD-007 | workspace/build tags | 记录go version和go env GOVERSION/GOARCH/GOOS/GOEXPERIMENT/GOWORK/GOTOOLCHAIN；对比父go.work与GOWORK=off，检查GoFiles/IgnoredGoFiles及专项测试名单 | 待补证据；不得因SIMD文件未参与编译而误报SIMD测试通过。父workspace配置冲突须可定位。 |
| SIMD-008 | 镜像与构建入口 | 针对实际验收ref解析CPU、CI、builder、dev、tester及GPU镜像的FROM/toolchain；核对发布可用性、目标架构manifest、digest并执行相关构建/启动校验 | 待验证。Issue最初未发布状态不能当成现在仍未发布；包build不能替代镜像验证。GPU构建作为受Go升级影响的交付入口，不作为NEON性能证明。 |
| SIMD-009 | lint与完整构建 | 使用实际ref要求的Go和golangci-lint；合入变更升级至v2.14.0。执行仓库static-check-analysis及目标发布构建，保存完整日志 | PR历史报告可参考；统一基线待验证。不能只跑metric包或关闭lint规则后宣称交付链通过。 |

### 2.3 数值、边界与调用入口

| 编号 | 项目 | 验证内容与通过标准 | 当前证据状态 |
| --- | --- | --- | --- |
| SIMD-010 | 类型×距离矩阵 | f32/f64覆盖L2sq、L2、L1、负内积、cosine distance/similarity、spherical；窄类型先覆盖已有l2sq/内积/L1/cosine专属入口，再检查实际暴露的派生dispatcher | darwin已有包UT报告；逐项映射并按平台补证据。不能假设每个类型都有每种独立NEON kernel。 |
| SIMD-011 | lane及尾部 | 0/1、每种kernel的lane宽W前后、展开块边界前后、非整除维度及128/768/1024/1536等；长度不匹配、零向量和clamp边界 | darwin已有报告；Linux/amd64补同矩阵。核对标量尾部，不仅测1024等整齐尺寸。 |
| SIMD-012 | 有限极值与抵消 | 有限大值、正负lane部分和溢出后抵消、零范数、真正不可表示结果；直接比较对应标量/高精度契约 | darwin报告覆盖；跨平台补证据。可恢复的内积/余弦须恢复，不能用“非有限一律丢弃”掩盖错误；真正溢出按既有sentinel/错误契约，不要求所有结果有限。 |
| SIMD-013 | 窄类型扩展 | int8 -128/127、uint8 0/255交错，符号扩展/零扩展与累加；f16/bf16负数、大值、小值及转换后oracle | darwin已有报告。指定整数L2sq/内积/L1 fixture精确相等；cosine及浮点按明确容差，不笼统要求全部bit-exact。 |
| SIMD-014 | dispatcher与直接消费者 | 验证generic API确实进入正确实现，错误/结果契约一致；复用`TestBruteForceRecoversNarrowCosineWinner29496`等直接调用方回归 | metric dispatcher已有报告；直接消费者在统一基线的执行证据待补。不新增SQL/MOTR来代替kernel选路证明。 |

### 2.4 性能补证据

| 编号 | 项目 | 验证内容与通过标准 | 当前证据状态 |
| --- | --- | --- | --- |
| SIMD-015 | 复核已测benchmark | 固定1024维、四种窄类型l2sq，在同一二进制/硬件对照NEON和scalar，重复测量并报告ns/op、B/op、allocs/op及波动 | 有单平台历史数值；本轮未重测。不得把原2.1x..2.9x设为所有CPU硬性门槛。 |
| SIMD-016 | 扩展性能覆盖 | 补f32/f64及其它实际支持距离、短向量/尾部/常用维度；缺benchmark入口时定向补充，逐组合记录 | 待执行；现有窄类型l2sq结论不能填满此矩阵。功能已测不等于性能已测。 |
| SIMD-017 | amd64与版本回归 | 比较AVX2/AVX-512和scalar；固定benchmark harness、数据池与构建条件，记录升级Go带来的混合影响 | PR历史报告含f16 scalar慢化，须复核适用版本/幅度，不能概括“所有路径无回退”。原版/新版若用不同Go，不把差异全归因于kernel。 |
| SIMD-018 | 测量可信度 | 至少10次样本，固定维度/seed/池大小、CPU状态和并发，交替比较并用benchstat等统计；确认基准真正调用目标路径且未被优化消除 | 待归档/补测。保留分布、误差与目标ISA；无预先阈值时报告事实，不自行宣称所有性能项验收完成。 |

## 3. 数值判定补充

- `InnerProduct`在该包语义下返回负点积，独立oracle不能误用正点积。L2与L2sq分别核对，spherical与cosine也不混为同一公式。
- f16/bf16参考值使用同一实际编码/转换后的输入，不能拿原始未量化f32数据要求零误差。
- PR源码的浮点用例使用InDelta/checkPair等容差断言，整数指定fixture才精确比较。验收记录实际assertion与绝对/相对容差；不能沿用PR正文“所有kernel逐位一致”的泛化描述。
- 保留“SIMD与同实现scalar尾部”及“独立数值oracle”两种证据。单个较宽容差随机测试可能漏掉lane处理错误，需要边界、极值和抵消样例共同验证。
- 区分有限输入的中间溢出可恢复、真实不可表示结果、输入本身NaN/Inf三类；各按现有metric契约验收。不得添加“所有NaN/Inf输入均必须返回有限值”之类新要求。

## 4. 可复用测试与命令

### 4.1 已核对的测试入口

| 验证项 | 已有入口 | 说明 |
| --- | --- | --- |
| SIMD-010/011/014 | `TestARM64KernelsAcrossDims`、`TestARM64NeonMatchesScalarTail`、`TestARM64DimensionMismatch`、`TestARM64ClampAndZeroEdges`、`TestARM64CosineSimilarityUpperClamp`、`TestARM64GenericDispatchers` | 真实arm64选路；保存测试是否运行/跳过。 |
| SIMD-010/013 | `TestBF16NeonMatchesScalar`、`TestF16NeonMatchesScalar`、`TestInt8NeonMatchesScalar`、`TestUint8NeonMatchesScalar`、`TestNarrowNeonExtremes`、`TestNarrowNeonDimensionMismatch` | SIMD与scalar对照、极值/尾部/长度错误；按函数实际支持类型执行。 |
| SIMD-012/014 | `TestMetricRecoversLaneCancellation29496`、`TestInnerProductFiniteProductCancellation29496`、`TestBruteForceRecoversNarrowCosineWinner29496` | 同名测试可受架构build tags控制，必须记录实际选中文件；直接消费者不一定在metric包内，执行前定位所属包。 |
| SIMD-015 | `Benchmark_Narrow_NeonVsScalar` | 当前入口固定1024维、四种窄类型、l2sq；不覆盖完整性能矩阵。 |
| SIMD-003/004/017 | `TestNarrowSIMDMatchesScalar`及既有amd64基准 | AVX-512/AVX2分别保留分支证据，硬件缺失导致SKIP不能算ISA通过。 |

以上名称来自PR实际差异，映射不代表本次已运行。执行前在实际ref定位并检查测试名、build tags、具体数据和assertion；用例移动或变化时更新证据，不为了通过而放宽断言。

### 4.2 metric包命令示例

以下在MatrixOne实际验收checkout执行，不是在本测试文档仓库执行；遵循该checkout的构建/CGo规则。若需要仓库测试wrapper，使用对应wrapper并记录展开环境。先检查MO_METRIC_NO_NEON、MO_METRIC_NO_AVX512、MO_METRIC_NO_AVX2等禁用变量，正向运行不能继承无意的禁用配置。

```sh
git rev-parse HEAD
go version
go env GOVERSION GOOS GOARCH GOEXPERIMENT GOTOOLCHAIN GOWORK

env GOWORK=off GOEXPERIMENT=simd go env GOVERSION GOOS GOARCH GOEXPERIMENT GOTOOLCHAIN GOWORK
env GOWORK=off GOEXPERIMENT=simd go list -f '{{.GoFiles}} {{.IgnoredGoFiles}} {{.TestGoFiles}}' ./pkg/vectorindex/metric
env GOWORK=off GOEXPERIMENT=simd go build ./pkg/vectorindex/metric
env GOWORK=off GOEXPERIMENT=simd go test ./pkg/vectorindex/metric -count=1 -v
env GOWORK=off GOEXPERIMENT=nosimd go test ./pkg/vectorindex/metric -count=1 -v
```

仅在真实arm64且NEON启用的环境执行以下直接对照；若正则未匹配到benchmark，不能把命令退出0当作性能测试完成：

```sh
env GOWORK=off GOEXPERIMENT=simd go test ./pkg/vectorindex/metric \
  -run '^$' -bench '^Benchmark_Narrow_NeonVsScalar$' \
  -benchmem -count=10 -benchtime=2s
```

仅在真实且支持AVX2的amd64主机执行AVX2控制：

```sh
env GOWORK=off GOEXPERIMENT=simd MO_METRIC_NO_AVX512=1 \
  go test ./pkg/vectorindex/metric -count=1 -v
```

ARCHSIMD是Makefile开关，GOEXPERIMENT是Go构建条件，运行时禁用变量控制ISA分派，三者分别记录，不能互相替代。镜像、完整构建与静态检查命令以验收ref的Makefile/镜像构建入口为准，避免把历史工具链标签硬套到已更新的main。

## 5. 证据归档与验收表

每一平台/模式填写下表，缺项明确标注未覆盖。原始日志应能辨别PASS、SKIP、build-tag排除和未命中benchmark，不只保存退出码。

| 字段 | 必填内容 |
| --- | --- |
| 基线 | source SHA、工作区改动、测试/benchmark harness版本、工具链实际版本 |
| 平台 | OS、CPU型号/ISA能力、真实运行或仅交叉构建、架构 |
| 构建 | GOEXPERIMENT、ARCHSIMD、GOAMD64、GOWORK、GOTOOLCHAIN、相关禁用变量、选中文件 |
| 功能 | 类型/距离/维度、oracle、容差、实际通过/失败/跳过数、完整日志 |
| 性能 | 单项名称、维度、输入池/seed、每轮ns/op及B/op/allocs/op、重复次数、统计结果 |
| 交付 | 镜像tag与digest/目标架构、构建入口、lint版本及日志、运行镜像校验 |
| 结论 | 已验证范围、待验证范围、阻塞原因；历史作者证据与本轮结果分列 |

推荐结论表述：

> “在提交 `<SHA>`、`<OS/CPU>`、Go `<version>`、`<构建及ISA模式>` 下，已完成 `<实际执行项目>`，结果 `<结果>`。性能结论仅适用于 `<类型/距离/维度/环境>`。`<平台/构建入口/性能组合>` 尚未验证。”

无需为本专项新增Proxy、多CN、事务分页或大表回归来证明NEON选路；但维度与向量批次规模的微基准、直接消费者回归，以及Go升级影响的发布构建仍有明确价值。GPU/CUDA不在NEON数值性能验收范围，GPU镜像的Go构建兼容性单独归到交付链。

本次补充完成标准是清单和证据边界完整、测试映射及Markdown渲染通过检查；18项均不因本文件提交而自动变成“本轮执行通过”。
