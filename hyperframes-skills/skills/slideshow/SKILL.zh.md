---
name: slideshow
description: >
  创作 HyperFrames slideshow —— presentation、pitch deck 或 interactive
  deck，支持 discrete slides、fragment reveals、branching、hotspot navigation，
  以及带 speaker notes 的内置 presenter mode；也可将现有页面转换为 deck。
  输出是可导航 deck，不是渲染出的 MP4。如果用户没有明确要求 slideshow，
  authoring 前先确认。不明确 → /hyperframes。
---

# Slideshow authoring contract

HyperFrames slideshow 是普通 HyperFrames composition —— scenes、clips、GSAP timelines —— 加上一个额外成分：声明哪些 scenes 是 slides 以及它们如何连接的 **JSON island**。player 的 `SlideshowController` 读取这个 island，并把连续的 GSAP timeline 转换为离散、可导航的 deck。

**先阅读 `/hyperframes-core`** 以了解基础 composition contract（clips、tracks、`data-*` attributes、determinism rules）。这个 skill 只覆盖新增部分：island schema、slide writing rules、fragments、branching、validation 和 wrapping component。

## Output — a navigable deck, not a linear MP4

slideshow 的输出是**运行中的 deck**：用 `hyperframes present <project-dir>`（或 Studio present mode）提供服务 —— player 的 `SlideshowController` 读取 island 并驱动 navigation、fragments、branching 和 presenter mode。见下方 **Presenting and handoff**。

**不要把 slideshow 用 `hyperframes render` 渲染成单个 MP4。** deck 被 author 为多个 top-level scene compositions（每个 slide 一个 `data-composition-id`），没有包裹它们的 master-root composition，所以 `render` 只会解析**第一个** composition，并输出一个**静默截断**的 MP4（例如 40 秒 deck 只输出 6 秒）。linear main-line export（仅 main slides，不含 branch sequences）**暂缓** —— 在它发布前，支持的输出是 live `present` deck 和 per-slide `snapshot` stills。如果用户今天需要 linear MP4，说明这个限制，而不是把 `render` 指向 deck。

## Intent confirmation

如果用户明确要求 slideshow、slide show 或 HyperFrames slideshow，就继续使用这个 skill。

如果 skill 是由相邻请求触发，例如 "presentation"、"pitch deck"、"deck"、"interactive deck" 或 "convert this page"，authoring 前先暂停并在询问确认前说明选择。简要解释 HyperFrames slideshow 意味着一个可运行的 deck，带 discrete slides、内置 navigation 和 presenter mode、可编辑 speaker notes、共享 media handling，以及 handoff 前 validation。对于 source-page conversions，也说明目标是在保留原页面 visual design、interactions、motion 和 media behavior 的同时，把页面 movement 转换为 slide-to-slide transitions。

然后问一个简短确认问题：

> Do you want this as a HyperFrames slideshow?

环境提供 yes/no choice UI 时使用它；否则用普通文本询问。

在用户说 yes 之前不要实现 slideshow。如果他们说 no，停止使用这个 skill，并切换到合适的非 slideshow workflow。

---

## The two pieces

### 1. Scenes — declared the normal way

每个 slide 都由一个 scene 支撑。用 `data-composition-id`、`data-start`、`data-duration` 和 `data-label` 声明 scenes：

```html
<div
  data-composition-id="problem"
  data-start="0"
  data-duration="8"
  data-label="The problem"
  data-width="1920"
  data-height="1080"
>
  <!-- clips go here -->
</div>
```

Branch slides（只能通过 hotspot 到达、排除在 main line 外）也完全按同样方式声明 —— 它们只出现在 island 的 `slideSequences` 条目中，而不在 main `slides` array 中。

### 2. The JSON island — one script block per composition

向 composition HTML 添加恰好一个 `<script type="application/hyperframes-slideshow+json">` block。它保存所有 slideshow metadata：

```html
<script type="application/hyperframes-slideshow+json">
  {
    "slides": [...],
    "slideSequences": [...]
  }
</script>
```

island 是 slide order、notes、fragment hold-points、hotspots 和 branch sequences 的唯一 source of truth。把它放在 `<body>` 顶部附近、scene divs 之前，方便查找。

不要把 slideshow manifest 隐藏在另一个 `<script type="application/json">` block 后，再用 runtime code 创建 island。`present` 命令会静态读取 composition HTML，并期望真实的 `application/hyperframes-slideshow+json` island 已经存在。

---

## Schema

### `SlideshowManifest` (the top-level island object)

```json
{
  "slides": [
    /* SlideRef[] — the main line, in order */
  ],
  "slideSequences": [
    /* SlideSequence[] — off-line branch sequences */
  ]
}
```

### `SlideRef`

```json
{
  "sceneId": "problem",
  "notes": "Lead with the pain, not the company.",
  "fragments": [3.5, 5.2, 7.0],
  "hotspots": [
    /* SlideHotspot[] */
  ],

  "ttsScript": null,
  "ttsAudioUrl": null,
  "ttsDurationMs": null
}
```

| Field                                       | Required | Notes                                                                                                                                                   |
| ------------------------------------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `sceneId`                                   | yes      | 必须与某个 scene 的 `data-composition-id` 精确匹配（或提供 explicit `startTime`/`endTime`）。lint rule 通过 `data-composition-id` 解析 scenes。 |
| `notes`                                     | no       | 仅 presenter 可见的文本。永远不展示给 audience。                                                                                                       |
| `fragments`                                 | no       | slide 的 `[start, end]` 范围内的 times（秒）数组 —— 见 Fragments。                                                                                     |
| `hotspots`                                  | no       | 触发 branch 的 interactive overlays —— 见 Branching。                                                                                                  |
| `startTime`                                 | no       | 可选。覆盖匹配 scene 的 time bounds；默认使用 scene 的 start/end。                                                                                      |
| `endTime`                                   | no       | 可选。覆盖匹配 scene 的 time bounds；默认使用 scene 的 start/end。                                                                                      |
| `ttsScript`, `ttsAudioUrl`, `ttsDurationMs` | no       | **Reserved.** schema fields 存在，但 TTS playback 尚未接线。除非为 future build 预填，否则省略。                                                       |

### `SlideHotspot`

```json
{
  "id": "h1",
  "label": "How did we calculate this?",
  "target": "market-deep-dive",
  "region": { "x": 60, "y": 10, "w": 35, "h": 20 }
}
```

| Field    | Required | Notes                                                                                                                           |
| -------- | -------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `id`     | yes      | slide 内唯一。                                                                                                                  |
| `label`  | yes      | 展示给 audience 的 tooltip / button text。                                                                                      |
| `target` | yes      | 必须匹配 `slideSequences` 中的某个 `SlideSequence.id`。                                                                          |
| `region` | no       | 以 slide 百分比表示的 bounding box：`{x, y, w, h}`，范围 `0–100`。省略时渲染为 full-slide labeled button。                       |

### `SlideSequence`

```json
{
  "id": "market-deep-dive",
  "label": "Market sizing methodology",
  "slides": [{ "sceneId": "mkt-1" }, { "sceneId": "mkt-2" }]
}
```

sequence 内的 `slides` 使用与 main line 相同的 `SlideRef` shape。允许 fragments 和 nested hotspots。

---

## Slide writing rules

这些是硬约束，不是建议。reviewer 看到违反它们的 slide 会直接替换。

- **Headline 是完整句子的 claim，不是 label。** 写 "SMBs spend 14 hours/week on manual scheduling"，不要写 "Scheduling problem"。即使忽略视觉，这个句子也应能独立成立。
- **每页一个 idea + 一个 visual。** 如果你想加第二组 bullets 或第二张 chart，就拆成两页。
- **Lead with the punchline.** 最强观点放在最前 —— 在 slide 上和 deck 顺序中都是如此。投资人按从左到右、从上到下阅读，然后就停下。
- **Bottom-up market sizing only.** 不要只写 "$50B TAM" 而不展示计算。自下而上构建：accounts × ACV，或 transactions × take-rate。
- **Font minimum 30pt equivalent.** 在 1920×1080 中，headline 为 72–96px；body copy 为 48px。任何 audience 必须阅读的文字都不要低于 40px。

## Porting source pages

将现有页面转换为 slideshow 时，source fidelity 是契约的一部分。除非用户明确要求 redesign，否则不要用简化近似替换 source-specific widgets。

- 尽可能保留原页面的 visual design、motion language、interactive behavior、media behavior 和 presentation affordances。当 slideshow system 支持 presenter mode 时，使用共享 editable-notes 行为加入 speaker notes，不要做 deck-specific implementation。
- 尽可能精确地移植 source DOM/CSS/JS 的 mechanical visuals：custom players、canvas visualizers、timelines、playheads、stems、expanding circles、hover states 和其他 interactive details 都应在 conversion 中保留。
- 将原生 `<video>` / `<audio>` elements 视为任何 custom media chrome、canvas visualizer、waveform、beat grid 或 playhead 的 source of truth。连接 source 的 media events（`play`, `pause`, `timeupdate`, `seeking`, `seeked`, `ended`, `ratechange`, `volumechange`），并从 `media.currentTime` 派生 visual state；不要运行会偏离实际 playback 的独立 timer。
- 每个带 `src` 的复制 `<video>` 或 `<audio>` 在 lint 前必须有 HyperFrames timing attributes：`data-start` 和 `data-duration`，并且当 audible native audio 应保留时加 `data-has-audio="true"`。slide-specific media 使用 scene 的 time range；可从多个 focused slides 播放的 user-controlled evidence videos 使用 deck-wide range。不要在 media 上留下 `preload="none"`；使用 `metadata` 或 `auto`。
- validation 前解析 source font tokens。如果保留 custom source fonts，为 local/captured font files 添加 `@font-face` rules。如果使用 system fallbacks，将 `font-family: var(--f-body)` 等 tokenized declarations 替换为 concrete render-safe stacks，例如 `system-ui, sans-serif` 或 `ui-monospace, monospace`；不要把 `var(...)` 留作 font family value。
- 审计 source 中不典型的 page movement，尤其是由 scroll、wheel、touch、hash state、resize 或 requestAnimationFrame loop 驱动的行为。将 fixed viewports with translated/scaled "world" layers、parallax、pinned panels、horizontal scrollers、scroll-scrubbed timelines、section snapping 和 zoom-to-element cameras 视为 source behavior。Scroll 常常是 source 的 transition trigger，因此要通过提取其 progress stops、easing 和 camera/focus states 来保留 transition，并把该 motion 重新托管到 slideshow navigation 的 timeline positions、fragments 或 reusable player/harness hook 上。独立 wrappers 即使跳到 slide hold-points，仍需要明确的 navigation-camera transition hook；仅计算 per-slide camera transforms 不够。不要在 slide 内模拟 literal page-scroll-down transition；观众应感到 camera travel/zoom 从一个焦点到另一个焦点，而不是看到网页被滚动。保持每个 slide-to-slide camera move 连续：避免在 source 没有同样表现时出现会反转 x/y 方向或 zoom 的 intermediate route stops。一个到处乱窜再落点的 transition 比更简单直接的 focal move 更糟。
- 保留 source 的 media crop semantics。将 screenshots、tweets/social posts、product UI captures、charts、docs、code、leaderboards，以及任何带可读文字的 image 视为 content evidence，而不是 decorative media：使用 source aspect ratio（`height: auto`）或在 stable frame 内用 `object-fit: contain`。只有当 source 本身如此，或作为刻意 decorative/background/cinematic thumbnails 时，才使用 `object-fit: cover`。把这些 captures 放进 slide 后，检查四边是否截断 text、logos、controls 或 captions；除非 source 本身裁切，否则对有意义内容的可见裁切就是 bug。
- 如果某行为是 slideshows 通用行为，把它放进 player/controller 或 reusable skill snippet。不要用 one-off deck scripts 解决。
- 堆叠的 scene frames 绝不能阻挡 active slide 上的 interaction。Hidden frames 需要 visual hiding 和 event gating 两者：

```css
.scene-frame {
  opacity: 0;
  visibility: hidden;
  pointer-events: none;
}

.scene-frame.is-active {
  opacity: 1;
  visibility: visible;
  pointer-events: auto;
}
```

如果 visibility 是 imperative 驱动，就在 visibility controller 中设置全部三个属性（`opacity`、`visibility` 和 `pointerEvents`）。仅 `opacity: 0` 仍会留下可吞掉点击的 invisible layer。

---

## Fragments: reveal hold-points within a slide

fragment 是 slide 的 `[start, end]` 范围内、composition-timeline 的绝对时间（秒），controller 应在这些时间点持有 reveal state。

**How it works:**

1. Player 进入 fragmented slide —— 直接 seek 到 `fragments[0]` 并 hold。
2. 用户按 Next（或 →）—— controller seek 到 `fragments[1]` 并 hold。
3. 最后一个 fragment 之后，Next 前进到下一张 slide。
4. 没有 fragments 的 slide 会进入 slide 内的 rest frame，通常是 midpoint，而不是精确在 `slide.end`。

Fragment times 必须落在 `[start, end]` 内（包含两端）。lint rule 只拒绝该范围外的 fragments（`time < start` 或 `time > end`）。

Fragment times 是**绝对 composition-timeline positions** —— 与 `data-start` 相同坐标系 —— 不是相对 scene start 的 offsets。

Navigation 是 seek-driven，不是 play-driven。controller 绝不会为了在 fragments 间移动而启动 playback；每个 navigation command 都是确定性 seek 到目标 hold time。设计 fragment states 时，要确保它们在目标 timeline time 上正确。

---

## Branching: hotspots and slide sequences

Branch slides 是同一个 composition timeline 中的真实 scenes。它们只列在 `slideSequences` 下，并从 main-line navigation 中排除 —— player 永远不会访问它们，除非 hotspot 被触发。

**Navigation model:**

- Clicking a hotspot pushes `{sequenceId, slideIndex: 0}` onto the nav stack and enters the branch's first slide.
- **back()** pops the stack and returns to the exact parent slide (the one that held the hotspot).
- **backToMain()** clears the entire stack and returns to the root slide.
- Breadcrumb renders from the stack: `Main deck › Market sizing methodology › Slide 2`.
- The slide counter inside a branch is scoped to that sequence (`1 of 2`, not the main-deck total).

**What to avoid:**

- 不要把 branch scene IDs 加到 main `slides` array。它们必须只出现在 `slideSequences` 条目中。lint rule 会标记重叠。
- Branch scenes 包含在 continuous timeline 中，所以 naive linear video export 会包含它们。Export 只读取 main-line slides（deferred；已在 spec 中标记）。

---

## Worked example: 3-slide deck with fragments and a branch

### Scene HTML (skeleton)

```html
<body style="margin: 0">
  <script type="application/hyperframes-slideshow+json">
    {
      "slides": [
        {
          "sceneId": "hook",
          "notes": "Open with the stat. Pause on the $40B number."
        },
        {
          "sceneId": "problem",
          "notes": "Walk through each pain point one at a time.",
          "fragments": [11.0, 15.0],
          "hotspots": [
            {
              "id": "h1",
              "label": "Where does the $40B figure come from?",
              "target": "market-detail",
              "region": { "x": 55, "y": 60, "w": 40, "h": 20 }
            }
          ]
        },
        {
          "sceneId": "solution",
          "notes": "One sentence: what we do and who it is for."
        }
      ],
      "slideSequences": [
        {
          "id": "market-detail",
          "label": "Market sizing methodology",
          "slides": [{ "sceneId": "mkt-math", "notes": "Bottom-up: 2.3M SMBs × $17k ACV." }]
        }
      ]
    }
  </script>

  <!-- Slide 1 — hook -->
  <div
    data-composition-id="hook"
    data-start="0"
    data-duration="6"
    data-label="The hook"
    data-width="1920"
    data-height="1080"
    style="position: relative; width: 1920px; height: 1080px; overflow: hidden; background: #0a0a0a"
  >
    <section
      class="clip"
      data-start="0"
      data-duration="6"
      data-track-index="1"
      style="position: absolute; inset: 0; display: grid; place-items: center"
    >
      <h1 id="hook-headline" style="font-size: 80px; color: #fff; font-family: sans-serif">
        SMBs lose $40B/year to manual scheduling
      </h1>
    </section>
  </div>

  <!-- Slide 2 — problem (3 fragments) -->
  <div
    data-composition-id="problem"
    data-start="6"
    data-duration="15"
    data-label="The problem"
    data-width="1920"
    data-height="1080"
    style="position: relative; width: 1920px; height: 1080px; overflow: hidden; background: #0a0a0a"
  >
    <section
      class="clip"
      data-start="6"
      data-duration="15"
      data-track-index="1"
      style="position: absolute; inset: 0; padding: 120px 160px; box-sizing: border-box"
    >
      <h2 id="pain-headline" style="font-size: 64px; color: #fff; font-family: sans-serif">
        Three gaps operators can not close
      </h2>
      <p id="pain-1" style="font-size: 48px; color: #ccc; opacity: 0; font-family: sans-serif">
        No-shows cost 23% of booked revenue
      </p>
      <p id="pain-2" style="font-size: 48px; color: #ccc; opacity: 0; font-family: sans-serif">
        Manual reminders take 4h/week per staff
      </p>
      <p id="pain-3" style="font-size: 48px; color: #ccc; opacity: 0; font-family: sans-serif">
        Rescheduling friction drives 40% churn
      </p>
    </section>
  </div>

  <!-- Slide 3 — solution -->
  <div
    data-composition-id="solution"
    data-start="21"
    data-duration="8"
    data-label="The solution"
    data-width="1920"
    data-height="1080"
    style="position: relative; width: 1920px; height: 1080px; overflow: hidden; background: #0a0a0a"
  >
    <section
      class="clip"
      data-start="21"
      data-duration="8"
      data-track-index="1"
      style="position: absolute; inset: 0; display: grid; place-items: center"
    >
      <h2 id="solution-headline" style="font-size: 72px; color: #fff; font-family: sans-serif">
        Acme automates scheduling for service SMBs — no-shows down 80% in 90 days
      </h2>
    </section>
  </div>

  <!-- Branch slide — excluded from main line -->
  <div
    data-composition-id="mkt-math"
    data-start="29"
    data-duration="7"
    data-label="Market math"
    data-width="1920"
    data-height="1080"
    style="position: relative; width: 1920px; height: 1080px; overflow: hidden; background: #111"
  >
    <section
      class="clip"
      data-start="29"
      data-duration="7"
      data-track-index="1"
      style="position: absolute; inset: 0; display: grid; place-items: center"
    >
      <p id="mkt-formula" style="font-size: 56px; color: #fff; font-family: sans-serif">
        2.3M SMBs × $17k ACV = $39B serviceable market
      </p>
    </section>
  </div>

  <script>
    window.__timelines = window.__timelines || {};

    // Slide 2 fragment entrance animations
    gsap.registerPlugin(); // load any plugins before use

    const tl = gsap.timeline({ paused: true });
    window.__timelines["problem"] = tl;

    // Insert positions are absolute composition-timeline times (same as data-start / fragment values).
    tl.from("#pain-1", { opacity: 0, y: 20, duration: 0.4 }, 11.0);
    tl.from("#pain-2", { opacity: 0, y: 20, duration: 0.4 }, 15.0);
    // pain-3 lands at end of slide
    tl.from("#pain-3", { opacity: 0, y: 20, duration: 0.4 }, 13.0);
  </script>
</body>
```

### Key points in the example

- island 的 `sceneId` 值（`"hook"`、`"problem"`、`"solution"`、`"mkt-math"`）与 scene divs 上的 `data-composition-id` 值精确匹配。
- `mkt-math` 只出现在 `slideSequences` 中 —— 它永远不在 top-level `slides` array 中。
- Fragment times（`11.0`、`15.0`）位于 `problem` scene 的 `[6, 21]` 范围内（times 是绝对 composition-timeline positions）。
- hotspot `region`（`x: 55, y: 60, w: 40, h: 20`）将 clickable area 放在 problem slide 的右下象限。
- GSAP timelines 注册在 `window.__timelines` 上并 paused —— HyperFrames engine 驱动 playback；不要在构造时调用 `.play()`。

---

## Wrapping component

在任何 embedding context 中，用 `<hyperframes-slideshow>` 包裹 `<hyperframes-player>`：

```html
<hyperframes-slideshow>
  <hyperframes-player src="deck.html"></hyperframes-player>
</hyperframes-slideshow>
```

`<hyperframes-slideshow>` 提供 navigation chrome（Present、Prev / Next、counter、当 `sound` 存在时的 global mute、fullscreen）、keyboard handling（← / →、Space / Backspace，以及 P for Present）、touch swipe 和 hotspot overlays。

slideshow 会在 mount 时自动给每个内部 `<hyperframes-player>` 设置 `interactive` attribute，因此 composition iframe 内的 clickable controls、links、native media controls 和 custom players 会按预期接收 pointer events。（在 slideshow wrapper 外，你必须手动给 `<hyperframes-player>` 添加 `interactive` —— player 默认在 iframe 上设置 `pointer-events: none`，以防 player host 上的 clicks 被劫持为 timeline playback toggle。）

**Presenter mode:** 使用 slideshow nav capsule 中内置的 Present icon button，或按 P。它调用 `window.open('?mode=audience')` 打开 fullscreen audience tab；原 tab 变成 presenter view（current slide 缩小、next-slide preview、notes、elapsed timer）。两个 tabs 通过 `BroadcastChannel('hf-slideshow:' + location.pathname)` 同步。不要添加自定义 wrapper-level Present button；共享组件拥有其 placement、icon、styling 和 audience-mode hiding。

**Presenting over Google Meet / Zoom (screen share):** 共享 _audience_ surface，把 presenter view 留在自己的屏幕上。

- **Google Meet (or any in-Chrome share):** Present → 在 Meet 中选择 **Share screen → A tab** → 选择 audience tab → 切回 presenter tab。Chrome 会在 backgrounded 时保持 captured tab rendering，因此 animations 和 slide nav 保持 live。不要共享 "A window" 或 "Entire screen" —— 完全被覆盖的 window 会停止 rendering（观众看到 frozen slides），而 entire-screen 会暴露 notes。
- **Zoom (desktop app):** 将 audience tab 拖出为独立 window 并共享该 window。Zoom 通过 OS 捕获，所以如果 audience window 被_完全_覆盖会冻结 —— 使用第二显示器，或让 audience window 的一小条在 presenter view 后可见。

Presenter-driven media playback 有 autoplay-policy 约束：`BroadcastChannel` 可以同步 intent、time 和 state，但不能把 presenter 的 user activation 转移到 audience tab。共享 slideshow player 会镜像 native media events，并先以 muted 启动 remote audience playback；只有当 muted `media.play()` 被拒绝，或 deck 明确需要 audible audience playback 时，才回退到 standalone harness 的 audience unlock behavior。play 被拒绝后不要继续应用 remote `timeupdate` messages，否则 audience 会在没有 playback 的情况下静默 seek through video。

Presenter notes 在 presenter view 中可编辑。Edits 按 deck 和 slide 存储在 `localStorage`，叠加在 manifest notes 上，不重写 composition file。不要向 decks 添加 one-off note-editing scripts；依赖共享 slideshow player 行为。如果 standalone/custom wrapper 确实需要在共享 player 外实现它，使用 `skills/slideshow/references/standalone-harness.md` 中的 deterministic storage snippet。

### Media cleanup on slide exit

slideshow controller 拥有 slide-exit media cleanup。当 navigation 改变 slide 或 sequence 时，它会在进入下一张 slide 前调用 `hyperframes-player.stopMedia()`。该命令：

- posts `stop-media` to the iframe runtime, which stops WebAudio and pauses native `<video>` / `<audio>` elements;
- pauses same-origin iframe media directly as a fallback; and
- pauses parent-frame proxies adopted from iframe media.

Same-slide fragment navigation **不会** stop media。Global/deck-level parent audio（例如通过 `audio-src` 接线的 background track）不被视为 slide media。

不要为普通 media players 添加 per-slide cleanup scripts。将 slide video/audio 保持为 composition 中的普通 media；只有当 player 应保留 audible native video audio，而不是把它当作 silent visual media 时，才使用 `data-has-audio="true"`。

如果 source page 有绑定到 media 的 custom controls 或 visualizations，这些 controls 必须监听 slideshow player 会 stop 和 mute 的同一个 native element。由 slide exit、presenter sync、native controls、custom controls 或 global mute button 引发的 pause，都应通过 media events 更新可见 custom UI，而不是通过 parallel state。

实现 direct iframe fallback cleanup 时，把 iframe media 当作 cross-realm DOM。不要用 parent page 的 `el instanceof HTMLMediaElement` 测试 iframe nodes；这在真实浏览器中会返回 false。在设置 `muted` 或调用 `pause()` 前，使用 `el.ownerDocument.defaultView.HTMLMediaElement`（或等效 tag/duck-type guard）。

### Global nav mute

当 `<hyperframes-slideshow sound>` 渲染 nav mute button 时，该 button 是页面的 global mute control。它必须 mute：

- child `<hyperframes-player>` instances, including same-origin iframe media;
- top-level page `<audio>` / `<video>` elements; and
- wrapper-owned SFX/global `Audio` objects via the `hf-sound` event.

不要在 composition 内添加第二个 mute button。如果 wrapper script 创建了未附加到 DOM 的 `new Audio(...)` objects，它必须监听 `hf-sound` 并对每个 object 设置 `clip.muted = detail.muted`，而不是仅跳过 future plays。

同样的 cross-realm rule 适用于这里：global mute 必须通过 child frame 的 DOM realm 触达 iframe `<video>` / `<audio>` elements。单一 DOM realm 中通过的 unit test 不够；在浏览器中验证点击 nav mute button 后，真实 iframe media elements 报告 `muted: true`。

`hyperframes present` 从 `packages/player/dist` 提供 built bundles。改变 player 或 slideshow chrome behavior 后，在 `packages/player` 中运行 `bun run build`，并重启 present server 后再在浏览器测试。

Presenter notes 在 presenter view 中可编辑。Edits 按 deck 和 slide 存储在 `localStorage`，叠加在 manifest notes 上，不重写 composition file。不要向 decks 添加 one-off note-editing scripts；依赖共享 slideshow player 行为。如果 standalone/custom wrapper 确实需要在共享 player 外实现它，使用 `skills/slideshow/references/standalone-harness.md` 中的 deterministic storage snippet。

### Media cleanup on slide exit

slideshow controller 拥有 slide-exit media cleanup。当 navigation 改变 slide 或 sequence 时，它会在进入下一张 slide 前调用 `hyperframes-player.stopMedia()`。该命令：

- posts `stop-media` to the iframe runtime, which stops WebAudio and pauses native `<video>` / `<audio>` elements;
- pauses same-origin iframe media directly as a fallback; and
- pauses parent-frame proxies adopted from iframe media.

Same-slide fragment navigation **不会** stop media。Global/deck-level parent audio（例如通过 `audio-src` 接线的 background track）不被视为 slide media。

不要为普通 media players 添加 per-slide cleanup scripts。将 slide video/audio 保持为 composition 中的普通 media；只有当 player 应保留 audible native video audio，而不是把它当作 silent visual media 时，才使用 `data-has-audio="true"`。

实现 direct iframe fallback cleanup 时，把 iframe media 当作 cross-realm DOM。不要用 parent page 的 `el instanceof HTMLMediaElement` 测试 iframe nodes；这在真实浏览器中会返回 false。在设置 `muted` 或调用 `pause()` 前，使用 `el.ownerDocument.defaultView.HTMLMediaElement`（或等效 tag/duck-type guard）。

### Global nav mute

当 `<hyperframes-slideshow sound>` 渲染 nav mute button 时，该 button 是页面的 global mute control。它必须 mute：

- child `<hyperframes-player>` instances, including same-origin iframe media;
- top-level page `<audio>` / `<video>` elements; and
- wrapper-owned SFX/global `Audio` objects via the `hf-sound` event.

不要在 composition 内添加第二个 mute button。如果 wrapper script 创建了未附加到 DOM 的 `new Audio(...)` objects，它必须监听 `hf-sound` 并对每个 object 设置 `clip.muted = detail.muted`，而不是仅跳过 future plays。

同样的 cross-realm rule 适用于这里：global mute 必须通过 child frame 的 DOM realm 触达 iframe `<video>` / `<audio>` elements。单一 DOM realm 中通过的 unit test 不够；在浏览器中验证点击 nav mute button 后，真实 iframe media elements 报告 `muted: true`。

`hyperframes present` 从 `packages/player/dist` 提供 built bundles。改变 player 或 slideshow chrome behavior 后，在 `packages/player` 中运行 `bun run build`，并重启 present server 后再在浏览器测试。

---

## Running a slideshow standalone (interim)

**durable answer** 是 engine-hosted：`hyperframes preview --slideshow` / studio present mode 会通过真实 HyperFrames engine host composition，该 engine 驱动 seek-timelines、拥有 gesture frame，并从 composition 中读取 island。该路径即将到来；发布后优先使用它。

在此之前，standalone demos（通过 bare player bundle 在浏览器中打开 composition，没有 engine）需要针对三个 gap 做 workaround：composition 必须暴露 seekable root timeline，island 必须复制到 wrapper 中，wrapper-owned SFX/global audio 应位于 parent frame。这些 patterns 记录在：

```
skills/slideshow/references/standalone-harness.md
```

不要把那里的 patterns 当作 blessed model —— 它们只是为了在 engine-hosted path 落地前过渡。

## Handoff

对于 public 或 user-facing slideshow project，root `index.html` 应是可运行的 slideshow entrypoint。在浏览器中打开它应显示 slideshow navigation 并响应 Next/Prev；不应只暴露 raw composition，让用户必须知道 Studio 或内部 wrapper file。如果 raw HyperFrames composition 必须为了 CLI compatibility 保持分离，把它放在 `composition/index.html` 等 subdirectory 中，并让 scripts/commands 指向该目录。

direct-open wrapper 必须依赖 `<hyperframes-slideshow>` 渲染的内置 Present icon button。不要添加自定义 `#present-btn`、fixed-position button 或 wrapper-specific Present styling。共享组件拥有 control bar，在 `?mode=audience` 中隐藏 Present，并支持 P 作为 keyboard shortcut。

handoff 前验证 direct-open path。如果 `file://` browser restrictions 破坏 iframe media、local scripts 或 same-origin player access，使用 self-contained wrapper，或让 handoff command 启动 local server 并打开可工作的 URL；不要留下 broken 或 ambiguous 的 `index.html`。

对于完成的 slideshow deck，主要 user-facing next step 是 presenter mode，而不是 Studio。运行或提供：

```bash
npx hyperframes present <project-dir>
```

Studio/`preview` 对编辑 composition 有用，但它不是 slideshow 用户清晰的最终目的地。如果你为 slideshow project 创建 `package.json`，且 raw composition 位于 `composition/`，让默认 runnable script 启动 presenter mode：

```json
{
  "scripts": {
    "dev": "npx hyperframes present ./composition",
    "studio": "npx hyperframes preview ./composition"
  }
}
```

handoff 时，包含命令打印的 local presenter URL 和最小说明："Click Present, or press P, to open the audience tab." 如果用户将通过 Google Meet 或 Zoom present，也传达上方 Presenting section 的 screen-share guidance（Meet 中 share audience tab；Zoom 中 share 拖出的 audience window）。如果用户要求你启动 server，就保持 server 运行。

---

## Validation

authoring 或 editing slideshow composition 后，运行：

```bash
npx hyperframes lint
```

然后运行 runtime validation：

```bash
npx hyperframes validate
```

即使命令成功退出，也要把 lint errors 和 validation `StaticGuard` contract messages 当作 blockers。修复文件并重新运行，直到 lint 报告 `0 error(s)` 且 validation 无 runtime errors。

slideshow lint rule 检查：

- 每个 `slide.sceneId` 都解析到现有 scene（通过 `data-composition-id`）。
- 每个 `hotspot.target` 都引用已定义的 `slideSequence` id。
- Fragment times 落在各 slide 的 `[start, end]` 范围内。
- 没有两个 main-line slides 在时间上重叠。

preview 前修复所有 violations。lint 失败的 composition 无法在 player 中正确 parse。
