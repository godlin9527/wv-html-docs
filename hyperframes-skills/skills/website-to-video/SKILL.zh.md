---
name: website-to-video
description: "捕获一个通用网站/URL，并将其转换为该网站的视频 —— 基于捕获截图和网站自身品牌素材制作的导览、展示或社交短片。用于 portfolio / blog / docs / landing-page 展示。即使来自 URL，也不是产品发布或宣传（/product-launch-video）。不明确 → /hyperframes。"
---

> **media-use**: 在获取 audio/images 之前，调用 `/media-use` 从 HeyGen catalog 解析 BGM/SFX/images。先运行 `--adopt` 以注册现有素材。见 `/media-use` skill。

# Website to HyperFrames

捕获一个网站，然后从中制作专业视频。

> **Confirm the route before Step 0.** 这个 skill 制作的是 _of / from a general site_ 的视频。如果用户真正想要**营销 / 发布 / 推广一个产品**（即使来自这个 URL，即使说 "promo for our site"）→ `/product-launch-video`。**没有 site 的 topic explainer** → `/faceless-explainer`；**GitHub PR** → `/pr-to-video`；**重新剪辑 / 重新调色 / 重新排序现有视频文件** → 超出范围。因为模糊的 "make a video" 路由到这里，或不确定 launch-vs-general-site？**先阅读 `/hyperframes`**（完整 routing table + § What HyperFrames cannot do）。

用户会说类似：

- "Turn this website into a 15-second social clip for Instagram"
- "Make a 30-second site tour / showcase from https://..."
- "Capture our homepage and build a video from its own visuals"

workflow 有 7 个步骤。每一步都生成一个 artifact 来 gate 下一步。默认是协作式 —— 标记 💬 的 gates 会停止并询问用户。如果用户表示 autonomous mode（"decide for me", "surprise me"），会跳过 💬 user-preference gates；其传播方式见 step-2-brief.md。

**Autonomous mode 不是 "skip all gates."** Auto mode 覆盖用户偏好问题（TTS provider、voice、color emphasis、beat count、music yes/no、captions yes/no —— agent 代表用户决定）。它不覆盖质量验证 gates。以下内容在 auto mode 中仍不可跳过：

- Asset Audit (Step 3) —— 查看 contact sheets，并为每个 asset 说明 USE/SKIP 理由
- Per-beat HTML read (Step 5) —— 每个 beat 的 structured evidence block
- DoD checklist (Step 6) —— 包括 animation-map、每条 warning 的 WCAG verification、audio/motion playback
- Honest disclosure section (Step 6) —— 最终总结必须出现 "What I did NOT verify"

如果你发现自己在推理 "auto mode says bias toward action, so I'll skip X" —— 而 X 是 verification gate，不是 preference question —— 这个推理就是错的。Bias toward action 适用于决定 _what to build_，不适用于决定 _whether to verify_。

---

## Step 0: Capture & Understand the Brand

**Read:** [references/step-0-capture.md](references/step-0-capture.md)

捕获站点，然后读取提取数据以理解**品牌和产品** —— 它做什么、面向谁、用什么语气说话、处在什么情绪中。捕获的素材是后续使用的品牌工具包，不是视频由其构成的积木。

**Show sign-in status before the brief** — 运行 `npx hyperframes auth status` 并**逐字转述其输出（不要改写或转述）。** 它会报告 voice/BGM 会使用 HeyGen 还是本地引擎，以及未登录时如何登录。**如果未登录，停止并等待用户选择 —— 登录，或说 "go"/"offline" 以继续使用本地引擎 —— 然后再询问 brief 或其他任何内容。** 把它当作真正的决策点，而不是顺带说明；不要把这个选择折进 brief 问题里，也不要把 key 写进每个 repo 的 `.env`。（在 autonomous mode 中，记录状态并继续离线。）标准指导见 `../media-use` → Preflight。

**Gate:** Site summary 已打印 —— strategy-first（产品做什么、面向谁、品牌语气）先于 asset / color / font inventory；sign-in status 已展示（已登录，或继续离线）。

---

## Step 1: Brand Identity

**Read:** [references/step-1-design.md](references/step-1-design.md)

写 DESIGN.md —— 一份覆盖视觉识别的品牌速查：colors、typography、component styles、layout principles。使用 `design-styles.json` 获取精确 computed values。

**Speed option:** 对快节奏视频（billboard-per-beat），DESIGN.md 可以是 50 行的 colors + fonts + do's/don'ts 摘要 —— 不必是 300 行文档。Step 5 的 sub-agent prompt 会直接粘贴品牌值，因此 DESIGN.md 的深度只对复杂 compositions 重要。

**Gate:** `DESIGN.md` 存在（任意长度），且至少包含：color palette、font choices 和 do's/don'ts。

---

## Step 2: Strategy & Messaging

**Read:** [references/step-2-brief.md](references/step-2-brief.md), [references/capabilities.md](references/capabilities.md)（扫描 Table of Contents —— 只在需要时深入特定章节）

先和用户对齐**视频必须传达什么**，再讨论视觉或素材。解析用户 prompt —— 他们很可能已经给了 video type 和 style。只问缺失内容：这个视频必须表达的 ONE thing、narrative arc 和 audience。

**Gate:** Video type、duration、format，以及关键的 message 和 narrative arc 已锁定。没有这些，Step 3 无法写 concept-first storyboard。

---

## Step 3: Storyboard + Script 💬

**Read:** [references/step-3-storyboard.md](references/step-3-storyboard.md)

以 concept-first 写 storyboard：message → narrative arc → 服务于 arc 的 beats → 每个 beat 的 techniques → 最后做 brand accents pass。然后写匹配的 narration script。把两者连同逐 beat 摘要展示给用户。迭代直到他们批准。

**Gate:** `STORYBOARD.md` + `SCRIPT.md` 存在，并且用户已批准计划。

---

## Step 4: VO, Timing + Captions 💬

**Read:** [references/step-4-vo.md](references/step-4-vo.md)

如果 Step 2 说不需要 narration —— 询问 background music，然后跳到 Step 5。否则：询问用户选择哪个 TTS provider（HeyGen TTS、ElevenLabs 或 Kokoro），生成 audio，转录，映射 timestamps 到 beats。然后询问 captions。

**Gate:** 要么 (a) 未请求 narration 且 storyboard 有 manual beat timings，要么 (b) `narration.wav` + `transcript.json` 存在，并且 beat timings 已用真实 durations 更新。

---

## Step 5: Build Compositions

**Read:** The `hyperframes` skill（加载它 —— 每条规则都重要）
**Read:** [references/step-5-build.md](references/step-5-build.md)

按照 storyboard（Step 3）中选择的 architecture 和 pacing 构建 index.html 和 compositions。Sub-agents 在每个 beat 上运行 `hyperframes lint` 和 `hyperframes snapshot` 后再报告。

**Gate:** 每个 `compositions/beat-N.html` 都已由 main agent 根据 DESIGN.md 和 STORYBOARD.md 从头到尾读过。per-beat checklist 位于 [step-5-build.md](references/step-5-build.md)。

---

## Step 6: Validate & Deliver

**Read:** [references/step-6-validate.md](references/step-6-validate.md)

Lint、validate、按视频长度缩放 snapshots（公式：`max(beats × 3, ceil(duration_seconds / 2))`），并 review 每一张。交付前修复问题。交付 localhost Studio project URL —— 只有在用户明确要求时才 render 到 MP4。只在 handoff 时暴露 Studio URL —— 它是最终稳定 preview；build 阶段 snapshots 是 headless 的，所以不要在 build 中途弹出 preview。

**Deliver something you're proud of.** 交付前问自己：我会用自己的名字把这个发到社交媒体上吗？如果不会，修复问题。

**Gate:** `npx hyperframes lint` 和 `npx hyperframes validate` 以零错误通过，且最终回复包含 active Studio project URL。

---

## Quick Reference

### Video Types

按视频类型的典型约束 —— 用作起点，不是公式。Beat count 应来自内容和 narration，而不是来自目标范围。

| Type                    | Typical duration | Duration driver    | Narration             |
| ----------------------- | ---------------- | ------------------ | --------------------- |
| Social clip (IG/TikTok) | 10–15s           | Platform limit     | Optional              |
| Site walkthrough        | 30–60s           | Script length      | Full narration        |
| Content announcement    | 15–30s           | Content complexity | Full narration        |
| Brand reel              | 20–45s           | Music track        | Optional, music focus |

（产品演示、功能公告或 launch teaser，只要是在 _selling_ 产品，就属于 `/product-launch-video` —— 见顶部 routing note。）

Beat count 故意不在此表中 —— 它应来自 storyboard，而不是来自 "social ad = 3-4 beats." 复杂产品的 social ad 可能需要 5 个节奏良好的 beats。一个有强视觉论点的 brand reel 可能只需要 3 个。

### Format

- **Landscape**: 1920x1080 (default)
- **Portrait**: 1080x1920 (Instagram Stories, TikTok)
- **Square**: 1080x1080 (Instagram feed)

### Reference Files

| File                                                                               | When to read                                                                                                                                   |
| ---------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| [step-0-capture.md](references/step-0-capture.md)                                  | Step 0 — capture、理解品牌和产品、写 strategy-first site summary                                                                                |
| [step-1-design.md](references/step-1-design.md)                                    | Step 1 — 写 DESIGN.md brand cheat sheet（5 sections, 250-350 lines；billboard-style social ads 可用 50-line fast-path）                         |
| [step-2-brief.md](references/step-2-brief.md)                                      | Step 2 — 与用户对齐 message、narrative arc、audience                                                                                            |
| [capabilities.md](references/capabilities.md)                                      | Steps 2 & 5 — HyperFrames 能力完整清单（24 sections）。brief 阶段扫描 TOC，build 阶段深入特定章节                                              |
| [step-3-storyboard.md](references/step-3-storyboard.md)                            | Step 3 — storyboard + script（combined），带 user review gate                                                                                   |
| [step-4-vo.md](references/step-4-vo.md)                                            | Step 4 — TTS provider choice、generation、timing                                                                                                |
| [step-5-build.md](references/step-5-build.md)                                      | Step 5 — 构建 index.html + compositions                                                                                                         |
| [step-6-validate.md](references/step-6-validate.md)                                | Step 6 — lint、validate、snapshots（按视频长度缩放）、preview                                                                                   |
| [techniques.md](../hyperframes/references/techniques.md)                           | Steps 3 & 5 — 13 种 primitive animation techniques with code patterns（adapt, don't copy-paste）                                                |
| [html-in-canvas-patterns.md](../hyperframes/references/html-in-canvas-patterns.md) | Step 5 — HTML-in-Canvas effects 的完整 code patterns（位于 hyperframes skill 中）                                                               |
