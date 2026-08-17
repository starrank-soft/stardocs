# Edit Video as Code：将视频工程映射为 Coding Agent 可读写的虚拟文件视图

![Edit Video as Code 封面：虚拟文件视图与专业时间线连接到同一权威项目状态](../assets/edit-video-as-code/cover.png)

## 引言：AI Coding 真正依赖的不是“代码”两个字

今天的 AI 已经可以在一个大型代码仓库中完成相当复杂的工作。它会先查看目录，搜索符号，打开相关文件，只修改几行代码，再运行编译或测试，根据错误继续修正。

这件事经常被概括成“模型会写代码了”，但模型能力只是其中一半。另一半来自软件工程为 AI 提供了一套非常适合工作的环境：

- 项目有稳定的文件和目录；
- 文件可以按需读取，不必每次装入整个仓库；
- `glob` 和 `grep` 可以帮助模型定位目标；
- 修改可以用局部 patch 表达；
- 编译器、类型系统和测试可以拒绝错误；
- 仓库本身是一份持久的外部记忆；
- 人类可以查看 diff，随时接过来继续工作。

反过来看视频剪辑，很多 AI 系统仍然以界面自动化、细粒度工具集合或完整时间线重写作为主要入口。这些方式各有适用范围；当它们成为唯一的工程接口时，模型往往缺少稳定的项目视图、按需读取能力和局部修改边界。

我们提出 **Edit Video as Code**：不要求视频工程本身变成代码，而是在视频运行态之上建立一套面向 AI 的虚拟文件视图，让 Coding Agent 可以像操作代码仓库一样操作视频项目。

它看到文件，使用文件工具，但修改的仍然是专业视频编辑器中的同一份权威工程。

![一份视频工程面向 Coding Agent 和人类编辑器呈现两种工作界面](../assets/edit-video-as-code/project-views-architecture.svg)

*Agent 使用虚拟文件接口，人类使用时间线界面；两者读写的是同一份权威项目状态。*

## 一、首先要区分三件容易混淆的事

### 1. Edit Video as Code 不是用代码重新生成一个视频

FFmpeg 脚本、Remotion 页面或 Blender 脚本都可以生成视频，但它们通常建立了一套新的渲染程序。生成结果未必仍然是一个能被普通剪辑界面继续编辑的工程。

Edit Video as Code 的目标不同：

- AI 修改的是用户当前打开的项目；
- 修改后，时间线、预览和属性面板立即看到同一结果；
- 用户可以继续通过编辑器原有的交互与撤销机制调整结果；
- AI 与人类不需要在两个工程之间导入、导出或同步。

### 2. 虚拟文件视图不是把数据库复制到临时目录

我们不会把协同状态、项目数据库和对象存储复制成一棵临时文件树，再建立一套双向同步逻辑。

虚拟文件视图是一层**实时投影**：

- `glob` 时，从当前项目、文档和素材资源计算文件列表；
- `read` 时，从当前权威项目状态生成对应的文件内容；
- `edit` 时，把文件式操作反向解析成领域操作；
- 写入仍落到原来的结构状态、素材资源或项目元数据。

文件视图本身不拥有第二份状态。

### 3. “像文件一样操作”不等于“把所有东西都当字符串”

代码仓库中的源文件通常以文本为权威状态，因此 Search/Replace 很自然。视频时间线不同：AI 看到的 VML 是运行态节点图的一种可读投影，不是系统内部保存的一整段 XML 字符串。

因此，文件是发现和读取边界，真正的写入仍然必须尊重节点身份、父子关系、顺序、引用和领域模式。

这也是为什么我们需要 DSL 和语义编辑协议，而不是只提供一个全局 Search/Replace。

## 二、AI 看到的视频项目是什么样的

一个 StarCut 项目可以被映射为：

```text
project.json

assets/
  product-shot.png
  product-demo.mp4
  narration.mp3
  logo.svg
  lower-third.mg

docs/
  brief.md
  script.md
  storyboard.md

compositions/
  main.vml
  intro.vml
```

这棵树专门面向 AI 的工作方式设计，而不是把内部数据库表逐项暴露出去。

### `project.json`：项目级信息

```json
{
  "name": "Product Launch",
  "nameMode": "custom",
  "description": "Thirty-second launch film",
  "coverUrl": "https://cdn.example.com/project-cover.webp",
  "coverMode": "custom"
}
```

它提供项目名称、描述和封面等项目级语义。该文件由系统管理，AI 可以读取，但不能把它当任意 JSON 根节点删除或覆盖；修改需要经过项目元数据自身的领域约束。

### `docs/*.md`：编辑过程中的文字资料

`docs` 可以保存 brief、脚本、分镜、旁白文本或持续更新的制作说明。

这些文档是 Agent 的持久工作材料，但不是另一份时间线模型。精确的 Clip 时间、轨道顺序和特效参数仍然只存在于 Composition 中。

### `assets/*`：项目资源

图片、视频、音频和文档是二进制素材资源（Artifact）；SVG 和 MG 则是可编辑文本资源。

AI 通过路径引用资源：

```text
assets/product-demo.mp4
```

路径负责可发现和可读，内部资源 ID 负责稳定身份，内容哈希可以服务于版本识别和缓存。AI 不需要理解数据库记录或对象存储结构。

### `compositions/*.vml`：视频时间线

Composition 使用 VML，也就是面向视频编辑领域的 XML-based DSL：

```xml
<Composition id="comp_main"
             width="1920"
             height="1080"
             fps="30"
             backgroundColor="#000000">
  <VideoTrack id="track_main" name="Main">
    <VideoClip id="clip_product"
               source="assets/product-demo.mp4"
               start="0"
               duration="5000000"
               sourceStart="0"
               sourceDuration="5000000" />
  </VideoTrack>

  <TextTrack id="track_title" name="Titles">
    <TextClip id="title_launch"
              start="500000"
              duration="2500000"><![CDATA[Product Launch]]></TextClip>
  </TextTrack>
</Composition>
```

AI 看到的是 Composition、Track、Clip 和 Effect，而不是协同存储容器、数据库字段、界面组件或解码器对象。

时间以整数微秒表达，用来减少浮点秒数在编辑和序列化中的漂移；需要落在精确帧边界时，仍应结合帧率或媒体时间基进行换算。

## 三、虚拟文件视图的本质：稳定路径与稳定节点身份

一个视频项目需要两种地址。

### 文件路径负责发现

```text
compositions/main.vml
assets/product-demo.mp4
docs/script.md
```

路径让 AI 可以使用它已经熟悉的目录浏览、模式匹配和文件读取方式。

### 节点 ID 负责精确修改

下面使用便于阅读的 ID 作为示例；生产系统中的 ID 可以是任意系统生成的稳定标识。

```text
compositions/main.vml#clip_product
```

文件内部的 Track、Clip 和 Effect 使用系统生成的稳定 ID。一个 Clip 从第一条轨道移动到第二条轨道之后，它仍然是同一个 Clip，不应因为层级路径变化而失去身份。

因此我们没有让 AI 使用这种地址：

```text
/compositions/0/tracks/3/clips/12
```

数组下标是序列化布局，不是语义身份。一旦其他用户在前面插入一条轨道，整条路径的含义就会变化。

![Clip 移动前后保持同一稳定节点 ID，而数组路径随位置变化](../assets/edit-video-as-code/stable-id-move.svg)

*移动一个 Clip 应当改变归属和顺序，而不是删除旧节点再创建新节点。*

文件路径和节点 ID 的组合同时解决了两个问题：

- AI 可以先像浏览仓库一样发现文件；
- 找到目标以后，可以像使用符号引用一样精确操作节点。

## 四、结构状态不是文件树，而是具有稳定身份的节点图

虚拟视图之所以叫“映射”，是因为底层权威状态并不按照展示出来的 XML 文件树存储。抽象上，它是一张结构化节点图；每个节点保存：

```text
稳定 ID      节点在移动和重排后仍保持同一身份
领域类型     Composition、Track、Clip、Effect 等
领域属性     时间、素材引用、变换、样式等
父级归属     节点属于哪个父节点的哪个成员字段
相对顺序     节点在同一成员集合中的位置
```

子节点列表由父级归属和相对顺序计算，而不是把整棵嵌套数组作为一个整体反复覆盖。这种模型同时服务三个要求：

1. **运行态稳定身份**：移动节点不需要删除再创建；
2. **协同编辑**：一次局部修改只触及对应节点的属性、归属或顺序；
3. **统一领域验证**：父节点的领域模式决定可以接受哪些子节点和属性。

读取 VML 时，正向投影把节点图恢复为树形结构，将内部属性转换为公共 VML 属性，并把内部资源引用转换成 AI 可以理解的项目路径。写入时，反向投影把标签、属性、父节点和相对位置解析为领域操作，验证后修改同一份结构状态。

StarCut 使用 Yjs 的 Y.Doc 承载协同节点状态，并以扁平节点、父级归属和相对顺序表达结构。这是状态模型的一种实现方式，不属于 Agent 接口本身。

![读取时将权威节点图投影为 VML，写入时把 VML 编辑解析为经过验证的语义操作](../assets/edit-video-as-code/projection-roundtrip.svg)

*VML 是节点图的双向投影：读路径生成可理解的树，写路径修改原节点的属性、归属或顺序。*

虚拟文件视图和人类编辑界面是权威项目状态面向不同使用者的投影，不形成需要事后合并的项目副本。

## 五、DSL 设计：让 AI 看到领域，而不是存储结构

### 1. 为什么使用 VML

视频时间线天然具有树形语义：Composition 包含 Track，Track 包含 Clip，部分 Clip 包含 Effect。XML 对这类结构有几个现实优势：

- 层级关系直接可见；
- 创建一个子树时不需要反复声明父子 ID；
- 属性形式紧凑；
- 文本和结构可以共存；
- 大模型对 XML/HTML 结构已经非常熟悉；
- 完整元素具有清晰的流式完成边界。

但 VML 不是为了映射任意 XML。每个标签、属性和父子关系都来自领域模式（schema）。

例如：

- `Composition` 可以包含 `VideoTrack`、`AudioTrack`、`TextTrack` 和 `EffectTrack`；
- `VideoTrack` 可以包含 `VideoClip`、`ImageClip` 和 `MotionGraphicClip`；
- `AudioTrack` 不能包含 `ImageClip`；
- `source` 必须引用项目中已经存在且类型兼容的 Artifact；
- 同一轨道中的 Clip 不能违反时间线规则；
- 未注册的 Effect tag 或属性会被拒绝。

DSL 提供的是一个面向领域、可验证的公共表面，而不是内部运行时对象的文本转储。

### 2. 读取时显示 ID，创建时不允许模型生成 ID

读取已有节点时必须显示 ID，因为 AI 需要精确定位：

```xml
<VideoClip id="clip_product" ... />
```

创建新节点时必须省略 ID：

```xml
<VideoClip source="assets/new-shot.mp4"
           start="5000000"
           duration="3000000" />
```

ID 由系统生成并在结果中返回。这样可以避免模型猜测、复用或制造重复 ID。

### 3. DSL 负责表达，领域模式负责判定

不能因为模型输出了看起来合理的 XML 就直接写入运行态。公共 VML 需要先被解析为候选领域操作，再依次验证节点模式、父子关系和素材引用；只有全部通过，才能修改权威结构状态。

![公共 VML 经过结构解析和领域验证后进入权威结构状态](../assets/edit-video-as-code/vml-validation-pipeline.svg)

*语法解决“能否理解”，领域验证决定“能否执行”；单次操作验证失败不会修改权威状态。*

这相当于 AI Coding 中解析器、类型检查器与编译边界的组合。

### 4. 状态 DSL 与编辑协议是两层设计

状态表示和状态修改承担不同职责：

- **VML 是状态 DSL**：它回答“这个视频工程现在是什么样”；
- **RESTful-like Edit Ops 是编辑协议**：它回答“要对现有状态做什么修改”。

创建节点时，编辑协议可以复用一段 VML 作为新子树的载荷，但这不意味着两者是同一种语言。前者负责可读的领域表示，后者负责可定位、可执行的状态变化。分开以后，读取格式可以保持完整和稳定，修改格式则可以保持局部和轻量。

## 六、为什么结构化时间线不能只用 Search/Replace

VML 是权威节点图的可读投影，不是系统保存的一整段 XML。Search/Replace 能准确描述“哪段字符变成哪段字符”，却没有直接表达这次变化究竟是属性修改、节点移动、创建还是删除。

例如，AI 读取到的是一段树形结构：

```xml
<VideoTrack id="track_main">
  <VideoClip id="clip_a" />
  <VideoClip id="clip_b" />
</VideoTrack>
```

但系统保存的是具有稳定身份、归属和顺序的节点。把 Clip 从一条 Track 移到另一条 Track，在文本中像一次删除和插入，在节点模型中却是同一个对象改变了位置。

系统当然可以比较修改前后的 VML，匹配节点身份，推导结构差异，再执行领域验证。但这实际上已经构成一层语义编辑器。与其让系统从文本差异猜测意图，不如让 Agent 直接给出稳定目标和操作语义。

差别尤其体现在三类结构变化上：

- **身份**：局部属性修改不应触发后代节点重建，已有引用和运行态身份需要继续成立；
- **移动**：Clip 从一条 Track 移到另一条 Track，改变的是归属和顺序，而不是“删除一段 XML，再插入一段相似 XML”；
- **写入范围**：全文替换依赖更大的旧文本匹配范围，语义操作只触及目标节点的属性、归属或顺序。底层协同机制仍负责合并，协议的价值是让写集合更接近用户意图。

Search/Replace 仍然适用于 Markdown、SVG、MG 等以文本为权威状态的资源，也适用于明确节点边界内的文本内容。边界可以概括为：

> 文本源使用文本编辑；结构化投影使用语义结构编辑；结构节点内部的文本属性可以在节点边界内使用精确文本编辑。

例如，标题节点内部的文字可以继续使用精确替换，因为目标身份和文本边界都已经确定：

```text
PATCH #title_launch
  <<<<<<< SEARCH
Product Launch
  =======
Summer Launch
  >>>>>>> REPLACE
```

![完整文档替换与语义节点编辑的写集合对比](../assets/edit-video-as-code/semantic-edit-vs-rewrite.svg)

*两种方案都能修改文本；差异在于系统是否直接获得目标身份和操作意图。*

## 七、RESTful-like Ops：像修改资源一样修改视频节点

VML 描述当前状态，RESTful-like Ops 描述针对稳定资源的局部变化。`RESTful-like` 指“以资源为中心、用短动词表达意图”的设计直觉；它既不要求符合 HTTP REST，也不是 JSON Patch 的变体。

### PATCH：修改属性或节点文本

```text
PATCH #clip_product
start="1000000"
duration="4000000"
opacity="0.8"
```

目标是稳定节点 ID，属性行提供清楚的增量解析边界。

### POST：在目标父节点下创建子树

```text
POST #track_main
before: #clip_outro
<VideoClip source="assets/product-demo.mp4"
           start="5000000"
           duration="3000000"
           sourceStart="0"
           sourceDuration="3000000" />
```

POST 载荷直接复用 VML。模型一次就能表达节点类型、属性和嵌套子树，不需要连续调用 `create_clip`、`set_source`、`set_start` 和 `set_duration`。

### PUT：移动或重排已有节点

```text
PUT #clip_product
to: #track_broll
before: #clip_outro
```

PUT 表达对已有资源的幂等放置：将节点放到指定父节点和相对位置。重复执行同一操作应得到相同位置，而不会创建新的节点身份。

### DELETE：删除节点

```text
DELETE #effect_vignette
```

删除仍需经过引用、必需结构和父节点领域模式验证。

## 八、为什么这套协议适合模型流式输出

大模型从左到右生成 token。若修改格式必须等待一个庞大的深层对象完全闭合，系统只能在整次输出结束后理解模型的意图。RESTful-like Ops 把一次编辑拆成若干短小、可独立识别的语义操作，使解析器能够更早确定目标、动作和载荷边界。

这种结构具有几个适合流式生成的特点：

- 文件路径先确定资源类型和解析方式；
- 操作头先给出动作与稳定目标；
- 每个完整操作都可以独立验证；
- 多个操作可以按生成顺序形成一条编辑序列。

工具的外层仍然是标准结构：

```json
{
  "path": "compositions/main.vml",
  "changes": "PATCH #clip_product\nopacity=\"0.8\""
}
```

流式可解析不等于必须逐字写入运行态。执行层可以在每个完整操作通过验证后渐进应用，也可以暂存整次调用并在最后统一提交。两种策略共享同一套协议，只是在延迟、原子性和错误恢复之间采用不同权衡。

因此，协议负责提供清晰的语义边界；事务、撤销、重试和回滚由具体执行环境定义。流式能力的价值是缩短“模型开始输出”到“系统理解第一个完整修改”之间的距离，而不是绕过领域验证。

![RESTful-like 编辑协议在模型输出过程中逐步形成可验证的语义边界](../assets/edit-video-as-code/streaming-edit-flow.svg)

*短操作让语义边界更早出现；是否立即提交，由执行策略决定。*

## 九、AI 使用的基本工具

Edit Video as Code 刻意把基础入口收敛到 Coding Agent 熟悉的几类，但“工具名像文件工具”不意味着底层按普通磁盘文件执行。

| 工具 | 作用 | 关键边界 |
|---|---|---|
| `glob` | 按模式发现 Composition、文档和素材路径 | 返回真实存在的项目资源，Agent 不应猜路径 |
| `grep` | 查找文本、标签、属性和素材引用 | 结构化结果同时返回文件路径与节点 ID |
| `read` | 读取文件或 `path#id` 指向的 VML 子树 | 局部任务不必读取整条时间线 |
| `head` | 读取素材类型、时长、尺寸、哈希和状态 | 语义接近 HTTP `HEAD`，不是读取文本文件的前几行 |
| `write` | 从零创建完整 Composition 或文本资源 | 已有 VML 的完整覆盖会扩大写入范围，不是局部编辑的默认方式 |
| `edit` | 修改已有文件 | VML 使用语义操作，真实文本源使用精确文本替换 |

因此，Agent 既可以读取完整 Composition，也可以只展开一个节点子树：

```text
read("compositions/main.vml")
read("compositions/main.vml#clip_product")
head("assets/product-demo.mp4")
```

`edit` 是统一的文件式入口，但不同文件类型拥有不同写入语义：

- VML 使用 `PATCH`、`POST`、`PUT`、`DELETE`；
- Markdown、SVG、MG 使用精确 Search/Replace；
- TextClip 和 CaptionClip 可以在明确的节点边界内修改文本数据。

搜索能力应声明自己的索引范围。能够按路径读取一种资源，并不意味着它必然进入全项目全文索引。

项目元数据等系统管理资源可以保留专用写入口。文件式工作流统一的是发现、读取和局部修改的方式，而不是消除所有领域边界。

## 十、一次完整剪辑如何形成闭环

假设用户要求“找到产品演示镜头，把它缩短到三秒，并在后面增加一张产品海报”。Agent 不必先读取整个工程，而是完成四个阶段：

1. **发现**：浏览 Composition 与素材资源，并读取必要的媒体元数据；
2. **定位**：搜索素材引用，只展开目标 Clip 附近的时间线结构；
3. **修改**：在一个逻辑编辑中缩短已有节点，并向目标 Track 插入新的图片节点；
4. **验证**：先检查结构、引用和时间约束；如果任务涉及构图、节奏或审美，再观看局部预览。

对应的文件式操作可以保持紧凑：

```text
glob  compositions/**/*.vml
head  assets/product-demo.mp4
grep  assets/product-demo.mp4 in compositions/**/*.vml
read  compositions/main.vml#track_main

edit  compositions/main.vml
  PATCH #clip_product        缩短已有镜头
  POST  #track_main          插入产品海报
```

这个闭环复用了 Coding Agent 已经熟悉的工作方式：先发现，再读取必要上下文，以局部修改表达意图，并选择与任务相匹配的验证手段。关键不是命令长什么样，而是模型不需要把完整工程搬进上下文，也不需要用一次全量重写覆盖当前状态。

## 十一、写入时序：一次 Agent 修改如何被人类立即接管

一次 Agent 修改从文件式请求开始，经过资源类型路由、领域解析和验证，最终进入项目原有的状态更新链路。

![Agent 的文件式修改经过类型路由和领域验证进入权威项目状态，人类编辑器立即观察到同一结果](../assets/edit-video-as-code/shared-project-write-sequence.svg)

*人类看到更新，不是因为系统把一份“AI 工程”同步回来了，而是因为双方始终读写同一份权威状态。*

读取操作从当前项目状态生成视图，写入操作进入项目既有的编辑入口；结构节点、素材资源和项目元数据即使由不同系统承载，也共同组成一份权威项目状态。修改完成后，人类界面沿原有的观察链路更新，后续手工调整继续作用于同一节点和同一工程。协同、事务与撤销由项目状态和执行环境提供，虚拟文件接口只负责把 Agent 的意图准确送入这条链路。

## 十二、为什么不把每个剪辑动作都做成一个工具

如果每个领域动作都是顶层工具，加入一种新 Clip 或 Effect 时，通常还要增加新的创建、修改、查询工具和参数描述。

工具数量会随领域能力一起增长：

```text
create_video_clip
create_image_clip
create_motion_graphic_clip
set_clip_timing
set_clip_transform
set_clip_opacity
move_clip
add_vignette_effect
...
```

专用工具具有明确的参数模式，适合生成、转写、播放控制和外部服务等边界清楚的能力。

但对于已经进入项目模型的结构编辑，更收敛的方式是：

- 顶层工具保持为文件发现、读取和编辑；
- 新的节点类型和属性由 DSL 的领域模式扩展；
- `edit` 的公共操作保持稳定；
- AI 通过技能或领域规范学习新 tag 和 attribute。

这样，系统扩展的是领域语言，而不是不断膨胀顶层工具面。

## 十三、Edit Video as Code 的核心价值

这套方法的价值不在于把视频伪装成文本，而在于建立一组适合 Agent 工作、又不破坏专业运行态的工程性质：

- 项目成为可持续发现和读取的外部记忆，不必完整复制进每一轮 Prompt；
- 读取成本随任务范围增长，局部任务只需展开相关文件或节点子树；
- 修改范围贴近用户意图，属性、归属、顺序和新子树各自保持清楚边界；
- 文件发现、搜索、局部编辑、错误反馈和验证闭环可以复用 Coding Agent 已经形成的能力。

[SWE-agent](https://arxiv.org/abs/2405.15793) 的研究已经表明，Agent-Computer Interface 的设计会显著影响模型完成软件工程任务的能力。Edit Video as Code 把同一问题带到具有连续时间、媒体引用、视觉验收和协同状态的视频编辑领域。

## 十四、AI Cut 与 Human Cut：从“能剪”到节奏、美感和网感

**可编辑不等于会剪辑。**

Edit Video as Code 让 AI 能够定位一个 Clip，把入点移动 12 帧，或者把一组镜头按计划排到时间线上。它解决的是动作接口和工程状态问题。但“为什么应该在这一帧切”“这个镜头再多留半秒是否更有力量”，并不会因为操作变得精确而自然得到答案。

这和人们面对 AI Coding 时的担心具有相同结构：AI 能生成可以运行的代码，不代表代码容易理解、经得住修改或符合整个系统的设计；AI 能排出结构合法的时间线，也不代表成片有节奏、有美感，或者符合特定平台和受众正在使用的表达方式。两者都在经历同一个问题转移：当“能不能操作”逐渐解决以后，真正的焦点会变成“产出的质量是否值得采用，以及人类是否愿意接管”。

但两者并不完全等价。代码质量可以借助类型检查、测试、静态分析、性能指标和 Code Review，把相当一部分判断外化为可重复验证的标准；剪辑质量更多发生在连续时间中，同时受内容、受众、类型、情绪和传播环境影响。它可以被分析和学习，却很难压缩成一个像“测试通过”那样稳定的单一信号。

当前 AI Cut 与有经验的 Human Cut，主要差在对连续时间的观察和判断方式：

| 维度 | 当前 AI Cut 常用的依据 | 有经验的剪辑师常用的判断 |
|---|---|---|
| 工程理解 | 读取时间码、字幕、标签、素材元数据和结构关系 | 同时理解工程结构、画面内容与创作意图 |
| 剪点选择 | 根据语义边界、规则、镜头检测结果或离散采样帧选点 | 感知动作的预备、发生与收势，在连续运动相位中选点 |
| 节奏控制 | 依据时长、语速、节拍点和模板调整 | 判断停顿、呼吸、期待、释放以及段落之间的能量变化 |
| 美感一致性 | 执行显式风格、构图和色彩约束 | 判断局部选择是否共同服务于完整作品的气质和重点 |
| 网感与受众 | 依据平台规则、参考案例和显式偏好 | 结合平台语境、内容类型和目标人群判断何时遵循或打破惯例 |
| 连贯性 | 检查素材引用、时间关系和显式约束 | 同时判断视线、构图、运动方向、声音和有意或无意的跳跃 |
| 结果评价 | 判断是否完成指令、结构是否合法 | 判断一个合法版本是否自然、准确、有力，是否服务整段叙事 |
| 迭代方式 | 修改结构后读取状态，按需查看少量画面 | 反复播放、拖动、逐帧比较，在上下文中微调几个帧 |

![三个结构上都合法的剪点落在动作的不同相位，只有一个更符合连续运动](../assets/edit-video-as-code/ai-human-cut.svg)

*DSL 能准确执行剪点；主动观看、候选比较和偏好判断决定剪点是否合适。*

剪辑中的“感受”并非不可研究的直觉。剪辑师会综合主体运动、镜头变化、姿态与视线、语音重音、音乐节拍以及前后镜头建立的预期。经验把这些连续信号压缩成快速判断，因此很多问题会先以“感觉不对”的形式出现。

节奏感、美感和网感都建立在这种综合判断之上。其中“网感”并不是追逐热门模板的固定分数，而是创作者对平台语境、观众预期和内容类型形成的动态先验；它会随环境变化，也可能被更准确的反常规选择推翻。

已有研究也说明，剪点并非完全没有统计规律。[Learning Where to Cut from Edited Videos](https://openaccess.thecvf.com/content/ICCV2021W/CVEU/html/Huang_Learning_Where_To_Cut_From_Edited_Videos_ICCVW_2021_paper.html) 通过用户研究验证了人们对部分好坏剪点存在共识，并学习未剪素材中的合适剪切区域；[Learning to Cut by Watching Movies](https://openaccess.thecvf.com/content/ICCV2021/html/Pardo_Learning_To_Cut_by_Watching_Movies_ICCV_2021_paper.html) 则从专业成片中学习跨镜头的视听模式，对剪点合理性进行排序。[VEU-Bench](https://openaccess.thecvf.com/content/CVPR2025/html/Li_VEU-Bench_Towards_Comprehensive_Understanding_of_Video_Editing_CVPR_2025_paper.html) 进一步把视频编辑理解拆成识别、推理和判断任务，并显示当前视频语言模型在这些任务上仍面临明显困难。这些工作表明“剪在哪里”可以学习，但也揭示了它不同于结构合法性检查：它需要连续视听证据、候选比较和人的偏好评价。

结构编辑协议不等于完整的剪辑智能。Edit Video as Code 解决的是工程接口：状态如何被发现、寻址和修改；下一步的核心问题是感知与判断，即 Agent 如何观看连续时间，以及如何从多个合法方案中选出更符合运动、节奏、情绪和叙事目标的一个。

Agent Skill（技能包）可以把显性的剪辑经验外化为可复用流程，例如观察顺序、候选生成、连续性检查、平台约束和评价标准。它并不等同于把审美和节奏感写入模型参数，也不能替代连续视听感知。Skill 能否稳定提升成片质量，需要通过不同题材、平台和目标受众下的实际任务验证。

进一步的能力来自感知—编辑—评价闭环：

![Agent 通过主动观察、时间对齐表征、候选生成、局部比较和语义写回形成感知编辑闭环](../assets/edit-video-as-code/perception-edit-loop.svg)

*结构协议提供可执行的“手”；连续播放与比较提供“眼睛和耳朵”；评价与反馈让判断逐轮改进。*

AI Cut 的突破不只是把时间线排得更完整，而是围绕候选剪点主动播放、seek、scrub 和比较局部结果，再根据运动、声音、语义证据与人的偏好修正选择。最终成片只保留了选择结果，而“看过什么、比较过什么、为何移动几帧”这类过程数据，可能成为学习剪辑判断更有价值的信号。

## 结语：不是把视频变成代码，而是给 AI 一套像代码一样可工作的界面

视频工程不会因为 AI 的出现就变成一堆普通文本。它仍然包含连续时间、媒体引用、渲染状态、协同关系和专业领域约束。

但 AI 不应该被迫通过像素坐标理解这一切，也不应该每次重新输出完整工程。

Edit Video as Code 的关键，是在专业运行态和 Coding Agent 之间建立一层准确的映射：

```text
视频运行态
  -> 虚拟项目文件
  -> 文件式发现和读取
  -> DSL 语义表达
  -> 局部结构操作
  -> 领域模式验证
  -> 回到同一视频运行态
```

AI 看到的是文件，但文件不是另一份数据；AI 输出的是文本，但执行的不是盲目字符串替换；AI 使用的是 Coding 工作流，但修改的仍然是真实、可继续编辑的视频工程。

当项目可发现、状态可寻址、修改可局部化、错误可验证、人类可接管时，AI 才不只是“会调用几个剪辑工具”，而是获得了像 Coding 一样持续操作视频工程的工作界面。它没有替代剪辑判断，而是为 AI 与人类共同编辑同一工程建立了稳定的基础。

## 相关资料

1. J. Yang et al., [SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering](https://arxiv.org/abs/2405.15793), 2024.
2. B. Wang et al., [LAVE: LLM-Powered Agent Assistance and Language Augmentation for Video Editing](https://arxiv.org/abs/2402.10294), IUI 2024.
3. Z. Cao et al., [AgenticVBench: Can AI Agents Complete Real-World Post-Production Tasks?](https://arxiv.org/abs/2605.27705), 2026.
4. P. Bryan and M. Nottingham, [RFC 6902: JSON Patch](https://www.rfc-editor.org/rfc/rfc6902), 2013.
5. Y. Huang et al., [Learning Where to Cut from Edited Videos](https://openaccess.thecvf.com/content/ICCV2021W/CVEU/html/Huang_Learning_Where_To_Cut_From_Edited_Videos_ICCVW_2021_paper.html), ICCV Workshops 2021.
6. A. Pardo et al., [Learning to Cut by Watching Movies](https://openaccess.thecvf.com/content/ICCV2021/html/Pardo_Learning_To_Cut_by_Watching_Movies_ICCV_2021_paper.html), ICCV 2021.
7. B. Li et al., [VEU-Bench: Towards Comprehensive Understanding of Video Editing](https://openaccess.thecvf.com/content/CVPR2025/html/Li_VEU-Bench_Towards_Comprehensive_Understanding_of_Video_Editing_CVPR_2025_paper.html), CVPR 2025.
