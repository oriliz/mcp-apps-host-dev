# mcp-apps-host-dev

一个面向 **MCP Apps 宿主端（Host）** 开发的 Agent Skill：教 AI 编码助手如何在 Hermes 桌面端渲染沙箱 iframe 卡片、打通 `postMessage ↔ gateway ↔ MCP Server` 三段桥接，以及如何守住卡片的安全边界。

## 这个 Skill 解决什么问题

MCP Apps（`io.modelcontextprotocol/ui` 扩展）让 MCP Server 在工具返回值里嵌一段交互式 HTML。这段 HTML 在宿主应用里以沙箱 iframe 的形式渲染，能反过来调用 MCP 工具、上报视图状态、发起对话回合。

**Server 端怎么写，社区已经有不少资料；Host 端怎么接，几乎没人写。** 而 Host 端恰好是最容易踩坑的一侧：

- 卡片明明返回了，界面上却只有一行灰色的 tool fallback
- 改了渲染器却没生效——因为仓库里有两个同名的 `ChainToolFallback`，你改的那个已经退役了
- iframe 一片空白，CSP 静默拦掉了脚本
- 一张搜索卡片理论上可以通过桥接调用「结账」「删单」等任意工具
- 卡片 HTML 动辄几十 KB，一不小心就进了模型上下文，prompt cache 直接失效
- 卡片渲染了但内容空白——旧版卡片 SDK 不把 `session_id` 转发到 `tools/call`，MCP Server 拒绝返回数据
- 开发模式下 HMR 一刷新，已渲染的卡片全部消失
- Gateway 进程挂了但 Electron 还活着，发消息完全没反应

这个 Skill 把这些知识固化成一份 AI 可读的排查手册：完整数据流图、关键文件地图、两层安全防护的实现、13 个高频踩坑的症状与排查步骤，以及一份改完代码后的验证清单。

## 什么时候会触发

Agent 检测到以下场景时会自动加载本 Skill：

- 在桌面端新增或调试 MCP Apps 卡片的渲染逻辑
- 排查卡片不显示、iframe 空白、postMessage 不通
- 修改 `mcp-app-card.tsx` / `mcp-app.ts` / `tui_gateway/server.py` / `tools/mcp_tool.py` 里的 MCP Apps 相关代码
- 给桥接层加安全校验（工具白名单、调用追溯审计等）
- 想搞清楚 MCP Server → 屏幕上那张卡片的完整链路

**不适用**：

| 你要做的事 | 该用什么 |
|---|---|
| 构建 MCP Server 本身 | `fastmcp` / `build-mcp-server` |
| 配置 MCP Server 连接 | Hermes 的 `mcp_servers` 配置 |
| 与卡片无关的纯前端 UI 调整 | `inspecting-hermes-desktop-dom` |

## Skill 里有什么

| 章节 | 内容 |
|---|---|
| **架构全景** | 7 层数据流 ASCII 图：MCP Server → `mcp_tool.py` → `tui_gateway` → assistant-ui 渲染管线 → `McpAppCard` → 桥接回网关 → 执行真实 MCP 调用 |
| **两张卡片形态** | inline form（RESULT 里嵌 HTML）vs referenced form（DEFINITION 里的 `ui://` URI + HTML 缓存）的差异与解析时机 |
| **关键文件地图** | 链路上每个函数的职责：`_extract_mcp_ui` / `_resolve_referenced_mcp_ui` / `_stash_mcp_ui_payload` / `pop_mcp_ui_payload` / `call_mcp_app_request` / `_capture_ui_tool_resources` / `_initialize_declaring_ui` |
| **渲染管线** | 卡片为什么按 `result.ui` 而不是 `toolName` 派发；`ChainToolFallback` 中 7 项检查的正确顺序；⚠️ 退役渲染器 `thread.tsx` 的陷阱 |
| **安全边界** | 沙箱 iframe 越权调用工具的攻击面 + 两层防护（前端传 `toolCallId` 做追溯、后端用 `_registered_tool_names` 做白名单校验）的完整代码 |
| **卡片→模型通信** | `ui/update-model-context`（静默状态快照，作为下一条用户消息的不可见前缀）与 `ui/message`（触发新对话回合）的实现 |
| **常见踩坑** | 13 个真实问题的症状 → 原因 → 排查步骤，含 session_id 注入、HMR 卡片持久化、Gateway 崩溃排查 |
| **验证清单** | 12 项改完代码必须逐条确认的检查项，含前后端测试命令 |

## 安装

### 方式一：npx skills（推荐）

```bash
npx skills add oriliz/mcp-apps-host-dev
```

支持 Claude Code、Codex、Cursor、Qoder 等 75+ agent。加 `-g` 全局安装，加 `-a claude-code` 指定目标 agent。

### 方式二：手动克隆

Skill 是一个包含 `SKILL.md` 的目录，放进 Agent 的 skills 目录即可，无需插件、无需配置。

```bash
git clone https://github.com/oriliz/mcp-apps-host-dev.git

# 按你使用的 Agent 选一个目标目录
cp -r mcp-apps-host-dev ~/.claude/skills/mcp-apps-host-dev    # Claude Code
cp -r mcp-apps-host-dev ~/.qoder/skills/mcp-apps-host-dev     # Qoder
cp -r mcp-apps-host-dev ~/.hermes/skills/mcp-apps-host-dev    # Hermes
```

或者直接放进项目里，作为仓库级 Skill 随代码一起版本化：

```bash
cp -r mcp-apps-host-dev <your-repo>/.agents/skills/mcp-apps-host-dev
```

## 使用方式

装好之后不需要显式调用，用自然语言描述问题即可：

```
MCP 工具返回了卡片但桌面端只显示一行 tool fallback，帮我查一下
```

```
我要给 mcp.app.request 桥接加一层工具白名单校验
```

```
讲一下 referenced-form 的卡片 HTML 是在哪一步 fetch 的
```

Agent 会命中触发条件、加载 `SKILL.md`，然后按里面的文件地图和排查步骤定位问题——而不是从头 grep 整个仓库。

## 前置条件

本 Skill 描述的是 [Hermes Agent](https://github.com/NousResearch/hermes) 桌面端的 Host 实现，涉及这些文件：

```
tools/mcp_tool.py                              # Python：提取、暂存、桥接执行
tui_gateway/server.py                          # Python：tool.complete 事件 + mcp.app.request 网关
apps/desktop/src/
  ├── components/assistant-ui/
  │   ├── thread/message-parts.tsx             # TS：★ 当前主力渲染派发器
  │   ├── thread.tsx                           # TS：⚠️ 退役渲染器，别往这加代码
  │   └── mcp-app-card.tsx                     # TS：iframe + postMessage 桥接 + session_id 注入
  ├── lib/chat-messages.ts                     # TS：tool result 组装 + MCP UI 缓存持久化 (localStorage)
  ├── store/mcp-app.ts                         # TS：卡片→模型通信通道
  └── app/contrib/wiring.tsx                   # TS：★ 实际控制器（ 订阅）
```

如果你在别的宿主应用里实现 MCP Apps，分层思路和安全边界依然通用（提取 → 暂存 → 事件下发 → iframe 渲染 → 桥接校验 → 执行真实调用），只需替换具体文件名与函数名。

改完代码后 Skill 会要求跑这两条测试：

```bash
cd apps/desktop && npx vitest run src/components/assistant-ui/mcp-app-card.test.tsx
python -m pytest tests/tools/test_mcp_apps_ui.py -x
```

## 相关 Skill

- `hermes-agent-skill-authoring` — 编写 Hermes Skill 本身
- `inspecting-hermes-desktop-dom` — 桌面端 DOM 检查与前端调试
- `systematic-debugging` — 通用的系统化排查方法论

## 贡献

发现踩坑没被覆盖、或者 Host 端实现有变动，欢迎开 Issue 或 PR。踩坑章节请按 **症状 → 原因 → 排查步骤** 的格式写，方便 Agent 直接照着执行。

## License

MIT
