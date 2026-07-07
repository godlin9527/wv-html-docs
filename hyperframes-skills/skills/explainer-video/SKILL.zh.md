---
name: explainer-video
description: 顶层 explainer video 编排器。用于创建、重做、比较或审查 explainer 视频，尤其是在用户需要清晰的标准工作流、faceless-explainer、演示风格 explainer、风格预设比较，或可复现的视频生产流程，而不是一次性 HTML 场景时使用。
metadata:
  tags: explainer-video, orchestrator, faceless-explainer, hyperframes, workflow
---

# Explainer Video

这是一个包装/编排 skill。它不会替代 `faceless-explainer`；它负责决定路线、加载正确的底层 skill，并防止工作流被压扁成单个 agent 的一次性执行。

## Routing

在产出任何内容之前，先对请求进行分类：

- **Text/topic/article -> faceless narrated explainer**：阅读并遵循 `../faceless-explainer/SKILL.md`。
- **Existing audio/video must be preserved and visuals sync to it**：使用 `../general-video/SKILL.md`，除非用户明确接受 faceless-explainer 变体。
- **Presentation/deck/slides only**：如果可用，使用 `slideshow` 或 `frontend-slides`。
- **Talking-head/interview footage with graphic packaging**：使用 `graphic-overlays` 或 `talking-head-recut`。
- **Product/site/URL demo**：使用 `product-launch-video` 或 `website-to-video`。
- **Short motion-first concept piece**：使用 `motion-graphics`。

如果路由到 `faceless-explainer`，该底层 skill 就是生产步骤的事实来源。

## Orchestration Rule

对于完整的 `faceless-explainer` 运行，主 agent 必须作为编排器行动。

不要把整个视频分配给一个 subagent。普通 subagent 不能派发 subagent，因此无法执行官方 faceless 工作流。只在 `faceless-explainer/SKILL.md` 要求的角色上使用 subagent：

- scriptwriting
- visual-design
- scene workers
- finalize

主 agent 运行确定性的 shell/script 阶段：

`init -> scaffold -> design-system -> audio -> prep -> captions -> assemble -> preflight`

然后在必要节点派发对应角色的 subagent。

## Clean Session Protocol

当用户要求 clean session 时，尽可能创建或使用一个独立的顶层 Codex thread，而不是 subagent。clean session prompt 应该简短且明确：

```text
Strictly follow explainer-video, then faceless-explainer if routed there.
You are the main orchestrator. Do not give the full video task to one subagent.
Run deterministic phases yourself and dispatch the required role subagents.
Use PROJECT_DIR: <path>.
Input: <brief/source>.
Style preset: <preset or auto>.
Special constraints: <audio/transcript/language/aspect/etc>.
```

普通 subagent 只用于当前被编排运行中的有边界角色工作。

## Style Presets

风格预设是一种视觉语言，不是固定布局。指定预设时，保留它的 token 和情绪，但要求场景多样性：

- 避免只改文字、重复同一种结构；
- 按场景改变信息架构：title card、contrast、map、timeline、flow、stack、network、ledger、quote、checklist、proof wall 等；
- 每个场景都应该有一个主导视觉机制，而不只是标题加卡片；
- 如果比较预设，使用独立项目目录，并保持源内容完全一致。

## Visual Design Gate

在 scene workers 开始之前，审查 `section_plan.md` 是否存在视觉同质化。如果大多数场景共享同一种布局模式，拒绝并重新做视觉设计，再生成场景。

最低检查：

- 每个场景都有不同的视觉机制，或一个有意延续的机制；
- 每个场景描述说明屏幕上发生了什么变化，而不仅是出现了什么文字；
- 转场在有用时保留一个可见的连续性元素；
- 如果启用 captions，需要预留空间。

## Audio Variants

标准 `faceless-explainer` 会生成自己的旁白。如果用户要求将 Seed Audio 作为单条完整影片旁白使用，则使用 `seed-audio-explainer-audio` 作为受控变体，但仍然让主 agent 编排 faceless pipeline 的其余部分。

如果用户提供了不能改动的原始音频，将其视为 existing-audio 工作流，并考虑使用 `general-video`，除非用户明确想要一个使用该音频的 faceless 变体。

## Deliverables

对于一次生产运行，报告：

- project directory；
- source brief/input path；
- chosen route and style preset；
- generated script / transcript / visual plan paths；
- final mp4 path；
- 是否遵循了官方工作流，或在哪些地方有意偏离。
