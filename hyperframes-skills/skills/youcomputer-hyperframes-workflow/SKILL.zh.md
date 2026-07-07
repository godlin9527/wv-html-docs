---
name: youcomputer-hyperframes-workflow
description: 当用户要求通过 Web 前端在 YouComputer 预览机器中运行、测试、调试或比较 HyperFrames workflow skills 时使用此 skill，尤其适用于预览计算机 URL、Hermes ACP 会话、Markdown skill mentions、motion-graphics/product-launch workflow skills，以及验证前端可见会话是否确实注入了请求的 skill 并渲染了视频输出。
---

# YouComputer HyperFrames Workflow

## 概述

通过 YouComputer 前端运行 HyperFrames workflow skills，而不是通过隐藏的 CLI 一次性命令运行；在报告成功之前，同时验证 skill 注入和视频输出。

当用户想要在类似 `https://preview.youmind.com/c/<shortId>/` 的预览计算机上运行 workflow skill、询问为什么会话在 UI 中不可见，或质疑 HyperFrames skill 是否真的被注入时，使用此 skill。

## 不可协商项

- 如果用户要求通过前端运行，请打开 `/c/<shortId>/` 页面，并在那里创建一个新的前端会话。不要悄悄替换为 `hermes -z`。
- 使用正确的 Markdown skill mention syntax：`[$Label](/absolute/path/to/SKILL.md)`。
- 对于沙盒中已安装的 workflow skills，优先使用 `/home/user/.agents/skills/<skill-name>/SKILL.md` 下的路径。
- 将 UI chip text（例如 `$motion-graphics`）仅视为显示内容。当正确性很重要时，应从 Hermes 状态或 relay 行为验证注入。
- 在声称成功之前，验证生成的项目文件、MP4，以及基本 HyperFrames 诊断结果。

## 正确的 Mention Syntax

使用带链接的 mention，而不是普通文本：

```markdown
[$motion-graphics](/home/user/.agents/skills/motion-graphics/SKILL.md)

请用上面自动注入的 /motion-graphics workflow skill 跑一个最小验证。
项目目录：/home/user/videos/motion-graphics-mention-smoke
目标：生成一个 3 秒、16:9、无外部素材依赖的 motion graphics 验证视频。
要求：
- 按 /motion-graphics workflow 生成 shot-plan.json。
- 渲染 MP4 到 /home/user/videos/motion-graphics-mention-smoke/renders/video.mp4。
- 最后汇报：是否通过 mention 自动注入、shot-plan.json 是否存在、video.mp4 是否存在、时长/大小、lint/inspect 结果。
```

原因：YouComputer 会将 skill mentions 序列化为 Markdown links，ACP relay 会从链接路径注入 `SKILL.md` 内容。普通的 `$name` 只有在前端解析器能够唯一解析时才可能工作；绝对路径 Markdown link 是稳健形式。

关于源契约，请阅读 `references/source-contract.md`。

## 前端工作流

1. 当任务依赖用户已登录的前端状态时，在用户的浏览器中打开目标预览计算机 URL，例如 `https://preview.youmind.com/c/wwpvbr5c/`。
2. 从 UI 创建新会话。用户应该能够在 YouComputer 会话下拉菜单中看到该会话。
3. 粘贴/发送一个以带链接的 skill mention 开头的 prompt。
4. 保持请求具体：说明 skill 名称、输出目录、时长/宽高比、预期渲染路径和所需验证。
5. 从前端等待运行完成。如果页面或 WebSocket 损坏，请报告该阻塞问题，而不是切换到 CLI 执行。

## 验证注入

在沙盒中，确认新会话是 ACP/前端会话：

```bash
hermes sessions list --source acp --limit 10
```

然后检查存储的用户消息中是否包含注入的 skill body：

```bash
sqlite3 /home/user/.hermes/state.db "select content from messages where session_id='<session-id>' and role='user' order by id asc limit 1;"
```

正确注入的 prompt 应包含类似以下标记：

```text
<!-- yc:skill_injection:start -->
[Skill: motion-graphics]
```

如果会话只出现在 `--source cli` 下，则它不是前端可见的运行。

## 验证输出

检查 workflow 的预期文件，而不要只依赖 assistant 的文本：

```bash
test -f /home/user/videos/<project>/shot-plan.json
test -f /home/user/videos/<project>/renders/video.mp4
ffprobe -v error -show_entries stream=codec_type,codec_name,width,height,r_frame_rate,nb_frames -show_entries format=duration,size -of default=noprint_wrappers=1 /home/user/videos/<project>/renders/video.mp4
cd /home/user/videos/<project> && npx hyperframes lint .
cd /home/user/videos/<project> && npx hyperframes inspect .
```

报告时长、尺寸、文件大小、`lint` 错误/警告，以及 `inspect` 布局问题。`gsap_studio_edit_blocked` 警告对已渲染输出可能是良性的，但涉及重叠 opacity/transform 的警告应报告，并在质量重要时修复。

## 调试会话经验

- `hermes -z` 会创建 CLI 会话。它们可以生成文件，但与通过 YouComputer 前端创建的会话不同。
- 只在仓库内搜索 `youmind/hyperframes/skills` 可能会漏掉已安装的运行时 workflow skills。请检查 `/home/user/.agents/skills`。
- 当需要自动注入时，"Use motion-graphics" 并不够。请使用 Markdown link mention。
- 前端 mention 可能在视觉上渲染为 `$skill-name` chip，而序列化后的 prompt 仍然是 Markdown link。应信任序列化内容和 Hermes DB，而不仅是 chip。
- 在同时检查已安装 skills 目录和 relay 注入路径之前，不要称某个 workflow “missing”。
- 不要假设预期渲染文件名是 `final.mp4`；许多 HyperFrames workflow skills 会渲染到 `renders/video.mp4`。

## 已知良好示例

在预览计算机 `wwpvbr5c` 上一次成功的前端 ACP 运行中，所用 prompt 为：

```markdown
[$motion-graphics](/home/user/.agents/skills/motion-graphics/SKILL.md)
```

存储的消息包含 `<!-- yc:skill_injection:start -->` 和 `[Skill: motion-graphics]`。输出存在于：

```text
/home/user/videos/motion-graphics-mention-smoke/renders/video.mp4
```

该渲染为 3.0 秒、1920x1080、30 fps、90 帧，并且 `npx hyperframes inspect .` 报告 0 个布局问题。
