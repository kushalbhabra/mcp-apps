# MCP Apps: Official Anthropic README

**Source**: https://github.com/modelcontextprotocol/ext-apps

This is the official README from the Anthropic MCP Apps repository. It documents the reference implementation and guidance for building interactive UI extensions for the Model Context Protocol.

---

## MCP Apps

Build interactive UIs for MCP tools — charts, forms, dashboards — that render inline in Claude, ChatGPT and any other compliant chat client.

### Why MCP Apps?

MCP tools return text and structured data. That works for many cases, but not when you need an interactive UI, like a chart, form, design canvas or video player.

MCP Apps provide a standardized way to deliver interactive UIs from MCP servers. Your UI renders inline in the conversation, in context, in any compliant host.

### How It Works

MCP Apps extend the Model Context Protocol by letting tools declare UI resources:

1. **Tool definition** — Your tool declares a `ui://` resource containing its HTML interface
2. **Tool call** — The LLM calls the tool on your server
3. **Host renders** — The host fetches the resource and displays it in a sandboxed iframe
4. **Bidirectional communication** — The host passes tool data to the UI via notifications, and the UI can call other tools through the host

### Getting Started

Requires Node.js 20+. The base MCP SDK packages are `^2.0.0` peers of `ext-apps` (`@modelcontextprotocol/core` is a required peer that `client` already depends on, so npm installs it for you).

For a View or host:

```bash
npm install -S \
  @modelcontextprotocol/ext-apps \
  @modelcontextprotocol/client@^2.0.0 \
  zod@^4.2.0
```

For an MCP server, add the server package (and, for HTTP transports, the Node and Express adapters):

```bash
npm install -S @modelcontextprotocol/ext-apps \
  @modelcontextprotocol/client@^2.0.0 \
  @modelcontextprotocol/server@^2.0.0 \
  @modelcontextprotocol/node@^2.0.0 \
  @modelcontextprotocol/express@^2.0.0 \
  zod@^4.2.0
```

The wire protocol is unchanged between `ext-apps` 1.x and 2.x: a 2.x View works in a 1.x host and a 2.x host renders 1.x Views.

### Using the SDK

The SDK serves three roles: app developers building interactive Views, host developers embedding those Views, and MCP server authors registering tools with UI metadata.

| Package | Purpose | Docs |
|---------|---------|------|
| `@modelcontextprotocol/ext-apps` | Build interactive Views (App class, PostMessageTransport) | [API Docs →](https://apps.extensions.modelcontextprotocol.io/api/modules/app.html) |
| `@modelcontextprotocol/ext-apps/react` | React hooks for Views (useApp, useHostStyles, etc.) | [API Docs →](https://apps.extensions.modelcontextprotocol.io/api/modules/_modelcontextprotocol_ext-apps_react.html) |
| `@modelcontextprotocol/ext-apps/app-bridge` | Embed and communicate with Views in your chat client | [API Docs →](https://apps.extensions.modelcontextprotocol.io/api/modules/app-bridge.html) |
| `@modelcontextprotocol/ext-apps/server` | Register tools and resources on your MCP server | [API Docs →](https://apps.extensions.modelcontextprotocol.io/api/modules/server-helpers.html) |

### Supported Clients

MCP Apps is an extension to the [core MCP specification](https://modelcontextprotocol.io/specification). Host support varies. Currently supported hosts include:

- **Claude Desktop** - Full support with native iframe sandboxing
- **ChatGPT** - Full support with OpenAI Apps SDK
- **VS Code** - MCP extension support for UI resources
- **Goose** - Community MCP client
- **Postman** - API platform with MCP integration
- **MCPJam** - MCP testing platform
- And more...

### Examples

The official repository includes production-ready example servers:

- **Map Server** - Interactive 3D globe viewer using CesiumJS
- **Three.js Server** - Interactive 3D scene renderer
- **ShaderToy Server** - Real-time GLSL shader renderer
- **Sheet Music Server** - ABC notation to sheet music visualization
- **Wiki Explorer** - Wikipedia link graph visualization
- **Cohort Heatmap** - Customer retention heatmap
- **Scenario Modeler** - SaaS business projections
- **Budget Allocator** - Interactive budget allocation
- **Customer Segmentation** - Scatter chart with clustering
- **System Monitor** - Real-time OS metrics
- **Transcript Server** - Live speech transcription
- **Video Resource Server** - Binary video via MCP resources
- **PDF Server** - Interactive PDF viewer with chunked loading
- **QR Code Server** - QR code generator (Python)
- **Text-to-Speech Demo** - Text-to-speech demo (Python)

### Starter Templates

The same app built with different frameworks — pick your favorite:

- React
- Vue
- Svelte
- Preact
- Solid
- Vanilla JS

### Specification

| Version | Status | Link |
|---------|--------|------|
| **2026-01-26** | Stable | [specification/2026-01-26/apps.mdx](https://github.com/modelcontextprotocol/ext-apps/blob/main/specification/2026-01-26/apps.mdx) |
| **draft** | Development | [specification/draft/apps.mdx](https://github.com/modelcontextprotocol/ext-apps/blob/main/specification/draft/apps.mdx) |

### Resources

- [Quickstart Guide](https://apps.extensions.modelcontextprotocol.io/api/documents/quickstart.html)
- [API Documentation](https://apps.extensions.modelcontextprotocol.io/api/)
- [Specification (2026-01-26)](https://github.com/modelcontextprotocol/ext-apps/blob/main/specification/2026-01-26/apps.mdx)
- [SEP-1865 Discussion](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/1865)

### Contributing

Contributions are welcome! Please read [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on how to get started, submit pull requests, and report issues.

---

**Status**: Stable (2026-01-26)  
**Repository**: https://github.com/modelcontextprotocol/ext-apps  
**License**: Apache 2.0
