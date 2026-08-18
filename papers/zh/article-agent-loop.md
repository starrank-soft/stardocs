# Video Agent Loop：StarCut 的硬核实践，DeepSeek Harness 能否满足视频场景？

![Agent Loop 封面：短暂内循环围绕唯一创作项目工作，持久外循环跨越消息、异步任务、休眠与人的接管](../assets/agent-loop/cover.png)

> 我们从视频创作出发，实现了双环、统一 Inbox、Monitor 与 MCP Canvas。DeepSeek Harness 为相同问题提供了另一套公开实现：哪些设计彼此印证，哪些边界并不相同，换用它是否真的更简单？

## 引言：会调用工具，不等于能够持续工作

DeepSeek Harness 最近让更多人开始讨论 Harness。这个讨论很重要：模型只是 Agent 的一部分，上下文怎样组织、工具怎样执行、Session 怎样恢复，以及新消息怎样进入正在运行的 Agent，同样决定了它能不能完成工作。

我们设计并实现这套 Video Agent Loop 时，并没有以 DeepSeek Harness 为蓝本。现在把两套独立实现放在一起，可以看到它们解决了一部分相同的问题，也在关键位置选择了不同的边界。这不是“通用方案先进、垂直方案落后”的关系。

这篇文章以我们的实现为主线。每遇到一个核心问题，我们先说明 Video Agent Loop 怎么做，再对照 DeepSeek Harness 会怎么处理，并追问一个更实际的问题：**如果换成 DeepSeek Harness，系统真的会更简单吗？**

## 一、我们为什么把 Agent 分成两个循环

一个 AI 可以连续调用十几个工具，并不意味着它已经成为一个可靠的视频 Agent。

在一次模型响应里，AI 可以读取项目、修改内容、查看结果，再继续修改。这类“思考—行动—观察”适合处理当前几秒钟内能够完成的工作。真正困难的是：如果素材要十分钟后才能生成，用户在等待期间补充了要求，浏览器断线又重连，执行进程刚好在发布时被替换，Agent 还能不能知道自己在等什么，并从正确的位置继续？

我们把这个过程划分为两个业务循环：

- **内循环**负责完成当前这一步：读取上下文、调用工具、检查结果，直到完成或进入等待；
- **外循环**负责跨越时间：接收后来发生的事实，在需要时启动一次新的内循环，并从持久状态继续。

网络监听、重连、租约续期和遗漏扫描不是新的业务循环，它们只是交付与恢复机制。Monitor 虽然持续观察任务，也不是第三个会思考的 Agent。这样划分的目的，是避免每个基础设施组件都保存一份“Agent 现在做到哪里”的状态。

![双环 Agent 总体架构：外循环承接持久事件，内循环完成当前推理，Monitor 负责观察，MCP Canvas 负责行动](../assets/agent-loop/two-loop-architecture.svg)

*内循环是短暂计算，外循环由持久事实驱动；观察面和行动面最终汇入同一份项目状态。*

DeepSeek Harness 对当前模型—工具执行的划分与我们基本一致：一个 Turn 可以包含多个 Step，Agent 在 Step 之间接收新的输入，执行结束后 Session 仍然可以恢复。[Agent Loop 设计](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent-loop/README.md)

区别在于我们从一开始就把外部世界作为 Loop 的组成部分：媒体 Task、浏览器 Editor、外部 MCP Agent 和人类操作都可能决定下一次执行何时发生。双环不是对 DeepSeek Harness 的补充说明，而是我们从视频创作生命周期出发形成的总体模型。

## 二、我们的内循环：一次可以被替换的 Session Run

每次内循环都是一个有边界的 Session Run。它启动时从权威存储恢复历史，领取当前 Inbox 输入，把消息、上下文或工具结果折叠进历史，然后运行模型与工具的多步迭代。执行中的增量结果可以实时显示，但只有完整消息和工具检查点会成为可恢复事实。

```text
恢复 Session 历史
  → 领取 Inbox 输入
  → 投影当前项目上下文
  → 模型与工具多步执行
  → 持久化完整结果
  → 确认 Inbox 输入
  → 结束本次 Run
```

这里有一个刻意的顺序：Inbox 输入先被领取到处理中区域，只有当结果成功持久化后才确认删除。如果进程在两者之间退出，下一次 Run 会重新领取同一批输入；折叠过程按消息身份保持幂等，因此不会因为恢复而生成重复对话。

这个设计允许 Process 结束、迁移或被部署替换。Agent 的连续性保存在 Session 历史和 Inbox 中，不绑定某个 JavaScript 对象或内存 Promise。

上下文压缩和记忆也被分成两个问题：压缩负责把一段 Session 历史投影成可继续推理的摘要；记忆负责保存跨 Session 仍然成立的用户偏好、当日信息和项目约定。二者都以已经提交的消息作为来源，并通过来源边界避免较晚完成的后台维护覆盖较新的状态。

### 对照 DeepSeek Harness

DeepSeek Harness 也拥有完整的内循环、追加式 Session 事件日志、工具执行管线、取消和恢复机制。它进一步把压缩记录为 Session 事件，并把 Agent Loop 周围的能力做成插件。[Session 设计](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/session.md)、[Compaction 设计](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/compaction.md)

如果只看模型与工具迭代，换用 DeepSeek Harness 可能减少我们维护 Loop、工具政策和上下文压缩的工作。但我们的内循环已经和项目上下文投影、客户端工具检查点、流式 UI Message 以及现有模型 SDK 紧密衔接。替换并不是删掉一段 Loop，而是把这些边界重新适配到 DSH 的 Session Event 和 Tool Runtime。

所以这一部分的结论不是“DSH 更好”，而是：**两种实现解决了相同问题。DSH 把插件与事件边界表达得很清楚；我们的实现把分布式恢复、视频项目上下文和客户端工具执行接进了同一个 Run。是否更简单，要由适配成本而不是代码行数决定。**

## 三、我们的统一 Inbox：不只接收对话消息

Video Agent Loop 的核心不是有一条消息队列，而是把所有能够推进 Session 的外部事实收敛到同一种持久入站协议。Inbox 目前可以承载四类输入：

- 用户可见的消息；
- 只进入模型上下文的隐藏信息；
- 客户端或服务端返回的 Tool Result；
- 长时 Task 的版本化状态更新。

Tool 发起跨边界工作时，会先打开一个 WaitPort，记录这次调用正在等待什么。客户端执行结果、Task 完成、超时和恢复扫描最终都通过这个 WaitPort 回到同一 Inbox。生产者不需要决定“结果应该送给当前 Promise，还是送给下一次 Agent Run”：如果原 Run 仍在等待，它可以领取结果继续当前 Step；如果 Run 已经结束，结果会留在 Inbox，由外循环启动下一次 Run。

Monitor 也不维护第二条结果队列。Task 的原始状态更新已经在 Inbox 中，Monitor 只把属于同一批任务的高频更新原子地压缩成一条隐藏上下文，再放回同一个 Inbox。

统一还发生在外部 MCP 调用上。我们的 Session Channel 和 Project Channel 共享同一套 Message Stream、Inbox、WaitPort 与 Tool Result 语义：

- Session Channel 拥有对话生命周期，会触发 Agent Run；
- Project Channel 只承载项目级 Tool Call，不会伪装成一个对话 Session。

因此，内置 Agent 调用编辑工具和外部 Agent 通过 MCP 调用编辑器，不需要各自实现一套结果等待、客户端路由与恢复逻辑。它们使用相同的运输和执行语义，但保留不同的地址与生命周期。

![DeepSeek Harness 在 Session 内统一消息与工具；Video Agent Loop 让 Session 与 Project Channel 共享同一套 Inbox 和执行运输](../assets/agent-loop/dsh-video-runtime-boundary.svg)

*我们统一的不是所有业务语义，而是跨边界调用的运输、关联、恢复和结果协议。*

### 对照 DeepSeek Harness

DeepSeek Harness 已经完成了两种相邻的统一：人的补充、Job 通知和定时提醒可以通过 `followup`、`steer` 或 `inject` 进入 Agent Inbox；MCP Tool 则会被注册进与内部 Tool 相同的 Tool Runtime。[Background Job Controller](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/jobs/tool-jobs/README.md)、[MCP Client](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/mcp/mcp-client/README.md)

但 DSH 的 Inbox 主要承载模型可消费的消息，当前 Step 的 Tool Result 由 Tool Runtime 直接写入 Session。它的 MCP 方向也是 Client——让 DSH 内部 Agent 使用外部 MCP Tool，而不是让外部 Agent 通过 MCP 进入一个浏览器编辑器。

如果换成 DSH，我们可以直接获得它成熟的消息 Inbox 和 Tool Runtime；但仍要实现 Project Channel、WaitPort、外部 MCP Server、客户端执行路由以及版本化 Task Update。把这些状态简单包装成普通用户消息会失去结果关联、幂等领取和超时竞争的语义；更合理的做法仍然是扩展 DSH 的事件与插件边界。

因此，在统一 Inbox 这一点上，我们的实现不是 DSH 的简化版，覆盖范围反而更贴近跨进程、跨客户端的视频工作流。

<!-- [研发待补 A-D1] 用同一组任务对比两种映射：保留 StarCut typed Inbox/WaitPort，或将其实现为 DSH Session Event 插件；记录适配代码、恢复窗口和状态重复数量。 -->

## 四、我们的外循环：Run Lease、运行提示与恢复扫描

最直观的 Agent 实现，是从收到用户消息开始，让一个进程一直运行，直到整件事结束。短任务中它很简单；一旦进入真实创作流程，问题会迅速出现。

假设 Agent 正在制作一段产品视频：它先提交三项素材生成任务，等待结果后编排时间线，再请用户确认。这个过程可能持续几十分钟，期间会发生：

- 生成服务多次上报进度，其中一项失败重试；
- 用户补充“不要使用第三张图”；
- 编辑器所在浏览器休眠或重连；
- Agent Worker 因部署而退出；
- 用户直接在画布上调整了已经生成的结果。

如果工作只存在一个内存 Promise 中，进程退出就意味着上下文丢失；如果用一个无限轮询维持生命，系统会长期占用资源，并且难以处理多个 Worker 的竞争。更麻烦的是，人在等待期间做出的修改可能与旧进程里的项目快照发生冲突。

可靠性的关键不是“让那个进程永远不死”，而是反过来：**允许执行过程随时结束，但让继续工作所需的事实不会消失。**

我们的 Gateway 收到新事实后先写入 Inbox，再发出运行提示。Worker 获得提示后，必须先取得这个 Session 的 Run Lease，才能创建一次新的 Agent Process。Lease 有明确的所有者和期限，运行期间持续续约；失去 Lease 的旧执行不能继续提交结果。

这些事实可能来自不同方向：

- 用户发送了新要求；
- 某个工具返回结果；
- 一组生成任务完成或失败；
- 预定的检查时间到达；
- 客户端重新连接并报告执行结果；
- 项目状态发生了需要 Agent 响应的变化。

这些事实先进入持久 Inbox，再通知运行系统。通知的作用只是降低延迟，不是保存唯一事实。即使通知重复或丢失，系统仍可从 Inbox 判断哪些事件尚未被处理。

```text
外部事实发生
  → 先持久化
  → 再发出运行提示
  → 获得执行权的 Worker 读取新事实
  → 启动或继续一次内循环
```

“先保存，后唤醒”看似只是顺序问题，实际决定了系统能否恢复。若先发通知、后写状态，Worker 可能在事实落盘之前醒来，发现无事可做并退出；随后写入的事实便失去触发机会。

Run 结束时，系统不会先判断 Inbox 为空、再单独释放 Lease，因为新输入可能刚好落在这两个动作之间。我们把“确认 Inbox 为空并释放 Lease”作为一个原子边界：如果已经有迟到输入，本次 Run 只返回 pending，由外循环继续调度，而不是在内部悄悄再创建一个永久循环。

通知仍可能丢失，Worker 也可能在取得 Lease 后崩溃。因此每个 Host 都可以运行恢复扫描：处理到期的 Clock、WaitPort 超时、Task 状态对账、Monitor 压缩，并重新调度 Inbox 非空但已无有效 Lease 的 Session。扫描可以重复，真正的互斥仍由 Lease 获取保证。

![一段 Agent 工作跨越多次短执行、异步等待、用户介入和进程重启](../assets/agent-loop/inner-outer-timeline.svg)

*Session 持续存在，但执行它的 Process 可以结束、迁移和重建。*

### 对照 DeepSeek Harness

DeepSeek Harness 同样保证一个已准备的 Session 只能由一个具体 Driver 推进，也处理运行中消息、取消与恢复。它的 Job 完成通知能够唤醒空闲 Agent，Schedule 也可以把提醒送入后续 Turn。[Jobs 设计](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/jobs.md)、[Schedule 设计](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/schedule.md)

但 DSH 当前公开的 Job Contract 是进程内边界；Schedule 要求原 Session 处于运行状态，也明确不提供冷 Session 调度器。对于单 Host Agent，这套生命周期更轻；对于可能跨越多个 Worker、部署重启和长时间媒体生成的系统，仍然需要分布式 Lease、持久 Task 对账和冷 Session 恢复。

如果把我们的内循环换成 DSH，外循环不会因此消失。较现实的方案是保留现有 Inbox、Run Lease 和 Sweep，让它们在取得执行权后驱动一个 DSH Agent；或者为 DSH 实现相应的分布式 Session Provider。前者适配更直接，后者架构更统一，但都不是零成本替换。

## 五、双环如何衔接：等待不是挂起一个进程

外循环要恢复工作，必须知道某个异步结果属于谁。我们的 WaitPort 就是这份持久的**等待关系**，而不是另一条结果队列：

```text
当前工作正在等待什么？
结果应关联到哪次调用？
完成、失败或超时后应该怎样继续？
```

当内循环提交一个长任务，它先打开 WaitPort，然后可以结束。任务结果到来后仍写入统一 Inbox；WaitPort 根据调用当前所处阶段，把结果交给仍在等待的 Tool，或者留给下一次内循环。

这种设计把两个问题分开了：

- 事件入口保存“发生了什么”；
- 等待关系说明“这件事与谁有关、如何继续”。

如果等待机制也保存一份结果，任务系统又保存一份进度，对话历史再复制一份工具状态，恢复就会变成多份记录之间的猜测。双环真正追求的不是多一层调度，而是**让每个事实只有一个权威来源**。

## 六、Monitor：观察任务，但不要成为第二个任务系统

长任务给 Agent 带来一个看似简单的问题：什么时候应该再看一眼？

一个视频生成任务可能上报数百次进度。如果每次从 41% 变成 42% 都唤醒模型，Agent 会重复读取几乎相同的上下文，消耗 token，却做不出新决定。反过来，如果完全不观察，失败和阶段性结果又无法及时触发后续动作。

Monitor 的职责不是接管任务，而是完成两种压缩：

1. **状态压缩**：把多项任务的大量增量通知，归并成每项任务的最新权威状态；
2. **注意力压缩**：只在出现需要判断的新信息时唤醒 Agent。

任务本身仍然只有一份权威状态。底层通知可以只是“某任务发生了变化”，Monitor 重新读取最新状态，并拒绝已经过期的版本。它只观察当前 Agent 明确等待的任务集合，不维护一份自己的全局任务注册表。

![Monitor 将大量任务进度压缩为少数几次有决策价值的 Agent 唤醒](../assets/agent-loop/monitor-compression.svg)

*界面可以连续显示进度，模型只需要在失败、阶段完成、全部完成或到达检查点时被唤醒。*

这也解释了为什么 **UI 更新** 和 **Agent 唤醒** 不应该是同一件事。用户希望随时看到 63% 变成 64%；模型却通常不需要为这 1% 重新推理。二者消费同一任务事实，但采用不同的注意力策略。

较合理的唤醒条件包括：

- 某项任务失败，需要重试或换方案；
- 一个完成结果已经足以解锁后续步骤；
- 整组任务全部结束；
- 到达预先声明的阶段检查点或截止时间；
- 用户主动要求检查当前进展。

因此，Monitor 更像 Agent 的观察面和信息编译器，而不是另一个 Agent，也不是第二套任务平台。

### 对照 DeepSeek Harness

DeepSeek Harness 的 Job Controller 已经避免了忙轮询：Job 完成后，忙碌 Agent 在下一 Step 收到通知，空闲 Agent 可以被唤醒；连续自动唤醒还有明确上限。这解决了“后台工作完成后怎样让模型知道”的通用问题。

我们的 Monitor 进一步处理版本化媒体 Task 和任务组。它不只发送“某个 Job 已结束”，还要把一组任务的大量中间状态压缩成少数几次有决策价值的上下文，并在事件通知丢失后从权威 Task 状态重新对账。换成 DSH 后，这一层仍需作为视频 Task Provider 和 Monitor 插件存在，不会由通用 Job 通知自动替代。

<!-- [研发待补 A-M1] 固定 Monitor 的 cohort、版本合并、关闭条件与唤醒策略；记录原始 task update 数、聚合 context 数、模型 wake-up 数及 token，用于计算压缩率和成本。 -->

## 七、MCP Canvas：给 Agent 语义动作，而不是远程鼠标

外循环解决 Agent 何时继续，Monitor 解决它应该注意什么；接下来还需要明确它如何行动。

对于外部 Coding Agent，[Model Context Protocol](https://modelcontextprotocol.io/specification/2026-07-28) 可以统一资源发现和工具调用。但 MCP 只规定连接与能力交换，不会自动理解“画布”“片段”或“时间线”。创作软件仍然必须设计自己的 Agent-Computer Interface。

最脆弱的做法，是让 Agent 通过坐标点击和拖动画布：

```text
点击 (817, 424)
拖到 (1200, 424)
```

只要窗口尺寸、缩放比例、摄像机位置或布局发生变化，这些坐标就失去意义。像素操作也很难表达用户真正的意图——“将这段素材放到标题下方”“把这个片段移动到第二条轨道”“将节点与旁白对齐”。

MCP Canvas 应暴露稳定的语义对象与领域动作：

```text
读取一个项目或节点
创建并放置内容
修改节点属性
移动或重排节点
删除节点
请求预览或验证
```

坐标可以作为视觉交互的输入，但不能成为持久身份。操作对象应拥有稳定标识，每次调用也应显式携带项目句柄，并在服务端重新验证授权。MCP 官方的[有状态工具设计指南](https://modelcontextprotocol.io/specification/2026-07-28/server/tools#stateful-tools)同样强调：跨调用状态应通过显式 handle 表达，而不是隐藏在传输连接中。

## 八、MCP 不是另一套编辑器

语义工具仍然可能落入另一个陷阱：为了让 MCP 在服务端工作，再实现一份“简化画布”。这样外部 Agent 修改的是服务器影子状态，人类看到的却是浏览器编辑器状态，二者早晚会产生不同的校验规则、撤销语义和渲染结果。

更合理的结构是让不同入口汇入同一条领域写路径：

```text
外部 Agent / MCP ─┐
内置 Agent        ├→ 同一组领域操作 → 权威项目状态 → 人类编辑界面
人类 UI           ┘
```

如果某项操作必须由持有实时项目状态的浏览器完成，服务端就把已经授权的语义调用交给当前获得执行权的 Editor，由它通过正常编辑链提交，再返回结果。多个标签页可以观察同一项目，但一次调用只能有一个有效执行者。

![MCP Canvas Bridge 将外部语义调用交给连接中的编辑器，并复用人类界面的领域写路径](../assets/agent-loop/mcp-canvas-bridge.svg)

*MCP 标准化连接；项目模型定义语义；连接中的 Editor 执行；所有入口最终修改同一份权威状态。*

短小的画布修改可以直接返回结果。视频生成、转写或大规模导出等长操作，则适合返回持久任务句柄，由调用方查询或订阅其状态。MCP 的 [Tasks 扩展](https://modelcontextprotocol.io/extensions/tasks/overview) 为这类工作提供了协议框架，但内部任务状态仍不应因此复制一份；协议中的 Task 应是权威任务的外部投影。

### 对照 DeepSeek Harness

DeepSeek Harness 的 MCP Client 把远端 MCP Tool 映射成内部 Tool，这对“让内置 Agent 使用外部能力”很方便。我们的 MCP Canvas 解决的是另一个方向：让外部 Agent 进入 StarCut，并与内置 Agent 共用 Client Lease、WaitPort、Tool Result 和领域写路径。

如果内循环换成 DSH，它可以继续作为 MCP Client 使用其他服务，但 StarCut 对外的 MCP Server、项目句柄、在线 Editor 选择和多窗口执行仲裁仍由我们的项目运行时承担。这不是重复实现 MCP，而是协议两端承担的职责不同。

<!-- [研发待补 A-C1] 明确哪些 Canvas 工具必须由连接 Editor 执行，哪些可以服务端直接完成；固定无在线 Editor、执行中断线、结果已提交但响应丢失时的语义。 -->
<!-- [研发待补 A-C2] 固定内部任务与 MCP Tasks 的状态、版本、取消、超时和输出映射，避免形成两套任务生命周期。 -->

## 九、为什么同时保留 SSE 与 WebSocket

Agent 的输出看起来像一条聊天流，但视频编辑器并不只有“服务端往页面发文字”这一种通信。

我们让 SSE 和 WebSocket 承载同一组有序的 Session Stream 事件。SSE 适合持续读取消息提交、运行状态和模型增量：连接简单，断开后可以从持久游标继续。WebSocket 除了读取同一条流，还负责双向的客户端执行：服务端授予 Client Lease、派发 Editor Tool，客户端确认执行权并返回 Tool Result。

```text
SSE
  服务端 → 浏览器
  Session 消息、运行状态、模型增量

WebSocket
  服务端 ↔ 浏览器
  同一事件流 + Lease + Tool Assignment + Tool Result
```

双 Transport 不代表两套业务协议。完整消息仍来自同一提交记录，实时 Delta 只是低延迟投影；客户端返回的结果仍进入统一 Inbox。无论页面选择 SSE 还是 WebSocket 观察 Session，都不能产生不同的对话真相。

### 对照 DeepSeek Harness

DeepSeek Harness 提供 Host/Client 远程调用与连接抽象，可以减少普通控制 API 的手工协议代码；但其公开 API Gateway 文档也把流式数据和实体子流留给独立协议处理。[API Gateway](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/api-gateway.md)

因此，换用 DSH 不会替我们决定 SSE 还是 WebSocket，也不会消除浏览器 Tool 执行所需的双向通道。它可能简化 Agent Host 的控制接口，而我们的双 Transport、持久游标与 Client Lease 仍有独立价值。

<!-- [研发待补 A-T1] 在同一事件源上测量 SSE 与 WebSocket 的重连恢复时间、游标重复率、Delta 丢失后的收敛时间，以及 Client Tool 在断线窗口中的结果归属。 -->

## 十、一次完整的创作任务如何运行

把前面的机制放在一起，可以看到一个跨越数十分钟的任务并不需要任何永久运行的 Agent 进程。

用户要求：“根据脚本生成三张产品图，挑选合适的两张放进画布，再排成一个十秒开场。”

第一段内循环读取脚本和项目，提交三项生成任务，记录等待关系，然后结束。生成服务持续更新任务状态，界面实时展示进度；Monitor 将高频变化合并，仅在一张图失败和三项任务全部结束时形成两次有决策价值的通知。

外循环看到最终结果后启动第二段内循环。Agent 比较候选图，选择两张，并通过语义工具请求把它们加入项目。MCP Canvas Bridge 把调用交给有执行权的 Editor，Editor 通过与人类操作相同的写路径更新权威项目。

此时用户在画布上手动换掉其中一张图，并补充“标题用刚才的蓝色版本”。这两个新事实进入外循环。下一段内循环读取的是用户已经修改后的项目，而不是二十分钟前留在旧进程里的快照，于是它保留人的选择，只继续完成标题和十秒编排。

整个过程中，Agent 的“连续性”来自可恢复事实之间的关联，不来自某个永远不退出的线程。

## 十一、人类接管不是异常，而是默认路径

创作任务很少能在开始时完全定义。人会观察中间结果、改变想法、直接修改项目，再让 AI 继续。一个只会独占执行到最终答案的 Agent，天然不适合这种协作。

双环结构使人类介入成为普通事件：

- 人的新要求进入外循环；
- 人对项目的修改直接改变权威状态；
- Agent 再次运行时读取最新项目，而不是覆盖整份工程；
- Agent 的动作仍可被界面展示、撤销和继续调整。

这和 Coding Agent 的协作方式很接近：人可以在两次 Agent 执行之间改代码，Agent 随后重新读取仓库并继续。但创作软件还多了一层连续视觉结果，因此 Agent 不仅要读取结构，还需要在关键阶段重新观察预览，确认人的修改是否改变了原先判断。

## 十二、系统可靠性与创作判断是两类问题

双环、Monitor 和 MCP Canvas 主要解决的是 Agent 的**系统能力**：

- 跨越等待、断线和重启继续工作；
- 把后来发生的事实放回正确上下文；
- 避免同一 Session 被多个执行过程并行推进；
- 控制长任务更新带来的模型唤醒和上下文噪声；
- 让外部 Agent 通过稳定语义操作真实项目；
- 允许人类随时修改并接管同一份工程。

它们不会自动让模型拥有审美，也不会保证每次规划都正确。一个运行可靠的 Agent 仍可能做出平庸的创作选择；一个语义清楚的 Canvas 工具也不能替代视觉理解。可靠运行时解决的是“它能否持续、准确地行动”，感知和评价解决的是“它是否知道该做什么”。两类问题需要配合，但不应混写成一个贡献。

## 十三、如何判断这个设计是否真的有效

架构图只能说明意图，不能证明可靠性。双环 Agent 最重要的证据应来自故障、并发和真实任务，而不只是一个顺利完成的演示。

我们需要回答至少四组问题：

### 可恢复性

- 在事件已保存但通知丢失、执行刚获得所有权就崩溃、客户端完成操作却没能回传结果等窗口中，任务最终能否继续？
- 恢复需要多久？是否遗漏消息或产生无法解释的中间状态？

### 单一执行与副作用

- 多个 Worker 和多个浏览器同时在线时，同一 Session 是否仍只有一个有效执行过程？
- 同一个画布操作、生成请求或外部副作用会不会重复执行？

### Monitor 的信息效率

- 原始任务更新被压缩成了多少次 Agent 唤醒？
- 压缩后是否错过需要决策的失败或阶段结果？
- token、延迟和完成率之间如何权衡？

### MCP Canvas 的操作质量

- 语义工具与像素 GUI 操作相比，任务完成率、操作步数和重试次数怎样变化？
- 浏览器断线、人类中途修改、多标签页竞争时，结果是否仍落到同一份权威项目？

<!-- [研发待补 A-E1] 建立按持久化边界注入故障的测试集，保存事件、执行权、任务版本、工具调用和项目事务的完整关联链。 -->
<!-- [研发待补 A-E2] 冻结双环与常驻 Process、多事件直启 Process、任务主动轮询等基线；至少记录完成率、恢复延迟、重复副作用、模型调用数和资源占用。 -->
<!-- [研发待补 A-E3] 建立 MCP 语义操作、像素 GUI 操作与服务端影子状态三组基线，使用同一批真实创作任务和同一模型评估。 -->

## 十四、如果换成 DeepSeek Harness，会更简单吗？

答案不是简单的“会”或“不会”，而是取决于替换哪一层。

| 层次 | 换成 DSH 可能省掉什么 | 仍然需要保留或重做什么 | 当前判断 |
|---|---|---|---|
| 模型—工具内循环 | Loop、工具管线、取消、插件事件 | AI SDK UI Message、项目上下文与客户端工具适配 | 可能简化，但需要实测 |
| Session 与压缩 | 追加式事件日志、可恢复压缩 | 现有历史迁移、产品消息投影、项目记忆 | DSH 值得重点对照 |
| 消息调度 | next-step / next-turn Inbox | Typed Inbox、WaitPort、Task Update | 不能直接替换 |
| 长时媒体任务 | 通用 Job 接口与完成通知 | 跨进程 Task、版本对账、Monitor、冷恢复 | 我们的实现更完整 |
| 外部 MCP 与 Editor | MCP Client Tool 接入 | MCP Server、Project Channel、Client Lease、浏览器执行 | 仍以我们的运行时为主 |
| 浏览器通信 | Host/Client 控制抽象 | SSE/WS 事件流、工具派发和断线恢复 | 只能局部复用 |

所以，完整替换目前不会让 Video Agent Loop 自动变简单。视频场景最难的部分——持久媒体任务、统一 Inbox/WaitPort、分布式 Run Lease、Monitor、Project Channel 和在线 Editor 仲裁——仍然存在，再加上一层 DSH 适配，整体概念甚至可能更多。

更值得验证的是一个清楚的替换边界：保留我们的外循环、Inbox、Task 和 Editor Runtime，只把一次 Session Run 的模型—工具内核替换为 DSH。这样既能检验它的插件系统、Session Event 和 Compaction 是否带来真实收益，也不会牺牲已经跑通的视频业务语义。

这不是因为我们的实现只能守成。相反，现有架构已经把通用 Session 执行和产品级工具、Task、Editor 调度分开，才使替换内核成为可能。DSH 的价值，是提供一套质量很高的独立实现来检验这个边界，而不是证明我们原来的方向错误。

还有一个现实因素：DeepSeek Harness 官方目前仍将项目标记为 Developer Preview，并提示接口可能快速变化。它适合进入替换实验和架构验证，但在证明收益之前，不适合仅凭热度成为视频生产链路的新核心依赖。[DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)

<!-- [研发待补 A-D2] 做一个可切换的 Session Run 实验：相同模型、相同工具和相同 Inbox 输入，分别驱动现有内核与 DSH；比较适配复杂度、恢复正确性、工具调用成本、上下文 token 和端到端完成率。 -->

## 十五、与已有工作的关系

这套设计并不是从零发明“循环”“消息”或“持久执行”。它把几类已经成熟但经常在 Agent 产品中混在一起的问题，重新放到创作软件的运行边界中。

[ReAct](https://arxiv.org/abs/2210.03629) 主要讨论推理与行动如何交错，对应本文的内循环。[Orleans 的 Virtual Actor](https://www.microsoft.com/en-us/research/publication/orleans-distributed-virtual-actors-for-programmability-and-scalability/)、[Azure Durable Functions](https://learn.microsoft.com/en-us/azure/durable-task/durable-functions/durable-functions-overview) 和 [Temporal](https://docs.temporal.io/) 等工作已经说明，长生命周期实体不必依赖永久进程，状态、调度与恢复可以由持久运行时承担。本文不把这些分布式系统基础声明为新发明。

本文更关心它们进入生成式创作 Agent 后的具体边界：模型工具迭代何时结束，外部事实如何重新进入上下文，任务进度怎样压缩为模型注意力，以及外部 MCP 调用如何进入一个由浏览器持有实时状态、又允许人类同时编辑的项目。

[SWE-agent](https://arxiv.org/abs/2405.15793) 强调 Agent-Computer Interface 会显著影响软件工程 Agent 的行为；[OSWorld](https://arxiv.org/abs/2404.07972) 揭示了通用 GUI Agent 在真实计算机环境中的视觉定位和操作困难；面向真实后期制作任务的 [AgenticVBench](https://arxiv.org/abs/2605.27705) 也表明执行框架会影响工具使用和失败方式。这些工作共同支持一个判断：Agent 能力不能只看模型，还要看系统向它提供了怎样的观察与行动界面。

与 [Reflexion](https://arxiv.org/abs/2303.11366) 一类跨尝试反思方法相比，本文的外循环不是新的推理策略。它不规定模型如何自我批评，而是确保无论采用哪种推理方法，新的外部事实都能被可靠保存、路由和恢复。

DeepSeek Harness 则提供了最直接的工程对照：它把 Agent Loop、Session、Tool Runtime、Compaction、Job 和 MCP Client 组织成可组合插件；我们的实现把相似能力放入分布式 Session、持久媒体 Task 和浏览器 Editor 的生命周期。相同设计说明双方面对的是通用 Agent Runtime 问题，不同边界则来自产品运行环境，而不是简单的先进与落后。

## 结语：Agent 的连续性来自状态，而不是线程

一个真正能进入创作流程的 Agent，不能只在模型响应的几十秒里存在。它需要等待生成、接收人的修改、跨越客户端断线和服务重启，再回到同一个项目继续工作。

实现这种连续性，不需要把所有能力塞进一个永久循环。更清楚的办法是：

```text
内循环负责完成当下
外循环负责跨越时间
Monitor 负责压缩观察
MCP Canvas 负责表达行动
权威项目与任务状态负责保存事实
```

当执行过程可以结束、事实不会丢失；当高频更新可以被看见，却不会无意义地占用模型注意力；当外部 Agent、内置 Agent 和人类编辑器通过不同入口操作同一项目，Agent 才从“会调用工具的模型”变成能够长期参与真实创作的软件角色。

DeepSeek Harness 的可取之处，是让通用 Agent Runtime 的插件边界、事件日志和上下文治理更加完整；它当前不足以直接替代的，是我们已经实现的跨进程外循环、统一 Inbox/WaitPort、媒体 Task Monitor 和项目级客户端执行。对照的结果不是推翻其中一方，而是让下一步实验有了更明确的替换边界。

## 相关资料

1. S. Yao et al., [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629), 2022/2023.
2. P. Bernstein et al., [Orleans: Distributed Virtual Actors for Programmability and Scalability](https://www.microsoft.com/en-us/research/publication/orleans-distributed-virtual-actors-for-programmability-and-scalability/), Microsoft Research Technical Report, 2014.
3. Microsoft, [Durable Functions Overview](https://learn.microsoft.com/en-us/azure/durable-task/durable-functions/durable-functions-overview).
4. Temporal Technologies, [Temporal Documentation](https://docs.temporal.io/).
5. Model Context Protocol, [Specification 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28).
6. Model Context Protocol, [Tasks Overview](https://modelcontextprotocol.io/extensions/tasks/overview).
7. J. Yang et al., [SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering](https://arxiv.org/abs/2405.15793), 2024.
8. T. Xie et al., [OSWorld: Benchmarking Multimodal Agents for Open-Ended Tasks in Real Computer Environments](https://arxiv.org/abs/2404.07972), 2024.
9. Z. Cao et al., [AgenticVBench: Can AI Agents Complete Real-World Post-Production Tasks?](https://arxiv.org/abs/2605.27705), 2026.
10. N. Shinn et al., [Reflexion: Language Agents with Verbal Reinforcement Learning](https://arxiv.org/abs/2303.11366), 2023.
11. DeepSeek AI, [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness), developer preview, 2026.
12. DeepSeek AI, [DeepSeek Harness Architecture](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.md), 2026.
