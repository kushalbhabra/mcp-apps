# MCP Apps: Architecture & Optimization Patterns

Documentation and examples for building interactive UI applications with the Model Context Protocol (MCP) Apps SDK.

## What is MCP Apps?

MCP Apps is a framework for building interactive, client-side UI experiences that run within Claude conversations. It enables:

- **Interactive widgets** displayed directly in Claude chat
- **Real-time interactions** without API round-trips (sliders, buttons, forms)
- **Minimal token overhead** through one-time payload delivery
- **Seamless host integration** with Claude's design system

## Core Concepts

### Tool + Resource Pattern

MCP Apps couples:
1. **Tools** (server-side functions Claude can call)
2. **Resources** (client-side UI/HTML served to the host)

```
User Query
    ↓
Claude generates HTML/JS → calls visualize:show_widget
    ↓
MCP Server returns resource URI
    ↓
Host (Claude UI) renders widget
    ↓
User interaction (sliders, buttons) → runs client-side JavaScript
```

### Why No API Calls for Interactions?

Traditional approach:
- User moves slider (1 call)
- Request sent to server (token cost: ~500)
- Claude recalculates and returns new values (token cost: ~500)
- User moves slider again (repeat)

MCP Apps approach:
- Claude generates full widget once (~2,500 tokens)
- User moves slider (0 tokens)
- JavaScript recalculates locally
- User can interact unlimited times without additional costs

**Result:** 50-75% token savings for interactive-heavy workflows.

## Optimization Patterns

### 1. Code Bloat Prevention

**Constraints by design:**
- No CSS frameworks (Tailwind, Bootstrap, Material)
- Inline styles only (no external CSS files)
- Vanilla JavaScript (no React, Vue, or framework runtimes)
- "Flat philosophy": No gradients, mesh backgrounds, noise textures
- Complexity budget: ≤2 color ramps, ≤5-word subtitles, ≤4 boxes/row

**Result:** Widgets stay 2-10KB HTML/CSS/JS (≈2,500 tokens max)

### 2. Rendering Optimization

**Design decisions:**
- CSS Grid with `minmax(190px, 1fr)` for responsive layouts
- No animation libraries
- Minimal paint operations via flat design
- Zero external JavaScript libraries (except Chart.js for data visualization)

### 3. Token Efficiency

**One-time payload model:**
- Initial widget generation: ~2,500 tokens
- Per interaction after that: 0 tokens
- Update entire widget: ~3,200 tokens (when adding features)

**vs. Traditional tool approach:**
- Each slider move: 500-1000 tokens
- 15 interactions = 7,500-15,000 tokens
- MCP Apps at scale: 2,500 + 3,200 = 5,700 tokens total

### 4. Caching Strategy

**Host-level DOM caching:**
- Widget code parsed once on load
- JavaScript event handlers run client-side indefinitely
- No round-trips during interactions
- Slider changes update DOM directly via JavaScript

## Example: Compound Interest Explorer

### Initial Version

User prompt: *"Create an interactive compound interest calculator with sliders"*

Claude generates widget with:
- 3 range sliders (principal, rate, years)
- Real-time balance calculation: `balance = principal × (1 + rate)^years`
- Chart.js visualization
- Display cards showing results

**Cost:** ~2,500 tokens, one-time

### Update: Add Monthly Compounding

User prompt: *"add monthly compounding to the widget pls"*

Claude regenerates entire widget with:
- Toggle buttons (Annually/Monthly)
- Updated formula: `balance = principal × (1 + rate/12)^(12 × years)`
- New chart re-render logic
- All client-side, zero API calls

**Cost:** ~3,200 tokens, one-time

User can now toggle compounding mode **unlimited times** with zero token cost.

## Design System Integration

### CSS Variables (Host-Injected)

MCP Apps widgets use semantic CSS variables from Claude's design system:

```css
--text-primary        /* Main text color */
--text-secondary      /* Secondary/muted text */
--surface-1          /* Background color */
--radius             /* Border radius */
```

No inline color values needed. Host automatically adapts to light/dark theme.

### Anthropic Fonts

Via CDN from assets.claude.ai:
- `anthropic-serif` — body text, narrative
- `anthropic-sans` — UI labels, headings

## Imagine Visual Creation Suite

Available modules (via `https://sandbox.claudemcpcontent.com/imagine_mcp`):

- **interactive** — User input widgets, forms, sliders, toggles
- **chart** — Data visualization (Chart.js)
- **diagram** — Architecture/flow diagrams
- **mockup** — UI mockups and prototypes
- **art** — Creative visual content

## Key Files in This Repo

- `examples/compound-interest-widget.html` — Complete working widget
- `examples/compound-interest-monthly.html` — Monthly compounding variant
- `patterns/` — Reusable patterns and templates
- `docs/` — Detailed architecture and performance documentation

## Quick Start

### Create an MCP App

1. Register a tool (server-side):
```python
@server.call_tool_handler
async def handle_call_tool(name, arguments):
    if name == "visualize:show_widget":
        widget_code = arguments.get("widget_code")
        # Serve to user
```

2. Generate HTML/JS in Claude and pass to tool:
```javascript
// Claude generates this and sends as parameter
const widget_code = `
  <h2>My Widget</h2>
  <input type="range" id="slider" />
  <script>
    document.getElementById('slider').addEventListener('change', (e) => {
      // Update UI client-side, zero API calls
    });
  </script>
`;
```

## Best Practices

- ✅ One large initial payload (2-5KB)
- ✅ Client-side interactions (sliders, buttons, toggles)
- ✅ Real-time calculations in JavaScript
- ✅ CSS Grid for responsive layout
- ✅ Semantic CSS variables for theming

- ❌ Don't call APIs for every interaction
- ❌ Don't use external CSS frameworks
- ❌ Don't add animation libraries
- ❌ Don't create bloated DOM structures
- ❌ Don't use inline colors (use CSS variables)

## Resources

- [MCP Specification](https://modelcontextprotocol.io/)
- [Imagine Visual Creation Suite Docs](https://sandbox.claudemcpcontent.com/imagine_mcp)
- [Anthropic Design System](https://www.anthropic.com)

## License

MIT
