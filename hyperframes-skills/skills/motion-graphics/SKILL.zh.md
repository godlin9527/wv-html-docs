---
name: motion-graphics
description: >
  一个短小、设计驱动的动态图形，motion 本身就是信息 —— kinetic
  typography、stat count-up、chart/data-viz hit、logo sting / brand lockup、
  lower-third / callout / social overlay、animated map（highlight regions、
  connect places、zoom to a location）、animated tweet / news-article /
  headline、webpage / UI animation（scroll、cursor、callouts），或把
  real image 的几何结构融合进 chart。通常低于 10s（最高约 30s），没有
  narration 或 live-action subject；渲染为 MP4 或 transparent overlay。
  Longer / narrated / multi-scene → /general-video。Unclear → /hyperframes。
metadata:
  {
    "tags": "orchestrator, motion-graphics, kinetic-type, data-viz, logo-reveal, lower-thirds, news, tweet, webpage, asset-fusion, short-form, overlay, no-narration",
  }
---

# motion-graphics — dispatch entry

> **Step 0 前确认路线。** 这个 skill 制作**短小、设计驱动、无旁白的动态图形**（motion 就是信息；约低于 10s，无 voice-over）。**更长、多场景或有旁白**的处理 → `/general-video`；**带旁白的网站视频** → `/website-to-video`；**主题 explainer** → `/faceless-explainer`；**产品 promo** → `/product-launch-video`；**给现有 footage 加 captions** → `/embedded-captions`。**超出范围**：live / at-render-time data，或它无法捕获的 footage。不确定是 motion-first 还是 narrated？**先读 `/hyperframes`。**

一个短小的设计驱动动态图形。**Asset-first**：先决定资产策略并 source real material，再设计 shot，然后围绕已有资产设计 shot，并通过复用 catalog capabilities 进行 compose。所有 artifacts 放到 `PROJECT_DIR = videos/<project-name>/`（Step 0 创建）；下面所有路径都相对于它。

| Phase    | Execution                                                             | Primary artifact                                                 | Detailed flow                 |
| -------- | --------------------------------------------------------------------- | ---------------------------------------------------------------- | ----------------------------- |
| init     | Bash                                                                  | `hyperframes.json`                                               | Step 0                        |
| plan     | subagent —— **decide search?** + classify + asset strategy             | `shot-plan.json`（draft: category, `asset_needs` queries, brief） | `agents/director.md`（Part 1） |
| source ◇ | Bash —— media-use resolve（**如果 `asset_needs` 为空则跳过**）         | `assets/` + `assets/index.md`                                    | `phases/source/guide.md`      |
| design   | subagent —— 围绕 resolved assets 设计 shot                             | `shot-plan.json`（final: block(s) + layout + motion + positions） | `agents/director.md`（Part 2） |
| build    | subagent —— reuse-first composition                                    | `compositions/index.html`                                        | `agents/builder.md`           |
| render   | Bash —— `hyperframes render`（MP4，或 overlay 用 `--format webm/mov`） | `renders/video.mp4`                                              | Step 5                        |
| verify   | Bash —— `lint` / `inspect` -> 失败时 repair subagent                   | （原地修复）                                                     | `agents/finalize.md`          |

`◇ source` 只在所选 category 声明 assets 时运行。纯代码/文本 categories（例如 `kinetic-type`、大多数 `charts`/`stat`）有 `asset_needs: []`，会从 plan 直接跳到 design。

## Categories —— 按 search decision 拆分

`plan` 的**第一个决策是：这需要 search 吗？** 这个 fork 将 categories 分成两组；然后选择具体 category —— 对于 search-driven，**由 search 返回的内容类型决定**。每个 category 都是一个 `categories/<id>/module.md`（它的 planning + build rules）；共享 motion vocabulary 位于 `references/motion-vocabulary.md`（→ `hyperframes-animation` rules/blueprints + registry blocks）。

**Form categories —— 不 search；用户提供内容：**

| Category       | Intent                                                                                                         | Leans on                                                                    |
| -------------- | -------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| `kinetic-type` | punchy line / quote / title，motion-first text                                                                 | `caption-*` blocks + animation rules                                        |
| `stat`         | single hero number / count-up + ring                                                                           | `apple-money-count` / `rules/{counting-dynamic-scale, stat-bars-and-fills}` |
| `charts`       | bar / line / pie / race / % from data                                                                          | `data-chart` block                                                          |
| `logo-reveal`  | logo sting / brand lockup（user logo）                                                                         | `logo-outro` / `rules/svg-path-draw`                                        |
| `lower-thirds` | name / title bars、callouts、social overlays                                                                   | `caption-*` + registry overlay blocks                                       |
| `maps`         | geographic motion —— highlight regions、connect places、zoom to a location（vector lane, or baked basemap lane） | `us-map` / `world-map` family + `bake-basemap.mjs`                          |

**Search-driven categories —— 先 search，再按内容类型 animate**（RWA path）：

| Returned content | Category       | Animation                                                      |
| ---------------- | -------------- | -------------------------------------------------------------- |
| webpage / link   | `webpage`      | webpage / UI animation（scroll、reveal、cursor、callouts）     |
| news article     | `news`         | headline reveal + source card + key-fact callouts              |
| tweet            | `tweet`        | animated tweet card                                            |
| image / entity   | `asset-fusion` | asset 的几何结构 _becomes_ chart（RWA diegetic fusion）        |

Build order：一次一个，coverage-first（粗糙也可以）。`kinetic-type` 从 prototype ported；其余跟随。

## Prerequisites

macOS Apple Silicon 或 Linux x64。System tools：`brew install node ffmpeg`。运行一次 `npx hyperframes doctor`。macOS GPU render：`export PRODUCER_BROWSER_GPU_MODE=hardware`。

Optional keys（未设置则使用 local fallbacks）—— 仅 source/generate assets via media-use 的 categories 需要：

| Key                                 | Used for                                                    | Fallback                        |
| ----------------------------------- | ----------------------------------------------------------- | ------------------------------- |
| `GEMINI_API_KEY` / `GOOGLE_API_KEY` | image generation（media-use resolve）                       | skip generate / search-only     |
| (asset_scout / search providers)    | `webpage`/`news`/`tweet` + `asset-fusion` real-asset search | category degrades to asset-free |

## Flow

### Step 0 —— Initialize

cwd 是 agent workspace root；所有 artifacts 写到 `PROJECT_DIR = videos/<project-name>/`。`<project-name>`：使用用户给定的 dir，否则从 intent 生成一个短 kebab-case name（`<subject>-motion`）。不要使用 workspace basename 或 timestamp。

仅当 `$PROJECT_DIR/hyperframes.json` 不存在时：

```bash
PROJECT_DIR="${MOTION_GRAPHICS_DIR:-videos/<project-name>}"
mkdir -p "$(dirname "$PROJECT_DIR")"
npx hyperframes init "$PROJECT_DIR" --non-interactive --example=blank
```

`init` 会检查已安装 skills 是否落后于 GitHub 最新版本，并在过期时更新 global set。

**Constraints:** 永远不要在 workspace root 中运行 `hyperframes init`；永远不要在 `PROJECT_DIR` 内嵌套另一个 `hyperframes/`；每个 Bash 命令（master + subagents）都是 `(cd "$PROJECT_DIR" && ...)` subshell —— 永远不要裸 `cd`。

### Step 1 —— Plan（subagent: Director Part 1）

Dispatch 一个 subagent。prompt = 完整 `agents/director.md` + `## Dispatch context`（`SKILL_DIR` / `PROJECT_DIR` / 用户请求 / `Schema: <SKILL_DIR>/references/shot-plan-ir.md`）。它必须：

1. **Decide: does this need a search?**（第一个 fork）
   - **No** → 选择一个 **form category**（kinetic-type / stat / charts / logo-reveal / lower-thirds）；内容由用户提供；`asset_needs: []`。
   - **Yes** → 将 **search plan** 写入 `asset_needs[]`（news / web / tweet / image；two-pole queries）。具体的 **search-driven category**（webpage / news / tweet / asset-fusion）由 Step 2 返回的内容类型确认，并在 Step 3 finalized。
2. 写入 draft `shot-plan.json`（envelope + chosen form category _or_ search intent + `asset_needs` + 一段 shot brief）。Schema：`references/shot-plan-ir.md`。

Validation：`[ -s "$PROJECT_DIR/shot-plan.json" ] && echo ok || echo missing`。

### Step 2 —— Source ◇（Bash: media-use, conditional）

如果 `shot-plan.json.asset_needs` 非空，resolve assets（search / generate / fetch → frozen project-local paths + ledger）。见 `phases/source/guide.md`（包装 `media-use resolve`；search-driven categories 使用 news/web/tweet/image search）。如果 `asset_needs` 为空，**跳到 Step 3**。

```bash
# illustrative — see phases/source/guide.md
(cd "$PROJECT_DIR" && node <SKILL_DIR>/phases/source/resolve.mjs --plan ./shot-plan.json --out ./assets)
```

优雅降级：如果 search/provider 不可用，category fallback 到 asset-free（在 `context.log` 中注明）。

### Step 3 —— Design（subagent: Director Part 2）

Dispatch 一个 subagent（prompt = `agents/director.md` Part 2 + dispatch context，包括 Step 2 运行后的 resolved `assets/index.md` + `catalog-map.md`）。它围绕**可用资产**设计 shot：选择 catalog block(s) + `hyperframes-animation` rules/blueprints、layout、motion、beats，以及（对 `asset-fusion`）`element_positions` + eyedropper palette。Finalize `shot-plan.json`（`content.block` + `content.customize` + per-category content）。

### Step 4 —— Build（subagent: Builder, reuse-first）

Dispatch 一个 subagent。prompt = 完整 `agents/builder.md` + dispatch context（`shot-plan.json`、`catalog-map.md`、category 的 `module.md`、`references/motion-vocabulary.md`、`references/builder-contract.md`）。**Reuse-first**：`npx hyperframes add <block>` + 原地 customize；只手写 gaps + asset-fusion affordance。输出遵守 HF contract 的 `compositions/index.html`（paused GSAP timeline on `window.__timelines`、`class="clip"` + stable ids、`tl.seek(0)`、deterministic）。

### Step 5 —— Render（Bash）

```bash
(cd "$PROJECT_DIR" && npx hyperframes render . --skill=motion-graphics -q draft -o ./renders/video.mp4)
# transparent overlay variant: --format webm  (or mov)
```

### Step 6 —— Verify（Bash → repair subagent on failure）

```bash
(cd "$PROJECT_DIR" && npx hyperframes lint . && npx hyperframes inspect .)
```

exit 0 → done。遇到 lint/inspect errors 时，dispatch repair subagent（`agents/finalize.md`：snapshot QA + 一次原地修复 pass + re-render）。Repair 时永远不要改变 fixed duration。

### Report + optional preview

报告最终输出（`renders/video.mp4`，或 `.webm` / `.mov` overlay variant）+ duration。**运行期间不要打开 preview。** 只在请求时提供 preview，并且在 render 后启动，这样它服务最终文件：

```bash
(cd "$PROJECT_DIR" && npx hyperframes preview)   # Studio UI; or `npx hyperframes play` for a shareable link
```

Flags 位于 `hyperframes-cli` skill（`references/preview-render.md`）。

## Resume table

| State                                                    | Continue from            |
| -------------------------------------------------------- | ------------------------ |
| no `shot-plan.json`                                      | Step 1（plan）           |
| `shot-plan.json` has `asset_needs`, no `assets/`         | Step 2（source）         |
| `shot-plan.json` final, no `compositions/index.html`     | Step 3/4（design+build） |
| `compositions/index.html` exists, no `renders/video.mp4` | Step 5（render）+ Step 6 |
| `renders/video.mp4` exists                               | Report + stop            |

## Design notes（maintainers —— execution does not read this）

- **Asset-first rationale:** sourcing 被前置，并影响 shot design（RWA flow：analyze → search → review → compose）。search-driven categories（`webpage`/`news`/`tweet`）和 `asset-fusion` 都依赖 media-use search（news/web/tweet/image），这是 media-use 记录的 RWA lineage。
- **Reuse-first:** LLM-generated templates 在生态内的类比是“compose catalog blocks + `hyperframes-animation` rules”。HF 的 paused GSAP timeline ≙ Remotion 的 `useCurrentFrame`。
- **Category module contract:** 一个 `categories/<id>/module.md`（planning + build），共享 `references/motion-vocabulary.md`（+ optional eval）。添加 category = 放入 folder + 在 `agents/director.md` 注册 classifier line + 在 `catalog-map.md` 添加 row；phase pipeline 不变。
- **Directory shape:**
  ```
  videos/<project-name>/
    hyperframes.json  context.log
    shot-plan.json            # the IR (Director output)
    assets/  assets/index.md  # media-use output (if sourced)
    compositions/index.html   # Builder output
    renders/video.mp4
  ```
- **Registration:** 在 `hyperframes` router 中 —— 添加 “design-led short motion graphic” intent + Workflow description；从 `/general-video` 中划出 motion-graphics triggers；添加反向 Do-NOT-use edges。见 `motion-graphics-genre.md` §5-7。
