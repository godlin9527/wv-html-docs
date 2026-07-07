---
name: embedded-captions
description: '为 talking-head 视频添加字幕。一个包含 36 种视觉身份的 catalog（CATALOG.md），背后有两个引擎：column-flow（字幕合成到场景内部 - matte occlusion + mix-blend；cream/ink/editorial/keynote/documentary/loud/neon/glitch/chrome/velocity）和 themed constitutions（anchor/ordnance/terminal/neonsign/stardust/stomp/scoreboard/transit/vhs/arcade/dossier/laser/thunder/hologram/biolume/aurora/spectrum/papercut/popup/chalkboard/graffiti/brush/inkwater/ransom/lastpage/nightcity - 例如 glyph-decode 高潮、逐笔写出的 neon sign，或安静的 `anchor` rail 默认方案）。按 identity 路由，绝不按 mode 路由。在出现 "captions/subtitles"、"embed/cinematic captions"、"VFX captions"、"炸/特效/酷炫字幕"、具名 identity，或顶级 motion-graphics 请求时触发。对多数 talking-head 内容来说，把每个词都嵌入是错误的 - `anchor` 是逐字默认方案。本地端到端运行（自行转录并抠出主体，无需 API key）。需要 hyperframes 和单主体片段（multi-shot 片段按镜头拆分）。'
metadata:
  tags: captions, embedded-captions, occlusion, matting, talking-head, rembg-matting, whisper, ffmpeg, cinematic
---

# Embedded Captions

**一个 catalog，开头就选定**（[CATALOG.md](CATALOG.md) - 36 种 identity；背后的引擎只是后端细节）。**Standard**（默认）会构建干净的逐字 **rail**（lower-third subtitle，承载大部分文本）+ 一个在峰值处合成到主体身后、_进入_ 场景的 **embed** 高潮。**Cinematic** 是纯 embed - 没有 rail，每条字幕都合成在主体身后（hero typography、accumulation、occlusion 即效果）。**Theme** 是完整的 themed constitution - body paradigm × hero setpiece × front fx × plate reaction，由 registries（[themes/README.md](themes/README.md)）组合而成：`ordnance` `terminal` `neonsign` `stardust` `stomp`。大多数 explainer / voiceover 都是 **Standard**；**embed 是稀缺、需要挣来的峰值** - 把每个词都嵌入是常见错误；Theme 用于 VFX 级请求（"炸"、"特效"、"像 AE 做的"）。

---

## Operational flow (TL;DR)

下面的工艺说明很长；但**流水线本身很短** - 所有确定性内容都通过计算或编译得到，绝不手写：

1. **Decision gate**（拒绝不合适的片段）→ **从 [CATALOG.md](CATALOG.md) 选择一个 identity**（36 种 identity；engine/compiler 由查表得出 - 绝不暴露 mode/category 问题）
2. `hyperframes init`（如果 project dir 已存在且里面有视频，则跳过 - `matte.cjs`/`transcribe.cjs` 会采用目录中的任意视频作为 source.mp4）→ **`bash scripts/prepare.sh <project>`**（matte ∥ transcribe ∥ audio-envelope 并行，然后使用 scene palette/optics/lighting 生成 safe-zones v2 - 一个命令，不会遗漏）
3. **编写一个小型 JSON 来表达创意选择**（先读 `safe-zones.json`）：
   Cinematic → `plan.json` → `fill-timings.cjs` → `fit-fonts.cjs` → `make-composition.cjs`；
   Theme → `theme.json` → `make-theme.cjs`（rail/panel/poem/takeover paradigms；`anchor` 是安静的 rail 默认方案）
4. **Visual QA**：`node scripts/preview-frames.cjs <project>` → 约 2s/frame 生成忠实 composite 预览（无需 render）。在为 render 付出成本前检查 § Visual QA。
5. `render-and-composite.sh` → gates（timing / occlusion+hero / overflow / hand-off）→ `final.mp4`

容易被忽略的承重规则：

- **rail（默认）+ embed（提升）。** `drop`（填充词，不显示）/ `rail`（逐字 lower-third subtitle，在前景，承载大部分文本）/ `embed`（合成在主体身后的峰值词）。**Standard mode 两者都会做**，只嵌入峰值。见 **§ Caption model**。
- **视频原片保持 UNTOUCHED（Standard/Cinematic；**Theme mode 的 PLATE budget 是唯一批准的例外** - register-gated reaction beats（charge-dim、punch、shake、grain）按 theme DNA 定义，并在 matte composite 之后应用，使 subject+text+plate 作为一整帧运动）** - 唯一新增的是字幕；matte 只是让主体遮挡 embed track。绝不要 grade/recolor/scanline 素材。
- 两本规则书：**rail → [references/rail.md](references/rail.md)**（简明），**embed craft → [references/composition-craft.md](references/composition-craft.md)**（丰富，仅 embed）。按需要略读。

---

## Caption model - rail + embed

每个口播短语属于三类之一：

|           | 内容                                             | 显示方式                                                                                                                                                    |
| --------- | ------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **drop**  | filler - um/uh、结巴、自我修正       | 不显示                                                                                                                                                         |
| **rail**  | 默认 - 普通口播内容（逐字） | 干净的 lower-third subtitle，**在前景**，可读。重点词可以得到行内 `emphasis` 高亮（accent colour / active-word pop）- 它仍留在 rail 上。 |
| **embed** | 被提升的峰值 - headline beat              | 一个大词合成在**主体身后**（matte occlusion），带有设计过的 entrance + exit                                                                        |

**rail 承载大部分文本；embed 是稀缺、需要挣来的峰值。** 稀缺性是**按 beat/block，而不是按 clip**：每个 block（thought）≤1 个 hero，绝不同时可见两个，hero window 之间至少留一个 beat 的空气（低于 0.6s 时 compiler 会警告）。短片段通常 1-2 个；长 explainer 约每节一个。在多个 hero 中，**作者设定的最大那个是 APEX**（只有它会得到完整的 lockup embed + width-fit raise）；较小的是 **MINOR peaks**，作为超大 emphasis lines 搭乘其 column（fg、damped motion）- 不是每个 beat 都需要 matte showcase，这正是让 apex 成为事件的原因。把每个词都嵌入仍然是常见错误。

Rail-surface identities 正是构建这个结构（rail = `rail.html`，embed = `index.html` 中的高潮）。Column-flow identities 会丢掉 rail，让所有内容变成 embed-style - 只在 mood-over-verbatim 请求中推荐它们，绝不要用于必须读清文字的 explainer / voiceover（CATALOG.md 按 identity 编码了这一点）。

---

## Step 0 - pick ONE identity from the CATALOG

**一个前端，背后三个引擎。** 用户从 [CATALOG.md](CATALOG.md) 选择一个 IDENTITY（36 个条目：10 classic + 26 themed）；engine、compiler 和 authoring file 都由 catalog 行查表得出。**绝不要把 "Standard vs Cinematic vs Theme" 作为问题暴露出来** - 这些是后端名称（一个产品即便有多个引擎，也只有一个 UX）。catalog 编码了路由所需的一切：reading surface、voice、recommend-for、scene needs，以及真正相近配对的 adjacency notes（loud↔ordnance、neon↔neonsign、cream↔stardust）。

流程：探查片段 → 从 catalog 中 shortlist 2-3 个 identities → 推荐一个并用一行说明原因 → **用户选择** → 编写该 identity 的文件。Identities 是 engine-locked（不允许交叉组合；打开一个就是 validation event - 见 dna/README.md）。

**始终给出你的推荐，并让用户在你 author 之前选择。** 不要静默默认。

（完整 identity 表在 [CATALOG.md](CATALOG.md) 中 - 它是路由的 single source of truth。下面的 engine docs 描述各后端的 authoring contract。）

**Recommendation heuristic**：使用 [CATALOG.md](CATALOG.md) 中的 "Shortlisting heuristics" - 它们是 identity-level（例如 "炸" shortlist ordnance/stomp/terminal/loud，并按应该爆炸的对象来选），绝不是 category-level。不确定 → `anchor`。

- **Cinematic** → 编写 `plan.json`，用于 locked template，由 `make-composition.cjs` 编译。
- **Theme** → 阅读 [themes/README.md](themes/README.md)，编写 `theme.json`，运行 `scripts/render-theme.sh`（compile + render + plate reaction → **final_fx.mp4**）。

---

## Decision gate - RUN FIRST

在任何 mode 之前，先探查视频并分类场景。

```bash
ffprobe <video.mp4>                    # specs
ffmpeg -ss <t> -i <video.mp4> -vframes 1 sample.png   # at 20/50/80%
```

读取样张。若出现以下情况则拒绝：

- 多个说话人 / hard cuts（拆分并逐镜头 render，或拒绝）
- 没有人类主体（此 skill 用于 talking-head）
- 短于 3 秒、**无 speech**，或脸从未清晰可见 - `transcribe.cjs` 会在音频接近静音时警告（Whisper 会在静音中幻觉出 "Thank you." 之类的词）；**听从警告并拒绝**，不要给伪造词加字幕
- **源视频已有 burned-in captions / subtitles / heavy text graphics** - 添加第二套字幕系统会冲突，并且素材必须原样交付（不覆盖/不 inpainting）。Burned text 常常只在片段中段出现：采样一个 **1fps contact sheet**（`ffmpeg -i in.mp4 -vf "fps=1,scale=160:-1,tile=10x5" sheet.png`），不要相信 3 个抽样帧。
- **Transcript is garbage** - 非母语/重口音 speech 可能被转写成自信的乱码。authoring 前要 sanity-read `transcript.json`；如果它不像语言，尝试一次 `WHISPER_MODEL=medium`，否则拒绝（伪造文字的逐字 rail 比没有字幕更糟）。
- 繁忙手持且快速运动（matte flickers）

### Pre-flight probes（零成本，避免最糟失败）

1. **Shot-cut probe.** 在 20%、50%、80% 采样帧。如果出现不同主体/场景，**在切点前 trim clip**。
2. **Letterbox / pillarbox probe.** 第一帧有黑边？计算 safe content rect，并将字幕位置限制在其中。
3. **Luminance probe.** 采样字幕区域的平均 luminance - `under 60` → 浅色文字可直接读取，`60-180` → 添加 glyph scrim，`180+` → opaque text + scrim（绝不要裸用浅色文字）。**Cinematic templates 是 cream+`screen` 且 LOCKED** - 用这个 probe 来_选择合适 identity_（明亮场景 → `ink`，或 opaque-rail `anchor` theme），而不是给某个 look 重新上色。
4. **Identity recommendation by tone（你推荐；用户选择 - 见 Step 0 + CATALOG.md）。** explainer / interview / must-read words → rail/panel-surface identities；poetic / social / "cinematic" → 按 register 选择 column-flow identities；"炸 / 特效 / VFX" / named worlds → themed identities。不确定 → `anchor`（文字可读，场景安全）- 但要呈现 shortlist 并让用户选择。

---

## Pipeline - 5 steps

```
1. hyperframes init <project> --non-interactive --video <video.mp4>
2. bash scripts/prepare.sh <project>       # matte ∥ transcribe (parallel) → safe-zones. One command.
                                           #   → frames_fg/ transcript.json safe-zones.json
3. [AGENT STEP — the only creative step] author a small JSON; see below by mode
   Cinematic: author plan.json → node scripts/fill-timings.cjs → fit-fonts.cjs → make-composition.cjs
   Theme:     author theme.json → bash scripts/render-theme.sh <project>   (compiles + renders + plate fx)
4. node scripts/preview-frames.cjs <project>   # ~2s/frame composite previews → § Visual QA (BEFORE the render)
5. bash scripts/render-and-composite.sh <project>  # gates → final.mp4 + history/ snapshot
   (Theme mode: SKIP steps 3b/5 — render-theme.sh already runs compile + render-and-composite
    + _postfx.sh; the deliverable is final_fx.mp4, final.mp4 is pre-plate-reaction)
```

Step 1 的 `init` 会检查已安装的 skills 是否与 GitHub 上最新版一致；若有过期项，会更新全局集合。

Step 3 因 mode 而异：

### Step 3 - Cinematic mode（纯 embed）

1. **先读 `safe-zones.json`。** Narration planes 放在 **`zones.hugLeft`/`hugRight`** - 这些是紧贴 silhouette 的干净条带（离身体太远的文字会显得漂浮，而非嵌入；远角是 fallback，不是默认）。hero 默认用 `heroAnchor`/`heroBands.best`（居中压在主体上，约 30-55% occluded）。`recommendation:"fg"` 会把 NARRATION 移到前景以提高可读性；**只要 `heroBands.feasible`，hero 仍保持 embedded** - hero-fg 是最后手段。
2. **DNA 是你在 Step 0 选择的 identity**（CATALOG.md）- 不要在这里重新打开选择。根据场景做 sanity-check（bright hero band luma > 150 需要 `ink`；完整选择指导在 catalog 中，覆盖全部十种 incl. neon / glitch / chrome / velocity）。说明你的选择和原因；由用户决定。DNA 会锁定 type/palette/blend/motion + hero three-act；safe-zones v2（`palette`/`optics`/`lighting`）会自动把它参数化到当前场景。
3. **编写 `<project>/cinematic.json`** - `"dna": "<name>"` + thought-BLOCKS，而不是 raw groups：每个 block = 多行词（在 clause boundaries 处按 2-5 个分组）+ 堆叠所在 plane + 每行 `css`（仅 size/weight/style - 不含 positions）+ 最多一行标记 `"hero": true`（被提升的词；`"text"` 用于显示形态）。Schema：`scripts/make-cinematic.cjs` header。
4. **Compile**：`node scripts/make-cinematic.cjs <project>` - lowers blocks → plan.json → index.html。自动生成：transcript-sequenced timings、accumulate-within-block、page-flip-between-blocks、**hero LOCKUP**（hero block 的 pre-context、HERO 和 post-context 作为一个绑定 composition 居中放在主体上 - top→bottom reading order 按构造等于口播顺序；context 浮在 FRONT，同时 hero 嵌入 BEHIND = depth sandwich；mass rule 保证 hero 支配 context）、apex/minor hero split、**reading order by construction**、按 safe-zones 的 fg fallback。_（直接手写 plan.json 仍然可用于 blocks 无法表达的设计 - 然后自行运行 `fill-timings.cjs` + `fit-fonts.cjs` + `make-composition.cjs`。）_

### Step 3 - Theme mode（themed constitution）

**先阅读 [themes/README.md](themes/README.md)** - paradigm/setpiece registries、linkages、hard rules，以及精确的 `theme.json` schema。

1. **按 content register 选择 theme DNA**（每个 `themes/<name>.json` 都有 `voice` + `when`）。说明你的选择和原因；由用户决定。
2. **编写 `<project>/theme.json`** - `dna`、`lines`（逐字、transcript order；每条 1-5 个词 - 对 `takeover` 来说每条 line 是一个 CARD）、`minors`（emphasis words）、`hero:{match}`（climax word/phrase；对于 embed setpieces，把它从 `lines` 中留出；对于 inline setpieces 和 panel+redact，则保留在里面）。
3. **Render**：`bash scripts/render-theme.sh <project>` - compiles（compile time 的 verbatim-completeness gate）、renders both layers、composites、applies the plate reaction → `final_fx.mp4`。在 Visual QA 中，在 compile 和 render 之间使用 `preview-frames.cjs`。

---

## Visual QA - preview BEFORE you render

`node scripts/preview-frames.cjs <project> [t…]` 会以约 2s each 合成**忠实 preview frames**（caption layers 在 seek-time 截图 + 真实 video frame + matte occlusion + rail overlay = 最终 composite 在该时刻的样子）。默认 samples = 每个 group/climax window。完整 render 要花几分钟 - 绝不要用它来_发现_ layout problems。

检查 previews（`<project>/preview/sheet.png`）是否符合这份清单 - 这些是 geometric gates **无法**捕捉的失败：

1. **Washout** - 浅色文字压在明亮区域（window/sign/sky）上：不可读 → 移动 plane 或更换 DNA/mode（bright scene → `ink`）。
2. **Text-on-text** - 字幕压到场景自身文字/图形上，或两个 caption groups 互相碰撞。
3. **Reading order** - 屏幕上的垂直顺序必须匹配口播顺序；hero 不能位于后续词的下方。
4. **Hero presence** - climax 应该很大，并且明显位于主体身后（约 30-55% occluded），而不是 margin 里的漂浮标签。
5. **Balance** - 一个连贯的 column/band，而不是散落碎片；margins 有呼吸；没有任何内容被裁切。

然后执行 [references/reference-bar.md](references/reference-bar.md) 中的 **5 positive checks**（poster test · timid test · one-glance hierarchy · scene handshake · dead-air audit）- 失败清单确保 render 不坏；positive list 才让它像是_被设计过_。两者都通过后再交付。

**Fresh-eyes review（推荐用于任何面向用户的内容）：** 你对自己的 layout 会有 confirmation bias。如果能 spawn subagent，就只给它 preview sheet + 这份 checklist，并要求它按帧给出 PASS/FIX verdicts（"review these caption previews against the 5-point checklist; answer PASS or the specific fix per frame"）。在 plan.json / theme.json 中应用修复，recompile，re-preview - 每轮只花几秒。只在 previews 通过时 render 一次。

---

## The DNA registry - ten visual languages（替代 template catalog）

两种 mode 都从 **[dna/](dna/README.md)** 取材 - 十种 art-directed visual languages，它们会**按场景参数化**（从素材采样 accent、沿测得的 light direction 添加 contact shadow、depth-match blur、RMS-coupled hero amplitude）：

| DNA             | Register       | Scene fit                                       | Voice                                                                                              |
| --------------- | -------------- | ----------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| **cream**       | premium-warm   | dark/mid warm scenes                            | Inter + warm cream + screen；glowing emergence hero（cinematic-cream 的继任者）                 |
| **ink**         | premium        | **bright scenes (luma > 150)**                  | near-black multiply - type printed ON the wall；bright-scene answer                            |
| **editorial**   | editorial-luxe | introspective / fashion / poetic                | Bodoni Moda、lowercase-italic hero - magazine elegance                                             |
| **keynote**     | tech-premium   | product / launch                                | opaque white Inter 800、dead-center stillness                                                      |
| **documentary** | formal         | interview / serious                             | burn-in reveals、no hero - gravitas IS the style                                                   |
| **loud**        | loud           | hype / sport / social                           | Anton + scene-sampled accent、single-unit slam + ripple；body ANNOUNCES in front（`bodyLayer: fg`） |
| **neon**        | loud-cyber     | cyberpunk / nightlife / tech-noir (dark scenes) | electric-cyan signage、ignition flicker、hero powers ON like a sign                            |
| **glitch**      | loud-cyber     | digital / hacker / AI                           | RGB-split echoes snap together on landing；machine-percussive timing                               |
| **chrome**      | loud-luxe      | Y2K / fashion-tech / music                      | liquid-metal gradient hero + one sheen sweep during the hold                                       |
| **velocity**    | loud-sport     | sport / auto / fitness                          | every word arrives along its motion vector（streak+skew），hero passes with speed trails            |

按 `safe-zones.json`（`heroAnchor.bandLuma`、`palette.temperature`）× content register 选择 - [dna/README.md](dna/README.md) 有 decision rule。Authoring：`cinematic.json` 接收 `"dna": "<name>"`。

引擎会根据 DNA 生成 **hero three-act**（无需 authoring）：co-visible captions dim（setup）→ per-letter entrance with amplitude ∝ spoken loudness（impact）→ breathe + glow until exit（afterglow）。

（Legacy：`plan.template:"cinematic-cream"` 会自动映射到 `dna:"cream"`。退役的 54-template library 位于 skill 之外的 `~/Downloads/embedded-captions-archive/standard-templates-54/`；`_motion.md` 仍保留在 skill 内，作为 motion-verb reference catalog。）

---

## Aesthetic decision - tone × shot × platform（输入 catalog shortlist，而不是第二个 router）

按 3 个轴对片段分类，并把结果输入 CATALOG.md 的 shortlisting - 本节本身绝不选择 mode/engine：

**Tone**（内容是什么感觉？）

- documentary | conversational | energetic | poetic | keynote | investigative | music-video

**Shot**（构图是什么？）

- close-up (head + shoulders) | mid-shot (torso+) | wide (full body+) | cut-montage (mixed shots)

**Platform**（播放在哪里？）

- 9:16 portrait (TikTok/IG/Shorts) | 16:9 landscape (YouTube/web) | 1:1 square | broadcast export

在 [references/direction-catalog.md § Classification matrix](references/direction-catalog.md) 中交叉引用 direction language - 然后回到 [CATALOG.md](CATALOG.md) shortlist identities（这个 matrix 只影响 shortlist；catalog 是唯一 routing surface）。

## Composition craft（embed track）- read before embedding

完整的 **embed-track** playbook 位于 **[references/composition-craft.md](references/composition-craft.md)**：transcript role-annotation、phrase grouping、planes & clean-zone anchoring、zone coherence、climax pop & readability、edge-breathing、occlusion 3-step judgement，以及 accumulation/persistence。它规定一个_被提升_短语如何坐进场景中 - 在 authoring 任何 embed（Cinematic `plan.json` 或 Standard `index.html`）之前都要阅读。默认 **rail** track 有自己更简单的 spec → **[references/rail.md](references/rail.md)**。

---

## Shared knowledge

| Doc                                                                      | What                                                                                                                               |
| ------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| [references/rail.md](references/rail.md)                                 | **rail track** - standard lower-third subtitle spec（默认；承载大部分文本）。                                          |
| [references/composition-craft.md](references/composition-craft.md)       | **embed-track playbook** - grouping、planes、climax pop、occlusion judgement、accumulation/persistence。嵌入前阅读。 |
| [dna/README.md](dna/README.md)                                           | **DNA registry** - 十种 scene-parameterized visual languages；如何选择。                                                      |
| [references/reference-bar.md](references/reference-bar.md)               | **taste bar** - per-register world-class references + 5 positive checks。                                                   |
| [references/aesthetic-principles.md](references/aesthetic-principles.md) | **18 rules.** 在品味上胜过 Veed AI。先读。                                                                               |
| [references/motion-vocabulary.md](references/motion-vocabulary.md)       | 10 个具名 motion primitives + tone→timing lookup                                                                                    |
| [references/direction-catalog.md](references/direction-catalog.md)       | 10 个 ship-ready aesthetics + tone×shot×platform matrix                                                                               |
| [references/anti-patterns.md](references/anti-patterns.md)               | 已锁定的 bugs（CoreML、letter-spacing reflow 等）                                                                      |
| [references/scene-types.md](references/scene-types.md)                   | wall surface 何时可用（4 conditions）                                                                                       |
| [references/layout-heuristics.md](references/layout-heuristics.md)       | Plane positioning、clean-zone selection、crown 3 conditions、pillarbox math                                                        |
| [references/typography-presets.md](references/typography-presets.md)     | Font-size × column-width matrix（起点）                                                                                  |
| [references/caption-grouping.md](references/caption-grouping.md)         | Word → group rules（pauses、sentence boundaries）                                                                                   |
| [references/failure-modes.md](references/failure-modes.md)               | 开发陷阱长尾                                                                                                           |
| [references/bespoke-vs-presets.md](references/bespoke-vs-presets.md)     | presets 有时失败的原因；clone-and-tweak pattern                                                                                |

**先阅读 aesthetic principles 和 direction catalog。** 其他一切都是实现细节。

---

## Non-negotiables

- **脸绝不能被连续 100% 覆盖** - 每个 0.3s window 中，face bbox 至少 30% 未被覆盖。
- **WCAG contrast** - final render 会 lint；失败则修正 palette。
- **Deterministic** - 禁用 `Math.random()`、`Date.now()`、`repeat:-1`。
- **Never grade/recolor the video.** 素材原样交付 - 唯一新增的是字幕。不要在 a-roll 上做 full-frame scanlines / duotone / darken / vignette。Cyberpunk/CRT texture 应该存在于_caption element 内部_，而不是覆盖整帧。
- **Rail-first for talking-head / explainer.** 不要嵌入整份 transcript - 大部分文本属于 rail；只嵌入 peaks。嵌入全部内容是默认错误。
- **Embed is scarce + spaced.** 每个 sentence/beat ≤1 个 embed，绝不相邻或同时可见，至少隔一个 beat，最多一个 `apex`。climax = 每个 beat 的峰值，**不是**"整段 clip 的唯一 payoff"。
- **Matte = the PERSON（hyperframes `remove-background`，u2net_human_seg，Apache-2.0）。** 目标是 human segmentation，但不是外科级：细长且偏离的家具（mic boom arms）通常会被排除 - 字幕会渲染在其上方、人物身后 - 而主体附近的大型显著物体（望远镜、桌面设备）仍可能泄漏进 matte 并遮挡字幕。主体手持的物体（products、phones）可能间歇性掉出，让字幕穿到前面。绝不要假设：放置 hero 前，在 2-3 个 timestamp 采样 `frames_fg/`，并优先选择避开任何泄漏家具的 hero 位置（`heroAnchor` 可能被泄漏物体 skew - 对照 frames_bg 交叉检查）。
- **safe-zones is PROP-BLIND - 目视检查你使用的每个 band。** Zones/heroBands 只评分 _subject_ occlusion + luma：位于 "clean" zone 内的 mic、telescope 或 screen 对它们不可见（并且泄漏到 matte 中的 prop 会把 `heroAnchor.centerXPct` 从人物上 skew 走）。authoring 前，为你打算使用的每个 band 提取一帧；如果那里有 prop，测量其 bbox 并移动/缩小 plane。两个真实案例之所以能干净交付，只是因为 agent 正是这么做的。（Auto prop-saliency 是已知缺口；zones 的 `peakLuma` 只捕捉_移动的_亮物体。）
- **Captions stay on-frame.** Cinematic mode 硬性 gate frame-overflow；Standard mode 将 `check-overflow.cjs` 作为 WARNING 运行（intentional bleed 是唯一例外 - 阅读 warning）。
- **每条 caption 在屏幕上 ≥ 0.5s** - 更短则不可读。
- **Word timings 必须与 transcript.json 在 80ms 内匹配** - 字幕 off-beat 500ms 会摧毁场景幻觉。Cinematic 在 rendering 前（通过 render-and-composite.sh）运行 `check-timing.cjs --strict`；THEME mode 在 compile time 改为强制执行相同 timings（make-theme 的 sequential transcript matcher + verbatim completeness gate - drift 是 compile error）。绝不要把多个 transcript words 塞进一个 entry（例如 `"FUTURE OF"`，或一个带 line-break 的 `IT` + `ALL` stack 却只有一个 start/end）- 第二个词会继承第一个词的 timestamp 并提前触发。把它们拆成各自拥有 timings 的 separate word entries，即使你想让它们位于同一条 visual line 上（用 CSS `white-space` / natural wrap，而不是 `<br>`）。支持 caption text ≠ transcript 的 creative substitutions（例如用 `"15%"` 替代 `"fifteen percent"`）- 在 `check-timing.cjs` 内的 `CREATIVE_SUBS` 注册它们。
- **Group windows must envelop their words** - 对每个 group，`group.in ≤ min(word.start)` 且 `group.out ≥ max(word.end)`。如果 `group.in` 晚于某个 word 的 start，该 word 会被静默延迟到 container mounts（我们曾因此交付过 800ms lag bugs）。validator 会强制执行。
- **没有两个 caption groups 可以在 time 和 screen region 两者上重叠** - 时间重叠的字幕会造成 text-on-text pileups。选项：(a) **spatial separation** - 把每个 group 放入不重叠的 vertical band，使其可以共存（memory-wall cascade style）；(b) **handoff** - 设置前一个 group 的 `out` ≤ 下一个 group 的 `in`，使屏幕上只有一个；(c) **deliberate layered typography** - 在其中一个 group 上添加 `"allow_overlap": true` 来静默 validator。validator 会根据 CSS 估计每个 group 的 vertical bbox 并标出 collisions。默认选 (a) - 这会让 cinematic-cream 感觉像一首累积的诗，而不是一个不断替换自己的 subtitle track。
- **Screen-blend 在明亮背景（>180 luminance）上会失败。** **Cinematic** templates 是 cream + `screen`，且该 DNA 是 **locked**（plan 不能 recolour 它们）→ 在明亮背景上会 wash out，因此选择 `ink`（专为明亮表面构建的 letterpress）或 `anchor` theme（opaque rail surface），而不是 override 某个 look。
- **不要在 word entrance 上动画 `letter-spacing` 或 `filter:blur`** - inline-block reflow 会导致 line-jumps。
- **CoreML banned for matting** - onnxruntime CoreML EP 的 mixed-precision partitioning 破坏了 face alpha（在之前的 RVM engine 中观察到；不要重试）。Matting 仅 CPU（约 2 fps @1080p ≈ 每 10s clip 2-3 min；长片段要预留预算）。

---

## Dependencies

- **hyperframes**，已 build（`packages/cli/dist/cli.js`）。Scripts 会自动解析 checkout：`HYPERFRAMES_ROOT` env → 如果此 skill 位于 hyperframes 内，则 repo root → `~/Downloads/hyperframes`。用 `bun install && bun run build` 构建。
- **Node-first；两个 Python touchpoints 通过 `uvx`（无需手动安装）：** transcription 通过 `uvx` 运行 WhisperX（word-level timings；按 SKILL §transcription fallback），Theme 的 `drawon` setpiece 在 compile time shell 调用 `python3 scripts/gen-stroke-path.py`。其他一切都运行在 hyperframes 已经自带的 toolchain 上：通过 hyperframes CLI 的 **`remove-background`** 做 matting（u2net_human_seg；weights 首次自动下载，约 168 MB，到 `~/.cache/hyperframes/`）、通过 **`sharp`** 做 image/alpha math、通过 **`puppeteer`** 做 layout/occlusion/overflow，外加 **`ffmpeg`**。scripts 会从 hyperframes checkout 自动解析这些工具 - 不需要额外安装。
- **Transcription = WhisperX via `uvx`**（word-level timings + alignment；无需手动安装 - `transcribe.cjs` 驱动 `uvx whisperx`）。如果已有 word-level `transcript.json`，则 fallback 使用它。
- **Source video** - `matte.cjs` / `transcribe.cjs` 自动解析 `source.mp4`（或 glob clip / 读取 `hyperframes.json`），所以 `hyperframes init --video X.mp4` 不需要手动重命名。
- **fps** - `matte.cjs` 以 source native rate 提取并记录 `matte.fps`；`render-and-composite.sh` 使用它，使 matte 保持 frame-aligned。
- Matting weights 不随包提供：`matte.cjs` 会 shell 调用 hyperframes CLI 的 `remove-background`，它会将 u2net_human_seg（约 168 MB，Apache-2.0）首次下载到 `~/.cache/hyperframes/background-removal/models/`。新机器上的第一次 prepare 需要网络完成这一次下载。

如果缺少 hard dependency，STOP 并询问用户 - 不要静默跳过步骤。
