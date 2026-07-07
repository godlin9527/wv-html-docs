---
name: hyperframes-core
description: HyperFrames composition contract——构建一个可渲染项目。用于 composition structure、`data-*` timing attributes、`class="clip"`、tracks、sub-compositions、variables、framework-owned media playback、deterministic-render rules 和 validation。也涵盖 Tailwind projects 以及 STORYBOARD.md / SCRIPT.md 计划格式。编写 composition HTML 前请先阅读。
---

# HyperFrames Core

HyperFrames 从 HTML 渲染视频。一个 composition 是一个 HTML 文件：其 DOM 用 `data-*` attributes 声明 timing，其 animation runtime 可 seek，其 media playback 由 framework 接管。

此 skill 是**技术 contract**——如何构建一个 hyperframes project。下文主体是 build guide；每个主题的细节位于 `references/`（索引见下一节），按需阅读。其他关注点位于相邻 domain skills——`hyperframes-animation`、`hyperframes-creative`、`media-use`、`hyperframes-cli`、`hyperframes-registry`。`/hyperframes` 中的 capability map 说明每个 skill 覆盖什么。

## References

| File                                 | 阅读它以…                                                                                         |
| ------------------------------------ | ------------------------------------------------------------------------------------------------- |
| `references/minimal-composition.md`  | 从最小可渲染 composition skeleton 开始                                                           |
| `references/composition-patterns.md` | 选择 monolithic vs modular；构建 modular `index.html`；选择 sub-comp archetype                    |
| `references/data-attributes.md`      | 查询任何 `data-*`（root / clip / sub-comp host / legacy aliases）；使用 `class="clip"`            |
| `references/tracks-and-clips.md`     | 选择 `data-track-index`，处理同 track overlap / z-index，将一个 clip 相对另一个计时               |
| `references/sub-compositions.md`     | 接线 sub-composition（host attrs、`<template>`、per-instance vars）并在其中制作动画               |
| `references/variables-and-media.md`  | 声明 variables；放置 `<video>`/`<audio>`，设置 volume、trim                                       |
| `references/determinism-rules.md`    | 构建可 seek timeline；determinism bans；animatable-property allowlist；layout / text fit          |
| `references/full-screen-motion.md`   | 使用 shared backgrounds 编写 full-frame motion                                                    |
| `references/storyboard-format.md`    | 编写 `STORYBOARD.md` plan（以及 parsed manifest）                                                 |
| `references/script-format.md`        | 编写可选的 `SCRIPT.md` locked narration                                                           |
| `references/subagent-dispatch.md`    | 将 subagent dispatch verbs（parallel fan-out / background / wait）映射到你的 harness              |
| `references/tailwind.md`             | 在 Tailwind v4 project 中工作（`init --tailwind`；runtime contract 不同于 Studio 的 v3）          |

关于 animation runtime 细节（GSAP API、Lottie、Three.js 等），请前往 `hyperframes-animation` → `adapters/<runtime>.md`。

## 构建 composition

### 两种 root 形式（不可互换）

- **Standalone**（top-level `index.html`）——root `<div data-composition-id="…">` 直接位于 `<body>` 中，**没有 `<template>` wrapper**（包裹它会隐藏全部内容并破坏渲染）。
- **Sub-composition**（通过 `data-composition-src` 加载）——root **必须**包裹在 `<template>` 中。

> ⚠ Transport rule：runtime **只 clone `<template>` contents**；外部所有内容（包括 `<head>` styles/scripts）都会被丢弃——将 `<style>`/`<script>` **放在** template 内。
> ⚠ Host-id rule：host slot 的 `data-composition-id` 必须**完全等于**内部 template 的 `data-composition-id` **以及** `window.__timelines["<id>"]` key——不要加 `-mount`/`-slot`/`-host` suffix。

File shape、host wiring 和 pre-render checklist → `references/sub-compositions.md`。

### Root 必须有尺寸（静默 layout bug）

standalone root 需要一个显式的**有尺寸 box**（px 中的 `width`/`height`），并且从它到 `height:100%` element 的每个 ancestor 都必须有已解析 height——否则 flex/`100%` child 会塌缩到约 0，内容堆到左上角。`lint`/`validate`/`inspect` **不会**捕获这个问题。Skeleton → `references/minimal-composition.md`。

### 一个暂停 timeline

每个 composition 在 `window.__timelines["<id>"]` 上注册**恰好一个** `gsap.timeline({ paused: true })`（key = root `data-composition-id`），并在 page load 时**同步**构建。Render duration = root `data-duration`，不是 timeline length。不要手动将 sub-timelines 嵌套进 host。完整 contract（包括非 GSAP runtimes）→ `references/determinism-rules.md` + `hyperframes-animation/adapters/`。

### 不可协商规则（`lint`/`validate`/`inspect` 不会捕获的静默 bugs）

这里先列出；完整 rationale 见链接 reference。不要违反：

- 无 render-time clocks / unseeded `Math.random` / network / input-state；无 `repeat: -1`（使用有限次数）。→ `determinism-rules.md`
- 只动画 visual-property allowlist；绝不动画 `display`/`visibility`；不要对后续场景 clips 使用 `gsap.set`。→ `determinism-rules.md`
- body text 中不要用 `<br>`；transformed elements 必须是 block-level + sized；pulsing absolute decoratives 需要 peak clearance。→ `determinism-rules.md`
- `<video>`/`<audio>` 必须是 **host root 的直接子元素**（绝不放在 sub-comp `<template>`/wrapper 内）；framework 接管 playback。→ `variables-and-media.md`
- 每个 `id` 在**组装后**的 page 中必须唯一；在 sub-comp 内，用 composition id 给 ids 加前缀（`#<id>-hero`）。重复的 `<video>`/`<img>` ids 会渲染为**空白**——producer 通过 `getElementById` 注入 frames，而跨文件重复会绕过 `lint`。→ `composition-patterns.md`
- full-screen scene fill 应放在 full-bleed **child** 上（`position:absolute; inset:0`），绝不要放在 composition root 自身——producer 的 frame compositing 可能丢弃 root element 自身的 `background`（frame 渲染为**黑色**），即使 preview/`snapshot` 显示正确。→ `composition-patterns.md`

## 编辑现有 compositions

- 先读取文件。保留无关 timing、tracks、IDs、variables、media paths。
- 匹配现有 composition IDs 和 timeline keys。
- 添加 clip：选择不重叠的 `data-track-index`，或有意调整周围 timing。
- 添加 sub-composition：接线 host 前验证其内部 `data-composition-id`。

## Validation

命令细节使用 `hyperframes-cli`

- [ ] `npx hyperframes lint` passes（0 errors）
- [ ] `npx hyperframes validate` passes（0 console errors）
- [ ] `npx hyperframes inspect` passes（0 errors）
- [ ] 带 sub-compositions 的项目：`npx hyperframes snapshot --at <midpoints>` 并目检每一帧
- [ ] `npx hyperframes preview` 用于 review（用户可在 Studio 的 timeline 中编辑任何内容）
- [ ] 只有在用户批准后才运行 `npx hyperframes render`
