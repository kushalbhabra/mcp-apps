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

**What Claude generates and sends as `widget_code` parameter:**

```html
<html>
  <head>
    <style>
      /* CRITICAL: Inline CSS ONLY, uses host CSS variables */
      body { 
        color: var(--text-primary);           /* ← NO hardcoded colors */
        background: var(--surface-1);         /* ← Uses host variables */
        font-family: var(--font-anthropic-sans);
      }
      input { 
        accent-color: var(--primary);
        border-radius: var(--radius);
      }
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

**Key requirements:**
- **No external stylesheets** (all inline)
- **No external scripts** (all inline)
- **CSS variables only** (no hardcoded colors, fonts, sizes)
- **Vanilla JavaScript** (no frameworks)

### 3. Parameters: String → Cache → URI

When Claude calls the tool:

```json
{
  "tool": "visualize:show_widget",
  "input": {
    "widget_code": "<html>...</html>"  // ← Entire widget as string parameter
  }
}
```

**The flow:**
1. Claude generates complete `<html>...</html>` as string (~2-10KB)
2. Claude calls tool with `widget_code` parameter
3. MCP Server **caches** the HTML/CSS/JS
4. Server **generates unique URI** pointing to cached widget
5. Host **creates iframe** pointing to that URI
6. Browser **loads iframe** from CDN
7. **Host injects CSS variables** into iframe document
8. **Browser renders** the widget with injected theming

**Size:** Typically 2-10KB (≈2,500 tokens for Claude)

---

## Widget Delivery & Rendering Pipeline

**This is the complete journey from Claude's code generation to rendered widget:**

### Step 1: Claude Generates Widget Code

Claude creates a complete HTML/CSS/JavaScript string containing:
- HTML structure
- Inline CSS using `var(--...)` for all styling
- Inline JavaScript for event handling

```javascript
// What Claude generates
const widgetCode = `
  <html>
    <head>
      <style>
        body { color: var(--text-primary); background: var(--surface-1); }
        button { background: var(--primary); border-radius: var(--radius); }
      </style>
    </head>
    <body>
      <h2>Calculator</h2>
      <input type="range" id="slider" ...>
      <script>
        document.getElementById('slider').addEventListener('change', ...);
      </script>
    </body>
  </html>
`;
```

### Step 2: Claude Calls Tool with Widget Code

```json
{
  "tool": "visualize:show_widget",
  "input": {
    "widget_code": "<html>...</html>"  // ← Entire string as parameter (~2-10KB)
  }
}
```

**Token cost:** ~2,500 tokens (one-time generation)

### Step 3: MCP Server Caches & Returns URI

```
MCP Server receives: widget_code string
↓
Server action: Store in cache with unique ID (e.g., abc123def)
↓
Server returns: Resource URI
  Example: /resources/widget/abc123def
```

### Step 4: Host Creates Double-Layer Iframe

Host (Claude Desktop) creates this HTML structure:

```html
<!-- Outer container (visible in outer.html) -->
<div id="mcp-app-container-...">
  <!-- IFRAME #1: CDN boundary -->
  <iframe 
    title="visualize: Compound interest explorer"
    sandbox="allow-scripts allow-same-origin allow-forms"
    src="https://85b3a463cdd59fba7003c8dcbb6a4976.claudemcpcontent.com/mcp_apps?...">
  </iframe>
</div>
```

**Why double-layer?**
1. **Outer iframe:** Isolates CDN-served content from host DOM
2. **Inner iframe:** (created by CDN) Adds additional sandbox restrictions

### Step 5: CDN Serves Cached Widget

```
Browser requests: https://85b3a463cdd59fba7003c8dcbb6a4976.claudemcpcontent.com/mcp_apps?...
↓
CDN retrieves: Cached widget HTML/CSS/JS from Step 3
↓
CDN returns: Raw HTML to outer iframe
↓
Outer iframe renders: Which contains inner iframe with widget
```

### Step 6: Host Injects Theming into Iframe

**CRITICAL STEP: This is where theming happens**

Host injects CSS variables into the iframe document:

```javascript
// Host runtime
const cssVariables = {
  '--text-primary': userThemeIsDark ? '#FFFFFF' : '#000000',
  '--surface-1': userThemeIsDark ? '#1A1A1A' : '#FFFFFF',
  '--primary': userThemeIsDark ? '#6699FF' : '#0066FF',
  '--radius': '8px',
  '--font-anthropic-sans': '"anthropic-sans", ui-sans-serif, ...',
  '--font-anthropic-serif': '"anthropic-serif", ui-serif, ...'
};

// Host injects into iframe
iframeDocument.documentElement.style.setProperty('--text-primary', cssVariables['--text-primary']);
iframeDocument.documentElement.style.setProperty('--surface-1', cssVariables['--surface-1']);
// ... etc for all variables

// Also injects @font-face rules via CDN
const fontFace = `
  @font-face {
    font-family: "anthropic-sans";
    src: url("https://assets.claude.ai/Fonts/AnthropicSans-Regular.otf");
  }
  @font-face {
    font-family: "anthropic-serif";
    src: url("https://assets.claude.ai/Fonts/AnthropicSerif-Text-Regular.otf");
  }
`;
iframeDocument.head.innerHTML += `<style>${fontFace}</style>`;
```

### Step 7: Browser Renders Widget with Injected Theme

```
iframe.contentDocument now contains:
  
  <html>
    <head>
      <style>
        :root {
          --text-primary: #FFFFFF;     ← Injected by host
          --surface-1: #1A1A1A;         ← Injected by host
          --primary: #6699FF;           ← Injected by host
          --radius: 8px;                ← Injected by host
          --font-anthropic-sans: "..."; ← Injected by host
          --font-anthropic-serif: "..."; ← Injected by host
        }
        @font-face { ... }             ← Injected by host
      </style>
    </head>
    <body>
      <h2>Calculator</h2>
      <input type="range" id="slider">
      <script>
        // Widget JavaScript runs here, uses host-injected variables
        document.getElementById('slider').addEventListener('change', ...);
      </script>
    </body>
  </html>
```

**Widget CSS now resolves:**
- `color: var(--text-primary)` → `#FFFFFF` (from host injection)
- `background: var(--surface-1)` → `#1A1A1A` (from host injection)
- `font-family: var(--font-anthropic-sans)` → Anthropic Sans from CDN (from host injection)

### Step 8: User Interacts (0 Token Cost)

```
User moves slider → JavaScript event listener fires → DOM updates → Chart re-renders
All client-side, no server calls, no tokens used
```

### Complete Diagram

```
┌──────────────────────────────────────────────────────────────┐
│  Claude Desktop (Host)                                       │
│                                                              │
│  Theme Detection: isDarkMode → true                         │
│  ↓                                                           │
│  CSS Variables Prepared:                                     │
│    --text-primary: #FFFFFF                                  │
│    --surface-1: #1A1A1A                                     │
│    --primary: #6699FF                                       │
│    --radius: 8px                                            │
│    --font-anthropic-*: CDN URLs                             │
│  ↓                                                           │
│  ┌─ IFRAME #1 ──────────────────────────────────────────┐  │
│  │  src="claudemcpcontent.com/mcp_apps?..."             │  │
│  │  sandbox="allow-scripts allow-same-origin..."        │  │
│  │                                                      │  │
│  │  CDN loads cached widget HTML/CSS/JS                │  │
│  │  ↓                                                   │  │
│  │  Host injects CSS variables into document           │  │
│  │  Host injects @font-face rules                      │  │
│  │  ↓                                                   │  │
│  │  ┌─ Browser DOM ─────────────────────────────────┐  │  │
│  │  │  <html>                                       │  │  │
│  │  │    <head>                                     │  │  │
│  │  │      <style>                                  │  │  │
│  │  │        :root {                                │  │  │
│  │  │          --text-primary: #FFFFFF; ✓           │  │  │
│  │  │          --surface-1: #1A1A1A; ✓              │  │  │
│  │  │          --primary: #6699FF; ✓                │  │  │
│  │  │          --font-anthropic-sans: "..."; ✓     │  │  │
│  │  │        }                                       │  │  │
│  │  │        body { color: var(--text-primary); }   │  │  │
│  │  │        button { background: var(--primary); } │  │  │
│  │  │      </style>                                  │  │  │
│  │  │    </head>                                     │  │  │
│  │  │    <body>                                      │  │  │
│  │  │      <input type="range">                      │  │  │
│  │  │      <script>                                  │  │  │
│  │  │        addEventListener(...)  ← User events   │  │  │
│  │  │      </script>                                 │  │  │
│  │  │    </body>                                     │  │  │
│  │  │  </html>                                       │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  │  Rendered Widget:                                    │  │
│  │    • Themed in dark mode (colors from host)          │  │
│  │    • Using Anthropic fonts (from host injection)     │  │
│  │    • Responding to user interactions (JS in iframe)  │  │
│  │    • Zero tokens spent on interactions               │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Theming & Host Integration

### How Host Injection Works

This is where Claude's clever design pattern emerges: **widgets never hardcode colors or fonts**. Instead, the host injects them at runtime.

#### 1. Widget Code (Claude generates)

```html
<style>
  body {
    color: var(--text-primary);           /* ← No hardcoded color */
    background: var(--surface-1);         /* ← No hardcoded bg */
    font-family: var(--font-anthropic-sans);  /* ← No embedded fonts */
  }
  
  button {
    background: var(--primary);           /* ← Semantic, not #0066FF */
    border-radius: var(--radius);         /* ← Flexible spacing */
  }
</style>
```

**Claude generates this once.** Widget is completely theme-agnostic.

#### 2. Host Detects Theme

```javascript
// Claude Desktop runtime
const isDarkMode = systemTheme === 'dark';

const cssVariables = {
  '--text-primary': isDarkMode ? '#FFFFFF' : '#000000',
  '--surface-1': isDarkMode ? '#1A1A1A' : '#FFFFFF',
  '--primary': isDarkMode ? '#6699FF' : '#0066FF',
  '--radius': '8px',
  '--font-anthropic-sans': '"anthropic-sans", ui-sans-serif, ...',
  '--font-anthropic-serif': '"anthropic-serif", ui-serif, ...'
};
```

#### 3. Host Injects at Runtime

When the iframe loads, the host injects CSS variables:

```html
<!-- Host adds this to iframe -->
<style>
  :root {
    --text-primary: #FFFFFF;
    --surface-1: #1A1A1A;
    --primary: #6699FF;
    --radius: 8px;
    --font-anthropic-sans: "anthropic-sans", ui-sans-serif, ...;
    --font-anthropic-serif: "anthropic-serif", ui-serif, ...;
    
    @font-face {
      font-family: "anthropic-serif";
      src: url("https://assets.claude.ai/Fonts/AnthropicSerif-Text-Regular-Static.otf");
    }
    @font-face {
      font-family: "anthropic-sans";
      src: url("https://assets.claude.ai/Fonts/AnthropicSans-Regular.otf");
    }
  }
</style>
```

#### 4. Widget Uses Injected Values

```
Dark mode? Widget automatically renders in dark colors.
Light mode? Host changes variables, widget re-renders.
User switches theme? Host updates variables, widget adapts instantly.
```

**Same widget code. Different themes. Zero work by Claude.**

### Architecture: Host → Sandbox → Widget

```
┌─────────────────────────────────────────────────────────────┐
│                    🖥️  HOST (Claude Desktop)                │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Theme Detection (Light/Dark)                       │   │
│  │  ↓                                                  │   │
│  │  CSS Variables:                                     │   │
│  │  --text-primary: #FFFFFF                            │   │
│  │  --surface-1: #1A1A1A                               │   │
│  │  --primary: #6699FF                                 │   │
│  │  ↓                                                  │   │
│  │  @font-face Injection (CDN Fonts)                   │   │
│  │  ↓                                                  │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │  🔒  SANDBOX (IFrame - Isolated Execution)  │   │   │
│  │  │                                             │   │   │
│  │  │  ┌──────────────────────────────────────┐   │   │   │
│  │  │  │  MCP App Widget                      │   │   │   │
│  │  │  │  (HTML generated by Claude)          │   │   │   │
│  │  │  │                                      │   │   │   │
│  │  │  │  <style>                             │   │   │   │
│  │  │  │    color: var(--text-primary) ✓     │   │   │   │
│  │  │  │    background: var(--surface-1) ✓   │   │   │   │
│  │  │  │    font: var(--font-...) ✓          │   │   │   │
│  │  │  │  </style>                            │   │   │   │
│  │  │  │                                      │   │   │   │
│  │  │  │  User Interactions (0 tokens):       │   │   │   │
│  │  │  │  • Slider moves → JS recalc          │   │   │   │
│  │  │  │  • Buttons → DOM updates             │   │   │   │
│  │  │  │  • Toggles → re-render chart         │   │   │   │
│  │  │  └──────────────────────────────────────┘   │   │   │
│  │  │                                             │   │   │
│  │  │  No same-origin access to host              │   │   │
│  │  │  No session data available                  │   │   │
│  │  │  Message passing only (isolated)            │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  User interactions trigger JavaScript → DOM updates         │
│  Widget automatically uses injected theme variables         │
└─────────────────────────────────────────────────────────────┘
```

**See [component diagram](claude-mcp-apps-components.mmd) for detailed architecture.**

### Why This Pattern Wins

| Aspect | Without Host Injection | With Host Injection |
|--------|----------------------|---------------------|
| **Widget code** | Contains hardcoded colors | Uses CSS variables |
| **Per-theme variants** | Different widget per theme | One widget, infinite themes |
| **Theme switching** | Regenerate widget | Update CSS variables |
| **Token cost** | High (recreate for each theme) | Zero (host handles it) |
| **Flexibility** | Limited to developer's choices | Host controls entire palette |

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

**See ["Theming & Host Integration"](#theming--host-integration) for complete explanation of how the host injects theme variables.**

When Claude generates widgets, it uses a shared design system provided by the host:

### CSS Variables (Semantic, not Hardcoded)

Claude generates widgets that reference these variables — the host injects actual values:

```css
--text-primary        /* Main text color (light/dark mode aware) */
--text-secondary      /* Secondary/muted text */
--surface-1          /* Background color */
--surface-2          /* Elevated surface */
--primary            /* Primary accent (buttons, links) */
--radius             /* Border radius for consistency */
```

**Claude's widget code:**
```css
body { color: var(--text-primary); }
button { background: var(--primary); border-radius: var(--radius); }
```

**Host's injection (light mode):**
```css
--text-primary: #000000;
--surface-1: #FFFFFF;
--primary: #0066FF;
--radius: 8px;
```

**Host's injection (dark mode):**
```css
--text-primary: #FFFFFF;
--surface-1: #1A1A1A;
--primary: #6699FF;
--radius: 8px;
```

**Result:** Same widget code renders perfectly in light and dark modes.

### How Claude Adds Theming (Exact Patterns)

**Claude NEVER hardcodes colors in the widget.** Instead:

#### Pattern 1: CSS Variables in Style Blocks
```html
<style>
  body {
    color: var(--text-primary);        /* ← Host injects value */
    background: var(--surface-0);      /* ← Host injects value */
  }
  input[type="range"] {
    accent-color: var(--text-primary); /* ← Host injects value */
  }
</style>
```

#### Pattern 2: Inline Styles with CSS Variables
```html
<div style="background: var(--surface-1); border-radius: var(--radius); padding: 12px;">
  <div style="font-size: 12px; color: var(--text-secondary);">Label</div>
  <div style="font-size: 20px; font-weight: 600; color: var(--text-primary);">Value</div>
</div>
```

#### Pattern 3: JavaScript Reading CSS Variables
```javascript
// For Chart.js or dynamic styling
chart = new Chart(ctx, {
  type: 'line',
  data: {
    datasets: [
      {
        label: 'Series 1',
        borderColor: 'var(--text-primary)',      // ← CSS var in JS config
        backgroundColor: 'rgba(0, 0, 0, 0.1)'
      },
      {
        label: 'Series 2',
        borderColor: 'var(--text-secondary)',     // ← CSS var in JS config
        backgroundColor: 'rgba(0, 0, 0, 0.05)'
      }
    ]
  }
});
```

#### Pattern 4: Font Family CSS Variables
```html
<style>
  @font-face {
    font-family: anthropic-sans;
    src: url(https://assets.claude.ai/fonts/claude-sans-regular.woff2);
  }
  body {
    font-family: anthropic-serif, serif;        /* Named font */
    /* OR */
    font-family: var(--font-anthropic-sans);    /* CSS variable */
  }
</style>
```

**Result:** Same widget code works in light mode, dark mode, or any theme the host provides. ✅

The host provides two typefaces via CDN:

- `anthropic-serif` — body text, narrative content
- `anthropic-sans` — UI labels, headings, interaction text

**Claude's widget code:**
```css
h1 { font-family: var(--font-anthropic-sans); }
p { font-family: var(--font-anthropic-serif); }
```

**Host injects:**
```css
@font-face {
  font-family: "anthropic-sans";
  src: url("https://assets.claude.ai/Fonts/AnthropicSans-Regular.otf");
}
@font-face {
  font-family: "anthropic-serif";
  src: url("https://assets.claude.ai/Fonts/AnthropicSerif-Text-Regular.otf");
}
```

**Result:** Widgets use Claude's official typography without embedding fonts (~2KB+ savings).

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

### Diagrams

- [claude-mcp-apps-architecture.mmd](claude-mcp-apps-architecture.mmd) — High-level flowchart of how Claude uses MCP Apps
- [claude-mcp-apps-components.mmd](claude-mcp-apps-components.mmd) — Component architecture showing Host, Sandbox, and Widget integration with theming
- [claude-mcp-apps-sequence.mmd](claude-mcp-apps-sequence.mmd) — Detailed sequence diagram showing user → Claude → MCP → Host workflow
- [examples/compound-interest-widget.html](examples/compound-interest-widget.html) — Complete working widget (annual compounding)
- [examples/compound-interest-monthly.html](examples/compound-interest-monthly.html) — Monthly compounding variant with toggle
- `patterns/` — Reusable patterns and templates
- `docs/` — Detailed architecture and performance documentation

## How to Use This Repository

### Understanding the Model
1. Read this README for **how Claude uses MCP Apps**
2. View [claude-mcp-apps-architecture.mmd](claude-mcp-apps-architecture.mmd) for **high-level overview**
3. Study [claude-mcp-apps-components.mmd](claude-mcp-apps-components.mmd) for **Host → Sandbox → Widget architecture**
4. Study [claude-mcp-apps-sequence.mmd](claude-mcp-apps-sequence.mmd) for **complete timing breakdown**
5. Read ["Theming & Host Integration"](#theming--host-integration) for **CSS variable injection details**

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

4. **Host-Injected Theming**
   - Widget never hardcodes colors or fonts
   - Host injects CSS variables based on user's theme
   - Light mode? Dark mode? Same widget, automatic adaptation
   - Zero theme regeneration cost

5. **Claude's Advantage**
   - Can generate sophisticated UIs that exceed user's request
   - Understands design patterns and best practices
   - Automatically uses host CSS variables and design system
   - Delivers polished widgets users can interact with immediately
   - Widgets stay small (2-10KB) and theme-agnostic

MIT
