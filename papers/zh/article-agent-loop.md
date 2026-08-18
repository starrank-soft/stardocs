# Video Agent Loop：StarCut 的硬核实践，DeepSeek Harness 能否满足视频场景？

![Agent Loop 封面：短暂内循环围绕唯一创作项目工作，持久外循环跨越消息、异步任务、休眠与人的接管](../assets/agent-loop/cover.png)

> 我们从视频创作出发，实现了一套能够跨越素材生成、用户修改、浏览器断线和服务重启的 Video Agent Loop。DeepSeek Harness 为其中一部分问题提供了另一套公开实现：哪些设计彼此印证，哪些视频需求仍然需要我们自己解决，换用它是否真的更简单？

## 引言：会操作时间线，不等于能完成一次视频创作

AI 已经可以读取素材、修改时间线、生成字幕，再根据预览结果继续调整。但真实的视频创作很少在一次模型响应里结束。

一张图片可能几分钟后才生成；一段视频可能经历排队、运行、失败和重试；用户会在等待期间改标题、替换素材，甚至直接重排时间线；同一个项目可能同时打开在多个浏览器窗口中；外部 Coding Agent 也可能通过 MCP 请求编辑当前项目。

这时，问题已经不再是模型会不会调用工具，而是：

- 视频、图片生成好之后，AI 怎么知道并继续创作？
- 用户在等待期间修改了项目，AI 怎样继续而不覆盖人的选择？
- 内置 Agent、外部 Agent 和多个浏览器，怎样操作同一个真实项目？
- 一次创作持续很久时，AI 怎样保留目标和重要决策？
- 浏览器断线或服务重启后，创作过程能不能继续？

我们设计并实现 Video Agent Loop 时，并没有以 DeepSeek Harness 为蓝本。现在把两套独立实现放在一起，可以看到它们解决了一部分相同的问题，也在关键位置选择了不同的边界。这不是“通用方案先进、垂直方案落后”的关系。

本文以 StarCut 已经实现的系统为主线。每解决一个视频创作问题，再对照 DeepSeek Harness 如何处理相同或相邻的问题。

## 一、为什么“会剪一下”还不等于“能完成一条视频”？

如果所有素材都已准备好，需求也不会改变，AI 可以在一段连续执行中完成很多操作：读取项目、选择素材、修改时间线、查看结果，再继续修正。

现实并不是这样。视频创作会跨越模型执行之外的时间：AI 发出生成请求后，结果尚不存在；用户不会停在原地等待；编辑器和服务进程也可能随时离线。如果把整次创作理解成一段永不结束的模型—工具循环，系统就必须长期保留进程、连接和内存状态，而且很难处理人的介入与多 Worker 竞争。

我们的处理方式，是把 Agent 的工作分成两个业务循环：

- **内循环**完成当前能够完成的工作：理解上下文、调用工具、检查结果，直到完成或暂时没有下一步；
- **外循环**关注后来发生的事实：新的用户要求、生成结果、时间事件和客户端返回，并在需要时启动下一次内循环。

一次内循环只是一个可结束、可替换的 Session Run。Session 和项目可以持续存在，但执行它们的 Process 不需要永久存活。Monitor 负责把外部变化整理成 AI 值得注意的信息；MCP Canvas 负责让外部 Agent 以语义方式作用于真实项目。

![双环 Agent 总体架构：外循环承接持久事件，内循环完成当前推理，Monitor 负责观察，MCP Canvas 负责行动](../assets/agent-loop/two-loop-architecture.svg)

*内循环完成当下，外循环跨越时间；观察和行动最终汇入同一份项目状态。*

### 对照 DeepSeek Harness

DeepSeek Harness 对当前模型—工具执行的处理与我们相近：一个 Turn 可以包含多个 Step，新输入可以进入下一 Step 或排队等待下一 Turn，Session 则在一次执行结束后继续存在。[Agent Loop 设计](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent-loop/README.md)

这说明“短暂执行、持久 Session”不是视频场景独有的判断。我们的不同起点是：外部媒体生成、浏览器 Editor、人类修改和外部 Agent 从一开始就是创作生命周期的一部分，而不是模型 Loop 外围的附加功能。

## 二、视频、图片生成好之后，AI 怎么知道并继续创作？

最简单的办法是让 AI 一直查询“生成好了吗”。但视频生成可能持续十几分钟，过程中还会产生大量相似进度。持续轮询既占用执行资源，也会让模型反复读取几乎没有决策价值的信息。

我们把生成状态保存在权威 Task 中。生成服务负责更新 Task；通知只表示“状态可能变了”，接收方仍然重新读取权威状态。这样即使通知重复、延迟或丢失，最终结果也不会只存在某条瞬时消息里。

AI 发起生成时，会记录这次工具调用正在等待哪个 Task。我们把这份等待关系称为 WaitPort。它不复制 Task 结果，只负责回答：结果属于哪次调用，现在应交给仍在执行的 Tool，还是留给下一次 Agent Run。

```text
AI 提交生成请求
  → 权威 Task 持续更新
  → WaitPort 保留调用关系
  → 结果进入统一 Inbox
  → 当前 Run 继续，或外循环启动下一次 Run
```

这意味着 AI 不需要关心自己届时是否还在同一个进程里。如果结果很快返回，当前 Tool 可以继续；如果生成时间很长，本次 Run 可以结束，结果仍会在稍后进入同一个 Session。

### 生成过程中的每次变化都要告诉 AI 吗？

不需要。用户希望界面连续显示 63% 变成 64%，模型却通常不能因为这 1% 做出新决定。UI 更新与 AI 注意力不是一回事。

Monitor 只观察当前 Session 真正等待的任务组。它读取 Inbox 中已经存在的版本化 Task Update，保留每项任务的最新状态，再把一批变化压缩成一条隐藏上下文。值得触发判断的通常是：

- 某项任务失败，需要重试或换方案；
- 一个阶段结果已经足以开始后续工作；
- 整组任务全部结束；
- 到达用户要求的检查点或截止时间。

![Monitor 将大量任务进度压缩为少数几次有决策价值的 Agent 唤醒](../assets/agent-loop/monitor-compression.svg)

*界面可以连续更新，AI 只在出现新的决策条件时重新工作。*

Task 仍然是状态真相，Monitor 不是第二个任务系统。通知丢失后，恢复扫描也可以根据 WaitPort 重新读取 Task 状态并补齐结果。

### 对照 DeepSeek Harness

DeepSeek Harness 的 Job Controller 已经解决了通用后台工作的完成通知：Agent 忙碌时，通知进入下一 Step；Agent 空闲时，可以开启新的 Turn；连续自动唤醒还有明确上限。[Background Job Controller](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/jobs/tool-jobs/README.md)

但它当前公开的 Job Contract 是进程内边界；Schedule 也要求原 Session 处于运行状态，不提供冷 Session 调度器。[Jobs 设计](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/jobs.md)、[Schedule 设计](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/schedule.md)

如果换成 DSH，Job 通知可以承担一部分“完成后告诉模型”的工作，但跨 Worker 的媒体 Task、权威状态对账、任务组压缩和冷 Session 唤醒仍然需要我们的 Task Provider、Monitor 与外循环。

<!-- [研发待补 A-M1] 固定 Monitor 的 cohort、版本合并、关闭条件与唤醒策略；记录原始 Task Update 数、聚合 Context 数、模型唤醒数及 token，用于计算注意力压缩率和成本。 -->

## 三、用户在等待期间改了项目，AI 怎样继续而不覆盖？

生成等待期间，用户可能替换素材、修改标题、删除一个片段，或者重新安排整个开场。如果 AI 保存一份开始执行时的项目快照，十分钟后再把旧快照整体写回，就会覆盖人的修改。

所以，Agent Session 不是视频项目的第二份真相。项目继续由编辑器的权威结构状态负责，人的编辑直接改变这份状态。AI 的上下文只保存目标、消息和必要引用；每次重新行动前，再从项目模型读取当前结构。

这也要求 AI 采用精确的领域操作，而不是重新生成整份项目：

```text
读取当前节点或时间线
  → 定位需要改变的对象
  → 提交局部语义操作
  → 由项目模型验证并修改权威状态
```

用户的补充要求进入 Session，用户对项目的直接修改则留在项目本身。下一次 Agent Run 同时看到最新要求和最新项目，不需要猜测哪份快照更新。

人类接管也因此不是异常路径。用户可以随时修改结果，AI 后续继续的是已经被人改过的项目，而不是试图恢复自己先前的完整计划。

### 对照 DeepSeek Harness

DeepSeek Harness 的 Session Event Log 很适合保存消息、工具调用和 Agent 运行事实，但它不会替产品定义视频项目的权威状态。即使内循环换成 DSH，我们仍然需要在每次运行时投影最新项目，并让工具通过同一套领域写路径提交局部修改。

DSH 可以帮助保存“Agent 发生过什么”，不能自动回答“当前视频项目究竟是什么”。这个边界不是它的缺陷，而是创作软件必须自己承担的数据建模责任。

## 四、不同来源的 AI，怎样安全操作人类正在编辑的同一个项目？

StarCut 同时面对两种 Agent：产品内部的 Agent 直接参与当前 Session，外部 Coding Agent 则通过 MCP 进入项目。两者最终都可能要求“把这个素材放进时间线”或“修改这个标题”。

如果分别为它们实现两套编辑接口，系统很快会出现两个画布：内置 Agent 使用真实编辑器，外部 Agent 修改服务端影子状态。两边的校验、协同、撤销和渲染结果会逐渐分叉。

我们的原则是：入口可以不同，项目真相和领域写路径只能有一份。

### MCP Canvas 表达对象与意图

外部 Agent 不应依赖屏幕坐标去点击和拖拽。窗口尺寸、缩放比例或布局一变，坐标就失去意义；而“将片段移动到标题轨下方”本来就是一个语义操作。

MCP Canvas 暴露稳定的项目句柄、对象身份和领域动作。MCP 负责连接与能力交换，视频项目模型负责解释操作语义并验证约束。[Model Context Protocol](https://modelcontextprotocol.io/specification/2026-07-28)

短小编辑可以直接返回结果；视频生成、转写和导出则返回可继续查询的 Task 句柄。MCP 的 [Tasks 扩展](https://modelcontextprotocol.io/extensions/tasks/overview) 可以表达这种跨调用生命周期，但协议 Task 只是内部权威 Task 的外部投影，不能因此再复制一套任务状态。

### 内置 Agent 与外部 MCP 共用调用运输

我们的统一 Inbox 不只承载对话消息，还能接收隐藏上下文、Tool Result 和版本化 Task Update。Session Channel 与 Project Channel 共用同一套 Message Stream、Inbox、WaitPort 和结果协议：

- Session Channel 拥有对话生命周期，会在需要时推动 Agent 继续；
- Project Channel 只承载项目级调用，不会为了执行一次 MCP Tool 伪造对话 Session。

因此，内置 Agent 和外部 MCP 不需要各自实现结果等待、超时竞争、断线恢复与客户端路由。它们共享调用运输，但保留不同地址和生命周期。

![DeepSeek Harness 在 Session 内统一消息与工具；Video Agent Loop 让 Session 与 Project Channel 共享同一套 Inbox 和执行运输](../assets/agent-loop/dsh-video-runtime-boundary.svg)

*我们统一的是调用关联、运输、恢复和结果语义，不是把所有业务对象塞进一个无类型队列。*

### 同一个项目打开多个窗口时，谁来执行？

很多编辑操作必须由持有实时项目状态的浏览器完成。多个标签页可以同时观察项目，但一条命令不能被所有窗口各执行一次。

Client Lease Coordinator 会从在线且具备能力的 Editor 中选择当前执行者。服务端发出的 Tool Assignment 带有执行身份；浏览器确认 Lease 后才执行，返回结果时再次校验身份。窗口断开或执行权变化后，旧窗口不能提交成一个有效结果。

Editor 最终调用的仍然是人类界面使用的领域写路径，因此 Agent 的操作会进入同一份项目状态，参与相同的协同、撤销与渲染。

![MCP Canvas Bridge 将外部语义调用交给连接中的编辑器，并复用人类界面的领域写路径](../assets/agent-loop/mcp-canvas-bridge.svg)

*外部 Agent、内置 Agent 和人类界面可以有不同入口，但不能拥有不同的项目真相。*

### 为什么同时存在 SSE 和 WebSocket？

两者承载的是同一套 Session Stream 语义，不是两套业务协议。

- SSE 适合服务端持续发送已提交消息、运行状态和模型增量，断线后从持久游标继续；
- WebSocket 除了读取同一事件流，还要双向传递 Lease、Tool Assignment 和 Tool Result。

完整消息来自同一提交记录，实时 Delta 只是低延迟投影，客户端结果最终仍进入统一 Inbox。选择哪种连接不会改变 Session 真相。

### 对照 DeepSeek Harness

DeepSeek Harness 的 MCP Client 会把远端 MCP Tool 注册进与内部 Tool 相同的 Tool Runtime。这对“让 DSH 中的 Agent 使用外部能力”很方便，也与我们追求统一工具政策的方向一致。[MCP Client](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/mcp/mcp-client/README.md)

我们的方向还包括协议另一端：StarCut 作为 MCP Server 接收外部 Agent 调用，再将其交给真实浏览器 Editor。DSH 当前的 MCP Client、Agent Inbox 和 Tool Runtime 不会直接提供 Project Channel、多窗口 Client Lease 或浏览器领域写路径。

它的 Host/Client 远程调用抽象可能减少普通控制接口的协议代码，但公开 API Gateway 也把流式数据和实体子流留给独立协议处理。[API Gateway](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/api-gateway.md) 因此，换成 DSH 不会消除我们对 SSE、WebSocket 和客户端执行的设计。

<!-- [研发待补 A-C1] 固定无在线 Editor、执行中断线、Lease 转移、结果已提交但响应丢失时的语义，并验证内部 Session Tool 与外部 Project Tool 得到一致结果。 -->
<!-- [研发待补 A-C2] 固定内部 Task 与 MCP Tasks 的状态、版本、取消、超时和输出映射，避免形成两套任务生命周期。 -->
<!-- [研发待补 A-T1] 测量 SSE 与 WebSocket 的重连恢复时间、游标重复率、Delta 丢失后的收敛时间，以及客户端执行在断线窗口中的结果归属。 -->

## 五、一次创作持续很久时，AI 怎样保留目标和重要决策？

把所有历史消息永久塞回模型，不是记忆，只是不断增长的上下文。视频创作持续越久，工具结果、生成状态和中间讨论越多；如果不治理，真正重要的创作目标反而会被噪声淹没。

我们把三个概念分开：

- **Session 历史**保存这次对话和工具执行已经发生的事实；
- **上下文压缩**把较早历史投影成可以继续推理的摘要；
- **记忆**保存跨 Session 仍然成立的信息，例如用户偏好、当天背景和项目约定。

压缩不会修改原始消息，只改变模型如何读取历史。记忆也不是对话的另一份副本，而是按用户长期、用户当日和项目三个范围保存完整快照。后台维护以已提交消息为来源，并携带来源边界；如果生成摘要或记忆期间源内容已经变化，较旧结果不能覆盖较新的状态。

项目本身仍然不进入记忆。时间线、素材和画布节点继续由项目模型保存；记忆只保存“为什么这样做”“用户偏好什么”“这个项目约定了什么”。

### 对照 DeepSeek Harness

这是 DeepSeek Harness 最值得认真对照的部分。它把 Session 设计成追加式事件日志，模型消息是日志的派生视图；Compaction 具有明确的开始、摘要和结束事件，能处理上下文压力、溢出恢复和 Tool Call/Result 配对。[Session 设计](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/session.md)、[Compaction 设计](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/compaction.md)

如果只替换 Session 历史和压缩内核，DSH 可能让通用机制更加统一。但 StarCut 的项目记忆、产品消息投影和视频项目状态仍然需要保留。这里值得做替换实验，而不是先得出谁一定更优。

## 六、浏览器断线或服务重启后，创作能不能继续？

创作连续性不能依赖某一个内存对象还活着。我们的每次 Session Run 都从权威历史和当前 Inbox 开始：先领取输入，运行模型与工具，把完整结果持久化，最后确认这批输入已经处理。

```text
恢复已提交历史
  → 领取 Inbox 输入
  → 读取最新项目上下文
  → 完成本次模型—工具执行
  → 持久化完整结果
  → 确认输入
```

如果进程在持久化之前退出，输入仍会重新出现；如果它在持久化之后、确认之前退出，下一次 Run 会再次领取，但按消息身份折叠后不会重复生成一个回答。

多个 Worker 同时收到运行提示时，只有取得 Run Lease 的 Worker 可以推进 Session。Lease 在运行期间续约，旧执行一旦失去所有权，就不能继续提交结果。Run 结束时，“确认 Inbox 为空并释放执行权”是一个原子边界，避免新输入刚好落在检查与释放之间。

运行提示只是降低延迟，不是正确性的唯一来源。恢复扫描会处理到期的时间事件、WaitPort 超时、Task 状态对账和未运行的非空 Inbox；多个扫描器即使重复发现同一 Session，也仍由 Run Lease 决定唯一执行者。

输出同样分为两层：完整消息进入可恢复的提交记录，模型 token 等实时 Delta 可以是低延迟、可丢失的投影。浏览器错过 Delta 后，会被下一条完整消息校正。

![一段 Agent 工作跨越多次短执行、异步等待、用户介入和进程重启](../assets/agent-loop/inner-outer-timeline.svg)

*Session 和创作任务持续存在，执行它们的 Process 可以结束、迁移和重建。*

### 对照 DeepSeek Harness

DeepSeek Harness 同样具有持久 Session、恢复、分叉和回放能力，也保证一个准备好的 Session 由一个具体 Driver 推进。对于单 Host Agent，这套生命周期很完整。

但视频生产环境还要求跨 Worker 的执行权、冷 Session 唤醒、权威媒体 Task 对账和浏览器执行恢复。DSH 当前公开的 Job 与 Schedule 生命周期没有直接覆盖这些边界。如果使用 DSH，较现实的做法仍是保留我们的 Inbox、Run Lease 和恢复扫描，由它们在取得执行权后驱动一个 DSH Agent；另一种做法则是为 DSH 实现分布式 Provider。两种方案都需要验证，不能把“支持 Session Resume”直接等同于完整的视频创作恢复。

## 七、把这些问题放在一起，一次视频创作如何完成？

假设用户要求：“根据脚本生成三张产品图，挑选合适的两张放进项目，再排成一个十秒开场。”

第一次 Agent Run 读取脚本和当前项目，提交三项生成请求，并记录每项结果属于哪次调用。它不需要留在内存里等待，可以在当前没有更多工作时结束。

生成服务继续更新 Task，界面实时显示进度。Monitor 不会因为每一个百分点唤醒 AI，而是在一张图失败以及全部任务结束时，分别形成有决策价值的上下文。

新的 Agent Run 读取生成结果和最新项目，比较候选图并选择两张。需要操作画布时，内置 Agent 的 Tool Call 通过 Client Lease 交给当前有效 Editor，由它使用正常领域操作修改项目。

此时用户手动换掉其中一张图，并补充“标题使用蓝色版本”。人的操作已经进入权威项目，新要求进入 Session。AI 下一次运行读取到的是人改过的项目，因此保留用户选择，只继续完成标题和十秒编排。

如果用户改用外部 Coding Agent，通过 MCP 发出同样的编辑请求，调用会进入 Project Channel，复用相同的客户端执行和结果协议，而不是修改一份服务端影子工程。

整个过程跨越了多次短暂执行、媒体生成、用户修改和浏览器连接变化，却不需要任何一个 Agent Process 永久存活。连续性来自事实之间可恢复的关系。

## 八、运行可靠之后，AI 就会剪出好视频吗？

不会自动发生。

Video Agent Loop 解决的是系统能力：AI 能否持续感知变化、在正确项目上行动、跨越等待和故障继续工作，并与人类共享同一份工程。它不直接解决节奏感、美感、运动感和网感。

这与早期人们担心 Coding Agent “会改代码但写不好代码”很相似。可靠的文件操作、测试与反馈循环没有替代工程判断，却让模型能够在真实环境中反复验证。视频 Agent 也需要类似的反馈条件：结构化项目信息、关键帧预览、播放结果、视觉评价和人的选择。

Skill 可能把一部分剪辑经验内化成可复用的方法，例如开场节奏检查、口播与画面对齐、字幕安全区和产品镜头选择。但 Skill 能否稳定提升创作质量仍有待实践；它可以约束过程，不能被预先当成审美能力的证明。

DeepSeek Harness 的 Skills、Subagents 和插件机制可以帮助组织这些方法，但通用 Harness 同样不会自动获得视频审美。无论使用哪套 Runtime，感知、评价与创作质量都需要单独的数据和实验。

<!-- [研发待补 A-Q1] 建立包含节奏、信息密度、运动连续性、主体突出和风格一致性的真实视频任务集，分别评估无 Skill、流程 Skill、视觉反馈和人工中途介入。 -->

## 九、如果换成 DeepSeek Harness，系统会不会更简单？

局部可能，全局目前不会。

| 视频创作问题 | StarCut 当前实现 | DeepSeek Harness 可提供什么 | 换用后的判断 |
|---|---|---|---|
| 完成当前模型—工具工作 | Session Run 与现有模型 SDK、UI Message、客户端 Tool 衔接 | Agent Loop、Tool Runtime、插件事件、取消 | 可替换实验 |
| 保留 Session 历史 | 权威消息历史与模型投影 | 追加式 Session Event Log、恢复、分叉、回放 | DSH 很有价值 |
| 控制上下文增长 | 后台压缩与来源边界 | 可恢复 Compaction、压力与溢出处理 | 值得重点对照 |
| 感知媒体生成结果 | 持久 Task、WaitPort、Monitor、恢复扫描 | Job 完成通知与 Session 内唤醒 | 仍需我们的媒体运行时 |
| 内外 Agent 共用项目 | Session/Project Channel、统一 Inbox、MCP Canvas | MCP Client 将远端 Tool 接入 DSH | 方向不同，不能直接替代 |
| 多窗口执行编辑命令 | Client Lease 与浏览器 Tool Runner | 通用 Harness 不拥有项目 Editor | 保留我们的实现 |
| 跨 Worker 与冷 Session 继续 | Run Lease、Task 对账、Sweep | Session Persistence；Job/Schedule 当前偏进程内和在线 Session | 需要分布式适配 |
| 浏览器实时通信 | 同源 SSE/WS Stream 与持久游标 | Host/Client 控制抽象 | 只能局部简化 |

DSH 的可取之处很明确：插件边界完整，Session Event Log 清楚，Tool Runtime、Compaction、Skills、Subagents 和通用 Job 都已经形成一套一致的运行框架。[DeepSeek Harness 架构](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.md)

我们的实现同样不是临时拼出来的补丁。Typed Inbox 把消息、上下文、工具结果和 Task Update 收敛到同一入站协议；WaitPort 统一当前执行与未来执行的结果关系；Run Lease 和恢复扫描处理分布式运行；Project Channel 与 Client Lease 让内外 Agent 操作同一个在线项目。这些正是视频业务最难、也最不适合被通用消息接口抹平的部分。

完整替换会保留上述视频运行时，再增加一层 DSH 适配，概念未必减少。更值得验证的边界是：保留 StarCut 的外循环、Inbox、Task 和 Editor Runtime，只替换一次 Session Run 内部的模型—工具引擎。

DeepSeek Harness 官方目前仍将项目标记为 Developer Preview，并提示接口可能快速变化。它适合进入替换实验和架构验证，但在证明收益之前，不适合仅凭热度成为视频生产链路的新核心依赖。[DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)

<!-- [研发待补 A-D1] 做一个可切换的 Session Run 实验：相同模型、工具、项目和 Inbox 输入，分别驱动现有内核与 DSH；比较适配复杂度、恢复正确性、工具成本、上下文 token 和端到端完成率。 -->

## 十、怎样证明这套设计真的解决了问题？

架构图只能说明意图。真正的证据应来自真实创作任务、故障与并发，而不只是一次顺利演示。

### 生成完成后，AI 是否真的能继续？

- 视频和图片生成完成后，到 AI 开始下一步的延迟是多少？
- 通知丢失、重复或乱序时，最终是否仍能感知结果？
- 同一任务结果会不会触发重复创作？

### 等待期间的人类修改是否被保留？

- 用户在素材生成期间修改项目，AI 后续是否读取最新状态？
- AI 是否只修改目标部分，而不是覆盖人的选择？
- 人类接管后继续交给 AI，任务完成率如何变化？

### 内外 Agent 是否真的操作同一个项目？

- 内置 Agent 与外部 MCP 执行相同操作时，结果和错误语义是否一致？
- 多个窗口同时在线时，一条命令是否只有一个有效执行者？
- Editor 断线、Lease 转移和响应丢失时，会不会产生重复副作用？

### Monitor 是否减少了无意义注意力？

- 原始 Task Update 被压缩成多少次模型唤醒？
- 压缩后是否错过需要决策的失败或阶段结果？
- token、响应延迟与任务完成率如何变化？

### DSH 是否真的让系统更简单？

- 替换 Session Run 后，减少了多少独立概念和维护代码？
- 为接回 Task、Inbox、Editor 与 UI，新增了多少适配层？
- 两种内核在崩溃恢复、Tool 顺序和上下文成本上有什么差异？

<!-- [研发待补 A-E1] 建立按持久化边界注入故障的测试集，保存消息、执行权、Task 版本、Tool Call 和项目事务的完整关联链。 -->
<!-- [研发待补 A-E2] 冻结常驻执行、主动轮询、StarCut 双环和 DSH 内核四组基线；记录完成率、恢复延迟、重复副作用、模型调用数和资源占用。 -->
<!-- [研发待补 A-E3] 建立内部 Agent、外部 MCP、像素 GUI 与服务端影子状态基线，使用同一批真实视频创作任务评估。 -->

## 十一、与已有工作的关系

这套设计并不是从零发明循环、消息或持久执行。它把几类成熟但经常在 Agent 产品中混在一起的问题，重新放进视频创作的运行边界。

[ReAct](https://arxiv.org/abs/2210.03629) 讨论推理与行动如何交错，对应本文一次 Session Run 内的模型—工具迭代。[Orleans 的 Virtual Actor](https://www.microsoft.com/en-us/research/publication/orleans-distributed-virtual-actors-for-programmability-and-scalability/)、[Azure Durable Functions](https://learn.microsoft.com/en-us/azure/durable-task/durable-functions/durable-functions-overview) 和 [Temporal](https://docs.temporal.io/) 已经说明，长生命周期实体不必依赖永久进程，状态、调度与恢复可以由持久运行时承担。

[SWE-agent](https://arxiv.org/abs/2405.15793) 强调 Agent-Computer Interface 会显著影响软件工程 Agent 的行为；[OSWorld](https://arxiv.org/abs/2404.07972) 展示了通用 GUI Agent 在真实计算机环境中的视觉定位和操作困难；面向真实后期制作任务的 [AgenticVBench](https://arxiv.org/abs/2605.27705) 也说明，执行框架会影响工具使用和失败方式。

与 [Reflexion](https://arxiv.org/abs/2303.11366) 一类跨尝试反思方法相比，本文的外循环不是新的推理策略。它不规定模型如何自我批评，而是让视频生成结果、人的修改和客户端执行能够可靠回到后续判断。

DeepSeek Harness 是最直接的工程对照：它把 Agent Loop、Session、Tool Runtime、Compaction、Job 和 MCP Client 组织成可组合插件；StarCut 则把相邻能力放进分布式 Session、持久媒体 Task 和浏览器 Editor 的生命周期。相同设计说明双方面对的是通用 Agent Runtime 问题，不同边界来自产品运行环境。

## 结语：让 AI 参与完整创作，而不只是完成一次调用

Video Agent Loop 要解决的不是怎样让一段代码循环得更久，而是怎样让 AI 在真实视频创作中持续参与：

```text
素材生成之后，它能够知道
项目被人修改之后，它能够理解
需要编辑时，它能够作用于真实项目
多个入口同时存在时，它不会重复执行
连接或服务中断之后，创作仍然能够继续
```

双环、Inbox、WaitPort、Monitor、Run Lease、MCP Canvas 和双 Transport 都只是实现这些目标的工程答案。它们不应该反过来成为文章的问题本身。

DeepSeek Harness 证明了通用 Agent Runtime 可以拥有清楚的插件与事件边界，也为内循环、Session 和上下文治理提供了很强的替换候选；StarCut 已经实现的媒体感知、跨进程外循环和项目级客户端执行，则是它目前不能直接替代的部分。

对照的价值，不是决定谁模仿谁，也不是把其中一方写成另一方的补丁，而是看清真正可以替换的边界，并用实验判断系统是否变得更简单、更可靠。

## 相关资料

1. DeepSeek AI, [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness), developer preview, 2026.
2. DeepSeek AI, [DeepSeek Harness Architecture](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.md), 2026.
3. S. Yao et al., [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629), 2022/2023.
4. P. Bernstein et al., [Orleans: Distributed Virtual Actors for Programmability and Scalability](https://www.microsoft.com/en-us/research/publication/orleans-distributed-virtual-actors-for-programmability-and-scalability/), Microsoft Research Technical Report, 2014.
5. Microsoft, [Durable Functions Overview](https://learn.microsoft.com/en-us/azure/durable-task/durable-functions/durable-functions-overview).
6. Temporal Technologies, [Temporal Documentation](https://docs.temporal.io/).
7. Model Context Protocol, [Specification 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28).
8. Model Context Protocol, [Tasks Overview](https://modelcontextprotocol.io/extensions/tasks/overview).
9. J. Yang et al., [SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering](https://arxiv.org/abs/2405.15793), 2024.
10. T. Xie et al., [OSWorld: Benchmarking Multimodal Agents for Open-Ended Tasks in Real Computer Environments](https://arxiv.org/abs/2404.07972), 2024.
11. Z. Cao et al., [AgenticVBench: Can AI Agents Complete Real-World Post-Production Tasks?](https://arxiv.org/abs/2605.27705), 2026.
12. N. Shinn et al., [Reflexion: Language Agents with Verbal Reinforcement Learning](https://arxiv.org/abs/2303.11366), 2023.
