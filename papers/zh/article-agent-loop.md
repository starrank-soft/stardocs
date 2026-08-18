# Video Agent Loop：StarCut 的硬核实践，DeepSeek Harness 能否满足视频场景？

![Agent Loop 封面：模型与工具的内循环工作在持续变化的视频项目中，外层创作循环连接用户、媒体任务、编辑器与外部 Agent](../assets/agent-loop/cover.png)

## 引言

AI Coding 从代码补全走向 Coding Agent，背后是一套能够持续工作的 Agent Loop：模型读取仓库、修改文件、运行命令和测试，再根据结果继续下一步。代码仓库为这个循环提供了成熟的工作环境：源文件表达工程状态，文件与终端提供稳定入口，编译器和测试返回执行反馈，人也可以随时检查并接管同一份工程。

创作软件正在经历相似的变化。[Descript](https://www.descript.com/underlord)、[ChatCut](https://chatcut.io/docs/what-is-chatcut) 等产品已经开始让 Agent 理解素材、生成初剪，并通过自然语言修改时间线。生成能力也进入了同一条工作流：一条产品视频做到一半，可能发现缺少合适的产品特写，于是临时生成图片，再把图片扩展为几秒钟的动态镜头；旁白、音乐、字幕和 Motion Graphics 也会陆续补充。剪辑、理解和生成在同一个项目中交替进行，素材集合与时间线都在持续变化。

新的工作方式也带来了一组问题。图片和视频生成需要十几秒到几分钟，生成完成后，Agent 如何感知结果并继续创作？等待期间用户可能已经调整过时间线，返回的素材又该怎样进入当前项目？除了 StarCut 内置 Agent，Codex、Claude Code 等外部 Agent 也可能操作同一个工程，不同入口如何共享项目状态并协调行动？浏览器掌握编辑现场，服务端负责模型调用和媒体任务，回答、指令与执行结果如何在两端之间持续传递？

围绕这些问题，StarCut 已经形成了一套连接媒体生成、项目协同、分布式调度和浏览器执行的完整 Video Agent Loop。近期 DeepSeek Harness 的讨论让 Agent Harness 再次受到关注。这套现成的业务链路正好提供了一组检验题：DeepSeek Harness 是否解决了这些问题，插件化设计能否减少系统复杂度。

## 一、Video Agent Loop 到底包含什么

Agent Loop 最容易理解的一层，是模型和工具之间的连续配合：模型判断下一步，发起 Tool Call，取得 Tool Result，再决定继续使用工具还是给出回答。

StarCut 使用 Vercel AI SDK 的 [`ToolLoopAgent`](https://ai-sdk.dev/docs/reference/ai-sdk-core/tool-loop-agent) 完成这一层。模型接入、多步 Tool Calling、参数校验、流式输出和停止条件，都可以交给成熟的通用框架处理。我们把它称为**内循环**，它负责解决“当前这件事下一步做什么”。DeepSeek Harness 的插件化 Agent Loop 也主要覆盖这一通用运行层，后文会结合视频业务留下的边界再作判断。

视频创作还有一个更长的过程。Agent 发出的工作可能由服务端完成，可能交给正在打开项目的浏览器，也可能变成持续几分钟的媒体任务；用户会在等待期间继续修改项目，Codex、Claude Code 等外部 Agent 也可能从另一个入口加入。几秒或几分钟后，新结果回来，Agent 需要面对当时的项目状态继续判断。我们把这段过程称为**外循环**或**业务循环**。

![Video Agent Loop 总体架构：模型工具内循环与持续创作外循环共同围绕一份项目工作](../assets/agent-loop/two-loop-architecture.svg)

*内循环负责当前判断，外循环负责让工作跨越等待、执行位置和人的介入继续推进。*

外循环不依赖一个始终运行的模型进程。真正持续存在的是几类业务事实：

- 对话历史保存用户要求、模型回答、Tool Call 和正式结果；
- 项目模型保存当前时间线、素材、文档和协同状态；
- Task 保存生成、转写等后台工作的进展和产物；
- Inbox 保存尚未被 Agent 消费的新消息与外部结果；
- 执行权记录确定当前由谁思考、由哪个浏览器真正执行命令。

一次连续思考可以结束。只要这些事实仍然存在，新的消息和结果到达以后，创作就可以从当时的工程状态继续。

## 二、Tool Call 的结果从哪里来

同样是一项 Tool Call，在 StarCut 中可能由完全不同的执行者完成。

- 查询模型、加载 Skill、准备素材资源，适合在服务端执行；
- 读取当前时间线、修改项目、播放、抽帧和渲染，需要交给打开项目的 Editor；
- 图片、视频、音频生成与转写，由独立的 Task Worker 完成；
- 信息不足或需要创作取舍时，结果来自用户回答。

如果每一类执行者都建立自己的返回协议，模型循环就必须同时理解服务端回调、浏览器消息、任务通知和人机交互。StarCut 的做法是把执行位置和结果协议分开。

Tool Call 完整形成以后，系统先保存这项调用，再交给相应执行者。模型流式输出的参数可以用于提前显示编辑过程，但正式副作用以已经提交的完整调用为准。这样，即使当前模型执行中断，尚未完成的工作仍然有据可查。

无论结果来自哪里，都携带原来的 `toolCallId` 回到同一个 Inbox。当前内循环仍在等待时，它领取匹配结果并继续；当前思考已经结束时，结果留在 Inbox 中，由外循环接手。执行者可以不同，Agent 看到的仍然是一套完整的 Tool Call—Tool Result 关系。

![Tool Call 提交后分发给服务端、浏览器、后台任务或用户，结果统一回到 Inbox](../assets/agent-loop/tool-execution-routing.svg)

这条统一返回路径是后面所有机制的基础。媒体任务可以晚几分钟回来，浏览器可以断线重连，用户可以过一会儿再回答，模型循环不用因此分裂成几套实现。

## 三、图片、视频生成完成后，Agent 如何感知

假设 Agent 要为一段产品视频同时尝试十张候选图。在当前接入的生成链路中，一张图片通常需要 10–20 秒，一段视频可能需要 200–600 秒，具体时间还会受到模型、队列和参数影响。

如果每次 Tool Call 都等图片生成完再返回，十张图会被生成时间串行阻塞。视频生成更不适合把模型执行和网络连接一直挂在原地。

StarCut 将“请求已经受理”和“工作最终完成”分开。生成请求提交以后，Agent 立即取得任务身份和预期产物信息，可以继续提交剩余任务或处理其他镜头；后台 Worker 独立、并行地完成这些工作。

### 同一套调用关系，支持不同等待方式

并非所有工具都应该立即交回。StarCut 当前有三种等待策略：

| 等待方式 | Agent 当下得到什么 | 典型工作 |
|---|---|---|
| 立即交回 | 请求已受理和任务身份 | 图片、视频、Motion Graphics、SVG 生成 |
| 有限等待 | 限时内完成就直接取得结果，否则转入后台 | 转写，当前等待 30 秒 |
| 等待结果 | 取得匹配结果以后才能继续，超时则失败 | 普通查询、读取和编辑工具 |

等待方式不同，结果仍然沿用同一个调用身份和 Inbox。它不是三套异步系统，只是同一套协议面对不同业务耗时的三种策略。

![媒体任务提交后可以立即交回、有限等待或等待结果，后台 Task 独立继续](../assets/agent-loop/media-task-lifecycle.svg)

### Tool Call 已经返回，最终结果怎么办

立即交回解决了并发，却带来一个新的问题：原 Tool Call 已经取得“请求已受理”的结果，最终图片不能再成为同一个调用的第二个 Tool Result。

系统需要保存这项调用和后台 Task 之间的返回关系：它来自哪次 Tool Call、属于哪段创作、应该回到哪个 Inbox、现在仍在等待还是已经转入后台。StarCut 内部用 WaitPort 表达这份关系。它像一张回执，但不保存另一份 Task，也不是第二个结果队列。

Task 每次提交新的权威版本后，变化都会先进入 Inbox。这里还缺最后一步：后台任务说的是状态，模型需要的是一次新的感知。

### Monitor 为什么存在

如果没有 Monitor，图片照样会生成，项目里也能看到结果，但 Agent 不会自然知道该继续。用户需要再说一句“图片生成好了，你继续”，或者 Agent 必须不断轮询任务。

Monitor 是 Task 系统旁边的感知 sidecar。它不执行任务、不拥有对话，也不维护另一份任务列表；它只把已经进入 Inbox 的 Task 更新转换成 Agent 能够理解的新输入。

十张图中有两张完成时，Agent 不只收到两个孤立的“完成”事件，还可以看到这一组工作的当前概览：总共十项、两项成功、八项仍在运行，以及已经产生的两个 Artifact。Agent 可以先检查这两张、先放入时间线、继续等待，或者调整剩余生成策略。

![每个 Task 新版本进入 Inbox，Monitor 将任务组状态转换成 Agent 的新感知](../assets/agent-loop/monitor-compression.svg)

*Monitor 自动完成“让 Agent 知道发生了什么”，后续怎样行动仍由 Agent 根据创作目标决定。*

当前实现让每个已经提交的权威 Task 版本进入 Inbox；同一批更新中如果一个 Task 出现多个版本，Monitor 只保留最新版本。未来是否需要控制模型继续判断的频率，应该通过数据评估，不能靠丢掉任务事实实现。

### 素材回来时，时间线可能已经变了

等待生成期间，用户可能移动了 Clip、替换了素材，甚至取消了原来的安排。Agent 不能拿几分钟前保存的整份时间线覆盖当前项目。

StarCut 将创作目标与项目现状分开：对话保留“为什么做”和原计划，当前时间线始终由项目模型维护。Monitor 把新结果交给 Agent 后，延迟行动应先读取受影响对象的当前状态，再把仍然适用的计划落实为局部操作。

```text
新素材完成
  → Agent 收到任务更新
  → 读取受影响对象的当前状态
  → 判断原目标是否仍然成立
  → 提交局部修改
```

自动感知让创作能够继续，项目工具提供当时的工程事实。是否能稳定做到“先读当前状态、再应用延迟结果”，仍需要通过人机并行编辑任务验证，不能只依赖提示词上的约定。

<!-- [研发待补 A-M1] 按图片、视频、转写记录 Task 耗时 P50/P95、并发度、Task 版本数、Monitor 上下文数、模型继续次数和端到端感知延迟。 -->
<!-- [研发待补 A-P1] 建立生成等待期间的人机协同任务，验证 Agent 恢复后不覆盖用户选择。 -->
<!-- [研发待补 A-P2] 验证延迟任务恢复后，Agent 是否读取受影响对象的当前状态；评估是否需要版本前置条件等协议保证。 -->

## 四、内置 Agent 怎样把决定落实到真实编辑器

Agent 决定“把产品特写放到开场第三秒”以后，还要把它准确写进同一份视频工程。

StarCut 将项目组织成 Agent 可读取的虚拟文件视图，并通过语义化工具开放查找、读取、创建和局部编辑能力。Agent 操作的是项目对象和领域语义，而不是屏幕坐标。当前时间线已经由 Y.Doc 建模并支持协同，Agent 不需要在真实 Editor 之外再维护一条 AI 专用时间线。

有些工具可以由服务端执行，但读取当前编辑现场、修改时间线、控制播放、抽帧和渲染必须进入打开项目的浏览器。浏览器收到正式 Tool Call 后，沿用人类界面相同的项目写入路径完成操作；协同、撤销、验证、预览和导出看到的是同一次修改。

### 同一个项目打开多个窗口，谁来执行

同一项目可能同时出现在浏览器标签页、桌面端和另一个工作窗口。所有窗口都能看到项目和 Tool Call，但不能都执行。

StarCut 会从在线、具备所需能力并满足读写权限的 Editor 中选择一个执行者。窗口需要确认自己已经准备好；每项 Tool Call 还会获得独立的执行凭证，返回结果时再次校验。旧窗口断线后即使带回迟到结果，也不能越过已经转移的执行权。

可安全重试的读取操作可以改派另一个窗口。已经开始产生副作用的写入和渲染不能盲目重放，需要重新检查项目状态。这是多窗口调度必须尊重的业务边界。

还要区分两种并发：同一段对话在多个窗口打开时，所有窗口可以观察和发送消息，但同一对话只进行一份模型思考；不同对话可以同时操作同一项目，Y.Doc 负责结构协同，创作意图冲突仍需要任务归属或人的决定。

![多个窗口可以共同观察项目，但同一项编辑命令只交给一个具备执行权的 Editor](../assets/agent-loop/client-execution-lease.svg)

*项目级执行权选择窗口，单次调用凭证约束这一次结果；两者共同避免重复编辑和旧结果回写。*

<!-- [研发待补 A-C2] 覆盖多窗口竞争、执行中断线、执行权转移、旧结果晚到和可重试/不可重试操作，验证没有重复副作用。 -->

## 五、回答、指令和执行结果如何在两端交回

Agent 一边输出回答，一边可能要求浏览器执行编辑。前者主要从服务端持续流向界面，后者需要服务端分配执行权、浏览器执行并把结果送回。StarCut 没有把它们强行塞进一条连接。

- **Session Stream** 负责对话历史快照、已提交消息、模型 Delta、token 使用和活动状态。默认使用 SSE，部署需要时也可以使用 WebSocket。
- **Project Stream** 使用 WebSocket，负责 Editor 在线状态、项目级执行租约、客户端 Tool Call 分配、撤销与结果回传。

![Session Stream 交付对话，Project Stream 协调浏览器执行；两者共享持久业务状态](../assets/agent-loop/dual-transport.svg)

这不是简单的“SSE 与 WebSocket 谁更好”。Session Stream 以服务端向界面持续交付为主，SSE 足够直接；Project Stream 需要双向的在线执行协调，更适合 WebSocket。

Transport 不拥有业务状态。模型 Delta 用来降低等待感，可以丢失；完整消息和 Tool Call 已经持久化，Task 与项目也各自保存权威状态。浏览器错过实时增量后，可以从已提交快照继续；无论结果通过 HTTP 还是 WebSocket 回来，最终都按调用身份进入 Inbox。

<!-- [研发待补 A-T1] 测量 SSE/WS 断线恢复、游标重复、增量丢失后的收敛时间，以及客户端结果回传到模型继续判断的延迟。 -->

## 六、Codex、Claude Code 如何控制同一个项目

StarCut 内置 Agent 已经处在当前项目和创作上下文中。Codex、Claude Code 等外部 Agent 则拥有自己的模型、对话和 Tool Loop，它们需要的是一个稳定的项目入口。

CLI 适合人类调试、脚本和固定流水线。对 Agent 而言，MCP 可以直接描述可发现的工具、资源和参数，并携带授权后的项目上下文。StarCut 因此把 MCP 作为外部 Agent 的主要入口，而不是让外部 Agent 模拟鼠标操作画布。

外部 Agent 不需要共享 StarCut 的内部对话。每个 MCP 上下文绑定用户和项目；需要浏览器能力时，还可以绑定到特定 Editor 连接。普通项目 Tool Call 进入项目消息通道，复用前面的客户端执行机制；生成和转写进入同一 Task 系统。

![外部 Agent 通过 MCP 进入项目通道，并复用同一项目模型、客户端执行与结果路径](../assets/agent-loop/mcp-canvas-bridge.svg)

内置 Agent 的后台任务由 Monitor 主动转成新输入。外部 Agent 当前通过 `poll` 观察异步结果，因为它自己的继续机制由 MCP Host 管理。两者可以有不同上下文和继续方式，但共享同一项目模型、Artifact、语义化工具和执行结果。

共享项目解决了结构状态的一致性，不会自动解决创作意图冲突。两个 Agent 同时修改同一段内容时，系统可以避免影子状态和重复执行，“哪个版本更符合目标”仍需要任务归属、版本检查或人的决定。

<!-- [研发待补 A-C1] 用同一批项目操作验证内部与外部 Agent 的结果语义，并覆盖两个 Agent 同时修改同一对象的冲突。 -->

## 七、如何压缩上下文、管理记忆和加载 Skill

创作可以持续数小时，但模型不能每次读取全部历史、全部项目和全部领域知识。StarCut 把上下文分成几层：

- **完整历史**保存用户消息、模型回答、Tool Call 和结果，是可追溯事实；
- **压缩视图**从完整历史派生，控制模型输入规模，不删除原始历史；
- **个人长期记忆、个人当日记忆和项目记忆**分别保存不同生命周期的信息；
- **当前项目**由工具按需读取，不整体复制进对话；
- **Skill**只先暴露名称和用途，需要时再加载完整内容与参考。

上下文压缩本身也是持久后台工作。压缩完成后，结果通过同一 Inbox 进入后续思考，因此不依赖原来的模型执行一直存活。记忆刷新则可以静默进行，不需要每次完成都打断当前创作。

![完整历史、压缩视图、分层记忆、当前项目与按需 Skill 共同组成模型输入](../assets/agent-loop/context-memory-skill.svg)

Skill 可以整理开场节奏检查、口播与画面对齐、字幕安全区、产品镜头选择和 Motion Graphics 制作流程。它能把显性的经验变成 Agent 可调用的方法，但“写进 Skill”不等于已经掌握节奏感、美感、运动感和网感。这部分需要通过真实成片、人工评价和返工次数验证。

DeepSeek Harness 的 Session Event Log 和 Compaction 与这里的完整历史、模型视图存在直接重合，可能减少通用上下文代码；项目现状和剪辑 Skill 仍然属于 StarCut 的领域上下文，不能被通用会话摘要替代。

<!-- [研发待补 A-X1] 比较压缩前后 token、目标保持率、Tool Call/Result 完整性和恢复时间。 -->
<!-- [研发待补 A-Q1] 对比无 Skill、流程 Skill、带案例 Skill 和人工中途介入的成片质量。 -->

## 八、系统中断以后，创作怎样继续

用户消息、浏览器结果和后台任务更新，都会先进入持久 Inbox，再发送低延迟的唤醒提示。Task 也先提交权威状态，再广播“它发生了变化”。提示可以重复或丢失，因为周期性的恢复检查会从尚未处理的 Inbox、开放调用和 Task 状态重新发现工作。

Inbox 条目被读取以后，不会立刻删除。只有它已经写入权威对话历史，系统才确认消费；中途崩溃时，下一次仍能重新读取同一输入。消息身份和版本使重复内容可以安全折叠。

![一次视频创作跨越多次有限思考、异步任务、用户介入和故障恢复](../assets/agent-loop/inner-outer-timeline.svg)

不同租约保护不同资源：对话租约避免同一段对话同时进行两次模型思考；Task 执行凭证阻止旧 Worker 继续写任务；客户端执行凭证阻止旧窗口提交迟到结果。它们共同保证持续创作可以恢复，又不会把所有并发问题塞进一把大锁。

这套系统追求的是持久事实最终可恢复、同一对话串行、Task 版本单调和客户端结果受执行权约束。外部模型供应商提交与本地记录之间、浏览器已经产生但尚未回报的副作用，仍然需要业务幂等键、状态重读或人工确认，不能笼统声称端到端 exactly-once。

<!-- [研发待补 A-R1] 在消息提交、Inbox 确认、Task 提交、任务通知、Monitor 转换和执行权转移后分别终止实例，统计遗漏、重复和恢复时间。 -->

## 相关工作

[ReAct](https://arxiv.org/abs/2210.03629) 讨论模型如何交错推理与行动，对应 Video Agent Loop 的内循环。[Orleans 的 Virtual Actor](https://www.microsoft.com/en-us/research/publication/orleans-distributed-virtual-actors-for-programmability-and-scalability/)、[Azure Durable Functions](https://learn.microsoft.com/en-us/azure/durable-task/durable-functions/durable-functions-overview) 和 [Temporal](https://docs.temporal.io/) 展示了持久状态、调度与恢复如何让工作跨越进程。

[SWE-agent](https://arxiv.org/abs/2405.15793) 表明 Agent-Computer Interface 会影响软件工程 Agent 的行为；[OSWorld](https://arxiv.org/abs/2404.07972) 展示通用 GUI Agent 在真实计算机环境中的操作困难；[AgenticVBench](https://arxiv.org/abs/2605.27705) 开始评估真实后期制作中的工具使用和失败方式。StarCut 关注的是这些能力进入一份持续变化、由人和多个 Agent 共同编辑的视频工程以后，运行过程怎样保持连续。

## 结语

StarCut 的 Video Agent Loop 由两层循环组成：Vercel AI SDK 负责当前模型与工具的连续交互；外层业务循环通过 Inbox 接回用户、服务端、浏览器和后台任务产生的新事实。等待策略与 Monitor 让长媒体任务不阻塞当前工作；项目模型为迟到结果提供当时的工程事实；双 Transport 连接对话交付与浏览器执行；租约避免多实例和多窗口重复行动；MCP 让外部 Agent 复用同一项目能力；压缩、记忆和 Skill 则维持较长创作中的上下文。

这些机制分别对应视频创作中真实发生的问题，又围绕少数权威状态收敛在同一业务循环中。接下来的研发重点不只是继续增加功能，还要用数据验证并发生成、自动续作、故障恢复、多窗口协同和 Skill 对创作质量的实际作用。

回到标题中的 DeepSeek Harness：它的模型适配、工具管线、事件日志、统一 Inbox 和上下文压缩，确实可能替换 StarCut 的一部分通用运行基础。检验标准不是能否接入，而是替换以后独立维护的概念、适配代码和恢复逻辑是否减少。媒体 Task 与 Artifact 的关系、Monitor 的任务组感知、浏览器执行权、MCP 项目绑定和 Y.Doc 协同，则仍然属于 Video Agent Loop 的视频业务层。

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
