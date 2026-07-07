---
name: remotion-to-hyperframes
description: '将现有 Remotion (React) composition 的源代码移植到 HyperFrames HTML。只在明确要求 port/convert/migrate/translate Remotion source 时使用 —— 单向、仅 Remotion。顺带提到 Remotion、仅作参考的代码，或 "make something like my Remotion video" 都是 fresh build（/general-video）。不明确 → /hyperframes。'
---

# Remotion to HyperFrames

> **Confirm the route before you build.** 只在把现有 **Remotion** (React) composition 源码移植到 HyperFrames 时使用。创作**新的** composition（即使灵感来自 Remotion video）→ creation workflows / `/general-video`。**Out of scope**（单向、仅 Remotion）：没有 reverse export（HyperFrames → Remotion 或任何 framework），并且 **non-Remotion** source（After Effects、Framer Motion、plain React / CSS）没有可翻译的 Remotion source → 通过 `/general-video` 重新创建。不确定，或只是顺带提到 Remotion？**先阅读 `/hyperframes`。**

## Overview

将 Remotion（基于 React）的视频 compositions 翻译为 HyperFrames（HTML + GSAP）compositions。大多数 Remotion idioms 都有直接的 HyperFrames equivalents —— 对约 80% 的典型 compositions，翻译是机械的。这个 skill 编码了 mapping，并通过拒绝翻译不适合 HF seek-driven model 的 patterns 来防止有损的 20%，改为推荐来自 [PR #214](https://github.com/heygen-com/hyperframes/pull/214) 的 runtime interop pattern。

这个 skill 随附一个**分层测试语料库**（T1–T4，共 4 个 fixtures），用 measured SSIM thresholds 给翻译评分。不要在不运行 eval 的情况下翻译 —— 一个“看起来对”但比 validated baseline 低 0.05 SSIM 的翻译是静默错误。

## When to use

**仅当用户明确要求从 Remotion 迁移时使用这个 skill。** 触发短语示例：

- "port my Remotion project to HyperFrames"
- "convert this Remotion code to HyperFrames"
- "migrate from Remotion"
- "translate this Remotion comp"
- "rewrite this as HyperFrames HTML"

**不要在以下情况使用这个 skill：**

- (a) 用户正在创作一个**新的** HyperFrames composition，即使他们有或正在 A/B-testing 一个相似的 Remotion video。
- (b) 用户只是顺带提到 Remotion，没有要求迁移。
- (c) 用户分享 Remotion code 作为 reference material，而不是要求 translation。
- (d) 用户要求 "the same video as my Remotion one"，但没有明确要求迁移 source —— 将其视为 fresh HyperFrames build。

**NOT SUPPORTED（拒绝 —— 这不是此 skill 的工作）：**

- **反方向。** 将 HyperFrames composition export 回 _to_ Remotion（或任何其他 framework）不是一个 workflow —— translation 只支持 Remotion → HyperFrames。直接说明这一点。
- **Non-Remotion sources.** After Effects project（`.aep`）、Framer Motion / plain-React / CSS animation，或任何其他工具的 source 都不是 Remotion composition —— 没有 Remotion source 可翻译。通过 `/general-video` 原生重建，或在 HyperFrames 无法表示时拒绝。

有疑问时，默认改用 `/general-video`（通用 HyperFrames authoring flow）创作 native HyperFrames composition。

## Workflow

### Step 1: Lint the source

在 Remotion source directory 上运行 [`scripts/lint_source.py`](scripts/lint_source.py)。lint 会检测无法干净翻译的 patterns：

- **Blockers**（拒绝 + 推荐 interop）：`useState`、`useReducer`、带 non-empty deps 的 `useEffect`/`useLayoutEffect`、async `calculateMetadata`、third-party React UI libraries（MUI、Chakra、Mantine、antd、shadcn、Radix、NextUI）。
- **Warnings**（丢弃该 construct 后翻译）：`@remotion/lambda` config、`delayRender`、`useCallback`、`useMemo`、custom hooks。
- **Info**（带 note 翻译）：`staticFile`、`interpolateColors`。

如果任何 blocker 触发，**停止**。阅读 [`references/escape-hatch.md`](references/escape-hatch.md) 并展示 recommendation message。Warnings 不阻止 translation —— 在 step 3 丢弃 offending construct，并在 `TRANSLATION_NOTES.md` 中记录 gap。`@remotion/lambda` config 是 canonical warning case：skill 会丢弃 import + `renderMediaOnLambda(...)` calls，但翻译 composition 的其余部分。

### Step 2: Plan the translation

阅读 [`references/api-map.md`](references/api-map.md) —— 每个 Remotion API 及其 HF equivalent 或 per-topic reference 的索引。根据 source 使用的内容识别需要哪些 topic references：

| Source contains                                                           | Load reference                                |
| ------------------------------------------------------------------------- | --------------------------------------------- |
| `Composition`, `defaultProps`, `schema`, `calculateMetadata`              | [`parameters.md`](references/parameters.md)   |
| `Sequence`, `Series`, `Loop`, `AbsoluteFill`, `Freeze`                    | [`sequencing.md`](references/sequencing.md)   |
| `useCurrentFrame`, `interpolate`, `spring`, `Easing`, `interpolateColors` | [`timing.md`](references/timing.md)           |
| `Audio`, `Video`, `Img`, `IFrame`, `staticFile`, `delayRender`            | [`media.md`](references/media.md)             |
| `TransitionSeries`, `@remotion/transitions`                               | [`transitions.md`](references/transitions.md) |
| `@remotion/lottie`                                                        | [`lottie.md`](references/lottie.md)           |
| `@remotion/google-fonts/<Family>`, `Font.loadFont`, `@font-face`          | [`fonts.md`](references/fonts.md)             |

不要全部加载 —— 只加载具体 source 需要的内容。

### Step 3: Generate the HF composition

输出 `index.html`，包含：

- Root `<div id="stage">`，携带 composition 的 `data-composition-id`、`data-start="0"`、`data-duration`（秒）、`data-fps`、`data-width`、`data-height`，以及每个 scalar prop 对应的一个 `data-*`。
- 使用 `data-start` / `data-duration` / `data-track-index` 的 flat scene div list。
- 用于 layout 的 inline `<style>`；CSS 设置每个 animated property 的 `from` state。
- 底部一个 `<script>` tag，包含一个 paused `gsap.timeline({paused: true})`。每个 Remotion `useCurrentFrame()` derivation 都成为该 timeline 上正确 offset 的 tween。
- `window.__timelines["<composition-id>"] = tl;` 将 timeline 注册给 HF runtime。

Custom React subcomponents 按 prop interface 作为 template inline 成重复 HTML（per-instance `data-*` pattern 见 [`parameters.md`](references/parameters.md)）。

### Step 4: Validate

运行 eval harness —— 完整指南见 [`references/eval.md`](references/eval.md)。快速路径：

```bash
# Render Remotion baseline (after npm install in the fixture)
cd remotion-src && npx remotion render <CompositionId> out/baseline.mp4

# Render HF translation
cd ../hf-src && npx hyperframes render --skill=remotion-to-hyperframes --output ../hf.mp4

# SSIM diff
../../scripts/render_diff.sh ./remotion-src/out/baseline.mp4 ./hf.mp4 ./diff
```

Threshold: 大约低于 source complexity tier 的 `p05` 0.02（见 `eval.md` 的 validated thresholds table）。如果 diff 失败，运行 [`scripts/frame_strip.sh`](scripts/frame_strip.sh) 查看_哪些_ frames 分歧，然后重新阅读相关 timing/sequencing/media reference。

**Critical**: 两个 renders 必须使用匹配的 pixel format。在 Remotion source 的 `remotion.config.ts` 中设置 `Config.setVideoImageFormat("png")` + `Config.setColorSpace("bt709")` —— 否则 diff 测量的是 encoder differences（约 0.05 SSIM hit），不是 translation fidelity。

### Step 5: Document gaps

任何没有干净翻译的内容（volume ramps dropped、custom presentations approximated、fonts substituted）都要在 HF output 旁边写入 `TRANSLATION_NOTES.md`。格式见 [`references/limitations.md`](references/limitations.md)。

## What this skill explicitly does NOT do

- **Translate React state machines.** 通过 `useState` + `useEffect` 驱动 animation 的 compositions 在 HyperFrames seek-driven model 中不是 deterministic frame-capture targets。推荐 runtime interop pattern。
- **Run Remotion's render pipeline alongside HyperFrames.** 那是来自 [PR #214](https://github.com/heygen-com/hyperframes/pull/214) 的 runtime interop pattern —— 是对无法通过此 skill lint 的 compositions 的独立解决方案。

（`@remotion/lambda` _不是_ blocker —— Lambda config 是 deployment，不是 animation。skill 将其作为 warning 丢弃，并翻译其余部分。见 [`references/escape-hatch.md`](references/escape-hatch.md)。）

## How to grade your own translation

运行 test corpus orchestrator：

```bash
./assets/test-corpus/run.sh
```

它运行 T1、T2、T3（render + diff）和 T4（lint validation），打印 per-tier pass/fail table，并输出 aggregate JSON report。用它在 clean checkout 上端到端验证此 skill 是否工作 —— 也可在编辑任何 reference 后作为 regression check。

Validated baseline (as of 2026-04-27):

| Tier | Composition shape                           | Mean SSIM | Threshold |
| ---- | ------------------------------------------- | --------- | --------- |
| T1   | single-element fade-in                      | 0.974     | 0.95      |
| T2   | multi-scene + spring + audio + image        | 0.985     | 0.95      |
| T3   | data-driven, custom subcomponents, count-up | 0.953     | 0.90      |
| T4   | escape-hatch (8 lint cases)                 | 8/8 pass  | n/a       |
