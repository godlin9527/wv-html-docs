---
name: product-launch-video
description: "将产品或营销 URL、粘贴的脚本或 brief 转换为产品发布 / 宣传视频 —— SaaS 宣传片、功能 reveal、产品演示、应用和公司发布。用于用户想要营销、发布、推广或 reveal 一个产品的情况；任何商业 URL 默认走这里。不是通用网站导览（/website-to-video）。不明确 → /hyperframes。"
---

> **media-use**: 在获取 audio/images 之前，调用 `/media-use` 从 HeyGen catalog 解析 BGM/SFX/images。先运行 `--adopt` 以注册现有素材。见 `/media-use` skill。

# Product Launch to HyperFrames

使用这个 skill 捕获产品、理解其品牌、规划发布视频，并在 HyperFrames 中逐 frame 构建。

> **Confirm the route before Step 0.** 你是编排者。运行每个步骤，验证其 gate，然后才继续下一步。这个 skill 用于**正在被营销、发布、推广或 reveal 的产品**，包括 "promo for our site" 这类目的为推广的请求。其他意图应路由 elsewhere：通用非发布网站导览 -> `/website-to-video`；没有产品的主题解释 -> `/faceless-explainer`；GitHub PR -> `/pr-to-video`；给现有素材加字幕 -> `/embedded-captions`；短的无旁白 motion graphic -> `/motion-graphics`。如果用户只说 "make a video" 或路由不确定，先阅读 `/hyperframes`。

你是编排者。在 `videos/<project>/` 中工作。按顺序运行步骤，并在继续前通过每个 gate。用户 gate 是 Step 0、Step 3 和 Step 6。除了 Step 5 以外，所有步骤都由你自己完成；Step 5 中你为每个 frame 派发一个 sub-agent。不要把设计或动效规则放在这里；它们位于 frame-worker sub-agent、这个 skill 本地的 `../hyperframes-animation/rules/` + `../hyperframes-animation/blueprints/`，以及 `hyperframes-creative`。

Workflow: Step 0 setup -> `hyperframes.json`; Step 1 capture -> `capture/`; Step 2 design system -> `frame.md`; Step 3 storyboard/script -> `STORYBOARD.md` and `SCRIPT.md`; Step 3.1 audio -> `audio_meta.json`; Step 4 visual design -> enriched `STORYBOARD.md`; Step 5 frames -> `compositions/frames/NN-*.html` and `index.html`; Step 6 final render -> `renders/video.mp4`.

---

## Step 0: Setup and Brief

Goal: 锁定核心视频 brief，并在需要时创建 HyperFrames 项目。

只有在缺少 `hyperframes.json` 时初始化。根据品牌或域名用 kebab-case 命名 `<project>`，例如 `acme-promo`；不要使用 workspace name 或 timestamp。

`npx hyperframes init "videos/<project>" --non-interactive --example=blank` — `init` 会检查已安装 skills 与 GitHub 最新版本，并在任何 skill 过期时更新全局集合。

**Show sign-in status before the brief** — 运行 `npx hyperframes auth status` 并**逐字转述其输出（不要改写或转述）。** 它会报告 voice/BGM 会使用 HeyGen 还是本地引擎，以及未登录时如何登录。**如果未登录，停止并等待用户选择 —— 登录，或说 "go"/"offline" 以继续使用本地引擎 —— 然后再询问 brief 或其他任何内容。** 把它当作真正的决策点，而不是顺带说明；不要把这个选择折进 brief 问题里，也不要把 key 写进每个 repo 的 `.env`。（在 autonomous mode 中，记录状态并继续离线。）标准指导见 `../media-use` → Preflight。

**Gate:** `hyperframes.json` 存在，并且 angle、length、aspect ratio 和 language 已锁定；sign-in status 已展示（已登录，或继续离线）。

---

## Step 1: Capture assets

Goal: 收集视频所需的源材料、品牌信号和可用素材。

分类输入并选择路径。显式 URL -> 捕获它，并用站点生成 narration 和 assets。粘贴的 script/brief -> 逐字保存为 `user_script.txt`，询问一次 "use it verbatim or restructure?"，将答案存为 `VO_MODE`，然后解析 capture target：文本中有 URL -> 使用它；只有品牌名 -> `WebSearch`，一行确认 URL，然后 crawl；没有 URL/site -> no-capture path。

用以下命令 capture: `npx hyperframes capture "<URL>" -o ./capture`

如果存在 `GEMINI_API_KEY`、`GOOGLE_API_KEY` 或 OpenRouter key，capture 会自动把 assets caption 到 `capture/extracted/asset-descriptions.md`。这不是 review gate。没有 vision key 时，使用 DOM context 并继续。

No-capture path: 手动创建 `capture/extracted/tokens.json`、`capture/extracted/visible-text.txt`、`capture/extracted/asset-descriptions.md` 和 `capture/assets/`。`tokens.json` 应为 `{ "title": "", "description": "", "colors": [], "fonts": [] }`；可行时从 brief 填 title/description。`visible-text.txt` 包含完整 brief 或 script。`asset-descriptions.md` 应说明没有捕获素材，除非用户给了素材说明。

**Gate:** `capture/extracted/tokens.json`、`capture/extracted/visible-text.txt`、`capture/extracted/asset-descriptions.md` 和 `capture/assets/` 存在；你能用一句清晰的话说明品牌。把 `asset-descriptions.md` 当作主要素材清单。如果真实 capture 后它缺失，停止并报告 capture incomplete。如果 `capture/BLOCKED.md` 存在，按其内容处理。

---

## Step 2: Design System

Goal: 选择一个随附的 frame preset；脚本会把它变成这个视频的 `frame.md` + caption skin。

你做唯一的判断 —— **选择哪个 preset**。阅读 `../hyperframes-creative/references/design-spec.md` 并选择最符合品牌和 brief 的 preset。然后运行：

```bash
node <SKILL_DIR>/scripts/build-frame.mjs --preset <name> --hyperframes .
```

脚本确定性完成剩余工作：复制 preset 的 `FRAME.md` → `frame.md`，并将其**remix** 到 `capture/extracted/tokens.json` 中的品牌 tokens 上（按角色把品牌颜色映射到 preset 的 color keys —— ink、canvas、accents —— 保留 keys/structure/components；将 preset 的 display + body fonts 换成品牌字体），把 preset 的 caption skin 复制到 `.hyperframes/caption-skin.html`，并自验证（映射损坏时退出 1）。一旦它退出 0 就进入下一步 —— 不要手动编辑 spec。

没有品牌 colors/fonts 的 `tokens.json`（例如 no capture）→ 脚本保留 preset 自带 palette，这是完整可交付设计。如果 brief 指定了 capture 漏掉的品牌 colors/fonts，在运行前把它们加到 `capture/extracted/tokens.json`（或用用户的 `design.md` 填充）；只有当映射确实需要时，才在之后手动调整 `frame.md`。

**Gate:** `build-frame.mjs` 退出 0 —— `frame.md` 来自命名 preset，并且（当 preset 随附时）`.hyperframes/caption-skin.html` 作为 caption skin source 存在。

---

## Step 3: Storyboard and Script

Goal: 将 brief 和捕获材料转换为获批的逐 frame 故事计划。

阅读 `references/story-design.md`、`../hyperframes-animation/blueprints-index.md`、`../hyperframes-core/references/storyboard-format.md` 和 `../hyperframes-core/references/script-format.md`。用它们写 `STORYBOARD.md`，并在需要 narration 时写 `SCRIPT.md`。

使用 `story-design.md` 处理 story blueprint、hook、persuasion logic、beats、`VO_MODE` 和 asset choices。作为**软指导**，参考 `../hyperframes-animation/blueprints-index.md` 中的 role→blueprint menu：对每个 beat，在适合时记录一个候选 blueprint id。故事真实性仍决定哪些 beats 存在 —— 不要强迫 beat 契合 blueprint，也不要因为有 proven shape 就发明 beat。每个 visual frame 的 `asset_candidates` 从 `capture/extracted/asset-descriptions.md`（标准清单）中选择 —— 不要浏览原始 `capture/assets/`。除非该清单缺失或不可用，否则不要让用户挑素材。使用 storyboard 和 script references 中要求的确切字段。

草拟后，展示逐 frame 摘要。在同一条消息中问用户两件事：(a) 批准或要求修改，(b) 是否想要 storyboard scaffold 的 live preview（npx hyperframes preview）—— 只有回答 yes 才打开。迭代直到获批，并把 preview 选择带到 Step 6。

**Gate:** `STORYBOARD.md` 存在，每个 visual frame 都有 `asset_candidates`，需要 narration 时 `SCRIPT.md` 存在，并且用户批准逐 frame 计划。

---

## Step 3.1: Audio

Goal: 从获批 script 生成 narration、word timings、music 和 audio metadata。

Step 3 批准后开始 audio。在后台运行它，然后继续 Step 4。

`node <SKILL_DIR>/scripts/audio.mjs --script ./SCRIPT.md --storyboard ./STORYBOARD.md --hyperframes . --out ./audio_meta.json &`

audio 脚本处理 narration、word timings、从 HeyGen music library 查找 BGM，以及 timing metadata。BGM mood 来自 storyboard 的 `music:` 字段。这使用 HeyGen Audio API 做 retrieval，不是 generation，并使用与 TTS 相同的 `~/.heygen` credential。provider 细节见 `../media-use/audio/references/tts.md`。

如果没有 narration 且没有 `SCRIPT.md`，跳过 voice generation。如果 storyboard 有 music mood，BGM 仍可能运行。

**Gate:** audio job 已启动，或项目标记为 silent。

---

## Step 4: Frame Visual Design

Goal: 为每个 storyboard frame 添加 visual direction、layout intent 和 motion choices。

原地编辑 `STORYBOARD.md`。不要创建另一个 storyboard。使用 `frame.md` 作为 color、type、layout feel 和 style 的 source of truth。

阅读 `references/visual-design.md`、`../hyperframes-animation/blueprints-index.md`、`references/motion-language.md` 和 `../hyperframes-animation/rules-index.md`。使用 `visual-design.md` 的方法（time-coded shot sequence、inline Layout vocabulary、必需的 `## Video direction` block）。使用 `../hyperframes-animation/blueprints-index.md` 选择每个 frame 的 shot shape。使用 `motion-language.md`（motion vocabulary + motion doctrine）和 `../hyperframes-animation/rules-index.md`（有效 rule names）处理 motion —— 不要发明 motion names。

对每个 visual frame，按 `visual-design.md` 的方法把 **time-coded shot sequence** 写入 `STORYBOARD.md`：选择 frame 的 blueprint（或 compose），用这个产品的内容实例化，并让每个 Scene 的 reveal 跟随 voiceover 节奏，使 frame 在整个时长中发展，而不是一开始堆满然后冻结。按 Scene **inline** 写明 layout 和 motion（词汇表在 `visual-design.md` 和 `motion-language.md` 中）。添加一个视频级 `## Video direction` block。

不要改变 story、script、asset choices、`asset_candidates`、`transition_in` 或 captured source material。此步骤不要写 HTML。

visual design 锁定后暂存命名素材：

`node <SKILL_DIR>/scripts/stage-assets.mjs --storyboard ./STORYBOARD.md --hyperframes .`

**Gate:** 每个 visual frame 都有 time-coded shot sequence，且 reveal 跟随 voiceover 节奏（无 front-loading）；`## Video direction` 存在；`assets/` 包含命名素材。

---

## Step 5: Build Frames

Goal: 将每个 storyboard frame 构建为 HTML composition，并组装可播放视频。

如果 Step 3.1 audio 已启动，先等待其完成。然后同步 durations 并获取 SFX；silent 时跳过两者。

`node <SKILL_DIR>/scripts/audio.mjs sync-durations --audio-meta ./audio_meta.json --storyboard ./STORYBOARD.md`

`node <SKILL_DIR>/scripts/audio.mjs fetch-sfx --storyboard ./STORYBOARD.md --hyperframes .`

Duration sync 是机械的：真实 voice duration 优先；silent frames 保留 estimates；绝不要手动编辑 synced durations。

派发前，阅读 `sub-agents/frame-worker.md` 和 `../hyperframes-core/references/subagent-dispatch.md`。为每个 frame 派发一个 sub-agent，尽可能并行；否则分批运行 workers。每个 worker 只得到一个 frame。

每个 worker context 必须包含 `PROJECT_DIR`、`frame_id`、canvas size、caption status 和 keep-out band（如果 captions enabled），以及作为绝对路径的 `RULES_DIR`，指向这个 skill 的 `../hyperframes-animation/rules/`。每个 worker 读取 `frame.md`、`STORYBOARD.md` 中自己的 `## Frame N` block、每个引用 motion 的本地 rule recipe（`../hyperframes-animation/rules/<id>.md`），以及 frame 的 blueprint template（`../hyperframes-animation/blueprints/<id>.md`）。每个 worker 只写 `compositions/frames/NN-*.html`。Workers 绝不能编辑 `STORYBOARD.md`。

**Full-bleed backgrounds ride on a `class="clip"` layer, never the `#root`.** frame 的 ground（color field / gradient / grid）是自己的全时长 background clip —— 设置在 `#root` / `data-composition-id` 元素上的 `background` 会被 clip-gated 到 frame window，并不是可靠的 ground，因此深色内容可能落在黑色 host `body` 上而不可见。视频 base ground 由 assembler 从 `frame.md` 的 `canvas` color 绘制到 index `#root`。（完整规则 + self-check: `sub-agents/frame-worker.md`。）

每个 worker 返回时，orchestrator 在 `STORYBOARD.md` 中将该 frame 标记为 `animated`。

audio timings 存在后，在后台构建 captions 并 assemble index：

`node <SKILL_DIR>/scripts/captions.mjs build --storyboard ./STORYBOARD.md --audio-meta ./audio_meta.json --hyperframes . --out ./caption_groups.json &`

`node <SKILL_DIR>/scripts/assemble-index.mjs --storyboard ./STORYBOARD.md --hyperframes .`

`captions.mjs` 使用项目的 `.hyperframes/caption-skin.html`（Step 2 中复制）作为 caption look，并从 `frame.md` 注入 brand tokens；没有 skin 时渲染内置默认 pill。`captions: skipped (<reason>)` 是有效状态。明确跳过 captions 时继续。

**Gate:** 每个 frame 都标记为 `animated`，`index.html` 存在，并且 captions 已构建或明确跳过。

---

## Step 6: Finalize

Goal: 验证组装后的视频，取得用户批准，并渲染最终 MP4。

注入 transitions、运行检查、暂停 review，然后 render。

`node <SKILL_DIR>/scripts/transitions.mjs inject --storyboard ./STORYBOARD.md --hyperframes .`

`node <SKILL_DIR>/scripts/transitions.mjs verify --storyboard ./STORYBOARD.md --index ./index.html`

`npx hyperframes lint`

`npx hyperframes validate`

`npx hyperframes inspect`

`npx hyperframes snapshot --at <frame-midpoints>`

`snapshot` 会把捕获的 frames 拼成一张 contact sheet（`snapshots/contact-sheet.jpg`）。看一眼；如果没有明显损坏，就继续 —— 不要在这里停留太久。

如果命令失败，暴露 stderr 并停止 —— 不要堆叠 recovery commands。自己修复：对 `compositions/frames/NN-*.html` 做最便宜且安全的编辑，然后重新运行失败的检查。

检查通过后，暂停让用户 review。视频已组装，可在 Studio 中查看和编辑。跨 Step 3 和 Step 6 只管理一次 preview：如果用户之前要求了就打开；如果之前拒绝了就再提供一次；如果他们已经在 Studio 中 review，就不要再问。

Preview: `npx hyperframes preview`

仅在用户批准后 render：

`npx hyperframes render --skill=product-launch-video --quality high --output renders/video.mp4`

除非用户要求，否则 render 后不要重新运行 `lint`、`validate`、`inspect` 或 `snapshot`。

**Gate:** render 前 `lint`、`validate` 和 `inspect` 已通过；用户在 review pause 批准；`renders/video.mp4` 存在。最终回复说明 MP4 路径和最终时长。

---

## Quick Reference

**Formats:** 默认 landscape `1920x1080`；portrait `1080x1920`；square `1080x1080`。在 storyboard frontmatter 中一次性设置 format。

**Background scripts:** workflow 在 `scripts/` 下只随附这些脚本：`build-frame` 用于 adopt + brand-remix frame preset 到 `frame.md`（+ caption skin）；`audio` 用于 TTS、transcription、BGM、SFX 和 duration syncing；`captions`；`transitions` 用于 inject 和 verify；`stage-assets` 用于把 frame 命名素材复制到 `assets/`；以及 `assemble-index`。其他一切由 `hyperframes` CLI 处理。

可复用、与产品无关的 shot shapes 位于 `../hyperframes-animation/blueprints/`（由 `../hyperframes-animation/blueprints-index.md` 索引）。

| Read                                                                                                                                                        | When                                                           |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| `[../hyperframes-creative/frame-presets/](../hyperframes-creative/frame-presets/)`                                                                          | Step 2: 选择并 adopt frame preset。                            |
| `[../hyperframes-creative/references/design-spec.md](../hyperframes-creative/references/design-spec.md)`                                                    | Step 2: 正确应用 brand tokens。                                |
| `[references/story-design.md](references/story-design.md)`                                                                                                  | Step 3: 规划 product-launch story。                            |
| `[../hyperframes-animation/blueprints-index.md](../hyperframes-animation/blueprints-index.md)`                                                              | Step 3: role→blueprint menu。Step 4: 选择 shot shape。         |
| `[../hyperframes-core/references/storyboard-format.md](../hyperframes-core/references/storyboard-format.md)`                                                | Step 3: 写 `STORYBOARD.md`。                                   |
| `[../hyperframes-core/references/script-format.md](../hyperframes-core/references/script-format.md)`                                                        | Step 3: 写 `SCRIPT.md`。                                       |
| `[../media-use/audio/references/tts.md](../media-use/audio/references/tts.md)`                                                                              | Step 3.1: 选择或理解 TTS providers 和 voices。                 |
| `[references/visual-design.md](references/visual-design.md)`                                                                                                | Step 4: 写 frame 的 shot sequence（+ Layout vocabulary）。     |
| `[references/motion-language.md](references/motion-language.md)`                                                                                            | Step 4: motion vocabulary + motion doctrine。                  |
| `[references/cut-catalog.md](references/cut-catalog.md)`                                                                                                    | Step 4-5: cut catalog（worker 构建 within-frame seams）。      |
| `[../hyperframes-animation/rules-index.md](../hyperframes-animation/rules-index.md)` + `[../hyperframes-animation/rules/](../hyperframes-animation/rules/)` | Step 5: 被引用 motions 的本地 rule recipe bodies。             |
| `[sub-agents/frame-worker.md](sub-agents/frame-worker.md)`                                                                                                  | Step 5: 派发 per-frame workers。                               |
| `[../hyperframes-core/references/subagent-dispatch.md](../hyperframes-core/references/subagent-dispatch.md)`                                                | Step 5: 安全派发 sub-agents。                                  |
