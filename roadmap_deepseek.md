# DeepSeek 系列调研路线与交付规划

本文规划如何以 model-research-atlas 的数据、证据与展示方式为模板，逐步建立 DeepSeek 系列的模型结构、推理计算、后端实现及 Ascend 算子研究。建议先完成一个 DeepSeek-V3 的可复核样例，再扩展 V3.2、V4 和 R1 相关分支；先把配置、shape、源码与证据对应起来，之后再决定是否扩展为完整网站、逐层 Excel 和硬件实测。

- 规划日期：2026-10-01
- 目标仓库：https://github.com/ddddwee1/model-research-atlas
- 调研模板基线：2d2411058b61529e25d64c7c96c26d4ec9992b0d
- 本次交付：仅新增本文，不包含 DeepSeek 数据接入、模型实现、权重审计或性能实验
- 文档约定：正文段落不按固定列宽硬换行；标题、列表、表格和代码块按语义换行
- 使用方式：本地新会话先阅读本文与仓库规范，复核最新状态，再从第 4 节的 P0 开始推进；本文中的待办、建议路径及阶段目标均不能当作已经完成的事实

## 1 目标与默认范围

### 1.1 最终要回答的问题

1. DeepSeek 各版本之间真正改变了什么：模型主干、注意力、专家结构、训练方式、推理模式、权重精度，还是服务配置？
2. 一个具体 checkpoint 的参数、层、矩阵、缓存和计算步骤分别是什么，结论能否回到固定版本的配置、权重元数据或源码？
3. 数学计算在参考实现、推理框架、后端分派与设备 kernel 之间如何对应，哪些路径有源码证据，哪些仍未知？
4. 面向一个明确的 Ascend 型号和软件栈，哪些模块值得优先研究或实验，如何验证正确性和收益？
5. 如何把以上结果整理为可复核、可维护、可继续扩展的数据和报告，而不是仅有一份论文摘要或性能排行榜？

### 1.2 首版默认范围

首版以主线文本模型的架构到推理计算为主，交付研究说明、版本对比、参数表、代表层算子表与来源清单。V3 作为第一份完整样例，V3.2 和 V4 作为架构增量研究；R1 单独说明后训练与底座关系。V2 用于理解 MLA 与 DeepSeekMoE 的来源。版本盘点可以更广，但逐矩阵和逐后端深读先聚焦代表版本。

Coder、Math、VL、OCR 等其他系列，所有历史小版本、全部蒸馏尺寸、第三方量化版本以及跨硬件实测，暂列为后续可选范围。正式版本清单必须通过官方目录逐项核实；本文给出的研究顺序不是截至某日的完整产品目录，也不是“最新版本”声明。

以下选择尚需确认，但不阻塞配置、论文和源码的只读研究：

- [ ] 最终交付优先级：研究报告、逐层算子 Excel，还是包含可运行网站
- [ ] 是否只覆盖主线文本模型，以及是否需要纳入所有后训练版本和蒸馏模型
- [ ] 首个目标 checkpoint：默认建议原始 DeepSeek-V3；如果任务直接面向 V4 部署，可在完成必要基础梳理后调整
- [ ] Ascend 目标型号、卡数、拓扑、可用驱动固件、CANN、torch_npu 和推理框架版本
- [ ] 是否有运行资源及允许下载的权重规模，是否需要权重文件头审计或真实 profiling
- [ ] 对方是否另有必须保持的 Excel 模板、字段顺序、评审标准和截止时间

### 1.3 首个最小交付

第一份可评审成果应包括：一页版本与架构关系说明；V3 固定版本的配置参数表；一层 Dense 和一个典型 MoE 层的数据流；区分 prefill 与 decode 的算子及 shape 表；参考实现到候选 Ascend 路径的证据记录；来源清单和缺口清单。可以先用符号 shape 和静态验证完成这一闭环，不以加载完整模型作为开始研究的前提。

最重要的验收标准是：评审者从表中的任意一个事实、shape 或算子映射出发，能找到相应配置字段、源码符号、条件分支或权重元数据；推导过程可以复算；未知项没有被默认值、相邻版本或经验猜测填满。

## 2 如何理解与使用现有仓库

### 2.1 仓库提供的模板能力

仓库包含家族与版本目录、章节报告、结构可视化、逐层数据、后端接口研究、硬件与部署说明、优化待办，以及 XLSX、CSV 和附件下载。动态模型家族当前包括 Kimi 与 GLM，另有 OpenBMB 独立报告。数据和静态资源是主要资产，Python 脚本负责生成部分导出与校验，前端从 JSON 加载内容。

仓库已有数据应当作为组织方法和接口示例使用。Kimi、GLM 或第三方部署配方里的参数、缓存假设、量化格式和性能说明不能直接移植到 DeepSeek；这些示例本身也有不同的证据深度。README 与验证记录明确区分静态研究、权重文件头审计和设备实测，当前规划继续保持这种区分。

### 2.2 建议阅读顺序

| 顺序 | 已有文件 | 要提取的内容 |
| --- | --- | --- |
| 1 | [README.md](README.md) | 仓库用途、交付形式、数据流和已声明的边界 |
| 2 | [ADDING_MODELS.md](ADDING_MODELS.md) | 新增家族、版本、结构、硬件与计算数据的规则 |
| 3 | [schema.json](schema.json) | fact、model、family、architecture、hardware、compute 的数据契约 |
| 4 | [data/catalog.json](data/catalog.json) | 家族入口和独立报告入口的接入方式 |
| 5 | [data/families/kimi/family.json](data/families/kimi/family.json) | 完整模型条目、事实来源、报告与结构引用 |
| 6 | [data/families/kimi/k2-instruct-architecture.json](data/families/kimi/k2-instruct-architecture.json) | 层、模块、专家模板、逻辑矩阵与存储矩阵 |
| 7 | [data/families/glm/hardware.json](data/families/glm/hardware.json) | 实现专题、平台条件、待验证优化与实验协议 |
| 8 | [dist/downloads/model-documents/manifest.json](dist/downloads/model-documents/manifest.json) | Excel 原字段、扩展证据列、模型覆盖口径 |
| 9 | [scripts/build_compute_atlas.py](scripts/build_compute_atlas.py) | 固定源码证据与计算步骤的组织方式，不直接运行作为 DeepSeek 生成器 |
| 10 | [scripts/build.py](scripts/build.py)、[scripts/validate_compute.py](scripts/validate_compute.py)、[scripts/validate_model_documents.py](scripts/validate_model_documents.py) | 当前实际执行的检查，以及 Kimi 专用逻辑和导出限制 |
| 11 | [VALIDATION.md](VALIDATION.md)、[THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md) | 验证范围、来源与许可边界 |

### 2.3 不能直接照搬的地方

- 现有 compute 生成器与验证器包含 Kimi 专用路径、组件和断言。新增 DeepSeek 需要对应生成与验证逻辑，并调整 build 的计算验证分派，不能只替换 familyId。
- schema 中 architecture.cache 当前要求 mla、kda、vision 字段。接入 DSA、CSA、HCA 或其他缓存结构时，需要明确兼容策略，不要把新结构塞进语义错误的旧字段。
- 结构页面可接受未知类型并使用中性色，但“模型/3D 模块跳转到实现专题”的映射仍可能需要扩展和回归检查。
- 原始私有 Excel 模板未公开。仓库已有独立导出和字段定义可参考；不能宣称可以重新生成未取得的原始模板，也不能修改原21列后仍声称保持模板兼容。
- 现有逐层字段说明与 Excel 使用不同符号口径：逐层说明里的 S 是注意力可见长度，Excel manifest 里的 S 是 verify 长度。新数据先统一符号，并为兼容输出建立显式转换。
- 普通 build 会生成导出、重写部分 family 数据并复制 data 到 dist/data。后续运行前先记录工作树状态，运行后检查 diff，防止把与 DeepSeek 无关的生成变更一并提交。
- 仓库未设置覆盖所有内容的统一开源许可证。参考方法、复用代码、转载材料和发布权重元数据分别核查相关权利；公开可读不等于所有材料都获得了再分发许可。

## 3 证据与记录规则

### 3.1 两条维度分别记录

第一条维度沿用现有 fact.evidence：official 表示可追溯的模型发布方披露；derived 表示由明确输入计算得到；interpretation 表示解释或判断；unknown 表示未知。fact.value 为 null 时，evidence 必须是 unknown，不能用 0、空字符串或估计日期伪装为已知。

第二条维度记录“核验到哪里”：论文或模型卡、配置、参考源码、推理框架、设备实现、权重文件头、运行正确性、性能实验。这些层级不是简单的高低替代关系。例如模型卡给出的参数规模和文件头统计的存储载荷回答不同问题；源码存在也不能证明运行环境可用。现有 schema 未直接规定的细分状态，可以先放在来源记录、scope、limit 或运行条件说明中；若未来扩展字段，应同步修改契约与消费者。

| 信息 | 最少需要的依据 | 不能由此推定 |
| --- | --- | --- |
| 官方公布的架构与规模 | 官方论文或模型卡，记录版本与位置 | 精确权重载荷、任意设备上的实际成本 |
| 配置值 | 对应 checkpoint 的固定 revision 配置文件 | 配置默认值一定等于已发布权重的真实取值 |
| 逻辑 shape 与参数量 | 配置加参数声明或可复算公式 | 文件存储布局、kernel 内部布局、实测内存 |
| 权重 dtype、打包和载荷 | 实际权重元数据或文件头，加完整覆盖范围 | 运行时计算精度、速度、数值质量 |
| 参考计算 | 固定版本的源码符号、输入输出与分支 | 高性能后端按相同步骤逐一执行 |
| 后端或 kernel 映射 | 固定代码版本、调用关系和分派条件 | 任意版本或型号兼容、整模型可运行 |
| 性能结论 | 可复现实验配置、原始结果、统计和正确性检查 | 超出负载、硬件、精度和版本条件的普遍收益 |

### 3.2 来源清单

每个结论使用稳定的 source_id。建议保存 url、repo 或 model_id、revision、path、symbol、line/end、accessed、source_kind、license、sha256、适用模型和证据说明。源码要固定到 commit，HF 配置和权重元数据要固定到 model revision；网页没有稳定版本时，记录访问日期和必要的合法快照。未完成固定的来源显式标为待固定，不能写一个未经核验的 SHA。

函数体哈希与 AST 调用检查可以辅助发现版本变化，但 AST 的调用集合不能当作执行顺序。并列 API 可能是互斥分支；一个 framework module 可能包含多个 kernel；多个数学步骤也可能融合成一个 kernel。shape 表要能表达这种多对多关系。

### 3.3 冲突与缺口

- [ ] 同一指标存在论文、config、源码默认值和权重头差异时，保留各方原值与来源，不静默选择一个
- [ ] 区分官方模型发布方与后端实现方；后端仓库对模型结构的复述不能替代模型官方配置核验
- [ ] 区分检查点 revision、推理框架 commit、设备栈版本和导出脚本版本
- [ ] 未确认的发布日期、设备型号、支持条件和性能数值保留未知
- [ ] 参数口径必须标明是否包含 embedding、output head、共享参数、MTP 和量化元数据
- [ ] 已检查范围与未检查范围写在报告和导出中，不能仅放在开发日志里

## 4 分阶段路线

各阶段以产物和验收门槛推进，不预设未经确认的完成日期。P0 和 P1 完成后即可形成第一份有价值的评审材料；网站和设备实验不是前两阶段的前置条件。

### P0 固定范围与来源

目标：确定研究对象和来源，不让版本名称或后端假设混入后续数据。

- [ ] 阅读本地适用的 AGENTS.md、仓库规范及本文；检查当前分支、工作树和已有 DeepSeek 工作
- [ ] 获取官方模型目录，建立 checkpoint 清单；区分 Base、后训练、蒸馏、量化衍生和推理模式
- [ ] 为 V2、V3、V3.2 与 V4 选定代表模型；R1 建立与底座的关系记录
- [ ] 对每个候选记录官方模型卡、论文、配置、参考实现、权重索引和许可入口
- [ ] 固定首个 V3 checkpoint revision、官方推理源码 commit 和研究输入文件哈希
- [ ] 建立 source_id 清单与缺口列表，记录本次读取范围
- [ ] 如果尚未确认目标硬件，在平台字段标记待确认，继续静态研究

产物：研究范围说明、版本关系清单、来源清单、首个 V3 的配置快照。

验收：每个纳入深读的模型有唯一名称和可核对的官方入口；不存在把推理档位作为独立 checkpoint、把不同底座的 Distill 当作同构模型，或把流动 main 链接冒充固定快照的情况。

### P1 完成 DeepSeek V3 的首个闭环

目标：把一个模型从配置一路拆到可解释的计算步骤和来源。

- [ ] 从官方配置逐项提取层数、隐藏维、heads、q/kv 低秩维度、qk_nope/qk_rope/v 维度、Dense宽度、expert宽度、专家数、top-k、共享专家和 Dense/MoE 层位置
- [ ] 阅读官方 README_WEIGHTS，单独记录主模型、MTP、共享 embedding/head 和 FP8 scale 的口径
- [ ] 阅读官方 inference/model.py 的模型参数、MLA、MLP、Gate、MoE、Block 和整体 forward，实际符号以固定版本为准
- [ ] 画出 embedding 到输出的整体数据流，标出一层 Dense 与一个典型 MoE 层，以及 MTP 的独立位置
- [ ] 给 attention、Dense FFN、router、routed experts、shared experts、norm 和 residual 建立参数与操作记录
- [ ] 对 prefill 和 decode 分别记录 query、KV/cache、中间张量和输出；吸收式 MLA 与展开式参考路径分别说明
- [ ] 按第 7 节符号约定写出 shape；对至少一组合法小规模场景进行独立公式/shape 复算
- [ ] 给每个结论附 source_id、固定路径、符号和条件；无法定位的项保留未知
- [ ] 第一版输出研究说明、参数表、代表层算子 CSV 和缺口清单，不填写无依据的性能数据

产物：可审阅的 V3 单模型研究包。建议先用可读 Markdown/CSV 和机器可读 JSON 表达，避免一开始就手工维护多份重复事实。

验收：两个代表层中的每项关键 shape 均能复算，模块参数能够按声明口径汇总，MTP不混进主干层数，prefill/decode分开，逻辑 shape 与存储格式分开。第一版即使没有任何设备 kernel 映射，也应明确完整的数学与参考源码链条。

### P2 建立推理框架与 Ascend 的证据映射

目标：从“模型如何计算”推进到“某个明确软件栈如何实现”。

- [ ] 确认目标硬件和首个框架；建议先选择一条主路径，不同时穷举所有框架、量化和设备
- [ ] 固定模型实现、attention backend、MoE backend、量化实现和相关设备扩展的源码版本
- [ ] 追踪模型 forward 到框架模块、分派器、后端函数与设备算子的调用关系
- [ ] 分开记录 prefill、decode、KV/cache管理、MoE路由、dispatch/combine、专家GEMM和通信路径
- [ ] 对每条映射记录条件：设备代际、dtype、head维度、batch/token形状、TP/EP、图模式及环境开关
- [ ] 阅读原生权重与转换权重之间的转换脚本，记录量化精度、scale、打包和模型输入要求
- [ ] 给未确认映射写具体缺口，例如“只定位到分派器”“未检查设备分支”“未验证转换权重”
- [ ] 形成候选优化列表，按问题、观察、假设、实验、指标、风险组织

优先专题建议：MLA 的投影与缓存、attention prefill/decode路径、MoE路由与通信、专家GEMM、量化与反量化融合。是否继续研究 DSA、CSA/HCA、mHC 或 Engram，取决于所选模型的真实结构和任务目标。

验收：每条“支持”声明都带具体条件；同一模型不同权重精度和不同设备型号不混用；源码可用、测试通过和性能实测三个状态分别标记。没有 profiler 证据时，只提出待验证瓶颈。

### P3 扩展 V3.2 与 V4 的架构增量

目标：复用已核实的共同部分，集中解释新结构怎样改变计算、缓存、通信和后端要求。

V3.2 任务：

- [ ] 从官方模型卡与固定配置确认 DSA、indexer 及其参数，核对与 V3.2-Exp 的结构关系
- [ ] 对比 V3 与所选 V3.2 checkpoint 的配置、权重命名、精度和模型 forward，不预设所有未提及部分完全相同
- [ ] 拆解索引打分、token选择、稀疏attention和缓存之间的数据关系，说明额外计算与可能节省的计算
- [ ] 分开记录训练侧机制与推理侧实际执行；不把论文算法直接当作某个后端kernel
- [ ] 在算子表中新增 indexer 和稀疏路径的输入输出、条件、来源与未知项

V4 任务：

- [ ] 明确研究 V4-Pro、V4-Flash、Base 或后训练 checkpoint，并固定各自 revision
- [ ] 从官方配置与参考实现读取真实层排布、压缩参数、head维度、专家配置和量化设置
- [ ] 分别拆解 CSA、HCA、局部/压缩缓存及相关索引，禁止沿用 V3 的缓存公式
- [ ] 拆解 mHC 的流数、投影、归一化/约束、混合和残差数据流，核对参数形状与dtype
- [ ] 对 FP4/FP8 混合权重记录模块归属、存储容器、scale 和实际后端输入格式
- [ ] 核对 hash、路由或记忆相关字段的真实语义，不仅根据字段名推断是 Engram
- [ ] 输出“共同部分、变化部分、待核验部分”的结构/算子差异说明，并说明对应测试需求

2026-10-01 阅读到的官方 V4-Pro 模型卡明确列出 V4-Pro、V4-Flash 和各自 Base，介绍 CSA/HCA、mHC 与 Muon；Pro-Max/Flash-Max 是推理档位。Engram 官方仓库提供论文和演示，其 README 明确说明演示中的 Attention/MoE/mHC 被 mock。仅凭这些材料，不能认定 Engram 已用于某个具体 V4 或其他版本的公开 checkpoint。未来版本名称及集成关系必须重新查官方证据。

验收：所有复用项都能说明复用依据；所有新模块都有独立shape与实现边界；版本对比不把训练优化器、后训练方法和推理kernel混为一类。

### P4 整理 R1 与 Distill 分支

目标：解释能力训练与模型结构的关系，避免为同一主干重复制作整份结构数据，也避免错误合并不同底座。

- [ ] 记录 R1/R1-Zero 与 V3-Base 的官方关系，再核对具体 checkpoint 配置
- [ ] 对每个纳入范围的 R1后续版本检查配置、权重和使用方式是否变化，不由命名自动推断同构
- [ ] Distill 分别记录底座家族、尺寸、配置、tokenizer变化和官方来源
- [ ] 为后训练方法、推理模式、采样设置和评测条件建立独立说明，不把它们写成架构差异
- [ ] 架构共享使用显式关系或模板引用；只有实际核对一致的部分才复用

验收：版本谱系能够回答“谁基于谁训练、谁共享结构、谁只是推理模式”；R1-Distill-Qwen/Llama 的结构按其具体底座研究，不套用 V3 的 MLA/MoE 表。

### P5 接入仓库数据与页面

目标：在内容经过审核后，将同一事实源接入现有展示和导出体系。

- [ ] 按第 6 节建立 DeepSeek 数据目录和报告，不修改已有家族事实
- [ ] 对照 schema 与前端实际读取字段，确认所有引用、ID、section和model关联
- [ ] 编写或适配 DeepSeek 数据生成与验证器，避免运行 Kimi 专用验证器得到虚假覆盖声明
- [ ] 按需要扩展 cache、模块类型、实现专题映射和计算验证分派，并为已有家族保留兼容性
- [ ] 从 canonical JSON 生成 Markdown/CSV等导出，避免手工编辑生成物与源数据发生漂移
- [ ] 如果需要 Excel，先确认模板；保留原字段时附加证据列，不把未实测指标填成0
- [ ] 执行静态检查、构建与浏览器验收，检查所有生成diff和附件哈希
- [ ] 只有在发布确有需要并已明确授权时，再处理托管或公开部署

验收：家族入口到版本、层、模块、矩阵和来源的路径可用；搜索、比较、下载、窄屏和二维回退正常；已有Kimi/GLM/OpenBMB主要路径无回归；构建成功的含义与未运行的模型实验明确分开。

### P6 可选的运行与性能实验

目标：验证一个明确软硬件与负载条件下的正确性和优化收益。此阶段需要真实设备、模型与环境，不能由静态调研替代。

- [ ] 固定完整环境和权重转换过程，先做启动与小规模正确性检查
- [ ] 先对目标模块做数值校验，再进行整模型或服务级实验
- [ ] 分开 prefill、decode、verify和端到端服务指标
- [ ] 记录warmup、重复次数、随机种子、输入长度、输出长度、batch、并发、TP/EP、拓扑及精度
- [ ] 一次只改变一个主要变量，保存基线与优化版本的原始结果
- [ ] 同时报性能、显存和质量/误差；量化收益不能只报告吞吐
- [ ] 给出统计分布、异常和失败条件；跨卡数、跨平台和跨精度不直接拼排名

验收：至少一项实验可由另一人按记录复现；性能结论只覆盖实际测试范围；无设备或无授权时，本阶段保持未执行状态，不影响前面静态成果的独立交付。

## 5 V3 首个闭环的具体拆解

### 5.1 固定输入

先取得同一 checkpoint 的 config、模型卡、权重说明、权重索引及需要的元数据，再取得对应官方推理实现。配置、推理demo默认参数和发布权重之间可能存在命名或默认值差异，必须通过字段映射核对。固定到本文记录的V3源码commit可作为起点，但正式产物还需要固定HF checkpoint revision。

官方 V3 demo 的 config_671B.json 在本次核查快照中包含61个主干层、前3个Dense层、hidden 7168、128个attention heads、q_lora_rank 1536、kv_lora_rank 512、qk_nope维128、qk_rope维64、v维128，以及256个routed experts、每token选8个、1个shared expert。这些值用于说明应读取哪些字段；后续必须与选定checkpoint配置核对，不能把demo配置替代完整权重审计。

### 5.2 逐模块记录问题

| 模块 | 需要回答的问题 | 第一版产物 |
| --- | --- | --- |
| Embedding与输出 | 词表和hidden是什么，是否共享，是否分片，是否包含在激活参数口径 | 全局组件参数表 |
| RMSNorm与Residual | 输入输出shape、累加dtype、是否与前后操作融合 | 数学关系与参考源码定位 |
| MLA投影 | Q与KV的低秩分解、RoPE与NoPE分量、输出投影如何构成 | 矩阵shape及中间张量表 |
| MLA注意力与缓存 | naive/absorb路径存什么，prefill/decode如何处理历史与新增token | 两阶段数据流与缓存公式 |
| Dense FFN | gate/up/down矩阵、激活、输出维度与并行切分 | 第一层完整计算样例 |
| Router | 打分、分组、top-k、校正和归一化在哪里，哪些仅训练使用 | 路由步骤与条件 |
| Routed专家 | 每个专家的矩阵、动态token数、计算合并和通信边界 | 一个专家模板加总数/top-k |
| Shared专家 | 是否独立执行、如何与路由专家合并、后端是否存在重叠 | 参考路径与实现缺口 |
| MTP | 额外模块、共享参数、训练目标与推理用法有什么区别 | 单列结构及支持状态 |

### 5.3 一份算子记录应包含什么

建议每行表示一个逻辑步骤，并允许关联零个、一个或多个实现条目。最少包含model_id、component_id、layer或template、step_id、阶段、操作含义、input/output/intermediate/cache shape、dtype、权重逻辑shape、数学关系、参数口径、reference_source_id、backend_source_id、实现层级、分派条件、证据状态和缺口。设备kernel尚未核查时保留unknown，不从“矩阵乘法”猜测某个具体API。

初始示意流程可以按“归一化 → MLA投影及位置处理 → 缓存与attention → 输出与residual → 归一化 → Dense或MoE → residual”拆解。MoE内部再区分routing、token重排/通信、专家GEMM和激活、combine与共享专家。最终步骤顺序必须回到所选实现核实；生产后端可能融合或重排，不能将示意流程当作实际kernel时序。

### 5.4 先做哪些检查

- [ ] 配置字段与源码参数声明一致，维度约束有显式断言
- [ ] 输入、输出和矩阵乘法的收缩维度一致
- [ ] Dense和MoE层映射完整，逻辑层编号与源码0-based编号明确转换
- [ ] 专家模板的单份参数、总专家参数和top-k激活代理分别计数
- [ ] 共享参数不重复计入unique参数，MTP计数口径单列
- [ ] 量化scale等元数据保留存储记录，但不误算为逻辑模型权重参数
- [ ] cache公式注明元素数、dtype、层数、batch/长度条件及是否包含额外工作区
- [ ] 数学推导、源码静态验证、文件头审计和设备测试的完成状态分别列出

## 6 建议的数据与文件映射

以下为后续实现建议。除本文以外，这些DeepSeek文件尚未创建；路径可根据最终交付方式调整，调整时同步更新引用和生成脚本。

| 建议路径 | 用途 | 现有契约或参考 |
| --- | --- | --- |
| data/families/deepseek/family.json | 家族概览、版本、技术主题、场景、边界与下载索引 | schema.family和Kimi family |
| data/families/deepseek/report.json | 已审核的章节内容与模型section引用 | schema.report |
| data/families/deepseek/configs/ | 固定revision的原始配置快照，保留来源与哈希关联 | 已有configs目录 |
| data/families/deepseek/sources.json | 统一来源记录；是否独立成文件需与硬件/计算消费者设计一致 | 建议新增，不是现有强制契约 |
| data/families/deepseek/versions.json | 更广的版本盘点与关系，若family足够则不另建重复事实源 | 可选研究中间数据 |
| data/families/deepseek/v3-architecture.json | V3层、模块、矩阵、参数和缓存说明 | schema.architecture |
| data/families/deepseek/v3-layers.json | 只有完成相应权重头审计后才提供的逐层审计结果 | model.auditPath及已有layers样例 |
| data/families/deepseek/compute.json | 全局组件、代表层模板、展开映射、操作与证明 | schema.compute |
| data/families/deepseek/hardware.json | 实现专题、平台条件、优化待办、协议与固定来源 | schema.hardware |
| dist/assets/deepseek/ | 报告、CSV、图、附件及按需下载资源 | 路径相对dist |
| scripts/build_deepseek_*.py | 从固定输入生成规范化数据与导出，按实际复杂度拆分 | 建议新增，不照搬Kimi硬编码 |
| scripts/validate_deepseek.py | DeepSeek特定结构、shape、证据和导出一致性检查 | 建议新增并接入build分派 |
| data/catalog.json | 内容达到接入门槛后新增deepseek家族入口 | 现有catalog |

### 6.1 family 与 model

沿用模型唯一id、准确名称、branch、summary、tags、facts、sections等字段。已知事实附可核对来源；releaseDate无可靠证据时为null；没有权重审计时auditPath为null。官方config、结构与计算数据的引用按真实完成程度设置，不能为了让页面有入口而创建空壳完成声明。

建议事实包括主干层数、hidden、heads、q/kv低秩维、attention类型、expert数量/top-k/shared数量、Dense与expert宽度、上下文配置、词表、精度与参数口径。部分字段当前前端未展示，即使schema允许facts扩展，也要检查比较表和详情页是否需要对应消费逻辑。

### 6.2 architecture

nodes按真实组件组织，语言层使用group=decoder并从1开始编号，全局组件为0。modules区分attention、norm、dense/routed/shared/router及模型真实包含的其他模块。MoE采用代表专家模板加实际count和selectedCount，不展开成数万个浏览器对象。matrices保留tensor_template、逻辑shape、逻辑参数、存储dtype/shape和量化元数据。

只有配置时可以给出配置层型与范围；由配置和源码推导矩阵时明确标derived。只有权重头或同等可核对存储依据才填写实际payload。既有“激活线性参数代理”不应改写成完整FLOPs或实际性能，计算口径必须随数据保存。

### 6.3 compute

组件映射、模板和步骤分开。层模板中的{i}只表示0-based层索引，{e}只表示专家索引；导出全层时用显式映射展开。每个步骤保留数学关系与参考实现，后端条目注明module、dispatcher或device kernel等层级。proof至少能定位repo、revision、path、symbol、line/end及内容哈希，存在条件分派时保存条件。

如果现有schema不能自然表达prefill/decode不同路径、MTP、压缩cache或多对多kernel映射，应先写小样例和兼容设计，再改schema与消费者。不要为了通过旧校验器丢失真实语义。

### 6.4 hardware

modules记录适用model IDs、flow、meaning、code、hardware、limit、refs；platforms按具体型号和软件栈分列；optimizations记录priority、status、scope、observation、proposal、metric、risk、refs。未profiling的项统一标为待验证，不借用上游README性能数字充当本项目实测。

### 6.5 Excel 与 CSV

现有独立模型文档前21列包含模块、算子、npu kernel、算子类型、input/output shape、dim、shape范围、dtype、format、后续dtype、并行资源、duration、计算利用率、mte、scalar和融合分析。若要求兼容，保持列顺序和含义，并使用附加证据列解释来源、参考调用、后端条件、缺口和数学关系。duration等字段只有实际测量才填数值，未知用明确标记；mte、scalar等profiler字段的单位和含义应按目标设备工具确认。

canonical JSON应作为事实来源，CSV/XLSX为导出。校验行数、列名、BOM约定、各模型覆盖范围、ZIP完整性和文件哈希；“有一份Excel”与“完成逐层研究”分别统计。未取得原始模板时，不宣称完整继承其公式、样式或隐藏逻辑。

## 7 Shape 与口径约定

新研究尽量使用语义明确的符号，避免S/T在不同产物中含义变化。若必须兼容旧Excel符号，应在导出说明和公式转换中明确对应关系。

| 符号 | 建议含义 | 注意事项 |
| --- | --- | --- |
| B | request batch | 与连续批处理的有效请求数及padding口径区分 |
| L_q | 本次每请求query/new-token长度 | 常规decode为1；verify不一定为1 |
| L_kv | 本次attention可见的KV长度 | 含历史和本次可见部分，受mask/窗口/压缩影响 |
| N_tok | 本rank实际参与计算的有效token数 | ragged batch使用求和，不无条件写成B乘L_q |
| H | residual hidden width | mHC额外stream轴单列，不合并成同一个hidden |
| N_head | attention head总数 | 局部head数按实际分片规则推导 |
| D_* | Q/K/V、NoPE、RoPE等明确子维度 | 不将不同模型所有head_dim视作相同 |
| R_q、R_kv | 低秩投影维度 | 仅在相应模型真实包含时使用 |
| E、K、E_local | 总专家数、每token选中数、本rank专家数 | 由具体EP布局和路由定义决定 |
| N_e | 第e个专家实际接收token数 | 动态量，padding、capacity和丢弃策略会改变关系 |
| TP、EP、DP | 具体并行维度 | 记录哪一层/哪种张量按哪一维切分或复制 |
| C_* | 明确命名的cache维度或压缩参数 | KV cache、indexer cache、状态和工作区分开 |

无丢弃且不计padding的路由分配数可检查为N_tok乘K，但不能把所有实现都假设成无padding或无容量限制。权重逻辑shape通常按输出维、输入维表示；实际转置、分片、NZ等设备布局和量化存储shape必须另列。单卡局部参数、全模型unique参数、文件载荷、运行时显存、FLOPs和激活参数代理各自保留口径。

## 8 Ascend 研究与实验设计

### 8.1 先确认平台再选实现

不要把“Ascend支持”当作单一布尔值。至少记录设备型号、卡数/拓扑、驱动固件、CANN、PyTorch、torch_npu、推理框架、设备插件、自定义算子包、模型revision、量化格式与转换脚本、图模式及启动参数。不同型号对dtype、指令、kernel和图模式的支持可能不同，应以所选版本官方文档与源码为准。

TileKernels在已核查的2026-09-30快照中新增Ascend后端，README列出Ascend 950和CANN 9.2.0及以上要求，并包含MoE路由、量化、Engram和mHC相关功能。这是值得进一步研究的具体来源，不代表本项目已经验证该栈，也不能推广为其他Ascend型号或完整DeepSeek模型的支持结论。

### 8.2 候选优化卡片

每个候选只写一个可验证问题，例如“某个decode形状下，MLA投影、cache处理和attention之间是否存在可消除的中间读写”。记录现有路径、源码证据、疑似开销、拟议变更、正确性参考、测量指标、前置条件、风险和停止条件。未拿到trace之前，不先断言瓶颈位置或收益百分比。

建议按以下顺序选择：先可运行且可测的基线，再查attention/cache，再查MoE计算与通信，之后研究量化/融合和图模式。若目标是V4，则在真实trace基础上纳入CSA/HCA与mHC；Engram只有在所选checkpoint或独立专题明确需要时才纳入，不凭热门关键词扩大模型范围。

### 8.3 实验记录最少字段

- 模型与权重：名称、revision、转换步骤、精度、scale、文件校验信息
- 环境：硬件、拓扑、软件版本、容器digest、编译选项、自定义算子版本
- 负载：B、L_q、L_kv、输入输出长度、并发、padding/ragged、TP/EP/DP
- 正确性：参考路径、输入种子、绝对/相对误差、logits或任务质量评测
- 性能：阶段、warmup、重复次数、统计分布、时间单位、吞吐口径和峰值显存
- 证据：启动命令、日志、trace、结果文件、失败记录、实验脚本版本

## 9 验证与完成定义

### 9.1 静态研究完成

- [ ] 选定模型列表、配置revision和源码commit均明确
- [ ] 每个已知事实有来源；null与unknown一致；冲突和缺口可见
- [ ] 配置、层型、逻辑矩阵、参数公式和计算shape一致
- [ ] 专家数量与top-k、共享参数、MTP和量化元数据的口径正确
- [ ] 参考实现、框架分派和设备kernel的证据层级没有混淆
- [ ] 后端支持条件和未检查范围随导出保留
- [ ] 报告、JSON、CSV中的数值来自同一事实源或有一致性校验

### 9.2 网站接入完成

- [ ] 新家族及版本ID唯一，所有路径和section引用存在
- [ ] 主干/其他组件层数与数据契约一致，模型特例由相应验证器处理
- [ ] 已审计数据的模块参数汇总与审计口径一致；未审计数据不冒称审计
- [ ] 新计算验证器明确覆盖DeepSeek，build按family或能力分派
- [ ] 附件大小、哈希、CSV行数、ZIP和下载路径核对通过
- [ ] 浏览器测试覆盖家族、版本、比较、结构、实现、来源、搜索和下载
- [ ] 窄屏及二维回退可用，现有家族无明显回归

### 9.3 实验完成

- [ ] 环境可复现，正确性检查先于性能结论
- [ ] 每项结果标明测量范围、统计方法、基线和变更
- [ ] 量化同时报告质量/误差，不只报告速度
- [ ] 失败、未运行和已通过分开记录
- [ ] 没有把上游自述、静态计算或推测写成本地实测

### 9.4 后续验证命令

以下命令供本地实施时使用，本规划没有执行这些构建或浏览器测试。先阅读脚本并确认工作树，尤其注意build可能更新生成物。

~~~sh
git status --short
python3 scripts/build.py
node --check dist/app.js
node --check dist/structure.js
node --check dist/hardware.js
node --check dist/compute.js
git diff --stat
git diff --check
python3 scripts/serve.py
~~~

scripts/validate_deepseek.py尚未存在，创建并验证后再把其调用加入实际流程。现有scripts/validate_compute.py的源码复验需要额外的research-root快照，而且是Kimi专用；即使它通过也不能证明DeepSeek已被验证。JSON Schema文件存在也不等于已运行完整第三方schema校验，应如实记录所用校验工具和覆盖范围。

## 10 已核实的来源起点

本节是下一次研究的导航与快照记录，不代表这些来源涉及的所有技术和支持声明都已独立验证。访问日期为2026-10-01。GitHub列出的SHA已查询真实远端；HF流动链接尚需在正式数据采集时固定到所选checkpoint revision。

| 来源 | 固定版本或状态 | 已核实用途与下一步 |
| --- | --- | --- |
| [研究模板基线](https://github.com/ddddwee1/model-research-atlas/tree/2d2411058b61529e25d64c7c96c26d4ec9992b0d) | 2d2411058b61529e25d64c7c96c26d4ec9992b0d | 已读README、目录、schema、新增指南、示例与验证脚本；后续检查仓库变更 |
| [DeepSeek-V2](https://github.com/deepseek-ai/DeepSeek-V2/tree/ec98ee3cbffc32104cd55dba8af884b3d772602a) | ec98ee3cbffc32104cd55dba8af884b3d772602a | 已确认仓库快照；作为MLA/MoE演进回溯入口，论文和实现需继续细读 |
| [DeepSeek-V3](https://github.com/deepseek-ai/DeepSeek-V3/tree/9b4e9788e4a3a731f7567338ed15d3ec549ce03b) | 9b4e9788e4a3a731f7567338ed15d3ec549ce03b | 已读README、权重说明、demo配置及部分模型代码；适合首个闭环 |
| [V3权重说明](https://github.com/deepseek-ai/DeepSeek-V3/blob/9b4e9788e4a3a731f7567338ed15d3ec549ce03b/README_WEIGHTS.md) | 与上述V3快照一致 | 主模型、MTP、共享参数和FP8 scale口径；正式统计仍需权重证据 |
| [V3 demo配置](https://github.com/deepseek-ai/DeepSeek-V3/blob/9b4e9788e4a3a731f7567338ed15d3ec549ce03b/inference/configs/config_671B.json) | 与上述V3快照一致 | 初始参数读取样例，不代替checkpoint配置 |
| [V3参考模型](https://github.com/deepseek-ai/DeepSeek-V3/blob/9b4e9788e4a3a731f7567338ed15d3ec549ce03b/inference/model.py) | 与上述V3快照一致 | 继续逐函数追踪MLA、MoE、Block与forward |
| [DeepSeek-R1](https://github.com/deepseek-ai/DeepSeek-R1/tree/0cf78561f1d51c84a21b2190626b21116d5c68bb) | 0cf78561f1d51c84a21b2190626b21116d5c68bb | README明确R1与V3-Base、Distill与Qwen/Llama底座关系 |
| [DeepSeek-V3.2模型卡](https://huggingface.co/deepseek-ai/DeepSeek-V3.2) | checkpoint revision待固定 | 已核实DSA说明及运行指向V3.2-Exp；后续取得配置和模型代码 |
| [DeepSeek-V3.2-Exp](https://github.com/deepseek-ai/DeepSeek-V3.2-Exp/tree/87e509a2e5a100d221c97df52c6e8be7835f0057) | 87e509a2e5a100d221c97df52c6e8be7835f0057 | 已确认远端快照；作为V3.2官方模型卡所指实现入口，需继续代码审阅 |
| [DeepSeek-V4-Pro模型卡](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro) | checkpoint revision待固定 | 已核实系列与模式区分、CSA/HCA及mHC概述，不据此生成全层结论 |
| [V4-Pro config](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro/blob/main/config.json) | 流动main，正式采集前固定 | 已读取结构字段；后续与所选权重和inference核对 |
| [V4-Pro inference目录](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro/tree/main/inference) | 流动main，正式采集前固定 | 官方模型卡指向的本地推理入口，需逐文件深读 |
| [Engram](https://github.com/deepseek-ai/Engram/tree/fb7f84a21f91223715394a33a1dc24bbfb7f788e) | fb7f84a21f91223715394a33a1dc24bbfb7f788e | 已读README，演示mock边界明确；具体checkpoint集成关系另查 |
| [TileKernels](https://github.com/deepseek-ai/TileKernels/tree/66258df6175d2f630ffecb04c5ab66bff8a2ae6a) | 66258df6175d2f630ffecb04c5ab66bff8a2ae6a | 已读README的Ascend更新和平台条件；后续检查API、参考实现与测试 |

推理框架和加速库的扩展候选包括vLLM、vLLM-Ascend、SGLang、FlashMLA、DeepGEMM、DeepEP以及相关CANN实现。它们在本文中只是后续检索清单，尚未完成针对所选checkpoint和目标设备的兼容性审阅；纳入正式来源前需验证真实仓库、许可证、固定commit、调用路径和测试条件。

## 11 风险与停止条件

| 风险 | 处理方式 | 何时暂停相关工作 |
| --- | --- | --- |
| 范围扩张到所有系列与后端 | 先交付V3样例，新增范围单列优先级 | 无法说明新增项对当前交付的价值时 |
| 官方配置、代码默认值或权重头冲突 | 保存双方证据，定位版本和加载路径 | 关键形状无法自洽时，不继续生成确定性结论 |
| 源码/API漂移 | 固定commit与哈希，记录更新差异 | 上游版本无法定位或证据已失效时 |
| 未知硬件与转换精度 | 先做静态数据流，平台与性能保持未知 | 需要设备专属结论却没有平台信息时 |
| 把静态检查当运行验证 | 报告明确检查类型和范围 | 发现支持或性能结论超出证据时 |
| 模板专用断言误用于DeepSeek | 新建验证器及分派，补回归测试 | 校验器未实际覆盖新数据时 |
| 自动构建污染其他家族 | 构建前后检查git状态和diff | 出现无法解释的非目标变更时 |
| 第三方许可或大文件问题 | 只保存必要、允许分发的材料与引用 | 许可不明、需要发布权重或大规模附件前 |

## 12 本地新会话启动说明

### 12.1 建议启动任务

可以把以下内容作为新会话的起点，再补充实际本地仓库路径、目标硬件和本轮交付要求。

~~~text
请先阅读仓库中的 roadmap_deepseek.md、README.md、ADDING_MODELS.md、schema.json，以及当前工作目录适用的 AGENTS.md。先检查 git 状态、分支和已有工作，保留他人的修改。

本轮先推进路线中的 P0 和 P1，以 DeepSeek-V3 做首个可复核样例。请先核对官方 checkpoint 与源码 revision，整理配置、主干/MTP、代表 Dense/MoE 层、prefill/decode shape、来源与缺口。每项事实区分官方披露、推导、解释与未知，并区分配置、参考源码、权重存储、后端实现和设备实测。

先告诉我仓库当前状态、已有可复用数据、需要确认的关键范围和本轮最小交付。能独立开展的静态研究可以继续；不要为了填满表格猜测kernel、性能或版本关系。未确认目标硬件时不要默认某种Ascend型号，不要直接下载巨型权重或启动昂贵实验。

正文段落不要按固定列宽硬换行。需要接入网站或新增验证器时，先说明变更范围与现有Kimi专用逻辑的处理方式。每轮结束给出已完成、待验证、产物路径、执行过的检查及下一步；提交和推送按照本次会话明确授权的范围处理。
~~~

### 12.2 第一轮工作检查清单

1. 检查本文是否仍与仓库当前状态一致，以及是否已有新来源或DeepSeek目录。
2. 确认本轮做静态研究、数据接入还是硬件实验；默认从静态研究开始。
3. 完成来源固定和一个V3配置映射，优先解决主干/MTP及shape口径。
4. 先提交可读的两层样例进行评审，再决定扩全层或扩版本。
5. 记录所有阻塞和具体缺口，避免下个会话重复猜测或重新开始。

### 12.3 后续会话应留下的交接信息

- 当前分支、基线与最新相关commit，以及尚未提交的目标文件
- 已确认的研究范围、checkpoint revisions和源码commits
- 已生成的数据/报告/导出路径及其canonical来源
- 本轮实际执行的静态检查、构建、浏览器检查或设备实验
- 失败与未执行项、尚未解决的事实冲突、需要用户决定的问题
- 下一项可以直接开始的具体任务及验收标准

## 13 当前完成状态

- [x] 阅读仓库真实README、目录、数据契约、新增说明、示例和校验入口，形成规划
- [x] 核实若干DeepSeek官方来源入口与GitHub快照，区分已读内容和后续深读任务
- [x] 明确V3首个闭环、V3.2/V4增量、R1/Distill关系、Ascend条件与证据规则
- [ ] 确认最终交付格式和目标硬件
- [ ] 固定首个HF checkpoint revision并采集研究输入
- [ ] 创建DeepSeek结构、计算与来源数据
- [ ] 实现DeepSeek验证器和所需导出
- [ ] 接入网站并完成回归验收
- [ ] 执行权重文件头审计、正确性测试或性能实验

本规划到此只定义研究方向、工作顺序与完成标准。下一步应从P0的来源与范围核对开始，用P1的V3样例验证方法，再根据实际需求扩展。
