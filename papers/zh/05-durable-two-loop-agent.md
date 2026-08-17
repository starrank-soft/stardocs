# 双环 Agent：基于持久 Inbox、WaitPort 与 Monitor 的可恢复创作代理架构

> 中文系统设计母稿，可继续发展为论文或系列文章
>
> 论文英文候选标题：*Durable Two-Loop Agents: Message-Driven Recovery, Monitored Tasks, and Semantic Canvas Execution*
>
> 系列文章总标题：**《Agent 为什么只需要两个业务循环》**
>
> 三篇分章标题：
>
> 1. 《内循环负责思考，外循环负责活着》
> 2. 《Monitor 不应该成为第二个任务系统》
> 3. 《MCP 如何操纵画布：发送语义操作，而不是远程点击像素》
>
> 状态：架构稿。本文描述业务循环；Redis listener、SSE、重连和 sweep 是交付/恢复机制，不应被误算成额外业务循环。

## 摘要

创作型 Agent 既需要在一次推理中连续调用工具，也需要跨越用户消息、客户端执行、异步生成任务、定时唤醒、网络重连和进程崩溃继续工作。将所有行为放进一个常驻循环会让会话状态、工具结果、任务进度和恢复逻辑相互纠缠；为每类异步事件再建立独立队列和循环，又会制造多份路由状态与竞争唤醒。

本文提出一种双环 Agent 架构。内循环是一次 Agent Process 内的模型—工具迭代，负责完成当前推理；外循环是由持久 Session Inbox 驱动的消息循环，负责把用户消息、工具结果、时钟事件和任务通知转化为可恢复的 Agent Process。每个 Session 通过租约保证最多一个活动 Process；所有生产者先写 Inbox 再发送运行信号，信号仅用于加速，周期 sweep 可从持久状态恢复丢失唤醒。

为支持长任务，系统使用 WaitPort 描述“某个异步结果应如何路由”，而不再建立第二个结果队列。Monitor 作为任务更新之上的 sidecar，从权威 Task 状态聚合进度，只对正在监控的 WaitPort 形成 cohort，并把多次底层更新压缩为一个稳定的隐藏上下文。对于外部 Coding Agent，系统通过无状态 MCP 接收带显式 `projectId` 的语义画布操作，将其写入普通 Project Message Stream，由获得执行租约的浏览器 Editor 通过同一编辑链修改画布，并以 `toolCallId` 关联结果。

本文给出循环边界、持久化不变量、故障恢复、Monitor 聚合和 MCP Canvas Bridge 的设计，并计划通过崩溃注入、多副本竞争、重复/乱序通知、长任务编排和画布操作任务评估可靠性、延迟、模型唤醒次数、进度压缩率和重复执行率。

## 1. 为什么要严格定义“循环”

Agent 系统中常见的“循环”至少有四种含义：

- 模型连续思考和调用工具；
- 收到一条新消息后重新运行 Agent；
- 网络 listener 持续读取 Redis、SSE 或 WebSocket；
- sweep 定期扫描遗漏工作；
- Monitor 持续观察任务状态。

如果把所有持续运行的代码都称为 Agent Loop，架构很快会失去边界。本文只把会改变 Agent 业务状态和推理阶段的循环称为业务循环：

1. **内循环**：当前 Process 内，模型 step 与工具结果之间的迭代；
2. **外循环**：持久事件触发新的或继续中的 Session Process。

listener、消息推送、租约续期、重连和 sweep 都是交付或恢复机制。Monitor 是对任务事件的聚合 sidecar；它不自行成为第三套 Agent 推理循环。

## 2. 系统模型

### 2.1 主要实体

| 实体 | 生命周期 | 责任 |
|---|---|---|
| Session | 持久 | 对话、记忆与 Agent 工作上下文 |
| Inbox | 持久 | Session 的唯一异步输入入口 |
| Run signal | 易失/可重复 | 提醒 Worker 某 Session 有工作 |
| Lease | 有期限 | 保证同一 Session 至多一个活动 Process |
| Agent Process | 短暂 | 加载上下文并运行一次内循环 |
| UIMessage | 持久 | 用户、助手、工具调用与结果的权威消息表示 |
| WaitPort | 持久路由状态 | 描述异步调用等待什么、如何结束 |
| Task | 持久权威状态 | 生成、转写等长任务生命周期 |
| Monitor | 逻辑 sidecar | 聚合被监控 Task 的更新并唤醒外循环 |
| Project Message Stream | 持久 | 无对话 Session 的项目级 MCP/Editor 工具调用 |

### 2.2 单一 Session Inbox

Inbox 接收四类条目：

```text
message       用户或系统消息
context       时钟/Monitor 等隐藏上下文
tool_result   客户端或异步工具结果
task_update   Task 权威快照的版本化投影
```

所有异步生产者都写同一个 Inbox：用户 API、客户端工具执行器、Clock、Task listener 和 Monitor。系统不再为“工具结果”“任务进度”“定时器”各建一条业务队列。

单 Inbox 的价值不是减少 Redis key 数量，而是让以下事实只有一个权威定义：Session 还有哪些未被 Agent 消费的外部事件。

### 2.3 WaitPort 是路由状态，不是队列

Agent 调用一个不能立即完成的工具时，系统创建 WaitPort：

```text
toolCallId -> expected producer / taskId / policy / deadline / status
```

WaitPort 回答：

- 哪个结果属于哪个工具调用；
- 结果到达后是唤醒 Agent、仅记录、还是严格等待；
- 是否有 deadline；
- 是否属于正在 Monitor 的 Task cohort；
- 何时可以关闭。

真正的结果仍进入 Inbox。若 WaitPort 自己再保存一份消息队列，系统就会出现 Inbox 与 WaitPort 双重确认、双重恢复和不一致状态。

## 3. 内循环：一次 Process 内的模型—工具迭代

内循环可以抽象为：

```text
prepare context
  -> model step
  -> zero or more tool calls
  -> tool results
  -> next model step
  -> committed assistant message or final response
```

它负责：

- 根据当前上下文选择工具；
- 执行服务端工具或等待客户端工具结果；
- 在 step 边界吸收 Process 运行期间到达的 Inbox 条目；
- 持久化完整 UIMessage 快照；
- 在当前目标完成、等待外部事件或达到停止条件时退出。

[ReAct](https://arxiv.org/abs/2210.03629) 将推理与行动交错，是内循环的典型思想来源。本文不主张发明模型—工具循环，而研究它如何嵌入持久、可恢复的外循环。

### 3.1 内循环不拥有永久生命

Agent Process 是可丢弃计算，不是 Session 本身。进程可以因扩缩容、部署、超时或崩溃结束；可恢复信息必须在消息、Inbox、WaitPort、Task 和数据库中，而不是只存在 JS Promise 或内存对象里。

### 3.2 客户端工具等待

某些操作只能由持有当前 Editor 状态的浏览器执行。内循环提交工具调用后：

1. 服务端分配唯一执行 token 给获得项目租约的客户端；
2. 客户端通过统一 ClientToolRunner 执行；
3. 结果以 `toolCallId` 写入 Inbox；
4. Process 若仍活动，可在下一个 step 吸收；若已退出，外循环会重新启动；
5. 已提交消息只用于权威展示和恢复，不能被任意观察标签页再次当作执行授权。

执行 token 与租约防止多个打开同一项目的标签页重复修改。

## 4. 外循环：持久事件驱动 Session Process

### 4.1 写入先于唤醒

每个生产者遵循：

```text
persist Inbox entry
  -> emit run signal
```

不能先发信号再写 Inbox。否则 Worker 可能先收到信号、发现没有内容并退出，随后写入的消息失去唤醒。

### 4.2 一 Session 一活动租约

Worker 收到信号后使用 NX/条件写竞争 Session lease。只有赢家可以创建 Process。其他 Worker 即使收到重复信号也不启动第二个 Process。

核心安全不变量：

\[
ActiveProcess(session) \le 1
\]

租约必须有 owner、期限和安全释放条件。仅依靠内存中的“正在运行”集合无法跨多副本或进程崩溃成立。

### 4.3 活动 Process 排空新事件

若新的 Inbox 条目在租约已经存在时到达，生产者仍然写入并发送信号。第二个 Worker不会创建 Process；当前 Process 在合适的 step 边界重新 claim Inbox，把新消息加入上下文。这样既没有并行 Session 推理，也不需要等待旧 Process 完全结束后才看到结果。

### 4.4 安全释放

Process 结束时，不能简单执行：

```text
release lease
```

因为“检查 Inbox 为空”和“释放租约”之间可能有新消息到达。需要以原子方式完成：

- 若 Inbox 为空，释放属于自己的租约；
- 若 Inbox 非空，继续运行或确保重新调度；
- 若租约 owner 已变化，不得释放别人的租约。

### 4.5 signal 是加速器，sweep 是恢复器

运行信号可以重复、延迟或丢失。系统的正确性不依赖每个信号必达，因为 sweep 会扫描“有未消费 Inbox 且无有效 lease”的 Session 并重新调度。

```text
durable Inbox = truth
run signal    = low-latency hint
sweep         = recovery
```

若 sweep 自己维护另一份“待运行 Session 表”，就又创造了第二个真相源。它应从 Inbox、lease 和持久 Session 状态推导工作。

## 5. Monitor：不要建立第二个任务系统

### 5.1 Task 状态与通知提示分离

Task Worker 完成一步后先提交权威数据库状态，再发送轻量提示：

```text
commit Task(taskId, version, status, progress, output)
  -> emit TaskChanged(taskId, version)
```

listener 收到提示后重新读取权威 Task。提示可以重复、乱序或丢失；版本比较与周期 sweep 使最终状态仍可恢复。

这比在事件消息里携带并信任完整 Task 状态更稳健，因为数据库提交和消息投递无需伪装成一个分布式事务。

### 5.2 只监控被显式等待的 cohort

Monitor 不扫描全系统任务，也不建立自己的任务 registry。它从当前 Session 的 WaitPort 中选择具有 monitoring policy 的 Task，形成 cohort：

```text
monitoring WaitPorts
  -> task IDs
  -> authoritative Task snapshots
  -> one aggregate context
```

没有对应 WaitPort 的任务不属于该 Session 的 Monitor。这样任务所有权和等待语义仍只有一个来源。

### 5.3 聚合而不是逐事件唤醒模型

长任务可能产生大量进度更新。如果每个 1% 更新都运行一次模型，会产生无意义成本和上下文噪声。Monitor 把同一 cohort 的最新版本聚合为一个隐藏上下文，例如：

```text
Monitored tasks: 2/5 terminal.
- image-a: running, 70%
- video-b: completed, artifact=...
- video-c: failed, reason=...
```

聚合必须保留每个 Task 的最新权威版本，不能因为只消费了本批 4 条事件，就把总进度错误地从 `2/5` 退化为 `1/4`。

建议触发 Agent 的条件：

- cohort 首次出现需要决策的失败；
- 有任务进入 terminal 且可能解锁后续工作；
- 全部任务 terminal；
- 达到显式检查点或 deadline；
- 用户消息要求立即查看。

纯进度刷新可以更新 UI，但不一定需要唤醒模型。

### 5.4 幂等与关闭

对每个 Task，Monitor 使用版本 CAS 或单调比较拒绝旧快照。全部 terminal 后：

1. 写入最终聚合上下文；
2. 关闭对应 WaitPort；
3. 触发外循环；
4. 重复 TaskChanged 不再重新打开 cohort。

> **[待补 A-C1：触发策略]** 当前需要固定“哪些 Task 更新写 Inbox、哪些只更新 UI、哪些真正触发模型”，否则成本实验无法复现。

## 6. Clock：把未来时间也写回同一个入口

Agent 可能要求“十分钟后检查”或“明早继续”。Clock 只负责在到期时产生一个 Inbox context，并触发外循环。逻辑上一个 Session 只有一个 Clock 能力，不为每个 alarm 建立独立 Agent。

Clock 与 Monitor 的共同点是：它们不是另一个会思考的 Agent，而是把外部世界的变化转化为持久消息。

## 7. MCP 如何操纵画布

### 7.1 不要把 MCP 变成远程鼠标

无限画布的屏幕坐标受 camera、zoom、窗口、布局和选中状态影响。外部 Coding Agent 若通过 MCP 发送“点击 (817, 424)，拖到 (1200, 424)”，得到的是脆弱的 UI 自动化。

MCP 应暴露画布语义：

- 创建或放置一个项目节点；
- 修改节点属性；
- 移动/重排节点；
- 删除节点；
- 读取项目文件或节点；
- 调用只能由连接 Editor 完成的客户端能力。

像素只在视觉验证和用户交互层存在，不应成为持久协议地址。

### 7.2 无状态 MCP，请求显式携带项目句柄

[MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28) 使用无状态、自包含请求；官方说明应用若需要跨调用状态，应创建显式 handle 并由模型在后续参数中传回，而不是把状态藏在传输 session 中。

这与项目画布非常契合。每个调用显式携带：

```json
{
  "projectId": "project_123",
  "path": "compositions/main.vml",
  "changes": "PATCH #clip_42\nopacity=\"0.8\""
}
```

`projectId` 是可授权、可审计、模型可见的应用句柄。MCP 层不需要伪造 Agent Session，也不需要 `replyTo` 路由字段。

### 7.3 MCP Canvas Bridge

完整链路为：

```text
External Agent / MCP Host
  -> OAuth and project authorization
  -> stateless MCP tools/call(projectId, ...)
  -> stable mcpToolCallId
  -> committed Project Message Stream
  -> assigned connected Editor
  -> ClientToolRunner
  -> EditorToolHost
  -> FileMutationTool / semantic canvas operation
  -> DocumentEditing / StructuredEditing
  -> Y.Doc
  -> tool result in Project Inbox
  -> MCP result
```

这条链路有三个重要约束：

1. MCP 只发布已提交的完整调用，不伪造模型流式 delta；
2. MCP、内置 Agent 和人类 UI 最终进入同一 Editor 领域写路径；
3. `toolCallId` 只负责关联调用和结果，不承担隐式会话身份。

### 7.4 Project Stream 与 Session Stream

内置对话 Agent 有 Session Message Stream；外部 MCP 调用可能没有对话 Session，因此使用 Project Message Stream。两者可以在入口和展示上不同，但连接浏览器必须复用同一 ClientToolRunner 与执行授权机制。

```text
Session stream ─┐
                ├─> ClientToolRunner -> Editor mutation chain
Project stream ─┘
```

如果为 MCP 另写一套服务器端画布修改逻辑，人类 Editor 和外部 Agent 就会形成两份领域语义。

### 7.5 连接 Editor 与有限等待

某些项目编辑需要浏览器持有本地协同状态或平台能力。MCP 调用到达时：

- 若有获得项目执行租约的 Editor，分配调用；
- 多个标签页只能有一个获得执行 token；
- 若客户端断开，按工具策略重分配、中断或等待；
- 若长期没有 Editor，调用在有限 deadline 后返回清晰状态；
- 不应在服务端悄悄维护一份不同步的“影子画布”。

### 7.6 与 MCP Tasks 的关系

最新 MCP 将 Tasks 作为可选扩展，支持长操作、轮询和持久 handle。画布结构编辑通常是短调用；生成视频、转写或大规模导出则适合返回 Task handle。内部 Task/Monitor 可以投影到 MCP Tasks，但不应为了适配协议复制一套权威任务状态。

> **[待补 A-C2：MCP Tasks 映射]** 固定内部 Task status/version/output 与 MCP task handle、`tasks/get`、`tasks/update` 的对应关系及取消语义。

## 8. 安全与一致性不变量

### 8.1 运行不变量

1. 每个 Session 最多一个活动 Process；
2. 所有可恢复异步输入先进入持久 Inbox；
3. run signal 可以丢失，不能承载唯一状态；
4. WaitPort 不复制消息，只描述路由；
5. Task 数据库状态先提交，通知后发送；
6. Monitor 只接受版本不旧于已见版本的快照；
7. 已提交消息本身不授予浏览器执行权；
8. 每个客户端工具调用只有一个有效 execution token；
9. MCP 调用必须显式授权 projectId 和操作权限；
10. MCP 与内置 Agent 复用同一领域修改链。

### 8.2 故障语义

必须定义并测试：

- Inbox 写入后、signal 前崩溃；
- signal 后、lease 前重复投递；
- lease 后、Process 启动前崩溃；
- 模型输出中途崩溃；
- 工具已经执行、结果写 Inbox 前断线；
- Task 提交后、TaskChanged 前崩溃；
- Monitor 写上下文后、关闭 WaitPort 前崩溃；
- MCP 调用分配后浏览器关闭；
- 工具结果已经提交但 MCP HTTP 响应丢失。

系统追求的是“最终可恢复 + 副作用有明确幂等边界”，而不能笼统声称所有外部副作用 exactly-once。

## 9. 与相关工作的关系

### 9.1 推理—行动 Agent

[ReAct](https://arxiv.org/abs/2210.03629) 和大量 tool-use Agent 研究了模型如何交错推理与行动，主要对应本文内循环。本文的重点是内循环之外的持久消息、单 Process 租约、异步结果和崩溃恢复。

### 9.2 Durable execution 与 actor/message systems

Actor、消息队列、event sourcing、workflow engine 和 durable execution 已经广泛研究单线程实体、持久事件和故障恢复。本文不把这些基础原理声明为新发明。潜在贡献是针对生成式 Agent 的特殊边界：模型 step、UIMessage 快照、客户端工具执行权、Task Monitor 压缩和项目画布桥接如何收敛为少数权威概念。

> **[待补 A-L1：分布式系统文献]** 精读 Orleans/virtual actor、Temporal/Durable Functions、event sourcing、transactional outbox 与 lease-based work scheduling 的论文或正式技术报告，明确已有保证和本文差异。

### 9.3 MCP

[MCP 2026-07-28 规范](https://modelcontextprotocol.io/specification/2026-07-28) 提供无状态工具互操作、显式能力和可选 Tasks 扩展。本文不是 MCP 协议论文，而是研究如何把 MCP 调用安全地桥接到一个有协同状态、客户端执行权和无限画布的创作应用。

### 9.4 GUI 与 ACI

[SWE-agent](https://arxiv.org/abs/2405.15793)、[OSWorld](https://arxiv.org/abs/2404.07972) 和 [AgenticVBench](https://arxiv.org/abs/2605.27705) 都说明 Agent harness/接口会显著影响行为与成功率。本文的 MCP Canvas 章节延续同一观点：外部 Agent 应调用稳定语义，而不是复刻人类鼠标路径。

## 10. 贡献与创新性评估

建议把潜在贡献收敛为：

1. 一个明确区分模型工具内循环与持久消息外循环的 Agent 运行模型；
2. 一个以单 Inbox、Session lease 和可恢复 sweep 保证串行 Session Process 的实现；
3. 一个将 WaitPort 作为路由状态、Monitor 作为权威 Task 更新 sidecar 的长任务聚合机制；
4. 一个把无状态 MCP 语义调用桥接到有状态浏览器 Editor、同时保持单一领域写路径的 Canvas 执行协议；
5. 一套覆盖崩溃、多副本、重复通知和浏览器断连的故障注入评测。

其中“内外循环”“Inbox”“lease”“Monitor”“MCP”单独都不是新概念。论文价值取决于概念压缩是否带来可测的可靠性与成本收益，以及是否能用故障实验而非架构图证明。

## 11. 实验设计

### 11.1 基线

1. 一个常驻进程承担整段 Session 生命周期；
2. 每个事件直接启动 Process、无 Session lease；
3. 工具结果、Task 更新和 Clock 各自独立队列；
4. Agent 主动轮询所有 Task；
5. 每个 Task 进度事件都唤醒模型；
6. 双环 + 单 Inbox + WaitPort + Monitor；
7. MCP 像素 GUI 操作；
8. MCP 服务器端影子状态修改；
9. MCP 语义操作 + 连接 Editor 执行。

### 11.2 工作负载

- 短对话、多次同步工具；
- 多标签页打开同一项目；
- 5、20、100 个并发生成 Task；
- Task 进度高频更新；
- Session 运行中连续用户追问；
- Clock 到期与用户消息同时到达；
- 外部 MCP 批量修改画布；
- 浏览器在线、重连、离线三种条件；
- Worker 多副本扩缩容与滚动部署。

### 11.3 指标

- 同 Session 并发 Process 违规次数；
- 重复客户端工具执行率；
- Inbox 写入到 Process 开始延迟；
- signal 丢失后的 sweep 恢复时间；
- Process 崩溃后的任务完成率；
- 每个用户目标的模型调用次数；
- 原始 Task update 数、Monitor context 数与压缩率；
- 进度倒退或 cohort 计数错误次数；
- 长任务期间 token 与费用；
- MCP 调用到画布提交/结果返回延迟；
- 多标签页执行权切换成功率；
- 无连接 Editor 时的明确失败/等待比例；
- Redis、数据库和 Worker 资源使用。

### 11.4 故障注入矩阵

> **[待补 A-D1]** 在每个持久化边界之后自动 kill Worker，至少重复 100 次并验证最终消息、Inbox、lease、WaitPort 和 Task 状态。

> **[待补 A-D2]** 重复、乱序和丢弃 TaskChanged，验证权威 DB 重读与版本 CAS。

> **[待补 A-D3]** 重复 run signal 并增加 Worker 数量，验证同 Session 单 Process。

> **[待补 A-D4]** 在客户端工具已修改 Y.Doc 但结果未提交时断网，验证幂等键、结果恢复和用户可见状态。

> **[待补 A-D5]** 对 MCP HTTP 响应丢失进行重试，验证同一调用 ID 不重复修改画布。

## 12. 研发阶段必须埋点

> **[待补 A-T1]** 为每个 Inbox entry 记录 producer、persistedAt、signaledAt、claimedAt、ackedAt、processId。

> **[待补 A-T2]** 为 lease 记录 owner、acquire/renew/release、失效原因和抢占失败。

> **[待补 A-T3]** 为每次 Process 记录加载消息数、吸收 Inbox 数、模型 step 数、退出原因和未完成 WaitPort。

> **[待补 A-T4]** 为 Task/Monitor 记录原始 version、丢弃旧版本、cohort 大小、聚合次数、唤醒原因和 token。

> **[待补 A-T5]** 为 Client Tool 记录 assignment、execution token、浏览器 identity、开始/完成/撤销和重复观察。

> **[待补 A-T6]** 为 MCP 记录授权 projectId、toolCallId、Project Stream messageId、Editor assignment、Y.Doc transaction 与最终结果的关联链。

## 13. 中文系列文章母稿

### 13.1 内循环负责思考，外循环负责活着

Agent 一次调用模型后，模型可能搜索、读取、编辑，再根据工具结果继续。这是内循环。它适合解决“当前这件事下一步做什么”。

但视频生成可能十分钟后才完成，用户可能在此期间追问，浏览器也可能断开。不能让一个 JavaScript Promise 永远挂着，假装进程不会重启。于是需要外循环：任何新事实先写进一个持久 Inbox，再唤醒这个 Session。

每个 Session 同时只能有一个 Process。新的消息到达时，如果旧 Process 还在，它会在下一步吸收；如果已经结束，Worker 获取租约后启动新的 Process。信号丢了也没关系，sweep 会从 Inbox 发现尚未处理的事实。

内循环让 Agent 会做事，外循环让 Agent 在真实系统里不会因为重启和等待而失忆。

### 13.2 Monitor 不应该成为第二个任务系统

一个 Agent 同时生成五个素材时，最容易出现的设计是再建一个 Monitor 服务，保存一份自己的任务列表和进度队列。很快，数据库里的 Task、Monitor 的列表和 Agent 的等待状态就会互相不一致。

更合理的做法是只保留一个权威 Task。Task 先提交数据库，再发“它变了”的提示。Monitor 重新读数据库，只观察那些已经有 WaitPort 表示 Agent 正在等待的任务。

进度也不应该每变化 1% 就叫醒一次模型。Monitor 把五个任务的最新状态压成一条上下文，在失败、完成或达到检查点时再唤醒 Agent。它是信息压缩器和唤醒策略，不是第二个任务平台。

### 13.3 MCP 如何操纵画布

让外部 Coding Agent 操纵画布，最直观的办法是给它远程鼠标。但画布只要缩放一下，昨天的坐标就失效了。Agent 真正想表达的通常不是“从 817 像素拖到 1200 像素”，而是“把这个片段移动到那条轨道”。

MCP 调用应该携带明确的项目 ID 和语义操作。服务端完成授权后，把调用写进项目消息流；当前获得执行权的浏览器 Editor 使用和内置 Agent 相同的编辑链修改 Y.Doc，再把结果按 toolCallId 返回。

外部 Agent、内置 Agent 和人类界面可以有不同入口，但不能各自拥有一套画布真相。MCP 的作用是标准化连接，不是绕过编辑器的数据模型。

## 14. 写作与研发检查清单

- [ ] 在所有文档中只承认内、外两个业务循环；
- [ ] 明确 listener、SSE、reconnect、sweep 和 Monitor 的非业务循环身份；
- [ ] 所有生产者坚持 Inbox first、signal second；
- [ ] 验证 Session lease 的 acquire/release 原子边界；
- [ ] WaitPort 不存第二份结果队列；
- [ ] TaskChanged 只作提示，消费者重读权威 Task；
- [ ] 固定 Monitor 唤醒策略和 cohort 关闭条件；
- [ ] 对每个崩溃窗口做自动故障注入；
- [ ] MCP 显式携带 projectId，不引入隐藏 transport session；
- [ ] MCP 与内置 Agent 复用同一 ClientToolRunner 和 Editor 写路径；
- [ ] 区分 MCP 短画布调用与长 Task；
- [ ] 不声称端到端 exactly-once，逐项说明幂等边界。

## 参考文献（首轮）

1. S. Yao et al., [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629), 2022/2023.
2. Model Context Protocol, [Specification 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28).
3. Model Context Protocol Blog, [The 2026-07-28 Specification](https://blog.modelcontextprotocol.io/posts/2026-07-28/), 2026.
4. J. Yang et al., [SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering](https://arxiv.org/abs/2405.15793), 2024.
5. T. Xie et al., [OSWorld: Benchmarking Multimodal Agents for Open-Ended Tasks in Real Computer Environments](https://arxiv.org/abs/2404.07972), 2024.
6. Z. Cao et al., [AgenticVBench: Can AI Agents Complete Real-World Post-Production Tasks?](https://arxiv.org/abs/2605.27705), 2026.
7. **[待补 A-L1]** Virtual actor、durable execution、transactional outbox、event sourcing、lease scheduling 与 workflow recovery 的系统论文。

