# Claude's Public MCP Services

This document catalogs Claude's publicly available MCP (Model Context Protocol) servers and services that integrate interactive UI capabilities and visual content generation.

---

## Available Services

### 1. **Imagine — Visual Creation Suite**

**Endpoint**: `https://sandbox.claudemcpcontent.com/imagine_mcp`

**Type**: HTTP MCP Server

**Status**: ✅ Active (verified Oct 6, 2026)

**Purpose**: Visual content creation and interactive UI generation

**Modules**:
- `diagram` — SVG flowcharts, structural diagrams, system architecture
- `interactive` — interactive explainers with sliders, toggles, and user controls
- `chart` — data visualization, analysis dashboards, geographic maps
- `mockup` — UI/UX design mockups and prototypes
- `data_viz` — complex data analysis and interactive visualizations
- `art` — artistic and creative visual content
- `elicitation` — interactive questionnaires and form-based UI

**How to Use in Claude Desktop**:

Add to `~/.claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "imagine": {
      "url": "https://sandbox.claudemcpcontent.com/imagine_mcp",
      "type": "http"
    }
  }
}
```

**Example Usage**:

In Claude.ai or Claude Desktop, ask:

```
Create an interactive diagram showing the stages of machine learning model training.
Use the interactive module with sliders to control parameters.
```

The Imagine service will:
1. Register available modules and platform support
2. Generate SVG or HTML widget code
3. Claude renders it in an MCP Apps iframe
4. You interact with the visualization

**Architecture**:

The Imagine MCP follows Claude's standard double-iframe pattern:

```
User Request
  ↓
Claude processes & calls Imagine module
  ↓
Imagine returns widget code (SVG/HTML)
  ↓
Claude passes to show_widget tool
  ↓
MCP Apps SDK renders in double-iframe:
  - Outer: src="https://user-origin.claudemcpcontent.com/mcp_apps?..."
  - Inner: srcdoc="<widget from Imagine>"
  ↓
You interact with the visualization
```

**Integration Pattern**:

Like the widget-visualizer-mcp, Imagine provides:
- Tool definitions with module/platform parameters
- Context loading (CSS variables, responsive rules)
- Widget code generation (responsive, constrained)
- MCP Apps `is_mcp_app: true` flag support

**Constraints** (inherited from MCP Apps):
- Widget code size: <2KB (gzipped)
- Technology: Vanilla JavaScript only
- Execution: 100% client-side, no external APIs
- Rendering: Instant feedback (no server roundtrips)

---

## Configuration in Claude Desktop

### Adding Imagine to Your Config

**File**: `~/.claude_desktop_config.json` (or `%APPDATA%\Claude\claude_desktop_config.json` on Windows)

**Complete example**:

```json
{
  "mcpServers": {
    "imagine": {
      "url": "https://sandbox.claudemcpcontent.com/imagine_mcp",
      "type": "http"
    },
    "widget-visualizer": {
      "command": "node",
      "args": ["C:\\Users\\YourName\\src\\widget-visualizer-mcp\\dist\\server.js"]
    }
  }
}
```

### Restarting Claude Desktop

After adding or updating the config:
1. Close Claude Desktop completely
2. Wait 2-3 seconds
3. Reopen Claude Desktop
4. Services should appear in the tools list

---

## How Imagine Differs from widget-visualizer-mcp

| Aspect | Imagine | widget-visualizer-mcp |
|--------|---------|----------------------|
| **Endpoint** | `sandbox.claudemcpcontent.com/imagine_mcp` | Local stdio/http |
| **Hosting** | Claude's managed infrastructure | Your machine |
| **Setup** | Add to config, restart | npm install + build |
| **Modules** | 7 specialized modules (diagram, chart, etc.) | Generic widget rendering |
| **Maintenance** | Claude maintains | You maintain |
| **Customization** | None (Anthropic-managed) | Full control |

**Use widget-visualizer-mcp when**: You want custom widget generation, need full control, or want to extend with domain-specific logic.

**Use Imagine when**: You want out-of-the-box visual content creation without maintaining infrastructure.

---

## Real-World Examples

### Example 1: Machine Learning Training Flow

**Request**:
> "Show me an interactive diagram of the ML training pipeline with sliders for learning rate and batch size. Use the interactive module."

**Imagine generates**: HTML widget with slider inputs, real-time parameter visualization

**Rendering**: MCP Apps iframe displays the interactive visualization

**User interaction**: Adjust sliders → widget updates instantly → parameters shown in table

### Example 2: Financial Dashboard

**Request**:
> "Create a chart showing portfolio performance. Use the chart module with line graphs and a currency selector dropdown."

**Imagine generates**: SVG/HTML with Chart.js or lightweight plotting library

**Rendering**: MCP Apps iframe with responsive layout

**User interaction**: Click dropdown → chart updates for selected currency

### Example 3: Data Analysis UI

**Request**:
> "Build an interactive data visualizer for this CSV data. Include filters for date range and category."

**Imagine generates**: Data-bound HTML widget with filtering logic

**Rendering**: MCP Apps iframe with full interactivity

**User interaction**: Select date range and category → data refilters instantly

---

## Best Practices

### ✅ DO

- **Use modules appropriately**: Match module to content type (chart for data, diagram for structure)
- **Request responsive design**: Ask for "works on desktop and mobile"
- **Specify constraints early**: "Keep it under 1.5KB" or "vanilla JavaScript only"
- **Test with platform**: Ask for desktop OR mobile based on your use case
- **Document the widget**: Ask Imagine to add comments explaining the code

### ❌ DON'T

- **Don't request external APIs**: Imagine widgets execute client-side, no backend calls
- **Don't ask for frameworks**: React, Vue, etc. won't fit in <2KB constraint
- **Don't request real-time updates**: No WebSocket or polling (MCP limitation)
- **Don't assume persistence**: localStorage works, but state resets on iframe reload
- **Don't request nested iframes**: Single-level widget HTML only

---

## Troubleshooting

### "Imagine tools don't appear in Claude"

**Cause**: Config not loaded or service offline

**Fix**:
1. Check `~/.claude_desktop_config.json` syntax (valid JSON)
2. Verify URL is accessible: `curl https://sandbox.claudemcpcontent.com/imagine_mcp`
3. Close and reopen Claude Desktop
4. Check Claude Desktop logs (`%APPDATA%\Claude\logs\`)

### "Widget doesn't render in the iframe"

**Cause**: Widget code is too large, has unsupported syntax, or uses external libraries

**Fix**:
- Ask Imagine to re-generate with explicit constraints: "Vanilla JavaScript only, no CDN libraries, keep it under 1.5KB"
- Check browser console for errors (F12 in Claude Desktop)
- Verify the widget code loads—ask Claude to show you the code before rendering

### "Widget is interactive but not responsive"

**Cause**: Missing CSS viewport or ResizeObserver not implemented

**Fix**:
- Ask Imagine: "Make sure it's responsive and uses ResizeObserver to send size-changed notifications"
- Test with different Claude window sizes

---

## Integration with Your MCP Servers

You can use both Imagine and local MCP servers together:

```json
{
  "mcpServers": {
    "imagine": {
      "url": "https://sandbox.claudemcpcontent.com/imagine_mcp",
      "type": "http"
    },
    "widget-visualizer": {
      "command": "node",
      "args": ["C:\\path\\to\\widget-visualizer-mcp\\dist\\server.js"]
    },
    "my-custom-server": {
      "command": "node",
      "args": ["C:\\path\\to\\my-server\\dist\\index.js"]
    }
  }
}
```

Claude can then:
- Call Imagine for standard visual content
- Call widget-visualizer for custom widgets
- Call my-custom-server for domain-specific tools
- Combine results in a single conversation

---

## Version History

| Date | Status | Details |
|------|--------|---------|
| Oct 6, 2026 | ✅ Verified | Imagine MCP endpoint active and working |
| - | - | Documented module system and integration patterns |

---

## See Also

- [CLAUDE_PROMPTING.md](CLAUDE_PROMPTING.md) - How to prompt for effective widgets
- [EXAMPLES.md](EXAMPLES.md) - Real-world widget code examples
- [REAL_WORLD_ANALYSIS.md](REAL_WORLD_ANALYSIS.md) - Analysis of Claude.ai conversation exports
- [ADVANCED_PATTERNS.md](ADVANCED_PATTERNS.md) - State management, async loading, workflows
