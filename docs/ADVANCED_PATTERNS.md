# Advanced Widget Patterns

For developers building sophisticated interactive widgets beyond basic templates.

---

## Pattern 1: Widget State Synchronization

**Use Case**: Keep widget state in sync between multiple tool calls

### Challenge
User calls tool multiple times. Each call regenerates widget HTML. How do you preserve user's previous interactions?

### Solution: localStorage + Window Data

```typescript
server.registerTool({
  name: "show_widget",
  inputSchema: ShowWidgetInputSchema,
  async handler(input) {
    currentWidget = {
      title: input.title,
      code: input.widget_code,
      timestamp: Date.now(),
      // Store initialization data
      initialState: input.initial_state || {}
    };
    
    return {
      content: [{
        type: "text",
        text: JSON.stringify({ 
          success: true,
          preservedState: true  // Signal to use localStorage
        })
      }]
    };
  }
});
```

**Widget HTML:**
```html
<script>
  // Initialize from localStorage (preserves across renders)
  const savedState = JSON.parse(localStorage.getItem('widget_state') || '{}');
  const initialData = window.__WIDGET_DATA__?.initialState || {};
  const state = { ...initialData, ...savedState };
  
  // When user makes changes
  function saveState(key, value) {
    state[key] = value;
    localStorage.setItem('widget_state', JSON.stringify(state));
    render();
  }
  
  // When widget regenerates, it reloads this state
  function render() {
    // Use 'state' to restore user's previous inputs
    document.getElementById('input1').value = state.input1 || '';
  }
  
  render();
</script>
```

**Benefits:**
- ✅ Survives widget regeneration
- ✅ Works across multiple tool calls
- ✅ No backend storage needed
- ✅ User sees their previous work

---

## Pattern 2: Real-Time Data Updates via Polling

**Use Case**: Widget displays data that changes, wants to refresh periodically

### Challenge
Widget code is generated once. How to refresh data without regenerating?

### Solution: Embedded Update Endpoint

```html
<div id="content" style="padding: 20px;">
  <h1>Live Data Dashboard</h1>
  <div id="stats" style="display: grid; grid-template-columns: repeat(auto-fit, minmax(150px, 1fr)); gap: 15px;">
    <!-- Stats render here -->
  </div>
</div>

<script>
  // Polling function
  async function fetchData() {
    try {
      // Option 1: Fetch from external API (if CORS allows)
      const response = await fetch('https://api.example.com/stats');
      const data = await response.json();
      updateDisplay(data);
    } catch (error) {
      // Fallback: Use cached data or placeholder
      const cached = JSON.parse(sessionStorage.getItem('last_data') || '{}');
      updateDisplay(cached);
    }
  }
  
  function updateDisplay(data) {
    const statsHtml = Object.entries(data).map(([key, value]) => `
      <div style="background: var(--surface-1); padding: 15px; border-radius: var(--radius);">
        <div style="color: var(--text-secondary); font-size: 12px;">${key}</div>
        <div style="font-size: 24px; font-weight: bold; color: var(--primary);">${value}</div>
      </div>
    `).join('');
    
    document.getElementById('stats').innerHTML = statsHtml;
    sessionStorage.setItem('last_data', JSON.stringify(data));
  }
  
  // Poll every 10 seconds
  fetchData();  // Initial load
  setInterval(fetchData, 10000);
</script>
```

**Considerations:**
- CORS must allow requests from widget domain
- Fallback to cached data if API unavailable
- Polling interval affects performance (balance freshness vs. load)

**Alternative: Event-Based (Future)**
```javascript
// If host supports message passing (not yet standard)
window.addEventListener('message', (e) => {
  if (e.data.type === 'data_update') {
    updateDisplay(e.data.payload);
  }
});
```

---

## Pattern 3: Multi-Step Workflows

**Use Case**: Widget guides user through sequential steps

### Challenge
Claude generates one big HTML chunk. How to show sequential UI?

### Solution: Step-Based State Machine

```html
<div id="wizard" style="padding: 20px;">
  <div id="progress" style="margin-bottom: 20px;">
    <div style="display: flex; gap: 10px; font-size: 12px;">
      <span id="step-indicator" style="color: var(--text-secondary);">Step 1 of 3</span>
      <div style="flex: 1; height: 4px; background: var(--surface-1); border-radius: 2px; overflow: hidden;">
        <div id="progress-bar" style="height: 100%; width: 33%; background: var(--primary); transition: width 0.3s;"></div>
      </div>
    </div>
  </div>
  
  <div id="step-1" class="step" style="display: block;">
    <h2>Step 1: Choose Category</h2>
    <select id="category" style="width: 100%; padding: 10px;">
      <option value="">-- Select --</option>
      <option value="business">Business</option>
      <option value="personal">Personal</option>
      <option value="technical">Technical</option>
    </select>
  </div>
  
  <div id="step-2" class="step" style="display: none;">
    <h2>Step 2: Enter Details</h2>
    <input type="text" id="details" placeholder="Details..." style="width: 100%; padding: 10px;">
  </div>
  
  <div id="step-3" class="step" style="display: none;">
    <h2>Step 3: Review & Confirm</h2>
    <div id="review" style="background: var(--surface-1); padding: 15px; border-radius: var(--radius);"></div>
  </div>
  
  <div style="display: flex; gap: 10px; margin-top: 20px;">
    <button id="prev" onclick="previousStep()" style="padding: 10px 20px; background: var(--surface-1); border: none; border-radius: var(--radius); cursor: pointer;">Back</button>
    <button id="next" onclick="nextStep()" style="padding: 10px 20px; background: var(--primary); color: white; border: none; border-radius: var(--radius); cursor: pointer;">Next</button>
  </div>
</div>

<script>
  let currentStep = 1;
  const maxStep = 3;
  const data = {};
  
  function showStep(step) {
    document.querySelectorAll('.step').forEach(s => s.style.display = 'none');
    document.getElementById(`step-${step}`).style.display = 'block';
    
    // Update progress
    document.getElementById('step-indicator').textContent = `Step ${step} of ${maxStep}`;
    document.getElementById('progress-bar').style.width = `${(step / maxStep) * 100}%`;
    
    // Update buttons
    document.getElementById('prev').style.display = step === 1 ? 'none' : 'block';
    document.getElementById('next').textContent = step === maxStep ? 'Finish' : 'Next';
  }
  
  function nextStep() {
    // Save current step data
    if (currentStep === 1) data.category = document.getElementById('category').value;
    if (currentStep === 2) data.details = document.getElementById('details').value;
    
    // Show review on final step
    if (currentStep === 3) {
      document.getElementById('review').innerHTML = `
        <strong>Category:</strong> ${data.category}<br>
        <strong>Details:</strong> ${data.details}
      `;
    }
    
    if (currentStep < maxStep) {
      currentStep++;
      showStep(currentStep);
    } else {
      // Finish
      console.log('Completed:', data);
      localStorage.setItem('wizard_data', JSON.stringify(data));
    }
  }
  
  function previousStep() {
    if (currentStep > 1) {
      currentStep--;
      showStep(currentStep);
    }
  }
  
  showStep(1);
</script>
```

**Benefits:**
- ✅ Guides users through complex flows
- ✅ All steps in single HTML/JS (no regeneration needed)
- ✅ Validation can happen per-step
- ✅ Results persist in localStorage

---

## Pattern 4: Form with Validation

**Use Case**: Collect user input with real-time validation

### Challenge
How to validate input and show errors without server communication?

### Solution: Client-Side Validation Schema

```html
<div style="padding: 20px; max-width: 500px;">
  <h1>Contact Form</h1>
  <form id="contactForm">
    <div style="margin-bottom: 15px;">
      <label style="display: block; margin-bottom: 5px; font-weight: bold;">Email *</label>
      <input type="email" name="email" id="email" 
        style="width: 100%; padding: 10px; border: 1px solid #ccc; border-radius: var(--radius);"
        placeholder="your@email.com" required>
      <div class="error-message" style="color: #F44336; font-size: 12px; margin-top: 5px; display: none;"></div>
    </div>
    
    <div style="margin-bottom: 15px;">
      <label style="display: block; margin-bottom: 5px; font-weight: bold;">Message *</label>
      <textarea name="message" id="message" 
        style="width: 100%; padding: 10px; border: 1px solid #ccc; border-radius: var(--radius); min-height: 100px;"
        placeholder="Your message..." required></textarea>
      <div class="error-message" style="color: #F44336; font-size: 12px; margin-top: 5px; display: none;"></div>
    </div>
    
    <div style="margin-bottom: 15px;">
      <label style="display: block; margin-bottom: 5px; font-weight: bold;">Phone (Optional)</label>
      <input type="tel" name="phone" id="phone"
        style="width: 100%; padding: 10px; border: 1px solid #ccc; border-radius: var(--radius);"
        placeholder="(123) 456-7890">
      <div class="error-message" style="color: #F44336; font-size: 12px; margin-top: 5px; display: none;"></div>
    </div>
    
    <button type="submit" style="width: 100%; padding: 12px; background: var(--primary); color: white; border: none; border-radius: var(--radius); cursor: pointer; font-weight: bold;">
      Send Message
    </button>
  </form>
  
  <div id="success-message" style="display: none; margin-top: 20px; padding: 15px; background: #4CAF50; color: white; border-radius: var(--radius);">
    ✓ Message sent successfully!
  </div>
</div>

<script>
  const validators = {
    email: (value) => {
      const regex = /^[^\\s@]+@[^\\s@]+\\.[^\\s@]+$/;
      return regex.test(value) ? null : 'Invalid email format';
    },
    message: (value) => {
      if (value.trim().length < 10) return 'Message must be at least 10 characters';
      if (value.length > 1000) return 'Message must be less than 1000 characters';
      return null;
    },
    phone: (value) => {
      if (!value) return null;  // Optional
      const regex = /^[\\d\\s\\-\\+\\(\\)]+$/;
      return regex.test(value) ? null : 'Invalid phone format';
    }
  };
  
  function validateField(name, value) {
    const validator = validators[name];
    if (!validator) return null;
    return validator(value);
  }
  
  const form = document.getElementById('contactForm');
  const fields = form.querySelectorAll('input, textarea');
  
  // Real-time validation
  fields.forEach(field => {
    field.addEventListener('blur', () => {
      const error = validateField(field.name, field.value);
      const errorDiv = field.parentElement.querySelector('.error-message');
      
      if (error) {
        field.style.borderColor = '#F44336';
        errorDiv.textContent = error;
        errorDiv.style.display = 'block';
      } else {
        field.style.borderColor = '#ccc';
        errorDiv.style.display = 'none';
      }
    });
  });
  
  // Form submission
  form.addEventListener('submit', (e) => {
    e.preventDefault();
    
    // Validate all fields
    let isValid = true;
    const data = {};
    
    fields.forEach(field => {
      const error = validateField(field.name, field.value);
      const errorDiv = field.parentElement.querySelector('.error-message');
      
      if (error) {
        isValid = false;
        field.style.borderColor = '#F44336';
        errorDiv.textContent = error;
        errorDiv.style.display = 'block';
      } else {
        data[field.name] = field.value;
      }
    });
    
    if (isValid) {
      // Save to localStorage
      localStorage.setItem('contact_form_data', JSON.stringify(data));
      
      // Show success
      form.style.display = 'none';
      document.getElementById('success-message').style.display = 'block';
      
      console.log('Form submitted:', data);
    }
  });
</script>
```

**Features:**
- ✅ Real-time validation on blur
- ✅ Clear error messages
- ✅ Prevents submission if invalid
- ✅ Data saved to localStorage

---

## Pattern 5: Async Data Loading with Fallback

**Use Case**: Widget needs to fetch data but must work offline

### Solution: Fetch with Graceful Degradation

```html
<div id="data-container" style="padding: 20px;">
  <div id="loading" style="text-align: center; color: var(--text-secondary);">
    ⏳ Loading data...
  </div>
  <div id="content" style="display: none;"></div>
  <div id="error" style="display: none; padding: 15px; background: #FFF3CD; border-radius: var(--radius); color: #856404;"></div>
</div>

<script>
  const DATA_CACHE_KEY = 'widget_data_cache';
  const DEFAULT_DATA = [
    { id: 1, title: 'Fallback Item 1', value: 100 },
    { id: 2, title: 'Fallback Item 2', value: 200 }
  ];
  
  async function loadData() {
    try {
      // Try fetching from API
      const response = await fetch('https://api.example.com/data', {
        signal: AbortSignal.timeout(5000)  // 5s timeout
      });
      
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      
      const data = await response.json();
      
      // Cache successful response
      sessionStorage.setItem(DATA_CACHE_KEY, JSON.stringify(data));
      
      return data;
    } catch (error) {
      console.warn('Fetch failed:', error);
      
      // Try cache first
      const cached = sessionStorage.getItem(DATA_CACHE_KEY);
      if (cached) {
        console.log('Using cached data');
        return JSON.parse(cached);
      }
      
      // Fall back to defaults
      console.log('Using default data');
      return DEFAULT_DATA;
    }
  }
  
  async function render() {
    const loadingDiv = document.getElementById('loading');
    const contentDiv = document.getElementById('content');
    const errorDiv = document.getElementById('error');
    
    try {
      const data = await loadData();
      
      contentDiv.innerHTML = data.map(item => `
        <div style="padding: 12px; border-bottom: 1px solid var(--surface-1);">
          <strong>${item.title}</strong>
          <div style="color: var(--text-secondary);">${item.value}</div>
        </div>
      `).join('');
      
      loadingDiv.style.display = 'none';
      contentDiv.style.display = 'block';
    } catch (error) {
      loadingDiv.style.display = 'none';
      errorDiv.innerHTML = `
        <strong>⚠️ Error loading data</strong><br>
        ${error.message}<br>
        <small>Using offline defaults</small>
      `;
      errorDiv.style.display = 'block';
      
      // Still show defaults on error
      const data = DEFAULT_DATA;
      contentDiv.innerHTML = data.map(item => `
        <div style="padding: 12px; border-bottom: 1px solid var(--surface-1); opacity: 0.7;">
          <strong>${item.title}</strong>
          <div style="color: var(--text-secondary);">${item.value} (offline)</div>
        </div>
      `).join('');
      
      contentDiv.style.display = 'block';
    }
  }
  
  // Load on widget init
  render();
  
  // Optional: Retry every 30 seconds if offline
  setInterval(() => {
    const cached = sessionStorage.getItem(DATA_CACHE_KEY);
    if (!cached) render();  // Only retry if no cache
  }, 30000);
</script>
```

**Features:**
- ✅ Timeout protection (doesn't hang forever)
- ✅ Session cache for recent success
- ✅ Graceful fallback to defaults
- ✅ Retry logic
- ✅ Works offline

---

## Pattern 6: Keyboard Shortcuts

**Use Case**: Power users want keyboard efficiency

### Solution: Keyboard Event Handling

```html
<div style="padding: 20px;">
  <h1>Quick Actions</h1>
  <div style="margin-bottom: 20px; padding: 12px; background: var(--surface-1); border-radius: var(--radius); font-size: 12px; color: var(--text-secondary);">
    💡 Tip: Press <kbd>Ctrl</kbd> + <kbd>S</kbd> to save, <kbd>?</kbd> for help
  </div>
  
  <input id="mainInput" placeholder="Type something..." style="width: 100%; padding: 10px; margin-bottom: 10px;">
  
  <div id="output" style="padding: 12px; background: var(--surface-0); border-radius: var(--radius);"></div>
</div>

<script>
  const shortcuts = {
    's': (e) => {  // Ctrl+S
      if (e.ctrlKey || e.metaKey) {
        e.preventDefault();
        save();
      }
    },
    '?': (e) => {  // ?
      if (!e.ctrlKey && !e.metaKey) {
        showHelp();
      }
    },
    'escape': (e) => {
      clearInput();
    }
  };
  
  function save() {
    const value = document.getElementById('mainInput').value;
    localStorage.setItem('saved_data', value);
    document.getElementById('output').textContent = '✓ Saved!';
    setTimeout(() => {
      document.getElementById('output').textContent = '';
    }, 2000);
  }
  
  function showHelp() {
    const help = `
      <strong>Keyboard Shortcuts:</strong><br>
      • Ctrl+S: Save<br>
      • Esc: Clear<br>
      • ?: Show this help
    `;
    document.getElementById('output').innerHTML = help;
  }
  
  function clearInput() {
    document.getElementById('mainInput').value = '';
    document.getElementById('mainInput').focus();
  }
  
  document.addEventListener('keydown', (e) => {
    const key = e.key.toLowerCase();
    const handler = shortcuts[key];
    if (handler) handler(e);
  });
  
  // Load saved data on init
  const saved = localStorage.getItem('saved_data');
  if (saved) document.getElementById('mainInput').value = saved;
  document.getElementById('mainInput').focus();
</script>
```

---

## Pattern 7: Composition: Embedded Widgets

**Use Case**: Show multiple related widgets in one container

### Solution: Sub-Widgets Pattern

```html
<div style="display: grid; grid-template-columns: 1fr 1fr; gap: 20px; padding: 20px;">
  <!-- Widget 1: Stats -->
  <div id="stats-widget" style="background: var(--surface-1); padding: 15px; border-radius: var(--radius);">
    <h3>Statistics</h3>
    <div>Total: <strong id="total">0</strong></div>
    <div>Average: <strong id="avg">0</strong></div>
  </div>
  
  <!-- Widget 2: Chart -->
  <div id="chart-widget" style="background: var(--surface-1); padding: 15px; border-radius: var(--radius);">
    <h3>Trend</h3>
    <div id="chart" style="height: 150px; background: var(--surface-0);"></div>
  </div>
  
  <!-- Shared Input -->
  <div style="grid-column: 1/-1;">
    <input type="range" id="slider" min="0" max="100" value="50" style="width: 100%;">
  </div>
</div>

<script>
  const slider = document.getElementById('slider');
  let state = {
    values: Array.from({ length: 10 }, (_, i) => i * 10)
  };
  
  // Shared update function for all widgets
  function updateAllWidgets() {
    updateStats();
    updateChart();
    saveState();
  }
  
  function updateStats() {
    const total = state.values.reduce((a, b) => a + b, 0);
    const avg = total / state.values.length;
    document.getElementById('total').textContent = total;
    document.getElementById('avg').textContent = avg.toFixed(1);
  }
  
  function updateChart() {
    // Simple canvas chart
    const canvas = document.getElementById('chart');
    canvas.innerHTML = state.values.map((v, i) => 
      `<div style="display: inline-block; margin-right: 2px; width: ${v}%; height: 20px; background: var(--primary);"></div>`
    ).join('');
  }
  
  function saveState() {
    localStorage.setItem('composite_state', JSON.stringify(state));
  }
  
  // Connect input to state
  slider.addEventListener('input', (e) => {
    state.values[Math.floor(Math.random() * 10)] = parseInt(e.target.value);
    updateAllWidgets();
  });
  
  // Load initial state
  const saved = localStorage.getItem('composite_state');
  if (saved) state = JSON.parse(saved);
  
  updateAllWidgets();
</script>
```

---

## Performance Tips

### Optimization 1: Lazy Load Heavy Content
```javascript
// Load chart.js only when needed
if (shouldShowChart) {
  const script = document.createElement('script');
  script.src = 'https://cdn.jsdelivr.net/npm/chart.js@4';
  document.head.appendChild(script);
  script.onload = renderChart;
}
```

### Optimization 2: Debounce Input Events
```javascript
function debounce(func, wait) {
  let timeout;
  return function(...args) {
    clearTimeout(timeout);
    timeout = setTimeout(() => func(...args), wait);
  };
}

input.addEventListener('input', debounce(() => {
  updateWidget();
}, 300));  // Only update after 300ms of inactivity
```

### Optimization 3: Virtual Scrolling for Lists
```javascript
// Only render visible items for large lists
const items = [];
const visibleStart = Math.max(0, scrollTop / itemHeight - 5);
const visibleEnd = Math.min(items.length, scrollTop / itemHeight + containerHeight / itemHeight + 5);
container.innerHTML = items.slice(visibleStart, visibleEnd).map(render).join('');
```

---

## Security in Advanced Patterns

⚠️ **Never** trust localStorage for sensitive data:
```javascript
// ❌ WRONG
localStorage.setItem('user_token', authToken);

// ✅ CORRECT: Use sessionStorage or memory only
sessionStorage.setItem('temp_session', sessionId);
```

⚠️ **Sanitize HTML** from external sources:
```javascript
// ❌ WRONG
div.innerHTML = userData;

// ✅ CORRECT: Use textContent for text, sanitize HTML
div.textContent = userData;  // or use DOMPurify library
```

