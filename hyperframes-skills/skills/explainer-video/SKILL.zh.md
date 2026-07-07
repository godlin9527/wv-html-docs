---
name: explainer-video
description: faceless-explainer 的薄适配层。用于创建、重做、比较或审查 faceless explainer 视频，包括标准 faceless-explainer 流程、Seed Audio 单条完整旁白变体，或既有音频 faceless 变体。
metadata:
  tags: explainer-video, orchestrator, faceless-explainer, hyperframes, workflow
---

# Explainer Video

这个 skill 只是 `faceless-explainer` 的薄适配层。它不定义另一套独立生产流程，也不是 HyperFrames 的通用路由器。

始终阅读并遵循 `../faceless-explainer/SKILL.md`；该 skill 是步骤、gate、sub-agent 派发、visual design、字幕、验证、预览和渲染的事实来源。

## Supported Modes

只在以下 faceless 模式中选择一种：

- **标准 faceless explainer**：文本 / 主题 / 文章 -> faceless narrated explainer。严格遵循 `../faceless-explainer/SKILL.md`。
- **Seed Audio 单条完整旁白变体**：遵循 `../faceless-explainer/SKILL.md`，只把 Step 3.1 的音频生成替换为 `../seed-audio-explainer-audio/SKILL.md`。
- **既有音频 faceless 变体**：保留用户提供的音频。不要生成新旁白。先把音频转成 word timings，清理/纠错 transcript，然后用这条音频时间轴驱动 storyboard timing、captions、visual design 和 render。

如果用户要做产品、网站、PR、幻灯片、音乐驱动、露脸、采访或录屏素材类 workflow，这个 skill 不处理。告诉用户从 `../hyperframes/SKILL.md` 或对应 workflow skill 开始。

## Orchestration

对于 `faceless-explainer`，主 agent 仍然是 orchestrator。不要把整条视频任务交给一个普通 sub-agent。按 `../faceless-explainer/SKILL.md` 要求的 sub-agent 边界执行。

如果用户要求 clean session，尽可能启动一个干净的顶层 thread/session。prompt 保持简短：

```text
Use explainer-video.
Strictly follow faceless-explainer/SKILL.md.
Input: <source>.
Style preset: <preset or auto>.
Special constraints: <existing audio / Seed Audio / transcript / language / aspect>.
```

## Style Preset

如果提供了 `stylePreset`，把它传给所选工作流即可。它约束的是视觉语言，不是固定场景布局。

## Deliverables

报告 faceless mode、project directory、source path、style preset、关键 planning files、final MP4 path，以及任何对 `faceless-explainer` 的有意偏离。
