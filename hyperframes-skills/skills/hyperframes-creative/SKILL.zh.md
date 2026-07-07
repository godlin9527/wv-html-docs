---
name: hyperframes-creative
description: HyperFrames 视频的非动画创意指导。用于处理设计规范（frame.md / design.md）、调色板、字体排印、旁白、节拍规划、音频响应视觉、构图模式，以及品牌 / 风格决策。原子级运动模式和场景蓝图请使用 `hyperframes-animation`。
---

# HyperFrames Creative

品牌、节奏、风格、旁白和构图方向。在 `hyperframes-core` 的技术契约就位后使用。

运动模式、场景蓝图、转场和 CSS 标记效果请使用 `hyperframes-animation` —— 这个 skill 有意不处理动画。

> **任何非简单 composition 都要先读这两份 —— 它们会覆盖网页直觉：**
>
> - `references/house-style.md` —— “解读 prompt，生成真实内容”、惰性默认清单，以及背景/前景图层配方。这会把字面换肤变成一个 _concept_。
> - `references/video-composition.md` —— 视频媒介密度、尺度、前景元数据（“制作出来，而不是生成出来”的细节：数据条、套准标记、等宽读数、每个场景 8-10 个元素）。
>
> 跳过这些是产出泛泛、像网页页面的最大单一原因。它们不是下面路由表里的可选行 —— 对于任何超过一行编辑的任务，在选择颜色或写 HTML 前都要打开两者。

## Workflow

1. 如果项目有设计规范，**先读取它**，并把它的 frontmatter token 视为品牌真相（颜色、字体、间距、语气、约束）。读取哪个文件（优先级 `frame.md` → `design.md` → `DESIGN.md`）以及如何解析（frontmatter = 规范，正文 = 上下文）统一定义在 [`references/design-spec.md`](references/design-spec.md) 中 —— 按该文档解析并加载。
2. 如果没有设计规范，而用户要求视觉方向，选择一条路线：
   - 现成 frame-preset（可选）→ `frame-presets/`（采用某个 `FRAME.md` 作为 `frame.md`；见 `references/design-spec.md`）
   - 具名风格或情绪 → `references/visual-styles.md`
   - 快速默认值 → `references/house-style.md`
   - 交互式选择 → `references/design-picker.md`
3. 多场景工作中，先规划节拍和节奏，再写 HTML → `references/beat-direction.md`。场景转场请跳到 `hyperframes-animation/transitions/`。
4. 对运动密集型工作，先读 `references/motion-principles.md`（高层护栏），然后去 `hyperframes-animation` 查看原子规则。

## Routing

| 主题                                                                     | 阅读                                           |
| ------------------------------------------------------------------------ | ---------------------------------------------- |
| 采用现成 frame-preset 作为 `frame.md`（可选）                            | `frame-presets/` · `references/design-spec.md` |
| 默认调色板、运动、字体排印、需要质疑的惰性默认值                         | `references/house-style.md`                    |
| 具名风格预设、情绪到风格的路由                                           | `references/visual-styles.md`                  |
| 特定调色板的颜色 token                                                    | `palettes/*.md`                                |
| 构图模式 —— PiP、主体后方文字、标题卡、slide show                        | `references/composition-patterns.md`           |
| 统计 / 信息图呈现                                                         | `references/data-in-motion.md`                 |
| 面向开放式 prompt 的结构化扩展                                           | `references/prompt-expansion.md`               |
| 视频媒介密度、尺度、颜色、画面构图                                       | `references/video-composition.md`              |
| 按节拍指导、节奏规划、转场时机                                           | `references/beat-direction.md`                 |
| 创作后规范验证（颜色、字体、圆角、间距、深度）                           | `references/design-adherence.md`               |
| 高层运动护栏和 GSAP 质量规则                                             | `references/motion-principles.md`              |
| 字体选择、搭配、渲染视频中的文字护栏                                     | `references/typography.md`                     |
| 脚本节奏、语气、开场、数字读法                                           | `references/narration.md`                      |
| 映射到运动的预计算音频频段                                               | `references/audio-reactive.md`                 |

## Scripts

- `scripts/contrast-report.mjs` —— 检查渲染帧中的对比度警告。
- `scripts/extract-audio-data.py` —— 为音频响应 composition 预提取音频频段。
- `scripts/package-loader.mjs` —— 捆绑创意工具的支持脚本。

`contrast-report.mjs` 会优先从当前项目解析 helper package，然后可以 bootstrap 捆绑的 HyperFrames package 版本。只有在捆绑 CLI/skill 安装之外运行该 skill，且需要显式固定 bootstrap 版本时，才设置 `HYPERFRAMES_SKILL_PKG_VERSION=<version>`。

从 repo root 运行并使用显式路径，例如：

```bash
python skills/hyperframes-creative/scripts/extract-audio-data.py <audio-file>
```

动画分析（`animation-map.mjs`）位于 `hyperframes-animation/scripts/`。

## Boundaries

- 不要覆盖 `hyperframes-core` 的技术规则。
- 不要为了最小技术 composition 强制要求设计系统。
- 除非请求要求，或你先提出扩展方案，否则不要添加额外场景、旁白、音乐、字幕或转场。
- recipe 引用要针对具体任务；简单编辑不要读取所有 reference。
