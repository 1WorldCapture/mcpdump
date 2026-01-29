# mcpdump

[English](./README.md)

一个用于将 MCP (Model Context Protocol) 服务器规格导出为 JSON 文件的命令行工具。

## 功能概述

mcpdump 连接到任意 MCP 服务器，导出其完整规格信息，包括：

- **Tools** - 可用的工具函数及其输入/输出 schema
- **Resources** - 静态或动态数据源
- **Resource Templates** - URI 模板模式
- **Prompts** - 预定义的提示词模板
- **Server Info** - 服务器名称、版本及能力声明

## 特性

- **多种传输方式**：stdio（本地进程）、Streamable HTTP、SSE
- **自动分页**：自动处理大量 tools/resources 的分页查询
- **容错处理**：部分操作失败时仍能捕获已成功的结果
- **环境变量控制**：精细控制传递给子进程的环境变量

## 安装

```bash
git clone https://github.com/user/mcpdump.git
cd mcpdump
npm install
```

## 使用方法

### 本地服务器 (stdio)

启动本地 MCP 服务器进程并导出其规格：

```bash
# 基础用法
npx tsx dump-mcp.ts stdio -- node ./build/server.js

# 指定输出文件
npx tsx dump-mcp.ts stdio --out my-dump.json -- node ./build/server.js

# 继承所有环境变量
npx tsx dump-mcp.ts stdio --inherit-env -- npx -y @modelcontextprotocol/server-filesystem /path/to/dir

# 传递特定环境变量
npx tsx dump-mcp.ts stdio --env API_KEY=xxx -- node ./server.js
```

### 远程服务器 (Streamable HTTP)

通过 HTTP 连接远程 MCP 服务器：

```bash
npx tsx dump-mcp.ts http --url http://localhost:3000/mcp

# 带认证
npx tsx dump-mcp.ts http --url http://localhost:3000/mcp \
  --header "Authorization: Bearer YOUR_TOKEN"
```

### 远程服务器 (SSE - 传统方式)

通过 Server-Sent Events 连接远程 MCP 服务器：

```bash
npx tsx dump-mcp.ts sse --url http://localhost:3000/sse \
  --header "Authorization: Bearer YOUR_TOKEN"
```

## 选项说明

| 选项 | 说明 |
|------|------|
| `--out, -o` | 输出 JSON 文件路径（默认：`dump.json`） |
| `--url` | MCP 端点 URL（http/sse 模式必需） |
| `--header, -H` | HTTP 请求头，可重复使用（如 `-H "Authorization: Bearer XXX"`） |
| `--session-id` | Streamable HTTP 会话 ID（可选） |
| `--cwd` | stdio 子进程的工作目录 |
| `--inherit-env` | 将当前所有环境变量传递给 stdio 子进程 |
| `--env` | 额外的环境变量覆盖，可重复使用（如 `--env API_KEY=xxx`） |

## 输出格式

输出的 JSON 文件结构如下：

```json
{
  "dumpedAt": "2025-01-29T00:00:00.000Z",
  "clientInfo": { "name": "mcp-spec-dumper", "version": "0.1.0" },
  "serverInfo": { "name": "...", "version": "..." },
  "serverCapabilities": { "tools": {}, "resources": {} },
  "transport": { "kind": "stdio", "url": null },
  "tools": [...],
  "resources": [...],
  "resourceTemplates": [...],
  "prompts": [...],
  "errors": {}
}
```

## 应用场景

- **文档生成**：为 MCP 服务器自动生成 API 文档
- **测试验证**：快照服务器能力用于回归测试
- **调试排查**：快速查看服务器暴露的 tools/resources
- **对比分析**：比较不同 MCP 服务器的能力差异

## 许可证

ISC
