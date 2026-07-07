---
name: explainer-video
description: Explainer video 工作的薄路由入口。用于创建、重做、比较或审查 explainer 视频，尤其是在 faceless-explainer、既有音频 explainer、presentation-style explainer 或 style-preset 对比之间做选择时使用。
metadata:
  tags: explainer-video, orchestrator, faceless-explainer, hyperframes, workflow
---

# Explainer Video

这个 skill 只是一个很薄的路由层。它不定义另一套独立生产流程。

对于文本、主题、文章类 explainer，阅读并遵循 `../faceless-explainer/SKILL.md`；该 skill 是步骤、gate、sub-agent 派发、visual design、字幕、验证、预览和渲染的事实来源。

## Route

生产任何资产前，先选择底层工作流：

- **文本 / 主题 / 文章 -> faceless narrated explainer**：使用 `../faceless-explainer/SKILL.md`。
- **文本 / 主题 / 文章 + 用户要求用 Seed Audio 一次生成完整旁白**：使用 `../faceless-explainer/SKILL.md`，只把 Step 3.1 的音频生成替换为 `../seed-audio-explainer-audio/SKILL.md`。
- **必须保留既有音频，并用发明出来的 faceless 画面与其同步**：把 `../faceless-explainer/SKILL.md` 作为 existing-audio variant 使用。不要生成新旁白。先把用户提供的音频转成 word timings，清理/纠错 transcript，然后用这条音频时间轴驱动 storyboard timing、captions、visual design 和 render。
- **必须保留既有视频 / 露脸 / 采访 / 录屏画面**：改用 `../talking-head-recut/SKILL.md`、`../graphic-overlays/SKILL.md` 或 `../general-video/SKILL.md`。
- **产品、网站、PR、音乐驱动、幻灯片或短 motion-first 作品**：路由到对应的 HyperFrames workflow skill。

如果路线不清楚，阅读 `../hyperframes/SKILL.md`，只问最少必要的路由问题。

## Orchestration

对于 `faceless-explainer`，主 agent 仍然是 orchestrator。不要把整条视频任务交给一个普通 sub-agent。按 `../faceless-explainer/SKILL.md` 要求的 sub-agent 边界执行。

如果用户要求 clean session，尽可能启动一个干净的顶层 thread/session。prompt 保持简短：

```text
Use explainer-video as the router.
If routed to faceless, strictly follow faceless-explainer/SKILL.md.
Input: <source>.
Style preset: <preset or auto>.
Special constraints: <existing audio / Seed Audio / transcript / language / aspect>.
```

## Style Preset

如果提供了 `stylePreset`，把它传给所选工作流即可。它约束的是视觉语言，不是固定场景布局。

## Deliverables

报告 route、project directory、source path、style preset、关键 planning files、final MP4 path，以及任何对底层 workflow 的有意偏离。
