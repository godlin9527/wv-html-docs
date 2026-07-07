---
name: pr-to-video
description: "将 GitHub pull request（PR URL、owner/repo#N，或 checked-out repo 中的 'this PR'）转换为代码变更讲解视频 —— changelog、feature reveal、fix 或 refactor walkthrough，基于 diff、commits 和 files 构建：输入是代码变更，不是网站。不是产品宣传（/product-launch-video），也不是无 PR 的主题解释（/faceless-explainer）。不明确 → /hyperframes。"
---

> **media-use**: 在获取 audio/images 之前，调用 `/media-use` 从 HeyGen catalog 解析 BGM/SFX/images。先运行 `--adopt` 以注册现有素材。见 `/media-use` skill。

# PR to HyperFrames

使用这个 skill 摄取 GitHub pull request，理解变更，规划代码变更讲解，并在 HyperFrames 中逐 frame 构建。输入是**代码变更**（通过 `gh` 读取），不是网站 —— **没有 capture step，也没有真实素材**，除了 contributors 的 avatars。

> **Confirm the route before Step 0.** 你是编排者。运行每一步，验证其 gate，然后才继续。这个 skill 用于 **GitHub pull request**（代码变更）。其他意图路由到 elsewhere：product launch/promo → `/product-launch-video`；general website tour → `/website-to-video`；没有 PR 的 topic explainer → `/faceless-explainer`；给现有 footage 加 captions → `/embedded-captions`；短的无旁白 motion graphic → `/motion-graphics`；whole-repo 或 multi-PR release walkthrough → `/general-video`。**Out of scope:** live / at-render-time data —— PR facts 在 author time 读取一次并 baked in。如果用户只说 "make a video" 或 route 不确定，先阅读 `/hyperframes`。

你是编排者。在 `videos/<project>/` 中工作。按顺序运行步骤，并在继续前通过每个 gate。用户 gate 是 Step 0、Step 3 和 Step 6。除了 Step 5 以外，所有步骤都由你自己完成；Step 5 中你为每个 frame 派发一个 sub-agent。不要把设计或动效规则放在这里；它们位于 frame-worker sub-agent、这个 skill 本地的 `../hyperframes-animation/rules/` + `../hyperframes-animation/blueprints/`，以及 `hyperframes-creative`。

Workflow: Step 0 setup → `hyperframes.json`; Step 1 ingest → `capture/extracted/` + `assets/<login>.png`; Step 2 design system → `frame.md`; Step 3 storyboard/script → `STORYBOARD.md` and `SCRIPT.md`; Step 3.1 audio → `audio_meta.json`; Step 4 visual design → enriched `STORYBOARD.md`; Step 5 frames → `compositions/frames/NN-*.html` and `index.html`; Step 6 final render → `renders/video.mp4`.

---

## Step 0: Setup and Brief

Goal: 锁定 PR reference 和核心视频 brief，并在需要时创建 HyperFrames 项目。

获取 **PR reference**（完整 URL、`<owner>/<repo>#<N>` ref，或 checked-out repo 中的 "this PR"），并在一条消息中确认 brief —— 对每项先给推荐默认值，并预填 `/hyperframes` 已设置的任何内容：**angle**（changelog / feature-reveal / fix-explainer / refactor-walkthrough —— 默认：从 PR 推断）、**audience**（默认：developers）、**length**（默认：**按 PR 变更规模缩放** —— 见下文）、**aspect**（默认 16:9）、**language**。style 始终是 **claude**。只有用户回复后才继续；"go" 接受默认值。

**根据 PR 变更规模推荐长度**，不要固定猜测。确认 brief 前先 peek 一次 PR —— 只读调用，也用于 grounding angle（Step 1 仍执行完整 deterministic fetch）：

```bash
gh pr view <PR_REF> --json title,additions,deletions,changedFiles
```

根据 `additions + deletions`（由 `changedFiles` 轻微上调）选择 tier，并把它作为默认值提出（用户可覆盖；硬上限约 3 min）：

| PR change size                    | Recommended length |
| --------------------------------- | ------------------ |
| trivial (≲ 50 lines changed)      | ~20–40s            |
| focused (~50–200 lines)           | ~40–70s            |
| substantial (~200–600 lines)      | ~70–110s           |
| large (≳ 600 lines, or 25+ files) | ~110–180s          |

提出时用一句话说明依据（例如 "~40s — small change, +44/−13 across 12 files"）。巨大 PR 不意味着长视频 —— 如果故事只是一个 headline change，就保持紧凑并说明原因。

只有缺少 `hyperframes.json` 时初始化。根据 PR 用 kebab-case 命名 `<project>`，例如 `acme-sdk-pr-1842`；不要使用 workspace name 或 timestamp。

`npx hyperframes init "videos/<project>" --non-interactive --example=blank` — `init` 会检查已安装 skills 与 GitHub 最新版本，并在任何 skill 过期时更新全局集合。

**Show sign-in status before the brief** — 运行 `npx hyperframes auth status` 并**逐字转述其输出（不要改写或转述）。** 它会报告 voice/BGM 会使用 HeyGen 还是本地引擎，以及未登录时如何登录。**如果未登录，停止并等待用户选择 —— 登录，或说 "go"/"offline" 以继续使用本地引擎 —— 然后再询问 brief 或其他任何内容。** 把它当作真正的决策点，而不是顺带说明；不要把这个选择折进 brief 问题里，也不要把 key 写进每个 repo 的 `.env`。（在 autonomous mode 中，记录状态并继续离线。）标准指导见 `../media-use` → Preflight。

**Gate:** `hyperframes.json` 存在；PR ref 已捕获；angle、length、aspect ratio 和 language 已锁定；sign-in status 已展示（已登录，或继续离线）。

---

## Step 1: Ingest the PR (no capture)

Goal: 获取 PR facts，并把它们作为信息源折入项目。这里**没有 website capture**。`fetch-pr.mjs` 确定性运行 `gh` —— 通过 paginated `gh api` 补全 files list，避免大 PR 在约 100 个文件处截断，并只写 `capture/pr.json` + `capture/diff.patch`（没有 scratch dir）。然后 `ingest.mjs` 离线把它折入 synthetic capture package。

```bash
PR="<url | owner/repo#N | N>"

# Fetch the PR deterministically: runs gh, completes the files list via paginated
# gh api (so a big PR doesn't truncate at ~100 files), writes only capture/pr.json +
# capture/diff.patch — no scratch dir. gh auth / not-found / private errors exit 1 here.
(cd "videos/<project>" && node <SKILL_DIR>/scripts/fetch-pr.mjs --pr "$PR" --out-dir ./capture)

# Offline transform → capture/extracted/{tokens.json (colors:[] → claude palette),
# visible-text.txt (the brief), people.json (contributors, bot-filtered, avatarFile=assets/<login>.png)}.
(cd "videos/<project>" && node <SKILL_DIR>/scripts/ingest.mjs \
  --pr-json ./capture/pr.json --diff ./capture/diff.patch --out-dir ./capture/extracted)

# The people front's one network step — download each contributor's GitHub avatar to
# assets/<login>.png for an optional credits close. Best-effort; always exits 0.
(cd "videos/<project>" && node <SKILL_DIR>/scripts/fetch-people-avatars.mjs \
  --people ./capture/extracted/people.json)
```

如果 `fetch-pr.mjs` 退出 1（gh auth / not found / private），报告其 stderr 并停止 —— **不要编造 PR contents**。如果 `ingest.mjs` 退出 1，读取 stderr（通常是 malformed `pr.json`），修复并重新运行（deterministic）。`fetch-people-avatars.mjs` 始终退出 0；缺失 avatars 只意味着没有 credits close 可 author。

**Gate:** `capture/pr.json`、`capture/diff.patch`、`capture/extracted/tokens.json`、`capture/extracted/visible-text.txt` 和 `capture/extracted/people.json` 存在；你能用一句清晰的话说明 PR 变更。`assets/<login>.png` 是 best-effort —— 缺失不是失败。

---

## Step 2: Design System

Goal: Adopt claude frame preset；脚本会把它转换成此视频的 `frame.md` + caption skin。

style 固定为 **claude**（warm editorial；为 diffs 构建的 navy code surface）。运行：

```bash
node <SKILL_DIR>/scripts/build-frame.mjs --preset claude --hyperframes .
```

脚本复制 claude preset 的 `FRAME.md` → `frame.md`，把它 remix 到 `capture/extracted/tokens.json` 中的任何 brand tokens 上（PR 没有 → `colors:[]`/`fonts:[]` 保持 claude 自带 palette，这是完整设计），将 preset 的 caption skin 复制到 `.hyperframes/caption-skin.html`，并自验证（映射损坏时退出 1）。一旦它退出 0 就继续 —— 不要手动编辑。

**Gate:** `build-frame.mjs` 退出 0 —— `frame.md` 来自 claude preset，且 `.hyperframes/caption-skin.html` 作为 caption skin source 存在。

---

## Step 3: Storyboard and Script

Goal: 将 PR 转换为获批的逐 frame 解释计划。

阅读 `references/story-design.md`、`../hyperframes-animation/blueprints-index.md`、`../hyperframes-core/references/storyboard-format.md` 和 `../hyperframes-core/references/script-format.md`。用它们写 `STORYBOARD.md`，并在需要 narration 时写 `SCRIPT.md`。

使用 `story-design.md` 处理 PR archetype（changelog / feature-reveal / fix-explainer / refactor-walkthrough）、PR-native frame types、hook、persuasion、beats、逐 frame word budget，以及可选 credits close。sequence 来自**叙事设计，不是 diff 的文件顺序** —— 解释变更，不要朗读 diff。作为**软指导**，参考 `../hyperframes-animation/blueprints-index.md` 中的 role→blueprint menu：对每个 beat，按候选 blueprint 暗示的形状写 voiceover，并在适合时标记该候选 `blueprint:` id（story truth 仍决定哪些 beats 存在 —— 绝不要强迫 beat 契合某个形状）。展示 2–4 个真实 diff hunks（来自 `capture/diff.patch`），每个都是小而清晰的 snippet；在 frame 的 `scene` 中命名每个需要的 `code-*` block。Frames 不携带 `asset_candidates`，除了可选 `credits` close（2–6 个 `assets/<login>.png` avatars）。使用 storyboard 和 script references 中要求的确切字段。

草拟后，展示逐 frame 摘要。在同一条消息中询问用户 (a) 批准或请求修改，(b) 是否想要 storyboard scaffold 的 live preview（`npx hyperframes preview`）—— 只有 yes 才打开。迭代直到获批；把 preview 选择带到 Step 6。

**Gate:** `STORYBOARD.md` 存在，每个 frame 都有必需 narrative fields，需要 narration 时 `SCRIPT.md` 存在，并且用户批准计划。

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

阅读 `references/visual-design.md`、`../hyperframes-animation/blueprints-index.md`、`references/motion-language.md`、`references/code-vocabulary.md` 和 `../hyperframes-animation/rules-index.md`。使用 `visual-design.md` 的方法（time-coded shot sequence、inline Layout vocabulary、code-beat treatment），以及必需的 `## Video direction` block。使用 `../hyperframes-animation/blueprints-index.md` 选择每个 frame 的 shot shape。使用 `code-vocabulary.md` 为每个 code beat 选择正确的 `code-*` block（diff = `code-diff`、refactor = `code-morph`、new code = `code-typing`，等等）。使用 `motion-language.md`（motion vocabulary + motion doctrine）和 `../hyperframes-animation/rules-index.md`（有效 rule names）处理 motion —— 不要发明 motion 或 block/blueprint names。

对每个 frame，按 `visual-design.md` 的方法把 **time-coded shot sequence** 写入 `STORYBOARD.md`：选择 frame 的 blueprint（或 compose），用这个 frame 的内容实例化，并让每个 Scene 的 reveal 跟随 voiceover，使 frame 在整个时长中发展，而不是一开始堆满然后冻结。**对 code beat，`code-*` block 是 frame 的 `focal`**，Scenes 编排周围的 claude Code Surface（file/header 进入、camera 到 hunk、landing line）—— **不是**代码动画本身，代码动画由 block 拥有。按 Scene **inline** 写明 layout 和 motion（词汇表在 `visual-design.md` 和 `motion-language.md` 中）。添加一个视频级 `## Video direction` block。

不要改变 story、script、`transition_in`、`asset_candidates` 或 PR source。此步骤不要写 HTML。**没有 asset-staging step** —— 唯一真实素材是 credits avatars，已在 `assets/`。

**Gate:** 每个 frame 都有 time-coded shot sequence，且 reveal 跟随 voiceover（无 front-loading）；code frames 将一个 `code-*` block 命名为 `focal`；`## Video direction` 存在。

---

## Step 5: Build Frames

Goal: 将每个 storyboard frame 构建为 HTML composition，并组装可播放视频。

如果 Step 3.1 audio 已启动，先等待其完成。然后同步 durations 并获取 SFX；silent 时跳过两者。

`node <SKILL_DIR>/scripts/audio.mjs sync-durations --audio-meta ./audio_meta.json --storyboard ./STORYBOARD.md`

`node <SKILL_DIR>/scripts/audio.mjs fetch-sfx --storyboard ./STORYBOARD.md --hyperframes .`

Duration sync 是机械的：真实 voice duration 优先；silent frames 保留 estimates；绝不要手动编辑 synced durations。

**派发前先一次性预安装 `STORYBOARD.md` 中命名的 registry blocks**，避免并行 workers 在 registry 上竞争：

`for b in <each registry block named in the storyboard>; do npx hyperframes add "$b"; done`

派发前，阅读 `sub-agents/frame-worker.md` 和 `../hyperframes-core/references/subagent-dispatch.md`。为每个 frame 派发一个 sub-agent，尽可能并行；否则分批运行 workers。每个 worker 只得到一个 frame。每个 worker 的 context 必须包含 `PROJECT_DIR`、`frame_id`、canvas size、caption status 和 keep-out band（如果 captions enabled）、`RULES_DIR`（指向此 skill 的 `../hyperframes-animation/rules/` 的绝对路径），以及 `references/code-vocabulary.md` 的绝对路径。每个 worker 读取 `frame.md`、`STORYBOARD.md` 中自己的 `## Frame N` block、每个引用 motion 的本地 rule recipe（`../hyperframes-animation/rules/<id>.md`）、frame 的 blueprint template（`../hyperframes-animation/blueprints/<id>.md`），以及 —— 对 code beat —— `code-vocabulary.md` 中命名 block 的 inputs。每个 worker 只写 `compositions/frames/NN-*.html`；workers 永远不编辑 `STORYBOARD.md`。

**Full-bleed backgrounds ride on a `class="clip"` layer, never the `#root`.** frame 的 ground（color field / gradient / grid）是自己的全时长 background clip —— 设置在 `#root` / `data-composition-id` 元素上的 `background` 会被 clip-gated 到 frame window，并不是可靠的 ground，因此深色内容可能落在黑色 host `body` 上而不可见。视频 base ground 由 assembler 从 `frame.md` 的 `canvas` color 绘制到 index `#root`。（完整规则 + self-check: `sub-agents/frame-worker.md`。）

每个 worker 返回时，你可以在 `STORYBOARD.md` 中将该 frame 标记为 `animated`。

audio timings 存在后，在后台构建 captions 并 assemble index：

`node <SKILL_DIR>/scripts/captions.mjs build --storyboard ./STORYBOARD.md --audio-meta ./audio_meta.json --hyperframes . --out ./caption_groups.json &`

`node <SKILL_DIR>/scripts/assemble-index.mjs --storyboard ./STORYBOARD.md --hyperframes .`

`captions.mjs` 使用项目的 `.hyperframes/caption-skin.html`（claude 的，Step 2 中复制），并从 `frame.md` 注入 brand tokens；`captions: skipped (<reason>)` 是有效状态。`assemble-index.mjs` 会作为 idempotent backstop 从 `assets/` 暂存 credits avatars。

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

**Known false-positive — do not chase it.** `inspect` 可能报告少量 **caption** highlight words（selector `#caption-word-*` / `.caption-line`）约 1–4px 的 `text_box_overflow` 错误。caption pill 使用刻意紧凑的 `line-height`（在 `scripts/captions.mjs` 中设置一次），且**没有 `overflow:hidden`**，所以重 display glyph 的 ink 会溢出几 px 到 pill 自己的 padding 中 —— 实际没有被裁切。把这些视为预期并继续。不要扩大 caption `line-height`（它会把 pill 膨胀得更糟）。只有当 `text_box_overflow` 指向 **frame** 元素（`#el-NN-*`）而不是 caption word 时才处理。

检查通过后，暂停让用户 review。视频已组装，可在 Studio 中查看和编辑。跨 Step 3 和 Step 6 只管理一次 preview：如果用户之前要求了就打开；如果之前拒绝了就再提供一次；如果他们已经在 Studio 中 review，就不要再问。

Preview: `npx hyperframes preview`

仅在用户批准后 render：

`npx hyperframes render --skill=pr-to-video --quality high --output renders/video.mp4`

除非用户要求，否则 render 后不要重新运行 `lint`、`validate`、`inspect` 或 `snapshot`。

**Gate:** render 前 `lint`、`validate` 和 `inspect` 已通过；用户在 review pause 批准；`renders/video.mp4` 存在。最终回复说明 MP4 路径和最终时长。

---

## Quick Reference

**Formats:** 默认 landscape `1920x1080`；portrait `1080x1920`；square `1080x1080`。在 storyboard frontmatter 中一次性设置 format。

**PR deltas vs a captured-asset workflow:** 没有 Step 1 capture（`gh` CLI 将 PR 摄取为 synthetic `capture/extracted/` package —— `tokens.json` + `visible-text.txt` + `people.json`）；唯一真实素材是 contributors 的 `assets/<login>.png` avatars（可选 credits close）；没有 `asset-descriptions.md`，没有 asset-staging step。Code beats 由 `code-*` registry blocks 在 claude 的 navy Code Surface 上渲染；style 始终是 **claude**。

**Background scripts:** workflow 在 `scripts/` 下随附这些脚本：`fetch-pr`（PR → 通过 `gh` 生成 `capture/pr.json` + `diff.patch`；large-PR-safe，无 scratch）、`ingest`（→ synthetic capture package；offline）、`fetch-people-avatars`（contributor avatars → `assets/`）；以及共享引擎 —— `build-frame`（adopt + brand-remix preset 到 `frame.md` + caption skin）、`audio`（TTS、BGM、SFX、duration sync）、`captions`、`transitions`（inject + verify）和 `assemble-index`。其他一切由 `hyperframes` CLI 处理。Code blocks 通过 `npx hyperframes add <name>` 安装。

可复用、domain-agnostic 的 shot shapes 位于 `../hyperframes-animation/blueprints/`（由 `../hyperframes-animation/blueprints-index.md` 索引）；`code-*` registry blocks 是 code-beat vocabulary（`references/code-vocabulary.md`）。

| Read                                                                                                                                                        | When                                                           |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| `[references/story-design.md](references/story-design.md)`                                                                                                  | Step 3: 规划 PR explanation。                                  |
| `[../hyperframes-animation/blueprints-index.md](../hyperframes-animation/blueprints-index.md)`                                                              | Step 3: role→blueprint menu。Step 4: 选择 shot shape。         |
| `[../hyperframes-core/references/storyboard-format.md](../hyperframes-core/references/storyboard-format.md)`                                                | Step 3: 写 `STORYBOARD.md`。                                   |
| `[../hyperframes-core/references/script-format.md](../hyperframes-core/references/script-format.md)`                                                        | Step 3: 写 `SCRIPT.md`。                                       |
| `[../media-use/audio/references/tts.md](../media-use/audio/references/tts.md)`                                                                              | Step 3.1: 选择或理解 TTS providers。                           |
| `[references/visual-design.md](references/visual-design.md)`                                                                                                | Step 4: 写 frame 的 shot sequence（+ Layout vocabulary）。     |
| `[references/code-vocabulary.md](references/code-vocabulary.md)`                                                                                            | Step 4 + 5: 为 code beat 选择 + 填写 `code-*` block。          |
| `[references/motion-language.md](references/motion-language.md)`                                                                                            | Step 4: motion vocabulary + motion doctrine。                  |
| `[references/cut-catalog.md](references/cut-catalog.md)`                                                                                                    | Step 4-5: cut catalog（worker 构建 within-frame seams）。      |
| `[../hyperframes-animation/rules-index.md](../hyperframes-animation/rules-index.md)` + `[../hyperframes-animation/rules/](../hyperframes-animation/rules/)` | Step 5: 被引用 motions 的本地 rule recipe bodies。             |
| `[sub-agents/frame-worker.md](sub-agents/frame-worker.md)`                                                                                                  | Step 5: 派发 per-frame workers。                               |
| `[../hyperframes-core/references/subagent-dispatch.md](../hyperframes-core/references/subagent-dispatch.md)`                                                | Step 5: 安全派发 sub-agents。                                  |
| `[../hyperframes-creative/frame-presets/claude/FRAME.md](../hyperframes-creative/frame-presets/claude/FRAME.md)`                                            | Step 2: claude preset（固定 style）。                          |
