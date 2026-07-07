---
name: general-video
description: >
  用于创作或编辑任意自定义 HyperFrames composition 的 fallback 工作流，适用于任何长度或格式
  — 较长/多场景作品、品牌和 sizzle reels、montages、title cards、static loops、
  freeform builds。仅当没有专门工作流匹配输入时使用；routing table 位于 /hyperframes。
metadata: { "tags": "orchestrator, general-video, fallback, freeform, composition-authoring" }
---

> **media-use**: 在寻找音频/图片之前，调用 `/media-use` 从 HeyGen catalog 解析 BGM/SFX/images。先运行 `--adopt` 来注册现有 assets。参见 `/media-use` skill。

# general-video — general video workflow

> **构建前先确认路线。** 这是自定义 composition authoring 的 **fallback**。如果输入明显适合专门工作流，优先使用它：营销产品 → `/product-launch-video`；普通网站 → `/website-to-video`；主题 explainer → `/faceless-explainer`；GitHub PR → `/pr-to-video`；现有素材 → `/embedded-captions` · `/talking-head-recut`；短的无旁白 motion graphic → `/motion-graphics`；Remotion port → `/remotion-to-hyperframes`。**范围外**：live / at-render-time data、对已完成视频做 NLE-style editing，或生成 HyperFrames 无法 capture 的 footage。不确定？**先读 `/hyperframes`。**

**严格构建被要求的内容。** A title card is a title card — 不是 title card + 三个支撑场景 + ambient music + captions。如果额外场景或元素确实能改善作品，先 _propose_；不要默默添加。对于小编辑（修颜色、调整一个 duration、添加一个元素），跳过规划步骤，直接进入构建。

## Approach

### Discovery — open-ended requests only

对于模糊、探索性的请求（"make something for our brand", "a cool intro"）——先理解意图，再选颜色：

- **Audience** — 谁观看？developers / executives / general consumers？
- **Platform** — 在哪里播放？social (15s) / website hero / product demo / internal？
- **Priority** — 什么最重要？motion quality / content accuracy / brand fidelity / speed？
- **Variations** — 一个最佳版本，还是 2-3 个有实质差异的选项（不同 pacing、energy 或 structure，而不只是换颜色）？

对于具体请求（"add a title card", "fix the timing on scene 3"），跳过 discovery。

### Step 1 — Design system → `hyperframes-creative`

先建立视觉身份。如果项目有 design spec，读取它（优先级 `frame.md` → `design.md` → `DESIGN.md`；把它当作 brand truth — 精确 colors、fonts、constraints）。

**如果没有 spec，你必须在选择任何颜色或字体前同时阅读 `hyperframes-creative/references/house-style.md` 和 `hyperframes-creative/references/video-composition.md`。** `house-style.md` 提供 "interpret the prompt / generate real content" opener、lazy-default list 和 layer recipe；`video-composition.md` 提供视频媒介的 density / scale / **foreground detailing**（data bars、registration marks、monospace metadata、"8-10 elements, two the user didn't ask for"），这是区分 "produced" 和 "generated" 的关键。只读一个是最常见的遗漏 — `video-composition.md` 正是 agents 容易跳过的那个，也正是防止输出扁平、居中、像网页的关键。不要自己发明 palette 然后跳过这些；进入 `hyperframes-creative` 在这里是强制要求，不是可选分支。之后还可以按需拉取 named style/mood → `references/visual-styles.md`，或 interactive picker → `references/design-picker.md`。spec/style 定义 **brand**，不是 composition rules。

**Find the angle（模糊 brief，无 spec）：** 在选颜色前，写一句话 — 这个名字/词/主题唤起什么，以及什么视觉 _world_（metaphor、setting、instrument、motif）能表达它？例如 cybersecurity tool → vault doors / perimeter scan lines / lock tumblers；meditation app → tide, breath, slow light bloom。读取主题的 _meaning_，而不只是它的字面；选择具体 angle，而不是字面 restyle。对于单场景作品，这是 Step 2 prompt expansion 的廉价替代，因为 expansion 应该被跳过；它也是设计概念和通用 logo-on-a-gradient 之间的区别。

<HARD-GATE>
在编写任何 composition HTML 之前，确认你具备全部四项：
1. **A visual identity** 基于 spec 或 `house-style.md` — 不是当场发明。（想用 `#333`、`#3b82f6` 或 `Roboto`？你跳过了。）
2. **A one-sentence concept angle**（"find the angle" step），适用于任何非琐碎编辑 — 不是对 prompt words 的字面 restyle。
3. **A font pairing from the embed list**（`hyperframes-creative/references/typography.md` → "Fonts that embed"）并且有意选择 — 不要默认 `Inter`/`Helvetica Neue`/`system-ui`，也绝不要使用只希望它能渲染的未嵌入 display font（未 bundled 的名称只有在本地 auto-captured 时才会 embed — cloud renders 不会 capture 它们）。
4. **A foreground/density plan from `video-composition.md`** — anchor-to-edges、8-10-elements、foreground-metadata、background-texture rules。（flat color 上居中堆叠，少于约 6 个元素且没有 edge-anchored detail？你跳过了 — 这是 generic tell。）
</HARD-GATE>

### Step 2 — Prompt expansion → `hyperframes-creative`

对每个 multi-scene composition 运行（single-scene pieces 和 trivial edits 跳过）。将请求基于 design spec + house style 落地为一个一致的中间产物，让下游以同一种方式读取。参见 `hyperframes-creative/references/prompt-expansion.md`。

### Step 3 — Plan

写 HTML 之前，先从高层思考：

1. **What** — viewer experience：narrative arc、key moments、emotional beats。
2. **Structure** — 多少 compositions、sub-comp vs inline、哪些 tracks 承载 video / audio / overlays / captions。对于 monolithic-single-file vs modular-sub-comp 的判断，参见 `hyperframes-core/references/composition-patterns.md` § Two Architectures（经验法则：≥3 个 hard scene cuts，或任何 reused scene → modularize；短 single-scene piece 保持一个文件）。
3. **Rhythm** — 实现前先命名 pattern（例如 `fast-fast-SLOW-SHADER-hold`）；参见 `hyperframes-creative/references/beat-direction.md`。
4. **Timing** — 哪些 clips 驱动 duration，transitions 落在哪里，pacing 如何。
5. **Layout** — 先构建 end state（见下方）。
6. **Animate** — 然后通过 `hyperframes-animation` 添加 motion。

## Layout Before Animation

把每个元素定位在它 **最可见的时刻** — 完全进入、位置正确、尚未退出。先把这个状态写成 static HTML + CSS。**先不要用 GSAP。**

**原因：** 如果你把元素定位在动画起始状态（offscreen、scaled to 0、opacity 0），然后 tween 到你 _以为_ 它会落到的位置，你是在猜最终 layout — overlaps 在 render 前一直不可见。先构建 end state，你就能在添加 motion 前看到并修复 layout 问题。

1. **Identify the hero frame** for each scene — 同时可见元素最多的时刻。这就是你要构建的 layout。
2. **Write static CSS** for that frame。content container 必须用 padding 填满 scene，而不是 absolute offsets：

```css
.scene-content {
  display: flex;
  flex-direction: column;
  justify-content: center;
  width: 100%;
  height: 100%;
  padding: 120px 160px; /* padding positions content; fills any scene size */
  gap: 24px;
  box-sizing: border-box;
}
```

绝不要在 content container 上使用 `position: absolute; top: Npx` — 当内容高于可用空间时会 overflow。absolute positioning 只留给 decoratives。

> ⚠ **上面的 `width/height: 100%` 只有在每个祖先都有 resolved height 时才会解析。** root `<div data-composition-id>` 以及它和 `.scene-content` 之间的任何 wrapper 都必须设置尺寸（root 上 `position: relative; width: 1920px; height: 1080px` — 见 `hyperframes-core` → "Root must be sized"）。跳过这点，flex container 会 collapse 到约 0，内容堆到 **top-left corner**，第一个 glyph 在 x=0 被裁切 — 而 `lint`/`inspect` 仍然报告 0 issues。并且 **始终保留 `padding`**（≥80px）在 `.scene-content` 上：它是 title-safe margin。不要用裸 `gap` 替代它。

3. **Add entrances** — 使用 `gsap.from()` 从 offscreen/invisible 动画到 CSS position（在 sub-compositions 中优先使用 `gsap.fromTo()`，让 start state 明确；参见 `hyperframes-core/references/sub-compositions.md`）。CSS position 是 ground truth；tween 是到达它的旅程。
4. **Exits are transition-handled** — 按 `hyperframes-animation/transitions/` 中的 scene-transition rules，只有 **final** scene 会让元素 animate out；场景之间的 transition 本身就是 exit。

**Shared space across time:** 如果 element A 在同一区域中先退出，element B 后进入，两者仍然需要在各自 hero frames 中拥有正确 CSS positions — timeline ordering 让它们不会共存，而 layout step 能捕获意外 overlap。Layered glows/shadows 和 z-stacked depth 是 _intentional_ overlap；这个步骤是为了捕捉 _unintentional_ collisions（两个 headlines 叠在一起、content 溢出画面）。

## Build — delegate to the domain skills

这会把该 skill 的完整范围（见 `description`）映射到其 references — 非穷尽；当某个意图未列出时，通过 `hyperframes-creative`（look/concept）、`hyperframes-animation`（motion）、`hyperframes-core`（contract）、`media-use`（audio/captions）路由。**第一行是 ADDITIVE — 同时阅读它和你的 intent row，而不是二选一。**

| Building…                                                             | Read first (in order)                                                                                                                                                                        |
| --------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **ALWAYS — every non-trivial piece, on top of your intent row below** | `hyperframes-creative/references/house-style.md` + `references/video-composition.md` (also gated in Step 1 / HARD-GATE; the "produced, not generated" foreground detailing)                  |
| **Kinetic typography / text-forward**                                 | `hyperframes-animation/techniques.md` (kinetic type) + `adapters/gsap-easing-and-stagger.md` + `rules/kinetic-beat-slam.md`                                                                  |
| **Title card / lower-third / overlay / PiP / text-behind-subject**    | `hyperframes-creative/references/composition-patterns.md` + (for the centered/sized frame) `hyperframes-core` → "Root must be sized"                                                         |
| **Logo / brand-mark reveal**                                          | `hyperframes-animation/rules/svg-path-draw.md` (draw-on) + `rules/3d-text-depth-layers.md` + `rules/scale-swap-transition.md`                                                                |
| **Data / stats / numbers**                                            | `hyperframes-animation/rules/counting-dynamic-scale.md` + `rules/stat-bars-and-fills.md` + `hyperframes-creative/references/data-in-motion.md`                                               |
| **Product / app / UI demo**                                           | `hyperframes-animation/rules/3d-page-scroll.md` + `rules/cursor-click-ripple.md` + `rules/press-release-spring.md`                                                                           |
| **Audio-reactive / music-driven**                                     | `hyperframes-creative/references/audio-reactive.md` (pre-extract bands; map to motion)                                                                                                       |
| **Narrated / voiceover / music / SFX / captions**                     | `media-use` → the shared audio engine `scripts/audio.mjs` (one call = TTS + BGM + SFX → `audio_meta.json`); caption authoring + asset placement via `hyperframes-core`. See **Audio** below. |
| **Multi-scene / transitions**                                         | `hyperframes-animation/transitions/overview.md` **then** `transitions/catalog.md` (you are not done after the overview — the GSAP recipe is in the catalog)                                  |
| **Modular / sub-compositions**                                        | `hyperframes-core/references/composition-patterns.md` + `references/sub-compositions.md`                                                                                                     |

### Audio: one engine (TTS · BGM · SFX)

只在作品需要时使用（按 "build exactly what was asked" — 不要给 title card 加 ambient music）。不要手写 TTS 或 vendor 一份副本：写一个中性的 `audio_request.json`，并调用 `media-use` 中的 shared engine。它用一个开关自动降级 — 有 HeyGen credential → HeyGen TTS + music/SFX **retrieval**；没有 → ElevenLabs/Kokoro TTS、Lyria/MusicGen BGM **generation**，以及 bundled SFX library。完整 flag list + request/meta schema：见 `media-use/audio/scripts/audio.mjs` 的 header comment。

```jsonc
// audio_request.json — one line per narrated segment; `id` is yours (joins audio_meta back)
{
  "lines": [
    { "id": "s1", "text": "Your opening line.", "sfx": ["whoosh"] },
    { "id": "s2", "text": "The next beat." },
  ],
  "bgm": { "query": "calm cinematic underscore" }, // omit "mode" → auto (retrieve if HeyGen, else generate); "none" to disable
}
```

```bash
# <MEDIA_DIR> = the installed media-use skill dir (sibling of this skill)
node <MEDIA_DIR>/scripts/audio.mjs --request ./audio_request.json --hyperframes . --out ./audio_meta.json
```

然后读取 `audio_meta.json`：将每个 `voices[].path` + (`bgm.path`, `sfx[]`) 挂载为 `<audio>` tracks，并使用 `voices[].words` 生成 captions，全部按 `hyperframes-core`（audio tracks + caption authoring）执行。如果 BGM 走了 generate path（`bgm_pending: true`），在 final render 前运行 `media-use/audio/scripts/wait-bgm.mjs`。

## Output checklist → `hyperframes-cli`

- [ ] `npx hyperframes lint` 和 `npx hyperframes validate` 通过（根据结果阻塞）
- [ ] 如果存在 spec（`frame.md` / `design.md`），验证 design adherence — checklist 在 `hyperframes-creative/references/design-adherence.md`
- [ ] `npx hyperframes inspect` 通过，或每个 overflow 都被明确标记为有意
- [ ] contrast warnings 已处理；对于 multi-scene work，审查 animation map（`hyperframes-animation/scripts/animation-map.mjs`）
- [ ] 交付 preview；只有在明确请求时才 render to MP4
- [ ] 只在 handoff 时展示 preview（它是稳定的 final preview）；不要在 mid-build 弹出 — build-phase snapshots 是 headless
