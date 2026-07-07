---
name: media-use
description: Agent Media OS，是 HyperFrames 项目中所有媒体需求的唯一 skill。将 BGM、SFX、image、icon 或 voice 解析成冻结的本地文件 + ledger 记录（一个动词，`resolve`）；当 catalog 未命中时通过 TTS / music / image models 生成；通过一个共享音频引擎生成 voiceover、transcription、captions 和 background removal；对媒体执行操作（cut / reframe / transform）；并在项目之间复用资产。把 search noise 留在磁盘上，把路径交给 agent。用于任何 audio、image、icon、voiceover、caption 或 media-asset 需求。
---

# media-use

HyperFrames 的 media OS：resolve · generate · operate · remember，所有媒体类型，一个 skill，零 context noise。

## What it owns（HyperFrames 留下的空白）

HyperFrames 负责 media _playback_；media-use 负责其他一切。每一行都由 `scripts/lib/coverage.test.mjs` 强制覆盖，确保声明不会腐化。

| HyperFrames gap                            | media-use owns it via                                                                                                                       |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------- |
| Audio-only, no image/icon                  | `resolve --type image\|icon`（heygen asset search）                                                                                          |
| No voice / audio generation                | `resolve --type voice` + audio engine（`audio/scripts/audio.mjs`）                                                                           |
| Scattered/duplicated audio engine          | `audio/` 下的一个 consolidated engine（hyperframes-media retired）                                                                           |
| No agent media-ops（cut/reframe/transform） | `references/operations.md` + `resolve --from` 注册输出                                                                                       |
| No transcript-driven cutting               | `scripts/transcript-cut.mjs` 将 word-timestamp edits 编译成 cut lists                                                                        |
| No auto-duck / publish loudness            | `scripts/audio-duck.mjs` + `references/operations.md` loudnorm/sidechain recipes                                                            |
| No cross-project memory                    | global content-addressed cache + auto-promote（`~/.media`）                                                                                  |
| No image generation                        | RAM-graded local mflux（FLUX）通过 `scripts/lib/mflux-provider.mjs`，codex `image_gen` upsell（`scripts/lib/codex-provider.mjs`）             |
| No video generation                        | spec-gated local LTX（`videogen` in `scripts/lib/local-models.mjs`）；`heygen video create` avatar upsell                                    |
| Weak local-model defaults                  | free-usage HeyGen first（TTS, bg-removal）通过 `heygen` CLI；local open-source 仅作为 opt-out fallback（`scripts/lib/local-run.mjs`）          |

## When to use

当 composition 需要媒体时调用 `resolve`：背景音乐、音效、图片、图标或 voice。对于 voiceover / TTS、music、SFX 和 caption timing，使用**音频引擎**（如下）；background removal 委托给 `hyperframes` CLI；transcription 默认通过 `scripts/transcribe.mjs` 使用 Parakeet（优于 whisper.cpp：6.05% vs 7.44% WER，快 5-10 倍），并带 whisper.cpp auto-fallback（见 `references/operations.md`）。对于 cut / reframe / transform 现有媒体，见 `references/operations.md`。media-use 先搜索 HeyGen catalog，将最佳匹配冻结到本地，在 manifest 中注册，并交给 agent 一行结果；所有 search noise 都留在磁盘上。

## Resolve

```bash
node <SKILL_DIR>/scripts/resolve.mjs --type <type> --intent "<description>" --project <dir>
```

返回一行：`resolved <id> → <path> (<type>, <metadata>)`

### Types

| Type    | 查找内容            | Provider                                 |
| ------- | ------------------- | ---------------------------------------- |
| `bgm`   | Background music    | HeyGen audio catalog（10k+ tracks）      |
| `sfx`   | Sound effects       | Bundled 19-file library + HeyGen catalog |
| `image` | Photos, backgrounds | HeyGen asset search（75k+ vectors）      |
| `icon`  | Icons, logos        | HeyGen asset search（type=icon）         |
| `voice` | TTS voiceover       | Local Kokoro（free）；HeyGen TTS upsell  |

### Examples

```bash
# Background music
node <SKILL_DIR>/scripts/resolve.mjs --type bgm --intent "upbeat tech launch" --project .
# → resolved bgm_001 → .media/audio/bgm/bgm_001.mp3 (bgm, 25s)

# Sound effect
node <SKILL_DIR>/scripts/resolve.mjs --type sfx --intent "whoosh" --project .
# → resolved sfx_001 → .media/audio/sfx/sfx_001.mp3 (sfx, 0.57s)

# Image
node <SKILL_DIR>/scripts/resolve.mjs --type image --intent "gradient tech background" --project .
# → resolved image_001 → .media/images/image_001.jpg (image)

# Icon
node <SKILL_DIR>/scripts/resolve.mjs --type icon --intent "rocket" --project .
# → resolved icon_001 → .media/images/icon_001.png (icon, transparent)
```

### Flags

| Flag            | Description                                                     |
| --------------- | --------------------------------------------------------------- |
| `--type, -t`    | Media type: bgm, sfx, image, icon, voice                        |
| `--intent, -i`  | 你需要什么（自然语言）                                          |
| `--entity, -e`  | 用于 cache matching 的 entity name（可选）                      |
| `--project, -p` | Project directory（默认：.）                                    |
| `--from`        | 冻结本地文件或直接 public URL（ingest）                         |
| `--local-only`  | Offline：跳过所有 network provider（仅 cache + local）          |
| `--provider`    | 强制使用一个 generator（例如 `codex`, `mflux`, `kokoro`, `heygen`） |
| `--adopt`       | 批量导入现有 assets/ 到 manifest                                |
| `--json`        | 输出 JSON，而不是单行结果                                       |

## Providers

media-use 不持有 key；每个外部工具拥有自己的 auth。Generation 是
local-first，并在有帮助时提供 cloud upsell。`resolve` 会检查 AVAILABLE RAM，
并自动选择最适合的本地模型（RAM-graded ladder，`describeModelLadder`）；
agent 可以看到 ladder 并 override。

| Type          | Provider（按顺序）                                                                      |
| ------------- | --------------------------------------------------------------------------------------- |
| bgm/sfx       | heygen catalog（free）                                                                  |
| image         | heygen search，然后 local mflux（适合你 RAM 的最佳 FLUX），然后 codex `image_gen` upsell |
| voice         | local **Kokoro**（free, on-device），然后 **heygen tts** paid upsell                     |
| icon          | heygen asset search                                                                     |
| video（local） | local LTX（`videogen` ladder）；`heygen video create` avatar upsell                      |

Local Kokoro（voice）、mflux（image）和 LTX（video）在设备上运行（free、
private，缓存后可 offline）。Paid/cloud upsells 位于它们之后：HeyGen TTS
用于 voice，`codex` CLI（ChatGPT sub）用于更好的 image，`heygen` CLI
用于 avatar video。Cost rule（X4）：agent 发起付费调用前需要确认；
用户明确请求的调用直接运行。

要强制特定 generator（例如用户说“make this image with codex”），
传 `--provider codex`：它会把 resolution 固定到该 provider，并跳过
free-first default。RAM ladders 和 upsell recipes 见 `references/operations.md`。

`--local-only` 会跳过所有 network provider，包括 free HeyGen provider，
只留下 project + global cache 以及任何 local provider。

## How it works

1. 检查 project `.media/manifest.jsonl` 是否有 exact-prompt match
2. 扫描现有 `assets/` directory，寻找匹配需求的未注册文件
3. 检查 global cache `~/.media/` 是否有可复用资产
4. 通过 provider 搜索（HeyGen audio catalog、HeyGen asset search）
5. 将文件冻结到 `.media/<type>/`，注册进 manifest，重新生成 `index.md`

agent 只拿到**一行**。Candidates、scores、provenance 都留在磁盘上。

## Adopt existing projects

多数 HyperFrames 项目已经在 `assets/` 中有资产。media-use 会 adopt 它们：

```bash
node <SKILL_DIR>/scripts/resolve.mjs --adopt --project .
# → adopted 9 assets from assets/
#   bgm_001 → assets/bgm/mango-fizz.mp3 (bgm, 146.6s)
#   image_001 → assets/images/avatar.jpg (image, 400×400)
```

`ffprobe` 提取真实 duration 和 dimensions。resolve 时，`assets/` 中匹配 intent 的未注册文件会被即时 adopt。

## Reading the inventory

resolve 或 adopt 后，读取 `.media/index.md` 查看完整 inventory：

```
# .media · 4 assets

id         type   dur   dims       path                          description
bgm_001    bgm    25s   -          .media/audio/bgm/bgm_001.mp3  upbeat tech launch
sfx_001    sfx    0.6s  -          .media/audio/sfx/sfx_001.mp3  whoosh
image_001  image  -     1920×1080  .media/images/image_001.jpg   gradient tech background
icon_001   icon   -     200×200    .media/images/icon_001.png    rocket
```

## Cross-project reuse

resolve 时资产会自动缓存。每个 resolved/ingested asset 都会 auto-promote 到 `~/.media/` 的 global cache，因此后续在任何项目中对同一 prompt 的 resolve 都会命中 cache，无需重新下载，也无需 provider call。

## Files

- `.media/manifest.jsonl`: machine SSOT，每行一个 JSON record
- `.media/index.md`: agent-readable table（id, type, dur, dims, path, description）
- `~/.media/`: global cross-project reuse cache（content-addressed, SHA-256）

## Audio engine: voiceover, music, SFX, captions, transcription

完整 audio pass（TTS voiceover + background music + sound effects 一次完成）
使用位于 `audio/scripts/audio.mjs` 的共享引擎。它接收中性的
`audio_request.json`，写入 `audio_meta.json`，并把资产放到
`.media/audio/{voice,bgm,sfx}` 下：

```bash
node <SKILL_DIR>/audio/scripts/audio.mjs --request ./audio_request.json --out ./audio_meta.json
```

- **Request** `{ provider?, lang?, speed?, lines: [{ id, text, sfx?: [names] }], bgm: { mode?, query?, prompt? } }`：`id` 会把每行关联回你的模型；`bgm.mode` = `retrieve | generate | none`（省略则 auto）。`--only tts,bgm,sfx` 运行子集，并合并到已有 `--out`。
- **Output** `audio_meta.json`（id-keyed）：`voices[].{path,duration_s,words[]}`（caption 用 word timestamps）、`sfx[]`、`bgm`、`total_duration_s`。
- **一个开关自动降级**：存在 HeyGen credential → HeyGen TTS + music/SFX retrieval；不存在 → ElevenLabs/Kokoro TTS、Lyria/MusicGen BGM generation，以及捆绑 SFX library（无需 credential）。
- 如果 BGM 走了 generate 路径（`bgm_pending: true`），在 final render 前运行 `audio/scripts/wait-bgm.mjs`。

单次 helper：`audio/scripts/heygen-tts.mjs`（一个 voice file）。Transcription / background removal / captions 使用 `hyperframes` CLI（`transcribe`, `remove-background`），见 `audio/references/` 中的分主题指南（`tts.md`, `bgm.md`, `sfx.md`, `transcribe.md`, `remove-background.md`, `captions/`）。

## Operating on media（cut, reframe, transform）

media-use 负责 resolve + remember；对资产进行**操作**请见
`references/operations.md`：本地工具 recipe（ffmpeg trim/reframe/montage、
auto-editor、scenedetect）以及 local-vs-HeyGen transform table（background
removal、upscale、lipsync、translate）。运行工具后，用
`resolve --from <output> --type <type>` 注册输出，让它加入 ledger + global
cache。

## CLI tools used（运行什么，以及如何启用）

`resolve` 会自动级联；每个 provider shell 一个 CLI。Local tools 是 OPT-IN：
如果缺少某个 local tool，resolve 会优雅降级到 free/cloud path，
所以除了 `ffmpeg`/`ffprobe` 之外，这里没有严格必需项。安装 local
tool 可解锁 free、private、on-device path。media-use 不持有 key。

| Tool               | Serves                                                                   | Install                                                                                                             |
| ------------------ | ------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------- |
| `ffmpeg`/`ffprobe` | adopt probing、cut、duck bake、loudnorm                                  | system package（`brew install ffmpeg`）                                                                             |
| `heygen`           | catalog（bgm/sfx/image/icon）、TTS + avatar upsell                       | `curl -fsSL https://static.heygen.ai/cli/install.sh \| bash` then `heygen auth login --key <key>`（needs >= v0.1.6） |
| `mflux-generate`   | local image gen（FLUX）、best-for-RAM                                    | `uv venv ~/.venvs/mflux && VIRTUAL_ENV=~/.venvs/mflux uv pip install mflux==0.9.6`                                  |
| `codex`            | image gen upsell（ChatGPT sub）                                          | Codex CLI, logged in via ChatGPT（owns its own auth）                                                               |
| `parakeet-mlx`     | local transcription（default ASR, best）                                 | `uv venv ~/.venvs/parakeet && VIRTUAL_ENV=~/.venvs/parakeet uv pip install parakeet-mlx`                            |
| `ltx-2-mlx`        | local video gen                                                          | `git clone https://github.com/dgrauet/ltx-2-mlx && cd ltx-2-mlx && uv sync --all-extras`                            |
| `npx hyperframes`  | Kokoro TTS（voice）、whisper.cpp（transcribe fallback）、remove-background | bundled with the hyperframes CLI                                                                                    |

RAM-graded local-model shortlist + 每个 tier 的精确 install/invoke 位于
`scripts/lib/local-models.mjs`（agent 可以读取 `describeModelLadder(cap, specs)`
来查看哪种模型适配这台机器）。如果 PATH 上没有某个工具，它的 provider
会向 stderr 打印一行 diagnostic，然后 resolve fall through 到下一个
provider（例如 no `mflux` -> codex image upsell；no `parakeet-mlx` -> whisper.cpp）。

`heygen asset search` 是隐藏在 `heygen --help` 之外的 pre-launch command，但它
可以运行；provider 会使用 allowlisted `X-HeyGen-Client-Source` header 标记请求
（v0.1.6+）。

## Telemetry

`resolve` 和 edit tools（transcribe / transcript-cut / audio-duck）会向
PostHog（`scripts/lib/telemetry.mjs`）发送匿名 usage event，这样我们能看到
哪些能力实际被使用。它只记录 media TYPE、resolution SOURCE 和 winning
PROVIDER：绝不记录 intent text、file names 或 paths，并且 `$ip:null`，所以不会存储 IP。
Best-effort 且 non-blocking（resolve 绝不会等待 telemetry，也不会因 telemetry 失败）。

用 `DO_NOT_TRACK=1` 或 `HYPERFRAMES_NO_TELEMETRY=1` opt out（CI 和 dev 中也关闭）。
使用与 `hyperframes` CLI 相同的 public PostHog project key 和 opt-outs。
