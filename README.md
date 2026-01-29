# mcpdump

[中文文档](./README.zh-CN.md)

A command-line tool to dump MCP (Model Context Protocol) server specifications to JSON files.

## What it does

mcpdump connects to any MCP server and exports its complete specification including:

- **Tools** - Available functions with input/output schemas
- **Resources** - Static or dynamic data sources
- **Resource Templates** - URI template patterns
- **Prompts** - Pre-defined prompt templates
- **Server Info** - Name, version, and capabilities

## Features

- **Multiple transports**: stdio (local), Streamable HTTP, and SSE
- **Automatic pagination**: Handles servers with large tool/resource lists
- **Error resilient**: Captures partial results even when some operations fail
- **Environment control**: Fine-grained control over env vars passed to spawned processes

## Installation

```bash
git clone https://github.com/1WorldCapture/mcpdump.git
cd mcpdump
npm install
```

### Global Installation (optional)

To use `mcpdump` command from anywhere:

```bash
npm link
```

Then you can run:

```bash
mcpdump stdio -- node ./server.js
mcpdump http --url http://localhost:3000/mcp
mcpdump --help
```

To uninstall: `npm unlink -g mcpdump`

## Usage

### Local server (stdio)

Spawn a local MCP server process and dump its specs:

```bash
# Basic usage
npx tsx dump-mcp.ts stdio -- node ./build/server.js

# With custom output file
npx tsx dump-mcp.ts stdio --out my-dump.json -- node ./build/server.js

# Pass all environment variables to the server
npx tsx dump-mcp.ts stdio --inherit-env -- npx -y @modelcontextprotocol/server-filesystem /path/to/dir

# Pass specific env vars
npx tsx dump-mcp.ts stdio --env API_KEY=xxx -- node ./server.js
```

### Remote server (Streamable HTTP)

Connect to a remote MCP server via HTTP:

```bash
npx tsx dump-mcp.ts http --url http://localhost:3000/mcp

# With authentication
npx tsx dump-mcp.ts http --url http://localhost:3000/mcp \
  --header "Authorization: Bearer YOUR_TOKEN"
```

### Remote server (SSE - legacy)

Connect to a remote MCP server via Server-Sent Events:

```bash
npx tsx dump-mcp.ts sse --url http://localhost:3000/sse \
  --header "Authorization: Bearer YOUR_TOKEN"
```

## Options

| Option | Description |
|--------|-------------|
| `--out, -o` | Output JSON file (default: `dump.json`) |
| `--url` | MCP endpoint URL (required for http/sse) |
| `--header, -H` | HTTP header, repeatable (e.g., `-H "Authorization: Bearer XXX"`) |
| `--session-id` | Streamable HTTP session ID (optional) |
| `--cwd` | Working directory for stdio spawned process |
| `--inherit-env` | Pass all current process.env to the stdio server |
| `--env` | Extra env override for stdio, repeatable (e.g., `--env API_KEY=xxx`) |
| `--help, -h` | Show help message |

## Output Format

The output JSON contains:

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

## Use Cases

- **Documentation**: Auto-generate API docs for your MCP server
- **Testing**: Snapshot server capabilities for regression testing
- **Debugging**: Quickly inspect what tools/resources a server exposes
- **Integration**: Compare capabilities across different MCP servers

## License

ISC
