# StarCut Papers

当前论文与技术文章以中文母稿为准：

- [中文路线图与投稿边界](./zh/README.md)
- [01 — 班车式解码与 GOP Flight](./zh/01-gop-shuttle.md)
- [02 — REST-Edit 协议](./zh/02-rest-edit.md)
- [03 — Edit-as-Code](./zh/03-edit-as-code.md)
- [Article — 为什么 AI 会写代码，却还不太会剪视频](./zh/article-edit-video-as-code.md)
- [04 — 单时钟 Motion Graphics](./zh/04-single-clock-motion-graphics.md)
- [05 — Durable Two-Loop Agent](./zh/05-durable-two-loop-agent.md)

## 旧英文工作稿

- [REST-Edit draft](./rest-edit-draft.md)
- [GOPFlight draft](./gopflight-draft.md)

这两份英文稿形成较早，仅用于保留研究过程。中文稿反映当前架构和实现，后续英文投稿稿应从中文母稿重新组织。其中 REST-Edit 的旧英文稿曾把渐进执行列为缺口，而当前实现已经统一了实时 `feed`、提交 `apply`、连续前缀检查和单一 Undo 捕获边界。

## Evidence Policy

任何最终论文中的数字都必须能够追溯到冻结的代码 commit、精确环境与模型版本、任务/媒体 manifest、原始 trace、可重复分析脚本、重复运行统计和公平对照组。研发阶段的具体缺口已经标在各中文母稿中。
