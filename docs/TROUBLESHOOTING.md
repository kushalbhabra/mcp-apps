# Troubleshooting & Deployment Checklist

## Troubleshooting Guide

### Build & Compilation Issues

#### ❌ "Cannot find module '@modelcontextprotocol/ext-apps'"

**Cause**: Module not installed or TypeScript can't resolve subpaths

**Fix:**
```bash
# Reinstall with legacy peer deps
npm install --legacy-peer-deps

# Clear cache and rebuild
rm -rf dist/ node_modules/
npm install --legacy-peer-deps
npm run build
```

**Verify:**
```bash
# Check if module exists
ls node_modules/@modelcontextprotocol/
# Should show: ext-apps, server, client, node, etc.
```

---

#### ❌ "TS2554: Expected 4 arguments, got 2"

**Cause**: Function signature mismatch between your code and SDK

**Example Error:**
```
registerAppTool(server, {
  name: "show_widget",
  ...
});
// Error: Expected 4 args but got 2
```

**Fix:** Check function signature in node_modules:
```bash
# View actual function signature
cat node_modules/@modelcontextprotocol/ext-apps/dist/src/server/index.d.ts | grep "registerAppTool"
```

**Then adjust calls to match:**
```typescript
// ✅ Correct signature from SDK v2
registerAppTool(
  server,           // arg 1: server instance
  "show_widget",    // arg 2: tool name
  config,           // arg 3: config object
  callback          // arg 4: implementation callback
);
```

---

#### ❌ "No TypeScript errors in editor, but npm run build fails"

**Cause**: Different TypeScript versions or tsconfig mismatch

**Fix:**
```bash
# Verify TypeScript version
npx tsc --version
# Should be 5.0+

# Check tsconfig
cat tsconfig.json | grep -A5 "compilerOptions"

# If moduleResolution is "node", change to "bundler":
# This enables proper subpath import resolution
```

**Required tsconfig settings:**
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "moduleResolution": "bundler",  // ← Important for @modelcontextprotocol subpaths
    "strict": true,
    "outDir": "./dist",
    "skipLibCheck": true,
    "esModuleInterop": true
  }
}
```

---

### Runtime Issues

#### ❌ "Widget UI not found. Make sure to run: npm run bundle"

**Cause**: `dist/widget-ui.html` doesn't exist

**Fix:**
```bash
# Rebuild everything
npm run build

# Check output
ls -la dist/
# Should show: server.js, widget-ui.html

# If only server.js exists, run bundler:
npm run bundle
```

**Bundler config (vite.config.ts):**
```typescript
import { defineConfig } from 'vite';
import { staticCopy } from 'vite-plugin-static-copy';

export default defineConfig({
  plugins: [
    staticCopy({
      targets: [
        { src: 'widget-ui.html', dest: '.' }
      ]
    })
  ]
});
```

---

#### ❌ "Port 3000 already in use"

**Cause**: Another process using port 3000

**Fix - Option 1: Change port**
```bash
PORT=3001 npm run test:http
# Server runs at http://localhost:3001
```

**Fix - Option 2: Kill existing process**
```powershell
# Windows
netstat -ano | findstr :3000
taskkill /PID <PID> /F

# Mac/Linux
lsof -i :3000
kill -9 <PID>
```

---

#### ❌ "ReferenceError: window is not defined"

**Cause**: Server-side code trying to access browser APIs

**Common mistake:**
```typescript
// ❌ WRONG: This runs on server
const html = fs.readFileSync('widget-ui.html', 'utf-8');
const injected = html.replace('<!-- DATA -->', 
  `<script>window.__DATA__ = {...}</script>`  // ✅ This is OK (generates HTML)
);

// ✅ CORRECT: window access only in <script> tags within HTML
```

**Fix:**
- Keep `window` usage inside `<script>` blocks in HTML
- Server-side code should only generate HTML strings, not execute browser code

---

#### ❌ "Widget renders but styling is wrong (colors, layout broken)"

**Cause**: CSS variables not injected by host

**Fix - Fallback styling:**
```html
<style>
  :root {
    /* Fallback values if host doesn't provide variables */
    --text-primary: #000;
    --text-secondary: #666;
    --surface-0: #fff;
    --surface-1: #f5f5f5;
    --primary: #007AFF;
    --radius: 8px;
    --pad-md: 16px;
  }
</style>
```

**Verify variables are being set:**
```html
<script>
  console.log('CSS Variables:');
  console.log('--primary:', getComputedStyle(document.documentElement).getPropertyValue('--primary'));
  console.log('--text-primary:', getComputedStyle(document.documentElement).getPropertyValue('--text-primary'));
</script>
```

**In Claude Desktop:** CSS variables should be injected automatically by the host

---

### Claude Desktop Issues

#### ❌ "Tool doesn't appear in Claude"

**Cause**: Configuration error in `~/.claude_desktop_config.json`

**Checklist:**
1. File exists at: `C:\Users\<YourUser>\AppData\Roaming\Claude\claude_desktop_config.json`
2. JSON is valid (not missing commas, quotes, etc.)
3. Path points to `dist/server.js` (not `server.ts`)
4. File permissions allow read access

**Verify config:**
```bash
# Show config
cat ~/.claude_desktop_config.json | jq .

# Should look like:
{
  "mcpServers": {
    "widget-visualizer": {
      "command": "node",
      "args": ["C:\\Users\\kusha\\src\\widget-visualizer-mcp\\dist\\server.js"]
    }
  }
}
```

**Test connection:**
```bash
# Start server manually
node dist/server.js
# Should produce output on stdin/stdout connection
```

---

#### ❌ "Claude shows tool but it's greyed out / not callable"

**Cause**: Server crashed or connection lost

**Fix:**
1. Restart Claude Desktop completely
2. Check server logs:
   ```bash
   DEBUG=true node dist/server.js 2>&1 | tee server.log
   ```
3. Look for errors in `server.log`

---

#### ❌ "Tool works but widget doesn't render"

**Cause**: Resource (`widget://visualizer`) not being served correctly

**Diagnosis:**
```bash
# Start in debug mode
DEBUG=true node dist/server.js

# In Claude, call the tool and check:
# 1. Does tool return success?
# 2. Does widget container load?
# 3. Check browser console for errors
```

**Common causes:**
- HTML template not bundled (`npm run bundle` not run)
- Resource URI mismatch (tool says `widget://visualizer`, resource registered as `app://ui`)
- HTML injection failed (check placeholder comment exists)

---

### Widget Code Issues

#### ❌ "Widget code Claude generates has errors"

**Cause**: Incomplete or incorrect prompt

**Fix your prompt:**
```
❌ Vague: "Create a calculator"
✅ Specific: "Create a mortgage calculator with:
   - Input: loan amount, rate %, years
   - Calculate: monthly payment (use formula M = P[r(1+r)^n]/[(1+r)^n-1])
   - Show: large number display
   - Keep under 1KB"
```

---

#### ❌ "Widget renders but doesn't work"

**Check:**
1. **Open DevTools** (right-click → Inspect in Claude Desktop)
2. **Check Console** for JavaScript errors
3. **Look for missing dependencies** (e.g., trying to use Chart.js without CDN link)
4. **Verify inputs are accessible**:
   ```javascript
   console.log(document.getElementById('myInput').value);
   ```

**Common mistakes:**
```html
<!-- ❌ Element ID doesn't match JavaScript -->
<input id="userName">
<script>
  document.getElementById('userName').value;  // OK
  document.getElementById('username').value;  // ERROR
</script>

<!-- ❌ Trying to access window.__DATA__ when not injected -->
<script>
  const data = window.__WIDGET_DATA__ || {};  // Provides fallback
</script>

<!-- ✅ Correct fallback pattern -->
<script>
  const DEFAULT_DATA = { /* ... */ };
  const data = window.__WIDGET_DATA__ || DEFAULT_DATA;
</script>
```

---

#### ❌ "Widget keeps getting cut off / doesn't fit container"

**Cause**: Container has fixed width/height that's too small

**Fix:**
```html
<!-- ❌ WRONG -->
<div style="width: 300px; overflow: hidden;">...</div>

<!-- ✅ CORRECT -->
<div style="max-width: 100%; box-sizing: border-box;">
  <!-- Content can grow/shrink with container -->
</div>
```

**Responsive pattern:**
```html
<div style="
  width: 100%;
  max-width: 500px;
  margin: 0 auto;
  padding: var(--pad-md);
  box-sizing: border-box;
">
  <!-- Content -->
</div>
```

---

## Deployment Checklist

### Pre-Deployment Verification

- [ ] **Builds successfully**
  ```bash
  npm run build
  # No errors or warnings
  ```

- [ ] **Widget-ui.html bundled**
  ```bash
  ls -la dist/widget-ui.html
  # File exists and is > 1KB
  ```

- [ ] **TypeScript compiles cleanly**
  ```bash
  npm run build 2>&1 | grep -i error
  # No output = no errors
  ```

- [ ] **Dependencies installed**
  ```bash
  npm list | head -20
  # All core packages present
  ```

### Local Testing Verification

- [ ] **HTTP server starts**
  ```bash
  npm run test:http
  # Output: "Widget Visualizer HTTP server running at http://localhost:3000"
  ```

- [ ] **Widget UI loads**
  ```bash
  curl http://localhost:3000
  # Returns HTML with <h1>Widget Visualizer</h1>
  ```

- [ ] **Tool endpoint works**
  ```bash
  curl -X POST http://localhost:3000/api/widget \
    -H "Content-Type: application/json" \
    -d '{"title":"test","widget_code":"<div>Test</div>"}'
  # Returns: {"success":true,"title":"test"}
  ```

- [ ] **Example widget renders**
  - Visit http://localhost:3000
  - Paste example widget code
  - Verify it displays correctly

### Claude Desktop Configuration

- [ ] **Config file created**
  ```bash
  cat ~/.claude_desktop_config.json
  # File exists and is valid JSON
  ```

- [ ] **Path is absolute**
  ```json
  {
    "mcpServers": {
      "widget-visualizer": {
        "command": "node",
        "args": ["/absolute/path/to/dist/server.js"]  // ← Must be absolute
      }
    }
  }
  ```

- [ ] **File permissions correct**
  ```bash
  # On Windows: right-click → Properties → check readable
  # On Mac/Linux:
  chmod 644 ~/.claude_desktop_config.json
  ls -la ~/.claude_desktop_config.json | awk '{print $1}'
  # Should show: -rw-r--r--
  ```

- [ ] **Executable is accessible**
  ```bash
  node /path/to/dist/server.js
  # Runs without "file not found" errors
  ```

### Runtime Verification

- [ ] **Server starts without errors**
  ```bash
  DEBUG=true node dist/server.js 2>&1 | head -20
  # No error messages
  ```

- [ ] **Restart Claude and test**
  - Quit Claude Desktop completely
  - Reopen Claude
  - Check that tool appears in tool list
  - Call the tool with test input
  - Verify widget renders

- [ ] **Tool produces correct output**
  ```
  show_widget({
    title: "test",
    widget_code: "<div style='padding: 20px;'>Hello World</div>"
  })
  # Should render text "Hello World" in app
  ```

### Code Quality

- [ ] **No hardcoded paths**
  ```bash
  grep -r "C:\\" src/
  grep -r "/Users/" src/
  # Should return nothing (no absolute paths)
  ```

- [ ] **No console.log spam**
  ```bash
  grep "console.log" server.ts
  # Should only have logging for DEBUG mode
  ```

- [ ] **TypeScript strict mode**
  ```json
  {
    "compilerOptions": {
      "strict": true  // ← Required
    }
  }
  ```

- [ ] **No unused imports**
  ```bash
  npm run build 2>&1 | grep "unused"
  # Should return nothing
  ```

### Performance

- [ ] **Build output size**
  ```bash
  ls -lh dist/server.js
  # Should be < 200KB (uncompressed)
  ```

- [ ] **Startup time**
  ```bash
  time npm run test:http
  # Should start in < 2 seconds
  ```

- [ ] **Widget render time**
  - Widget displays immediately when tool called
  - No noticeable delay with complex widgets
  - Interactions responsive (< 100ms)

### Security Checklist

- [ ] **No secrets in code**
  ```bash
  grep -r "password" src/
  grep -r "api_key" src/
  grep -r "token" src/
  # Should return nothing
  ```

- [ ] **No eval() or dynamic code execution**
  ```bash
  grep "eval" server.ts
  # Should return nothing
  ```

- [ ] **Input validation**
  - All tool inputs validated with Zod
  - HTML escaped properly
  - No injection vulnerabilities

- [ ] **.gitignore configured**
  ```bash
  cat .gitignore | grep -E "(dist|node_modules|\.env)"
  # Should exclude build artifacts and secrets
  ```

### Final Deployment

- [ ] **Code committed and pushed**
  ```bash
  git status
  # Clean working directory
  git log --oneline | head -1
  # Latest commit visible
  ```

- [ ] **Version bumped** (in package.json)
  ```json
  {
    "version": "1.0.0"  // Incremented from previous
  }
  ```

- [ ] **README updated**
  - Deployment instructions current
  - No outdated references
  - Links work

- [ ] **Documented in CHANGELOG**
  ```
  ## [1.0.0] - 2026-10-02
  ### Added
  - Initial production release
  
  ### Changed
  - [list changes]
  
  ### Fixed
  - [list fixes]
  ```

---

## Common Solutions Quick Reference

| Error | Cause | Fix |
|-------|-------|-----|
| Module not found | Not installed | `npm install --legacy-peer-deps` |
| TypeScript TS2554 | Wrong function signature | Check node_modules types, adjust calls |
| Widget UI not found | Not bundled | `npm run bundle` |
| Port in use | Other process using 3000 | `PORT=3001 npm run test:http` |
| Window undefined | Server-side code accessing browser | Move to `<script>` blocks |
| CSS variables wrong | Host not injecting | Add fallback values in `<style>` |
| Tool not in Claude | Config path wrong | Use absolute path in `claude_desktop_config.json` |
| Tool greyed out | Server crashed | Check DEBUG logs, restart Claude |
| Widget doesn't render | Resource not found | Verify URI match, rebuild with `npm run bundle` |

---

## Getting Help

1. **Check recent changes**: `git diff HEAD~1`
2. **View server logs**: `DEBUG=true npm run test:http 2>&1 | tee debug.log`
3. **Test CLI directly**: `node dist/server.js`
4. **Verify TypeScript**: `npx tsc --noEmit`
5. **Read MCP_APPS_SDK_v2.md** for patterns

