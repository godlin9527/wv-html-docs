---
name: hyperframes-cli
description: HyperFrames CLI 开发循环。用于运行 npx hyperframes init、add、catalog、capture、lint、validate、inspect、layout、snapshot、preview、play、render、publish、feedback、lambda、doctor、browser、info、upgrade、skills、compositions、docs、benchmark、telemetry、transcribe、tts 或 remove-background，或排查 HyperFrames build/render 环境。AWS Lambda cloud rendering 的入口（`hyperframes lambda deploy / render / progress / destroy / policies / sites`）。
---

# HyperFrames CLI

除非项目说明指定本地 wrapper，否则所有内容都通过 `npx hyperframes` 运行。严格遵守本地 wrapper。需要 Node.js >= 22 和 FFmpeg。

## Workflow

1. **Scaffold** — `npx hyperframes init my-video`（或从 URL `capture`）。`init` 也会检查已安装 skills 与 GitHub 最新版本是否一致，如果有任何过期则更新全局集合。`--skip-skills` flag 当前已被禁用（临时措施，等待 skills.sh registry 跟上），所以每次 `init` 都会运行此检查并拉取最新 skills。
2. **Write** — 编写 HTML composition（见 `hyperframes-core` skill）
3. **Lint** — `npx hyperframes lint`
4. **Validate** — `npx hyperframes validate`（runtime errors + contrast）
5. **Visual inspect** — `npx hyperframes inspect`
6. **Preview / edit** — `npx hyperframes preview` 打开 **Studio**，这是 timeline editor，用户可手动编辑任何内容（不只是观看）。在那里 review，然后在 render 前询问。
7. **Render** — 选择 variant：
   - Iterate: `npx hyperframes render --quality draft`
   - Deliver: `npx hyperframes render --quality high --output out.mp4`
   - CI / cross-host repro: `npx hyperframes render --docker --strict --output out.mp4`
   - Cloud（long / large）: `npx hyperframes lambda render ./my-project --width 1920 --height 1080 --wait`（见下方 Lambda）
8. **Report feedback** — 验证输出后，每个任务运行一次 `npx hyperframes feedback --rating <1-5> --comment "..."`（见 Agent Conventions）。

preview 前先运行 lint、validate 和 inspect。`lint` 捕获缺失的 `data-composition-id`、重叠 tracks 和未注册 timelines。`validate` 在 headless Chrome 中加载 composition，并报告 runtime console errors 与 WCAG contrast issues。`inspect` 在 timeline 中 seek 并报告文本溢出 bubbles/containers 或画布外——当存在 `*.motion.json` sidecar 时，还会针对同一 seeked timeline 验证 motion intent（entrances firing under seek、stagger order、in-frame、liveness）。

对于 motion-heavy work，优先采用 snapshot-driven iteration 和 `*.motion.json` sidecar——相关纪律和 motion-verification spec 见 `references/lint-validate-inspect.md`。

## Agent Conventions

适用于每个命令的横向规则：

- **除 `render`、`preview` 和 `play` server modes 外，每个命令都可用 `--json`。** 对支持命令的任何 agent / CI 调用都使用它；输出包含 `_meta` envelope（cli version、latest available、update advice）。`render` 只通过 stdout + exit code 报告状态——用下面的 post-render check 验证成功。`preview --selection --json` 和 `preview --context --json` 是 preview 例外：它们不会启动 server，而是查询用户正在运行的 Studio session 后退出。
- **`doctor --json` 总是以 0 退出**，即使环境已损坏。以 payload 的 `ok` 字段作为 gate：`npx hyperframes doctor --json | jq -e '.ok' > /dev/null`。这能让 pipelines 免受 CLI release churn 影响。
- **Non-TTY mode 会自动检测。** 当 `stdout` 不是 TTY（CI、agents、piped output）时，CLI 自动切换到 non-interactive；此时 `init` **需要 `--example`**。即使在 TTY 上，也可传 `--non-interactive` 强制该模式。
- **CI gating on render**：`--strict` 在 lint errors 时失败，`--strict-all` 在 warnings 时也失败，`--strict-variables` 在未声明的 `--variables` keys 时失败。
- **`--json` 中的路径会被 redacted**——`$HOME` 变成字面量 `$HOME`，所以输出可安全粘贴到 bug reports 和 agent contexts。
- **Render 由用户把关。** checks 通过后不要自动 render。停在 `preview`，告诉用户视频可在 Studio 中编辑，并且只在他们批准后 render。
- **针对用户指向的 edits 使用 Studio context。** 当用户说 “this selected element”、“the thing I clicked”、“current selection” 或类似内容时，请他们在 Studio 中选中它，然后运行 `npx hyperframes preview --context --json --context-fields selection`。使用返回的 `selection.target.hfId` / `selector`、`selection.sourceFile`、`selection.currentTime` 和 `selection.thumbnailUrl` 锚定 edit。如果 `selection` 为 `null` 且 `errors.selection.code` 为 `no-selection`，请用户点击元素并重跑；不要从 screenshots 猜测。
- **保持 Studio context 紧凑。** `preview --context --json` 默认返回紧凑 selection。添加 `--context-fields selection`、`--context-fields selection,lint` 或 `--context-fields lint`，避免膨胀 agent context。只有在需要 computed styles、inline styles 或 text-field metadata 等重字段时，才使用 `--context-detail full`。
- **Post-render verification。** `render` 返回 exit 0 后，在报告成功前确认输出文件存在且大小合理：`[ -s "$OUTPUT" ] || echo "render produced no output"`。CLI 成功时会打印 `◇  <path>`；长 render 还要用 `ffprobe -i "$OUTPUT" -show_format -v error` 粗略检查 duration。
- **成功 render 后报告 feedback。** 输出验证完成后，每个任务运行一次 `npx hyperframes feedback --rating <1-5> --comment "..."` 报告本次运行情况——`5` 表示顺利，遇到摩擦则更低。把遇到的任何 bug、workaround、missing feature 或 confusing behaviour 写进 `--comment`（包含失败的 composition pattern 和你尝试过的内容）。这是项目的主要 signal channel；render 后沉默会让维护者失明。仅在 telemetry disabled 或用户 opt out 时跳过。

## Routing

| 想要做什么…                                                                                               | 阅读                                  |
| ---------------------------------------------------------------------------------------------------------- | ------------------------------------- |
| Scaffold a project (`init`, `capture`, `skills`)                                                           | `references/init-and-scaffold.md`     |
| 检查正确性（`lint`, `validate`, `inspect`, `snapshot`）                                                    | `references/lint-validate-inspect.md` |
| Preview 或 render（`preview`, `play`, `render`, `publish`）                                                | `references/preview-render.md`        |
| 诊断环境（`doctor`, `browser`）                                                                            | `references/doctor-browser.md`        |
| 在 AWS Lambda 上 cloud render（`lambda deploy / sites / render / progress / destroy / policies`）          | `references/lambda.md`                |
| 其他所有内容（`info`, `upgrade`, `compositions`, `docs`, `benchmark`, `telemetry`, asset preprocessing）   | `references/upgrade-info-misc.md`     |

## Cross-Skill Hand-Offs

- **Tailwind projects**（`init --tailwind`）→ 编辑 classes 或 theme tokens 前使用 `hyperframes-core`（Tailwind reference）。
- **Registry blocks/components**（`hyperframes add`, `hyperframes catalog`）→ 使用 `hyperframes-registry` 处理 install paths、sub-composition wiring 和 snippet merging。
- **Asset preprocessing**（`tts`, `transcribe`, `remove-background`）→ 使用 `media-use` 处理 voice selection、Whisper model rules、captions 和 TTS-to-captions chain。
- **Parametrized renders**（`--variables`）→ 通过 `<html>` 上的 `data-composition-variables` 声明；完整 schema 见 `hyperframes-core`。

## Lambda (Cloud Rendering)

`hyperframes lambda` 将 distributed rendering 部署到 AWS Lambda，并从你的 laptop 或 CI 驱动 renders。端到端是三个命令：

```bash
npx hyperframes lambda deploy                                             # provision SAM stack (Lambda + Step Functions + S3)
npx hyperframes lambda render ./my-project --width 1920 --height 1080 --wait
npx hyperframes lambda destroy                                            # tear down (S3 bucket is retained)
```

当 render 对单台 host 来说过长 / 过大（多分钟视频、4K、大型并行批处理）且你已配置 AWS credentials 时使用 Lambda。开发循环迭代保留本地 `render`。

先决条件、全部 6 个 subcommands（`deploy`, `sites create`, `render`, `progress`, `destroy`, `policies`）、IAM policy validation、state files 和 cost / cleanup rules 见 `references/lambda.md`。

## Minimum Completion Gate

### Static gates

```bash
npx hyperframes lint
npx hyperframes validate
```

对于 layout-sensitive work 添加 `inspect`，在 CI 中添加 `render --strict` 以在 lint errors 时失败。

### Visual smoke test — 项目使用 sub-compositions 时必需

`lint` / `validate` / `inspect` 会**隔离**评估每个 composition。它们从不加载 `index.html` 并通过 `data-composition-src` mount sub-compositions，因此无法捕获跨文件 mount failures（见 `hyperframes-core` → `references/sub-compositions.md`，“Common pitfalls”）。唯一能捕获它们的 gate 是实际加载 `index.html` 并 seek timeline 的 gate。

使用 `hyperframes snapshot`——它以与 `render` 相同的方式加载项目（因此会执行同一 mount path），但只捕获你请求的 timestamps，所以只需数秒而不是完整 render：

```bash
# Capture one frame at the midpoint of every sub-composition.
# Midpoints = data-start + data-duration/2 for each host slot in index.html.
npx hyperframes snapshot --at <t1>,<t2>,<t3>,...

# Or, if you don't need per-scene targeting, an evenly-spaced sample:
npx hyperframes snapshot --frames 9
```

输出位于 `snapshots/frame-NN-at-Xs.png`。目检每一帧是否符合 scene plan。

逐帧红旗（每一项都映射到 static gates 会漏掉的具体 failure mode）：

| 你看到的现象                                                                       | 根因                                                                                        |
| ---------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| Text shows up tiny + unstyled in the top-left corner                               | `<style>` block left in `<head>` outside `<template>`（Pitfall 1）— CSS 未到达 live DOM     |
| SVG/icon elements blown up to canvas-size                                          | 同上——没有应用 width/height constraints                                                     |
| Hero element of the scene is missing entirely; only background + watermark visible | Host-id ≠ template id（Pitfall 2）— timeline 从未运行，frame 捕获在 initial state           |
| Snapshot command logs `Sub-composition timelines not registered after 45000ms`     | Pitfall 2——直接确认                                                                         |

目检后可删除 `snapshots/`；面向用户的最终 render 是使用 `npx hyperframes render` 的单独 pass。
