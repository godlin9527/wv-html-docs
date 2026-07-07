---
name: hyperframes-media
description: HyperFrames composition 的音频和媒体资产，由一个共享音频引擎（`scripts/audio.mjs`）生成 —— 多提供商 TTS（HeyGen / ElevenLabs / Kokoro local）、背景音乐 + 音效（默认通过 HeyGen audio-library 检索，本地 Lyria / MusicGen BGM 生成和捆绑 SFX 库作为无凭据 fallback）、Whisper 转写、背景移除和字幕创作。用于 voiceover / TTS、BGM、SFX / sound effects、转写、captions / subtitles / lyrics / karaoke / 逐词样式、声音 + provider 选择，以及音乐情绪 prompt。
---

# HyperFrames Media

创建 composition 所需的音频和媒体资产 —— voiceover（TTS）、背景音乐 + 音效、转写、字幕、背景移除 —— 然后在 HTML 中消费并动画化这些数据。将资产放入 composition 请见 `hyperframes-core`。

## 音频引擎 —— TTS · BGM · SFX 的唯一来源

Workflow 不要手写音频，也不要 vendored copy。只有一个引擎 —— **`scripts/audio.mjs`** —— 它接收中性的 `audio_request.json`，并写入 `audio_meta.json`（以及 `assets/voice|bgm|sfx` 下的资产）：

```bash
# <MEDIA_DIR> = this skill's directory
node <MEDIA_DIR>/scripts/audio.mjs --request ./audio_request.json --hyperframes . --out ./audio_meta.json
```

三个能力都在**一个开关**上降级 —— 是否存在 HeyGen 凭据（从 `$HEYGEN_API_KEY` / `$HYPERFRAMES_API_KEY` / `~/.heygen` 解析，**不是** CLI）：

| 能力 | 存在 HeyGen 凭据                                    | 不存在                                               |
| ---- | --------------------------------------------------- | ---------------------------------------------------- |
| TTS  | HeyGen Starfish REST（原生 word timestamps）        | → ElevenLabs → Kokoro（串联 `transcribe` 获取 words） |
| BGM  | HeyGen music **retrieval**                          | Lyria → MusicGen local **generation**（detached）     |
| SFX  | HeyGen sound-effects **retrieval**（min_score 0.4） | 捆绑 21-file library（`assets/sfx/`）                 |

- **Request**（`audio_request.json`）：`{ provider?, lang?, speed?, lines: [{ id, text, sfx?: [names] }], bgm: { mode?, query?, prompt? } }`。`id` 会把每一行关联回调用方模型（帧号、场景 id，等等）。`bgm.mode` = `retrieve | generate | none`；省略则自动（有凭据时 retrieve，否则 generate）。显式的 `retrieve` 是严格的 —— 它会跳过，而不是启动 detached generate（面向没有 `wait-bgm` 步骤的调用方）。
- **Output**（`audio_meta.json`，按 id keyed）：`{ tts_provider, voice_id, bgm, bgm_pending, …, voices: [{ id, path, duration_s, words }], sfx: [{ id, name, file, source, offset_s, duration_s, volume }], total_duration_s }`。
- `--only tts,bgm,sfx` 运行子集并**合并**到已有的 `--out`（例如早期生成 TTS+BGM，有 cue 后再生成 SFX）。
- BGM generate 会以 **detached** 方式启动（`bgm_pending: true`）—— assemble 前运行 `scripts/wait-bgm.mjs`。
- `scripts/heygen-tts.mjs` 是基于同一代码的单次 CLI（一个 text → wav + words），用于只需要 HeyGen TTS 且不想写 request file 的场景。

完整 flag 列表和 `audio_meta.json` schema 位于 `scripts/audio.mjs` 的头部。下面的 reference 覆盖每项能力背后的 provider 细节和边界情况。

## Preflight —— 任何音频前显示登录状态

**生成 voice 或 BGM 之前始终运行这个 —— 无论是在完整 workflow 中，还是一次性的“给我生成 BGM/voiceover”请求。** 没有 HeyGen 凭据**不是**静默 fallback 到本地引擎的理由：先建议登录，让用户决定。运行共享 preflight 并**逐字转述其输出** —— 不要即兴写自己的“missing key”提示，也不要提议把 key 写进每个 repo 的 `.env`：

```bash
npx hyperframes auth status
```

- **Signed in** → 它会打印账户；继续。
- **Not signed in**（这里 `exit 1` 是预期的 —— “not signed in” 是正常状态，不是失败）→ 它会打印 registration-first 指引。建议登录：`npx hyperframes auth login` 是 browser OAuth —— 它会**登录并创建账户**（始终可通过此 repo 的 CLI 使用）。要使用现有 HeyGen API key（来自 app.heygen.com/settings/api），运行 `npx hyperframes auth login --api-key` —— 它会保存到共享的 `~/.heygen`（没有 per-repo `.env`）。输出也会列出 voice/BGM 会 fallback 到的本地引擎，以及依赖缺失时的 `pip` 提示。**原样转述该输出 —— 不要改写成自己的措辞。** 然后**停止并等待**用户选择 —— 登录，或者说 “go” / “local” 以离线继续 —— **在生成任何东西之前。** 这是一个真实决策点，不是顺带提示：不要把它合并进另一个问题，也不要自行越过它。（例外：在 autonomous / non-interactive 模式下，记录状态并离线继续。）
- `npx hyperframes auth status --json` 返回 `{ configured, recommended_action, offline_engines }`，用于确定性分支。
- **如果 CLI 无法运行**（不在 PATH，且 `npx` 无法 fetch）→ 仍然**建议登录**（`npx hyperframes auth login`）并**停止等待用户选择** —— 不要把“没有凭据”视为本地生成的静默绿灯。

凭据解析、完整 key 优先级和本地依赖清单位于 `references/requirements.md`。

## Provider chains（引擎背后的细节）

**TTS** —— 第一个可用 provider 获胜（引擎，或 `npx hyperframes tts "..."`）：

| 顺序 | Provider                      | 检测条件                                      | Word timestamps                                                   |
| ---- | ----------------------------- | --------------------------------------------- | ----------------------------------------------------------------- |
| 1    | HeyGen（Starfish）            | `$HEYGEN_API_KEY` / `hyperframes auth login`  | **Yes, native** —— 传 `--words narration.words.json` 来捕获       |
| 2    | ElevenLabs                    | 设置了 `$ELEVENLABS_API_KEY`                  | No —— 后接 `transcribe`                                           |
| 3    | Kokoro-82M（local, 54 voices） | 始终可用（无需 key）                          | No —— 后接 `transcribe`                                           |

> 已发布的 `hyperframes tts` CLI 往往是 local-only build（它的 `--help` 显示 “Kokoro-82M”，没有 `--provider`/`--words`），即使设置了 `$HEYGEN_API_KEY` 也会静默 fallback 到 Kokoro。这就是为什么引擎的 HeyGen 路径是自包含的 `scripts/heygen-tts.mjs`（REST），而**不是** CLI；CLI 只用于 Kokoro 路径。见 `references/tts.md`。

**BGM & SFX** —— 默认从 HeyGen audio library（`/v3/audio/sounds`）**检索**，使用与 HeyGen TTS 相同的凭据；无凭据 fallback 如上面的开关：

| Asset | HeyGen `type`                   | 落地位置                                                   | Fallback（无凭据）                                       |
| ----- | ------------------------------- | ---------------------------------------------------------- | -------------------------------------------------------- |
| BGM   | `music`                         | `assets/bgm/track.mp3`（retrieve） · `track.wav`（generate） | Lyria / MusicGen generation                              |
| SFX   | `sound_effects`（min_score 0.4） | `assets/sfx/<slug>.mp3`                                    | 捆绑 21-file library（`assets/sfx/*` + `manifest.json`） |

见 `references/bgm.md` 和 `references/sfx.md`。

## Routing

| 任务                                                                 | 阅读                                         |
| -------------------------------------------------------------------- | -------------------------------------------- |
| 音频引擎 —— request/meta schema、`--only`、开关                      | `scripts/audio.mjs`（header comment）        |
| `npx hyperframes tts` / `heygen-tts.mjs` —— providers、voices、words | `references/tts.md`                          |
| BGM —— HeyGen retrieval + local Lyria / MusicGen generation          | `references/bgm.md`                          |
| SFX —— HeyGen retrieval（min_score 0.4）+ 捆绑本地库                 | `references/sfx.md`                          |
| `npx hyperframes transcribe` —— Whisper、model 规则、输出形状        | `references/transcribe.md`                   |
| `npx hyperframes remove-background` —— transparent cutouts           | `references/remove-background.md`            |
| TTS → transcription → captions（无录制 voiceover）                   | `references/tts-to-captions.md`              |
| Caption authoring —— style detection、layout、word grouping、exit    | `references/captions/authoring.md`           |
| Transcript handling —— input formats、quality gates、cleanup、APIs   | `references/captions/transcript-handling.md` |
| Caption motion —— karaoke、marker effects、audio-reactive            | `references/captions/motion.md`              |
| Model caches、system dependencies、troubleshooting                   | `references/requirements.md`                 |

## Non-negotiable rules

- **一个引擎，不要 vendored copies。** 通过 `scripts/audio.mjs` 生成音频（或用 `heygen-tts.mjs` 做一次性 HeyGen TTS）。不要在 workflow 内重新实现 TTS/BGM/SFX —— 写一个 `audio_request.json` adapter 并调用引擎。
- **“HeyGen available” = 可解析的凭据，不是 CLI。** 整个开关以 `heygenCredential()` 为准；已发布的 `hyperframes tts` 可能只支持 Kokoro，而且根本没有 `hyperframes bgm` / `hyperframes sfx` 命令。
- **Voice IDs 是 provider-specific。** `am_michael` 仅属于 Kokoro；HeyGen UUID 不能用于 Kokoro。如果传 `--voice`，也要固定 `--provider`，避免用户环境变化时 provider 静默漂移。
- **始终给 `transcribe` 传 `--model`。** CLI 默认 `small.en` 会静默翻译非英语音频。见 `references/transcribe.md` → “Language Rule”。
- **HeyGen 返回 word timestamps；ElevenLabs / Kokoro 不返回。** 引擎会为后两者自动串联 `transcribe`；独立使用时，给 HeyGen 传 `--words` 或对音频文件运行 `transcribe`。
- **Captions 消费扁平 word-array format**，形如 `{ id, text, start, end }`。见 `references/transcribe.md` → “Output Shape”。
- **`remove-background --background-output` 是 hole-cut，不是 inpainted。** 对于“没有人物的场景”，需要另一个工具。见 `references/remove-background.md` → “When NOT the right tool”。
- **BGM/SFX 默认走 HeyGen retrieval；无凭据 fallback 是 generation（BGM）或捆绑库（SFX）。** `/audio/sounds` 按文本 query 排名 —— 具体命名效果（`glass shatter`，不要写 `dramatic sound`）；无匹配会**跳过**，绝不阻塞 render。SFX 在 voice + BGM 下方约 0.35 音量。见 `references/sfx.md` / `references/bgm.md`。
- **将 workflow caption HTML 视为生成产物。** 对于基于 preset 的视频，可复用 skin 源文件位于 `.hyperframes/caption-skin.html`，workflow script 写入 `compositions/captions.html`；不要编辑生成的 `compositions/captions.html` 来修复 skin。通过 workflow 的 `captions.mjs` 重建，或在存在时使用该 workflow 的显式 overrides 机制。
