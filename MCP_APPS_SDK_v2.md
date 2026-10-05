# MCP Apps SDK v2 - Implementation Guide

This document covers the **MCP Apps SDK v2 patterns** used in widget-visualizer-mcp, including tool registration, resource serving, dual transports, and lifecycle management.

## SDK v2 Overview

The MCP Apps SDK v2 enables **interactive UI in MCP servers** through:

- **Tool registration** with input/output schemas
- **Resource serving** (HTML, JSON, binary)
- **Dual transport** - HTTP (dev) + Stdio (production)
- **Lifecycle handlers** - Input, result, context changes, teardown
- **Type safety** - Zod schema validation

## Installation

```bash
npm install \
  @modelcontextprotocol/server@^2.0.0 \
  @modelcontextprotocol/client@^2.0.0 \
  @modelcontextprotocol/express@^2.0.0 \
  zod@^4.2.0 \
  express@^4.18.0 \
  cors@^2.8.0
```

Use `--legacy-peer-deps` if needed for beta versions.

## Core Concepts

### 1. MCP Server

The MCP server implements the protocol, manages tools and resources:

```typescript
import { Server } from "@modelcontextprotocol/server";

const server = new Server({
  name: "widget-visualizer",
  version: "1.0.0",
});
```

### 2. Tool Registration

Register tools with schemas using Zod:

```typescript
import { z } from "zod";

const InputSchema = z.object({
  title: z.string(),
  widget_code: z.string(),
  loading_messages: z.array(z.string()).optional(),
});

type ToolInput = z.infer<typeof InputSchema>;

server.registerTool({
  name: "show_widget",
  description: "Display interactive widget",
  inputSchema: InputSchema,
  async handler(input: ToolInput) {
    // Handle tool call
    return {
      type: "text",
      text: JSON.stringify({ success: true })
    };
  }
});
```

### 3. Resource Serving

Serve resources (HTML, JSON, files) to hosts:

```typescript
server.registerResource({
  uri: "widget://visualizer",
  name: "Widget Visualizer",
  description: "Interactive widget UI",
  mimeType: "text/html",
  async readContents() {
    return {
      contents: [{
        uri: "widget://visualizer",
        mimeType: "text/html",
        text: "<html>...</html>"
      }]
    };
  }
});
```

### 4. Transport Selection

Support both HTTP (dev) and Stdio (prod):

```typescript
import { StdioServerTransport } from "@modelcontextprotocol/node";
import { WebSocketServerTransport } from "@modelcontextprotocol/server";

// HTTP mode
if (process.env.HTTP === "true") {
  const app = express();
  app.listen(3000);
}

// Stdio mode
if (process.env.HTTP !== "true") {
  const transport = new StdioServerTransport();
  await server.connect(transport);
}
```

## SDK v2 vs v1 Differences

| Feature | v1 | v2 |
|---------|----|----|
| **Tool registration** | `server.setRequestHandler()` | `server.registerTool()` |
| **Resources** | Via handler | `server.registerResource()` |
| **Schemas** | JSON Schema | Zod (recommended) |
| **Async/await** | Callbacks | Native async |
| **TypeScript** | Partial | Full type safety |
| **Transport** | HTTP only | HTTP + Stdio + WebSocket |

## SDK v2 Implementation in widget-visualizer-mcp

### server.ts Structure

```typescript
// 1. Imports
import { Server } from "@modelcontextprotocol/server";
import { z } from "zod";
import express from "express";

// 2. Define schemas
const ShowWidgetInputSchema = z.object({
  title: z.string().describe("Widget title"),
  widget_code: z.string().describe("HTML/CSS/JS code"),
  loading_messages: z.array(z.string()).optional(),
});

// 3. Create server
const server = new Server({
  name: "widget-visualizer",
  version: "1.0.0",
});

// 4. Register tool
server.registerTool({
  name: "show_widget",
  description: "Display widget",
  inputSchema: ShowWidgetInputSchema,
  async handler(input) {
    // Store widget state
    currentWidget = input;
    
    return {
      type: "text",
      text: JSON.stringify({ success: true })
    };
  }
});

// 5. Register resources
server.registerResource({
  uri: "widget://visualizer",
  name: "Widget UI",
  mimeType: "text/html",
  async readContents() {
    // Inject widget code into template
    const html = fs.readFileSync('widget-ui.html', 'utf-8');
    const injected = html.replace(
      '<!-- WIDGET_CODE_PLACEHOLDER -->',
      `<script>window.__WIDGET_DATA__ = ${JSON.stringify(currentWidget)}</script>`
    );
    
    return {
      contents: [{ uri: "widget://visualizer", mimeType: "text/html", text: injected }]
    };
  }
});

// 6. Start server
async function main() {
  if (process.env.HTTP === "true") {
    // HTTP mode for development
    const app = express();
    app.get("/", (req, res) => res.send(html));
    app.post("/api/widget", (req, res) => {
      currentWidget = req.body;
      res.json({ success: true });
    });
    app.listen(3000);
  } else {
    // Stdio mode for production
    // (Host communicates via stdin/stdout)
  }
}

main().catch(console.error);
```

## Tool Lifecycle

```
1. User Input
   └─→ Claude sends tool invocation

2. Tool Handler
   └─→ server.registerTool() handler executes
   └─→ Validates input with Zod schema
   └─→ Returns { type: "text", text: "..." } or error

3. Resource Request
   └─→ Host requests "widget://visualizer" resource
   └─→ server.registerResource() handler executes
   └─→ Returns HTML with injected widget code

4. Browser Rendering
   └─→ Host renders HTML in sandboxed container
   └─→ CSS variables injected by host
   └─→ JavaScript executes in app context

5. User Interaction
   └─→ All interactions client-side (no server calls)

6. Teardown
   └─→ When widget closed or user navigates away
   └─→ Optional cleanup handlers fire
```

## Schemas & Validation

### Using Zod Schemas

```typescript
// Simple schema
const SimpleInput = z.object({
  name: z.string(),
  age: z.number().int().min(0).max(120),
});

// Complex schema
const ComplexInput = z.object({
  title: z.string().min(1).max(100),
  items: z.array(z.object({
    id: z.string().uuid(),
    value: z.number(),
  })).min(1),
  options: z.enum(["A", "B", "C"]),
  metadata: z.record(z.unknown()).optional(),
});

// Register with schema
server.registerTool({
  name: "process_data",
  inputSchema: ComplexInput,
  async handler(input) {
    // input is type-safe, validated by server
    return { type: "text", text: JSON.stringify(input) };
  }
});
```

### Schema Descriptions

Add `.describe()` for Claude to understand parameters:

```typescript
const WidgetInput = z.object({
  title: z.string()
    .describe("Widget identifier in snake_case"),
  widget_code: z.string()
    .describe("Complete HTML/CSS/JavaScript code for the widget"),
  loading_messages: z.array(z.string())
    .describe("Messages to show during widget rendering"),
});
```

Claude uses these descriptions to generate better widget code.

## Resource Types

### HTML Resource

```typescript
server.registerResource({
  uri: "widget://visualizer",
  mimeType: "text/html",
  async readContents() {
    return {
      contents: [{
        uri: "widget://visualizer",
        mimeType: "text/html",
        text: "<html><body>...</body></html>"
      }]
    };
  }
});
```

### JSON Resource

```typescript
server.registerResource({
  uri: "config://styles",
  mimeType: "application/json",
  async readContents() {
    const data = {
      colors: { primary: "#007AFF" },
      spacing: { sm: 8, md: 16 }
    };
    
    return {
      contents: [{
        uri: "config://styles",
        mimeType: "application/json",
        text: JSON.stringify(data)
      }]
    };
  }
});
```

### Binary Resource

```typescript
server.registerResource({
  uri: "image://icon.png",
  mimeType: "image/png",
  async readContents() {
    const buffer = fs.readFileSync("icon.png");
    
    return {
      contents: [{
        uri: "image://icon.png",
        mimeType: "image/png",
        blob: buffer
      }]
    };
  }
});
```

## Dual Transport Pattern

### HTTP + Stdio Strategy

```typescript
async function main() {
  // HTTP mode: For local development/testing
  if (process.env.HTTP === "true") {
    const app = express();
    app.use(express.json());
    
    app.get("/", (req, res) => {
      res.send(getWidgetUI());
    });
    
    app.listen(process.env.PORT || 3000, () => {
      console.log("HTTP server running");
    });
  }
  
  // Stdio mode: For Claude Desktop / production
  if (process.env.HTTP !== "true") {
    const transport = new StdioServerTransport();
    await server.connect(transport);
  }
}
```

### Configuration

**Development (HTTP):**
```bash
HTTP=true npm run test:http
# Runs at http://localhost:3000
```

**Production (Stdio):**
```bash
npm start
# Communicates via stdin/stdout
```

**Claude Desktop Config (~/.claude_desktop_config.json):**
```json
{
  "mcpServers": {
    "widget-visualizer": {
      "command": "node",
      "args": ["./dist/server.js"]
    }
  }
}
```

## Error Handling

### Tool Error Response

```typescript
server.registerTool({
  name: "process_widget",
  inputSchema: WidgetInputSchema,
  async handler(input) {
    try {
      if (!input.widget_code) {
        return {
          type: "text",
          text: "Error: widget_code is required"
        };
      }
      
      // Process widget
      return {
        type: "text",
        text: JSON.stringify({ success: true })
      };
    } catch (error) {
      return {
        type: "text",
        text: `Error: ${error instanceof Error ? error.message : "Unknown error"}`
      };
    }
  }
});
```

### Validation Errors

Zod automatically validates:

```typescript
server.registerTool({
  name: "calc",
  inputSchema: z.object({
    numbers: z.array(z.number()).min(1),
  }),
  async handler(input) {
    // input.numbers is guaranteed to be number[] with >=1 item
    return { type: "text", text: String(input.numbers.sum()) };
  }
});
```

Invalid input → automatic error response.

## State Management

### In-Memory State

```typescript
let currentWidget: WidgetState | null = null;

server.registerTool({
  name: "show_widget",
  async handler(input) {
    currentWidget = {
      title: input.title,
      code: input.widget_code,
      timestamp: Date.now(),
    };
    
    return { type: "text", text: "Widget stored" };
  }
});

server.registerResource({
  uri: "widget://visualizer",
  async readContents() {
    if (currentWidget) {
      return injectCode(currentWidget.code);
    }
    return defaultUI();
  }
});
```

### Persistent State (localStorage in widget)

```html
<script>
  // In browser context
  const state = JSON.parse(localStorage.getItem('app_state') || '{}');
  
  function save(key, value) {
    state[key] = value;
    localStorage.setItem('app_state', JSON.stringify(state));
  }
  
  // All interactions persist across renders
</script>
```

## Type Safety Throughout

### TypeScript Compilation

```typescript
// Strict mode enabled
{
  "compilerOptions": {
    "strict": true,
    "moduleResolution": "bundler",
    "target": "ES2022"
  }
}
```

### Zod Inference

```typescript
// Define schema
const UserInput = z.object({
  name: z.string(),
  age: z.number(),
});

// Automatically infer type
type User = z.infer<typeof UserInput>;
// Type User = { name: string; age: number; }

// Handler gets typed input
server.registerTool({
  name: "create_user",
  inputSchema: UserInput,
  async handler(input: User) {  // input is typed!
    console.log(input.name, input.age);  // No type errors possible
  }
});
```

## Best Practices

✅ **Always use Zod schemas** - Type safety + validation  
✅ **Add descriptions** - Help Claude generate better code  
✅ **Handle errors gracefully** - Return user-friendly messages  
✅ **Test with HTTP mode first** - Faster iteration  
✅ **Use CSS variables** - Theme-aware widgets  
✅ **Keep state minimal** - Prefer client-side storage  
✅ **Validate early** - Zod catches bad input automatically  
✅ **Document resources** - Explain what each URI provides  

## Common Patterns

### Resource Template Injection

```typescript
const template = fs.readFileSync("template.html", "utf-8");

server.registerResource({
  uri: "app://ui",
  async readContents() {
    const injected = template.replace(
      "<!-- DATA_PLACEHOLDER -->",
      `<script>window.DATA = ${JSON.stringify(currentData)}</script>`
    );
    
    return {
      contents: [{
        uri: "app://ui",
        mimeType: "text/html",
        text: injected
      }]
    };
  }
});
```

### Multi-Resource App

```typescript
server.registerResource({
  uri: "app://index.html",
  async readContents() { /* ... */ }
});

server.registerResource({
  uri: "app://styles.json",
  async readContents() { /* ... */ }
});

server.registerResource({
  uri: "app://data.json",
  async readContents() { /* ... */ }
});
```

### Tool + Resource Interaction

```typescript
server.registerTool({
  name: "update_data",
  inputSchema: UpdateSchema,
  async handler(input) {
    currentData = input;  // Update state
    return { type: "text", text: "Updated" };
  }
});

server.registerResource({
  uri: "app://data",
  async readContents() {
    // Resource reflects latest tool state
    return {
      contents: [{
        uri: "app://data",
        mimeType: "application/json",
        text: JSON.stringify(currentData)
      }]
    };
  }
});
```

## Testing SDK v2 Servers

### Unit Test (Tool Handler)

```typescript
import { describe, it, expect } from "vitest";
import { InputSchema } from "./server";
import { z } from "zod";

describe("Tool Handler", () => {
  it("validates input schema", () => {
    const valid = { title: "test", widget_code: "<div/>" };
    expect(() => InputSchema.parse(valid)).not.toThrow();
  });

  it("rejects invalid input", () => {
    const invalid = { title: "test" }; // missing widget_code
    expect(() => InputSchema.parse(invalid)).toThrow();
  });
});
```

### Integration Test (HTTP)

```bash
# Start server
npm run test:http &

# Test endpoint
curl http://localhost:3000/api/widget -X POST -d '...'

# Run browser tests
npx playwright test
```

## Debugging

### Enable Debug Logging

```typescript
// In server.ts
const DEBUG = process.env.DEBUG === "true";

server.registerTool({
  name: "my_tool",
  async handler(input) {
    if (DEBUG) console.log("Tool input:", input);
    const result = process(input);
    if (DEBUG) console.log("Tool output:", result);
    return result;
  }
});
```

Run with:
```bash
DEBUG=true npm start
```

### DevTools Inspection

```bash
# HTTP mode
npm run test:http
# Open http://localhost:3000
# Press F12 for full DevTools access
```

### Network Inspection

```bash
# Check Stdio communication
DEBUG=true npm start 2>&1 | tee server.log
# Examine server.log for protocol messages
```

## Performance Tips

1. **Cache resources** - Don't regenerate HTML each call
2. **Stream large data** - Use SSE if available
3. **Minimize injected code** - Keep widget HTML small
4. **Use CDN for libs** - Don't bundle heavy libraries
5. **Lazy load** - Load JS on demand in widget

## Security Considerations

⚠️ **Input Validation**
- All tool inputs validated by Zod
- Schema ensures type safety
- Prevents injection attacks

✅ **Sandboxing**
- Widget runs in host's sandbox
- No direct file system access
- Network access via CORS

⚠️ **Output Escaping**
- Ensure HTML is properly escaped
- Sanitize user-generated content
- Validate external resources

## Resources

- [MCP Specification](https://modelcontextprotocol.io)
- [MCP SDK GitHub](https://github.com/modelcontextprotocol/node-sdk)
- [Express.js Docs](https://expressjs.com)
- [Zod Documentation](https://zod.dev)
- [TypeScript Handbook](https://www.typescriptlang.org/docs)

## Example: Complete Tool

```typescript
// 1. Schema
const MyToolInput = z.object({
  data: z.string().min(1),
  format: z.enum(["json", "text"]),
});

// 2. Handler
async function myToolHandler(input: z.infer<typeof MyToolInput>) {
  const processed = input.format === "json"
    ? JSON.parse(input.data)
    : input.data;
  
  return {
    type: "text" as const,
    text: JSON.stringify({ success: true, data: processed })
  };
}

// 3. Registration
server.registerTool({
  name: "my_tool",
  description: "Process data in JSON or text format",
  inputSchema: MyToolInput,
  handler: myToolHandler
});
```

---

**Last Updated:** October 2024  
**MCP SDK Version:** v2  
**Node.js Minimum:** 18 LTS
