---
name: graphic-overlays
description: 通过在播放的视频上叠加定时设计的图形覆盖卡片（标题、字幕条、数据标注、引言、侧边面板、画中画）来包装现有的谈话式/访谈/播客视频——这些卡片与文字记录同步。源视频完整播放；代理在与用户对话中设计和编写每个卡片的HTML，然后通过超帧渲染为MP4。当用户要求在视频上添加图形覆盖、屏幕图形/字幕条/数据标注/动态标题、“包装/美化我的视频”、“添加覆盖卡片/图形卡片”，或对现有视频进行AI创作的图形包装时使用。不适用于普通字幕（→嵌入式字幕）或从头开始构建视频（→创作工作流程）；在不确定是覆盖层还是字幕时，请先阅读/hyperframes-read-first。
---

# 图形覆盖

图形覆盖功能接受一个**完整播放**的本地视频，并在其上叠加一系列定时设计的**图形卡片**——标题、字幕条、数据标注、引言、侧边面板、画中画——与正在说的内容同步。代理设计卡片（定时+内容）并**直接在对话中编写每个卡片的HTML**，然后组装一个组合HTML，并通过`hyperframes`渲染为MP4。没有固定的原型列表，也没有规定的卡片结构——覆盖层是根据文字记录的实际内容生成的。

> **在构建之前确认路线。** 该技能使用**设计的图形卡片**（标题、字幕条、数据标注、引言、侧边面板、画中画）包装一个**现有的谈话式视频片段**。如果用户想要**普通字幕/字幕**（说话内容的文本）→ `/embedded-captions`；一个**单一的简短无叙述**元素（一个标志动画/字幕条）→ `/motion-graphics`。**视频片段保持不变**——重新定时、重新着色、重新构图、重新排序或音频是NLE编辑，**不在范围内**。通过URL/主题/新闻稿构建→创作工作流程。不确定是覆盖层还是字幕？**先阅读`/hyperframes-read-first`。**

> **`embedded-captions`的图形包装兄弟。** 字幕添加了_说话内容_作为可读的字幕；这个功能在播放的视频上添加了_设计的图形_。普通字幕→ `embedded-captions`。从头开始构建视频→创作工作流程（`product-launch-video` / `faceless-explainer` / …）。

工作目录中可检查的中间文件：

- `metadata.json` — 时长/宽度/高度/帧率
- `audio.mp3` — 提取的音频
- `transcript.json` — 一个扁平的**单词数组** `[{ text, start, end }, …]`（Whisper；没有`segments`，没有`words`包装）
- `storyboard.json` — 轻量级卡片大纲（代理的计划）
- `public/cards/card-XX.html` — 每个卡片的单个HTML片段
- `public/index.html` — 最终组装的组合
- `output.mp4` — 渲染的视频

## 命令行界面解析```bash
# hyperframes — transcription (local Whisper) + rendering the assembled HTML to MP4
npx hyperframes --help
```
--- 
name: `ffmpeg`
description: 本技能完全在 **hyperframes** CLI 以及系统 `ffmpeg` / `ffprobe` 上运行。转录是通过本地 **Whisper** 使用 `hyperframes transcribe` 完成的 —— 无需第三方服务、API 密钥或受限的代理。
---

## 工作流程

### 1. 检查环境```bash
npx hyperframes doctor          # ffmpeg, headless browser, render deps
# confirm bundled assets:
ls "<SKILL_DIR>/assets/fonts" "<SKILL_DIR>/assets/vendor/gsap.min.js"
```
--- 

name: `ffmpeg`
description: 所需内容：
- `ffmpeg` / `ffprobe` (系统)
- `<SKILL_DIR>/assets/fonts/*.woff2`, `<SKILL_DIR>/assets/vendor/gsap.min.js` (捆绑在此技能中，在第9步中暂存到工作目录)

转录不需要密钥 — `hyperframes transcribe` 在本地运行 Whisper (第4步)。

强烈推荐在 macOS 上使用 `hyperframes render`：```bash
export PRODUCER_BROWSER_GPU_MODE=hardware
```
--- 

### 2. 创建工作目录

所有工件都位于 `videos/<project-name>/` 下 —— 与其他视频工作流程（`product-launch-video` / `faceless-explainer` / `pr-to-video`）的约定相同。将当前工作目录保持在工作区根目录；以下所有内容都写入这个子目录。```bash
VIDEO_PATH="/absolute/path/input.mp4"
WORK_DIR="videos/$(basename "$VIDEO_PATH" | sed 's/\.[^.]*$//')"
mkdir -p "$WORK_DIR"
```
--- 
name: Extract Audio and Metadata
description: 提取音频和元数据
---

### 3. 提取音频和元数据

在本节中，我们将介绍如何从视频文件中提取音频轨道和相关的元数据。

#### 3.1 提取音频轨道

要从视频文件中提取音频轨道，可以使用以下命令：

```sh
ffmpeg -i ⟦VIDEO_FILE_PATH⟧ -vn -acodec ⟦AUDIO_CODEC⟧ ⟦OUTPUT_AUDIO_PATH⟧
```

- **⟦VIDEO_FILE_PATH⟧**: 输入视频文件的路径。
- **⟦AUDIO_CODEC⟧**: 目标音频编解码器（例如 `libmp3lame` 用于 MP3，`aac` 用于 AAC）。
- **⟦OUTPUT_AUDIO_PATH⟧**: 输出音频文件的路径。

#### 3.2 提取元数据

提取视频文件的元数据可以使用 `ffprobe` 工具。以下是一个示例命令：

```sh
ffprobe -v quiet -print_format json -show_format -show_streams ⟦VIDEO_FILE_PATH⟧ > ⟦OUTPUT_METADATA_PATH⟧
```

- **⟦VIDEO_FILE_PATH⟧**: 输入视频文件的路径。
- **⟦OUTPUT_METADATA_PATH⟧**: 输出元数据文件的路径。

##### 3.2.1 示例 JSON 输出

提取的元数据将以 JSON 格式保存。以下是一个简化的示例：

```json
{
  "format": {
    "filename": "⟦VIDEO_FILE_NAME⟧",
    "format_name": "⟦FORMAT_NAME⟧",
    "duration": "⟦DURATION_SECONDS⟧",
    "size": "⟦FILE_SIZE⟧"
  },
  "streams": [
    {
      "index": 0,
      "codec_name": "⟦VIDEO_CODEC⟧",
      "codec_type": "video",
      "width": "⟦VIDEO_WIDTH⟧",
      "height": "⟦VIDEO_HEIGHT⟧"
    },
    {
      "index": 1,
      "codec_name": "⟦AUDIO_CODEC⟧",
      "codec_type": "audio",
      "channels": "⟦CHANNEL_COUNT⟧",
      "sample_rate": "⟦SAMPLE_RATE⟧"
    }
  ]
}
```

- **⟦VIDEO_FILE_NAME⟧**: 视频文件名。
- **⟦FORMAT_NAME⟧**: 视频格式名称。
- **⟦DURATION_SECONDS⟧**: 视频时长（秒）。
- **⟦FILE_SIZE⟧**: 视频文件大小。
- **⟦VIDEO_CODEC⟧**: 视频编解码器名称。
- **⟦VIDEO_WIDTH⟧**: 视频宽度。
- **⟦VIDEO_HEIGHT⟧**: 视频高度。
- **⟦AUDIO_CODEC⟧**: 音频编解码器名称。
- **⟦CHANNEL_COUNT⟧**: 音频通道数。
- **⟦SAMPLE_RATE⟧**: 音频采样率。

#### 3.3 处理提取的音频和元数据

提取的音频和元数据可用于后续处理，例如音频分析、元数据驱动的视频处理等。确保在处理过程中考虑到音频和视频的同步问题。

##### 3.3.1 音频分析

可以使用音频分析工具对提取的音频进行分析，例如检测音量峰值、频率分析等。

##### 3.3.2 元数据驱动的视频处理

根据提取的元数据，可以实现自动化的视频处理流程，例如根据视频时长调整处理参数、根据分辨率选择不同的处理算法等。

---

通过以上步骤，您可以成功提取视频文件中的音频轨道和元数据，并为后续的处理和分析做好准备。```bash
# metadata — duration / width / height / fps
ffprobe -v error -select_streams v:0 \
  -show_entries stream=width,height,r_frame_rate \
  -show_entries format=duration -of json "$VIDEO_PATH" > "$WORK_DIR/metadata.json"
# audio
ffmpeg -y -i "$VIDEO_PATH" -vn -acodec libmp3lame -q:a 2 "$WORK_DIR/audio.mp3"
```
--- 
name: `metadata.json`
description: 输出：`metadata.json`（读取 `width`/`height`/`duration`；fps 表示评估的 `r_frame_rate` 分数，例如 `30000/1001 → 29.97`）加上 `audio.mp3`。
---

### 4. 转录```bash
npx hyperframes transcribe "$WORK_DIR/audio.mp3" -d "$WORK_DIR" --json --model small.en
```
--- 待翻译片段 7 ---

**本地 Whisper** — 无需 API 密钥、无需代理、无需速率限制。将一个逐词级别的 `transcript.json` 写入工作目录（单词 `text` + `start` / `end` 时间戳）。
阅读它以获取驱动 Step 6 中卡片时序的单词/句子时间；如果你需要分段级别的块，请在标点符号/停顿处自行将单词分组为句子。

**限制为媒体时长。** Whisper 可能会返回最终单词的 `end` 略超出实际剪辑长度 — 将每个卡片的 `endSec` 和 `composition.durationSeconds` 限制为 `metadata.json` 时长，否则渲染将在视频末尾显示黑色尾巴。

### 5. 修正文本

`transcript.json` 是一个 **单词对象的扁平数组** — `[{ "text": "...", "start": s, "end": s }, …]`（没有 `segments` 数组，没有 `words` 包装器；每个单词的键是 **`text`**）。阅读它并修正明显的 ASR 错误：

- 同音词、产品名称、技术术语、标点符号
- 就地编辑单词的 `text`；**保留其 `start` / `end`** 时间戳
- 没有预分组的 `segments` 数组 — **自行将单词分组为句子**（在终端标点/停顿处分隔），当你需要分段级别的块用于卡片时序时

### 6. 草拟一个轻量级故事板（在聊天中）

**不涉及 CLI。** 阅读 `transcript.json` + `metadata.json` 并直接设计卡片。`storyboard.json` 是一个代理内部的规划工件 — 没有 CLI 命令会使用它；它的存在是为了让你在编写每个卡片的 HTML 之前可以清晰地思考时间和内容。保持与下方示例一致的形状，以便相同的轮廓可以驱动你在第 9 步中创作的内容：```json
{
  "schemaVersion": 3,
  "composition": {
    "fps": 30,
    "width": 1080,
    "height": 1920,
    "durationSeconds": 121.2,
    "layout": "portrait",
    "themeId": "noir",
    "seed": 42
  },
  "videoTrack": {
    "sourcePath": "input-video.mp4",
    "startSec": 0,
    "endSec": 121.2,
    "bounds": { "x": 0, "y": 0, "width": 1080, "height": 1920 }
  },
  "subtitles": { "enabled": false },
  "cards": [
    {
      "id": "card-01",
      "intent": "Hook with the speaker's anxious midnight question",
      "startSec": 0.5,
      "endSec": 13.0,
      "accentIndex": 0,
      "zone": "fullscreen",
      "contentHints": {
        "kicker": "AN HONEST QUESTION",
        "title": "The soul-searching question at 11 PM",
        "detail": "Client's 60-second voice message: 'If the RMB appreciates, does that mean my USD policy is a terrible loss?'"
      }
    }
  ]
}
```
--- 

**现有卡片字段：**

| 字段                   | 类型                                       | 用途                                                                                                   |
| ----------------------- | ------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| `id`                    | 字符串                                     | 用于卡片 HTML 和 GSAP 选择器的稳定 ID                                                                 |
| `intent`                | 字符串                                     | 自然语言描述；用于卡片合成                                                                           |
| `startSec` / `endSec`   | 数字                                     | 时间（秒）（endSec > startSec）                                                                      |
| `accentIndex`           | 0 \| 1 \| 2 \| 3 \| 4                      | 该卡片使用的 5 种主题强调色中的哪一种                                                               |
| `zone`                  | 枚举（见下文）                             | 卡片在画布上的位置                                                                                     |
| `contentHints`          | 对象                                       | 自由格式的数据包；代理将 kicker/title/detail/data/quote 放在这里                                                             |
| `archetype`（可选）  | 字符串                                     | 可附加的自由格式标签，用于记住卡片的模式；缺失 = 自由格式，这是默认值                                 |
| `transition`（可选） | 枚举：`cut` \| `fade` \| `slide` \| `wipe` | 声明性的卡片到卡片过渡                                                                                 |

**五种 `zone` 值：**

| 区域              | 解析后的边界                                | 何时使用                             |
| ----------------- | ---------------------------------------------- | --------------------------------------- |
| `fullscreen`      | 覆盖整个画布                            | 英雄时刻，重要数字，格言      |
| `whiteboard-area` | 内嵌 40px 边距（或纵向高度 45%）  | 密集数据 / 注释内容          |
| `lower-third`     | 底部 30% 区域                                | 可见视频上的注释           |
| `side-panel`      | 右侧 42%（横向）或底部 40%（纵向） | 数据侧，视频另一侧             |
| `video-overlay`   | 整个画布，期望卡片大部分透明   | 全出血视频上的注释叠加 |

在第 9 步组装构图时，按照上表将每个卡片的 `zone` 解析为卡片宿主包装器上的像素边界。视频边界在构图级别设置 **一次**（`videoTrack.bounds`）；为了使视频看起来像是在“卡片之间移动”，请在构图的 `<script>` 中针对 `#video-wrap` 使用 GSAP 补间（参见第 9 步）。

**没有规定的卡片角色，没有规定的叙述弧线。** 卡片是根据视频实际所说的内容产生的——可能是全部引用或全部数据，可能是以数字或故事开头。让抄本驱动节奏。

**多少要点？——从时长 + 密度自动推断。** 没有固定的上限。从视频时长中选择一个 **基础节奏**，然后根据 **信息密度** 调整。只有 **下限是固定的：至少 5 张卡片**，这样即使是短视频也有节奏。

**第 1 步——按时长的基础节奏**（中等密度的自然 sec/card）：

| 视频时长     | 基础节奏（秒/卡片） | 理由                                   |
| ------------------ | ------------------------ | ------------------------------------------- |
| < 60 秒（短视频） | **6–8 秒**                 | 观众期望短视频中有快速切换      |
| 60 秒 – 3 分钟        | **8–12 秒**                | 正常的社交节奏                          |
| 3 – 10 分钟         | **12–20 秒**               | 给予喘息空间；每张卡片承载更多 |
| 10 – 30 分钟        | **20–35 秒**               | 长篇讲座/访谈节奏        |
| > 30 分钟           | **30–60 秒**               | 分段，近似章节感                 |

**第 2 步——密度乘数**（乘以基础节奏）：

| 抄本中的信号                                                                                                    | 乘数 | 效果                   |
| --------------------------------------------------------------------------------------------------------------------------- | ---------- | ------------------------ |
| **高密度** — 许多数字，明显的论点，断断续续的节奏，类列表的枚举，每 1-2 句是一个新想法 | **× 0.7**  | 切割更快，更多卡片  |
| **中等密度** — 混合流动，既有数据又有叙述                                                                | **× 1.0**  | 基础节奏                |
| **低密度** — 一个扩展的故事，重复的重新构建，缓慢的反思节奏，单一论点逐步展开                 | **× 1.5**  | 切割更慢，更少卡片 |

**第 3 步——计算：**```
secPerCard = basePace × densityMultiplier
cardCount  = max(5, round(videoDurationSec / secPerCard))
```
--- 待翻译片段 9 ---

示例（注意 — **没有上限**；长视频自然会产生更多卡片）：

- **30秒短片，单一笑点（低密度）** → 7 × 1.5 = 10.5s/card → round(30/10.5)=3 → 向下取整到 **5** 个卡片
- **60秒反思独白（低密度）** → 10 × 1.5 = 15s/card → **4** → 向下取整到 **5** 个卡片
- **121秒谈话式视频，包含丰富数据（高密度）** → 10 × 0.7 = 7s/card → **17** 个卡片
- **5分钟访谈，混合密度** → 16 × 1.0 = 16s/card → **19** 个卡片
- **10分钟深度探讨，高密度** → 16 × 0.7 = 11s/card → **55** 个卡片
- **30分钟讲座，中等密度** → 28 × 1.0 = 28s/card → **64** 个卡片
- **1小时播客，低密度** → 45 × 1.5 = 67.5s/card → **53** 个卡片

当一个卡片的时长超过约15秒时，计划制作一个更丰富的卡片（数据块、多步骤展示、几个子点随着交错的动画展开）——静态的单行文字在超过8秒后会变得无聊。对于许多卡片超过30秒的长片段，考虑**将时间线分割成子组合**（每个章节一个 .html 文件，通过 `data-composition-src` 挂载），以便每个文件中的 GSAP 时间线保持可管理性 — 请参阅 `timeline_track_too_dense` HyperFrames 的 lint 警告。

`content` 可以是一个简单的字符串（"标题：年增长率 5.69%\n备注：..."）或任何捕获数据的 JSON 格式。代理决定每个卡片的格式。

**可选的结尾部分**。这个技能**没有固定的品牌结尾**。如果用户想要一个结束卡片，请自行设计一个中性的卡片（文字标志 + 一行标语，约1.5-2秒，淡入 -> 短暂停留 -> 淡出），将其附加到 `cards[]`，并将 `composition.durationSeconds` 延长到其 `endSec`。否则，以最后一个内容卡片结束。

### 7. 确定渲染策略

#### 首先与用户确认视觉方向（先做这个）

在开始设计卡片或确定边界之前，**请用户选择输出比例、布局、风格和卡片密度预设**。框架是根据所选布局 × 风格组合自动选择的（参见下面的“自动选择框架”表格）。在发送问题之前，**预先计算两件事**：

1. **`recommendedRatio`** 从源视频的宽高比（`metadata.json` 宽度 / 高度）：
   - `sourceAspect = width / height`
   - `sourceAspect ≥ 1.5`（≥ 约3:2 宽）→ 推荐 **`16:9`**
   - `sourceAspect ≤ 0.7`（≤ 约9:13 高）→ 推荐 **`9:16`**
   - `0.7 < sourceAspect < 1.5`（接近正方形）→ 推荐 **`4:5`**

   在推荐的选项标签上标记 "（推荐 · 与源视频 X:Y 匹配）"，以便用户看到为什么推荐它。

2. **`autoCount`** 从第6步（`max(5, round(videoSec / (basePace × densityMultiplier)))`）计算得出，以便“自动”选项的标签可以显示具体的数字。

**环境兼容性 — 选择最佳可用的问题渠道。** 并非每个运行时都暴露相同结构的问答工具。按此顺序应用：

1. **`AskUserQuestion`**（Claude Code, Anthropic Console）— 使用下面的结构化4问题调用。
2. **其他本地澄清工具**（例如 `ask_question`, `request_user_input`, IDE特定的提示）— 使用相同的4个问题文本和选项列表。使用该工具。保留推荐标记和预计算的值。
3. **没有本地工具**（Codex CLI, 仅纯文本运行时）— **直接在普通对话中提问**。使用本节末尾的纯文本模板。保持**一条消息，4个编号问题**（全局上限是每轮2-5个问题；我们保持在其中）。

适用于每个渠道的规则：

- **最多每轮提问2-5个问题**。我们这里的4个符合。
- 即使缺少的信息不会阻止渲染，**提问一次以确认对最终输出有实质性影响的参数**（比例、布局、风格、卡片数量）。
- 如果用户已经预先批准了默认值（“直接使用默认值”、“不需要问”、“自动选择一切”）或要求你不要提问 — **完全跳过问题**，并使用：`recommendedRatio`, `layout="stack"`（最安全的跨比例默认值）, `style` 从对话基调中选择最中立的组（editorial/data）, `autoCount`。用一句话告诉用户你选择了什么，然后继续。

**渠道A — 本地 `AskUserQuestion`：**```
// Precompute before the call:
//   recommendedRatio = "16:9" | "9:16" | "4:5"
//   autoCount        = integer (from Step 6)

AskUserQuestion({
  questions: [
    {
      question: "Output video aspect ratio (canvas):",
      header: "Aspect ratio",
      multiSelect: false,
      // Reorder so the recommended option appears FIRST (per AskUserQuestion convention).
      // Append " (recommended · matches source video W×H)" to the recommended option's label.
      options: [
        { label: "16:9 (1920×1080) landscape", description: "TV / YouTube / desktop playback. Most natural when the source video is already landscape; widest canvas." },
        { label: "9:16 (1080×1920) portrait", description: "TikTok / Reels / short-form mobile. Most natural for portrait source; native mobile experience." },
        { label: "4:5 (1080×1350) near-portrait", description: "Instagram feed / WeChat Moments. Best when source is near-square or you want to cover both platforms." }
      ]
    },
    {
      question: "Choose the overall layout: how should the video and cards coexist on the canvas?",
      header: "Layout",
      multiSelect: false,
      options: [
        { label: "side-by-side (split)",  description: "Video and card each take half the canvas. Most stable for interview / data side-by-side; clear visual separation." },
        { label: "top-bottom (stack)",    description: "Video on top (~52%), card below. Classic combo of speaker face + summary card; works well in portrait too." },
        { label: "picture-in-picture (pip)", description: "Card fills the canvas, video shrinks to a rounded corner window. Use when content is primary and speaker is secondary." },
        { label: "full-screen overlay (overlay)", description: "Video plays full-bleed, card floats as a glass layer on top. Strong cinematic / emotional feel." }
      ]
    },
    {
      question: "Choose the card visual style (style):",
      header: "Style group",
      multiSelect: false,
      // NOTE: these 3 groups intentionally match the frame auto-pick matrix
      // rows below, so picking a group resolves both `style` group AND the
      // frame matrix column in one step. Memberships are mutually exclusive.
      options: [
        { label: "warm paper (warm-paper)", description: "academic notebook · editorial big-type · whiteboard hand-drawn · xhs social. Best for interview reflections, product launches, lifestyle, emotional stories." },
        { label: "clinical / cold (clinical)",   description: "audit magazine · swiss grid · terminal CLI · minimal modern. Best for financial analysis, investigative reports, technical tutorials, serious presentations." },
        { label: "experimental / avant-garde (experimental)", description: "geom color-clash geometry · spotlight dark-background. Best for short-form highlights, product launches, strong emotion, cinematic feel." }
      ]
    },
    {
      question: "Card count (takeaway pacing): how many cards to cut?",
      header: "Card count",
      multiSelect: false,
      options: [
        { label: "Auto (recommended) · approx N cards", description: "Inferred automatically from video duration and information density (see Step 6 rules). This run estimates approx N cards. Substitute the real N (your autoCount) into the label." },
        { label: "Fewer · approx round(N × 0.6) cards", description: "Sparser cuts, each card holds longer — suits reflective / slow-paced content." },
        { label: "More · approx round(N × 1.5) cards", description: "Tighter cuts, faster rhythm — suits staccato / data-dense / short-form highlight content." }
      ]
    }
  ]
})
```
--- 

**关于“其他”** — `AskUserQuestion` 自动向卡片计数问题添加一个“其他”选项。用户可以直接输入一个数字（例如“8”、“20”）作为 cardCount 目标。将输入解析为整数：如果解析成功 → 使用该值（最小值为5）；如果解析失败 → 回退到“自动”。

**频道 B — 纯文本回退**（Codex CLI，没有原生问答工具的运行时）。将其作为一条普通消息发布，然后等待回复。使用项目符号样式的 1/2/3/4 保持回复的可解析性：

---```
I need to confirm four visual decisions with you before I start cutting cards:

1) Output aspect ratio (canvas):
   A. 16:9 landscape (1920×1080) — TV / YouTube / desktop playback
   B. 9:16 portrait (1080×1920) — TikTok / Reels / short-form mobile
   C. 4:5 near-portrait (1080×1350) — Instagram feed / works for both platforms
   ▸ My recommendation:  <recommendedRatio>  (matches source video W×H = <sourceW>×<sourceH>)

2) Overall layout (how video & card coexist):
   A. split   side-by-side (50/50)
   B. stack   top-bottom (video top, card bottom)
   C. pip     picture-in-picture (card full canvas, video rounded corner window)
   D. overlay full-screen glass overlay (video full-bleed, card glass layer)

3) Card style group (maps to frame auto-pick matrix, pick 1 of 3):
   A. warm paper (warm-paper)      (academic / editorial / whiteboard / xhs)
   B. clinical / cold (clinical)   (audit / swiss / terminal / minimal)
   C. experimental (experimental)  (geom / spotlight)

4) Card count (takeaway pacing):
   A. Auto (recommended) — approx <autoCount> cards
   B. Fewer — approx round(<autoCount> × 0.6) cards
   C. More — approx round(<autoCount> × 1.5) cards
   D. Give me a specific number (e.g. "8", "20")

Reply format: "1A 2C 3B 4A" or natural language is fine.
If you want all recommended defaults, reply "default" / "auto" / "use all recommendations".
```
--- 翻译后的片段 11 ---

解析纯文本回复：

- 接受宽松的格式：`"1A 2C 3B 4A"`, `"A C B A"`, `"16:9 / pip / data / auto"`, full sentences, or `默认`。
- 如果任何回答含糊不清 → 仅重新询问含糊的部分（仍在2-5的限制内）。
- 如果用户说“默认 / 自动 / 使用所有推荐” → 跳过而不重新询问。

用户回答后（任何渠道）：

1. **解析输出画布** 从比例答案中 — 这些是确切的 `storyboard.composition.width / height` 值：

   | 用户选择 | composition.width × height | storyboard.layout 字段                                       |
   | -------- | -------------------------- | ------------------------------------------------------------- |
   | `16:9`   | **1920 × 1080**            | `"landscape"`                                                 |
   | `9:16`   | **1080 × 1920**            | `"portrait"`                                                  |
   | `4:5`    | **1080 × 1350**            | `"portrait"` (模式将4:5视为纵向 — 高度 > 宽度) |

   对于 **4:5 在 `references/layouts/*.html` 内的边界** — 这些文件仅记录横向（1920×1080）和纵向（1080×1920）。对于4:5（1080×1350），通过 **从纵向按比例缩放** 得出边界：保持水平值，垂直值按 `1350/1920 ≈ 0.703` 缩放。例如：`overlay` 纵向卡片 = `{ x: 24, y: 1280, w: 1032, h: 564 }` → 4:5 卡片 = `{ x: 24, y: round(1280 × 0.703), w: 1032, h: round(564 × 0.703) }` = `{ x: 24, y: 900, w: 1032, h: 397 }`。

2. **将风格组映射到特定风格**，通过观察对话语气选择最适合的，但保持在用户选择的组内。如果在组内两个特定风格之间不确定，发送第二个 `AskUserQuestion`，包含这些2-4个特定风格选项。

3. **解析最终卡片数量** 从密度答案中：

   | 用户选择             | 最终卡片数量                           |
   | ----------------------- | ----------------------------------------- |
   | 自动（推荐）      | 你已经计算出的 `autoCount`      |
   | 更少                   | `max(5, round(autoCount × 0.6))`          |
   | 更多                    | `round(autoCount × 1.5)` (无上限) |
   | 其他 = "<n>" (整数) | `max(5, parseInt(n))`                     |
   | 其他 = 其他任何内容   | 回退到 `autoCount`                  |

4. **自动选择视频帧** 从此表中（帧不询问用户 — 它们由布局 × 风格决定）：

   | 布局    | 暖色调纸张风格（学术 / 白板 / 编辑 / 小红书） | 临床风格（审计 / 瑞士 / 终端 / 极简） | 实验风格（几何 / 聚光灯） |
   | --------- | ----------------------------------------------------------- | ---------------------------------------------------- | -------------------------------------- |
   | `split`   | `polaroid`                                                  | `hairline`                                           | `clean`                                |
   | `stack`   | `polaroid`                                                  | `hairline`                                           | `clean`                                |
   | `pip`     | `clean` (pip pill已经有边框)                       | `clean`                                              | `clean`                                |
   | `overlay` | `clean` (full-bleed禁止deco帧)                    | `clean`                                              | `clean`                                |

5. **用一句话告诉用户你选择了什么** — 比例（+ 画布大小）、布局、特定风格、帧和最终卡片数量 — 然后继续执行第7步的其余部分（每张卡片的布局、动态模式）。
6. 将五个值（比例 / 布局 / 风格 / 帧 / 卡片数量）记录在工作记忆中（不需要模式字段）；在编��第8步中每张卡片的HTML以及读取匹配的 `references/<dim>/<key>.html` 以获取标记和结构时，你将引用它们。

如果用户通过“其他”选项选择了一个不在10种风格库中的自由文本风格名称，将其视为设计全新卡片视觉的提示，但仍需基于所选布局的边界。

#### 渲染策略输入

在第7.0步中锁定比例 / 布局 / 风格 / 卡片数量 / 帧后，剩余的每张卡片决策是：

- **源视频在GSAP目标内的适配**：视频元素具有 `object-fit: cover` 并被裁剪到 `#video-wrap` 的补间边界。
  如果你不想裁剪（例如，横向画布上的纵向源不应被裁剪其 top/bottom），将补间目标对准与源纵横比匹配的矩形，并让周围的画布显示出来（或填充卡片 / 背景）。
- **每张卡片的 `card.zone`**：从你选择的构图布局中得出（分割 → 侧面板，堆叠 → 下三分之一，PIP → 全屏，叠加 → 视频叠加），或者为一次性变体选择不同的区域（全屏用于英雄 / 引言，白板区域用于密集数据）。
- **每张卡片的 `accentIndex`**：每张卡片选取5种主题强调色之一。在卡片之间变化以形成节奏；当两张卡片属于同一叙事节拍时，重用相同的索引。
- **动态词汇**：从 `data-anim` 种类中选择2-3种可重复的模式（见后面的表格）并坚持使用，以便构图感觉连贯。--- 待翻译片段 12 ---

从这些 `themeId` 调色板中选择（在你的组合 `<style>` 块中使用它们作为 `--accent-N` / `--bg` / `--text` CSS 变量）：

| themeId | 强调色板（5 种颜色）                     | 棋盘背景          | 文字      |
| ------- | ---------------------------------------- | ----------------- | --------- |
| classic | `#1971c2 #e03131 #2f9e44 #e8590c #9c36b5` | `#FFF9E3`（纸张） | `#1e1e1e` |
| noir    | `#4cc9f0 #f72585 #4ade80 #fb923c #a78bfa` | `#1a1a1a`         | `#f1f1f1` |
| mint    | `#0077b6 #d62828 #2d6a4f #e76f51 #7209b7` | `#e8faf0`         | `#1b4332` |
| craft   | `#bf5700 #d62728 #6c757d #e9b54a #3d5a80` | `#f6efe1`         | `#2d2d2d` |
| slate   | `#0ea5e9 #ef4444 #22c55e #f97316 #a855f7` | `#1e293b`         | `#f1f5f9` |
| mono    | `#000 #555 #888 #aaa #ccc`                | `#fff`            | `#000`    |

可用字体（woff2 格式，位于 `<SKILL_DIR>/assets/fonts/`，在第 9 步中暂存到工作目录）：`Caveat`（手写体），
`LXGW WenKai TC`（中文手写体），`Inter`（现代无衬线字体），`Virgil`
（几何手写体）。通过 `@font-face` 或 `font-family` 直接引用。

关于视觉模式的灵感，`<SKILL_DIR>/references/styles/`
提供了 10 个自包含的参考卡片（学术 / 编辑 / 极简
/ 聚光 / 几何 / 白板 / 审计 / 终端 / 瑞士 / xhs），你可以复制这些作为起点 —— 但**不要觉得必须匹配其中任何一个**。每张卡片都是你自己的设计。

#### 视觉设计库（<SKILL_DIR>/references/）

除了组合级别的 `themeId`，该技能还提供了一个更丰富的**参考库**，位于 `<SKILL_DIR>/references/`，涵盖了你可以自由组合的三个**正交**视觉维度：```
Style  ×  Layout  ×  VideoFrame
 (10)      (4)         (3)
```
--- 翻译后的片段 13 ---

| 维度       | 键                                                                                                  | 它决定了什么                                                            |
| ---------- | ------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| **样式**   | `academic` `editorial` `minimal` `spotlight` `geom` `whiteboard` `audit` `terminal` `swiss` `xhs` | 卡片视觉语言 — 字体、颜色、装饰、卡片内布局                             |
| **布局**   | `split` `stack` `pip` `overlay`                                                                   | 源视频和卡片如何在画布上共享空间                                       |
| **框架**   | `clean` `hairline` `polaroid`                                                                     | 视频元素周围的装饰性框架                                               |

阅读 `<SKILL_DIR>/references/DESIGN_INDEX.md`
以获取完整矩阵和宽松决策指南（访谈/产品发布/数据分析/
社交视频/技术教程/情感故事……）。当你决定使用特定的
样式/布局/框架时，请阅读相应的文件：

- `references/styles/<key>.html` — 包含该样式的CSS标记（颜色、字体、内边距、装饰）和占位符
  要点的独立卡片片段。复制 `.card[data-card-id="ref-<key>"]` 样式块，重命名
  data-card-id 为你的卡片的ID，替换占位符内容为
  实际的要点，你就完成了。
- `references/layouts/<key>.html` — 精确的 `videoBounds` + `cardBounds` 适用于
  横版和竖版，带有可复制粘贴的JSON片段用于
  `storyboard.json` 的每个卡片的 `layout` 字段。
- `references/frames/<key>.html` — 作为 `#video-wrap` 的同级元素添加的装饰性HTML，以及组合CSS的放置说明。

**每个卡片选择 `style × layout × frame`** — 只要过渡流畅，你可以在卡片之间更改所有三个
元素。一个常见的节奏是：
打开 `editorial × overlay × clean`，切换到 `audit × split × hairline`
用于数据卡片，以 `whiteboard × pip × polaroid` 结束。

10种样式是技能端的设计标记，**不是组合级主题** —
它们不需要在 `storyboard.composition` 中声明；它们存在于
每个卡片的HTML中。`themeId` 字段仍然可以选择
组合级调色板（上表）来控制页面主体背景和视频边框框架。

#### 布局组合（卡片 + 视频）

每个卡片的两个协调决策定义了它如何与源视频共享画布：

- **`card.zone`**（在 `storyboard.json` 中声明） — 5个模式值之一；在第9步中编写卡片主机包装器的内联 `style` 时，将其解析为像素边界（根据第6步中的表格）。
- **`#video-wrap` 在此卡片的时间窗口的边界**（在组合的GSAP时间轴中命令式声明） — 代理对每个布局过渡将 `#video-wrap` 过渡到目标矩形。

模式不存储每个卡片的视频边界。`videoTrack.bounds` 是
**一次性** 在组合级别（默认为全画布）。视频在卡片之间的“移动”纯粹是由GSAP动画在 `index.html` 中创作的。没有 `card.layout` 字段 — 早期版本的文件发明了一个；真正的模式只有 `card.zone`。

**4种组合布局**（来自 `references/layouts/`） — 每个都是将 `zone` 与 `#video-wrap` 过渡目标配对的配方：

| 组合布局         | 推荐的 `card.zone` | GSAP目标用于 `#video-wrap`（横版1920×1080）                       | GSAP目标用于 `#video-wrap`（竖版1080×1920）                | 何时使用                                     |
| ------------------ | ----------------------- | ------------------------------------------------------------------------- | ----------------------------------------------------------------- | ----------------------------------------------- |
| `split`            | `side-panel`            | `{ left: 960, top: 0, width: 960, height: 1080 }`                         | `{ left: 0, top: 960, width: 1080, height: 960 }`（下半部分）   | 演讲者 + 数据并排 / 50:50 权重      |
| `stack`            | `lower-third`           | `{ left: 14, top: 14, width: 1892, height: 548 }`（顶部52%）               | `{ left: 0, top: 0, width: 1080, height: 844 }`（顶部44%）         | 演讲者在上 + 摘要卡片在下             |
| `pip`              | `fullscreen`            | `{ left: 1480, top: 760, width: 400, height: 300 }` + 添加 `.framed` 类 | `{ left: 690, top: 28, width: 360, height: 203 }` + 添加 `.framed` | 内容繁重的卡片 + 角落管道                 |
| `overlay`          | `video-overlay`         | `{ left: 0, top: 0, width: 1920, height: 1080 }`（全出血）             | `{ left: 0, top: 0, width: 1080, height: 1920 }`                  | 电影般的/戏剧性的/玻璃卡片在全视频上 |

对于4:5（1080×1350），将竖版 y/h 值按 `1350/1920 ≈ 0.703` 缩放
（参见第7.0步通道A / 通道B `recommendedRatio` 分辨率
表格）。

**其他一次性变体的区域值**（仍使用 `card.zone`；没有
假的“布局”字段）：--- 待翻译片段 14 ---

| `zone`            | 解析后的边界                                           | 常见用途                              |
| ----------------- | ------------------------------------------------------ | ------------------------------------- |
| `fullscreen`      | 覆盖整个画布                                           | 英雄卡片，视频渐变到 hidden/pip |
| `whiteboard-area` | 内缩40像素边距（横屏）或底部45%（竖屏）               | 密集数据卡片，自由边距               |
| `lower-third`     | 底部30%区域                                           | 谈话式注释                           |
| `side-panel`      | 右侧42%（横屏）或底部40%（竖屏）                      | 侧边栏 / "分割"页脚                  |
| `video-overlay`   | 整个画布；预期透明卡片根                               | 全屏视频上的玻璃叠加层               |

你可以根据每张卡片混合使用不同的方案 —— 根据当下情况选择 `card.zone`，然后编写卡片间 GSAP 渐变的 `#video-wrap`。

#### 故事板渲染合同

`storyboard.json` 是一个内部规划工具 —— 没有 CLI 命令会解析它。它的存在是为了在你编写每张卡片的 HTML 之前，明确你的时间和内容决策。坚持使用下面的 v3 风格结构，以便相同的概述能驱动你在第9步中组合的内容。

必需的结构（参见第6步的完整示例）：

- `schemaVersion: 3`
- `composition: { fps, width, height, durationSeconds, layout, themeId, seed }` — 注意 `durationSeconds`/`fps`/`themeId`/`layout` **位于** `composition` **内部**，而不是顶层
- `videoTrack: { sourcePath, startSec, endSec, bounds? }` — 视频边界默认为整个画布
- `subtitles: { enabled, ... }`
- `cards[]` — 每张卡片都有6个必需字段：`id`, `intent`, `startSec`, `endSec`, `accentIndex`, `zone`, `contentHints`

规则：

- 卡片时间保持在 `composition.durationSeconds` 内，除非有意，否则不应重叠（使用 `data-track-index` 来控制它们重叠时的 z 顺序）。
- 视觉细节位于卡片 HTML 片段中（步骤8），而不是在 `contentHints` 中。`contentHints` 是你自己设计的用于设计卡片的结构化提示；渲染的外观是 HTML。
- 保持故事板结构的稳定性 —— 即使没有任何东西解析它，你在编写步骤 8/9 时会回读它，一致性保持了卡片 ID 和时间同步。
- 代理端的决策，如“我选择了叠加 × 几何 × 简洁”不应放在 `storyboard.json` 中 —— 将它们保留在工作记忆中，并在编写卡片 HTML + GSAP 渐变时使用它们。

**对于与视频共享画布的卡片，使用透明卡片背景。**
当 GSAP 渐变使视频保持可见 behind/beside 卡片（叠加方案、PIP 方案或任何 `card.zone = 'lower-third' | 'video-overlay'` 时刻），卡片的 `.root` 一定不能绘制完全不透明的背景 —— 否则会遮挡视频。两种模式：```css
/* Pattern A: transparent root, page body provides the cream backdrop */
html,
body {
  background: var(--bg);
}
.card[data-card-id="card-X"] .root {
  background: transparent;
}

/* Pattern B: explicit per-card background ONLY for fullscreen cards */
.card[data-card-id="card-hero"] .root {
  background: var(--bg);
}
.card[data-card-id="card-overlay"] .root {
  background: transparent;
}
```
--- 
name: `side-panel`-zone-cards-split-recipe
description: 对于 `side-panel` 区域卡片（分割配方），卡片宿主已经是画布的一半，因此不透明的卡片背景是可以的——它只覆盖自己的一半。

### 8. 编写每张卡片的 HTML

为每张卡片创建 `$WORK_DIR/public/cards/{card-id}.html`。每个文件包含一个遵循以下契约的单一根 HTML 片段：

#### 卡片 HTML 契约```html
<div class="card" data-card-id="{cardId}">
  <style>
    /* MUST: every rule starts with .card[data-card-id="{cardId}"] */
    .card[data-card-id="card-01"] .root {
      width: 100%; height: 100%;
      display: flex; ...;
      font-family: 'Caveat', 'LXGW WenKai TC', serif;
      color: var(--text);
      background: var(--bg);
    }
    .card[data-card-id="card-01"] .title { font-size: 84px; ... }
  </style>

  <div class="root">
    <h1
      id="card-01-title"
      data-anim="kinetic-chars"
      data-anim-at="0.3"
      data-anim-duration="0.5"
      data-anim-stagger="0.04"
      data-anim-pattern="pop"
    >
      <span class="char">S</span>
      <span class="char">u</span>
    </h1>
    <div
      id="card-01-line"
      data-anim="grow-x"
      data-anim-at="0.65"
      data-anim-duration="0.5"
      data-anim-target-w="420"
      style="width:0;height:8px;background:var(--accent-0);border-radius:4px;"
    ></div>
  </div>
</div>
```
--- 待翻译片段 16 ---

**硬性规则**（`hyperframes` lint 将拒绝违规行为）：

- 单个根 `<div class="card" data-card-id="{cardId}">`
- 内联 `<style>` 规则必须以前面的作用域选择器为前缀
- **不允许使用 `<script>` 标签**
- **不允许在** `src=` / `href=` **中使用外部 URL**（不允许使用 CDN，不允许使用远程字体）
- **不允许使用内联事件处理器**（`onclick=` 等）
- 所有资源通过相对路径进入同一个 `public/` 目录
- 使用 `var(--accent-N)` 等颜色，以确保在不同主题之间具有可移植性

**动画是声明的，而不是编码的。** 仅使用 `data-anim-*` 属性；不要编写 `<script>` 来制作动画。在第9步中，将每个 `data-anim-*` 声明编译到单一的 GSAP 主时间轴中。

#### 卡片尺寸 — 纵向优先的移动端

10 个 `references/styles/*.html` 尺寸适用于 **1920×1080 横向** 预览。当 `storyboard.layout = "portrait"`（1080×1920，社交/移动端的主要情况），**将每个视觉尺寸放大** — 手机将屏幕拿得很近，同样的像素数在横向电视式画布上看起来比在手机上小。

| 标记                     | 横向基准           | **纵向目标**        | 缩放比例      |
| ------------------------ | ------------------ | ------------------- | ------------- |
| 标题（h1/h2 英雄）        | 64–96px            | **88–132px**        | ×1.35         |
| 详情/正文                | 24–30px            | **30–40px**         | ×1.30         |
| 提示/芯片标签            | 14–16px            | **18–22px**         | ×1.30         |
| 时间码/元数据            | 12–14px            | **16–18px**         | ×1.30         |
| 数据块主要数字           | 48–60px            | **64–88px**         | ×1.40         |
| 行高倍数                 | 1.05–1.5           | 相同                 | （不缩放）    |

**经验法则：** `portraitPx = round(landscapePx × 1.3)`，然后向下取整到最近的4px倍数以保持视觉节奏。英雄标题可以放大到×1.4；小型元文本保持在×1.2以避免拥挤。

**内边距在纵向中略微缩小** — 卡片更窄，因此大的横向内边距（40–64px）会占用太多宽度。在纵向中使用24–36px的水平内边距。

如果您正在制作一个必须在**两种**布局中都能工作的单一卡片，请优先考虑在卡片根上使用 `@container` 查询，而不是硬编码尺寸：```css
.card[data-card-id="X"] .root {
  container-type: inline-size;
}
.card[data-card-id="X"] .title {
  font-size: clamp(64px, 8.5cqi, 132px);
}
.card[data-card-id="X"] .detail {
  font-size: clamp(24px, 3.2cqi, 40px);
}
```
--- 待翻译片段 17 ---

但对于大多数卡片来说，单一布局选择是可行的 —— 只需选择与故事板的 `layout` 字段匹配的尺寸表格列即可。

#### 可用的 `data-anim` 类型

| 类型             | 用于                 | 关键参数                                                                                        |
| ---------------- | -------------------- | ----------------------------------------------------------------------------------------------- |
| `fade-in`       | 进入                 | `at`, `duration`, `ease?`                                                                       |
| `fade-out`      | 退出                 | `at`, `duration`, `ease?`                                                                       |
| `slide-in`      | 滑动进入             | `at`, `duration`, `from=left\|right\|top\|bottom`, `distance`                                   |
| `kinetic-chars` | 逐字符弹出           | `at`, `duration`, `stagger`, `pattern=pop\|fade` — 元素需要 `<span class="char">` 子元素 |
| `typewriter`    | 逐字符淡入           | 与 kinetic-chars 相同，但默认延迟更慢                                                |
| `count-up`      | 动画数字             | `at`, `duration`, `from`, `to`, `format=.0f\|.1f\|.2f\|,d`                                      |
| `draw-path`     | SVG 路径揭示         | `at`, `duration` — 元素应该是 `<path>`                                                 |
| `grow-y`        | 柱状图高度           | `at`, `duration`, `target-h` (像素) — 元素从 `height:0` 开始                                   |
| `grow-x`        | 柱状图宽度           | `at`, `duration`, `target-w` (像素) — 元素从 `width:0` 开始                                    |
| `scale-pop`     | 弹出式进入           | `at`, `duration`                                                                                |
| `blur-in`       | 从非聚焦到聚焦       | `at`, `duration`                                                                                |
| `mask-reveal`   | 剪辑揭示             | `at`, `duration`, `direction=left\|right\|top\|bottom`                                          |
| `morph-to`      | 补间任何 CSS         | `at`, `duration`, `props='{...JSON...}'`                                                        |

`data-anim-at` 是 **相对于卡片开始时间的秒数** — 当你在第 9 步中将每个声明编译到 GSAP 时间轴时，添加卡片的 `startSec` 以获得绝对时间并量化到 1/fps.

### 9. 组装合成 HTML

安排好资源并编写 `$WORK_DIR/public/index.html`：```bash
# SKILL_DIR is injected by the host ("Base directory for this skill: …")
SKILL_DIR="<SKILL_DIR>"

mkdir -p "$WORK_DIR/public/fonts" "$WORK_DIR/public/vendor" "$WORK_DIR/public/cards"
cp -n "$SKILL_DIR/assets/fonts/"*            "$WORK_DIR/public/fonts/"
cp -n "$SKILL_DIR/assets/vendor/gsap.min.js" "$WORK_DIR/public/vendor/"
# stage the input video — RE-ENCODE with dense keyframes. Sources with a sparse GOP
# (keyframe interval > ~1s) freeze on seek in the renderer (a frozen frame under the
# overlays); -g / -keyint_min set to your composition fps make every frame seekable.
# (Set both to your fps — 30 shown; use 24/25/60 to match.)
ffmpeg -y -i "$VIDEO_PATH" -c:v libx264 -crf 18 -g 30 -keyint_min 30 \
  -pix_fmt yuv420p -movflags +faststart -c:a aac "$WORK_DIR/public/input-video.mp4"
```
--- 
name: Composition Template
description: 组合模板
---

#### 组合模板

**组合模板**用于定义如何将多个技能组合成一个复杂的对话流。以下是组合模板的结构：

```yaml
composition:
  ⟦P0⟧: ⟦P1⟧
  ⟦P2⟧:
    ⟦P3⟧: ⟦P4⟧
    ⟦P5⟧: ⟦P6⟧
```

### 主要组成部分

1. **⟦P0⟧**: 这是组合模板的**名称**。它用于在对话流中引用这个组合。
2. **⟦P1⟧**: 这是组合模板的**类型**。常见的类型包括：
   - `sequence`: 按顺序执行技能。
   - `parallel`: 并行执行技能。
   - `conditional`: 根据条件执行技能。
3. **⟦P2⟧**: 这是**技能列表**，包含要组合的技能。
   - **⟦P3⟧**: 第一个技能的**名称**。
   - **⟦P4⟧**: 第一个技能的**参数**。
   - **⟦P5⟧**: 第二个技能的**名称**。
   - **⟦P6⟧**: 第二个技能的**参数**。

### 示例

以下是一个简单的组合模板示例，它按顺序执行两个技能：

```yaml
composition:
  myComposition:
    type: sequence
    skills:
      - name: ⟦Skill1⟧
        params:
          ⟦Param1⟧: ⟦Value1⟧
      - name: ⟦Skill2⟧
        params:
          ⟦Param2⟧: ⟦Value2⟧
```

### 详细说明

- **⟦Skill1⟧** 和 **⟦Skill2⟧** 是要执行的技能名称。
- **⟦Param1⟧** 和 **⟦Param2⟧** 是传递给技能的参数。
- **⟦Value1⟧** 和 **⟦Value2⟧** 是参数的具体值。

### 注意事项

- 确保组合模板中的技能名称和参数正确无误。
- 组合模板可以嵌套使用，以创建更复杂的对话流。
- 在设计组合模板时，考虑技能执行的顺序和依赖关系。

### 高级用法

组合模板还支持**条件执行**和**错误处理**。例如：

```yaml
composition:
  myConditionalComposition:
    type: conditional
    condition: ⟦Condition⟧
    true:
      - name: ⟦SkillIfTrue⟧
        params:
          ⟦ParamIfTrue⟧: ⟦ValueIfTrue⟧
    false:
      - name: ⟦SkillIfFalse⟧
        params:
          ⟦ParamIfFalse⟧: ⟦ValueIfFalse⟧
```

在这个示例中，根据 **⟦Condition⟧** 的结果执行不同的技能。

---

通过组合模板，您可以灵活地管理和组织技能，以实现复杂的对话逻辑。```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <style>
      @font-face {
        font-family: "Caveat";
        src: url("fonts/Caveat-400-latin.woff2") format("woff2");
        font-weight: 400;
        font-display: block;
      }
      @font-face {
        font-family: "Caveat";
        src: url("fonts/Caveat-700-latin.woff2") format("woff2");
        font-weight: 700;
        font-display: block;
      }
      @font-face {
        font-family: "LXGW WenKai TC";
        src: url("fonts/LXGWWenKaiTC-400-latin.woff2") format("woff2");
        font-weight: 400;
        font-display: block;
      }
      @font-face {
        font-family: "Inter";
        src: url("fonts/Inter-400-latin.woff2") format("woff2");
        font-weight: 400;
        font-display: block;
      }
      @font-face {
        font-family: "Inter";
        src: url("fonts/Inter-700-latin.woff2") format("woff2");
        font-weight: 700;
        font-display: block;
      }
      @font-face {
        font-family: "Virgil";
        src: url("fonts/Virgil.woff2") format("woff2");
        font-display: block;
      }

      :root {
        /* Pick from the themeId palette table in Step 7 — example: classic */
        --bg: #fff9e3;
        --text: #1e1e1e;
        --accent-0: #1971c2;
        --accent-1: #e03131;
        --accent-2: #2f9e44;
        --accent-3: #e8590c;
        --accent-4: #9c36b5;
        --font-family: "Caveat", "LXGW WenKai TC", serif;
      }
      * {
        box-sizing: border-box;
      }
      /* Body font-family MUST list concrete font names (not just var(--font-family)) —
   the HyperFrames renderer's static analyzer doesn't expand CSS variables when
   resolving fonts, so a var-only chain triggers `font_family_without_font_face`
   lint and falls back to a generic. Use the concrete chain here; cards that
   want the theme font can still reference var(--font-family) internally. */
      html,
      body {
        margin: 0;
        padding: 0;
        width: 100%;
        height: 100%;
        overflow: hidden;
        background: #000;
        font-family: "Inter", "Caveat", "LXGW WenKai TC", ui-sans-serif, system-ui, sans-serif;
      }
      #stage {
        position: relative;
        width: 100%;
        height: 100%;
        overflow: hidden;
      }

      /* video-wrapper holds the source video. Its position / size are animated
   over time by the master timeline (one tween per layout transition). */
      .video-wrapper {
        position: absolute;
        left: 0;
        top: 0;
        width: 1920px;
        height: 1080px;
        overflow: hidden;
        border-radius: 0;
        box-shadow: none;
      }
      .video-wrapper video {
        width: 100%;
        height: 100%;
        object-fit: cover;
      }

      .card-host {
        position: absolute;
        pointer-events: none;
        overflow: hidden;
      }
      .card-host .card {
        position: relative;
        width: 100%;
        height: 100%;
        overflow: hidden;
      }
      .card-host .char {
        display: inline-block;
        visibility: visible;
      }

      /* Subtle drop shadow + rounded corners for non-fullscreen video framings */
      .video-wrapper.framed {
        border-radius: 16px;
        box-shadow: 0 12px 40px rgba(0, 0, 0, 0.35);
      }
    </style>
  </head>
  <body>
    <div
      id="stage"
      data-composition-id="graphic-overlays"
      data-start="0"
      data-duration="121.2"
      data-fps="30"
      data-width="1920"
      data-height="1080"
    >
      <!-- Layer 1: source video — initial position matches card-01's layout -->
      <div class="video-wrapper" id="video-wrap">
        <video
          id="bg-video"
          src="input-video.mp4"
          muted
          playsinline
          data-start="0"
          data-duration="121.2"
          data-track-index="1"
        ></video>
      </div>

      <!-- Layer 2: each card-host sits at the bounds dictated by its layout. -->
      <!-- IMPORTANT: every card-host MUST carry BOTH "card-host" and "clip" classes. -->
      <!--   - "card-host"  → our positioning + pointer-events styles                 -->
      <!--   - "clip"       → HyperFrames runtime uses this to enforce visibility     -->
      <!--                    only during data-start … data-start+data-duration.      -->
      <!--                    Without "clip" the host stays visible the whole video   -->
      <!--                    (lint: timed_element_missing_clip_class).               -->
      <!-- Example: card-01 with zone="fullscreen" → card-host covers (0,0,1920,1080) -->
      <div
        class="card-host clip"
        data-card-id="card-01"
        data-start="1.0000"
        data-duration="6.5000"
        data-track-index="2"
        style="left:0;top:0;width:1920px;height:1080px;visibility:hidden;opacity:0;"
      >
        <!-- paste the contents of public/cards/card-01.html here -->
      </div>

      <!-- Example: card-02 with zone="side-panel" (split composition layout) → card on left half -->
      <div
        class="card-host clip"
        data-card-id="card-02"
        data-start="8.0000"
        data-duration="12.0000"
        data-track-index="2"
        style="left:0;top:0;width:960px;height:1080px;visibility:hidden;opacity:0;"
      >
        <!-- card-02 HTML -->
      </div>

      <!-- ...one "card-host clip" per card with inline bounds matching resolveZoneBounds(card.zone)... -->

      <script src="vendor/gsap.min.js"></script>
      <script>
        (function () {
          // count-up formatter helper
          window.__fmt = function (v, fmt) {
            if (typeof fmt === "string" && /^\.[0-9]+f$/.test(fmt)) {
              return Number(v).toFixed(Number(fmt.slice(1, -1)));
            }
            if (fmt === ",d") return Math.round(v).toLocaleString();
            return String(Math.round(v));
          };

          const tl = window.gsap.timeline({ paused: true });

          // ── Card lifecycle (one block per card) ──
          // Example for card-01 [1.0, 7.5] with kinetic-chars at +0.3, grow-x at +0.65:

          // Enter (fade in over 0.4s)
          tl.set('.card-host[data-card-id="card-01"]', { visibility: "visible" }, 1.0);
          tl.fromTo(
            '.card-host[data-card-id="card-01"]',
            { opacity: 0 },
            { opacity: 1, duration: 0.4, ease: "power2.out" },
            1.0,
          );

          // Card-internal anims (compile each data-anim-* declaration here)
          tl.from(
            '.card[data-card-id="card-01"] #card-01-title .char',
            { opacity: 0, y: 8, scale: 0.8, duration: 0.5, ease: "power2.out", stagger: 0.04 },
            1.3,
          );
          tl.fromTo(
            '.card[data-card-id="card-01"] #card-01-line',
            { width: 0 },
            { width: 420, duration: 0.5, ease: "power2.out" },
            1.65,
          );

          // Exit (fade out over 0.35s, ending at endSec)
          tl.to(
            '.card-host[data-card-id="card-01"]',
            { opacity: 0, duration: 0.35, ease: "power2.in" },
            7.15,
          );
          tl.set('.card-host[data-card-id="card-01"]', { visibility: "hidden" }, 7.5);

          // ── Video framing transitions ──
          // When the next card uses a different composition layout, animate the
          // video-wrapper to its new bounds. Example: card-01 = fullscreen
          // (video hidden behind), card-02 = split composition (zone="side-panel"
          // → video on right, card on left).

          // Card-02 enters at 8.0s with the split composition. Animate video to
          // the right half during the card-01 → card-02 gap (between 7.5 and 8.0s).
          tl.set("#video-wrap", { className: "video-wrapper framed" }, 7.5);
          tl.to(
            "#video-wrap",
            { left: 960, top: 0, width: 960, height: 1080, duration: 0.6, ease: "power2.inOut" },
            7.5,
          );

          // Card-02 enter — same pattern as card-01
          tl.set('.card-host[data-card-id="card-02"]', { visibility: "visible" }, 8.0);
          tl.fromTo(
            '.card-host[data-card-id="card-02"]',
            { opacity: 0 },
            { opacity: 1, duration: 0.4, ease: "power2.out" },
            8.0,
          );
          // ...card-02 internal anims...

          // ── repeat for each card; if the NEXT card's layout differs,
          //    insert another tl.to('#video-wrap', ...) tween before its enter ──

          window.__timelines = window.__timelines || {};
          window.__timelines["graphic-overlays"] = tl;
        })();
      </script>
    </div>
  </body>
</html>
```
--- 翻译后的片段 19 ---

#### GSAP 语句速查表

将每个 `data-anim` 属性编译成 GSAP 语句。时间是**绝对秒数** = card.startSec + data-anim-at，量化到 1/fps.。选择器是 `.card[data-card-id="X"] #elementId`。

| data-anim                       | GSAP 语句模板                                                                                                                                                                                            |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `fade-in`                       | `tl.fromTo(SEL, { opacity: 0 }, { opacity: 1, duration: D, ease: 'power2.out' }, T);`                                                                                                                              |
| `fade-out`                      | `tl.to(SEL, { opacity: 0, duration: D, ease: 'power2.in' }, T);`                                                                                                                                                   |
| `slide-in` (from=left, dist=80) | `tl.fromTo(SEL, { opacity: 0, x: -80 }, { opacity: 1, x: 0, duration: D, ease: 'power2.out' }, T);`                                                                                                                |
| `kinetic-chars` (pop)           | `tl.from(SEL + ' .char', { opacity: 0, y: 8, scale: 0.8, duration: D, ease: 'power2.out', stagger: S }, T);`                                                                                                       |
| `count-up`                      | `(function(){const o={v:FROM};tl.to(o,{v:TO,duration:D,ease:'power2.out',onUpdate:function(){const el=document.querySelector(SEL);if(el)el.textContent=__fmt(o.v,'FMT');}},T);})();`                               |
| `draw-path`                     | `(function(){const el=document.querySelector(SEL);if(el){const L=el.getTotalLength();tl.set(SEL,{strokeDasharray:L,strokeDashoffset:L},T);tl.to(SEL,{strokeDashoffset:0,duration:D,ease:'power2.inOut'},T);}})();` |
| `grow-x` (target-w=W)           | `tl.fromTo(SEL, { width: 0 }, { width: W, duration: D, ease: 'power2.out' }, T);`                                                                                                                                  |
| `grow-y` (target-h=H)           | `tl.fromTo(SEL, { height: 0 }, { height: H, duration: D, ease: 'power2.out' }, T);`                                                                                                                                |
| `scale-pop`                     | `tl.fromTo(SEL, { opacity: 0, scale: 0.6 }, { opacity: 1, scale: 1, duration: D, ease: 'back.out(1.6)' }, T);`                                                                                                     |
| `mask-reveal` (direction=left)  | `tl.fromTo(SEL, { clipPath: 'inset(0 100% 0 0)' }, { clipPath: 'inset(0 0 0 0)', duration: D, ease: 'power2.inOut' }, T);`                                                                                         |

量化：`T = Math.round(absSec * fps) / fps`。在30fps时，最小步长是`1/30 ≈ 0.0333s`；在JS字面量内四舍五入到4位小数（`.toFixed(4)`）是可行的。

#### 视频框架参考（每个`layout`值）

视频容器的选择器是`#video-wrap`。使用`tl.to('#video-wrap', { ...bounds }, T)`在卡片之间动画化其边界。
初始边界应在元素上内联设置，以匹配card-01的布局。选择0.5-0.7秒的过渡持续时间，并使用`ease: 'power2.inOut'`。

**装饰性框架**（`clean` / `hairline` / `polaroid`）作为`#video-wrap`的**兄弟**存在，并在布局过渡中跟随它。
有关每个框架的放置HTML，建议的CSS以及它与哪些布局配对，请参见[⟦P33⟧](references/frames/)。快速规则：
`overlay`布局抑制装饰性框架（全屏视频与chrome冲突）；PiP布局已经有自己的药丸处理（border-radius + 白色圆环 + 阴影），因此仅在`split` / `stack`之上添加装饰性框架。

**GSAP目标查找表**，用于每个组合布局的`#video-wrap`（横向1920×1080 — 对于纵向和4:5，请参见`references/layouts/*.html`，其中列出了所有三种比例）：--- 

| 布局结构                   | 典型 card.zone | `#video-wrap` GSAP 目标                                                 | 额外 CSS 类                            |
| -------------------------- | ----------------- | ------------------------------------------------------------------------- | ------------------------------------------ |
| `split`                      | `side-panel`      | `{ left: 960, top: 0, width: 960, height: 1080 }`                         | —                                          |
| `stack`                      | `lower-third`     | `{ left: 14, top: 14, width: 1892, height: 548 }` (顶部 52%)               | —                                          |
| `pip` (右下角)                 | `fullscreen`      | `{ left: 1480, top: 760, width: 400, height: 300 }`                       | `pip-pill` (圆角 + 圆环 + 阴影) |
| `pip` (左上角)                     | `fullscreen`      | `{ left: 40, top: 40, width: 400, height: 300 }`                          | `pip-pill`                                 |
| `overlay` (视频全屏)         | `video-overlay`   | `{ left: 0, top: 0, width: 1920, height: 1080 }` (与默认无变化) | —                                          |
| **隐藏视频** (纯图形时刻) | `fullscreen`      | `{ opacity: 0 }` (或移出画布)                                     | —                                          |

在进入或离开画中画时刻时切换画中画药丸式装饰（圆角 + 白色圆环 + 阴影）：```js
// Enter pip — add chrome
tl.set("#video-wrap", { className: "video-wrapper pip-pill" }, T);
tl.to(
  "#video-wrap",
  { left: 1480, top: 760, width: 400, height: 300, duration: 0.6, ease: "power2.inOut" },
  T,
);

// Leave pip — back to clean full-bleed
tl.set("#video-wrap", { className: "video-wrapper" }, T_NEXT);
tl.to(
  "#video-wrap",
  { left: 0, top: 0, width: 1920, height: 1080, duration: 0.6, ease: "power2.inOut" },
  T_NEXT,
);
```
--- 待翻译片段 21 ---

**卡片宿主边界与区域匹配**。使用步骤6顶部的表格将卡片的 `zone` 解析为像素边界，然后将它们写入卡片宿主的内联 `style="left:Xpx;top:Ypx;width:Wpx;height:Hpx;..."`. For `video-overlay` 区域（覆盖配方），卡片宿主填满整个画布 - 你在 `.card .root` 内的CSS决定了实际可见卡片的位置。

#### HyperFrames 布局 / 动画 QA 规则

- 首先构建每个卡片的静态主画面帧：即卡片完全可见且可读的时刻。
- 确认视频、卡片、subtitles/captions 和图表不会意外重叠。
- 确认隐藏的视频区域被帧剪切，并且在预期边界之外不可见。
- 注册一个暂停的主时间轴作为 `window.__timelines["graphic-overlays"]`。
- 在页面加载时同步构建时间轴；不使用 `async`、`setTimeout`、Promise 或媒体 `play()` 调用。
- 在渲染路径中不要使用 `Math.random()` 或 `Date.now()`。
- 不要使用 `repeat: -1`；根据视频时长计算有限重复。
- 优先使用 GSAP 变换和透明度（`x`、`y`、`scale`、`rotation`、`opacity`）而不是布局属性（`top`、`left`、`width`、`height`）进行动画。
- 对包装器进行动画处理，例如 `#video-wrap`，而不是直接对视频元素尺寸进行动画处理。
- 避免同时从多个时间轴对同一元素的同一属性进行动画处理。
- 使用 `data-track-index`，而不是 `data-layer`；使用 `data-duration`，而不是 `data-end`。
- 每个定时元素（`card-host`、子合成等）必须在其自己的类旁边包含 `class="clip"` - 例如 `class="card-host clip"`。HyperFrames 运行时使用 `.clip` 来控制对 `data-start … data-start+data-duration` 窗口的可见性。没有它，元素将在整个视频中可见（lint: `timed_element_missing_clip_class`）。
- 对于 body / 全局 `font-family`，列出**具体的字体名称**（`'Inter', 'Caveat', …`）- 而不是像 `var(--font-family)` 这样的 CSS 变量。HyperFrames 字体解析器在静态分析期间不会扩展 CSS 变量（lint: `font_family_without_font_face`）。卡片可能仍在内部使用 `var(--font-family)`，因为它们的 `@font-face` 声明已加载。

### 10. 渲染为 MP4```bash
cd "$WORK_DIR"
PRODUCER_BROWSER_GPU_MODE=hardware npx hyperframes render public \
  -o output.mp4 \
  --fps 30
```
--- 待翻译片段 22 ---

`hyperframes render <dir>` 读取 `<dir>/index.html` 并生成 MP4 文件。
强烈建议在 macOS 上使用 `PRODUCER_BROWSER_GPU_MODE=hardware`（或 `--browser-gpu`）标志——仅使用软件的 Chromium 渲染在大多数笔记本电脑上会超时。

在进行完整渲染之前进行合理性检查，可以捕获特定时间戳的单帧：

```bash
npx hyperframes snapshot public --at 5    # → public/snapshots/frame-00-at-5s.png (a single --at ignores --out)
```
--- 待翻译片段 23 ---

### 11. 报告结果

告知用户：

- 工作目录路径
- `storyboard.json`（您设计的卡片大纲）
- `public/cards/*.html`（每个卡片的单个HTML）
- `public/index.html`（组合后的作品）
- `output.mp4`（最终视频）
- 使用的ASR提供商
- 卡片数量 + 您如何选择它们（用一句话说明）
- 任何缺失的键或质量注意事项

**可选的实时预览（仅在请求时提供）**。剪辑在 `public/index.html` 中不变地播放，叠加层位于顶部，因此预览是忠实的。**不要在运行时打开它**。当用户要求时，在渲染后启动一个长期运行的服务器并报告URL：```bash
(cd "$WORK_DIR/public" && npx hyperframes preview)   # or `npx hyperframes play` for a shareable link
```
--- 
name: ⟦P0⟧
description: 除非用户要求，否则不要删除工作目录。
---