# Claude MCP Apps: Interactive Visualization & Widget Generation

Documentation, patterns, and examples for how **Claude generates interactive UI widgets** that users can interact with in real-time.

## How Claude Uses MCP Apps

Claude's MCP Apps model enables interactive visualizations by combining:

1. **Tool** (Claude-callable) — `visualize:show_widget` tool that Claude invokes
2. **Resource** (User-viewable) — HTML/CSS/JavaScript bundle Claude generates and passes as a parameter
3. **Host Rendering** (Claude Desktop) — Host injects CSS variables, renders widget in sandbox, handles events

### Three UI Generation Patterns

Based on ["Beyond Components: Designing Generative UI for MCP Apps"](https://ai.engineer/) (AI Engineer Europe 2026, Ruben Casas):

#### Pattern 1: Static Components
- Developer builds components in advance
- Claude provides **data/properties** only
- Example: Pre-built chart component, Claude chooses dataset
- Trade-off: Constrained, safe, low-token cost

#### Pattern 2: Declarative UI
- Claude generates **JSON/YAML descriptors** (not code)
- Rendering engine translates descriptor → approved components
- Example: `{ type: "chart", data: [...], title: "..." }`
- Trade-off: Balanced flexibility, lower tokens than full code generation

#### Pattern 3: Generative Components (Your Compound Interest Widget)
- Claude generates **complete HTML/CSS/JavaScript at runtime**
- Example: Full widget with sliders, formulas, Chart.js visualization
- Trade-off: Maximum flexibility, requires sandboxing
- **MCP Apps provides the sandbox** — execution is isolated from host

### Claude's Workflow

```
User: "Create an interactive compound interest calculator"
    ↓
Claude analyzes request
    ↓
Claude generates complete HTML/CSS/JS (~2,500 tokens)
    ↓
Claude calls visualize:show_widget(widget_code)  ← Tool invocation
    ↓
MCP Server caches widget_code, returns resource URI
    ↓
Host receives URI, injects CSS variables, renders widget in sandbox
    ↓
Browser executes JavaScript
    ↓
User moves slider → JavaScript updates chart (0 tokens, no server call)
    ↓
User: "Add monthly compounding"
    ↓
Claude regenerates widget with new toggle (~3,200 tokens)
```

**See [sequence diagram](claude-mcp-apps-sequence.mmd) for complete timing breakdown.**

### Why This Is Efficient

| Approach | Cost Per Interaction | Bottleneck |
|----------|-------------------|-----------|
| **Traditional** (API per action) | 500-1000 tokens | Round-trip to Claude |
| **MCP Apps** (write-once) | 0 tokens after initial generation | None (client-side only) |
| **15 slider moves (traditional)** | 7,500-15,000 tokens | Cumulative |
| **15 slider moves (MCP Apps)** | 0 tokens | Browser only |

**Initial generation cost:** ~2,500 tokens
**Per-interaction cost:** 0 tokens (unlimited interactions)
**Feature update cost:** ~3,200 tokens

## MCP Primitives: Tool + Resource + Parameters

Every MCP App uses three core primitives:

### 1. Tool (Server-side, Claude-callable)

```typescript
{
  name: "visualize:show_widget",
  description: "Display an interactive widget in Claude chat",
  inputSchema: {
    type: "object",
    properties: {
      widget_code: {
        type: "string",
        description: "Complete HTML/CSS/JavaScript bundle"
      }
    },
    required: ["widget_code"]
  }
}
```

- **Claude can call this tool** when user asks for interactive visualization
- Tool receives `widget_code` parameter (the full HTML string)
- Tool returns a resource URI (pointer to the widget for host to render)

### 2. Resource (Client-side, host-rendered)

```html
<!-- What Claude generates and sends as widget_code parameter -->
<html>
  <head>
    <style>
      /* Inline CSS, uses host CSS variables */
      body { color: var(--text-primary); background: var(--surface-1); }
      input { accent-color: var(--primary); }
    </style>
  </head>
  <body>
    <h2>Compound Interest Calculator</h2>
    <input type="range" id="principal" min="500" max="10000" value="5000">
    <div id="chart"></div>
    <script>
      // Inline JavaScript, runs in host's sandbox
      document.getElementById('principal').addEventListener('change', (e) => {
        // Recalculate and update chart (0 tokens)
      });
    </script>
  </body>
</html>
```

- **Host renders this widget** in an iframe sandbox
- No external stylesheets or scripts (all inline)
- Host injects CSS variables for theming

### 3. Parameters

When Claude calls the tool:

```json
{
  "tool": "visualize:show_widget",
  "input": {
    "widget_code": "<html>...</html>"  // ← Single large parameter
  }
}
```

The widget_code parameter contains:
- **HTML structure** (headings, sliders, display areas)
- **CSS styling** (inline, using host CSS variables)
- **JavaScript logic** (event handlers, calculations)

**Size:** Typically 2-10KB (≈2,500 tokens for Claude)

---

## Sandboxing & Trust Boundary

### Why MCP Apps Is Safe for Runtime-Generated UI

When Claude generates full HTML/CSS/JavaScript at runtime (Pattern 3), the code must be isolated from the host:

**Without sandboxing:**
- Widget could access user's session data
- Widget could intercept keystrokes
- Widget could inject malicious code

**MCP Apps provides containment:**
- ✅ Double-iframe sandboxing (resource runs in isolated iframe)
- ✅ No same-origin access to host
- ✅ No access to user's Claude session
- ✅ All network requests require explicit CSP configuration
- ✅ Message passing only (app.sendMessage())

**Result:** Runtime-generated UI is as safe as third-party code.

### When to Use Each Pattern

| Pattern | Risk | Token Cost | Use Case |
|---------|------|-----------|----------|
| **Static** | None (pre-approved) | Low | Charts, tables, fixed layouts |
| **Declarative** | Low (constrained by schema) | Medium | Personalized layouts, dynamic data |
| **Generative** | Mitigated by sandbox | Medium | Full interactivity, complex widgets, real-time calcs |

---

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

**Pattern: Generative Components** (Full HTML/CSS/JavaScript generation)

### Initial Version

User prompt: *"Create an interactive compound interest calculator with sliders"*

Claude generates widget with:
- 3 range sliders (principal, rate, years)
- Real-time balance calculation: `balance = principal × (1 + rate)^years`
- Chart.js visualization
- Display cards showing results
- Responsive CSS Grid layout using host design variables

**Token cost:** ~2,500 tokens, one-time

**Why Claude can do this:**
- Claude can generate complete, working HTML/CSS/JavaScript
- No external dependencies needed (except Chart.js)
- Sandbox ensures safety despite runtime code generation
- Host CSS variables auto-theme the widget

### Update: Add Monthly Compounding

User prompt: *"add monthly compounding to the widget pls"*

Claude regenerates entire widget with:
- Toggle buttons (Annually/Monthly)
- Updated formula: `balance = principal × (1 + rate/12)^(12 × years)`
- New chart re-render logic based on toggle state
- All client-side, zero API calls
- Full state management in vanilla JavaScript

**Token cost:** ~3,200 tokens, one-time

User can now toggle compounding mode **unlimited times** with **zero token cost** (pure JavaScript).

## Claude's Design System Integration

When Claude generates widgets, the host automatically provides:

### CSS Variables (Theme-aware)

```css
--text-primary        /* Main text color (light/dark mode aware) */
--text-secondary      /* Secondary/muted text */
--surface-1          /* Background color */
--primary            /* Accent color */
--radius             /* Border radius */
```

**Claude embeds:** `color: var(--text-primary)` instead of hardcoded colors
**Host injects:** CSS variables on widget load based on user's theme

### Fonts

Anthropic fonts via CDN:
- `anthropic-serif` — body text, narrative
- `anthropic-sans` — UI labels, headings

**Result:** Widgets automatically match Claude's interface without any effort.

## References

### Conference Talks

- **"Beyond Components: Designing Generative UI for MCP Apps"** — Ruben Casas (Postman), AI Engineer Europe 2026
  - Explains three UI generation patterns (Static, Declarative, Generative)
  - Discusses trust boundaries and sandboxing requirements
  - Shows Excalidraw MCP App example for human-agent collaboration

### MCP Documentation

- [Model Context Protocol (MCP) Specification](https://modelcontextprotocol.io/)
- [MCP Apps SDK](https://github.com/modelcontextprotocol/ext-apps)

---

- [claude-mcp-apps-architecture.mmd](claude-mcp-apps-architecture.mmd) — High-level flowchart of how Claude uses MCP Apps
- [claude-mcp-apps-sequence.mmd](claude-mcp-apps-sequence.mmd) — Detailed sequence diagram showing user → Claude → MCP → Host workflow
- [examples/compound-interest-widget.html](examples/compound-interest-widget.html) — Complete working widget (annual compounding)
- [examples/compound-interest-monthly.html](examples/compound-interest-monthly.html) — Monthly compounding variant with toggle
- `patterns/` — Reusable patterns and templates
- `docs/` — Detailed architecture and performance documentation

## How to Use This Repository

### Understanding the Model
1. Read this README for **how Claude uses MCP Apps**
2. View [claude-mcp-apps-architecture.mmd](claude-mcp-apps-architecture.mmd) for **high-level overview**
3. Study [claude-mcp-apps-sequence.mmd](claude-mcp-apps-sequence.mmd) for **complete timing breakdown**

### Seeing Examples
- [examples/compound-interest-widget.html](examples/compound-interest-widget.html) — Working widget (annual compounding)
- [examples/compound-interest-monthly.html](examples/compound-interest-monthly.html) — Enhanced widget (with monthly toggle)

Copy an example and ask Claude: *"Can you generate an interactive calculator based on this pattern?"*

---

## Best Practices for Claude MCP Apps

### ✅ Do This
- **Large initial payload** — Send complete HTML/CSS/JS (2-5KB is fine, ≈2,500 tokens)
- **Client-side interactions** — Sliders, buttons, toggles run in JavaScript
- **Real-time calculations** — No server calls during user interaction
- **CSS Grid layouts** — Responsive, no framework needed
- **CSS variables** — Use `var(--text-primary)` instead of hardcoded colors
- **Vanilla JavaScript** — Plain event handlers, no libraries needed
- **Chart.js for visualization** — Pre-approved, safe choice for data viz

### ❌ Don't Do This
- **API calls per interaction** — Each slider move shouldn't call server
- **External CSS frameworks** — No Tailwind, Bootstrap, Material (too large)
- **Framework runtimes** — No React/Vue/Svelte (adds 100KB+)
- **Hardcoded colors** — Always use host CSS variables
- **Animation libraries** — No AOS, Framer Motion, anime.js
- **Complex DOM structures** — Keep it flat and minimal
- **Inline colors/gradients** — Use semantic variables, not hex codes

---

## Summary: Why Claude Uses MCP Apps This Way

1. **Write Once, Interact Unlimited Times**
   - ~2,500 tokens to generate widget
   - 0 tokens per user interaction
   - 50-75% token savings vs. API-per-action model

2. **Three Distinct Patterns**
   - Static Components: Pre-built, data-driven (lowest risk)
   - Declarative UI: JSON descriptors, engine-assembled (balanced)
   - Generative Components: Runtime HTML/CSS/JS, sandboxed (your compound interest widget)

3. **Sandbox Ensures Safety**
   - Runtime-generated code is isolated by iframe
   - No access to user session or host data
   - Same security model as third-party code

4. **Claude's Advantage**
   - Can generate sophisticated UIs that exceed user's request
   - Understands design patterns and best practices
   - Automatically uses host CSS variables and design system
   - Delivers polished widgets users can interact with immediately

MIT
