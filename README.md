# skill-mcp

将 MCP (Model Context Protocol) 服务器的工具暴露给 OctoSucker Agent，使 LLM 可调用任意 MCP 服务（如 OpenBB、Exa、OpenNews 等）提供的工具。

## 配置

在 Agent 的 `skills` 配置中为 `github.com/OctoSucker/skill-mcp` 指定 `servers` 列表，每项包含：

- **id**: 服务器标识，用于生成工具名前缀（如 `mcp_exa_web_search_exa`）
- **url**: MCP 端点 URL
- **transport**（可选）：`streamable`（默认）或 `sse`。若连 OpenBB/FastMCP 出现 400，可改为 `sse`，并让服务端用 `--transport sse` 启动。

示例：

```json
"github.com/OctoSucker/skill-mcp": {
  "servers": [
    { "id": "exa", "url": "https://mcp.exa.ai/mcp" },
    { "id": "openbb", "url": "http://127.0.0.1:8001/mcp", "transport": "sse" }
  ]
}
```

## 行为

- **Init**：解析 `servers`，不建立连接。
- **Register**：对每个 server 连接 MCP、拉取 `ListTools`，将每个远端工具注册为本地 Tool，名称为 `mcp_{id}_{工具名}`，描述与参数来自 MCP。
- **调用**：Agent 调用 `mcp_*` 时，Skill 再连接对应 MCP、执行 `CallTool`，将结果返回给 Agent。

单次调用采用「按需连接」：每次执行工具时新建连接并调用，结束后关闭，避免长连接与 SDK 已知的 transient error 问题。

## 依赖

- [modelcontextprotocol/go-sdk](https://github.com/modelcontextprotocol/go-sdk)（官方 Go MCP SDK，Streamable HTTP 客户端）

## 示例 MCP 服务

- **Exa 搜索**：`https://mcp.exa.ai/mcp`（免 API Key 有额度）
- **OpenBB**：见下方「接入 OpenBB」小节
- 其他支持 Streamable HTTP 的 MCP 服务均可按上述格式加入 `servers`

---

## 接入 OpenBB（[OpenBB-finance/OpenBB](https://github.com/OpenBB-finance/OpenBB)）

OpenBB 是金融数据平台（股票、ETF、宏观、衍生品等），通过 **openbb-mcp-server** 暴露 MCP 工具，可与 skill-mcp 对接。

### 步骤

1. **安装并启动 OpenBB MCP 服务**（需 Python 3.9–3.12）：

   ```bash
   pip install "openbb[all]"
   pip install openbb-mcp-server
   openbb-mcp --transport sse --port 8001
   ```

   **重要**：OpenBB 默认的 streamable-http 与 Go MCP 客户端握手时可能返回 400，建议用 **SSE**：`--transport sse`。端点以启动日志 `on http://...` 为准（如 `http://127.0.0.1:8001/mcp`），配置时 URL 须含完整路径。

2. **在 OctoSucker 中配置 skill-mcp**（`agent_config.json`）：

   ```json
   "github.com/OctoSucker/skill-mcp": {
     "servers": [
       { "id": "openbb", "url": "http://127.0.0.1:8001/mcp", "transport": "sse" }
     ]
   }
   ```
   若用 SSE 启动 openbb-mcp，此处必须加 `"transport": "sse"`。

3. 启动 OctoSucker Agent。加载时会对 OpenBB 做 `ListTools`，所有 OpenBB 工具会以 `mcp_openbb_*` 形式注册，供 LLM 调用（如查股价、宏观指标等）。

说明：`openbb-api` 提供的是 REST API（默认 6900 端口），不是 MCP；要经 skill-mcp 接入需使用 **openbb-mcp**（上述命令）。
