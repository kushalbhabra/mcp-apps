# Model Instructions Summary: HTML Generation for MCP Widgets

**Comprehensive dissection of all instructions given to Claude/models for widget HTML generation across the widget-visualizer-mcp project.**

---

## A. CORE TOOL INTERFACE

### Input Parameters
```
show_widget({
  "title": string,              // [REQUIRED] Widget ID/identifier
  "widget_code": string,        // [REQUIRED] Complete HTML/CSS/JavaScript as single string
  "loading_messages": string[]  // [OPTIONAL] Array of status messages during render
})
```

### Required Output Format
```json
{
  "success": boolean,
  "message": string
}
```

### Expected Behavior
- Tool validates input using Zod schema
- Stores widget state in memory
- Returns success/failure response
- Server injects widget code into HTML resource
- Browser executes injected code client-side

---

## B. CONTENT CONSTRAINTS

### 1. Size Limits
- **HARD LIMIT**: Widget code must be <2KB (smaller is better)
- **RECOMMENDED**: <1.5KB for optimal token efficiency
- **GOAL**: Prioritize core functionality; cut unnecessary features
- **STRATEGY**: 
  - Minify CSS (no comments, single-line rules)
  - Remove extra whitespace
  - Use efficient DOM selectors
  - Avoid duplicate code

### 2. Technology Stack
- **ALLOWED**: Vanilla HTML5, CSS3, JavaScript (ES6+)
- **FORBIDDEN**: 
  - External frameworks (React, Vue, Angular)
  - Package managers (require, import for external modules)
  - Backend APIs without error handling
  - External dependencies (must be CDN-only if used)
- **PERMITTED CDN**: 
  - Chart.js (charting)
  - D3.js (visualization)
  - Other lightweight CDN libraries
- **EXECUTION**: 100% client-side, no server roundtrips

### 3. Environment Constraints
- **NO external APIs** without clear fallback/offline support
- **NO real backend calls** without try/catch error handling
- **NO authentication** requirements
- **NO file uploads** to external services
- **NO WebSocket** connections

---

## C. STYLING & THEMING REQUIREMENTS

### CSS Variables (MUST USE, NOT HARDCODE)

#### Text Colors
```css
--text-primary       /* Primary foreground (black in light, white in dark) */
--text-secondary     /* Secondary text (gray) */
```

#### Surface/Background Colors
```css
--surface-0          /* Primary background */
--surface-1          /* Elevated components, cards */
--surface-2          /* Borders, dividers, subtle accents */
```

#### Interactive Elements
```css
--primary            /* Interactive elements (blue) - buttons, links, accents */
```

#### Spacing Scale (8px base)
```css
--pad-sm             /* 8px - small padding */
--pad-md             /* 16px - medium padding */
--pad-lg             /* 24px - large padding */
--pad-xl             /* 32px - extra large padding */

--gap-xs             /* 4px - tiny gap */
--gap-sm             /* 8px - small gap */
--gap-md             /* 16px - medium gap */
--gap-lg             /* 24px - large gap */
--gap-xl             /* 32px - extra large gap */
```

#### Shape/Border
```css
--radius             /* Border radius (4px default) */
```

#### Typography
```css
--font-anthropic-sans    /* Sans-serif system font */
--font-anthropic-serif   /* Serif font (Georgia) */
```

### Styling Rules
- **USE**: `var(--primary)`, `var(--surface-1)`, `var(--text-primary)`, etc.
- **DON'T USE**: Hardcoded hex colors like `#3498db` or `rgb(52, 152, 219)`
- **REASON**: Hardcoded colors break in dark mode; variables ensure theme consistency
- **STRATEGY**: All color usage must reference CSS variables
- **FALLBACK**: If variable unavailable, use neutral color that works in both themes

### Responsive Design Requirements
- **MOBILE FIRST**: Design for mobile (< 400px) first, then scale up
- **BREAKPOINTS**: 
  - Mobile: < 400px
  - Tablet: 400px - 1024px
  - Desktop: > 1024px
- **LAYOUT PATTERNS**:
  - Single column on mobile
  - 2+ columns on desktop
  - Use flexbox/grid for layout
  - No horizontal scroll
- **FONT SIZING**: 
  - Base: 16px
  - Scale proportionally
  - Test readability on small screens

---

## D. HTML STRUCTURE PATTERNS

### Pattern 1: Generative Components (Calculation/Transformation)

**When to use**: Claude generates complete widget code from goal description

**Structure**:
```html
<div style="padding: 20px; max-width: 500px; font-family: var(--font-anthropic-sans);">
  <h2 style="color: var(--text-primary); margin: 0 0 15px 0;">Widget Title</h2>
  
  <!-- INPUT SECTION -->
  <div style="display: flex; flex-direction: column; gap: var(--gap-md);">
    <div>
      <label style="display: block; margin-bottom: 4px; color: var(--text-secondary);">Label</label>
      <input type="[type]" id="input1" value="[default]" 
        style="width: 100%; padding: var(--pad-sm); border: 1px solid var(--surface-2); border-radius: var(--radius);">
    </div>
    <!-- More inputs -->
  </div>
  
  <!-- OUTPUT/RESULT SECTION -->
  <div id="result" style="margin-top: var(--pad-lg); padding: var(--pad-md); background: var(--surface-1); border-radius: var(--radius);">
    <div style="color: var(--text-secondary); font-size: 14px;">Result Label</div>
    <div id="resultValue" style="color: var(--primary); font-size: 28px; font-weight: bold; margin-top: 8px;">$0.00</div>
  </div>
  
  <script>
    function calculate() {
      // Logic here
    }
    
    document.querySelectorAll('input, select').forEach(el => {
      el.addEventListener('input', calculate);
      el.addEventListener('change', calculate);
    });
    
    calculate(); // Initial render
  </script>
</div>
```

**Requirements**:
- ✅ Real-time updates (no buttons)
- ✅ Clear input→output flow
- ✅ Prominent result display
- ✅ Client-side logic only

---

### Pattern 2: Static Components with Data (Templates)

**When to use**: Claude provides template, you provide data structure

**Structure**:
```html
<div style="font-family: var(--font-anthropic-sans); padding: var(--pad-md); max-width: 600px; color: var(--text-primary);">
  <h1 id="title" style="margin: 0 0 10px 0; font-size: 28px;"></h1>
  <p id="description" style="color: var(--text-secondary); margin-bottom: var(--pad-lg);"></p>
  
  <!-- CONTROLS -->
  <div style="display: flex; gap: var(--gap-sm); margin-bottom: var(--pad-lg); align-items: center;">
    <label for="scale" style="color: var(--text-secondary);">Scale:</label>
    <select id="scale" style="padding: 6px 10px; border: 1px solid var(--surface-2); border-radius: var(--radius);">
      <option value="0.5">0.5x</option>
      <option value="1" selected>1x</option>
      <option value="2">2x</option>
    </select>
  </div>
  
  <!-- CONTENT GRID -->
  <div style="display: grid; grid-template-columns: 1fr 1fr; gap: var(--pad-lg);">
    <div>
      <h2 style="font-size: 18px; margin-bottom: var(--gap-md);">Section 1</h2>
      <div id="section1"></div>
    </div>
    <div>
      <h2 style="font-size: 18px; margin-bottom: var(--gap-md);">Section 2</h2>
      <div id="section2"></div>
    </div>
  </div>
  
  <script>
    // Read data from window.__WIDGET_DATA__ (injected by server)
    const data = window.__WIDGET_DATA__ || {};
    
    document.getElementById('title').textContent = data.title;
    
    function render() {
      const scale = parseFloat(document.getElementById('scale').value);
      // Render with scale factor
    }
    
    document.getElementById('scale').addEventListener('change', render);
    render();
  </script>
</div>
```

**Data Access**:
- Read from: `window.__WIDGET_DATA__`
- Format: Any JSON structure
- Access: Direct property access or nested objects
- Fallback: Provide defaults for missing data

**Requirements**:
- ✅ Flexible data binding
- ✅ Dynamic scaling/multipliers
- ✅ Clear data structure in comments
- ✅ No external data fetching

---

### Pattern 3: CDN Library Integration

**When to use**: Need visualization (charts, graphs) with minimal code

**Structure**:
```html
<div style="font-family: var(--font-anthropic-sans); padding: var(--pad-md);">
  <h1 style="color: var(--text-primary); margin-bottom: var(--pad-lg);">Dashboard</h1>
  <div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(250px, 1fr)); gap: var(--gap-md);">
    <div id="chart" style="background: var(--surface-1); padding: var(--pad-md); border-radius: var(--radius);">
      <!-- Chart renders here -->
    </div>
  </div>
  
  <script src="https://cdn.jsdelivr.net/npm/chart.js@4"></script>
  <script>
    // Initialize library
    const ctx = document.getElementById('chart').getContext('2d');
    new Chart(ctx, {
      type: 'line',
      data: { /* ... */ }
    });
  </script>
</div>
```

**CDN Rules**:
- ✅ Use established CDNs (jsdelivr.net, unpkg.com)
- ✅ Pin specific versions (e.g., `@4` or `@2.3.1`)
- ✅ Test CORS compatibility
- ✅ Provide fallback if CDN unavailable
- ❌ Don't use unverified/new CDNs
- **OVERHEAD**: Adding CDN increases widget size; keep rest of code tight

---

### Pattern 4: Client-Side State (localStorage)

**When to use**: Data needs to persist across renders

**Structure**:
```html
<div style="font-family: var(--font-anthropic-sans); padding: var(--pad-md); max-width: 400px;">
  <h1 style="color: var(--text-primary); margin-bottom: var(--pad-lg);">My App</h1>
  
  <!-- INPUT -->
  <div style="display: flex; gap: var(--gap-sm); margin-bottom: var(--pad-lg);">
    <input type="text" id="input" placeholder="Enter..." 
      style="flex: 1; padding: var(--pad-sm); border: 1px solid var(--surface-2); border-radius: var(--radius);">
    <button onclick="add()" style="padding: var(--pad-sm) var(--pad-md); background: var(--primary); color: white; border: none; border-radius: var(--radius); cursor: pointer;">Add</button>
  </div>
  
  <!-- DISPLAY -->
  <div id="list"></div>
  
  <script>
    const STORAGE_KEY = 'app_data';
    
    // Load from localStorage
    let items = JSON.parse(localStorage.getItem(STORAGE_KEY) || '[]');
    
    function save() {
      localStorage.setItem(STORAGE_KEY, JSON.stringify(items));
      render();
    }
    
    function render() {
      document.getElementById('list').innerHTML = items.map((item, i) => `
        <div style="padding: var(--pad-sm); border-bottom: 1px solid var(--surface-2);">
          ${item.text}
          <button onclick="delete(${i})" style="background: none; border: none; color: red; cursor: pointer;">×</button>
        </div>
      `).join('');
    }
    
    function add() {
      const text = document.getElementById('input').value.trim();
      if (text) {
        items.push({ text, done: false });
        document.getElementById('input').value = '';
        save();
      }
    }
    
    render();
  </script>
</div>
```

**localStorage Rules**:
- ✅ Use simple, namespaced keys (e.g., `app_myfeature_data`)
- ✅ Validate data on load (parse errors, missing fields)
- ✅ Provide defaults for missing data
- ❌ DON'T store sensitive data (passwords, tokens)
- ❌ DON'T store large objects (>1MB)
- **PERSISTENCE**: Data survives widget re-renders, survives browser refresh

---

### Pattern 5: Algorithmic Generation

**When to use**: Content/output generated from computation

**Structure**:
```html
<div style="font-family: var(--font-anthropic-sans); padding: var(--pad-md); max-width: 500px;">
  <h1 style="color: var(--text-primary); margin-bottom: var(--pad-lg);">Generator</h1>
  
  <!-- INPUT -->
  <div style="margin-bottom: var(--pad-lg);">
    <label style="display: block; margin-bottom: var(--gap-sm); color: var(--text-secondary);">Input</label>
    <input type="text" id="seed" value="" 
      style="width: 100%; padding: var(--pad-sm); border: 1px solid var(--surface-2); border-radius: var(--radius);">
  </div>
  
  <button onclick="generate()" style="width: 100%; padding: var(--pad-md); background: var(--primary); color: white; border: none; border-radius: var(--radius); cursor: pointer; font-weight: bold;">Generate</button>
  
  <!-- OUTPUT GRID -->
  <div id="output" style="display: grid; grid-template-columns: repeat(auto-fit, minmax(120px, 1fr)); gap: var(--gap-md); margin-top: var(--pad-lg);"></div>
  
  <script>
    function generate() {
      const seed = document.getElementById('seed').value || 'default';
      const results = [];
      
      // Algorithm to generate results
      for (let i = 0; i < 9; i++) {
        results.push(generateItem(seed, i));
      }
      
      document.getElementById('output').innerHTML = results.map(item => `
        <div style="padding: var(--pad-sm); background: var(--surface-1); border-radius: var(--radius); text-align: center; cursor: pointer;" 
          onclick="copyToClipboard('${item}')">
          ${item}
        </div>
      `).join('');
    }
    
    function generateItem(seed, index) {
      // Pure function, no side effects
      return seed + '-' + index;
    }
    
    function copyToClipboard(text) {
      navigator.clipboard.writeText(text);
    }
    
    generate();
  </script>
</div>
```

**Requirements**:
- ✅ Pure functions (no side effects)
- ✅ Deterministic (same input = same output)
- ✅ No API calls for generation
- ✅ Fast computation (<100ms)
- ✅ Copy-to-clipboard UX

---

## E. INTERACTIVE PATTERNS

### Real-Time Updates (No Button)
```javascript
// ✅ CORRECT: Real-time as user types
document.getElementById('input').addEventListener('input', () => {
  updateResult();
});

// ❌ WRONG: Requires user to click button
document.getElementById('button').addEventListener('click', () => {
  updateResult();
});
```

### Form Input Handling
```html
<!-- TEXT INPUT -->
<input type="text" id="name" placeholder="Enter name" value="" />

<!-- NUMBER INPUT -->
<input type="number" id="amount" value="1000" min="0" max="1000000" step="0.01" />

<!-- SELECT/DROPDOWN -->
<select id="option">
  <option value="a">Option A</option>
  <option value="b" selected>Option B</option>
</select>

<!-- CHECKBOX -->
<input type="checkbox" id="agree" onchange="handleChange()" />

<!-- RADIO -->
<input type="radio" name="choice" value="1" />
<input type="radio" name="choice" value="2" />

<!-- COLOR PICKER -->
<input type="color" id="color" value="#3498db" />

<!-- RANGE/SLIDER -->
<input type="range" id="slider" min="0" max="100" value="50" />
```

### Event Listeners
```javascript
// Real-time input
input.addEventListener('input', updateWidget);

// On value change (form submission)
input.addEventListener('change', updateWidget);

// Click events
button.addEventListener('click', handleClick);

// Multiple inputs
document.querySelectorAll('input').forEach(el => {
  el.addEventListener('input', updateWidget);
});
```

### State Updates
```javascript
// Update DOM
document.getElementById('result').textContent = value;
document.getElementById('result').innerHTML = htmlContent;
document.getElementById('result').style.color = 'var(--primary)';

// Update input values
document.getElementById('input').value = newValue;

// Toggle visibility
element.style.display = 'block'; // or 'none'

// Add/remove classes
element.classList.add('active');
element.classList.remove('inactive');
```

---

## F. DATA HANDLING PATTERNS

### Receiving Data from Server
```javascript
// Data injected by server at render time
const data = window.__WIDGET_DATA__ || {};

// Structure example:
// {
//   "title": "My Widget",
//   "items": [{ "name": "Item 1", "value": 100 }],
//   "config": { "theme": "light" }
// }

// Safe access with defaults
const title = data.title || 'Default Title';
const items = data.items || [];
const count = data.count ?? 0; // Nullish coalescing
```

### Data Scaling/Transformation
```javascript
// Scale numeric values
function scale(value, factor) {
  return (value * factor).toFixed(2);
}

// Filter/sort collections
const filtered = items.filter(item => item.active);
const sorted = items.sort((a, b) => a.value - b.value);

// Map transformations
const transformed = items.map(item => ({
  ...item,
  displayValue: item.value * 2
}));
```

### External Data (If Needed)
```javascript
// ✅ With error handling and fallback
async function fetchData() {
  try {
    const response = await fetch('https://api.example.com/data', {
      signal: AbortSignal.timeout(5000)
    });
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return await response.json();
  } catch (error) {
    console.warn('Fetch failed:', error);
    return DEFAULT_DATA; // Fallback
  }
}

// ❌ Without error handling
const data = await fetch(url).then(r => r.json()); // Breaks silently
```

---

## G. LAYOUT PATTERNS

### Flexbox (Preferred)
```css
/* Container */
.container {
  display: flex;
  flex-direction: column; /* or row */
  gap: var(--gap-md);
  align-items: flex-start; /* or center, stretch */
  justify-content: space-between; /* or center, flex-start */
}

/* Item */
.item {
  flex: 1; /* Equal width */
}
```

### Grid (For multi-column)
```css
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: var(--gap-md);
}

/* Responsive */
@media (max-width: 600px) {
  .grid {
    grid-template-columns: 1fr;
  }
}
```

### Stack (Vertical spacing)
```css
.stack {
  display: flex;
  flex-direction: column;
  gap: var(--gap-md);
}
```

### Inline (Horizontal spacing)
```css
.inline {
  display: flex;
  gap: var(--gap-sm);
  align-items: center;
}
```

---

## H. ACCESSIBILITY & UX

### Labels for Inputs
```html
<!-- ✅ CORRECT -->
<label for="email" style="display: block; margin-bottom: 4px;">Email:</label>
<input type="email" id="email" />

<!-- ❌ WRONG -->
<input type="email" />
<span>Email:</span>
```

### Placeholder vs Label
```html
<!-- Use placeholder + label -->
<label for="name">Full Name:</label>
<input type="text" id="name" placeholder="John Doe" />

<!-- Placeholder alone is poor UX (disappears when typing) -->
<input type="text" placeholder="Enter name" /> <!-- ❌ -->
```

### Focus & Keyboard Navigation
```javascript
// Auto-focus on load
document.getElementById('input').focus();

// Keyboard shortcuts
document.addEventListener('keydown', (e) => {
  if (e.key === 'Enter') {
    handleSubmit();
  }
  if (e.key === 'Escape') {
    handleCancel();
  }
});
```

### Loading States
```html
<!-- Show while loading -->
<div id="loading" style="text-align: center; color: var(--text-secondary);">
  ⏳ Loading...
</div>

<!-- Hide when loaded -->
<div id="content" style="display: none;"></div>

<script>
  async function load() {
    document.getElementById('loading').style.display = 'block';
    const data = await fetchData();
    document.getElementById('loading').style.display = 'none';
    document.getElementById('content').style.display = 'block';
    render(data);
  }
</script>
```

### Error Handling
```html
<div id="error" style="display: none; padding: var(--pad-md); background: #FFF3CD; color: #856404; border-radius: var(--radius);">
  ⚠️ <span id="error-message"></span>
</div>

<script>
  try {
    doSomething();
  } catch (error) {
    document.getElementById('error-message').textContent = error.message;
    document.getElementById('error').style.display = 'block';
  }
</script>
```

---

## I. COMMON MISTAKES (DO NOT FOLLOW)

### ❌ Hardcoded Colors
```css
/* WRONG */
button { background-color: #3498db; }
h1 { color: #2c3e50; }

/* RIGHT */
button { background: var(--primary); }
h1 { color: var(--text-primary); }
```

### ❌ Oversized Widgets
```javascript
// WRONG: 10KB+ widget (too large)
// Includes: animation library, unnecessary features, verbose CSS

// RIGHT: <2KB widget (tight, focused)
// Core feature only, minified CSS
```

### ❌ Blocking API Calls
```javascript
// WRONG: No error handling, hangs if API fails
const data = await fetch(url).then(r => r.json());

// RIGHT: Timeout + fallback
async function safeFetch(url) {
  try {
    const response = await fetch(url, {
      signal: AbortSignal.timeout(5000)
    });
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return await response.json();
  } catch {
    return DEFAULT_DATA;
  }
}
```

### ❌ No Responsive Design
```css
/* WRONG */
.container { width: 600px; }  /* Fixed width, breaks on mobile */

/* RIGHT */
.container { 
  max-width: 600px;
  width: 100%;
  padding: var(--pad-md);
}

@media (max-width: 600px) {
  .grid { grid-template-columns: 1fr; }
}
```

### ❌ Using External Frameworks
```html
<!-- WRONG: Requires React (adds 40KB+) -->
<script src="https://cdn.jsdelivr.net/npm/react@18"></script>
<script>ReactDOM.render(...)</script>

<!-- RIGHT: Vanilla JavaScript (already included) -->
<script>document.getElementById('app').innerHTML = template;</script>
```

### ❌ DOM Manipulation in Loops
```javascript
// WRONG: 100 DOM writes
items.forEach(item => {
  document.getElementById('list').appendChild(createElement(item));
});

// RIGHT: 1 DOM write
document.getElementById('list').innerHTML = items.map(renderItem).join('');
```

---

## J. PROMPTING INSTRUCTIONS FOR CLAUDE

### Core Principle
1. **Describe the goal** (what should it do)
2. **Explain the structure** (how should it look)
3. **Specify constraints** (size, technology, styling)
4. **Show examples** (optional but powerful)

### Perfect Prompt Template
```
Create a [WIDGET TYPE] widget that:
- [Main feature]
- [Secondary features]
- [Interaction pattern]

Structure:
- Title: [description]
- Inputs: [input fields]
- Output: [what displays]

Styling:
- Use CSS variables: --primary, --text-primary, --surface-1, --radius
- Keep under 2KB of code
- Make it responsive (works on mobile)

Example interaction: [user does X, widget shows Y]
```

### Pattern-Specific Prompts

**Generative (Calculation)**:
```
Create a [TOOL TYPE] widget that transforms [INPUT] to [OUTPUT].
Compute [algorithm/formula].
Update results in real-time as users modify inputs.
Keep under 1.5KB. Use vanilla JavaScript.
Use CSS variables for theming.
```

**Static + Data**:
```
Create a widget template to display [DATA TYPE] with:
- Title: [from data.field]
- Sections: [describe layout]
- Interactive: [scaling, filtering]

Data format (JSON):
{
  "field1": "type",
  "field2": [{"nested": "structure"}]
}
```

**Algorithmic**:
```
Create a [CONTENT] generator widget that:
- Accepts [input parameters]
- Generates [output type] based on [algorithm]
- Displays/exports result

All processing in-browser.
Cache results in state.
Use vanilla JS, no libraries.
```

### Constraint Phrases to Include
```
"Keep under 2KB of code"
"Use CSS variables, not hardcoded colors"
"No external APIs"
"Responsive design (mobile + desktop)"
"Vanilla JavaScript only"
"Real-time updates (no buttons)"
"No external dependencies"
```

### Issue Diagnosis Phrases
```
If widget is "too large":
  "Remove non-essential features. Keep core functionality only."

If colors look wrong:
  "Always use CSS variables: var(--primary), var(--text-primary), var(--surface-1)"

If no offline support:
  "Add fallback data for when API is unavailable."

If too many features:
  "Reduce scope. This widget should do ONE thing well."

If doesn't work on mobile:
  "Use flexbox/grid. Single column on mobile, multi-column on desktop."
```

---

## K. SPECIFICATION COMPLIANCE

This project follows **MCP Apps SDK v2** specification:
- 📄 [Official Spec](https://github.com/modelcontextprotocol/ext-apps/blob/main/specification/2026-01-26/apps.mdx)

### Compliant Features
✅ Tool-based UI injection (show_widget tool)  
✅ Resource serving (widget://visualizer resource)  
✅ HTML/CSS/JS template injection  
✅ CSS variable theming  
✅ Client-side execution  
✅ Type-safe input validation (Zod)  
✅ Dual transport (HTTP + Stdio)  

---

## L. SUMMARY CHECKLIST

Before generating widget code, Claude should verify:

- [ ] **Goal is clear**: Describe what widget does in 1 sentence
- [ ] **Structure defined**: Inputs, layout, outputs documented
- [ ] **Constraints specified**: Size <2KB, no APIs, responsive
- [ ] **CSS uses variables**: All colors use `var(--primary)`, etc.
- [ ] **Technology vanilla**: No React/Vue/frameworks
- [ ] **Responsive**: Works on mobile (<400px) and desktop
- [ ] **No external dependencies**: Only vanilla JS or jsdelivr CDN
- [ ] **Error handling**: All async calls have try/catch
- [ ] **Real-time updates**: Changes reflected immediately
- [ ] **Accessible**: Labels for inputs, keyboard navigation
- [ ] **Documented**: Comments on complex logic
- [ ] **Example data**: Shows expected input/output format

---

**End of Model Instructions Summary**

This document comprehensively maps all points and instructions given to Claude/models for HTML generation in the widget-visualizer-mcp project.
