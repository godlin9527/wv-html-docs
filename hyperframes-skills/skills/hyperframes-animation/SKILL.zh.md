---
name: hyperframes-animation
description: "HyperFrames 的全部动画知识——原子动作规则、多阶段场景蓝图、场景转场、更广泛的动效设计技术，以及七种运行时适配器（默认 GSAP，另含 Lottie、Three.js、Anime.js、CSS keyframes、Web Animations API、TypeGPU）。用于任何动作或动画任务：选择 2-4 条规则并组合，或加载一个蓝图，或查找运行时专用 API（例如 GSAP eases / Lottie player / Three.js mixer）。也涵盖审计现有 composition 的编舞（animation map）和 24 个命名文本动画效果。HyperFrames 原生：单一暂停 timeline、可 seek、确定性。"
---

# HyperFrames Animation

一个 skill 汇集全部动作知识：**规则**（原子配方）、**蓝图**（多阶段场景模板）、**转场**（场景到场景）、**技术**（更广泛的动效设计模式）和**适配器**（各运行时 API）。

关于 composition contract（data attributes、sub-compositions、determinism），请参见 `hyperframes-core`。

## 默认：组合原子规则

从 `rules-index.md` 选择 2-4 条规则，用单一暂停 GSAP timeline 把它们粘合起来，即可完成。这比从蓝图开始更快，代码也更少。

## 何时加载蓝图

- 场景匹配已有的预设计多阶段模板（brand-reveal、social-proof 等），复用其 phase pipeline 能节省实际创作时间
- 你需要复杂 4-5 阶段编舞的可运行事实基准代码

蓝图位于 `blueprints-index.md`。每个条目指向 `blueprints/<id>.md`（recipe）。不要试探性读取；只有在你已经决定需要场景级编排时才加载。

## 路由

| 想要做什么…                                                                     | 阅读                                                |
| ------------------------------------------------------------------------------- | --------------------------------------------------- |
| 按 trigger / tag 选择原子动作模式                                               | `rules-index.md`                                    |
| 阅读某条规则的完整 HTML / CSS / GSAP recipe                                     | `rules/<name>.md`                                   |
| 选择多阶段场景模板                                                              | `blueprints-index.md`                               |
| 阅读某个蓝图的完整 recipe                                                       | `blueprints/<id>.md`                                |
| 编写场景转场（CSS 驱动，位于两个 clips 之间）                                   | `transitions/overview.md`, `transitions/catalog.md` |
| 查找更广泛的动效设计技术                                                        | `techniques.md`                                     |
| 分析现有 composition 的 animation map                                           | `scripts/animation-map.mjs`                         |
| GSAP API — timeline / tweens / position parameters                              | `adapters/gsap.md`                                  |
| GSAP — 可直接使用的效果 recipe                                                  | `rules/gsap-effects.md`                             |
| GSAP — transforms / perf                                                        | `adapters/gsap-transforms-and-perf.md`              |
| GSAP — eases / stagger                                                          | `adapters/gsap-easing-and-stagger.md`               |
| GSAP — timeline / labels                                                        | `adapters/gsap-timeline-and-labels.md`              |
| Lottie / dotLottie（After Effects exports, `window.__hfLottie`）                | `adapters/lottie.md`                                |
| Three.js / WebGL（3D scenes, `AnimationMixer`, `hf-seek`）                      | `adapters/three.md`                                 |
| Anime.js (`window.__hfAnime`)                                                   | `adapters/animejs.md`                               |
| CSS keyframes (`animation-delay` / `play-state` / `fill-mode`)                  | `adapters/css-animations.md`                        |
| Web Animations API (`element.animate()`, `currentTime` seek)                    | `adapters/waapi.md`                                 |
| TypeGPU / WebGPU (`navigator.gpu`, WGSL, compute pipelines)                    | `adapters/typegpu.md`                               |
| HTML-as-texture + WebGL/GLSL post-fx（通过 `drawElementImage` 捕获实时 DOM）    | `adapters/html-in-canvas-patterns.md`               |
| 命名文本动画效果（通过外部 `animate-text` skill 的 24 个 ID）                   | `adapters/animate-text.md`                          |

## 选择运行时

- **GSAP** 是 95% 动作工作的默认选择——覆盖 timeline 编排、transforms、easing、stagger。本 skill 中的所有原子规则都基于 GSAP。
- **Lottie** 用于资产自带预烘焙 timeline 的情况（通常是 After Effects exports）。
- **Three.js** 用于 3D scenes、camera motion、shader-driven visuals。
- **Anime.js** 用于轻量 tweening，适合 GSAP 过重的场景。
- **CSS** 用于简单重复图案、装饰、shimmer——没有 JavaScript 动画成本。
- **WAAPI** 用于无需 GSAP 依赖的原生浏览器 keyframes。
- **TypeGPU / WebGPU** 用于 GPU 渲染 canvases（particles、liquid glass、custom shaders）。

多个运行时可以共存于一个 composition 中。每个运行时都把其实例注册到运行时专用 global 上，使 HyperFrames 能在一次 pass 中 seek 全部实例。

## 关键约束

**前置条件：`hyperframes-core` → Non-Negotiable Rules**（单一暂停 timeline、`data-duration` 决定长度、无 `Math.random` / `Date.now` / `performance.now`、无 `repeat: -1`、无对后续场景 clips 的 `gsap.set`、不动画 `display` / `visibility`、不在 `async` / `setTimeout` / `Promise` 内构建 timeline）。不要在这里重复说明。

在 core contract 之上的动画工艺补充：

- **预计算布局常量**——永远不要在 tween 时从 `getBoundingClientRect()` 推导位置。Tween-time DOM measurements 会因 renderer 并行采样而不同步；在 composition setup 时计算一次坐标并复用。
- **空间动作只使用 GSAP transform aliases**（`x`, `y`, `scale`, `rotation`）。Core 的 allowlist 也允许 `opacity` / `color` / `backgroundColor` / `borderRadius` 这些非空间属性 tweens——但绝不要用 `width` / `height` / `top` / `left` 做布局变化。

## Scripts

```bash
node skills/hyperframes-animation/scripts/animation-map.mjs <composition-dir> \
  --out <composition-dir>/.hyperframes/anim-map
```

读取注册在 `window.__timelines` 上的每个 GSAP timeline，枚举 tweens、采样 bboxes、计算 flags，并输出 `animation-map.json`。创作完成后用它审计编舞（dead zones、stagger consistency、lifecycle warnings）。

`animation-map.mjs` 会优先从当前项目解析 helper packages，然后可以 bootstrap bundled HyperFrames package version。仅当在 bundled CLI/skill install 之外运行该 skill 且需要显式固定 bootstrap version 时，才设置 `HYPERFRAMES_SKILL_PKG_VERSION=<version>`。

## See Also

- `hyperframes-core` — composition structure、data attributes、sub-compositions、deterministic render contract
- `hyperframes-creative` — palettes、typography、narration、beat planning（非动画创意方向）
- `hyperframes-cli` — `npx hyperframes lint / validate / inspect / preview / render`
