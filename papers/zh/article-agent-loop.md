# Video Agent Loop：StarCut 的硬核实践，DeepSeek Harness 能否满足视频场景？

![Agent Loop 封面：短暂内循环围绕唯一创作项目工作，持久外循环跨越消息、异步任务、休眠与人的接管](../assets/agent-loop/cover.png)

## 引言

AI Coding 从代码补全走向 Coding Agent，背后是一套能够持续工作的 Agent Loop：模型读取仓库、修改文件、运行命令和测试，再根据结果继续下一步。代码仓库为这个循环提供了成熟的工作环境：源文件表达工程状态，文件与终端提供稳定入口，编译器和测试返回执行反馈，人也可以随时检查并接管同一份工程。

创作软件正在经历相似的变化。[Descript](https://www.descript.com/underlord)、[ChatCut](https://chatcut.io/docs/what-is-chatcut) 等产品已经开始让 Agent 理解素材、生成初剪，并通过自然语言修改时间线。生成能力也进入了同一条工作流：一条产品视频做到一半，可能发现缺少合适的产品特写，于是临时生成图片，再把图片扩展为几秒钟的动态镜头；旁白、音乐、字幕和 Motion Graphics 也会陆续补充。剪辑、理解和生成在同一个项目中交替进行，素材集合与时间线都在持续变化。

新的工作方式也带来了一组问题。图片和视频生成需要十几秒到几分钟，生成完成后，Agent 如何感知结果并继续创作？等待期间用户可能已经调整过时间线，返回的素材又该怎样进入当前项目？除了 StarCut 内置 Agent，Codex、Claude Code 等外部 Agent 也可能操作同一个工程，不同入口如何共享项目状态并协调行动？浏览器掌握编辑现场，服务端负责模型调用和媒体任务，回答、指令与执行结果如何在两端之间持续传递？

围绕这些问题，StarCut 已经形成了一套连接媒体生成、项目协同、分布式调度和浏览器执行的完整 Video Agent Loop。近期 DeepSeek Harness 的讨论让 Agent Harness 再次受到关注。这套现成的业务链路正好提供了一组检验题：DeepSeek Harness 是否解决了这些问题，插件化设计能否减少系统复杂度。

## 一、Video Agent Loop 的基本结构

用户让 Agent 把标题提前两秒时，工作可以在一次连续执行中完成：读取当前时间线，找到标题，提交修改，再根据执行结果确认项目状态。模型可能连续调用几次工具，但结果都能及时回来，整个过程不会离开当前上下文。这就是最基本的 Agent Loop。下文把这样一次连续执行称为 **Run**。

StarCut 使用 Vercel AI SDK 7 的 [`ToolLoopAgent`](https://ai-sdk.dev/docs/reference/ai-sdk-core/tool-loop-agent) 运行这部分循环。它已经处理了多模型接入、多步 Tool Calling、参数校验、流式输出和停止条件，[Loop Control](https://ai-sdk.dev/docs/agents/loop-control) 还允许应用在步骤之间加入新的输入。StarCut 因而可以把精力放在视频项目、工具和执行策略上，而不是重新实现一套通用的模型调用循环。

图片和视频生成改变了事情的时间尺度。Agent 提交一段视频生成以后，当前能够完成的工作可能已经结束，素材却要几分钟后才会回来；这段时间里，用户还可能继续修改项目。此时继续工作的已经不只是一次 Run，而是贯穿多次运行、等待和人工操作的创作过程。

StarCut 因此把 Video Agent Loop 分成两层：

- **内循环**完成当前可以连续进行的模型判断和工具调用，由 Vercel AI SDK 驱动；
- **外循环**保留创作过程中的项目、任务和后续输入，在新的结果或要求出现时启动下一次 Run。

![Video Agent Loop 总体架构：一次短暂的模型工具循环工作在持续存在的创作过程之中](../assets/agent-loop/two-loop-architecture.svg)

*一次 Run 可以结束；项目、后台工作和后来发生的事情仍然继续。*

外循环平时可以没有任何模型运行，只在用户提出新要求、媒体任务完成或编辑器返回结果时，把这些变化带入下一次判断。

## 二、一项媒体生成任务怎样跨过几分钟

以开头那条产品视频为例。Agent 发现缺少产品特写，先生成一张图片，再把它扩展成动态镜头。在 StarCut 当前接入的生成链路中，一张图片的常见耗时是 10–20 秒，一段视频大约需要 200–600 秒，具体时间还会受到模型、队列和生成参数影响。

这几分钟不能只用“等待”概括。Agent 需要决定图片和视频能否并行、是否先整理其他镜头、第一项结果回来时要不要立即检查，以及当前 Run 已经结束以后由谁接收结果。如果让模型不断查询进度，大部分调用只会得到“仍在生成”；如果让一次请求一直保持到视频完成，运行时间又会和媒体任务绑在一起。

StarCut 为每次生成建立一项持久 **Task**。Task 保存排队、运行、完成、失败和最终产物等状态，由后台 Worker 独立执行。Agent 提交任务以后，可以继续完成不依赖结果的工作；没有其他事情可做时，当前 Run 正常结束，Task 仍然继续。

任务完成通知只表示“状态可能发生了变化”。接收方重新读取 Task 的最新版本，再处理重复、延迟或乱序的通知。即使某次通知丢失，最终结果仍保存在 Task 中，可以被后续检查重新发现。

转写等紧接着就要使用、耗时相对可控的工作，可以保留一个有限等待窗口：窗口内完成，结果直接回到当前 Run；超过窗口，便和图片、视频生成一样转入后台。系统同时保存生成任务与原工具调用的关系，确保稍后返回的素材仍能回到原来的创作任务。

![媒体任务从当前 Run 提交到持久 Task；短等待结果回到当前 Run，长任务完成后启动新的 Run](../assets/agent-loop/media-task-lifecycle.svg)

*等待方式可以不同，任务状态和最终产物始终只有一份。*

### 进度变化到什么程度，才值得 Agent 再次判断

界面需要连续展示生成进度，Agent 却没有必要从 63% 到 64% 就重新运行一次。真正会改变下一步行动的，通常是这些时刻：

- 某项任务失败，需要重试或更换方案；
- 一部分结果已经足够开始下一阶段；
- 整组任务全部完成；
- 用户设置的检查点或截止时间到达。

这才引出了 **Monitor**。它观察当前创作真正等待的任务，保留每项任务的最新状态，再把大量进度变化整理成少数几次有决策价值的上下文。UI 仍然可以高频刷新，模型只在需要重新判断时被唤醒。

![Monitor 将大量任务进度压缩为少数几次有决策价值的 Agent 唤醒](../assets/agent-loop/monitor-compression.svg)

*Monitor 不保存第二份任务状态，它负责判断什么时候值得让 Agent 重新介入。*

### 素材回来时，项目可能已经变了

视频生成期间，用户可能已经替换了镜头、修改了标题，甚至重新安排了开场。Agent 不能拿几分钟前的时间线快照直接覆盖当前工程。

StarCut 把创作目标与项目现状分开保存：对话记录保留用户的要求、工具调用和必要引用；项目模型持续保存时间线、素材和画布当前是什么。新的素材回来以后，Agent 重新读取项目，只把仍然适用的计划落实为局部操作。

```text
新的素材完成
  → 读取当前项目
  → 判断原计划是否仍然适用
  → 定位需要修改的对象
  → 提交局部语义操作
```

用户手动换掉一张图、移动一个 Clip 或取消原来的安排，这些变化已经存在于项目中。下一次 Run 看到的就是人修改后的版本，不需要再把人的选择同步到一份 Agent 私有工程。

### 页面已经关闭，任务完成以后怎么办

只要创作可能跨越几分钟，浏览器关闭、网络中断和服务更新就会成为正常情况。系统需要保存尚未处理的用户消息、任务结果和客户端返回；StarCut 将这类后续输入放入持久接收队列，内部称为 **Inbox**。新的 Run 从未处理输入和已经提交的历史开始，而不是依赖上一个进程仍然活着。

如果多个 Worker 同时发现同一段创作有新进展，只能由其中一个继续推进。此时才需要 **Run Lease**：它在一段时间内授予唯一 Worker 执行权，失去执行权的旧 Worker 不能继续提交结果。恢复检查还会重新查看未处理输入和未结束 Task，因此实时通知影响响应速度，不决定任务最终能否被发现。

![一段 Agent 工作跨越多次短执行、异步等待、用户介入和进程重启](../assets/agent-loop/inner-outer-timeline.svg)

*项目、Task 和创作记录持续存在，执行它们的 Run 可以结束并重新开始。*

<!-- [研发待补 A-M1] 按图片、视频、转写分别记录 Task 耗时的 P50/P95、并发度、原始更新数、Monitor 上下文数、模型唤醒数和端到端继续时延。当前 10–20 秒与 200–600 秒为 StarCut 当前链路的经验区间，正式发布前补真实样本分布。 -->
<!-- [研发待补 A-P1] 建立生成等待期间的协同编辑任务集，验证 Agent 恢复后读取最新项目、局部修改不覆盖人的选择，并记录人工接管前后的完成率。 -->
<!-- [研发待补 A-R1] 在消息持久化、Inbox 确认、Task 完成和 Run Lease 转移等边界注入故障，记录恢复延迟、重复执行和遗漏结果。 -->

## 三、Agent 的决定怎样落到正在运行的编辑器

模型判断“把新生成的产品特写放到开场第三秒”以后，还要把这个决定准确落实到真实项目。StarCut 给 Agent 提供虚拟文件视图和语义化编辑工具：它可以查找素材、读取时间线、定位对象，再提交局部修改。相关的项目建模和编辑协议已经在 [《Edit Video as Code：将视频工程映射为 Coding Agent 可读写的虚拟文件视图》](./article-edit-video-as-code.md) 中详细说明，这里只讨论工具由谁执行。

视频编辑器同时跨越服务端和客户端，两边掌握的能力不同：

- 模型调用、素材检索和生成任务提交可以在服务端完成；
- 读取当前编辑现场、修改时间线、提取当前帧和渲染预览，需要交给连接中的编辑器；
- 生成、转写等长时间工作由后台 Worker 执行。

客户端工具并没有另建一套只供 AI 使用的编辑 API。浏览器收到操作以后，沿用人类界面相同的领域写入路径更新项目。协同、撤销、验证、预览和最终渲染看到的是同一次修改，Agent 不会在真实编辑器之外维护另一条时间线。

这条边界也决定了工具返回什么。服务端工具可以直接得到结果；浏览器工具需要等待在线编辑器执行并回传；后台工作先返回可追踪的 Task，最终产物稍后再进入创作过程。

## 四、回答、指令和执行结果怎样在两端之间传递

Agent 一边生成文字回答，一边可能要求浏览器执行编辑。前者主要从服务端持续流向界面，后者还需要浏览器把执行结果送回服务端。StarCut 同时保留 SSE 和 WebSocket，原因来自这两个不同方向的工作。

- **SSE** 发送已经提交的消息、运行状态和模型增量，适合回答与状态展示；连接中断以后，客户端可以从持久位置继续读取。
- **WebSocket** 维持服务端与在线编辑器之间的双向通道，用于下发客户端操作并返回结果。

两条连接共享相同的消息和调用身份，不分别维护业务状态。实时增量主要降低等待感；完整消息会在 Run 结束后保存。浏览器即使错过一段增量，也能通过已经提交的完整结果恢复到一致状态。

![SSE 将回答和状态持续发送给浏览器，WebSocket 双向传递客户端操作与执行结果](../assets/agent-loop/dual-transport.svg)

浏览器返回结果时，系统先根据调用身份找到原来的工作。当前 Run 仍在等待，结果就继续推动当前循环；当前 Run 已经结束，结果便作为新的输入启动后续 Run。连接方式不会改变结果属于谁。

### 同一项目打开了多个窗口，命令交给谁

同一个项目可能同时出现在桌面端、浏览器标签页和另一个工作窗口中。它们都能观察项目，一条带有副作用的编辑命令却只能执行一次。

StarCut 会从在线、可写且具备所需能力的编辑器中选择一个执行者，并给这次选择附带有限时效的执行身份，内部称为 **Client Lease**。窗口执行前确认身份，提交结果时再次校验；旧窗口断线后重新连接，即使带回迟到结果，也不能覆盖已经转移的执行权。

读取操作可以在窗口中断后安全地换一个执行者重试。已经开始产生副作用的写操作则不能盲目重放，需要先确认项目状态和原调用是否已经生效。这是多窗口调度必须保留的业务边界。

<!-- [研发待补 A-T1] 测量 SSE 与 WebSocket 的断线恢复时间、游标重复率、增量丢失后的收敛时间，以及客户端结果从提交到进入下一次模型判断的延迟。 -->
<!-- [研发待补 A-C2] 覆盖无在线 Editor、执行中断线、Client Lease 转移、旧结果晚到和响应已提交但客户端未收到等场景，验证写操作没有重复副作用。 -->

## 五、Codex、Claude Code 怎样控制 StarCut

产品内的 Agent 已经处在当前项目和创作上下文中。Codex、Claude Code 等外部 Coding Agent 则需要一个公开入口，用来发现 StarCut 的能力、选择项目并发起操作。

CLI 适合人在终端中调试、编写脚本和组合固定流程；对 Agent 而言，MCP 可以直接描述可发现的工具、资源和参数，并在调用中携带项目上下文。StarCut 因此把 MCP 作为外部 Agent 的主要入口，将画布、时间线和媒体任务开放为领域能力，而不是让外部 Agent 操作屏幕坐标。

短小的项目编辑可以直接返回，视频生成和转写等后台工作则返回可以继续查询的 Task。MCP 的 [Tasks 扩展](https://modelcontextprotocol.io/extensions/tasks/overview) 提供了跨调用的任务生命周期，能够与 StarCut 的媒体 Task 建立明确映射。

内置 Agent 与外部 Agent 不必共享一段对话。StarCut 内置 Agent 保留自己的创作上下文，Codex 或 Claude Code 也保留各自的上下文；它们共同读取同一项目，并通过同一套语义化能力提交修改。外部调用需要浏览器执行时，也复用前面已经建立的客户端执行通道。

![MCP Canvas Bridge 将外部语义调用交给连接中的编辑器，并复用人类界面的领域写入路径](../assets/agent-loop/mcp-canvas-bridge.svg)

*入口和上下文可以不同，项目状态与领域写入路径保持一致。*

共享项目解决了结构状态的一致性，却不会自动解决创作意图冲突。两个 Agent 同时修改同一段内容时，系统可以避免重复执行和结构损坏，但“哪一个版本更符合创作目标”仍然需要任务归属、版本策略或人的决定。

<!-- [研发待补 A-C1] 固定 MCP 短调用与长 Task 的状态、版本、取消、超时和输出映射；使用同一批项目操作验证内部 Agent 与外部 Agent 的结果和错误语义，并覆盖多个 Agent 同时修改同一对象时的任务归属与意图冲突。 -->

## 六、长时间创作怎样处理上下文、记忆和 Skill

运行机制可以让创作跨越几小时，模型每次能够读取的内容仍然有限。这里要区分历史、记忆和项目状态承担的不同作用。

### 上下文压缩与记忆

- **完整历史**保留对话、工具调用和执行结果，服务于回看、恢复和审计；
- **上下文压缩**把较早的过程整理成当前模型可以继续使用的摘要；
- **用户记忆**保存相对稳定的个人偏好，以及当天仍然有效的背景；
- **项目记忆**保存项目约定、已经做出的创作决定及其原因；
- **项目模型**始终回答时间线、素材和画布现在是什么。

压缩产生新的模型读取视图，不删除原始历史。项目现状也不需要反复复制进聊天记录：每次行动前从当前项目重新读取，才能包含等待期间发生的人类编辑和其他 Agent 修改。

记忆更新还需要来源和版本边界。后台整理出的旧摘要如果晚于新决策返回，不能覆盖已经变化的项目约定；用户偏好、项目决定和当前工程也不应混成同一份自由文本。

### Skill 把创作经验带入当前任务

视频剪辑经验可以整理成 Skill，例如开场节奏检查、口播与画面对齐、字幕安全区、产品镜头选择和 Motion Graphics 制作流程。它为当前任务补充观察顺序、方法和约束，实际行动仍通过已有工具完成。

StarCut 在固定提示中只保留 Skill 的名称和简介。Agent 判断当前任务需要某项指导时，再按需加载完整内容或相关参考，避免所有领域知识长期占用上下文。

Skill 能否稳定改善节奏感、美感、运动感和网感，仍需通过真实成片验证。它可以把显性的剪辑流程变成可复用经验，但不能仅凭流程文本证明最后的创作判断已经成立。

<!-- [研发待补 A-X1] 用长 Session 比较压缩前后 token、目标保持率、Tool Call/Result 配对完整性和恢复时间；分别评估用户记忆与项目记忆的命中和污染。 -->
<!-- [研发待补 A-Q1] 建立包含节奏、信息密度、运动连续性、主体突出和风格一致性的真实任务集，对比无 Skill、流程 Skill、视觉反馈和人工中途介入。 -->

## 七、用视频问题检验 DeepSeek Harness

近期受到关注的 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) 提供了一套插件化 Agent Runtime。按照它公开的 [Architecture](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.md)，模型适配、工具、Session、Agent Loop 和上下文压缩都处在可替换边界上，运行时还提供持久 Session Event Log、统一 Inbox、Background Jobs 以及 UI 或 Editor 集成扩展点。这些能力与 StarCut 已有运行层存在实质重合。

前面的产品特写任务，正好给出了一组比“能否调用工具”更具体的检验题。

| 视频场景 | DeepSeek Harness 已提供的基础 | StarCut 仍需验证的边界 |
|---|---|---|
| 当前模型与工具循环 | 可替换 Agent Loop、模型和工具插件 | 能否承载现有视频工具语义并减少适配层 |
| 长对话与上下文压缩 | Session Event Log 与 Compaction | 如何接入用户记忆、项目记忆和当前项目读取 |
| 图片、视频后台生成 | Background Jobs 与统一 Inbox | 媒体 Task 的版本、产物、Monitor 和成组决策如何表达 |
| 浏览器执行编辑命令 | UI / Editor 扩展点 | 多窗口选择、执行权转移和副作用边界是否需要独立插件 |
| 内置与外部 Agent 操作项目 | 插件化工具运行时 | MCP 调用、内部调用与共享项目状态如何复用同一执行链路 |

其中，上下文压缩是最直接的接入候选。[Session](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/session.md) 从事件日志派生模型消息，[Compaction](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/compaction.md) 显式记录压缩过程，与 StarCut 的完整历史和模型读取视图较为接近。内循环也值得验证：如果现有工具与消息语义可以自然适配，插件化设计有机会减少通用模型运行代码。

视频运行时的检验更严格。Background Job 解决“工作可以在后台继续”，媒体 Task 还包含产物版本、生成进度和成组决策；Editor 扩展点解决“可以接入界面”，多窗口执行还需要选择唯一执行者并处理迟到结果；工具插件可以开放能力，共享视频项目仍由协同模型维护。这些不是对通用 Harness 的否定，而是判断哪些视频语义可以成为插件，哪些必须继续由 StarCut 的项目运行时负责。

真正的收益也不能只看是否成功接入。引入 DeepSeek Harness 以后，如果模型运行、事件日志、Inbox、Background Job 和 Compaction 能替换现有通用实现，系统概念会减少；如果外部再包一层相同的 Session、任务和结果通道，插件化反而会增加两套边界。接入实验应以减少独立维护的概念和适配代码为判断标准。

## 相关工作

[ReAct](https://arxiv.org/abs/2210.03629) 讨论推理与行动如何在一次 Agent 运行中交错；StarCut 的内循环属于这一问题范围。[Orleans 的 Virtual Actor](https://www.microsoft.com/en-us/research/publication/orleans-distributed-virtual-actors-for-programmability-and-scalability/)、[Azure Durable Functions](https://learn.microsoft.com/en-us/azure/durable-task/durable-functions/durable-functions-overview) 和 [Temporal](https://docs.temporal.io/) 则展示了持久状态、调度和恢复如何让工作跨越进程。Video Agent Loop 需要同时处理这两种时间尺度。

[SWE-agent](https://arxiv.org/abs/2405.15793) 表明 Agent-Computer Interface 会显著影响软件工程 Agent 的行为；[OSWorld](https://arxiv.org/abs/2404.07972) 展示了通用 GUI Agent 在真实计算机环境中的视觉定位和操作困难；面向后期制作的 [AgenticVBench](https://arxiv.org/abs/2605.27705) 开始进一步评估视频 Agent 的工具使用和失败方式。StarCut 关注的是这些能力进入一份持续变化、可以由人和多个 Agent 共同编辑的视频工程以后，运行过程怎样保持连续。

## 结语

回到那条缺少产品特写的视频：Agent 提交生成任务以后可以继续整理其他镜头；用户在等待期间照常修改时间线；素材回来时，Agent 从当前项目重新判断；编辑命令由当时在线的编辑器执行；Codex、Claude Code 或 StarCut 内置 Agent 都可以通过同一套项目能力继续工作。一次创作由许多段短暂运行组成，中间穿插着几秒到几分钟的等待、人的选择和来自不同入口的行动。

StarCut 的 Video Agent Loop 组织的正是这段完整过程。Vercel AI SDK 负责当前的模型与工具循环，持久 Task 和 Monitor 承接媒体生成，项目模型保留持续变化的视频工程，客户端通道把决定落实到真实编辑现场。每项机制都对应一个在创作中实际发生的问题，不需要依靠某个永久运行的模型进程把它们强行串在一起。

DeepSeek Harness 已经覆盖其中相当一部分通用运行能力。下一步不是重新描述两套架构，而是用同一批视频任务进行接入实验：它是否能替换现有内循环和上下文实现，Background Job 与插件边界能否自然承载媒体生成和浏览器执行，以及最终是否减少 StarCut 需要独立维护的概念。答案将决定它在 Video Agent Loop 中适合成为运行基础，还是只承担其中一部分通用能力。

## 相关资料

1. Vercel, [AI SDK: ToolLoopAgent](https://ai-sdk.dev/docs/reference/ai-sdk-core/tool-loop-agent).
2. Vercel, [AI SDK: Loop Control](https://ai-sdk.dev/docs/agents/loop-control).
3. DeepSeek AI, [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness), developer preview, 2026.
4. DeepSeek AI, [DeepSeek Harness Architecture](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.md), 2026.
5. S. Yao et al., [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629), 2022/2023.
6. P. Bernstein et al., [Orleans: Distributed Virtual Actors for Programmability and Scalability](https://www.microsoft.com/en-us/research/publication/orleans-distributed-virtual-actors-for-programmability-and-scalability/), Microsoft Research Technical Report, 2014.
7. Microsoft, [Durable Functions Overview](https://learn.microsoft.com/en-us/azure/durable-task/durable-functions/durable-functions-overview).
8. Temporal Technologies, [Temporal Documentation](https://docs.temporal.io/).
9. Model Context Protocol, [Tasks Overview](https://modelcontextprotocol.io/extensions/tasks/overview).
10. J. Yang et al., [SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering](https://arxiv.org/abs/2405.15793), 2024.
11. T. Xie et al., [OSWorld: Benchmarking Multimodal Agents for Open-Ended Tasks in Real Computer Environments](https://arxiv.org/abs/2404.07972), 2024.
12. Z. Cao et al., [AgenticVBench: Can AI Agents Complete Real-World Post-Production Tasks?](https://arxiv.org/abs/2605.27705), 2026.
13. Descript, [Underlord: Your AI Video & Podcast Editing Assistant](https://www.descript.com/underlord).
14. ChatCut, [What is ChatCut?](https://chatcut.io/docs/what-is-chatcut).
