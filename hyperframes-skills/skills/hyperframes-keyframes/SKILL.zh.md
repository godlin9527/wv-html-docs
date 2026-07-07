---
name: hyperframes-keyframes
description: >
  当 HyperFrames composition 需要 seek-safe 2D/3D keyframes、GSAP
  timelines、CSS keyframes、Anime.js、WAAPI、FLIP、路径、遮罩、SVG morph/draw、
  text trails、3D depth，或 `hyperframes keyframes` diagnostics 时使用。
  不要用于宽泛的场景策略、品牌设计、媒体 sourcing、字幕，或
  通用视频规划。
---

# HyperFrames Keyframes

Keyframes 是一个姿态契约：可见状态、连续的主体身份、seek-safe runtime、经过验证的像素。

宽泛场景 recipe 使用 `hyperframes-animation`。
完整命令文档使用 `hyperframes-cli`。
仅在选择实现机制时使用 `references/keyframe-patterns.md`，不要用它决定视觉风格。

## Procedure

1. 识别动画主体、可见状态、最终状态和 runtime。
2. 选择能证明 prompt 的最小机制。只有机制不清楚时才读取 `references/keyframe-patterns.md`。
3. 在声明的 runtime 中创作 seek-safe keyframes。同步构建并注册 runtime instance。
4. 用 lint、validate、`hyperframes keyframes`、一个聚焦的 `--shot`，以及 proof times 的 snapshots 验证。
5. 如果 proof 失败，修复源 keyframes，并在 render 前重新运行最小的失败 diagnostic。

## Contract

- 命名运动主体。
- 命名证明预期运动所需的姿态，包括最终状态。
- Keyframe 可见 channel，而不是隐藏的 helper state。
- 当连续性重要时，保留 object identity。
- 只有当预期运动是替换或 dissolve 时才 crossfade。
- 可读或语义状态要保持足够长，确保能看见。
- 最后一帧是动画的一部分，不是 cleanup。
- 除非请求要求，不要 reset to rest。
- 除非请求要求，不要以黑屏结束。
- 如果编辑 starter scene，除非要求 redesign，否则保留 layout、copy、assets、colors 和 final state。

## Runtime Rules

GSAP:

- 在 page load 时同步构建
- 使用 `gsap.timeline({ paused: true })`
- 注册为 `window.__timelines[compositionId]`
- registry key 必须匹配 `data-composition-id`
- 不要为 render-critical motion 调用 `tl.play()`
- repeats 保持 finite

CSS keyframes:

- 有限 duration 和 iteration count
- 确定性 delay
- `animation-fill-mode: both`
- 当 timing 属于 clip 时使用 `data-start`

Anime.js:

- 同步创建
- `autoplay: false`
- finite duration 和 loops
- 将每个 instance push 到 `window.__hfAnime`

WAAPI:

- finite `duration`
- `fill: "both"`
- 确定性 construction
- text surface 不列出 WAAPI；用 `--shot`（它会 seek WAAPI）和 snapshots 验证

永远不要用于 render-critical motion：

- `Date.now()`
- `performance.now()`
- 未设 seed 的 `Math.random()`
- hover/scroll triggers
- timers
- async-created timelines
- 未注册的 `requestAnimationFrame`
- infinite loops

## GSAP Skeleton

```js
const root = document.querySelector("[data-composition-id]");
const compositionId = root.dataset.compositionId;
const tl = gsap.timeline({ paused: true });

tl.addLabel("state-a", 0);
tl.to(".subject", {
  keyframes: [
    { x: 0, opacity: 1, duration: 0.2 },
    { x: 120, opacity: 1, duration: 0.4, ease: "power2.out" },
    { x: 100, opacity: 1, duration: 0.2, ease: "power2.inOut" },
  ],
  ease: "none",
});

window.__timelines = window.__timelines || {};
window.__timelines[compositionId] = tl;
```

用 labels 表示语义状态。
使用 position parameters，而不是 chained delays。
对后续触碰同一 property 的 `from()`/`fromTo()` tweens 使用 `immediateRender: false`。

## Keyframe Forms

- Array keyframes：带有每步 duration/ease 的 pose ladder。
- Percentage keyframes：一个 tween 内的精确 timing。
- Property arrays：紧凑的 multi-stop changes。
- 当每个 stop 都带自己的 easing 时，parent 上使用 `ease: "none"`。
- 当每个 segment 都应共享同一感觉时使用 `easeEach`。

不要从示例复制数值距离或 timing。要从实际 composition geometry 和 duration 推导。

对于一个主体在两个 box 之间移动，优先使用一个连续 transform tween 或 FLIP。只有当观众应感受到不同 beats 时，才把 `x/y/scale` 拆成多个 eased keyframes；每个 segment 都会改变速度，可能读起来像 hitch。

## Channels

优先使用 compositor/visual channels：
`x/y/z`、`xPercent/yPercent`、`scale`、`rotationX/Y/Z`、`skew`、`transformOrigin`、`svgOrigin`、`opacity`、`autoAlpha`、`clip-path`、masks、CSS vars、SVG path/dash values、camera transforms、shader uniforms。

避免 layout/lifecycle channels：
`top/left/right/bottom`、`width/height`、`margin/padding`、`display`、`visibility`、late DOM creation、执行 subject motion 的 helper overlays。

## Mechanism Choice

选择能证明 prompt 的最小机制：

| 需求                                  | 机制                                               |
| ------------------------------------- | -------------------------------------------------- |
| 同一主体改变 box 或 hierarchy         | shared element / FLIP                              |
| 主体沿可见路线移动                    | path travel                                        |
| Stroke 生长或描绘                     | stroke draw                                        |
| 一个形状变成另一个形状                | shape interpolation                                |
| Reveal 边界可见                       | clip, mask, or shader uniform                      |
| 许多 items 按顺序运动                 | stagger / indexed delay                            |
| 文本自身运动                          | line, word, character, or band subdivision         |
| Surface 弯曲、拉伸或裁切              | parent/child counter-transform                     |
| UI 有状态                             | explicit state machine                             |
| Scene 有深度                          | DOM 3D, Three.js, or WebGL camera/object keyframes |

机制可以组合，但每一个都必须让想法更清楚。装饰不是 proof。

## Timing

- Anticipation 只在能澄清原因或方向时使用。
- Acceleration 离开静止状态。
- Peak proof 要 unmistakably 展示机制。
- Follow-through 传达能量和方向。
- Overshoot 只在主体应有弹性或触感时使用。
- Constant-speed path travel 通常需要 `ease: "none"`。
- Discrete UI states 通常需要 sharp ease-out。
- Repeated elements 需要 ordered offsets，而不是 identical timing。
- Final lockups 需要比 transition poses 更长的 holds。
- Smoothness 意味着同一主体上的连续速度。
- 除非 overlap 是有意且已验证的，否则不要让写入同一 transform property 的 tweens 重叠。
- 避免在同一个 hero surface 还在 scaling 或 traveling 时动画化大幅 `clip-path`/mask changes；主移动 settled 后使用 nested reveals。

## Text

保留 line boxes、word spacing、readability 和 final fit。如果文本内部运动，移动 glyphs 或 masked bands，而不只是移动文本周围的装饰。对可读帧做 snapshot。

## SVG

Stroke growth 优先使用 `DrawSVGPlugin`，然后是 `stroke-dasharray`/`stroke-dashoffset`。
Shape interpolation 优先使用 `MorphSVGPlugin`；需要时把 primitives 转成 paths，并把复杂 silhouettes 拆成更简单的 parts。

## 3D

单独 scale 是假的 depth。
当物体交叉时，使用稳定 parent 上的 perspective、`transform-style: preserve-3d`、z travel、rotation、camera/world motion、occlusion 和 layer order。

使用一两个能暴露 depth relationship 的 diagnostic angles。如果 angled proof 显示没有 depth crossing，改进 z/camera/occlusion。

## Canvas / WebGL

通过 deterministic state keyframe camera position、camera target、object transform、material opacity、shader uniforms 和 postprocess intensity。从 HyperFrames time 渲染。使用 `--ghost`，因为 marker boxes 看不到内部 canvas motion。

## CLI Proof

```bash
npx hyperframes lint
npx hyperframes validate
npx hyperframes keyframes .
npx hyperframes keyframes . --json
npx hyperframes keyframes . --runtime all
npx hyperframes keyframes . --selector "<selector>" --shot "<file>" --samples <n>
npx hyperframes keyframes . --selector "<selector>" --shot "<file>" --layout strip --from <t0> --to <t1>
npx hyperframes keyframes . --shot "<file>" --ghost --angle <angle>
npx hyperframes snapshot . --at <times>
```

为真实动画主体选择 `<selector>`。
为 first frame、proof poses、final-minus-hold 和 exact final 选择 `<times>`。
只有必须证明 depth 时才选择 `<angle>`。

| Tool             | Proves                                                                                              |
| ---------------- | --------------------------------------------------------------------------------------------------- |
| `keyframes`      | targets、explicit stops、paths、traces、composed parent/child motion、CSS stops、Anime registration |
| `--shot`         | ghosts、route shape、time spacing、DOM 3D projection、focused selector proof                        |
| `--layout strip` | in-place motion、overlaps、contact、subtle scale/opacity、text waves                                |
| `--ghost`        | canvas、WebGL、shader motion、rendered 3D                                                           |
| `snapshot --at`  | masks、text readability、full state、final lockup、black/reset tails                                |

如果 selector proof 看起来不对：

1. 重新运行 `--json`
2. 找到实际动画目标
3. 拍摄该目标
4. snapshot full frames
5. 相信 painted pixels，而不是 logs

## Diagnostic Reading

`flat` 表示没有显式 middle poses。`keyframes` 表示存在 explicit stops。`motionPath` 表示存在 route。`trace` 表示 multi-stroke drawing。`composed with` 表示 child motion 继承 parent motion。

均匀 ghost spacing 表示 constant speed。聚集的 ghosts 表示 slow-in 或 settle。大间隙表示 fast travel。

helper-selector shot 不是 proof。broken full frame 上的 onion shot 不是 proof。

## Error Handling

| Failure            | Fix                                                                                |
| ------------------ | ---------------------------------------------------------------------------------- |
| endpoint-only      | 添加 middle poses，保持 peak proof，重新运行 `--shot`                              |
| identity break     | 保持一个 element alive，使用 shared source/final boxes，移除 substitute crossfade |
| fake 3D            | 添加 z/camera travel、occlusion、angled proof                                      |
| wrong final        | 添加 final hold，snapshot final-minus-hold 和 exact final                          |
| unseekable runtime | pause autoplay，register instance，移除 timers，同步构建                           |
| unreadable text    | 保留 line boxes，减少 displacement，添加 final hold，snapshot text frames          |

## Done

运行 lint、validate、keyframes、一个聚焦的 `--shot` 和 snapshots。确认 first frame、proof poses、final-minus-hold、exact final、subject-owned motion，并且没有 debug overlays。
