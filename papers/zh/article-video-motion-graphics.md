# Video Motion Graphics：把动画变成时间的函数

![Video Motion Graphics 封面：时间线从任意时刻驱动一段可编辑 SVG 动画，预览、Scrub 与导出共享同一份视觉程序](../assets/motion-graphics/cover.svg)

在视频编辑器里说一句“做一个产品发布标题”：文字从画面中央弹出，光带扫过，数字从 0 走到 100，最后干净退场。几秒钟后，这段 Motion Graphics（下文简称 MG）出现在时间线上。创作者把播放头拖到 1.7 秒查看中间状态，把英文标题换成中文，裁掉最后半秒，再把同一个样式复用到另一个镜头。

这一连串操作要求 AI 交付的不是一段已经封死的 MP4，也不是只能从头播放的网页动画。它需要保留文字、颜色和运动逻辑，能够在任意时刻准确停住，还要透明地叠在视频上，最终导出的每一帧与编辑器中的预览保持同一语义。

我们没有绕开成熟方案。Remotion、HyperFrames、iframe、HTML-in-Canvas、服务端 Chromium 与派生视频，我们都沿着实际产品链路试过。它们分别解决了程序化动画、样式隔离或视频输出，却始终有一个问题没有自然消失：浏览器里的动画，怎样在任意时刻直接交给视频合成器一张 Bitmap？这段路线最后收敛为一条原则：**MG 是一个以时间为输入的视觉程序。编辑器问它某一时刻长什么样，它就直接给出那一帧。**

## 一、MG 不是另一段视频

![像素视频保存已经展开的画面，MG 保存可被时间线反复求值的视觉逻辑](../assets/motion-graphics/pixels-vs-program.svg)

*视频保存结果，MG 保存产生结果的方法。*

视频将画面压缩成一串有时间顺序的像素；MG 保存的是图形、文字、参数和运动关系。前者适合拍摄素材与复杂自然运动，后者适合标题、Lower Third、数据图表、产品标注、字幕包装和品牌动画。

这个区别直接决定了编辑体验。视频中的标题已经成为像素，换一个词通常要回到源工程重新渲染；MG 的标题仍然是文字，颜色仍然是颜色，数字仍然可以参与计算。同一份动画可以在不同 Clip 中填入不同内容，也可以在不降低清晰度的情况下适配不同分辨率。

数据量能把这种表示差异说得更直观。当前系统库中的 18 个 MG 源文件大小为 2.9–4.9 KB，平均约 3.6 KB。一个 5 秒、30 fps、1920×1080 的动画若完全展开为未压缩 RGBA，则对应约 1.24 GB 的帧数据。两者不是编码压缩率的公平比较：播放器并不会把所有 RGBA 帧永久存下来，成品视频也会经过压缩。它说明的是另一件事——几 KB 的程序描述了一个可以在 150 个时刻被求值、仍然保留语义和参数的视觉对象。

对 AI 而言，这也比直接生成视频多了一层可用性。模型生成的不只是“看起来像一个标题”的画面，而是一份人和 Agent 可以继续读取、修改、验证和复用的源文件。

<!-- RESEARCH: Record source size, SVG node count, variable count, first-frame time and rendered-video size for the curated MG corpus. Do not present raw RGBA expansion as a codec compression benchmark. -->

## 二、为什么没有停在 Remotion 和 HyperFrames

![MG 方案从 Remotion 与 HyperFrames、HTML-in-Canvas 和服务端渲染，逐步收敛为单一 SVG Component 运行时](../assets/motion-graphics/architecture-evolution.svg)

*成熟的动画框架解决了怎样让网页按帧运动，我们还需要解决怎样让这一帧进入编辑器的像素管线。*

我们并不是一开始就想设计自己的 MG 格式。Remotion 已经证明 React 组件可以由当前帧驱动，并提供浏览器 Player、Scrub 和完整的渲染体系；HyperFrames 则更直接地把 HTML 变成可逐帧寻址的视频。两者都比从零搭一套网页动画基础设施成熟，所以我们的第一选择也是沿着它们走。

Remotion 很适合以 React 工程构建一支完整视频。我们当时试用的生产链路里，浏览器承担 Player 与预览，可靠成片仍然要进入 Chromium 服务端渲染；今天 Remotion 已经补充了浏览器渲染能力，但没有改变我们面对的集成边界。我们需要嵌入的不是另一支由 React 接管的视频，而是现有时间线中的一个 Clip：它要服从编辑器的时钟，与 WebCodecs 视频共同分层，还要在缩略图和导出请求到来时直接返回 `ImageBitmap`。Player 能在页面里显示某一帧，不等于这个 DOM 画面已经成为现有 Canvas 合成器可以消费的像素对象；引入完整的 React Composition，也会在已有工程模型旁边再建立一套组件、资源与渲染边界。

HyperFrames 与我们的目标更接近，我们也实际用过。它让普通 HTML 与 GSAP 按确定时间 Seek，浏览器预览自然，模型也很容易生成。真正卡住我们的不是动画能不能播放，而是像素怎样出来：HyperFrames 的可靠成片路径由 Headless Chrome 逐帧 Capture，再交给 FFmpeg；在我们的旧方案中，缩略图与最终视频也因此进入服务端 Task Worker。浏览器负责实时 HTML，服务端负责 PNG 或 MP4，同一段 MG 再次出现两条运行路径。

HTML-in-Canvas 看起来像缺失的那座桥：如果浏览器能把已经排版完成的 DOM 直接画入 Canvas，HyperFrames 页面就能继续留在浏览器里。我们专门调研并测试过 `drawElementImage()`。问题是它仍依赖实验性 Chrome Flag，不能作为面向用户的浏览器基线；在高频取帧、多实例和复杂页面下，我们还遇到过不稳定与崩溃。`html2canvas` 也不是等价替代，它会用 JavaScript 重新解释 DOM 和 CSS，既损失浏览器原生渲染能力，也难以承担逐帧视频管线。

于是我们也走过服务端 Chromium 与预渲染视频：先把 MG 编译成 PNG 或 MP4，编辑器再按普通媒体处理。它能产出像素，却把一次改字、换色变成远程渲染任务；透明叠加需要额外的 Alpha 编码与解码链路，实时预览看到的是网页，最终合成使用的却是派生视频。字体、时钟、资源加载或运行时版本的任何差异，都可能让两边逐渐偏离。

SVG 改变了这条链路。它既可以作为活的 DOM 场景留在页面里，又是浏览器原生认识的图像格式。组件执行 `render(time)` 后，系统克隆当前 SVG、嵌入字体并序列化为 Blob，浏览器即可解码、画入 Canvas，再生成 `ImageBitmap`。整个过程留在客户端，不需要截取一张网页，也不需要启动另一台 Chromium。动画状态和像素结果第一次来自同一个对象。

这才是方案收敛的决定性原因。我们保留 Web 最有价值的部分——可编程、矢量、模型熟悉——同时把画面边界收窄为一个 SVG：每个 MG 是一个自包含的 JavaScript Component，宿主提供动画能力、参数与准确时间，组件既不拥有自己的播放世界，也不需要第二套像素运行时。

## 三、把时间从动画内部拿出来

![无论时间线按怎样的顺序询问，同一个时间输入都必须得到同一个视觉状态](../assets/motion-graphics/time-as-input.svg)

*播放是连续提问，Scrub 是乱序提问，导出是按帧提问。MG 回答的是同一个问题。*

这套运行时最重要的契约可以写成一个函数：

```text
Frame = Render(Source, Props, Time)
```

`Source` 是动画源，`Props` 是这个 Clip 的文字、颜色或数字等实例参数，`Time` 是时间线换算后的片段局部时间。只要三者相同，得到的视觉状态就应该相同。

这意味着动画不能靠“上一帧再往前走一点”得到下一帧，也不能读取 `Date.now()`、`performance.now()` 或自行启动计时器。用户可能按 0 秒、4 秒、1 秒、3 秒的顺序拖动，缩略图也可能先取结尾再取开头。每次调用都必须直接重建目标时刻，而不是假设此前的帧已经按顺序出现。

播放并没有因此变成另一套机制。播放只是时间线连续提供 `0、1/30、2/30……`；暂停 Seek 提供一个确定时间；Scrub 提供不断变化且可能反向的时间；导出则按照输出帧率逐帧询问。不同交互共享同一个求值模型。

组件内部仍然可以使用 GSAP。区别在于 GSAP Timeline 始终暂停，宿主每次将它 Seek 到绝对时间；程序化图表、路径和数字动画也直接由 `time` 计算。动画库负责插值与缓动，编辑器拥有唯一时钟。

时间线还会先把全局时间映射为 Clip 的局部时间，处理片段起点、源内偏移、裁切和播放速率，再按 Composition 帧率量化。组件不需要理解工程中自己被放在第几秒，也不需要知道用户正在播放还是 Scrub。

这个边界让 MG 获得了视频编辑最需要的能力：随机访问。它不是一个只能控制 `play` 和 `pause` 的播放器，而是一张可以在任意时刻直接求值的视觉函数。

<!-- RESEARCH: Build a random-order determinism suite. For each MG, compare canonical SVG state and rasterized output across forward, reverse, shuffled, repeated and large-jump time queries. -->

## 四、为什么是 JavaScript Component + SVG

![一份 MG 源文件由静态元数据、可替换变量、SVG 场景和确定性时间函数组成](../assets/motion-graphics/mg-source-contract.svg)

*静态部分让系统不执行代码也能理解资源；可执行部分保留动画表达力。*

一份 `.mg` 源文件只有几个稳定的公共面：静态元数据声明画布、时长、字体和可替换变量；默认导出的 Web Component 建立一个 SVG；`props` 接收实例参数；`render(time)` 给出准确时刻的状态。

```js
export const metadata = {
  version: 1,
  width: 1920,
  height: 1080,
  duration: 5,
  variables: [
    { id: "title", type: "string", label: "Title", default: "Launch" },
    { id: "accent", type: "color", label: "Accent", default: "#67e8f9" },
  ],
};

export default class LaunchTitle extends HTMLElement {
  #props = {};
  set props(value) { this.#props = value; /* update semantic content */ }
  get props() { return this.#props; }
  connectedCallback() { /* mount one SVG and a paused timeline */ }
  render(time) { /* reconstruct the exact state at time */ }
  disconnectedCallback() { /* release retained resources */ }
}
```

元数据必须是直接、可静态读取的字面量。服务端可以用 AST 解析宽高、时长、字体与变量，而不执行任意项目代码；导入、生成、资源列表、时间线默认时长和运行时校验都从同一份描述投影。模块真正进入浏览器时，系统还会再次确认运行时导出的元数据与静态结果一致。

选择 JavaScript 不是为了重新发明一个通用动画标准。它首先是一种模型已经非常熟悉的表达：语义分组、循环、条件、数字插值和程序化路径都可以直接写清楚，Coding Agent 也能局部修改源文件。近期的 OmniLottie 与 LottieGPT 都专门设计了 Lottie Tokenizer，用来压缩原始 JSON 中大量结构与格式 Token；这也从另一个方向说明，动画表示是否适合模型生成，并不是文件扩展名层面的小事。

SVG 则提供了适合 MG 的画面边界：文字、路径、渐变、遮罩、裁剪与有限滤镜保持矢量表达，背景天然可以透明，DOM 节点也能被程序精确定位。Shadow DOM 隔离组件内部的样式和节点命名，让多个 MG 实例可以共存。

表达力越大，运行边界越需要明确。当前格式静态拒绝模块导入，创作契约也把网络、宿主文档、视频、Canvas、WebGL、`foreignObject` 和外部媒体排除在外；宿主只注入一份统一动画运行时。这里的 Shadow DOM 是组件封装，不是安全沙箱，网络与全局访问也不能只靠语法检查彻底封住。当前 `.mg` 属于可信项目代码，公开接收第三方可执行模块需要另一套隔离与配额设计。

## 五、预览保持为矢量，需要像素时再栅格化

![同一 MG Component 在实时预览中保持为活的 SVG，需要缩略图和导出时才按准确时间转成 ImageBitmap](../assets/motion-graphics/live-svg-pixel-path.svg)

*不要为了最终需要像素，就让整个编辑过程提前变成像素。*

实时预览不需要每帧截图。MG Component 直接作为活的 SVG 留在 DOM 中，时间线只更新它的局部时间、可见性与外部变换。文字和路径始终保持矢量，修改参数也不需要等待一次离屏渲染。

普通视频、图片和文字合成主要位于 Canvas；MG 则位于 HTML/SVG 表面。为了让一个 MG 既能盖在视频上，也能被更高轨道的视频盖住，预览区不是“一张 Canvas 加一个最上层 DOM”，而是按轨道顺序派生出交替的表面：

```text
Canvas band   视频 / 图片 / 文字
HTML band     Motion Graphics
Canvas band   更高的视频 / 图片 / 文字
HTML band     更高的 Motion Graphics
```

这套层级只在工程结构变化时重建。播放过程中，Canvas Worker 继续解码和合成媒体，主线程只让已经挂载的 SVG 走到新时间。MG 不需要被逐帧搬进 Worker，也不会为了叠加顺序退化成截图视频。

缩略图、单帧抽取和最终导出确实需要像素。此时系统在准确时间调用同一份组件实现，克隆当前 SVG，补齐字体，将它序列化并栅格化为 `ImageBitmap`，再按时间线的坐标、缩放、旋转、透明度、圆角、混合模式与轨道顺序合成到 Canvas。

```text
render(time)
  → current SVG state
  → clone + embed fonts
  → ImageBitmap
  → Canvas composition
  → thumbnail / frame / encoder
```

预览与导出不一定使用同一个 DOM 实例，但它们使用同一份源、同一套参数解析、同一时钟映射和同一段 `render(time)`。系统没有第二个 MG 解释器，也不需要让服务端重新猜一遍动画。

这保证的是语义同源，而不是在不同设备和浏览器上承诺每个抗锯齿像素完全相同。字体文件、DPR、色彩空间和 SVG 栅格化实现仍然会影响像素；因此字体被显式声明并在像素输出时嵌入，导出环境也需要固定。

<!-- RESEARCH: Measure live seek latency, SVG serialization, font embedding, ImageBitmap rasterization and Canvas composition separately at 1080p and 4K. Record P50/P95 and long tasks. -->

<!-- RESEARCH: Establish preview-vs-export image-difference thresholds under a fixed browser, font set, DPR and color space. Report semantic equality separately from pixel equality. -->

## 六、一份源，可以有很多时间线实例

![稳定的 MG 资源保存源与变量模式，不同时间线 Clip 只保存各自参数和外部呈现](../assets/motion-graphics/artifact-and-instances.svg)

*模板拥有内部设计，Clip 拥有这一次使用的内容与位置。*

MG 源与时间线实例承担不同职责。源文件拥有 SVG 场景、内部运动、默认文字、颜色和变量模式；Clip 拥有这一次使用的参数覆盖，以及位置、缩放、旋转、透明度、图层、裁切和播放速率。

因此，一个 Lower Third 可以在十个镜头中复用，每个 Clip 填入不同姓名和职位，却仍然引用同一份动画。修改某个 Clip 的名字不会复制源代码；改进源中的入场缓动，则会让所有引用它的实例获得新版本。

时间线保存的是稳定的 Artifact ID，不是 CDN 地址，也不是内容哈希。每次编辑生成一个不可变的新内容 Blob，Artifact 的当前哈希原子地前进；Clip 的引用保持不变，运行时却能根据哈希准确替换组件、失效帧与刷新资源卡片。

AI 生成 MG 时，模型输出的是源代码。系统先静态检查模块、元数据和组件契约，再把合法内容提交成 Artifact；AI 或人类后续修改同一源文件时，也经过同一提交边界。生成、直接编写和局部编辑没有三套资源格式。

这让 MG 自然进入 Coding Agent 的工作方式。Agent 可以读出源文件，看见 `title`、`accent` 和具体视觉分组，针对一个范围修改动画；普通创作者则只需要在参数面板里换字、换色。两种编辑深度作用于同一个资源，而不是分别维护“模板版”和“代码版”。

<!-- RESEARCH: Record model generation validity, automatic repair rate, manual edit distance, variable-schema quality and time-to-first-usable-MG across representative prompts. -->

## 七、Lottie、Remotion、HyperFrames 都提供了重要参照

![Lottie、Remotion、HyperFrames、Rive 与可执行 SVG MG 分别从交换格式、程序化视频、HTML 视频、交互状态机和编辑器内可寻址组件切入](../assets/motion-graphics/related-approaches.svg)

*这里不是选出一个普遍赢家，而是确认哪一种边界适合视频编辑器中的可编辑 MG。*

Lottie 是成熟的 JSON 矢量动画格式，跨平台播放器和 After Effects 工作流是它最强的价值。它非常适合设计师制作、导出并交付一个可移植动画。2026 年的 OmniLottie 与 LottieGPT 已经进一步证明，模型可以原生生成高质量、可编辑的 Lottie；它们通过专用 Tokenizer 解决原始 JSON 过长、结构冗余和训练困难。

Remotion 把 React 组件变成当前帧的函数，HyperFrames 把 HTML、确定性 Seek、Headless Chrome Capture 与 FFmpeg 串成完整链路。它们共同证明了 Web 技术和绝对时间可以可靠地描述视频，也直接影响了我们的方案。区别不在“能不能做动画”，而在交付边界：它们擅长组织并渲染一支视频，我们需要的是已有视频工程中一段可复用、可编辑，并能直接进入 `ImageBitmap` 合成路径的 MG 资源。

Rive 把矢量动画、数据绑定和交互状态机做成完整的设计与跨平台运行时，尤其适合 App、游戏和交互界面。它的状态机通常按每帧 `delta time` 前进，而非把 NLE 的任意绝对时间作为唯一状态输入；二进制资源与专用编辑器也对应另一种创作方式。

Motion Canvas 同样证明了 TypeScript、矢量场景和实时编辑器可以支撑程序化动画，擅长讲解视频与旁白同步。它以 Generator 描述动画流，而我们需要把一个小型、可复用的视觉组件嵌入更大的非线性时间线，并经受乱序 Scrub 和逐帧抽取。

| 方案 | 最擅长的边界 | 对当前视频 MG 的主要取舍 |
|---|---|---|
| Lottie | 标准化矢量交换与跨平台播放 | 声明式、生态成熟；通用模型直接生成和程序化逻辑需要额外表示工作 |
| Remotion | React 驱动的程序化整支视频 | 时间模型接近；完整 Composition 边界大于一个需要直接进入既有 Bitmap 管线的 MG Clip |
| HyperFrames | HTML 驱动、可逐帧 Capture 的视频 | 模型友好且确定性强；浏览器 DOM 到 Bitmap 缺少稳定的生产级桥梁 |
| Rive | 交互动画与状态机 | 设计工具和跨端运行时完整；不是面向文本源与 NLE 随机访问建立的边界 |
| 预渲染视频 | 通用播放与交付 | 像素管线简单；参数编辑、矢量缩放、透明叠加和快速修改受限 |
| 可执行 SVG MG | AI 可读写、浏览器内随机访问和同源输出 | 表达直接；必须治理确定性、性能与可信代码边界 |

近期研究也在从不同方向接近相同问题。MG-Gen 将图片拆为 HTML 图层并生成动画脚本；Decomate 先把 SVG 重组为语义部件，再通过自然语言协同制作动画；Vector Prism 指出，VLM 要可靠动画化 SVG，必须先恢复视觉对象的语义层级。它们提醒我们：让图形“动起来”只是第一步，源表示中的语义结构决定了后续是否真的可编辑、可控制。

<!-- RESEARCH: Build matched visual cases for Lottie Web, Remotion, HyperFrames and .mg. Compare only shared capabilities, and report authoring/generation representation separately from runtime performance. -->

## 八、格式解决确定性，品味仍然来自创作经验

![MG 运行时保证可寻址与可编辑，创作 Skill、视觉评审与真实成片反馈决定动画是否好看](../assets/motion-graphics/runtime-and-taste.svg)

*运行时保证动画按要求抵达每一帧，不能替创作者决定哪一种运动值得出现。*

单一时钟、SVG 组件和同源输出解决的是工程确定性：动画可以随机访问，文字可以修改，预览和导出不会各自解释一遍。它们不会自动带来好的节奏、版式和品牌感。

MG 的质量仍然取决于进入、停留与退出怎样分配时间，文字层级是否清楚，缓动是否符合内容，透明元素在真实视频上是否可读，以及运动有没有帮助观众理解信息。模型很容易生成“每个元素都动一下”的页面，却未必知道什么时候应该克制。

这部分经验可以逐步进入 Skill：标题安全区、长文本压力测试、常用节奏、数据动画、品牌变量、入场与退场检查，都可以变成 Agent 在生成和修改 MG 时遵循的方法。但 Skill 是否真正内化了创作经验，需要靠真实成片、人工返工和可重复的视觉评测验证，不能仅凭一份提示词宣布完成。

运行时也有清楚的边界。当前方案选择受控 SVG 场景，不承载视频、WebGL、3D 和任意外部网页；复杂粒子、流体与依赖历史积分的物理效果需要另一类可重建模型。可信项目代码可以直接运行，公共模板市场则必须增加签名、权限、隔离、CPU 时间与内存配额。

这些限制不是把 MG 缩成一个模板播放器，而是让它在视频编辑器里拥有可靠的物理规律。先确保任意一帧都能被准确找到，才有资格继续扩大表达能力。

## 结语

回到开场那段产品标题。AI 写出源文件后，时间线可以直接把它停在 1.7 秒；创作者换掉文字和颜色，不需要重新生成一段视频；同一动画放到另一个 Clip，只保存新的实例参数；最终导出逐帧询问同一个视觉程序，没有另一套服务端动画等着与预览对齐。

MG 因此不再是编辑器外部制作、渲染完成后导入的一段素材。它是视频工程中的可编辑对象：保留语义，接受时间，随时给出准确画面。

这套 Motion Graphics 运行时已经用于 [StarCut](https://starcut.io) 的标题、标注、数据图表和社交包装。你可以让 AI 直接生成一段 MG，把它放进时间线，再像编辑普通素材一样播放、Scrub、改字与导出。

## 相关资料

1. W3C, [Web Animations](https://www.w3.org/TR/web-animations-1/).
2. W3C, [Scalable Vector Graphics (SVG) 2](https://www.w3.org/TR/SVG2/).
3. Lottie Animation Community, [Lottie Animation Format](https://lottie.github.io/lottie-spec/dev/specs/format/).
4. Remotion, [The fundamentals](https://www.remotion.dev/docs/the-fundamentals) and [`useCurrentFrame()`](https://www.remotion.dev/docs/use-current-frame).
5. HyperFrames, [Introduction](https://hyperframes.heygen.com/introduction) and [HTML-in-Canvas](https://hyperframes.heygen.com/guides/html-in-canvas).
6. Rive, [State Machine Playback](https://rive.app/docs/runtimes/state-machines).
7. Motion Canvas, [Introduction](https://motioncanvas.io/docs/) and [Animation flow](https://motioncanvas.io/docs/flow/).
8. Y. Yang et al., [OmniLottie: Generating Vector Animations via Parameterized Lottie Tokens](https://arxiv.org/abs/2603.02138), CVPR 2026.
9. J. Chen et al., [LottieGPT: Tokenizing Vector Animation for Autoregressive Generation](https://arxiv.org/abs/2604.11792), CVPR 2026.
10. T. Shirakawa et al., [MG-Gen: Single Image to Motion Graphics Generation](https://openaccess.thecvf.com/content/ICCV2025W/GDUG/html/Shirakawa_MG-Gen_Single_Image_to_Motion_Graphics_Generation_ICCVW_2025_paper.html), ICCV Workshops 2025.
11. J. Park et al., [Decomate: Leveraging Generative Models for Co-Creative SVG Animation](https://arxiv.org/abs/2511.06297), 2025.
12. J. Yun and J. Choo, [Vector Prism: Animating Vector Graphics by Stratifying Semantic Structure](https://openaccess.thecvf.com/content/CVPR2026/html/Yun_Vector_Prism_Animating_Vector_Graphics_by_Stratifying_Semantic_Structure_CVPR_2026_paper.html), CVPR 2026.
