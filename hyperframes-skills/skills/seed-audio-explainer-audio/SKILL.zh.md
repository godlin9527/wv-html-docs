---
name: seed-audio-explainer-audio
description: "用于通过 Seed Audio 生成 explainer video 音频：一条完整的旁白优先音轨，可选 BGM/SFX，然后基于转录进行 timing。"
---

# Seed Audio Explainer Audio

## Overview

当 explainer、faceless video、motion graphic 或 storyboard 需要用 Seed Audio 生成音频底座时，使用这个 skill。

生成一个完整的全片音频文件。不要分段生成旁白再后期拼接。场景或 beat 边界仍然有用，但只作为 prompt 内部的时间顺序和 timing 提示。生成之后，转录完整音频，并让转录驱动 animation timing。

硬规则：绝不要为主 explainer 音轨使用逐场景、逐镜头或逐段落音频生成。

## Workflow

1. 锁定旁白。
2. 根据旁白和 beat plan 准备 audio prompt。
3. 使用 Seed Audio 生成一条完整音轨音频文件。
4. 收听并验证音频。
5. 转录完整音频。
6. 将 storyboard、beats、captions 和 HyperFrames timing 对齐到真实转录。

在音频存在之前，不要把计划中的 beat durations 当作最终 timing。真实音频才是 timing 的事实来源。

## Inputs

生成前收集或推断这些内容：

- Locked narration：要朗读的精确文字。
- Target duration：大致完整音轨长度。
- Audience and tone：例如直接的中文 explainer、快但清楚、略带挑衅、不戏剧化。
- Beat order：视频的语义顺序。
- Audio style：仅旁白、旁白加轻 BGM，或旁白加克制 SFX。
- Output path：通常在 `assets/audio/`、`assets/voice/` 或项目约定目录下。

如果旁白还在变化，不要生成生产音频。只创建 dry run，或继续打磨脚本。

## Prompt Strategy

Seed Audio 应该作为音频生成模型来 prompt，而不是普通 TTS。

按时间顺序编写 prompt。把所有 spoken text 放在清楚标记的 narration block 中，并让 sound-direction text 足够短，降低它被误读出来的概率。

对于 explainer videos：

- 除非 brief 明确需要多种声音，否则使用一个旁白。
- 优先清晰旁白，而不是戏剧化表演。
- 只在有助于节奏时使用 BGM；让它低于人声。
- 谨慎使用 SFX：UI blips、subtle whoosh、soft impact、typing、notification 或 transition accents。
- 避免密集的场景描述。音频生成应该支撑视频，而不是复述 storyboard。
- 不要把每个视觉 beat 都塞进 audio prompt。
- 避免在 spoken narration 本身中使用 "pause"、"sfx" 或括号指令之类的词。

## Full-Track Only

主 explainer 音频使用一条完整音轨请求。这一点对以下情况尤其重要：

- Opening samples。
- 大约 1-2 分钟以内的 explainers。
- 语音节奏和音乐连续性很重要的视频。
- 需要单条连续 timing spine 的 motion graphics。

如果脚本无法可靠放进单次请求，不要拆成场景音频。而是：

- 收紧旁白。
- 减少非朗读的 sound-design 指令。
- 先生成一个较短 sample。
- 使用支持所需长度的 provider mode 或 model configuration。
- 停止并报告 full-track generation 被阻塞。

不要用多次旁白生成拼接出主音轨。

## Prompt Template

使用这个形状并按项目调整：

```text
目标：生成一条完整的中文 explainer 视频音频，时长约 {duration} 秒。

声音：一位中文旁白，语速偏快但清楚，语气直接、有观点、有解释感，不要播音腔，不要夸张表演。

音乐：极轻的现代科技感底噪，音量低于旁白，不抢词。没有明显旋律，不要热血宣传片感。

音效：只在少数关键转场加入很轻的 UI click / whoosh / soft hit。不要频繁使用音效。

结构：按下面顺序生成。每一段“旁白：”后的文字是需要朗读的台词。声音设计只用于控制节奏，不要盖过旁白。

00:00-00:04
声音设计：干净开场，轻微冲击。
旁白：Harness Engineering 还没整明白呢，Loop Engineering 又火了。

00:04-00:08
声音设计：节奏略微加快。
旁白：我们先把它叫作“循环工程”。

...

结尾：最后一句结束后留 0.3 秒自然收尾，不要额外口播。
```

让 spoken text 尽可能贴近 locked narration。prompt 可以包含 beat timestamps，但这些 timestamps 是指导，不是保证。

## API Shape

使用当前 Seed Audio API 文档确认精确 endpoint 和 headers。不要把 API keys 硬编码到 scripts、skills、prompts 或 source files 中。

请求通常包含：

- `model`：例如 `seed-audio-1.0`，如果它是所选 endpoint 文档中的 model ID。
- `text_prompt`：完整 audio prompt。
- `references`：可选 reference audio list，在支持时使用。
- `audio_config`：根据 API docs 设置 output format、sample rate、speech rate、loudness、pitch 和 watermark options。

除非当前官方 API docs 或可运行示例确认了 `speaker` parameter，否则不要依赖它。对于 voice control，优先使用 prompt wording 和可选 reference audio。

## Reference Audio

只有在需要稳定 narrator voice 时才使用 reference audio。

规则：

- 尽量使用一个干净的 narrator reference。
- 每个 reference 保持简短，理想情况下低于 30 秒。
- 除非 API 明确支持所选 mode 的 image references 和 audio references 组合，否则不要混用。
- 如果 API 支持 `@audio1` 这类 labels，在 prompt 中清楚标记 references。

对于大多数 first drafts，不使用 reference audio 也可以。先验证 pacing 和 narration。

## Validation

在接入 animation 前始终验证音频：

1. 收听完整文件。
2. 检查是否有任何 instruction text 被意外朗读。
3. 检查开头或结尾是否被截断。
4. 检查语速是否过快或过慢。
5. 检查 BGM 或 SFX 是否盖住文字。
6. 检查 pauses 是否落在重要概念转换附近。
7. 转录完整音频。
8. 将 transcript 与 locked narration 比较。

如果 transcript 有实质差异，重新生成或简化 prompt。不要围绕有缺陷的音频文件修补 animation timing。

## HyperFrames Integration

对于 HyperFrames projects：

- 将完整音频文件存为单个 asset。
- 将 transcript timing 存为 metadata。
- 让 storyboard beats 引用 transcript ranges。
- 使用从 transcript anchors 派生出的 per-shot timings，而不是猜测的 script durations。
- 在 main timeline 中保持一条完整音轨。

目标是：

```text
locked narration -> one full audio file -> transcript -> beat timing -> animation
```

不要把它作为主工作流：

```text
scene script -> scene audio -> stitched audio -> manually repaired timing
```

## Common Failure Modes

- 过早生成 per-shot audio，然后改写脚本，导致所有 timing 丢失。
- 把旁白拆成 scene files，再试图隐藏拼接点。
- 把 planned beat durations 当作最终值。
- 添加过多 sound-design text，导致模型困惑。
- 让 BGM 比旁白更重要。
- 在中文 explainer 中不一致地使用英文技术词。
- 要求 Seed Audio 解决本该通过视觉解决的 storyboard 问题。
- 跳过 transcript extraction，试图凭直觉做 animation。

## Output Checklist

对于可用交付物，产出或更新：

- 完整音频文件。
- 带 timestamps 的 transcript。
- timeline 使用的 audio metadata。
- 关于任何 regenerated sections 或 known timing issues 的 notes。
- 使用过的精确 prompt，保存时不包含 secrets。
