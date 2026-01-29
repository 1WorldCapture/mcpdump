## 1) 怎么查看一个 MCP Server 具体有哪些 Tools / Resources？

MCP 本质上是 JSON-RPC 协议：**客户端连接并完成 initialize** 后，就能通过几个标准方法把“规格/清单”列出来：

* **Tools 列表**：`tools/list`（支持分页 cursor/nextCursor） ([Model Context Protocol][1])
* **Resources 列表**：`resources/list`（支持分页 cursor/nextCursor） ([Model Context Protocol][2])
* **Resource Templates 列表**：`resources/templates/list`（也支持分页） ([Model Context Protocol][2])
  -（可选）**Prompts 列表**：`prompts/list`（同理）

也就是说：你想“看规格”，本质上就是 **connect → list →（必要时循环分页）**。

### 方式 A：官方 MCP Inspector（最快上手、最直观）

官方有一个交互式调试工具 **MCP Inspector**，直接就能看到 Tools/Resources/Prompts，并能查看 schema、metadata，甚至测试调用。它可以直接用 `npx` 启动： ([Model Context Protocol][3])

示例（检查一个 npm 上的 server 包）：

```bash
npx -y @modelcontextprotocol/inspector npx @modelcontextprotocol/server-filesystem /Users/username/Desktop
```

或者检查你本地自己跑的 server：

```bash
npx @modelcontextprotocol/inspector node path/to/server/index.js args...
```

Inspector 的 Tools/Resources 标签页会把“有哪些 + schema/metadata”完整展示出来。([Model Context Protocol][3])

### 方式 B：写一个最小 MCP Client 自己列举（可自动化、可 Dump）

用官方 TypeScript SDK 的 `Client` 连接后，直接调用：

* `client.listTools()`
* `client.listResources()`
* `client.listResourceTemplates()`
* `client.listPrompts()`

官方文档的 client 示例里就是这么做的（并且用 `mcp.close()` 清理连接）：([Model Context Protocol][4])

---

## 2) 把这些规格 Dump 到文件：思路与方法

你要 Dump 的“规格信息”，通常包含：

* initialize 得到的：

  * `serverInfo`（server 名称/版本等）
  * `serverCapabilities`（是否支持 tools/resources/prompts 等）
* list 得到的：

  * `tools[]`（每个 tool 的 `name/title/description/inputSchema/outputSchema/...`）([Model Context Protocol][1])
  * `resources[]`（每个 resource 的 `uri/name/title/mimeType/...`）([Model Context Protocol][2])
  * `resourceTemplates[]`（每个 template 的 `uriTemplate/...`）([Model Context Protocol][2])
  * `prompts[]`（可选）

关键点：**list 方法都可能分页**（返回 `nextCursor`），所以脚本要循环直到没有 `nextCursor` 为止。([Model Context Protocol][1])

---

## 3) 具体实现脚本（TypeScript）：`dump-mcp.ts`

特点：

* 支持三种 transport：

  * `stdio`：本地起进程（最常用来检查本地 MCP server）
  * `http`：Streamable HTTP（推荐远程部署方式）
  * `sse`：老的 SSE 方式（一些服务还在用）
* 自动处理分页 cursor/nextCursor
* 结果写入 JSON 文件（默认 `mcp-spec.json`）
* 遇到某个能力不支持时不直接崩溃，会把错误记到 `errors` 字段里

> 备注：`StdioClientTransport` 默认会用“安全的默认环境变量集合”，如果你的 server 依赖很多环境变量（API Key 等），可能需要 `--inherit-env` 把 `process.env` 全量传给子进程。SDK 的 stdio transport 确实有“默认继承哪些 env / 如何设置 env”的设计。([jsDelivr][5])
> 对于 HTTP/SSE，脚本支持用 `--header` 设置鉴权 header；Streamable HTTP transport 的参数是 `requestInit`（以及可选 `sessionId`），SSE transport 也有类似的 `requestInit/eventSourceInit`。([jsDelivr][6])

把下面代码保存为 `dump-mcp.ts`：

```ts
#!/usr/bin/env node
/**
 * Dump an MCP server's advertised specs (tools/resources/prompts) to a JSON file.
 *
 * Supports:
 *   - stdio: spawn a local server process and talk over stdin/stdout
 *   - http : Streamable HTTP transport (recommended for remote servers)
 *   - sse  : legacy SSE transport
 *
 * Examples:
 *   # 1) stdio: run a local server command
 *   npx tsx dump-mcp.ts stdio --out dump.json -- node ./build/server.js --flag value
 *
 *   # 2) stdio + pass all env vars to the spawned server
 *   npx tsx dump-mcp.ts stdio --inherit-env --out dump.json -- npx -y @modelcontextprotocol/server-filesystem /Users/me/Desktop
 *
 *   # 3) streamable-http:
 *   npx tsx dump-mcp.ts http --url http://localhost:3000/mcp --out dump.json --header "Authorization: Bearer XXX"
 *
 *   # 4) sse:
 *   npx tsx dump-mcp.ts sse --url http://localhost:3000/sse --out dump.json --header "Authorization: Bearer XXX"
 */

import { mkdir, writeFile } from "node:fs/promises";
import path from "node:path";

import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import {
  StdioClientTransport,
  getDefaultEnvironment,
} from "@modelcontextprotocol/sdk/client/stdio.js";
import { StreamableHTTPClientTransport } from "@modelcontextprotocol/sdk/client/streamableHttp.js";
import { SSEClientTransport } from "@modelcontextprotocol/sdk/client/sse.js";

type TransportKind = "stdio" | "http" | "sse";

type HeadersMap = Record<string, string>;
type EnvMap = Record<string, string>;

type ParsedArgs = {
  transport: TransportKind;
  outFile: string;
  url?: string; // for http/sse
  sessionId?: string; // for streamable http (optional)
  headers: HeadersMap;

  // stdio only
  stdioCommandAndArgs: string[]; // after `--`
  cwd?: string;
  inheritEnv: boolean;
  envOverrides: EnvMap;
};

function printUsageAndExit(code = 1): never {
  const msg = `
Usage:
  # stdio (local):
  dump-mcp.ts stdio [--out FILE] [--cwd DIR] [--inherit-env] [--env KEY=VALUE ...] -- <command> [args...]

  # streamable-http (remote):
  dump-mcp.ts http --url URL [--out FILE] [--header "K: V" ...] [--session-id ID]

  # sse (remote legacy):
  dump-mcp.ts sse --url URL [--out FILE] [--header "K: V" ...]

Options:
  --out, -o           Output JSON file (default: mcp-spec.json)
  --url               MCP endpoint URL (for http/sse)
  --header, -H        HTTP header, repeatable. e.g. -H "Authorization: Bearer XXX"
  --session-id        Streamable HTTP session ID (optional)
  --cwd               Working directory for stdio spawned process
  --inherit-env       Pass ALL current process.env to the stdio server process (unsafe but often needed)
  --env               Extra env override for stdio, repeatable. e.g. --env API_KEY=xxx
  --help, -h          Show help

Examples:
  npx tsx dump-mcp.ts stdio --out dump.json -- node ./build/server.js
  npx tsx dump-mcp.ts stdio --inherit-env -- npx -y @modelcontextprotocol/server-filesystem /Users/me/Desktop
  npx tsx dump-mcp.ts http --url http://localhost:3000/mcp -H "Authorization: Bearer XXX"
`.trim();

  console.error(msg);
  process.exit(code);
}

function isTransportKind(x: string): x is TransportKind {
  return x === "stdio" || x === "http" || x === "sse";
}

function splitKeyValue(input: string): [string, string] {
  // Accept "K: V" OR "K=V"
  const colonIdx = input.indexOf(":");
  const eqIdx = input.indexOf("=");

  let idx = -1;
  let sep = "";

  if (colonIdx !== -1 && (eqIdx === -1 || colonIdx < eqIdx)) {
    idx = colonIdx;
    sep = ":";
  } else if (eqIdx !== -1) {
    idx = eqIdx;
    sep = "=";
  }

  if (idx === -1) {
    throw new Error(`Invalid key/value format: "${input}". Use "K: V" or "K=V".`);
  }

  const key = input.slice(0, idx).trim();
  const value = input.slice(idx + sep.length).trim();

  if (!key) throw new Error(`Invalid key in "${input}"`);
  return [key, value];
}

function envFromProcess(): EnvMap {
  const env: EnvMap = {};
  for (const [k, v] of Object.entries(process.env)) {
    if (typeof v === "string") env[k] = v;
  }
  return env;
}

function parseCli(argv: string[]): ParsedArgs {
  const args = [...argv];

  let transport: TransportKind = "stdio";
  if (args.length > 0 && isTransportKind(args[0])) {
    transport = args.shift() as TransportKind;
  }

  let outFile = "mcp-spec.json";
  let url: string | undefined;
  let sessionId: string | undefined;
  const headers: HeadersMap = {};
  const envOverrides: EnvMap = {};
  let cwd: string | undefined;
  let inheritEnv = false;

  let stdioCommandAndArgs: string[] = [];

  for (let i = 0; i < args.length; i++) {
    const a = args[i];

    if (a === "--") {
      stdioCommandAndArgs = args.slice(i + 1);
      break;
    }

    if (a === "--help" || a === "-h") {
      printUsageAndExit(0);
    }

    if (a === "--out" || a === "-o") {
      outFile = args[++i];
      if (!outFile) printUsageAndExit(1);
      continue;
    }

    if (a === "--url") {
      url = args[++i];
      if (!url) printUsageAndExit(1);
      continue;
    }

    if (a === "--session-id") {
      sessionId = args[++i];
      if (!sessionId) printUsageAndExit(1);
      continue;
    }

    if (a === "--header" || a === "-H") {
      const raw = args[++i];
      if (!raw) printUsageAndExit(1);
      const [k, v] = splitKeyValue(raw);
      headers[k] = v;
      continue;
    }

    if (a === "--cwd") {
      cwd = args[++i];
      if (!cwd) printUsageAndExit(1);
      continue;
    }

    if (a === "--inherit-env") {
      inheritEnv = true;
      continue;
    }

    if (a === "--env") {
      const raw = args[++i];
      if (!raw) printUsageAndExit(1);
      const [k, v] = splitKeyValue(raw);
      envOverrides[k] = v;
      continue;
    }

    // Support shorthand: dump-mcp.ts http https://host/mcp
    if (!a.startsWith("-") && (transport === "http" || transport === "sse") && !url) {
      url = a;
      continue;
    }

    console.error(`Unknown argument: ${a}`);
    printUsageAndExit(1);
  }

  if ((transport === "http" || transport === "sse") && !url) {
    console.error(`Missing --url for transport "${transport}"`);
    printUsageAndExit(1);
  }

  if (transport === "stdio" && stdioCommandAndArgs.length === 0) {
    console.error(`For "stdio", you must provide a command after "--".`);
    printUsageAndExit(1);
  }

  return {
    transport,
    outFile,
    url,
    sessionId,
    headers,
    stdioCommandAndArgs,
    cwd,
    inheritEnv,
    envOverrides,
  };
}

async function paginate<T>(
  fetchPage: (cursor?: string) => Promise<{ items: T[]; nextCursor?: string }>,
  opts?: { maxPages?: number }
): Promise<T[]> {
  const maxPages = opts?.maxPages ?? 10_000;

  const all: T[] = [];
  let cursor: string | undefined = undefined;

  // Guard against buggy servers returning same cursor forever
  const seenCursors = new Set<string>();

  for (let page = 1; page <= maxPages; page++) {
    const { items, nextCursor } = await fetchPage(cursor);
    all.push(...items);

    if (!nextCursor) break;
    if (seenCursors.has(nextCursor)) {
      throw new Error(`Pagination loop detected: nextCursor repeated (${nextCursor})`);
    }
    seenCursors.add(nextCursor);
    cursor = nextCursor;
  }

  return all;
}

async function safe<T>(label: string, fn: () => Promise<T>, errors: Record<string, string>): Promise<T | undefined> {
  try {
    return await fn();
  } catch (e) {
    errors[label] = e instanceof Error ? `${e.name}: ${e.message}` : String(e);
    return undefined;
  }
}

async function main() {
  const parsed = parseCli(process.argv.slice(2));

  const clientInfo = { name: "mcp-spec-dumper", version: "0.1.0" };
  const mcp = new Client(clientInfo);

  let transport:
    | StdioClientTransport
    | StreamableHTTPClientTransport
    | SSEClientTransport;

  if (parsed.transport === "stdio") {
    const [command, ...args] = parsed.stdioCommandAndArgs;

    // Default is "safe env subset" unless you explicitly pass env. See SDK behavior. :contentReference[oaicite:12]{index=12}
    const baseEnv = parsed.inheritEnv ? envFromProcess() : getDefaultEnvironment();
    const env = { ...baseEnv, ...parsed.envOverrides };

    transport = new StdioClientTransport({
      command,
      args,
      cwd: parsed.cwd,
      env,
      // stderr default is "inherit" (SDK default). :contentReference[oaicite:13]{index=13}
    });
  } else if (parsed.transport === "http") {
    const url = new URL(parsed.url!);
    transport = new StreamableHTTPClientTransport(url, {
      requestInit: {
        headers: parsed.headers,
      },
      sessionId: parsed.sessionId,
      // Other options exist (authProvider/reconnection/fetch...). :contentReference[oaicite:14]{index=14}
    });
  } else {
    const url = new URL(parsed.url!);
    transport = new SSEClientTransport(url, {
      requestInit: {
        headers: parsed.headers,
      },
      // For SSE GET stream headers, use eventSourceInit. :contentReference[oaicite:15]{index=15}
      eventSourceInit: {
        headers: parsed.headers,
      } as any,
    });
  }

  const errors: Record<string, string> = {};

  try {
    await mcp.connect(transport);

    const dump: any = {
      dumpedAt: new Date().toISOString(),
      clientInfo,
      serverInfo: mcp.getServerVersion?.() ?? undefined,
      serverCapabilities: mcp.getServerCapabilities?.() ?? undefined,
      transport: {
        kind: parsed.transport,
        url: parsed.url,
      },
      tools: [],
      resources: [],
      resourceTemplates: [],
      prompts: [],
      errors: {},
    };

    const tools = await safe("tools/list", async () => {
      return paginate(async (cursor) => {
        const res = await mcp.listTools(cursor ? { cursor } : undefined);
        return { items: res.tools ?? [], nextCursor: res.nextCursor };
      });
    }, errors);
    if (tools) dump.tools = tools;

    const resources = await safe("resources/list", async () => {
      return paginate(async (cursor) => {
        const res = await mcp.listResources(cursor ? { cursor } : undefined);
        return { items: res.resources ?? [], nextCursor: res.nextCursor };
      });
    }, errors);
    if (resources) dump.resources = resources;

    const resourceTemplates = await safe("resources/templates/list", async () => {
      return paginate(async (cursor) => {
        const res = await mcp.listResourceTemplates(cursor ? { cursor } : undefined);
        return { items: res.resourceTemplates ?? [], nextCursor: res.nextCursor };
      });
    }, errors);
    if (resourceTemplates) dump.resourceTemplates = resourceTemplates;

    const prompts = await safe("prompts/list", async () => {
      return paginate(async (cursor) => {
        const res = await mcp.listPrompts(cursor ? { cursor } : undefined);
        return { items: res.prompts ?? [], nextCursor: res.nextCursor };
      });
    }, errors);
    if (prompts) dump.prompts = prompts;

    dump.errors = errors;

    const outPath = parsed.outFile;
    await mkdir(path.dirname(outPath), { recursive: true }).catch(() => {});
    await writeFile(outPath, JSON.stringify(dump, null, 2), "utf8");

    console.log(`✅ Dump completed: ${outPath}`);
    console.log(
      `   tools=${dump.tools.length}, resources=${dump.resources.length}, templates=${dump.resourceTemplates.length}, prompts=${dump.prompts.length}`
    );
    if (Object.keys(errors).length > 0) {
      console.warn("⚠️ Some list operations failed (see dump.errors):");
      for (const [k, v] of Object.entries(errors)) console.warn(`   - ${k}: ${v}`);
    }
  } finally {
    // Official client examples call mcp.close() for cleanup. :contentReference[oaicite:16]{index=16}
    await mcp.close().catch(() => {});
    await transport.close().catch(() => {});
  }
}

main().catch((e) => {
  console.error("❌ Fatal error:", e);
  process.exitCode = 1;
});
```

---

## 4) 怎么运行

### 安装依赖

```bash
npm init -y
npm i @modelcontextprotocol/sdk zod
npm i -D tsx typescript @types/node
```

> 如果你的项目是 ESM（推荐），记得在 `package.json` 里加上：

```json
{ "type": "module" }
```

### 例子 A：stdio（本地 server）

```bash
npx tsx dump-mcp.ts stdio --out mcp-dump.json -- node ./build/server.js --yourArg 123
```

如果你的 server 需要环境变量（API key 等），用：

```bash
npx tsx dump-mcp.ts stdio --inherit-env --out mcp-dump.json -- node ./build/server.js
```

### 例子 B：Streamable HTTP（远程/本地 http server）

```bash
npx tsx dump-mcp.ts http --url http://localhost:3000/mcp --out mcp-dump.json \
  --header "Authorization: Bearer XXX"
```

### 例子 C：SSE（老式远程）

```bash
npx tsx dump-mcp.ts sse --url http://localhost:3000/sse --out mcp-dump.json \
  --header "Authorization: Bearer XXX"
```

---

## 5) 你 Dump 出来的 JSON 里会有什么

大概结构类似：

```json
{
  "dumpedAt": "...",
  "clientInfo": { "name": "...", "version": "..." },
  "serverInfo": { "name": "...", "version": "..." },
  "serverCapabilities": { "tools": { ... }, "resources": { ... }, ... },
  "tools": [ ... tool definitions with inputSchema/outputSchema ... ],
  "resources": [ ... ],
  "resourceTemplates": [ ... ],
  "prompts": [ ... ],
  "errors": { ... }
}
```

---

[1]: https://modelcontextprotocol.io/specification/2025-06-18/server/tools "https://modelcontextprotocol.io/specification/2025-06-18/server/tools"
[2]: https://modelcontextprotocol.io/specification/2025-06-18/server/resources "https://modelcontextprotocol.io/specification/2025-06-18/server/resources"
[3]: https://modelcontextprotocol.io/docs/tools/inspector "https://modelcontextprotocol.io/docs/tools/inspector"
[4]: https://modelcontextprotocol.io/docs/develop/build-client "https://modelcontextprotocol.io/docs/develop/build-client"
[5]: https://cdn.jsdelivr.net/npm/%40modelcontextprotocol/sdk%401.20.2/dist/esm/client/stdio.d.ts "https://cdn.jsdelivr.net/npm/%40modelcontextprotocol/sdk%401.20.2/dist/esm/client/stdio.d.ts"
[6]: https://cdn.jsdelivr.net/npm/%40modelcontextprotocol/sdk%401.20.2/dist/esm/client/streamableHttp.d.ts "https://cdn.jsdelivr.net/npm/%40modelcontextprotocol/sdk%401.20.2/dist/esm/client/streamableHttp.d.ts"

