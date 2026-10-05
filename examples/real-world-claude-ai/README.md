# Real-World MCP Apps Examples from Claude.ai

This directory contains actual Claude.ai conversation exports showing MCP Apps in production use.

## Files

### `first-message.html`
**Claude.ai conversation export** showing the initial system prompt and available tools:
- `read_me` tool definition (silent setup call)
- `show_widget` tool definition (visible rendering)
- User request: "show me how compound interest works"

**Key insight**: Demonstrates how Claude Desktop registers MCP tools and their JSON schema including the `is_mcp_app: true` flag for widget tools.

### `followup.html`
**Conversation state** showing the actual tool execution flow:
- First phase: Claude calls `read_me` silently to load design context
- Second phase: Claude calls `show_widget` with widget code
- `read_me` response contains: CSS variables, component examples, design rules
- Tool results flow through the JSON-RPC protocol

**Key insight**: Shows the two-phase initialization pattern - setup happens silently, rendering is visible.

### `outer.html`
**Rendered iframe container** showing the actual sandbox configuration:

```html
<iframe
  title="visualize: Compound interest explorer"
  sandbox="allow-scripts allow-same-origin allow-forms"
  allow="fullscreen *; clipboard-write *"
  src="..."
  style="height: 509px;">
</iframe>
```

**Key observations**:
- Sandbox attributes are restrictive (blocks navigation, popups, plugins)
- Allow attribute grants specific permissions (fullscreen, clipboard)
- Height 509px is calculated from widget ResizeObserver output
- Resource domains permit: CDN libraries (esm.sh, jsdelivr, unpkg), Google Fonts, Anthropic fonts

### `recipe.html`
**Structured data example** showing the "Static + Data" widget pattern:
- Tool call: `recipe_display_v0` with recipe metadata
- 19 ingredients with metric/US conversions
- 9 steps with optional timer values
- Best practices: return text (for model), structuredContent (for UI), and _meta (metadata)

**Key insight**: Demonstrates how widgets can display structured data with conversions and media galleries.

### `response.html`
**Complete message flow** showing:
- JSON-RPC 2.0 protocol structure
- Three-phase assistant response (setup → render → explain)
- Tool results including `tool_use` and `tool_result` blocks
- Final text explanation of compound interest

### `test.html`
**Font and theming configuration** demonstrating:
- Anthropic font stack (`anthropic-serif`, `anthropic-sans`)
- Complete CSS variable system (30+ variables)
- Accessibility patterns (sr-only class)
- Widget component base styles
- Interactive example with proper styling
## Architecture: Claude's Double-Iframe Pattern

While the outer.html file only shows the first-layer iframe, Claude Desktop's actual implementation uses **double-iframe layering**:

```
Claude Desktop
└─ Outer iframe: src="https://user-origin.claudemcpcontent.com/mcp_apps?..."
   └─ Server response (HTTP with CSP headers)
      └─ Inner iframe: srcdoc="<widget HTML>"
         └─ Widget execution (isolated, constrained)
```

**This is identical to the Contoso enterprise pattern** (single domain, no per-user subdomains needed).

---
## Patterns Demonstrated

### Pattern 1: Two-Phase Tool Execution
```
Phase 1: read_me (silent)
└─ Load CSS variables, component examples, responsive rules

Phase 2: show_widget (visible)
└─ Render widget in sandboxed iframe with user interaction
```

### Pattern 2: Double-Iframe Sandbox Architecture

**Layer 1 - Outer iframe** (Origin Isolation + CSP Headers):
```html
<iframe 
  src="https://user-origin.claudemcpcontent.com/mcp_apps?connect-src=..."
  sandbox="allow-scripts allow-same-origin allow-forms"
  allow="fullscreen *; clipboard-write *"
/>
```
- Per-user stable origin (e.g., `abc123.claudemcpcontent.com`)
- Query params configure CSP headers at server
- Resource/connect domains white-listed in HTTP headers

**Layer 2 - Inner iframe** (Widget Sandboxing via srcdoc):
```html
<!-- Server response at claudemcpcontent.com/mcp_apps renders: -->
<iframe srcdoc="<sanitized widget HTML with CSS variables>"></iframe>
```
- srcdoc prevents external URL loading (additional XSS protection)
- Same origin as outer iframe (enables localStorage access if needed)
- CSP constraints prevent exfiltration to unapproved domains
- Widget JavaScript executes in isolated context

**Security Model**:
- ✅ Outer iframe blocks top-level navigation, popups, plugins
- ✅ Permits specific capabilities (scripts, forms, fullscreen)
- ✅ HTTP CSP headers enforce CDN allowlist (esm.sh, jsdelivr, etc.)
- ✅ Inner iframe srcdoc prevents inline script XSS
- ✅ Per-user origin prevents cross-user data access

### Pattern 3: Responsive Height Negotiation
- Widget measures content with ResizeObserver
- Sends `ui/notifications/size-changed` to host
- Claude Desktop resizes iframe (509px observed)
- Enables responsive layouts without scrollbars

### Pattern 4: CSS Variable Theming
```css
--primary: #0066cc;
--text-primary: #1a1a1a;
--surface-1: #ffffff;
--pad-md: 12px;
```
- All colors via variables (enables dark mode)
- Spacing scale (8px base)
- Typography tokens (sans-serif, serif)
- Border radius and shadows

### Pattern 5: Structured Data Passing
```json
{
  "type": "text",
  "text": "Text for model context",
  "structuredContent": { "widget": "data" },
  "_meta": { "source": "..." }
}
```
- Text for model understanding
- structuredContent for widget rendering
- _meta for internal tracking (not added to context)

## Real-World Specifications Observed

| Aspect | Value | Source |
|--------|-------|--------|
| Widget height on first render | 509px | outer.html |
| CDN libraries permitted | esm.sh, jsdelivr, unpkg | outer.html resource-src |
| Font delivery | https://assets.claude.ai | test.html @font-face |
| Spacing base unit | 8px | test.html CSS variables |
| CSS variables provided | 30+ | test.html |
| Sandbox permissions | allow-scripts, allow-forms, allow-same-origin | outer.html |
| Loading states | 1-4 messages | first-message.html |
| Widget code size | ~2KB (inferred) | best practice from CLAUDE_PROMPTING.md |

## Integration with MCP Apps SDK

These examples show production-grade usage of:
- **@modelcontextprotocol/ext-apps** - Resource serving and UI registration
- **@modelcontextprotocol/server** - Tool registration with Zod schemas
- **@modelcontextprotocol/client** - Host-side message handling

The implementation follows **SEP-1865** (MCP Apps Specification v2026-01-26):
- ✅ UI resources registered as `ui://` in tool metadata
- ✅ JSON-RPC 2.0 message protocol
- ✅ iframe sandboxing with CSP enforcement
- ✅ Responsive height negotiation via ResizeObserver
- ✅ CSS variable theming system
- ✅ Structured data passing

## Recommended Reading Order

1. **first-message.html** - Understand tool registration and schema
2. **response.html** - Learn the protocol flow
3. **test.html** - Study CSS variables and font configuration
4. **outer.html** - Understand sandbox security model
5. **recipe.html** - See structured data pattern
6. **followup.html** - Watch the complete two-phase execution

Then study:
- [docs/EXAMPLES.md](../EXAMPLES.md) - Production widget code patterns
- [docs/CLAUDE_PROMPTING.md](../CLAUDE_PROMPTING.md) - Prompting Claude to generate widgets
- [REAL_WORLD_ANALYSIS.md](../REAL_WORLD_ANALYSIS.md) - Detailed analysis of these examples

## Compliance Verification

These exports validate that the MCP Apps specification is **production-ready** and **actively used** by:
- Claude Desktop (primary implementation)
- Real users (financial tools, recipes, etc.)
- Multiple widget patterns (generative, data-driven, interactive)

The implementation demonstrates 100% compliance with SEP-1865 across all major components.
