---
name: hyperframes-registry
description: 安装 registry blocks 和 components，并把它们接入 HyperFrames compositions。用于运行 hyperframes add、安装 block 或 component、把已安装项接入 index.html，或处理 hyperframes.json。涵盖 add 命令、安装位置、block sub-composition 接线、component snippet 合并、registry discovery，以及创作一个新的 block 或 component 并贡献到上游（idea → scaffold → validate → PR）。
---

# HyperFrames Registry

registry 提供可通过 `hyperframes add <name>` 安装的可复用 blocks 和 components。

- **Blocks** — 独立的 sub-compositions（自有 dimensions、duration、timeline）。通过 host composition 中的 `data-composition-src` 引入。
- **Components** — effect snippets（没有自有 dimensions）。直接粘贴到 host composition 的 HTML 中。

## 快速参考

```bash
hyperframes add data-chart              # install a block
hyperframes add grain-overlay           # install a component
hyperframes add shimmer-sweep --dir .   # target a specific project
hyperframes add data-chart --json       # machine-readable output
hyperframes add data-chart --no-clipboard  # skip clipboard (CI/headless)
```

安装后，CLI 会打印写入了哪些文件，以及一个可粘贴到 host composition 的 snippet。该 snippet 是起点——接线 blocks 时，你需要添加 `data-composition-id`（必须匹配 block 的内部 composition ID）、`data-start` 和 `data-track-index` attributes。

注意：`hyperframes add` 只适用于 blocks 和 components。对于 examples，请改用 `hyperframes init <dir> --example <name>`。

## 安装位置

Blocks 默认安装到 `compositions/<name>.html`。
Components 默认安装到 `compositions/components/<name>.html`。

这些路径可在 `hyperframes.json` 中配置：

```json
{
  "registry": "https://raw.githubusercontent.com/heygen-com/hyperframes/main/registry",
  "paths": {
    "blocks": "compositions",
    "components": "compositions/components",
    "assets": "assets"
  }
}
```

完整细节见 [install-locations.md](./references/install-locations.md)。

## 接线 blocks

Blocks 是独立 compositions——在你的 host `index.html` 中通过 `data-composition-src` 引入它们：

```html
<div
  data-composition-id="data-chart"
  data-composition-src="compositions/data-chart.html"
  data-start="2"
  data-duration="15"
  data-track-index="1"
  data-width="1920"
  data-height="1080"
></div>
```

关键 attributes：

- `data-composition-src` — block HTML file 的路径
- `data-composition-id` — 必须匹配 block 的内部 ID
- `data-start` — block 在 host timeline 中出现的时间（秒）
- `data-duration` — block 播放多久
- `data-width` / `data-height` — block canvas dimensions
- `data-track-index` — 图层顺序（更高 = 在前）

完整细节见 [wiring-blocks.md](./references/wiring-blocks.md)。

## 接线 components

Components 是 snippets——把它们的 HTML 粘贴到你的 composition markup，把它们的 CSS 粘贴到你的 style block，把它们的 JS 粘贴到你的 script（如果有）：

1. 读取已安装文件（例如 `compositions/components/grain-overlay.html`）
2. 将 HTML elements 复制到你的 composition 的 `<div data-composition-id="...">` 中
3. 将 `<style>` block 复制到你的 composition styles 中
4. 将任何 `<script>` 内容复制到你的 composition script 中（放在你的 timeline code 之前）
5. 如果 component 暴露 GSAP timeline integration（见 snippet 中的 comment block），将这些 calls 添加到你的 timeline

完整细节见 [wiring-components.md](./references/wiring-components.md)。

## Discovery

浏览可用项目：

```bash
# Read the registry manifest
curl -s https://raw.githubusercontent.com/heygen-com/hyperframes/main/registry/registry.json
```

每个 item 的 `registry-item.json` 包含：name、type、title、description、tags、dimensions（仅 blocks）、duration（仅 blocks）和 file list。

按 type 和 tags 过滤的细节见 [discovery.md](./references/discovery.md)。

## 贡献新的 block 或 component

要创作一个新的 registry item（caption style、VFX block、transition、lower third 或 reusable component）并作为上游 PR 交付——不是安装现有 item——请遵循 [contributing.md](./references/contributing.md) 中完整的 idea → scaffold → build → validate → preview → ship workflow。可复制粘贴的 starter templates（caption / VFX / component / `registry-item.json`）位于 [templates.md](./references/templates.md)。
