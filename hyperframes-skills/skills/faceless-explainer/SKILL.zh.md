---
name: faceless-explainer
description: "将任意文本 — 文章、笔记、主题、brief — 转换成 faceless explainer video：没有要 capture 的网站或素材，因此视觉按场景发明（typography、abstract graphics、diagrams、data-viz）。用于 topic explainers、concept breakdowns、how-tos、listicles。不是 product promo (/product-launch-video) 或 site tour (/website-to-video)。不清楚 → /hyperframes。"
---

> **media-use**: 在寻找音频/图片之前，调用 `/media-use` 从 HeyGen catalog 解析 BGM/SFX/images。先运行 `--adopt` 来注册现有 assets。参见 `/media-use` skill。

# Faceless Explainer to HyperFrames

使用这个 skill 将一段文本转换成 explainer video：选择 design system，规划教学故事，并在 HyperFrames 中逐帧构建。**Faceless** 意味着每个视觉都是下游发明出来的 — 没有 capture step，也没有真实 asset inventory。

> **Step 0 前先确认路线。** 你是编排器。运行每一步，验证其 gate，然后才继续。这个 skill 用于 **从文本解释一个主题，没有产品，也没有要 capture 的网站**。将其他意图路由到别处：product launch/promo → `/product-launch-video`；真实网站 tour → `/website-to-video`；GitHub PR → `/pr-to-video`；给现有 footage 加 captions → `/embedded-captions`；短的无旁白 motion graphic → `/motion-graphics`。如果用户只说 "make a video" 或路线不确定，先阅读 `/hyperframes`。

你是编排器。在 `videos/<project>/` 中工作。按顺序运行步骤，并在继续前通过每个 gate。需要用户确认的步骤是 Step 0、Step 3 和 Step 6。除了 Step 5 之外，每一步都由你自己完成；Step 5 中为每个 frame 派发一个 sub-agent。不要把 design 或 motion 规则写在这里；那些属于 frame-worker sub-agent、本 skill 本地的 `../hyperframes-animation/rules/` + `../hyperframes-animation/blueprints/`，以及 `hyperframes-creative`。

工作流：Step 0 setup → `hyperframes.json`；Step 1 brief → `capture/extracted/`；Step 2 design system → `frame.md`；Step 3 storyboard/script → `STORYBOARD.md` and `SCRIPT.md`；Step 3.1 audio → `audio_meta.json`；Step 4 visual design → enriched `STORYBOARD.md`；Step 5 frames → `compositions/frames/NN-*.html` and `index.html`；Step 6 final render → `renders/video.mp4`。

---

## Step 0: Setup and Brief

目标：锁定核心 video brief，并在需要时创建 HyperFrames project。

仅当 `hyperframes.json` 缺失时初始化。从主题中用 kebab-case 命名 `<project>`，例如 `compound-interest-explained`；绝不要使用 workspace name 或 timestamp。

`npx hyperframes init "videos/<project>" --non-interactive --example=blank` — `init` 会检查已安装 skills 是否落后于 GitHub 最新版本，如果有过期项则更新 global set。

**在 brief 前展示 sign-in status** — 运行 `npx hyperframes auth status` 并 **逐字转述其输出（不要 paraphrase 或 rewrite）。** 它会报告 voice/BGM 是否使用 HeyGen 或 local engines，并在未登录时说明如何登录。**如果未登录，停止并等待用户选择 — 登录，或说 "go"/"offline" 继续使用 local engines — 然后再询问 brief 或其他任何内容。** 把它当作真正的决策点，而不是顺带提示；不要把这个选择折进 brief question，也不要把 keys 写入每个 repo 的 `.env`。（在 autonomous mode 中，记录状态并继续 offline。）参见 `../media-use` → Preflight 的 canonical guidance。

**Gate:** `hyperframes.json` 存在，并且 angle、length、aspect ratio 和 language 已锁定；sign-in status 已展示（已登录，或继续 offline）。

---

## Step 1: Brief (no capture)

目标：将用户文本纳入项目，作为信息来源。这里 **没有 website capture，也没有 real assets** — 这是 faceless explainer。

逐字保存用户的完整输入，然后手动创建 synthetic capture package：

- `capture/extracted/visible-text.txt` — 完整 article / notes / topic / brief，逐字保存。这是 **information** 来源，而不是 story template（Step 3 会重塑它）。
- `capture/extracted/tokens.json` — `{ "title": "", "description": "", "colors": [], "fonts": [] }`。从 brief 填入 `title`/`description`。除非用户明确给了 brand colors 或 fonts，否则保持 `colors`/`fonts` 为空 — 如果给了，就添加进去（design preset 无论如何都会提供完整 palette）。

如果用户粘贴了 script 或希望保留其措辞，逐字保存为 `user_script.txt`，问一次 "use it verbatim or restructure?"，并将答案作为 `VO_MODE` 存给 Step 3。

**不要** 运行 `npx hyperframes capture`（没有 URL）。不要创建 `asset-descriptions.md` 或填充 `capture/assets/` — faceless visuals 在 Steps 4-5 中发明，不是 capture。唯一例外：如果用户提供了真实图片，把它放在 `public/<basename>` 下，并在 Step 3 记录。

**Gate:** `capture/extracted/visible-text.txt` 和 `capture/extracted/tokens.json` 存在；你能用一句清楚的话说明 explainer 的 topic 和 audience。

---

## Step 2: Design System

目标：选择一个随附的 frame preset；脚本会把它转换成这个视频的 `frame.md` + caption skin。

你只做一个判断 — **哪个 preset**。阅读 `../hyperframes-creative/references/design-spec.md` 并浏览 `../hyperframes-creative/frame-presets/`；选择视觉最适合 topic、tone 和 audience 的 preset。然后运行：

```bash
node <SKILL_DIR>/scripts/build-frame.mjs --preset <name> --hyperframes .
```

脚本会确定性地完成其余工作：复制 preset 的 `FRAME.md` → `frame.md`，并将它 **remixes** 到 `capture/extracted/tokens.json` 中的任何 brand tokens 上（brand colors 按角色映射到 preset 的 color keys；preset 的 display + body fonts 替换为品牌字体），把 preset 的 caption skin 复制到 `.hyperframes/caption-skin.html`，并自我验证（mapping 损坏时 exits 1）。它 exits 0 后即可继续 — 不要手工编辑 spec。

faceless explainer 通常 **没有 brand colors/fonts**（`tokens.json` colors/fonts 为空）→ 脚本保留 preset 自己的 palette，这是一个完整可交付的 design。只有当用户明确给出 brand colors/fonts 时，才在运行前将其添加到 `tokens.json`，并且只有在 mapping 确实需要时，才在之后手动调整 `frame.md`。

**Gate:** `build-frame.mjs` exited 0 — `frame.md` 来自 named preset，并且（当 preset 随附 caption skin 时）`.hyperframes/caption-skin.html` 作为 caption skin source 存在。

---

## Step 3: Storyboard and Script

目标：将文本转换成已批准的逐帧教学计划。

阅读 `references/story-design.md`、`../hyperframes-animation/blueprints-index.md`、`../hyperframes-core/references/storyboard-format.md` 和 `../hyperframes-core/references/script-format.md`。用它们编写 `STORYBOARD.md`，并在需要旁白时编写 `SCRIPT.md`。

使用 `story-design.md` 处理 explainer structure（concept / how-to / listicle / story）、hook strategy、clarity techniques、emotional beats、type-enum mapping 和 `VO_MODE`。视频序列来自 **narrative design，而不是输入文本的段落顺序** — 重排、合并、省略、压缩。作为 **soft guide**，参考 `../hyperframes-animation/blueprints-index.md` 中的 role→blueprint menu：对每个 beat，按候选 blueprint 暗示的形状编写 voiceover，并在适配时标记该候选 `blueprint:` id。Teaching truth 仍然决定哪些 beats 存在 — 绝不要强行让 beat 套进 blueprint，也不要仅因为有成熟形状可用就发明 beat。Faceless visuals 是下游发明的，所以 frames 不携带 asset inventory：除非用户提供了真实 `public/<basename>` image，否则保持 `asset_candidates` 为空。使用 storyboard 和 script references 中要求的精确字段。

起草后，展示逐帧 summary。在同一条消息中问用户两件事：(a) 批准或请求修改，(b) 是否想要 storyboard scaffold 的 live preview（`npx hyperframes preview`）— 只有回答 yes 才打开。迭代直到批准，并将 preview choice 带到 Step 6。

**Gate:** `STORYBOARD.md` 存在，每个 frame 都有必需的 narrative fields，需要旁白时 `SCRIPT.md` 存在，并且用户批准了逐帧计划。

---

## Step 3.1: Audio

目标：从已批准脚本生成 narration、word timings、music 和 audio metadata。

Step 3 批准后开始 audio。让它在后台运行，然后继续 Step 4。（sign-in status 已在 Step 0 展示；engine 会自动 fallback。）

`node <SKILL_DIR>/scripts/audio.mjs --script ./SCRIPT.md --storyboard ./STORYBOARD.md --hyperframes . --out ./audio_meta.json &`

audio script 处理 narration、word timings、从 HeyGen music library 查找 BGM，以及 timing metadata。BGM mood 来自 storyboard 的 `music:` field。这里使用 HeyGen Audio API 做 retrieval，而不是 generation，并使用与 TTS 相同的 `~/.heygen` credential。provider details 见 `../media-use/audio/references/tts.md`。

如果没有 narration 且没有 `SCRIPT.md`，跳过 voice generation。如果 storyboard 有 music mood，BGM 仍可能运行。

**Gate:** audio job 已启动，或项目被标记为 silent。

---

## Step 4: Frame Visual Design

目标：向每个 storyboard frame 添加 visual direction、layout intent 和 motion choices。

就地编辑 `STORYBOARD.md`。不要创建另一个 storyboard。将 `frame.md` 作为 color、type、layout feel 和 style 的事实来源。

阅读 `references/visual-design.md`、`../hyperframes-animation/blueprints-index.md`、`references/motion-language.md` 和 `../hyperframes-animation/rules-index.md`。使用 `visual-design.md` 中的方法（time-coded shot sequence、inline Layout vocabulary 和 invented-visual treatment），以及必需的 `## Video direction` block。使用 `../hyperframes-animation/blueprints-index.md` 选择每个 frame 的 shot shape。使用 `motion-language.md`（motion vocabulary + motion doctrine）和 `../hyperframes-animation/rules-index.md`（valid rule names）处理 motion — 不要发明 motion names。

对每个 frame，按 `visual-design.md` 的方法，将 **time-coded shot sequence** 写入 `STORYBOARD.md`：选择 frame 的 blueprint（或 compose），用此 frame 的 **invented** content 实例化它，并将每个 Scene 的 reveal 配合 voiceover 进行 pacing，让 frame 在完整 duration 中展开，而不是前置加载后冻结。因为 explainer 是 faceless，`focal`/`roles` 命名的是 **invented visual elements**（hero word、diagram node、data-viz series）— 你是在设计它们，而不是选择 captured assets。按 Scene **inline** 写明 layout 和 motion（vocabularies 在 `visual-design.md` 和 `motion-language.md` 中）。添加一个全视频的 `## Video direction` block。

不要改变 story、script、`transition_in` 或 source text。不要在这一步写 HTML。这里 **没有 asset-staging step** — faceless visuals 由 Step 5 的 workers 构建。如果用户提供了真实 `public/<basename>` image，在相关 frame 的 `focal`/`roles` 中按路径引用它；否则没有任何东西需要 staging。

**Gate:** 每个 frame 都有与 voiceover pacing 对齐的 time-coded shot sequence（不能 front-loading）；每个 frame 都命名了它的 invented `focal` 和/或 `roles`；`## Video direction` 存在。

---

## Step 5: Build Frames

目标：将每个 storyboard frame 构建为 HTML composition，并组装可播放视频。

如果 audio 已启动，等待 Step 3.1 audio 完成。然后 sync durations 并 fetch SFX；如果 silent，则都跳过。

`node <SKILL_DIR>/scripts/audio.mjs sync-durations --audio-meta ./audio_meta.json --storyboard ./STORYBOARD.md`

`node <SKILL_DIR>/scripts/audio.mjs fetch-sfx --storyboard ./STORYBOARD.md --hyperframes .`

Duration sync 是机械操作：真实 voice duration 优先；silent frames 保持 estimates；绝不要手工编辑 synced durations。

派发前，阅读 `sub-agents/frame-worker.md` 和 `../hyperframes-core/references/subagent-dispatch.md`。为每个 frame 派发一个 sub-agent，尽可能并行；否则分批运行 workers。每个 worker 只处理一个 frame。

每个 worker context 必须包含 `PROJECT_DIR`、`frame_id`、canvas size、caption status，以及启用 captions 时的 keep-out band，还有 `RULES_DIR`，其值是本 skill 的 `../hyperframes-animation/rules/` 的绝对路径。每个 worker 读取 `frame.md`、自己在 `STORYBOARD.md` 中的 `## Frame N` block、每个被引用 motion 的本地 rule recipe（`../hyperframes-animation/rules/<id>.md`），以及 frame 的 blueprint template（`../hyperframes-animation/blueprints/<id>.md`）。每个 worker 只写 `compositions/frames/NN-*.html`。Workers 绝不能编辑 `STORYBOARD.md`。

**Full-bleed backgrounds ride on a `class="clip"` layer, never the `#root`.** Frame 的 ground（color field / gradient / grid）是它自己的 full-duration background clip — 在 `#root` / `data-composition-id` element 上设置的 `background` 会被 clip-gated 到 frame window，不是可靠 ground，因此 dark content 可能落到黑色 host `body` 上并渲染不可见。视频 base ground 由 assembler 根据 `frame.md` 的 `canvas` color 绘制到 index `#root`。（完整规则 + self-check：`sub-agents/frame-worker.md`。）

每个 worker 返回后，orchestrator 将该 frame 在 `STORYBOARD.md` 中标记为 `animated`。

audio timings 存在后，在后台构建 captions 并 assemble index：

`node <SKILL_DIR>/scripts/captions.mjs build --storyboard ./STORYBOARD.md --audio-meta ./audio_meta.json --hyperframes . --out ./caption_groups.json &`

`node <SKILL_DIR>/scripts/assemble-index.mjs --storyboard ./STORYBOARD.md --hyperframes .`

`captions.mjs` 使用项目的 `.hyperframes/caption-skin.html`（Step 2 中复制）作为 caption look，并注入 `frame.md` 中的 brand tokens；如果没有 skin，则渲染 built-in default pill。`captions: skipped (<reason>)` 是有效状态。明确跳过 captions 时，继续执行。

**Gate:** 每个 frame 都标记为 `animated`，`index.html` 存在，并且 captions 已构建或明确跳过。

---

## Step 6: Finalize

目标：验证 assembled video，获得用户批准，并 render final MP4。

注入 transitions，运行 checks，暂停等待 review，然后 render。

`node <SKILL_DIR>/scripts/transitions.mjs inject --storyboard ./STORYBOARD.md --hyperframes .`

`node <SKILL_DIR>/scripts/transitions.mjs verify --storyboard ./STORYBOARD.md --index ./index.html`

`npx hyperframes lint`

`npx hyperframes validate`

`npx hyperframes inspect`

`npx hyperframes snapshot --at <frame-midpoints>`

`snapshot` 会把 captured frames 拼成一张 contact sheet（`snapshots/contact-sheet.jpg`）。快速看一眼；如果没有明显损坏，就继续 — 不要在这里停留太久。

如果命令失败，展示 stderr 并停止 — 不要堆叠 recovery commands。自己修复：对 `compositions/frames/NN-*.html` 做最便宜的安全编辑，然后重新运行失败的 check。

**Known false-positive — do not chase it.** `inspect` 可能报告少量约 1–4px 的 `text_box_overflow` errors，位于 **caption** highlight words（selector `#caption-word-*` / `.caption-line`）。caption pill 使用刻意紧凑的 `line-height`（在 `scripts/captions.mjs` 中统一设置），并且 **没有 `overflow:hidden`**，所以较重 display glyph 的 ink 会向 pill 自身 padding 中溢出几个 px — 实际上没有被裁切。把这些视为预期并继续。**不要** 增大 caption `line-height`（它会膨胀 pill，反而更糟）。只有当 `text_box_overflow` 指向 **frame** element（`#el-NN-*`）时才处理，而不是 caption word。

checks 通过后，暂停等待用户 review。视频已经 assembled，可在 Studio 中查看和编辑。Step 3 和 Step 6 之间只管理一次 preview：如果用户之前要求打开，则打开；如果之前拒绝，则再次提供；如果他们已经在 Studio 中 review，就不要再问。

Preview: `npx hyperframes preview`

用户批准后才 render：

`npx hyperframes render --skill=faceless-explainer --quality high --output renders/video.mp4`

render 后不要重新运行 `lint`、`validate`、`inspect` 或 `snapshot`，除非用户要求。

**Gate:** render 前 `lint`、`validate` 和 `inspect` 已通过；用户在 review pause 批准；`renders/video.mp4` 存在。最终回复说明 MP4 path 和 final duration。

---

## Quick Reference

**Formats:** 默认 landscape `1920x1080`；portrait `1080x1920`；square `1080x1080`。在 storyboard frontmatter 中设置一次 format。

**Faceless deltas vs a captured-asset workflow:** 没有 Step 1 capture（synthetic `tokens.json` + `visible-text.txt`）；没有 `asset-descriptions.md`，也没有 `capture/assets/`；Step 4 没有 asset-staging；`asset_candidates` 默认空；每个 visual 都由 Step 5 workers 发明（typography / abstract graphics / diagrams / data-viz）。用户提供的 `public/<basename>` image 是唯一真实 asset path。

**Background scripts:** workflow 在 `scripts/` 下只随附这些：`build-frame` 用于将 frame preset adopt + brand-remixing 到 `frame.md`（+ caption skin）；`audio` 用于 TTS、transcription、BGM、SFX 和 duration syncing；`captions`；`transitions` 用于 inject 和 verify；以及 `assemble-index`。其他都是 `hyperframes` CLI。

可复用、与领域无关的 shot shapes 位于 `../hyperframes-animation/blueprints/`（由 `../hyperframes-animation/blueprints-index.md` 索引）。

| Read                                                                                                                                                        | When                                                           |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| `[../hyperframes-creative/frame-presets/](../hyperframes-creative/frame-presets/)`                                                                          | Step 2: choose and adopt a frame preset.                       |
| `[../hyperframes-creative/references/design-spec.md](../hyperframes-creative/references/design-spec.md)`                                                    | Step 2: apply brand tokens correctly.                          |
| `[references/story-design.md](references/story-design.md)`                                                                                                  | Step 3: plan the explainer story.                              |
| `[../hyperframes-animation/blueprints-index.md](../hyperframes-animation/blueprints-index.md)`                                                              | Step 3: role→blueprint menu. Step 4: pick the shot shape.      |
| `[../hyperframes-core/references/storyboard-format.md](../hyperframes-core/references/storyboard-format.md)`                                                | Step 3: write `STORYBOARD.md`.                                 |
| `[../hyperframes-core/references/script-format.md](../hyperframes-core/references/script-format.md)`                                                        | Step 3: write `SCRIPT.md`.                                     |
| `[../media-use/audio/references/tts.md](../media-use/audio/references/tts.md)`                                                                              | Step 3.1: choose or understand TTS providers and voices.       |
| `[references/visual-design.md](references/visual-design.md)`                                                                                                | Step 4: write the frame's shot sequence (+ Layout vocabulary). |
| `[references/motion-language.md](references/motion-language.md)`                                                                                            | Step 4: the motion vocabulary + the motion doctrine.           |
| `[references/cut-catalog.md](references/cut-catalog.md)`                                                                                                    | Step 4-5: the cut catalog (worker builds within-frame seams).  |
| `[../hyperframes-animation/rules-index.md](../hyperframes-animation/rules-index.md)` + `[../hyperframes-animation/rules/](../hyperframes-animation/rules/)` | Step 5: local rule recipe bodies for the cited motions.        |
| `[sub-agents/frame-worker.md](sub-agents/frame-worker.md)`                                                                                                  | Step 5: dispatch per-frame workers.                            |
| `[../hyperframes-core/references/subagent-dispatch.md](../hyperframes-core/references/subagent-dispatch.md)`                                                | Step 5: dispatch sub-agents safely.                            |
