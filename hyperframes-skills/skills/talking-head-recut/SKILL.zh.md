---
name: talking-head-recut
description: 使用按时间编排、经过设计的 GRAPHIC OVERLAY 卡片包装现有 talking-head / interview / podcast 视频，包括 kinetic titles、lower-thirds、数据标注、引语、侧边面板、picture-in-picture，并与转写稿同步；可选择 16:9 / 9:16 / 4:5 canvas，片段在底下保持原样播放。当用户提到 "graphic overlays"、"on-screen graphics"、"package / dress up my video" 时触发。不是普通字幕（/embedded-captions）。不清楚时 → /hyperframes。
---

# Talking Head Recut

Talking Head Recut 接收一个会**完整播放**的本地视频，并在其上叠加一系列
按时间编排、经过设计的**图形卡片**，包括标题、lower-thirds、数据标注、
引语、侧边面板、picture-in-picture，并与正在讲述的内容同步。agent
设计这些卡片（时间 + 内容），并**直接在对话中编写每张卡片的 HTML**，
然后组装成一个 composition HTML，并通过 `hyperframes` 渲染为 MP4。
这里没有固定的 archetype 列表，也没有规定的卡片结构，overlay 应从
转写稿实际表达的内容中自然生长出来。

> **构建前先确认路线。** 此技能用于给**现有 talking-head 片段**添加**经过设计的图形卡片**（标题、lower-thirds、数据标注、引语、侧边面板、PiP）。如果用户想要**普通 captions / subtitles**（把口播文字显示为文本）→ `/embedded-captions`；**单个短的无旁白**元素（一个 logo sting / lower-third）→ `/motion-graphics`。**片段本身保持原样播放**，重新计时、调色、重构图、重排序或音频处理属于 NLE 编辑，**不在范围内**。从 URL / topic / PR 构建 → creation workflows。不确定 overlays-vs-captions？**先阅读 `/hyperframes`。**

> **`embedded-captions` 的图形包装姊妹技能。** Captions 会把 _spoken words_
> 作为可读字幕添加；此技能会在播放中的视频之上添加 _designed graphics_。
> 普通字幕 → `embedded-captions`。从零构建视频 → creation
> workflows（`product-launch-video` / `faceless-explainer` / …）。

工作目录中可检查的中间文件：

- `metadata.json` — duration / width / height / fps
- `audio.mp3` — 提取出的音频
- `transcript.json` — 扁平**词数组** `[{ text, start, end }, …]`（Whisper；没有 `segments`，没有 `words` wrapper）
- `storyboard.json` — 轻量级卡片大纲（agent 的计划）
- `public/cards/card-XX.html` — 每张卡片一个 HTML 片段
- `public/index.html` — 最终组装的 composition
- `output.mp4` — 渲染后的视频

## CLI Resolution

```bash
# hyperframes — transcription (local Whisper) + rendering the assembled HTML to MP4
npx hyperframes --help
```

此技能完全基于 **hyperframes** CLI 以及系统 `ffmpeg` / `ffprobe` 运行。
转写通过 `hyperframes transcribe` 在本地运行 **Whisper**，不需要第三方
服务、API key 或有速率限制的代理。

## Workflow

### 1. Check Environment

```bash
npx hyperframes doctor          # ffmpeg, headless browser, render deps
# confirm bundled assets:
ls "<SKILL_DIR>/assets/fonts" "<SKILL_DIR>/assets/vendor/gsap.min.js"
```

必需：

- `ffmpeg` / `ffprobe`（系统）
- `<SKILL_DIR>/assets/fonts/*.woff2`, `<SKILL_DIR>/assets/vendor/gsap.min.js`（打包在此技能中，在 Step 9 暂存到工作目录）

转写不需要 key，`hyperframes transcribe` 会在本地运行 Whisper（Step 4）。

在 macOS 上强烈建议为 `hyperframes render` 设置：

```bash
export PRODUCER_BROWSER_GPU_MODE=hardware
```

### 2. Create a Work Directory

所有产物都放在 `videos/<project-name>/` 下，这与其他视频工作流
（`product-launch-video` / `faceless-explainer` / `pr-to-video`）
使用同一约定。保持 cwd 位于 workspace root；下面所有内容都会写入这一
个子目录。

```bash
VIDEO_PATH="/absolute/path/input.mp4"
WORK_DIR="videos/$(basename "$VIDEO_PATH" | sed 's/\.[^.]*$//')"
mkdir -p "$WORK_DIR"
```

### 3. Extract Audio and Metadata

```bash
# metadata — duration / width / height / fps
ffprobe -v error -select_streams v:0 \
  -show_entries stream=width,height,r_frame_rate \
  -show_entries format=duration -of json "$VIDEO_PATH" > "$WORK_DIR/metadata.json"
# audio
ffmpeg -y -i "$VIDEO_PATH" -vn -acodec libmp3lame -q:a 2 "$WORK_DIR/audio.mp3"
```

输出：`metadata.json`（读取 `width`/`height`/`duration`；fps = 对
`r_frame_rate` 分数求值，例如 `30000/1001 → 29.97`）+ `audio.mp3`。

### 4. Transcribe

```bash
npx hyperframes transcribe "$WORK_DIR/audio.mp3" -d "$WORK_DIR" --json --model small.en
```

本地 **Whisper**，没有 API key、没有代理、没有速率限制。它会向工作目录
写入一个词级别的 `transcript.json`（每个词的 `text` + `start` / `end`
时间戳）。读取它，获取 Step 6 中驱动卡片 timing 的词 / 句子时间；如果
需要 segment-level chunks，请根据标点 / 停顿自行把词组合成句子。

**夹紧到媒体时长。** Whisper 可能会返回略微超过实际片段长度的最后一个
词 `end`，请把每张卡片的 `endSec` 和 `composition.durationSeconds`
都夹紧到 `metadata.json` 的 duration，否则渲染会在视频之后显示黑尾。

### 5. Correct Transcript

`transcript.json` 是一个**扁平的词对象数组**，形如 `[{ "text": "...", "start": s, "end": s }, …]`（没有 `segments` 数组，没有 `words` wrapper；每个词的键是 **`text`**）。读取它并修正明显的 ASR 错误：

- 同音词、产品名、技术术语、标点
- 就地编辑某个词的 `text`；**保留其 `start` / `end`** 时间戳
- 没有预先分组的 `segments` 数组；当你需要用于卡片 timing 的 segment-level chunks 时，**自行把词组合成句子**（按终止标点 / 停顿切分）

### 6. Draft a Lightweight Storyboard (in chat)

**不涉及 CLI。** 读取 `transcript.json` + `metadata.json` 并直接设计
卡片。`storyboard.json` 是 agent 内部的规划产物，不会被任何 CLI 命令
消费；它存在的目的是让你在编写每张卡片 HTML 之前，清晰思考 timing 和
content。保持其形状与下面示例一致，这样同一份大纲可以驱动你在 Step 9
中编写的 composition：

```json
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

**必需的 Card 字段：**

| field                   | type                                       | purpose                                                        |
| ----------------------- | ------------------------------------------ | -------------------------------------------------------------- |
| `id`                    | string                                     | 用于 card HTML 和 GSAP selectors 的稳定 id                    |
| `intent`                | string                                     | 自然语言描述；提供给卡片 synthesis                            |
| `startSec` / `endSec`   | number                                     | 秒为单位的时间（endSec > startSec）                           |
| `accentIndex`           | 0 \| 1 \| 2 \| 3 \| 4                      | 此卡片使用 5 个 theme accent colors 中的哪一个                |
| `zone`                  | enum (see below)                           | 卡片位于 canvas 的哪个区域                                    |
| `contentHints`          | object                                     | 自由形式包；agent 把 kicker/title/detail/data/quote 放在这里  |
| `archetype` (optional)  | string                                     | 可附加的自由形式标签，用来记住卡片模式；缺省 = free-form，也是默认 |
| `transition` (optional) | enum: `cut` \| `fade` \| `slide` \| `wipe` | 声明式 card-to-card transition                                |

**五个 `zone` 值：**

| zone              | resolved bounds                                | when to use                 |
| ----------------- | ---------------------------------------------- | --------------------------- |
| `fullscreen`      | 覆盖整个 canvas                                | hero moments、大数字、mantras |
| `whiteboard-area` | 40px margin 内缩（或 portrait height 的 45%）  | 密集数据 / 注释内容         |
| `lower-third`     | 底部 30% band                                  | 在可见视频上的注释          |
| `side-panel`      | 右侧 42%（landscape）或底部 40%（portrait）    | 数据侧边，视频在另一侧      |
| `video-overlay`   | 全 canvas，预期 mostly-transparent card        | full-bleed video 上的注释 overlay |

在 Step 9 组装 composition 时，按照上表把每张卡片的 `zone`
解析为 card-host wrapper 上的像素 bounds。Video bounds 在 composition level
只设置**一次**（`videoTrack.bounds`）；如果要让视频看起来在卡片之间“移动”，
请在 composition 的 `<script>` 中针对 `#video-wrap` 编写 GSAP tweens
（见 Step 9）。

**没有规定的卡片角色，也没有规定的叙事弧线。** 卡片应从视频实际表达的
内容中产生，可以全是引语或全是数据，可以以一个数字开场，也可以以一个
故事开场。让转写稿驱动节奏。

**需要多少个 takeaways？—— 根据时长 + 密度自动推断。** 没有固定上限。
先根据视频时长选一个 **base pace**，再根据 **information density** 调整。
只有 **floor 是固定的：最少 5 张卡片**，让短视频也有节奏。

**Step 1 — base pace by duration**（中等密度下自然的 sec/card）：

| video duration     | base pace (sec per card) | rationale                    |
| ------------------ | ------------------------ | ---------------------------- |
| < 60s (short reel) | **6–8s**                 | 短视频观众期待更快切换       |
| 60s – 3 min        | **8–12s**                | 常规社交媒体节奏             |
| 3 – 10 min         | **12–20s**               | 留出呼吸空间；每张卡片承载更多 |
| 10 – 30 min        | **20–35s**               | 长篇 lecture / interview 节奏 |
| > 30 min           | **30–60s**               | episodic，接近章节感         |

**Step 2 — density multiplier**（乘到 base pace 上）：

| signal in the transcript                                                                                  | multiplier | effect                 |
| --------------------------------------------------------------------------------------------------------- | ---------- | ---------------------- |
| **High density** — 数字很多、明确 claims 很多、节奏断奏、列表式枚举、每 1–2 句就是一个新想法             | **× 0.7**  | 切得更快，卡片更多     |
| **Medium density** — 数据与叙事混合的流动                                                                  | **× 1.0**  | base pace              |
| **Low density** — 一个延展故事、重复 reframing、缓慢反思节奏、单一论点逐步展开                            | **× 1.5**  | 切得更慢，卡片更少     |

**Step 3 — compute:**

```
secPerCard = basePace × densityMultiplier
cardCount  = max(5, round(videoDurationSec / secPerCard))
```

示例（注意：**没有上限 clamp**；长视频会自然产生更多卡片）：

- **30s reel, single punchline (low density)** → 7 × 1.5 = 10.5s/card → round(30/10.5)=3 → floor to **5** cards
- **60s reflective monologue (low density)** → 10 × 1.5 = 15s/card → **4** → floor to **5** cards
- **121s talking-head with rich data (high density)** → 10 × 0.7 = 7s/card → **17** cards
- **5 min interview, mixed density** → 16 × 1.0 = 16s/card → **19** cards
- **10 min deep-dive, high density** → 16 × 0.7 = 11s/card → **55** cards
- **30 min lecture, medium density** → 28 × 1.0 = 28s/card → **64** cards
- **1 hr podcast, low density** → 45 × 1.5 = 67.5s/card → **53** cards

当一张卡片停留超过约 15s 时，计划做一张更丰富的卡片（数据块、
multi-step reveal、若干 sub-points 以 staggered animations 展开）；
一个静态 one-liner 超过 8s 会变无聊。对于很多卡片超过 30s 的长内容，
考虑把 timeline **chunking 成 sub-compositions**（每章一个 .html，并用
`data-composition-src` 挂载），这样每个文件中的 GSAP timeline 保持可管理；
参见 `timeline_track_too_dense` HyperFrames lint warning。

`content` 可以是普通字符串（"Title: annualized 5.69%\nNotes: ..."）或任何能捕捉数据的 JSON
形状。agent 按卡片决定形状。

**Optional outro.** 此技能不附带**固定 brand outro**。如果用户需要 closing card，
你可以自行设计一个中性的（wordmark + one-line tagline，约 1.5-2s，
fade in -> short hold -> fade out），把它追加到 `cards[]`，并把
`composition.durationSeconds` 延长到其 `endSec`。否则在最后一张内容卡片处结束。

### 7. Decide Render Strategy

#### Confirm Visual Direction with User (DO THIS FIRST)

在开始设计卡片或决定 bounds 之前，**请用户选择输出比例、layout、style 和
card-density preset**。Frames 会根据所选 layout × style 组合自动选择
（见下面的 “Auto-pick frame” 表）。发送问题前，**预先计算两件事**：

1. 根据源视频宽高比（`metadata.json` width / height）计算
   **`recommendedRatio`**：
   - `sourceAspect = width / height`
   - `sourceAspect ≥ 1.5` (≥ ~3:2 wide) → 推荐 **`16:9`**
   - `sourceAspect ≤ 0.7` (≤ ~9:13 tall) → 推荐 **`9:16`**
   - `0.7 < sourceAspect < 1.5` (near-square) → 推荐 **`4:5`**

   在推荐选项标签后标注 " (recommended · matches source video X:Y)"，
   让用户明白为什么推荐。

2. 根据 Step 6 计算 **`autoCount`**（`max(5, round(videoSec / (basePace ×
densityMultiplier)))`），这样 “auto” 选项标签可以显示具体数字。

**环境兼容性 —— 选择可用的最佳提问通道。** 并非每个 runtime 都暴露同样的
结构化提问工具。按以下顺序应用：

1. **`AskUserQuestion`**（Claude Code, Anthropic Console）—— 使用下面的
   structured 4-question call。
2. **其他原生澄清工具**（例如 `ask_question`,
   `request_user_input`, IDE-specific prompt）—— 使用该工具并保持相同的
   4 个问题文本和选项列表。保留推荐标记和预先计算的值。
3. **没有原生工具**（Codex CLI, plain text-only runtimes）—— **直接用普通对话提问**。
   使用本节末尾的 plain-text template。保持为**一条消息、4 个编号问题**
   （全局上限是每轮 2–5 个问题；这里符合）。

适用于所有通道的规则：

- 每轮**最多问 2–5 个问题**。这里 4 个问题符合。
- 即使缺失信息不阻塞渲染，也要**询问一次以确认会实质影响最终输出的参数**
  （ratio、layout、style、cardCount）。
- 如果用户已经预先批准 defaults（"just use defaults"、"no need to ask"、
  "auto-pick everything"）或要求你不要提问，**完全跳过问题**，并使用：
  `recommendedRatio`、`layout="stack"`（最稳妥的跨比例默认）、
  `style` 从 transcript tone 中选择最中性的组（editorial/data），
  `autoCount`。用一句话告诉用户你选择了什么，然后继续。

**Channel A — native `AskUserQuestion`:**

```
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

**About "Other"** —— `AskUserQuestion` 会自动向 card count question 添加一个
"Other" 选项。用户可以直接输入一个数字（例如 "8"、"20"）作为 cardCount
目标。把输入解析为整数：如果解析成功 → 使用该值（最小以 5 为 floor）；
如果解析失败 → 回退到 "auto"。

**Channel B — plain-text fallback**（Codex CLI、没有原生问题工具的 runtimes）。
把下面作为一条普通消息发送，然后等待回复。Bullet-style 1/2/3/4 让回复更易解析：

```
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

解析 plain-text reply：

- 接受宽松格式：`"1A 2C 3B 4A"`、`"A C B A"`、`"16:9 / pip /
data / auto"`、完整句子或 `default`。
- 如果任何回答含糊 → 只重新询问含糊项（仍在 2–5 上限内）。
- 如果用户说 "default / auto / use all recommendations" → 跳过，不再追问。

用户回答后（任意通道）：

1. 根据 ratio answer **解析输出 canvas**，这些就是要写入的精确
   `storyboard.composition.width / height` 值：

   | user choice | composition.width × height | storyboard.layout field                                       |
   | ----------- | -------------------------- | ------------------------------------------------------------- |
   | `16:9`      | **1920 × 1080**            | `"landscape"`                                                 |
   | `9:16`      | **1080 × 1920**            | `"portrait"`                                                  |
   | `4:5`       | **1080 × 1350**            | `"portrait"`（schema 把 4:5 视作 portrait，即 height > width） |

   对于 **`references/layouts/*.html` 内的 4:5 bounds**，这些文件只记录
   landscape（1920×1080）和 portrait（1080×1920）。对于 4:5（1080×1350），
   通过**从 portrait 按比例缩放**来推导 bounds：保留水平值，垂直值按
   `1350/1920 ≈ 0.703` 缩放。例如：`overlay` portrait card =
   `{ x: 24, y: 1280, w: 1032, h: 564 }` → 4:5 card =
   `{ x: 24, y: round(1280 × 0.703), w: 1032, h: round(564 × 0.703) }`
   = `{ x: 24, y: 900, w: 1032, h: 397 }`。

2. 通过查看 transcript tone **把 style group 映射到一个具体 style**；
   选择最贴合的，但必须留在用户选择的组内。如果你在同一组内的两个具体
   style 之间犹豫，发送第二个 `AskUserQuestion`，提供 2–4 个具体 style
   选项。

3. 根据 density answer **解析最终 cardCount**：

   | user choice             | final cardCount                           |
   | ----------------------- | ----------------------------------------- |
   | Auto (recommended)      | 你已经计算出的 `autoCount`               |
   | Fewer                   | `max(5, round(autoCount × 0.6))`          |
   | More                    | `round(autoCount × 1.5)`（无上限 clamp） |
   | Other = "<n>" (integer) | `max(5, parseInt(n))`                     |
   | Other = anything else   | 回退到 `autoCount`                       |

4. **从此表自动选择 video frame**（frames 不询问用户，随 layout × style 而定）：

   | layout    | warm-paper styles (academic / whiteboard / editorial / xhs) | clinical styles (audit / swiss / terminal / minimal) | experimental styles (geom / spotlight) |
   | --------- | ----------------------------------------------------------- | ---------------------------------------------------- | -------------------------------------- |
   | `split`   | `polaroid`                                                  | `hairline`                                           | `clean`                                |
   | `stack`   | `polaroid`                                                  | `hairline`                                           | `clean`                                |
   | `pip`     | `clean` (pip pill already has chrome)                       | `clean`                                              | `clean`                                |
   | `overlay` | `clean` (full-bleed forbids deco frames)                    | `clean`                                              | `clean`                                |

5. **用一句话告诉用户你选择了什么**：ratio（+ canvas size）、layout、
   specific style、frame 和 final cardCount，然后继续 Step 7 的其余部分
   （per-card layouts、motion patterns）。
6. 在 working memory 中记录这五个值（ratio / layout / style / frame /
   cardCount，无需 schema field）；在 Step 8 编写每张卡片 HTML，以及读取匹配的
   `references/<dim>/<key>.html` 以获取 tokens 和 structure 时会引用它们。

如果用户通过 "Other" 选择了一个不在 10-style library 中的自由文本 style name，
把它视作自行设计新鲜卡片视觉的提示，但仍以所选 layout 的 bounds 为锚点。

#### Render Strategy Inputs

当 Step 7.0 中的 ratio / layout / style / cardCount / frame 锁定后，
剩余的 per-card 决策是：

- **GSAP target 内源视频的 fit**：video element 使用
  `object-fit: cover`，并被裁切到 `#video-wrap` 的 tween bounds 中。
  如果你不想裁切（例如 portrait source 放在 landscape canvas 上时不应裁掉上下），
  让 tween 指向一个匹配源视频宽高比的 rect，并让周围 canvas 露出
  （或用卡片 / backdrop 填充）。
- **每张卡片的 `card.zone`**：从你选定的 composition layout 推导
  （split → side-panel、stack → lower-third、pip → fullscreen、overlay
  → video-overlay），或者为一次性变体选择不同 zone（hero / quote 用
  fullscreen，密集数据用 whiteboard-area）。
- **每张卡片的 `accentIndex`**：每张卡片从 5 个 theme accent colors
  中取一个。跨卡片变化以形成节奏；当两张卡片属于同一个 narrative beat 时，
  复用同一个 index。
- **Motion vocabulary**：从 `data-anim` kinds（见后面的表）中选 2–3 种可重复的
  pattern 并坚持使用，这样 composition 会显得连贯。

从这些 `themeId` palettes 中选择（在 composition `<style>` block 中把它们作为
`--accent-N` / `--bg` / `--text` CSS variables 使用）：

| themeId | accent palette (5 colors)                 | board bg          | text      |
| ------- | ----------------------------------------- | ----------------- | --------- |
| classic | `#1971c2 #e03131 #2f9e44 #e8590c #9c36b5` | `#FFF9E3` (paper) | `#1e1e1e` |
| noir    | `#4cc9f0 #f72585 #4ade80 #fb923c #a78bfa` | `#1a1a1a`         | `#f1f1f1` |
| mint    | `#0077b6 #d62828 #2d6a4f #e76f51 #7209b7` | `#e8faf0`         | `#1b4332` |
| craft   | `#bf5700 #d62728 #6c757d #e9b54a #3d5a80` | `#f6efe1`         | `#2d2d2d` |
| slate   | `#0ea5e9 #ef4444 #22c55e #f97316 #a855f7` | `#1e293b`         | `#f1f5f9` |
| mono    | `#000 #555 #888 #aaa #ccc`                | `#fff`            | `#000`    |

可用字体（位于 `<SKILL_DIR>/assets/fonts/` 的 woff2，在 Step 9 暂存到工作目录）：
`Caveat`（手写体）、`LXGW WenKai TC`（中文手写脚本）、`Inter`（现代 sans）、
`Virgil`（几何手写）。通过 `@font-face` 或直接 `font-family` 引用。

视觉 pattern 灵感方面，`<SKILL_DIR>/references/styles/` 提供 10 个自包含
reference cards（academic / editorial / minimal / spotlight / geom /
whiteboard / audit / terminal / swiss / xhs），可以作为起点复制，但**不要觉得必须受它们限制**。
每张卡片都是你自己的设计。

#### Visual Design Library (<SKILL_DIR>/references/)

除了 composition-level `themeId` 之外，此技能还在 `<SKILL_DIR>/references/`
提供更丰富的 **reference library**，覆盖三个可自由混合的**正交**视觉维度：

```
Style  ×  Layout  ×  VideoFrame
 (10)      (4)         (3)
```

| dimension  | keys                                                                                              | what it decides                                      |
| ---------- | ------------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| **style**  | `academic` `editorial` `minimal` `spotlight` `geom` `whiteboard` `audit` `terminal` `swiss` `xhs` | 卡片的视觉语言，包括 fonts、colors、ornament、card 内部 layout |
| **layout** | `split` `stack` `pip` `overlay`                                                                   | 源视频和卡片如何共享 canvas                         |
| **frame**  | `clean` `hairline` `polaroid`                                                                     | 视频元素周围的装饰 chrome                            |

阅读 `<SKILL_DIR>/references/DESIGN_INDEX.md`
以获取完整矩阵和宽松决策指南（interview / product launch / data analysis /
social clip / technical tutorial / emotional story …）。当你决定使用某个具体
style / layout / frame 时，读取对应文件：

- `references/styles/<key>.html` — 带有该 style 的 CSS tokens
  （colors、fonts、padding、ornament）和 placeholder takeaway 的自包含
  card fragment。复制 `.card[data-card-id="ref-<key>"]` style block，
  把 data-card-id 重命名为你的 card id，用真实 takeaway 替换 placeholder content，
  即可完成。
- `references/layouts/<key>.html` — landscape 和 portrait 的精确
  `videoBounds` + `cardBounds`，并包含可复制粘贴到 `storyboard.json`
  per-card `layout` field 的 JSON snippet。
- `references/frames/<key>.html` — 作为 `#video-wrap` sibling 添加的
  decorative HTML，以及 composition CSS 的 placement instructions。

按卡片选择 `style × layout × frame`；只要 transitions 读起来顺滑，
你可以在卡片之间改变三者。常见节奏：开场 `editorial × overlay × clean`，
数据卡切换到 `audit × split × hairline`，结尾用 `whiteboard × pip × polaroid`。

这 10 个 styles 是技能侧 design tokens，**不是 composition-level themes**；
它们不需要在 `storyboard.composition` 中声明，而是存在于每张卡片的 HTML 内部。
`themeId` field 仍可选择一个 composition-level palette（见上表），用于控制
page-body background 和 video border chrome。

#### Layout Compositions (Card + Video)

每张卡片有两个协同决策定义它如何与源视频共享 canvas：

- **`card.zone`**（在 `storyboard.json` 中声明）—— 5 个 schema values 之一；
  在 Step 9 编写 card-host wrapper 的 inline `style` 时，将其解析为像素 bounds
  （按 Step 6 的表）。
- **此卡片时间窗口内的 `#video-wrap` bounds**（在 composition 的 GSAP timeline
  中以命令式声明）—— agent 会把 `#video-wrap` tween 到每个 layout transition
  的目标 rect。

Schema 不存储 per-card video bounds。`videoTrack.bounds` 是 composition level
的**一次性**字段（默认 full canvas）。视频在卡片之间“移动”完全是写在
`index.html` 中的 GSAP animation。没有 `card.layout` field；此文档早期版本
曾发明过一个，但真实 schema 只有 `card.zone`。

**4 种 composition layouts**（来自 `references/layouts/`）—— 每一种都是把
一个 `zone` 与 `#video-wrap` tween target 配对的 recipe：

| composition layout | recommended `card.zone` | GSAP target for `#video-wrap` (landscape 1920×1080)                       | GSAP target for `#video-wrap` (portrait 1080×1920)                | when to use                          |
| ------------------ | ----------------------- | ------------------------------------------------------------------------- | ----------------------------------------------------------------- | ------------------------------------ |
| `split`            | `side-panel`            | `{ left: 960, top: 0, width: 960, height: 1080 }`                         | `{ left: 0, top: 960, width: 1080, height: 960 }` (bottom half)   | speaker + data side-by-side / 50:50 weight |
| `stack`            | `lower-third`           | `{ left: 14, top: 14, width: 1892, height: 548 }` (top 52%)               | `{ left: 0, top: 0, width: 1080, height: 844 }` (top 44%)         | speaker on top + summary card below  |
| `pip`              | `fullscreen`            | `{ left: 1480, top: 760, width: 400, height: 300 }` + add `.framed` class | `{ left: 690, top: 28, width: 360, height: 203 }` + add `.framed` | content-heavy card + corner pip      |
| `overlay`          | `video-overlay`         | `{ left: 0, top: 0, width: 1920, height: 1080 }` (full-bleed)             | `{ left: 0, top: 0, width: 1080, height: 1920 }`                  | cinematic / dramatic / glass card on full video |

对于 4:5（1080×1350），按 `1350/1920 ≈ 0.703` 缩放 portrait y/h 值
（见 Step 7.0 Channel A / Channel B `recommendedRatio` resolution table）。

**一次性变体的其他 zone values**（仍使用 `card.zone`；没有假的 "layout" field）：

| `zone`            | resolved bounds                                        | common use                       |
| ----------------- | ------------------------------------------------------ | -------------------------------- |
| `fullscreen`      | 覆盖整个 canvas                                        | hero card，video tweens 到 hidden/pip |
| `whiteboard-area` | 40px margin 内缩（landscape）或底部 45%（portrait）    | dense data card，自由 margins    |
| `lower-third`     | 底部 30% band                                          | talking-head annotation          |
| `side-panel`      | 右侧 42%（landscape）或底部 40%（portrait）            | sidebar / "split" recipe         |
| `video-overlay`   | 全 canvas；预期 transparent card root                  | full-bleed video 上的 glass overlay |

你可以按卡片混合 recipes：根据当前时刻选择合适的 `card.zone`，然后为卡片之间的
`#video-wrap` 编写 GSAP tween。

#### Storyboard Render Contract

`storyboard.json` 是 agent 内部规划产物，不会被任何 CLI 命令解析。它存在的
目的是在你编写每张卡片 HTML 之前，让 timing 和 content decisions 明确。
保持下面的 v3-style 形状，这样同一份大纲可以驱动你在 Step 9 中组装的
composition。

必需结构（完整示例见 Step 6）：

- `schemaVersion: 3`
- `composition: { fps, width, height, durationSeconds, layout, themeId, seed }` — 注意 `durationSeconds`/`fps`/`themeId`/`layout` 位于 **`composition` 内部**，不是顶层
- `videoTrack: { sourcePath, startSec, endSec, bounds? }` — video bounds 默认 full canvas
- `subtitles: { enabled, ... }`
- `cards[]` — 每张卡片有 6 个必需字段：`id`, `intent`, `startSec`, `endSec`, `accentIndex`, `zone`, `contentHints`

规则：

- Card times 保持在 `composition.durationSeconds` 内，且除非有意为之，否则不应重叠（重叠时使用 `data-track-index` 控制 z-order）。
- 视觉细节存在于 card HTML fragments（Step 8），**不在** `contentHints` 中。`contentHints` 是你自己的结构化提示，用来设计卡片；渲染出来的外观是 HTML。
- 保持 storyboard shape 稳定；即使没有东西解析它，你也会在编写 Step 8/9 时回读它，一致性可以让 card IDs 和 timing 保持同步。
- agent-side decisions，比如 "I picked overlay × geom × clean"，**不属于** `storyboard.json`；把它们留在 working memory 中，并在编写 card HTML + GSAP tweens 时使用。

**与视频共享 canvas 的卡片要使用透明 card backgrounds。**
当 GSAP tween 让视频在卡片后方/旁边可见时（overlay recipe、pip recipe，
或任何 `card.zone = 'lower-third' | 'video-overlay'` 时刻），卡片的 `.root`
**不得**绘制完整不透明背景，否则会遮挡视频。两种模式：

```css
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

对于 `side-panel`-zone cards（split recipe），card-host 本身已经只占半个
canvas，所以不透明 card bg 可以接受，它只覆盖自己的那一半。

### 8. Write Each Card's HTML

为每张卡片创建 `$WORK_DIR/public/cards/{card-id}.html`。每个文件包含一个遵循
以下 contract 的单根 HTML fragment：

#### Card HTML Contract

```html
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

**硬性规则**（`hyperframes` lint 会拒绝违规）：

- 单个根 `<div class="card" data-card-id="{cardId}">`
- Inline `<style>` 规则必须以前述 scope selector 为前缀
- **禁止 `<script>` tags**
- `src=` / `href=` 中**禁止 external URLs**（无 CDN、无远程 fonts）
- **禁止 inline event handlers**（`onclick=` 等）
- 所有 assets 通过相对路径指向同一个 `public/` directory
- Colors 使用 `var(--accent-N)` 等，以便跨 themes 迁移

**Animations 是声明式的，不是编码式的。** 只使用 `data-anim-*` attributes；
不要编写 `<script>` 来做动画。你会在 Step 9 中把每个 `data-anim-*`
declaration 编译进一个 master GSAP timeline。

#### Card Sizing — Mobile-First in Portrait

这 10 个 `references/styles/*.html` 是为 **1920×1080 landscape** preview
而定尺寸的。当 `storyboard.layout = "portrait"`（1080×1920，社交 / 移动端的
主流情况）时，**放大所有视觉尺寸**。手机离屏幕更近，同样像素数量会比
landscape TV-style canvas 显得更小。

| token                     | landscape baseline | **portrait target** | scale         |
| ------------------------- | ------------------ | ------------------- | ------------- |
| title (h1/h2 hero)        | 64–96px            | **88–132px**        | ×1.35         |
| detail / body             | 24–30px            | **30–40px**         | ×1.30         |
| kicker / chip label       | 14–16px            | **18–22px**         | ×1.30         |
| timecode / meta           | 12–14px            | **16–18px**         | ×1.30         |
| data block primary number | 48–60px            | **64–88px**         | ×1.40         |
| line-height multiplier    | 1.05–1.5           | same                | (don't scale) |

**经验法则：** `portraitPx = round(landscapePx × 1.3)`，然后向下取到接近的
4px 倍数以保持视觉节奏。Hero headlines 可升到 ×1.4；小 meta text 保持在
×1.2，以免拥挤。

Padding 在 portrait 中要**略微缩小**；卡片更窄，landscape 的大 padding
（40–64px）会吃掉太多宽度。portrait 中使用 24–36px 水平 padding。

如果你制作的单张卡片必须同时适配**两种** layouts，优先在 card root 上使用
`@container` query，而不是硬编码尺寸：

```css
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

但对大多数卡片来说，单一 layout choice 就足够；只需选择与 storyboard 的
`layout` field 匹配的尺寸表列。

#### Available `data-anim` Kinds

| kind            | use for             | key params                                                                                      |
| --------------- | ------------------- | ----------------------------------------------------------------------------------------------- |
| `fade-in`       | enter               | `at`, `duration`, `ease?`                                                                       |
| `fade-out`      | exit                | `at`, `duration`, `ease?`                                                                       |
| `slide-in`      | slide enter         | `at`, `duration`, `from=left\|right\|top\|bottom`, `distance`                                   |
| `kinetic-chars` | per-char pop        | `at`, `duration`, `stagger`, `pattern=pop\|fade` — element needs `<span class="char">` children |
| `typewriter`    | per-char fade       | same as kinetic-chars but slower default stagger                                                |
| `count-up`      | animate number      | `at`, `duration`, `from`, `to`, `format=.0f\|.1f\|.2f\|,d`                                      |
| `draw-path`     | SVG path reveal     | `at`, `duration` — element should be a `<path>`                                                 |
| `grow-y`        | bar height          | `at`, `duration`, `target-h` (px) — element starts `height:0`                                   |
| `grow-x`        | bar width           | `at`, `duration`, `target-w` (px) — element starts `width:0`                                    |
| `scale-pop`     | pop entrance        | `at`, `duration`                                                                                |
| `blur-in`       | unfocused → focused | `at`, `duration`                                                                                |
| `mask-reveal`   | clip reveal         | `at`, `duration`, `direction=left\|right\|top\|bottom`                                          |
| `morph-to`      | tween any CSS       | `at`, `duration`, `props='{...JSON...}'`                                                        |

`data-anim-at` 是**相对卡片 startSec 的秒数**。在 Step 9 把每个 declaration
编译进 GSAP timeline 时，加上卡片的 `startSec` 得到绝对时间，并量化到 1/fps。

### 9. Assemble the Composition HTML

暂存 assets 并编写 `$WORK_DIR/public/index.html`：

```bash
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

#### Composition Template

```html
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
      data-composition-id="talking-head-recut"
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
          window.__timelines["talking-head-recut"] = tl;
        })();
      </script>
    </div>
  </body>
</html>
```

#### GSAP Statement Cheat Sheet

把每个 `data-anim` attribute 编译成一个 GSAP statement。时间是
**绝对秒数** = card.startSec + data-anim-at，并量化到 1/fps。
Selector 为 `.card[data-card-id="X"] #elementId`。

| data-anim                       | GSAP statement template                                                                                                                                                                                            |
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

量化：`T = Math.round(absSec * fps) / fps`。在 30fps 下，最小步长是
`1/30 ≈ 0.0333s`；在 JS literal 中四舍五入到 4 位小数（`.toFixed(4)`）即可。

#### Video Framing Reference (per `layout` value)

视频容器的 selector 是 `#video-wrap`。使用 `tl.to('#video-wrap', { ...bounds }, T)`
在卡片之间动画其 bounds。初始 bounds 应作为元素 inline 设置，并匹配 card-01
的 layout。选择 0.5–0.7s 的 transition duration，并使用 `ease: 'power2.inOut'`。

**Decorative frames**（`clean` / `hairline` / `polaroid`）作为
`#video-wrap` 的**sibling** 放置，并随它完成 layout transitions。参见
[`references/frames/`](references/frames/) 中每个 frame 的 placement HTML、
suggested CSS 以及它适配的 layouts。快速规则：`overlay` layout 会抑制
decorative frames（full-bleed video 与 chrome 冲突）；PiP layouts 已有自己的
pill treatment（border-radius + white ring + shadow），所以只在 `split` / `stack`
上添加 decorative frame。

每个 composition layout 下 `#video-wrap` 的 **GSAP target lookup table**
（landscape 1920×1080；portrait 和 4:5 见 `references/layouts/*.html`，
其中列出了三种 ratios）：

| composition layout                   | typical card.zone | `#video-wrap` GSAP target                                                 | extra css class                            |
| ------------------------------------ | ----------------- | ------------------------------------------------------------------------- | ------------------------------------------ |
| `split`                              | `side-panel`      | `{ left: 960, top: 0, width: 960, height: 1080 }`                         | —                                          |
| `stack`                              | `lower-third`     | `{ left: 14, top: 14, width: 1892, height: 548 }` (top 52%)               | —                                          |
| `pip` (bottom-right)                 | `fullscreen`      | `{ left: 1480, top: 760, width: 400, height: 300 }`                       | `pip-pill` (border-radius + ring + shadow) |
| `pip` (top-left)                     | `fullscreen`      | `{ left: 40, top: 40, width: 400, height: 300 }`                          | `pip-pill`                                 |
| `overlay` (video full-bleed)         | `video-overlay`   | `{ left: 0, top: 0, width: 1920, height: 1080 }` (no change from default) | —                                          |
| **hide video** (pure-graphic moment) | `fullscreen`      | `{ opacity: 0 }` (or move off-canvas)                                     | —                                          |

进入或离开 pip moment 时，要切换 pip-pill chrome（border-radius + white ring + drop shadow）：

```js
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

**Card-host bounds 匹配 zone**。使用 Step 6 顶部的表把卡片的 `zone`
解析为像素 bounds，然后写入 card-host 的 inline
`style="left:Xpx;top:Ypx;width:Wpx;
height:Hpx;..."`。对于 `video-overlay` zone（overlay recipe），
card-host 填满整个 canvas，`.card .root` 内的 CSS 决定实际可见卡片位于哪里。

#### HyperFrames Layout / Animation QA Rules

- 先构建每张卡片的静态 hero frame：即卡片完全可见且可读的时刻。
- 确认 video、cards、subtitles/captions 和 diagrams 不会无意重叠。
- 确认被隐藏的视频区域由 frame 裁切，不会在预期 bounds 之外可见。
- 注册一个 paused master timeline 为 `window.__timelines["talking-head-recut"]`。
- Timelines 在 page load 时同步构建；禁止 `async`、`setTimeout`、Promises 或 media `play()` calls。
- 不要在 render paths 中使用 `Math.random()` 或 `Date.now()`。
- 不要使用 `repeat: -1`；根据 video duration 计算有限 repeats。
- 优先使用 GSAP transforms 和 opacity（`x`, `y`, `scale`, `rotation`, `opacity`），而不是 layout properties（`top`, `left`, `width`, `height`）做 motion。
- 动画 wrapper，例如 `#video-wrap`，不要直接动画 video element dimensions。
- 避免在同一时间从多个 timelines 动画同一元素的同一属性。
- 使用 `data-track-index`，不要使用 `data-layer`；使用 `data-duration`，不要使用 `data-end`。
- 每个 timed element（`card-host`、sub-composition 等）都必须在自身 classes 旁包含 `class="clip"`，例如 `class="card-host clip"`。HyperFrames runtime 使用 `.clip` 将 visibility 限制在 `data-start … data-start+data-duration` 窗口内。没有它，元素会在整个视频中可见（lint: `timed_element_missing_clip_class`）。
- 对 body / global `font-family`，列出**具体 font names**（`'Inter', 'Caveat', …`），不要只使用 CSS variable，如 `var(--font-family)`。HyperFrames font resolver 在 static analysis 时不会展开 CSS vars（lint: `font_family_without_font_face`）。Cards 内部仍可使用 `var(--font-family)`，因为它们的 `@font-face` declarations 已加载。

### 10. Render to MP4

```bash
cd "$WORK_DIR"
PRODUCER_BROWSER_GPU_MODE=hardware npx hyperframes render public \
  --skill=talking-head-recut \
  -o output.mp4 \
  --fps 30
```

`hyperframes render <dir>` 读取 `<dir>/index.html` 并生成 MP4。
在 macOS 上强烈建议使用 `PRODUCER_BROWSER_GPU_MODE=hardware` flag
（或 `--browser-gpu`）；software-only Chrome rendering 在大多数笔记本上会超时。

在完整渲染前，可在指定 timestamp 捕获单帧做 sanity check：

```bash
npx hyperframes snapshot public --at 5    # → public/snapshots/frame-00-at-5s.png (a single --at ignores --out)
```

### 11. Report Results

告诉用户：

- Work directory path
- `storyboard.json`（你设计的 card outline）
- `public/cards/*.html`（每张卡片一个 HTML）
- `public/index.html`（组装后的 composition）
- `output.mp4`（最终视频）
- 使用的 ASR provider
- Card count + 你如何选择它们（用 1 句话）
- 任何缺失 keys 或 quality caveats

**Optional live preview (on request only).** 片段在 `public/index.html` 内保持不变播放，
overlays 叠加在上方，因此预览是可信的。**运行期间不要打开它。** 当用户要求时，
在 render 之后启动一个 long-lived server，并报告 URL：

```bash
(cd "$WORK_DIR/public" && npx hyperframes preview)   # or `npx hyperframes play` for a shareable link
```

除非用户要求，否则不要删除工作目录。
