---
name: figma
description: 将 Figma 内容导入 HyperFrames composition — 已渲染资产、品牌 tokens、components、storyboard sections → 重建的 motion（frames 作为 states 读取，而不是 slides）（REST/CLI）、Figma Motion animations（MCP）以及 shaders（MCP source / native export）。当用户粘贴 figma.com 链接，或要求将 Figma design、frame、logo、brand 或 animation 带入 video/composition 时使用。
---

# Figma → HyperFrames

将用户的 Figma 工作带入 composition。**按能力拆分**（design spec §2）：

| Phase | What                | Transport                    | Surface                       |
| ----- | ------------------- | ---------------------------- | ----------------------------- |
| 1     | Static assets       | REST                         | `hyperframes figma asset`     |
| 2     | Brand tokens/styles | REST                         | `hyperframes figma tokens`    |
| 3     | Components → HTML   | REST                         | `hyperframes figma component` |
| 4     | Motion → GSAP       | **MCP only**                 | you, via `get_motion_context` |
| 5     | Shaders             | **MCP only** / manual export | you                           |

凡是可用之处都使用 REST（适合批量、headless）；只有在 Figma 没有暴露 REST 等价能力时才使用 MCP（motion、shaders）。每条路径都会在本地冻结资产，确保 renders 保持确定性。Storyboard reconstructions 会把 Phase-1 asset exports（REST）与 agent-driven timeline assembly 组合起来 — 不需要 MCP。已有的 frozen assets、manifest records 和 bindings 不受 routing changes 影响 — 该拆分只改变下一次 import 使用哪种 credential。

## Auth — 两种 credentials，限定 scope

**Preflight — 在第一次 CLI call 之前，检查 token 是否存在**：shell env（`[ -n "$FIGMA_TOKEN" ]`）**或**项目 `.env`（CLI 会自动加载它 — `.env` 条目算作已配置）。如果两者都没有，不要运行 command 去收集 error — 先带用户完成一次性设置，然后停止并等待：

1. figma.com/settings → **Security** → **Personal access tokens** → Generate new token.
2. Scopes — 该 integration 永远只需要 read-only（它永远不会写入 Figma）：**File content: Read-only** + **File metadata: Read-only**。可选 **Variables: Read-only** 用于 brand variables — 该 scope 只适用于 Figma Enterprise；没有它时，`tokens` 会自动降级为 published styles（这是预期行为，不是 error — 要说明）。
3. `export FIGMA_TOKEN="figd_…"` — 并建议将其持久化（shell profile 或项目 `.env`），这样未来 session 不会重复此步骤。

在 onboarding 时，也用一句话设定预期：每次 import 都会落为**带有记录 provenance 的本地 frozen file** — renders 永远不会调用 Figma，重新运行 command 只会重新 import Figma 中改变的内容，并且一个 token 可用于 assets、brand tokens 和 components，覆盖其 Figma account 有权查看的每个 file。

- **Phases 4–5 (motion/shaders):** Figma MCP connector（one-click OAuth），这是独立于 token 的另一种 credential。如果 MCP tools 返回 unauthenticated error，告诉用户连接 Figma connector，然后停止。
- 明确说明失败的 phase 需要哪一种 credential — 不要把这种拆分说成坏了。
- `BAD_TOKEN` (401) 在流程中出现 → token 已 expired/revoked；重新 mint。`FORBIDDEN` (403) → 缺少 read scope 或没有该 file 的 access — 检查 scopes + file visibility。`REQUIRES_ENTERPRISE`（variables 上的 403）→ 不是 failure：styles fallback 已经运行。

**Rate-limit awareness（spec §2.1）：** Starter plan 上 MCP 是 6 tool calls/**month**（figma plan matrix as of 2026-07 — 如果 quotas 看起来不对需重新验证）— 在 parent node 上用 `recursive:true` 批处理，除非用户要求否则跳过 verification screenshots，并缓存 raw MCP responses，这样重新 derivation 不会消耗第二次 call。REST 是 per-minute（10+/min，per-endpoint buckets）— 适合批量；遇到 429 时 back off。

## Routing

用 `parseFigmaRef` 解析用户的 figma link（URL、`fileKey:nodeId`、bare `fileKey`）。然后按 intent：

- "use this layer / logo / image" → **Asset** (CLI)
- "pull my brand / colors / tokens" → **Tokens** (CLI)
- "build a scene from this frame" → **Component** (CLI)
- "import this animation / motion" → **Motion** (MCP, below)
- storyboard section / scene frames 的 filmstrip → **Storyboard** (below)
- shader fill/effect → **Shaders** (below)

**向用户叙述每一步** — 每个 command 之前说明你将从 Figma 拉取什么；之后说明 artifact 落在何处（frozen path / sidecar / component dir）、composition 中改变了什么，以及下一步 immediate action（preview、add printed variables、re-import to link bindings）。用户不应该需要问“成功了吗？”或“现在怎么办？”。

## Assets (Phase 1 — CLI)

```bash
hyperframes figma asset '<url-or-fileKey:nodeId>' [--format svg|png|jpg|pdf] [--scale 2] [--description "..."] [--entity "..."]
```

通过 REST 渲染，sanitize SVG，冻结到 `.media/images/` 下，把 provenance 追加进 manifest，重新生成 `.media/index.md`（shared media-use inventory），并打印一个 `<img>` snippet。按 `fileKey:nodeId:format:scale:version` 幂等。vectors/logos 优先使用 SVG（scalable, animatable），raster fidelity 使用 PNG `--scale 2`。**始终传入 `--description "<what it is>"`**（它会成为 index row + `<img alt>`）；对 named brand marks 添加 `--entity "<name>"`，这样 media-use `resolve --entity` 后续能找到它们（entity hits 会跨 image/icon 匹配）。

## Tokens (Phase 2 — CLI)

```bash
hyperframes figma tokens <fileKey>
```

导入 variables 为 composition brand-variable entries + `figma-tokens.json` sidecar + binding-index records（`.media/figma-bindings.jsonl`）。Variables 在 upstream 受 Enterprise 限制：在其他 plans 上，command 会降级为 published-style metadata（values 在 component-import time 解析）。将打印出的 entries 添加到 composition 的 `data-composition-variables`。

当两者都需要时，**先导入 tokens 再导入 components** — 这样 component colors 才能链接到 brand variables，而不是烘焙出 duplicates。

## Components (Phase 3 — CLI)

```bash
hyperframes figma component '<url-or-fileKey:nodeId>'
```

Node tree → editable HTML，保持精确 figma geometry，打包为 `compositions/components/<name>/` 下的 registry item。Vectors/boolean-ops 会通过 Phase-1 export 自动 rasterize。Binding pass（spec §7.1，exact-ID only — 绝不 value matching）：

- Fill 绑定到一个**已导入的** token → `var(--slug, #literal)` — brand refresh 会传播。
- 绑定到一个**未知的** token → literal + `data-figma-unresolved` flag。command 会告诉你；给用户提供选择：在 source（或 library）file 上运行 `tokens`，然后重新 import component 来链接它们。每个 unknown library 只问**一次**它是哪个 file — 不要猜，不要按 hex 匹配。

## Motion (Phase 4 — MCP, the headline)

**Usage beacon:** MCP phases 没有 CLI touchpoint，所以在开始和结束时触发 skill beacon（anonymous、consent-gated、never fails）：开始时 `npx hyperframes events --skill=figma-motion`，完成时 `npx hyperframes events --skill=figma-motion --event=skill_completed --outcome=success|error`。shaders（`figma-shaders`）和 storyboards（`figma-storyboard`）同理。

不存在 REST equivalent。你驱动 MCP tools，然后把 output 交给 `@hyperframes/core/figma` 中的 pure helpers：

1. `get_motion_context(fileKey, nodeId)` — 在 parent frame 上使用 `recursive:true`（整个 scene 一次 call，而不是每个 element 一次）。将 raw JSON 保存到 project 旁边（`.media/figma-cache/`），这样 retranslation 免费。
2. Normalize 为 `MotionDoc`：每个 animated property 一个 `MotionTrack` { property (motion.dev name), values, times (0..1), ease[] (named or `[x1,y1,x2,y2]` bezier), duration, repeat }。Selector = element 的 stable id（Phase-3 output 或 authored scene 中的 `#<id>`）。
3. `motionToGsap(doc)` → `emitTimelineScript(spec)` → 作为 `<script>` 注入到 GSAP + CustomEase CDN tags 之后。Paused、finite，并以 literal key 注册在 `window.__timelines` 上。
4. Untranslatable track（shader-driven、unsupported prop、complex masks）→ bake：`export_video` → freeze MP4 → 作为 `<video class="clip">` embed。例外：shader-driven tracks — figma 的 export path 会把 shaders flatten 到 base color（见下面 Shaders），所以在那里 bake 会静默丢失 shader；改为要求用户提供 native figma export。始终说明你使用了哪条 path 以及原因。mapped set 之外的 named eases 会 fallback 到 linear — mapping table 位于 `motionEase.ts`；当 fallback 发生时要向用户标明。
5. 在称为完成之前运行 `npx hyperframes lint && npx hyperframes validate`。

## Shaders (Phase 5 — mostly manual)

Figma 的 MCP render path 不会执行 shaders（它们会 flatten 到 base color），并且 shader source 只对**library-published** styles 可访问（paid Full seat）。默认路径：要求用户在 Figma 中 natively export shader frame（PNG 或 Motion MP4），然后将其作为 Phase-1 asset / clip 导入。不要尝试用 MCP pixel capture 捕获 shader — 它会静默产生错误结果。

## Storyboards (a SECTION of scene frames → animation)

**核心规则：storyboard frames 是 KEYFRAMES，不是 slides。** 两个包含相同 element 的 frames 描述的是该 element 随时间变化的 state — 在 states 之间 animate 这个 ELEMENT；绝不要把 frames 当作 stills 序列播放。在四个连续 frames 中以递减 y 绘制的 logo，是一个 element 经过四个 keyframes 上升。逐帧播放 storyboard frames 是 failure mode；重建它们暗示的 element timelines 才是工作。

Storyboard files 遵循可机械解析的 grammar — 不要目测，要 decode：

1. **Scene units**：SECTION 内，每个 frame-sized node 都是一个 scene — 包括命名 FRAMEs 和 loose full-frame RECTANGLEs（designers 会把 stills 直接粘进 section）。按 size 过滤（约等于 composition aspect，例如 >1400×900），不要按 node type 或 name。
2. **Order = x-position**（如果 strip 换行则 row-major）。按 `absoluteBoundingBox.x` 排序 scenes。
3. **将相邻 frames diff 为 element chains** — animation 就在这里。跨连续 frames 匹配 children：先按**name**（same name = same element → 在 states 之间 tween 其 relative x/y/w/h），再按**geometry similarity**（similar size + nearby center = 像素变化的同一 logical element → 在 tweening geometry 的同时原地 crossfade 两次 exports；覆盖 typed-text progressions 和 morph states）。Unmatched children 在其 scene 的 beat enter/exit。Frame background fills 作为 color track tween。每个 chain 导出一个 asset（只有在 pixels 确实不同时才每个 state 一个）— 绝不每个 frame 一个 still。
4. **Stills 是 fallback，不是 default** — 只用于无法 decompose 的 frames（没有 shared elements 的 flat full-frame screenshots）；这些按下面的 animatic treatment 处理。
5. **Director notes**：strip 下方的 TEXT nodes 是 motion intent，与其 x-range 重叠的 scene 配对。它们描述_如何_ animate — 不是 on-screen copy。
6. **Batch exports**（elements 或 stills）：`GET /v1/images` 接受 comma-separated ids，但大型 scene frames 在超过约 12 ids 后会 hit "Render timeout" — chunk 到每次约 4 个，并带 retry。（每个 scene 一次 call 会浪费 rate budget；26 scenes 通过 single-asset path 约等于 52 calls。）
7. **Note verbs → transitions**（starter vocabulary，随着遇到的情况扩展）：

| Note says                       | Do                                                                |
| ------------------------------- | ----------------------------------------------------------------- |
| EXPLOSION / BURST               | incoming scale ~1.5→1 + fade, `power3.out`                        |
| SLIDES / SLIDE TO THE… / SCROLL | directional slide in from that edge                               |
| MORPH / REVEALS                 | crossfade — or Phase-3 import if the motion is inside one scene   |
| CYCLE THROUGH / EACH ONE        | longer hold — or Phase-3 import if items animate within the scene |
| (no note)                       | crossfade + slow Ken-Burns drift                                  |

8. **Stills vs. components routing**：描述 scenes _之间_ motion 的 note → 对 still 应用 transition（如上）。描述 scene _内部_ motion 的 note（"TEXT LINES REVEAL ONE AFTER THE OTHER", "PILLS ANIMATE IN"）→ 该 frame 应做 Phase-3 component import（真实 elements），并按 note animate，而不是 flat PNG。先用 stills 做 animatic pass，然后升级 notes 点名的 scenes。
9. 一个 `main` timeline 按 absolute times 排列所有内容（每个 scene 的 opacity/x/y）— animatic 不需要 per-scene sub-compositions。
10. **Escalation — frames 描绘一个 product UI → rebuild the app，而不是 element chains。** 当每个 frame 都是同一个 application screen 的 successive states（signup flow、settings panel、player）时，element chains 会低估它。把 UI 重建为 live DOM — 对变化 state 的 parts 做 Phase-3 component import，对 static chrome 使用真实 exported pixels（**code what changes state, freeze what doesn't**）— 并把每个 frame delta 当作要执行的**interaction**，而不是要应用的 tween：cursor enters、clicks the control、state responds、screens push/slide as real navigation。结果应读起来像一个 working app 的 continuous screen recording。这是核心规则在 UI flows 中的结论；stills/element-chain treatments 适用于不是一个 coherent application 的 storyboards。

## Determinism

绝不要在 composition 中留下 Figma URL — 先 freeze。绝不要 emit `repeat: -1`。Timelines paused、finite、literal `window.__timelines` keys。所有 Figma I/O 都发生在 import time；render 只看到 local files。
