# Widget Visualizer MCP App

**Interactive widget renderer** using MCP Apps SDK v2, demonstrating production patterns for interactive UI in MCP hosts.

## 30-Second Start

```bash
npm install
npm run build
npm run test:http  # → http://localhost:3000
```

Deploy to Claude Desktop: Add to `~/.claude_desktop_config.json`

```json
{
  "mcpServers": {
    "widget-visualizer": {
      "command": "node",
      "args": ["C:\\Users\\kusha\\src\\widget-visualizer-mcp\\dist\\server.js"]
    }
  }
}
```

## What It Does

Provides a `show_widget` tool in Claude that renders interactive HTML/CSS/JavaScript widgets. Ask Claude:

> "Create a compound interest calculator widget"

Claude generates widget code → tool renders it → you interact with it.

## Features

✅ **Type-safe** - Zod schemas for inputs/outputs  
✅ **Themed** - Claude Design System CSS variables  
✅ **Testable** - HTTP mode + Playwright integration  
✅ **Local** - No external services required  
✅ **Fast** - Client-side execution, no server roundtrips  

## Scripts

| Command | Purpose |
|---------|---------|
| `npm run build` | Compile + bundle |
| `npm run dev` | Watch mode |
| `npm run test:http` | Start HTTP server |
| `npm start` | Production (Stdio) |

## Documentation

| Doc | Purpose |
|-----|---------|
| **[MODEL_INSTRUCTIONS_SUMMARY.md](MODEL_INSTRUCTIONS_SUMMARY.md)** | ⭐ All HTML generation instructions for models |
| **[README_COMPLETE.md](README_COMPLETE.md)** | Complete technical reference |
| **[MCP_APPS_SDK_v2.md](MCP_APPS_SDK_v2.md)** | SDK patterns & implementation guide |
| **[docs/EXAMPLES.md](docs/EXAMPLES.md)** | Real-world widget examples (5 patterns) |
| **[docs/CLAUDE_PROMPTING.md](docs/CLAUDE_PROMPTING.md)** | How to ask Claude for good widgets |
| **[docs/ADVANCED_PATTERNS.md](docs/ADVANCED_PATTERNS.md)** | State sync, async loading, workflows, etc. |
| **[docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md)** | Debugging, deployment checklist |

## Quick Concepts

### The Tool

```typescript
show_widget({
  title: "my_widget",           // Widget ID
  widget_code: "<html>...</html>", // HTML/CSS/JS
  loading_messages: ["Loading..."] // Optional
})
```

### CSS Variables

```css
--text-primary          /* Text color */
--surface-0            /* Background */
--primary              /* Interactive elements */
--pad-md               /* 16px spacing */
--radius               /* Border radius */
--font-anthropic-sans  /* System font */
```

### Local Testing

```bash
# Terminal 1
npm run test:http

# Terminal 2 (optional)
npx playwright codegen http://localhost:3000
```

Visit http://localhost:3000, type widget code, see it render.

## Project Structure

```
server.ts           # MCP server + HTTP endpoints
widget-ui.html      # Container template
test-playwright.js  # Browser tests
tsconfig.json       # TypeScript settings
vite.config.ts     # Asset bundler
package.json        # Dependencies
dist/               # Build output
```

## Troubleshooting

| Issue | Fix |
|-------|-----|
| "Widget UI not found" | Run `npm run build` |
| Port 3000 in use | `PORT=3001 npm run test:http` |
| TypeScript errors | `rm -r node_modules && npm install` |
| Claude doesn't see tool | Restart Claude, check path in config |

## API (HTTP Mode)

```bash
# GET /          - Serve widget UI
# POST /api/widget - Update widget code
# GET /api/styles  - Get CSS variables
```

Example:
```bash
curl -X POST http://localhost:3000/api/widget \
  -H "Content-Type: application/json" \
  -d '{"title":"calc","widget_code":"<div>2+2=4</div>"}'
```

## Examples

**Clock widget:**
```html
<div id="time" style="font-size: 48px; text-align: center;"></div>
<script>
  setInterval(() => {
    document.getElementById('time').textContent = new Date().toLocaleTimeString();
  }, 1000);
</script>
```

**Chart from CDN:**
```html
<script src="https://cdn.jsdelivr.net/npm/chart.js@4"></script>
<canvas id="chart"></canvas>
<script>
  new Chart(document.getElementById('chart'), {
    type: 'bar',
    data: { labels: ['A','B','C'], datasets: [{data: [10,20,30]}] }
  });
</script>
```

**Form:**
```html
<form style="padding: 20px;">
  <input placeholder="Name" style="padding: 8px; margin-bottom: 10px;" />
  <input placeholder="Email" style="padding: 8px; margin-bottom: 10px;" />
  <button type="submit" style="padding: 10px 20px;">Submit</button>
</form>
```

## Getting Started

### 1. For Quick Start
```bash
npm install && npm run build && npm run test:http
```
Then visit http://localhost:3000

### 2. For Prompting Claude
Start with [docs/CLAUDE_PROMPTING.md](docs/CLAUDE_PROMPTING.md) - learn how to ask Claude for widgets

### 3. For Real Examples
Check [docs/EXAMPLES.md](docs/EXAMPLES.md) - 5 production patterns:
- Compound interest calculator
- Recipe display with scaling
- Real-time stock dashboard
- Todo list with localStorage
- Color palette generator

### 4. For Complex Widgets
See [docs/ADVANCED_PATTERNS.md](docs/ADVANCED_PATTERNS.md):
- State synchronization
- Real-time polling
- Multi-step workflows
- Form validation
- Async data loading
- Keyboard shortcuts

### 5. If Something Breaks
Check [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) first

## Tech Stack

- **Node.js** - Server runtime
- **TypeScript** - Type safety
- **Express.js** - HTTP server
- **Zod** - Input validation
- **Vite** - Asset bundler
- **Playwright** - Browser testing

## Resources

- [MCP Specification](https://modelcontextprotocol.io)
- [MCP Apps SDK Docs](https://modelcontextprotocol.io/docs/apps)
- [Full Documentation](README_COMPLETE.md)

## License

MIT

---

**Questions?** Check [README_COMPLETE.md](README_COMPLETE.md) for detailed documentation.
