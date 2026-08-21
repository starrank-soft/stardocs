# StarCut 中文论文与技术文章路线图

> 更新：2026 年 8 月
>
> 本目录保存中文母稿。每一份同时包含论文结构、面向公众的技术文章、相关工作、实验设计和研发待补项。

## 文稿清单

| 编号 | 论文母稿 | 面向公众的文章 | 当前定位 |
|---|---|---|---|
| 01 | [班车式解码：统一时序需求与 GOP Flight 调度](./01-gop-shuttle.md) | [Video GOP Decode Flight：像调度高铁一样实现丝滑剪辑](./article-video-gop-decode-flight.md) | 多媒体系统与调度优化 |
| 02 | [REST-Edit：面向 AI 的轻量级、流式、语义化精确编辑协议](./02-rest-edit.md) | 让 AI 像改代码一样精确剪视频 | 协议与 Agent 接口优化 |
| 03 | [Edit-as-Code：让 AI 像 Coding Agent 一样完成视频剪辑](./03-edit-as-code.md) | [Edit Video as Code：将视频工程映射为 Coding Agent 可读写的虚拟文件视图](./article-edit-video-as-code.md)（面向技术同学） | Agent-Computer Interface 与创作系统 |
| 04 | [单时钟可寻址 Motion Graphics](./04-single-clock-motion-graphics.md) | [Video Motion Graphics：把动画变成时间的函数](./article-video-motion-graphics.md) | 浏览器图形运行时 |
| 05 | [双环 Agent：Inbox、WaitPort、Monitor 与 MCP Canvas](./05-durable-two-loop-agent.md) | [Video Agent Loop：StarCut 的硬核实践，DeepSeek Harness 能否满足视频场景？](./article-agent-loop.md) | Agent 运行时与系统设计 |

## 为什么是五份母稿

最初列出的主题可以分成四篇 Paper/文章和三个 Agent-Loop 章节。Agent-Loop 的三个部分共享同一组不变量：外部事件统一进入 Inbox，WaitPort 只保存路由状态，Monitor 只聚合权威 Task，MCP Canvas 调用也通过普通消息和工具结果收口。当前先写成一篇完整公开文章；只有后续内容量和实验结果确实需要时才拆成系列，避免预先重复解释同一运行时。

因此当前结构是：

- 四篇相互独立的媒体/Agent Paper；
- 一份双环 Agent 总设计，以及一篇贯通“内外循环”“Monitor”“MCP 画布”的公开长文。

如果后续实验量足够，第 05 稿可以再拆为“持久 Agent Loop”和“MCP Canvas Bridge”两篇；现在不提前拆，以免制造重复贡献。

## 投稿边界

### 01 GOP Flight

只研究压缩视频的时序需求、GOP 路线规划、在途覆盖、缓存与多交互复用。不要把普通关键帧 seek、WebCodecs、缩略图或缓存本身写成创新。

**主实验终点：** 解码工作放大、首帧/精确帧延迟、路线复用、资源占用和交互质量。

### 02 REST-Edit

只研究局部协议表示、短语义地址、逐行/结构流式解析、实时连续前缀与提交围栏。JSON Patch 是相邻标准，不是本协议的来源或别名；也不能通过贬低 JSON Patch 的协同和 Undo 能力来证明差异。

**主实验终点：** 正确率、首个有效修改延迟、token/字节、无关写集合、断流恢复和工具调用轮数。

### 03 Edit-as-Code

研究完整 ACI：项目文件视图、导航、局部编辑、模式/引用/视觉验证、人类接管和 Agent 工作闭环。REST-Edit 只是它的一个组件。

**主实验终点：** 真实后期制作任务成功率、工具步数、上下文、可恢复性和人工接管成本。

### 04 Motion Graphics

研究静态元数据 + 可执行 SVG 组件 + 单一编辑器时钟 + 随机访问 `render(time)` + 同源预览/导出。不要声称发明 SVG、矢量动画、seek 或代码动画。

**主实验终点：** 随机访问确定性、预览/导出一致性、交互延迟、导出吞吐和缓存复用。

### 05 Durable Two-Loop Agent

研究内外两个业务循环、单 Inbox、Session lease、WaitPort、Monitor 聚合与 MCP Canvas Bridge。Redis listener、SSE、重连和 sweep 是交付/恢复机制，不是更多业务循环。

**主实验终点：** 崩溃恢复、单 Session 串行性、重复副作用、Monitor 压缩率、模型唤醒成本和 MCP 画布完成率。

## 主题关系

```text
Edit-as-Code（完整 Agent 编辑范式）
  ├─ REST-Edit（结构化工程的局部编辑协议）
  ├─ Durable Two-Loop Agent（消息、等待、任务与外部 MCP）
  └─ Project DSL / Artifact / verifier

Browser media runtime
  ├─ GOP Flight（压缩视频帧怎样按需求到达）
  └─ Single-Clock MG（程序化 SVG 怎样在任意时间求值）

Composition preview/export
  └─ 同时消费视频帧与 MG 像素，但二者的研究问题不同
```

## 当前创新性判断

| 选题 | 可发表贡献的清晰度 | 最大风险 | 建议 |
|---|---|---|---|
| REST-Edit | 较高 | 若只有语法、没有模型与断流实验，会像工程格式说明 | 优先建立基准和流式故障实验 |
| Edit-as-Code | 较高但范围大 | 容易写成产品介绍，或与 REST-Edit 重复 | 以真实任务和 ACI 消融为主 |
| GOP Flight | 中等，系统深度高 | 既有解码/调度研究多，单项机制不新 | 证明统一协调和跨模式复用的净贡献 |
| Single-Clock MG | 中等 | Lottie/Web Animations/浏览器渲染已有大量工作 | 以随机访问确定性和同源语义为核心 |
| Durable Two-Loop Agent | 中等到较高 | 基础概念来自消息系统和 durable execution | 必须用多故障窗口和多副本实验支撑 |

论文不需要“翻天覆地”。系统性优化、接口改进和负结果都可以贡献知识。关键是准确承认已有工作，并用可复现实验证明新组合或新取舍在目标场景里的效果。

## 共同证据规范

每个最终数字都必须能够追溯到：

- 冻结的代码 commit；
- 精确的浏览器、操作系统、设备、模型与模型版本；
- 媒体、项目、MG 或 Agent 任务 manifest；
- 原始模型输出、工具事件、解码事件或任务事件；
- 可重复运行的分析脚本；
- 重复次数、分布和置信区间；
- 只改变被研究机制的公平基线；
- 失败案例和排除规则。

建议所有埋点共享下列顶层字段：

```text
experimentId
runId
paperTrack
codeCommit
environmentId
workloadId
variantId
startedAt
completedAt
outcome
```

每篇再扩展自己的事件 schema。不要等投稿前才从产品日志反推实验数据。

## 共同研发优先级

1. 冻结最小可执行 benchmark schema；
2. 为 REST-Edit 和 Edit-as-Code 保存完整模型/工具 trace；
3. 为 GOP Flight 保存 sample、flight、decoder、cache 全生命周期；
4. 为 MG 保存 `render(t)`、SVG hash 与像素 hash；
5. 为 Agent Loop 保存 Inbox、lease、Process、WaitPort、Task version 和 MCP call 的关联链；
6. 建立统一 fault injection harness；
7. 建立 artifact/trace 脱敏与长期保存规则；
8. 每月做一次相关工作更新，特别关注 2026 年 Agent 视频编辑与 MCP 论文。

## 旧英文稿

目录上层的 `rest-edit-draft.md` 与 `gopflight-draft.md` 是较早的英文工作稿。中文稿反映当前实现，尤其是 REST-Edit 已经具备渐进输入与提交收口的统一路径。后续翻译英文时应以中文母稿为准，不要反向复制旧稿中的过期结论。
