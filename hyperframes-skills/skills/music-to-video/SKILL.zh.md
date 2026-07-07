---
name: music-to-video
description: "将一段音乐轨道（音频文件、从视频中提取的音频，或根据情绪 brief 生成的曲目）转换为与节拍同步的视频 —— 歌词视频、幻灯片或动态宣传片。音乐驱动所有节奏；任何用户提供的图片/视频都会切到同一个节拍网格上，并且完整视频可以不需要任何素材。带旁白的作品 → 与输入匹配的 workflow（见 /hyperframes）。不明确 → /hyperframes。"
---

# music-to-video — 一个基于音乐、与节拍同步的视频 workflow

使用这个 skill 将一条**音乐轨道**转换为与节拍同步的 HyperFrames 视频。你只分析一次曲目，规划 frames，填写逐 frame 计划，并把每个 frame 构建为一个 composition。输入是一条音乐轨道，以及可选的用户图片或视频 —— **没有旁白，也没有网站捕获**。排版和模板是最低保障（完整视频可以零素材完成）；用户提供的任何媒体都会按同一个节拍网格剪入。

你是**编排者**。在 `videos/<project>/` 中工作。按顺序运行步骤，并在继续之前通过每个 **Gate**。两个步骤需要用户：**Step 3**（计划审批）和 **Step 6**（渲染审批）。除了 **Step 4** 以外，所有步骤都由你自己完成；Step 4 中你要为每个 frame 派发**一个 sub-agent**。不要把设计和动效规则放进这个文件 —— 它们位于 `references/` 和 `frame-worker` sub-agent 中。

`SKILL_DIR` = 这个 skill 目录。`PROJECT_DIR` = `videos/<project-name>/`。

Workflow: Step 0 setup → `hyperframes.json` + `assets/bgm.mp3`; Step 1 analyze → `audiomap.json`; Step 2 skeleton → `STORYBOARD.md`（frames，groups 为 `TBD`）；Step 3 plan → 完整的 `STORYBOARD.md` + `frame.md`; Step 4 build → `compositions/frames/NN-*.html`; Step 5 assemble → `index.html`; Step 6 render → `renders/video.mp4`。

## 塑造一切的两个理念

- **一个分析器，并且信任它。** `analyze-beatgrid.py` 是唯一的节拍分析器 —— 不要用其他工具或凭耳朵重新测量节拍。它的 energy / density / rolls / onsets / silences 始终可靠。它的 `bpm` 和 `beats_sec` **只有在音乐真正有节奏时**才可靠；在平静音乐中，网格是 tracker 强加的节拍器，因此应按乐句和能量来节奏化，绝不要硬切到这个网格。判断属于哪种情况，是每个 frame 的 `pacing`（Step 2）。
- **一个 frame = 一个文件；groups 存在于其中。** Step 2 将曲目切成 **frames**，每个 frame 都成为一个 composition 文件 `compositions/frames/NN-<frame_id>.html`，由一个 frame-worker 构建。一个 frame 可以再细分为 **groups**（每个 group 是一个模板或 motion-primitives 组合）。额外密度放在 group _内部_，所以 **frame 数量跟随不同处理方式，而不是跟随节拍** —— 快节奏曲目不会让 sub-agent 数量爆炸。

---

## Step 0: Setup, BGM, and inputs

Goal: 确定音乐来源，创建 HyperFrames 项目，并记录用户提供的任何媒体。

**音乐是脊梁** —— 在做其他事之前先确定一条曲目。这个 skill 针对**快速、高能 BGM** 调优：强节拍网格驱动剪辑（平静曲目也能用，但按乐句而不是节拍来安排节奏）。如果用户给了音频 —— 一个音乐文件，或一个需要提取音频的视频 —— 就使用它。如果没有，就生成一条：根据用户描述选择情绪（例如 "driving synthwave", "trap beat", "upbeat corporate"），并通过 `/media-use` 生成曲目（`references/bgm.md` —— 有凭据时用 HeyGen retrieval，否则用本地 Lyria / MusicGen；ElevenLabs 或其他生成器也可以）。生成前，运行 `npx hyperframes auth status` 并**逐字转述其输出（不要改写或转述）** —— 它会显示 BGM 来自 HeyGen 还是本地 MusicGen，以及未登录时如何登录。**如果未登录，停止并等待用户选择 —— 登录，或继续离线使用本地 MusicGen —— 然后再生成曲目**；不要把 key 写入每个 repo 的 `.env`。（在 autonomous mode 中，记录状态并继续离线。）标准指导见 `/media-use` → Preflight。无论哪种方式，曲目都落到 `assets/bgm.mp3`。暂存用户提供的任何图片或视频，以便 frames 能按节拍网格把它们编织进去；否则整个视频由排版承载。

**Lyric videos:** 对于与人声同步的歌词，通过 `/media-use` 转录曲目来获取词/行时间，或向用户索取歌词文本并将行放到节拍网格上。

只有在缺少 `hyperframes.json` 时初始化。根据 brief 用 kebab-case 命名 `<project>`，例如 `midnight-drive-loop` —— 不要用时间戳。`init` 会检查已安装 skills 与 GitHub 最新版本，并在任何 skill 过期时更新全局集合。

```bash
npx hyperframes init "videos/<project>" --non-interactive --example=blank
mkdir -p "$PROJECT_DIR/assets" "$PROJECT_DIR/renders"
cp "<user-music>" "$PROJECT_DIR/assets/bgm.mp3"   # extract from a video first if needed
# only if the user gave you images/videos:
node <SKILL_DIR>/scripts/stage-assets.mjs --from <dir> --hyperframes "$PROJECT_DIR" --into public
```

**品牌**（字体 + 调色板）在 Step 3 选择，不在这里选择。不要预先选择流派或曲目类型 —— 素材只是可选原料，流派会从逐 frame 的选择中浮现。

**Gate:** `hyperframes.json` + `assets/bgm.mp3` 存在；aspect / length / fps 以及（如有）素材清单已记录。

---

## Step 1: Analyze the music

Goal: 生成整个视频唯一的标准 timing analysis。

`analyze-beatgrid.py` 是**唯一**节拍分析器 —— 不要用其他工具或凭耳朵重新测量节拍。它读取曲目一次并写入 `audiomap.json`：energy phases（level / density / feel）、onsets + `onset_rate`、rolls、silences、`hard_stops`、`key_moments`、phrases、tempo / grid，以及 `audio.duration_sec`。它是确定性的 —— 同一个文件始终给出同一张 map。大多数字段对任何音乐都可靠；`bpm` 和 `beats_sec` 只有在音乐真正有节奏时可靠，而判断这一点是你在 Step 2 做出的决定。

Prerequisites: Python 3 且可用 `librosa`, `numpy`, `soundfile`。如果 import 失败，把它们安装到当前 Python 环境后再运行分析器：

```bash
python3 -m pip install librosa numpy soundfile
```

```bash
python3 <SKILL_DIR>/scripts/analyze-beatgrid.py "$PROJECT_DIR/assets/bgm.mp3" \
  -o "$PROJECT_DIR/audiomap.json" --print
```

**Gate:** `audiomap.json` 存在；`audio.duration_sec` 已知。

---

## Step 2: Frame skeleton (structure only)

Goal: 读取音乐并布局 frames —— `STORYBOARD.md` 的骨架。

阅读 [`references/frame-skeleton.md`](references/frame-skeleton.md)。你自己把 `audiomap.json` 转换为 `STORYBOARD.md` 的**骨架** —— 没有中间 JSON。在真实音乐变化处切分曲目为 **frames**（`hard_stops`、SURGE / DROP `key_moments`、roll 的边缘、没有 onsets 的一段、大能量跳变），并将每个边界吸附到 audiomap anchor。为每个 frame 设置 `span_sec`、`pacing`（来自 Step 1 信任判断的结论 —— 当网格真实时为 `beat_cut`，当它只是强加到平静音乐上的节拍器时为 `phrase_flow`）、`mood`，以及一行 `feel`（Step 3 用于匹配模板的朴素音乐情境）。这里只做分类和布局：把每个 frame 的 `### Groups` 留为 `TBD (Step 3)`，frontmatter `style` 留空 —— 不要写模板、文案、颜色或字体。预计约 1–6 个 frames。

**Gate:** frames 铺满曲目（第一个从 0 开始，最后一个到 `duration_s`）；每个 frame 都带有 `span_sec` + `pacing` + `mood` + `feel`；所有 `### Groups` 都是 `TBD`；任何地方都没有内容。

---

## Step 3: Fill the plan (user-gated)

Goal: 将骨架变成已获批准、完整的 `STORYBOARD.md`。

阅读 [`references/planning.md`](references/planning.md)、[`storyboard-format.md`](references/storyboard-format.md)、[`template-catalog.md`](references/template-catalog.md)、[`motion-primitive-catalog.md`](references/motion-primitive-catalog.md)，以及 [`montage.md`](references/montage.md)（仅当用户提供素材时）。原地编辑同一个文件，做两件事：

1. **Pick the brand.** 使用 `../hyperframes-creative/references/design-spec.md` 中的表格，从 `../hyperframes-creative/frame-presets/` 选择一个 preset（匹配曲目情绪；**只有它的字体和颜色重要** —— templates 拥有 composition）。将它**不做修改**复制到 `frame.md`，并用其中内容填写 frontmatter `style`（font + ≤4–6 个 swatch palette）。
2. **Fill every frame.** 决定每个 frame 的 groups，并为每个 group 给出处理方式：catalog 中匹配的 template（带绑定 params 和真实 audiomap anchors）、来自 primitive catalog 的 free-compose，或一个**遵守 `pacing`** 的素材处理。写文案。你负责 WHAT（template / primitives + content + anchors）；frame-worker 负责 HOW —— **永远不要在 storyboard 中写 millisecond tweens**。

```bash
node <SKILL_DIR>/scripts/validate-plan.mjs --storyboard "$PROJECT_DIR/STORYBOARD.md" \
  --audiomap "$PROJECT_DIR/audiomap.json" --templates <SKILL_DIR>/references/templates
```

修复每个 `✗`（硬错误：duration mismatch、frames 未铺满曲目、缺少 `src`）；warnings 尽力处理。然后向用户展示逐 frame 摘要并迭代，直到他们批准。

**Gate:** `frame.md` 是逐字 preset 副本；`validate-plan.mjs` 退出码为 0；用户批准计划。

---

## Step 4: Build frames from the plan

Goal: 将每个 frame 构建为自包含的 composition 文件。

创建 `compositions/frames/`。阅读 [`sub-agents/frame-worker.md`](sub-agents/frame-worker.md) 和 `../hyperframes-core/references/subagent-dispatch.md`。为每个 frame 派发**一个 frame-worker**，尽可能并行（否则分批）。每个 worker 只得到一个 frame 和以下上下文：

```text
PROJECT_DIR: <abs path>
frame_id: <NN-frame_id>              # = the frame file stem, e.g. 02-f2; the composition id
Your block: the `## Frame N — <frame_id>` block in PROJECT_DIR/STORYBOARD.md
audiomap: PROJECT_DIR/audiomap.json
frame.md: PROJECT_DIR/frame.md
Materials: for each group, <SKILL_DIR>/references/templates/<id>/index.html (templates) and
           <SKILL_DIR>/references/motion-primitives/<id>/ (free); staged assets/ (asset groups)
Contracts: ../hyperframes-core/references/sub-compositions.md + determinism-rules.md
Canvas: <w>×<h>   Pacing: <beat_cut|phrase_flow>
Write to: PROJECT_DIR/compositions/frames/<frame_id>.html
```

worker fork 被引用的 materials，将每个 anchor 转为 frame-local seconds（`local_t = track_t − span_sec[0]`），用 0ms cuts 控制 groups，并写入一个 seek-safe 的 frame 文件。**worker 永远不运行 `hyperframes` CLI** —— 那些命令作用于组装后的项目，而此时它还不存在，所以会报告错误文件。worker 只按契约写文件然后停止；你在组装后验证（Step 6）。每个 worker 返回时，你可以确认它的文件已落盘。

**Gate:** 每个 frame 都有对应的 `compositions/frames/NN-*.html` 在磁盘上。

---

## Step 5: Assemble

Goal: 将构建好的 frames + BGM 接入可播放的 `index.html`。

`assemble-index.mjs` 是确定性的 —— 不用 subagent，也不用判断。它在每个 frame 文件的累计 `data-start` 处引用它，将 `assets/bgm.mp3` 挂到 track 11，并硬切 frame → frame（frames 无缝铺满曲目，所以**没有 transition injector**）。

```bash
node <SKILL_DIR>/scripts/assemble-index.mjs --storyboard "$PROJECT_DIR/STORYBOARD.md" \
  --hyperframes "$PROJECT_DIR" --audiomap "$PROJECT_DIR/audiomap.json"
```

修复它报告的任何 `✗` —— 缺失或空白 frame 文件意味着该 worker 写了部分文件；重新派发它（Step 4）并重新 assemble。

**Gate:** `index.html` 存在；总时长 == `audiomap.audio.duration_sec`。

---

## Step 6: Verify and render

Goal: 验证组装后的视频，取得用户批准，并渲染最终 MP4。

在**组装后的项目**上运行 CLI —— 这是正确的单位（逐 frame workers 不能运行它）。`lint` 检查结构，`validate` 运行 headless Chrome（捕获 JS 错误和缺失素材），`inspect` 快照 frames。

```bash
( cd "$PROJECT_DIR" && npx hyperframes lint . && npx hyperframes validate . && npx hyperframes inspect . )
```

检查 `t=0`、每个 frame start、最强 DROP / SURGE、每个 `hard_stops[].t`，以及最后一个 frame。失败时，自己做**最便宜且安全的修复**：编辑出问题的 `compositions/frames/NN-*.html`。绝不要为了掩盖同步问题而改变 duration 或 audio timing。通过 gates 后，暂停让用户 review，然后仅在批准后渲染：

```bash
( cd "$PROJECT_DIR" && npx hyperframes render . --skill=music-to-video -q draft -o renders/video.mp4 --fps 30 )
```

**Gate:** `lint` / `validate` / `inspect` 通过；用户批准；`renders/video.mp4` 存在且带音频，duration == `audiomap.audio.duration_sec`。最终回复说明 MP4 路径和时长。

---

## Resume table

| You have                   | Continue from |
| -------------------------- | ------------- |
| `assets/bgm.mp3` only      | Step 1        |
| `audiomap.json`            | Step 2        |
| `STORYBOARD.md` (skeleton) | Step 3        |
| `STORYBOARD.md` (complete) | Step 4        |
| all frame files            | Step 5        |
| `index.html`               | Step 6        |

## Quick Reference

**Formats:** 默认 landscape `1920x1080`；portrait `1080x1920`；square `1080x1080`。在 storyboard frontmatter 中一次性设置 canvas（`canvas: { w, h, fps }`）。

**Scripts** under `scripts/`: `analyze-beatgrid.py`（唯一分析器）、`validate-plan.mjs`（计划检查）、`assemble-index.mjs`（index assembly）、`stage-assets.mjs`（暂存用户媒体）、`lib/storyboard.mjs`（vendored parser）。其他所有内容都是 `hyperframes` CLI。

| Read                                                                                                           | When                                                    |
| -------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| [`references/frame-skeleton.md`](references/frame-skeleton.md)                                                 | Step 2: 读取音乐、布局 frames、设置 pacing              |
| [`references/planning.md`](references/planning.md) · [`storyboard-format.md`](references/storyboard-format.md) | Step 3: 选择品牌、填写每个 frame、写计划                |
| [`references/template-catalog.md`](references/template-catalog.md)                                             | Step 3: 为每个 group 选择 template                      |
| [`references/motion-primitive-catalog.md`](references/motion-primitive-catalog.md)                             | Step 3/4: free-compose 的 L0 recipes                    |
| [`references/montage.md`](references/montage.md)                                                               | Step 3/4: 素材处理（beat-cut / ken-burns）              |
| [`sub-agents/frame-worker.md`](sub-agents/frame-worker.md)                                                     | Step 4: 派发 + 构建一个 frame                           |
| `../hyperframes-core/references/subagent-dispatch.md`                                                          | Step 4: 安全派发 sub-agents                             |
| `../hyperframes-creative/references/design-spec.md`                                                            | Step 3: 选择 preset（品牌）                             |

## Directory layout

```
music-to-video/
  SKILL.md
  references/   frame-skeleton.md · planning.md · storyboard-format.md
                template-catalog.md · motion-primitive-catalog.md · montage.md
                templates/<id>/          { index.html (+ assets/ · program.json) }  ← L1 catalog impls
                motion-primitives/<id>/  { index.html } (+ ../assets/gsap.min.js shared by recipes) ← L0 catalog impls
  scripts/      analyze-beatgrid.py · assemble-index.mjs · validate-plan.mjs · stage-assets.mjs · lib/storyboard.mjs
  sub-agents/   frame-worker.md   ← the one subagent (one per frame)
```
