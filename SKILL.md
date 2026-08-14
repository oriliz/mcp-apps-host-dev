---
name: mcp-apps-host-dev
description: Use when developing or debugging the MCP Apps host layer in the Hermes desktop app — rendering sandboxed iframe cards, bridging postMessage ↔ gateway ↔ MCP server, or fixing security/architectural issues in the card pipeline. Covers both the TypeScript renderer and the Python bridge backend.
version: 1.1.0
author: Hermes Agent
license: MIT
platforms: [macos, linux, windows]
metadata:
  hermes:
    tags: [mcp-apps, host, desktop, iframe, sandbox, bridge, security]
    related_skills: [hermes-agent-skill-authoring, inspecting-hermes-desktop-dom, systematic-debugging]
---

# MCP Apps Host 开发

## Overview

MCP Apps（`io.modelcontextprotocol/ui` 扩展）让 MCP Server 在工具返回值里嵌入交互式 HTML 卡片。这张卡片是一个沙箱 iframe，通过 `postMessage` 与宿主应用通信，可以调用 MCP 工具、上报状态、发送对话消息。

本文档覆盖 Hermes 桌面端作为 **Host（宿主端）** 的完整开发知识——从卡片渲染、桥接通信、安全边界到常见踩坑。

**这不是 MCP Server 端开发指南**（用 `fastmcp` skill 构建 MCP Server），而是 **Host 端**：接收卡片、渲染 iframe、管理沙箱安全、桥接消息。

## When to Use

- 在桌面端新增或调试 MCP Apps 卡片的渲染逻辑
- 排查卡片不显示、iframe 空白、postMessage 不通的问题
- 修改 `mcp-app-card.tsx`、`mcp-app.ts`、`tui_gateway/server.py`、`tools/mcp_tool.py` 中的 MCP Apps 相关代码
- 给桥接层加安全校验（新增工具白名单、追溯审计等）
- 理解 MCP Server → 桌面端卡片的完整数据流

**不要用这个 skill 来做**：
- 构建 MCP Server 本身 → 用 `fastmcp`
- 配置 MCP Server 连接 → 用 Hermes 的 `mcp_servers` 配置
- 纯前端 UI 调整（与卡片无关的）→ 用 `inspecting-hermes-desktop-dom`

## 架构全景

### 数据流：从 MCP Server 到屏幕上的卡片

```
┌────────────────────────────────────────────────────────────┐
│ 1. MCP Server (Python, 你的服务)                            │
│    tool 返回 _meta.ui.resource = {uri, mimeType, html}      │
│    + _meta.ui.csp = {scriptSrc, connectDomains, ...}        │
└──────────────────────┬─────────────────────────────────────┘
                       │ MCP 协议 (stdio / HTTP)
┌──────────────────────▼─────────────────────────────────────┐
│ 2. tools/mcp_tool.py                                        │
│    _extract_mcp_ui() 从 CallToolResult._meta.ui 提取卡片    │
│    _stash_mcp_ui_payload() 按 tool_call_id 暂存             │
│    → 卡片 HTML 不进模型上下文（保护 prompt cache）            │
└──────────────────────┬─────────────────────────────────────┘
                       │ pop_mcp_ui_payload(tool_call_id)
┌──────────────────────▼─────────────────────────────────────┐
│ 3. tui_gateway/server.py (_on_tool_complete)                │
│    pop_mcp_ui_payload() → payload["ui"] = {server, uri,     │
│      mimeType, html, csp}                                   │
│    → 作为 tool.complete 事件发送给桌面端                     │
└──────────────────────┬─────────────────────────────────────┘
                       │ JSON-RPC (tool.complete event)
┌──────────────────────▼─────────────────────────────────────┐
│ 4. apps/desktop/ — @assistant-ui/react 渲染管线              │
│    message-parts.tsx::ChainToolFallback                      │
│    → hasMcpUi(result) 检测 result.ui 字段                   │
│    → <McpAppCard toolCallId={...} result={...} />           │
└──────────────────────┬─────────────────────────────────────┘
                       │ React 渲染
┌──────────────────────▼─────────────────────────────────────┐
│ 5. McpAppCard (mcp-app-card.tsx)                             │
│    ┌──────────────────────────────────────────────┐         │
│    │ sandbox iframe (allow-scripts allow-forms)    │         │
│    │                                              │         │
│    │  [UTP 搜索] [蓝牙耳机 ¥14] [加购]             │         │
│    │                                              │         │
│    │  用户点击 → postMessage({jsonrpc, id,         │         │
│    │    method: "tools/call",                     │         │
│    │    params: {name: "utp_cart_add", ...}})     │         │
│    └──────────────────┬───────────────────────────┘         │
│                       │ window.addEventListener('message')  │
│                       ▼                                     │
│  McpAppCard 桥接逻辑:                                        │
│    ui/* 方法 → 本地处理（初始化、resize、model-context）      │
│    tools/call、resources/* → requestGateway('mcp.app.request')│
└──────────────────────┬─────────────────────────────────────┘
                       │ JSON-RPC
┌──────────────────────▼─────────────────────────────────────┐
│ 6. tui_gateway/server.py (mcp.app.request handler)          │
│    解析 {server, toolCallId, message}                        │
│    → call_mcp_app_request(server, method, params,            │
│        tool_call_id=tool_call_id)                           │
└──────────────────────┬─────────────────────────────────────┘
                       │
┌──────────────────────▼─────────────────────────────────────┐
│ 7. tools/mcp_tool.py::call_mcp_app_request()                │
│    校验 tool_call_id + tool name whitelist                   │
│    → 执行实际的 MCP 调用 (session.call_tool / read_resource) │
│    → 返回 JSON-RPC response                                 │
└──────────────────────┬─────────────────────────────────────┘
                       │
                       ▼ 原路返回 postMessage → iframe
```

### 两张卡片形态

MCP Server 可以通过两种方式提供卡片 HTML：

| 形态 | 来源 | 何时解析 | 示例 |
|---|---|---|---|
| **inline form** | 工具 RESULT 的 `_meta.ui.resource` 直接嵌 HTML | 每次工具调用 | UTP login / checkout / address |
| **referenced form** | 工具 DEFINITION 的 `_meta.ui.resourceUri`（`ui://` URI）| 工具定义发现时缓存 HTML，调用时复用 | UTP catalog_search / cart_list |

referenced form 的 HTML 是静态的（每个 `ui://` URI 只 fetch 一次，缓存在 `_mcp_ui_resource_html_cache`），卡片通过 `ui/initialize` 的 `lastToolResult` 拿到工具数据来渲染。

### ui/initialize 握手

卡片 iframe 加载后，向 host 发送 `ui/initialize` 请求。Host 响应中携带卡片初始化所需的全部上下文：

```typescript
// mcp-app-card.tsx — ui/initialize 响应
reply({
  result: {
    protocolVersion: ...,
    hostInfo: HOST_INFO,
    hostCapabilities: { updateModelContext: { text: {} }, message: { text: {} } },
    lastToolResult: buildLastToolResult(resultRef.current),  // 工具原始 structuredContent
    sessionId: readSessionId(resultRef.current)               // 从 structuredContent.session_id 提取
  }
})
```

- **`lastToolResult`**：从原始工具结果的 `structuredContent` 构建（`buildLastToolResult`）。referenced-form 卡片（如 UTP 搜索）用它渲染初始视图（商品列表），而非空白搜索页。当 `structuredContent` 不存在或 `error` 时返回 `undefined`。
- **`sessionId`**：从 `structuredContent.session_id` 提取（`readSessionId`）。某些 MCP Server（如 UTP）要求每次 `tools/call` 都带 `session_id`。卡片通过 `ui/initialize` 拿到后，理论上应在后续 `tools/call` 中带上——但旧版卡片 SDK 不一定会这样做（见踩坑 #11）。

## 关键文件地图

```
工具调用 → 卡片渲染的完整链路：

tools/mcp_tool.py
  ├── _extract_mcp_ui()           # 从 CallToolResult._meta.ui 提取 inline 卡片
  ├── _resolve_referenced_mcp_ui() # 从 tool definition 解析 referenced 卡片
  ├── _stash_mcp_ui_payload()     # 按 tool_call_id 暂存（不入模型上下文）
  ├── pop_mcp_ui_payload()        # 一次性取出，供 gateway 消费
  ├── call_mcp_app_request()      # ★ 桥接安全入口：校验 + 执行 MCP 调用
  ├── _capture_ui_tool_resources()# 工具发现时记录 referenced-form 元数据
  └── _initialize_declaring_ui()  # 握手时声明 host 支持 UI 扩展

tui_gateway/server.py
  ├── _on_tool_complete()         # pop_mcp_ui_payload → payload["ui"]
  └── @method("mcp.app.request")  # iframe 桥接请求的网关入口

apps/desktop/src/
  ├── components/assistant-ui/
  │   ├── thread/message-parts.tsx    # ★ ChainToolFallback → hasMcpUi → McpAppCard
  │   ├── thread.tsx                  # ⚠️ 退役架构，不要往这加 MCP Apps 代码
  │   └── mcp-app-card.tsx            # ★ 卡片组件：iframe + postMessage 桥接
  ├── lib/chat-messages.ts            # ★ tool result 组装 + MCP UI 缓存持久化
  │                                   #   cacheMcpUi / preserveMcpUiCards (localStorage)
  ├── store/mcp-app.ts                # 卡片→模型通信通道（model-context + message）
  └── app/contrib/wiring.tsx          # ★ 实际控制器（不是 DesktopController）
                                      #   $mcpAppUserMessage 订阅 → ui/message 渲染
```

## 渲染管线

### 卡片如何被选中渲染

`message-parts.tsx` 的 `ChainToolFallback` 是所有工具结果的派发器。MCP Apps 卡片不按 `toolName` 匹配（工具名是动态的），而是检查 `result` 是否携带 `ui` 字段：

```typescript
// message-parts.tsx: ChainToolFallback
import { hasMcpUi, McpAppCard } from '@/components/assistant-ui/mcp-app-card'

// ...在 ChainToolFallback 中：
// MCP Apps: any tool whose result carries an interactive UI card renders
// as a sandboxed iframe, regardless of the (dynamic) MCP tool name.
if (hasMcpUi(props.result)) {
  return <McpAppCard {...props} />
}
```

`hasMcpUi()` 检查 `result.ui` 是否存在且包含必需的 `server`、`html`、`uri` 字段。

### 检查顺序

`ChainToolFallback` 中的检查顺序很重要：

1. `todo` — 提升到专用面板，不渲染为行内卡片
2. `react_to_message`（非错误）— 隐藏（reaction 通过 message.reaction 事件走）
3. **`hasMcpUi(result)`** — MCP Apps 卡片 ← 我们的位置
4. `delegate_task` — 子代理委派
5. `image_generate` — 图片生成
6. `clarify` — 澄清工具
7. 其他 → `ToolFallback` 通用行

### ⚠️ 退役架构：thread.tsx

`thread.tsx` 有自己的本地 `ChainToolFallback` 和 `MESSAGE_PARTS_COMPONENTS`，但这是**退役的旧渲染器**。当前主力渲染路径是 `message-parts.tsx`（被 `assistant-message.tsx` 引用）。**不要在 `thread.tsx` 里加 MCP Apps 代码**——两个文件各有一个 `ChainToolFallback`，只改 `thread.tsx` 会导致卡片不显示。

## 安全边界

### 问题：沙箱 iframe 可以越权调用工具

卡片跑在 `sandbox="allow-scripts allow-forms"` 的 iframe 里，通过 postMessage 桥接到宿主。如果网关不加检查，卡片发的任何 `tools/call` 都会被直接执行。一张搜索卡片理论上可以调用结账、删除订单等任意工具。

### 两层防护

**第一层 — 前端传 `toolCallId`**（`mcp-app-card.tsx`）：

```typescript
// 每次桥接请求都带上原始工具调用的 ID，用于审计追溯
const res = await requestGateway('mcp.app.request', {
    server,
    toolCallId,  // ← 追溯：这个调用来自哪个卡片
    message: msg
})
```

**第二层 — 后端校验工具白名单**（`tools/mcp_tool.py::call_mcp_app_request`）：

```python
if method == "tools/call":
    name = params.get("name")
    # 校验：iframe 要调用的工具必须在 MCP Server 注册的工具列表中
    registered = getattr(server, "_registered_tool_names", None)
    if name and (registered is None or name not in registered):
        return {"error": {"code": -32601,
                "message": f"tool '{name}' not registered on server '{server_name}'"}}
```

`_registered_tool_names` 在 MCP 握手时从 `tools/list` 的结果中自动填充，记录该 Server 实际暴露的所有工具名。

**注意**：校验条件用了 `registered is None or name not in registered`，而不是 `registered and name not in registered`。旧代码用 `or []` 把空列表转为 falsy 值，导致 `[] and ...` 短路跳过校验——在 tools 尚未发现或已重置为空列表时白名单形同虚设。现在的版本：`None`（未初始化）和 `[]`（已初始化但无工具）均阻断所有桥接调用。

**resources/read 白名单 — `ui://` 前缀检查**（同上文件）：

```python
if method == "resources/read":
    uri = params.get("uri")
    # 安全：只有 ui:// 资源可通过桥接访问。非 ui:// URI
    # （db://、file:// 等）是给模型用的，不是给沙箱 iframe 的。
    if not isinstance(uri, str) or not uri.startswith("ui://"):
        return {"error": {"code": -32601,
                "message": f"resource not accessible from a card: {uri}"}}
```

和 `tools/call` 白名单是同一类防御——桥接层对每个出站方法都要校验参数合法性，不能无条件透传。

### 新增 MCP Server 时

如果新增了一个 MCP Server 并希望它的卡片能通过桥接调用工具，不需要额外配置——只要 Server 在 `tools/list` 中声明了工具，`_registered_tool_names` 就会自动包含它。

### CSP 策略：卡片资源加载

卡片在 sandbox iframe 中运行，服务端通过 `_meta.ui.csp` 声明其需要的资源权限。`buildCsp()`（`mcp-app-card.tsx`）根据此配置构建 `Content-Security-Policy`：

```typescript
function buildCsp(csp: McpUiCsp): string {
    const res = (csp.resourceDomains ?? []).join(' ').trim()
    const conn = (csp.connectDomains ?? []).join(' ').trim()
    const script = (csp.scriptSrc || "'unsafe-inline' 'unsafe-eval'").trim()

    return [
        "default-src 'none'",
        `script-src ${script}`,
        `style-src 'unsafe-inline' ${res}`.trim(),
        `img-src data: blob: ${res}`.trim(),
        `font-src data: ${res}`.trim(),
        `media-src ${res || "'none"'}`.trim(),
        `connect-src ${conn || "'none'"}`.trim(),
        "base-uri 'none'",
        'form-action *'
    ].join('; ')
}
```

**关键约束**：

- `img-src` 只放行 `data: blob:` 和 `resourceDomains` 中的域名。如果卡片 HTML 引用了外部图片（如 CDN 商品图），其域名**必须**在 `_meta.ui.csp.resourceDomains` 中声明，否则图片将被 CSP 静默阻挡。
- `connect-src` 默认为 `'none'`（sandbox iframe 无 `allow-same-origin`，`'self'` 无意义）。卡片需要发起网络请求（fetch/WebSocket）时，其域名**必须**在 `connectDomains` 中声明。
- `script-src` 默认 `'unsafe-inline' 'unsafe-eval'`——卡片 HTML 中的内联脚本可以执行，但外部脚本文件需要 `scriptSrc` 额外声明。

**MCP Server 端最佳实践**：在构造 `_meta.ui.csp` 时，列出卡片 HTML 中所有引用到的外部资源域名：

```python
"_meta": {
    "ui": {
        "csp": {
            "resourceDomains": ["img.alicdn.com", "gw.alicdn.com"],  # 图片、字体、CSS
            "connectDomains": ["api.example.com"],                     # fetch/XHR 目标
            "scriptSrc": "'unsafe-inline' 'unsafe-eval' https://cdn.example.com",
        }
    }
}
```

## 卡片→模型通信

MCP Apps 规范定义了两个卡片到模型的通道（在 `store/mcp-app.ts` 中实现）：

### ui/update-model-context（静默状态快照）

卡片把当前视图状态（如搜索关键词、筛选条件）发给模型，作为**下一条用户消息的不可见前缀**：

```typescript
// 规范：每次更新覆盖同一 view 的上一个快照
stageModelContext(viewId, extractTextContent(params))

// 发送下一条用户消息时：
buildOutgoingUserText(userText)
// → "卡片状态:\n搜索: 蓝牙耳机 价格<100\n\n用户输入的文本"
```

### ui/message（触发对话回合）

卡片请求模型发起一个新的对话回合。

```typescript
requestMcpAppUserMessage(text)
// → 通过 $mcpAppUserMessage atom → submitText 发送路径
```

**安全：per-card 2 秒 debounce**。恶意/有问题的卡片在循环中连续发 `ui/message` 会触发无限次模型对话，每次消耗 API 往返 + token。`requestMcpAppUserMessage(text, debounceKey)` 按 `toolCallId` 隔离限流窗口——同一张卡片 2 秒内只放行一次，不同卡片互不阻塞。

## 常见踩坑

### 1. 卡片不显示 → 检查渲染器

**症状**：工具调用成功但看不到卡片，只看到普通的 tool fallback 行。

**排查**：
1. 确认 `message-parts.tsx`（不是 `thread.tsx`）里有 `hasMcpUi` 检查
2. 打开 DevTools，在 `ChainToolFallback` 里加 `console.log` 看 `result.ui` 是否存在
3. 检查 `hasMcpUi()` 的校验逻辑——需要 `server`、`html`、`uri` 三个字段都存在

### 2. 只在 thread.tsx 加了代码，卡片不显示

`thread.tsx` 是退役渲染器。虽然它还有自己的 `ChainToolFallback`，但当前走的是 `message-parts.tsx`。改动必须放在 `message-parts.tsx` 中。

### 3. iframe 空白 / postMessage 不通

**检查清单**：
- iframe 的 `sandbox` 属性是否包含 `allow-scripts`
- CSP 策略是否阻止了卡片需要的脚本/连接域名（`buildCsp()` 从 `_meta.ui.csp` 构建）
- postMessage 的 `event.source !== iframe.contentWindow` 校验是否误杀（其他 iframe 也会发 message）
- DevTools Console 搜 `[mcp-app<-card]` / `[mcp-app->card]` 看消息流（仅 DEV 模式输出）

### 4. 桥接调用失败"tool not registered"

**原因**：卡片试图调用的工具名不在 `_registered_tool_names` 中。

**排查**：
1. 确认 MCP Server 的 `tools/list` 确实声明了该工具
2. 确认 Server 在 Hermes 中已成功连接（`hermes tools` 能看到 `mcp_<server>_*` 工具）
3. 重启 Hermes 后首次连接会重新 populate `_registered_tool_names`

### 5. referenced-form 卡片始终不渲染

referenced form 需要两步：
1. 工具定义发现时 `_capture_ui_tool_resources()` 记录 `_meta.ui.resourceUri`
2. 工具调用时 `_resolve_referenced_mcp_ui()` fetch `ui://` 资源

**排查**：
- 确认 Hermes 在 `initialize` 时声明了 `io.modelcontextprotocol/ui` 扩展（`_initialize_declaring_ui`），否则 Server 不会在工具定义中带 `_meta.ui`
- 检查 `_mcp_ui_resource_html_cache` 是否已有缓存（key 是 `(server_name, resourceUri)`）
- 检查 `resources/read` 调用是否成功

### 6. 大 HTML 撑爆模型上下文

MCP Apps 卡片的 HTML 完全不进模型上下文。`_stash_mcp_ui_payload` 把 HTML 暂存在进程内存中（按 `tool_call_id` 索引），只通过 `tool.complete` 事件发送给桌面渲染端。模型看到的是不含 HTML 的工具结果。

### 7. 新增桥接方法时忘记加安全校验

**这是最危险的模式。** 任何 `call_mcp_app_request` 中新增的 `if method == "xxx":` 分支，都必须检查参数合法性再透传。

- `tools/call`: 校验 name 在 `_registered_tool_names` 中 ✅
- `resources/read`: 校验 uri 以 `ui://` 开头 ✅
- 如果新增其他 bridge 方法，必须有对应的参数校验

不加校验 = 沙箱 iframe 可以调用任意 MCP Server 能力，绕过所有权限控制。

### 8. 图片 / 外部资源加载不出来

**症状**：卡片正常渲染，但图片显示空白/裂图，或 CSS 字体/媒体资源加载失败。

**原因**：CSP `img-src` / `font-src` / `media-src` 只放行了 `resourceDomains` 中声明的域名。如果 MCP Server 在 `_meta.ui.csp.resourceDomains` 里遗漏了卡片 HTML 中引用到的 CDN/图片域名，CSP 会静默阻挡这些资源。

**排查**：
1. 打开 DevTools Console，搜 `Content-Security-Policy` 违规报告（Chrome 会打印具体被挡的 URL 和指令）
2. 确认卡片 HTML 中所有 `<img src="...">`、`<link href="...">` 的域名都在 `resourceDomains` 中
3. 注意 `connect-src` 默认是 `'none'`（不是 `'self'`）——sandbox iframe 无同源概念，`'self'` 无意义。如果卡片需要 fetch/WebSocket，域名必须在 `connectDomains` 中
4. 确认 MCP Server 在构造 `_meta.ui.csp` 时正确填入了所需域名

**修复**：在 MCP Server 端补充 `_meta.ui.csp.resourceDomains` 和/或 `connectDomains`，列出卡片引用的所有外部资源域名。

### 9. `_registered_tool_names` 为空时桥接白名单被绕过（已修复）

**历史坑**：旧代码用 `registered = getattr(...) or []` + `if name and registered and ...` 做校验。当 `_registered_tool_names` 为 `[]`（尚未填充、或 disconnect 后重置）时，`[]` 是 falsy，`[] and ...` 短路跳过整个校验块——白名单形同虚设。

**已修复**：改为 `registered = getattr(...)` + `if name and (registered is None or name not in registered)`。`None`（未初始化）和 `[]`（已填充但为空）均阻断所有桥接调用。新增测试覆盖这两种情况：`test_tools_call_blocks_when_registered_tool_names_empty` 和 `test_tools_call_blocks_when_registered_tool_names_unset`。

### 10. CSP 排查技巧

在 DevTools Console 中执行以下检查：

```javascript
// 查看 iframe 实际应用的 CSP
const iframe = document.querySelector('iframe[title*="ui://"]')
console.log(iframe?.contentDocument?.querySelector('meta[http-equiv="Content-Security-Policy"]')?.content)

// 监听 CSP 违规（在卡片 iframe 的 console 中）
document.addEventListener('securitypolicyviolation', e => {
    console.warn('CSP blocked:', e.blockedURI, '→ directive:', e.violatedDirective)
})
```

### 11. 卡片 tools/call 缺少 session_id → 数据不渲染

**症状**：卡片渲染了但内容空白（如结账卡片"应付金额 -"，无商品信息），agent 文本回复里有完整数据。

**根因**：卡片初始化后主动通过 `tools/call` 再次调用 MCP 工具刷新数据（如 `utp_checkout_get`、`utp_address_form`），但旧版卡片 SDK 不会把 `ui/initialize` 响应中的 `sessionId` 转发到 `tools/call` 的 `arguments` 里。UTP 等 MCP Server 要求每次调用必须带 `session_id`，缺少时返回 `isError: true`，卡片拿不到数据。

**修复**：Host 在代理 `tools/call` 时自动注入 `session_id`（`mcp-app-card.tsx`）：

```typescript
if (method === 'tools/call') {
    const params = (msg.params ?? {}) as Record<string, unknown>
    const args = (params.arguments ?? {}) as Record<string, unknown>
    if (args.session_id === undefined) {
        const sid = readSessionId(resultRef.current)
        if (sid) {
            outgoingMsg = { ...msg, params: { ...params, arguments: { ...args, session_id: sid } } }
        }
    }
}
```

**调试技巧**：在 `ui/initialize` handler 和 `tools/call` proxy 中加 `console.log` 输出 `hasLastToolResult`、`sessionId`、`tools/call` 的请求/响应 `isError` 和 `hasSC`，通过 CDP `evaluate` 在 iframe 重载后捕获。

### 12. HMR 重载后卡片消失 → localStorage 持久化

**症状**：开发模式下 Vite HMR 触发页面重载后，已渲染的 MCP App 卡片全部消失。

**根因**：`_mcpUiBySession` 是进程内 `Map`，HMR 重载清空内存。`ui` payload 只在 `tool.complete` 事件中一次性传递（server 端 `pop_mcp_ui_payload` 是单次消费），不会被持久化到 session 数据库。

**修复**（`chat-messages.ts`）：
- `cacheMcpUi()` 同时写内存 Map 和 `localStorage`（key: `mcp-ui:<sessionId>`）
- `preserveMcpUiCards()` 在内存 Map 未命中时回退读 `localStorage`
- `clearCachedMcpUi()` 同时清内存和 `localStorage`
- 跨 session ID 匹配：当 `preserveMcpUiCards` 收到的 runtime session ID 与存储时的 event session ID 不一致时，扫描所有 `mcp-ui:*` localStorage 条目按 tool-call ID 匹配

**注意**：用 `localStorage` 而非 `sessionStorage`——`localStorage` 跨 HMR 重载和 app 重启都存活，匹配 MCP server 连接的生命周期（连接在 app 重启后也会重建）。

### 13. Gateway 进程挂了但 Electron 活着 → 消息无回复

**症状**：在桌面端发消息，完全没有回复，也不报错。

**根因**：Electron 主进程和 Python gateway（`hermes_cli.main serve`）是两个独立进程。Gateway 可能因 API 端点超时（`stream_interrupt_abort` + `tcp_force_closed=1`）、内存耗尽或其他原因崩溃，但 Electron 壳不会自动重启它。用户看到的是"活着但没反应"的 app。

**排查**：
```bash
# 1. 检查 gateway 进程是否存活
ps aux | grep "hermes_cli.main serve" | grep -v grep

# 2. 检查最新 agent 日志（是否有新 turn 记录）
tail -10 ~/.hermes/logs/agent.log

# 3. 如果日志停在几分钟前且没有新 turn → gateway 挂了

# 4. 检查 API 端点是否可达
curl -s --max-time 10 "<base_url>/chat/completions" -H "Authorization: Bearer <key>" -d '{"model":"<model>","messages":[{"role":"user","content":"hi"}],"max_tokens":5}'
```

**修复**：杀掉残留 Electron 进程后重启 `npm run dev`。如果 API 端点本身不稳定（如偶发超时），考虑在 `config.yaml` 中配置 `request_timeout` 或使用更稳定的 provider。

## 验证清单

修改 MCP Apps Host 代码后，检查以下项目：

- [ ] `message-parts.tsx` 的 `ChainToolFallback` 中有 `hasMcpUi` 检查，且位于 `react_to_message` 之后、`delegate_task` 之前
- [ ] 没有在 `thread.tsx` 里添加 MCP Apps 代码
- [ ] 实际控制器是 `contrib/wiring.tsx`（`ContribWiring`），不是 `DesktopController`——`$mcpAppUserMessage` 订阅必须在 `ContribWiring` 中
- [ ] 如果改了桥接层，`toolCallId` 从 `mcp-app-card.tsx` → `tui_gateway/server.py` → `call_mcp_app_request` 完整传递
- [ ] `call_mcp_app_request` 中对 `tools/call` 做了工具名白名单校验（`registered is None or name not in registered`，不依赖空列表的 falsy 短路）
- [ ] `call_mcp_app_request` 中对 `resources/read` 做了 `ui://` 前缀校验
- [ ] `tools/call` 代理路径在卡片参数缺少 `session_id` 时自动注入（`readSessionId(resultRef.current)`）
- [ ] `cacheMcpUi` 同时写内存 Map 和 `localStorage`；`preserveMcpUiCards` 在内存未命中时回退读 `localStorage`
- [ ] `requestMcpAppUserMessage` 有 per-card debounce（`debounceKey` 参数），不会在紧循环中无限触发模型回合，也不会跨卡片误伤
- [ ] `vitest` 通过：`cd apps/desktop && npx vitest run src/components/assistant-ui/mcp-app-card.test.tsx src/store/mcp-app-security.test.ts`
- [ ] Python 测试通过：`python -m pytest tests/test_mcp_apps_ui.py -x`
- [ ] 在桌面端实际渲染一张 MCP Apps 卡片，确认 iframe 显示、点击按钮能触发工具调用、卡片内数据完整（非空白）
- [ ] 开发模式下触发 HMR 重载后，已渲染的卡片仍然存活
