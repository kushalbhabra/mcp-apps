# Real-World Widget Examples

This guide showcases production-ready widget patterns with Claude-generated code examples.

## 1. Compound Interest Calculator (Generative)

**Pattern**: Generative Components (Claude generates full code)

**Use Case**: Financial calculation with visualization

```typescript
show_widget({
  title: "compound_interest_calc",
  widget_code: `
    <div style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI'; padding: 20px; max-width: 500px;">
      <h2 style="color: var(--text-primary);">Compound Interest Calculator</h2>
      <div style="display: flex; flex-direction: column; gap: 12px;">
        <div>
          <label style="display: block; margin-bottom: 4px; color: var(--text-secondary);">Initial Amount ($)</label>
          <input type="number" id="principal" value="1000" style="width: 100%; padding: 8px; border: 1px solid #ccc; border-radius: var(--radius);">
        </div>
        <div>
          <label style="display: block; margin-bottom: 4px; color: var(--text-secondary);">Annual Rate (%)</label>
          <input type="number" id="rate" value="5" step="0.1" style="width: 100%; padding: 8px; border: 1px solid #ccc; border-radius: var(--radius);">
        </div>
        <div>
          <label style="display: block; margin-bottom: 4px; color: var(--text-secondary);">Years</label>
          <input type="number" id="years" value="10" style="width: 100%; padding: 8px; border: 1px solid #ccc; border-radius: var(--radius);">
        </div>
        <div>
          <label style="display: block; margin-bottom: 4px; color: var(--text-secondary);">Compounding</label>
          <select id="compound" style="width: 100%; padding: 8px; border: 1px solid #ccc; border-radius: var(--radius);">
            <option value="1">Annually</option>
            <option value="2">Semi-annually</option>
            <option value="4">Quarterly</option>
            <option value="12" selected>Monthly</option>
            <option value="365">Daily</option>
          </select>
        </div>
      </div>
      <div id="result" style="margin-top: 20px; padding: 15px; background: var(--surface-1); border-radius: var(--radius); text-align: center;">
        <div style="color: var(--text-secondary); font-size: 14px;">Final Amount</div>
        <div style="color: var(--primary); font-size: 32px; font-weight: bold; margin-top: 5px;">$0.00</div>
      </div>
      <script>
        function calculate() {
          const P = parseFloat(document.getElementById('principal').value) || 0;
          const r = parseFloat(document.getElementById('rate').value) || 0;
          const t = parseFloat(document.getElementById('years').value) || 0;
          const n = parseInt(document.getElementById('compound').value) || 1;
          
          const A = P * Math.pow(1 + r/100/n, n*t);
          document.querySelector('#result div:last-child').textContent = '$' + A.toFixed(2);
        }
        
        document.getElementById('principal').addEventListener('input', calculate);
        document.getElementById('rate').addEventListener('input', calculate);
        document.getElementById('years').addEventListener('input', calculate);
        document.getElementById('compound').addEventListener('change', calculate);
        
        calculate();
      </script>
    </div>
  `
})
```

**Why This Works:**
- ✅ Pure calculation, no API calls needed
- ✅ User input drives output (immediate feedback)
- ✅ Fits in single string (< 3KB)
- ✅ CSS variables for theming
- ✅ No external dependencies

**Token Cost**: ~2,500 tokens for generation, ~0 per interaction

---

## 2. Recipe Display (Structured Data)

**Pattern**: Static Components with dynamic data

**Use Case**: Display recipe with scalable ingredients

```typescript
show_widget({
  title: "recipe_display",
  widget_code: `
    <div style="font-family: -apple-system, BlinkMacSystemFont; padding: 20px; max-width: 600px; color: var(--text-primary);">
      <h1 id="title" style="margin: 0 0 10px 0; font-size: 28px;"></h1>
      <p id="description" style="color: var(--text-secondary); margin-bottom: 20px;"></p>
      
      <div style="display: flex; gap: 10px; margin-bottom: 20px; align-items: center;">
        <label for="servings" style="color: var(--text-secondary);">Servings:</label>
        <select id="servings" style="padding: 6px 10px; border: 1px solid #ccc; border-radius: var(--radius);">
          <option value="0.5">0.5x</option>
          <option value="1" selected>1x</option>
          <option value="1.5">1.5x</option>
          <option value="2">2x</option>
        </select>
      </div>
      
      <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 30px;">
        <div>
          <h2 style="font-size: 18px; margin-bottom: 12px;">Ingredients</h2>
          <ul id="ingredients" style="list-style: none; padding: 0; margin: 0;"></ul>
        </div>
        <div>
          <h2 style="font-size: 18px; margin-bottom: 12px;">Instructions</h2>
          <ol id="steps" style="padding-left: 20px; line-height: 1.8;"></ol>
        </div>
      </div>
      
      <div id="notes" style="margin-top: 20px; padding: 12px; background: var(--surface-1); border-radius: var(--radius); font-size: 14px; color: var(--text-secondary);"></div>
      
      <script>
        const recipeData = window.__RECIPE_DATA__ || {};
        
        document.getElementById('title').textContent = recipeData.title || 'Recipe';
        document.getElementById('description').textContent = recipeData.description || '';
        document.getElementById('notes').textContent = recipeData.notes || '';
        
        const servings = document.getElementById('servings');
        function updateRecipe() {
          const scale = parseFloat(servings.value);
          const ingredientsList = document.getElementById('ingredients');
          ingredientsList.innerHTML = (recipeData.ingredients || []).map(ing => {
            const amount = ing.amount * scale;
            return \`<li style="padding: 6px 0; border-bottom: 1px solid var(--surface-1);">\${amount.toFixed(2)} \${ing.unit} <strong>\${ing.name}</strong></li>\`;
          }).join('');
          
          const stepsList = document.getElementById('steps');
          stepsList.innerHTML = (recipeData.steps || []).map(step => {
            return \`<li style="margin-bottom: 12px;"><strong>\${step.title}:</strong> \${step.content}</li>\`;
          }).join('');
        }
        
        servings.addEventListener('change', updateRecipe);
        updateRecipe();
      </script>
    </div>
  `,
  loading_messages: ["Fetching recipe...", "Preparing ingredients..."]
})
```

**Data Structure** (passed as separate call):
```json
{
  "title": "Indian-Style Paneer Tikka Pizza",
  "description": "A desi twist on pizza with spiced paneer, onions, capsicum...",
  "ingredients": [
    { "amount": 300, "unit": "g", "name": "all-purpose flour" },
    { "amount": 1.5, "unit": "tsp", "name": "instant dry yeast" }
  ],
  "steps": [
    { "title": "Make the dough", "content": "Mix water with sugar and yeast..." },
    { "title": "Let dough rise", "content": "Cover with damp cloth for 1 hour..." }
  ],
  "notes": "Can substitute paneer with tandoori chicken..."
}
```

**Why This Works:**
- ✅ Data-driven (Claude provides parameters, you control HTML)
- ✅ Scaling built-in (0.5x - 2x recipe multiplier)
- ✅ Semantic structure (organized ingredients + steps)
- ✅ Mobile-responsive grid layout

**Token Cost**: ~1,500 tokens for widget template, ~500 per recipe data update

---

## 3. Real-Time Stock Dashboard (With CDN)

**Pattern**: External library integration

**Use Case**: Market data visualization with no backend

```typescript
show_widget({
  title: "stock_dashboard",
  widget_code: `
    <div style="font-family: -apple-system, BlinkMacSystemFont; padding: 20px;">
      <h1 style="margin: 0 0 20px 0; color: var(--text-primary);">Stock Dashboard</h1>
      <div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(250px, 1fr)); gap: 15px;" id="stocks"></div>
      <script src="https://cdn.jsdelivr.net/npm/chart.js@4"></script>
      <script>
        const stocks = [
          { symbol: 'AAPL', price: 182.45, change: 2.3 },
          { symbol: 'GOOGL', price: 139.20, change: -1.2 },
          { symbol: 'MSFT', price: 378.91, change: 3.4 },
          { symbol: 'AMZN', price: 172.58, change: 1.1 }
        ];
        
        const container = document.getElementById('stocks');
        stocks.forEach(stock => {
          const isPositive = stock.change > 0;
          const card = document.createElement('div');
          card.style.cssText = \`
            padding: 15px; 
            background: var(--surface-1); 
            border-radius: var(--radius); 
            border-left: 4px solid \${isPositive ? '#4CAF50' : '#F44336'};
          \`;
          card.innerHTML = \`
            <div style="font-weight: bold; color: var(--text-primary); margin-bottom: 5px;">\${stock.symbol}</div>
            <div style="font-size: 24px; font-weight: bold; color: var(--primary);">\$\${stock.price}</div>
            <div style="color: \${isPositive ? '#4CAF50' : '#F44336'}; font-size: 14px;">\${isPositive ? '+' : ''}\${stock.change}%</div>
          \`;
          container.appendChild(card);
        });
      </script>
    </div>
  `
})
```

**Why This Works:**
- ✅ CDN libraries (Chart.js, lightweight)
- ✅ No API calls (data embedded)
- ✅ Color-coded indicators
- ✅ Responsive grid

---

## 4. Todo List with localStorage (Stateful)

**Pattern**: Client-side state persistence

**Use Case**: Simple task management

```typescript
show_widget({
  title: "todo_list",
  widget_code: `
    <div style="font-family: -apple-system; padding: 20px; max-width: 400px;">
      <h1 style="color: var(--text-primary); margin-bottom: 15px;">My Tasks</h1>
      <div style="display: flex; gap: 8px; margin-bottom: 15px;">
        <input type="text" id="newTodo" placeholder="Add a task..." 
          style="flex: 1; padding: 10px; border: 1px solid #ccc; border-radius: var(--radius);">
        <button style="padding: 10px 20px; background: var(--primary); color: white; border: none; border-radius: var(--radius); cursor: pointer;">Add</button>
      </div>
      <ul id="todos" style="list-style: none; padding: 0; margin: 0;"></ul>
      <script>
        // Load from localStorage
        let todos = JSON.parse(localStorage.getItem('app_todos') || '[]');
        
        function save() {
          localStorage.setItem('app_todos', JSON.stringify(todos));
          render();
        }
        
        function render() {
          const list = document.getElementById('todos');
          list.innerHTML = todos.map((todo, i) => \`
            <li style="padding: 10px; border-bottom: 1px solid var(--surface-1); display: flex; gap: 10px;">
              <input type="checkbox" \${todo.done ? 'checked' : ''} 
                onchange="todos[\${i}].done = this.checked; save();"
                style="cursor: pointer;">
              <span style="flex: 1; text-decoration: \${todo.done ? 'line-through' : 'none'};">\${todo.text}</span>
              <button onclick="todos.splice(\${i}, 1); save();" style="background: none; border: none; color: red; cursor: pointer;">✕</button>
            </li>
          \`).join('');
        }
        
        document.querySelector('button').onclick = () => {
          const text = document.getElementById('newTodo').value.trim();
          if (text) {
            todos.push({ text, done: false });
            document.getElementById('newTodo').value = '';
            save();
          }
        };
        
        render();
      </script>
    </div>
  `
})
```

**Why This Works:**
- ✅ Persistent state via localStorage
- ✅ No server needed
- ✅ Survives widget re-renders
- ✅ Simple interactive UI

---

## 5. Color Palette Generator (Algorithmic)

**Pattern**: Generative content from computation

**Use Case**: Design tools & utilities

```typescript
show_widget({
  title: "color_palette_gen",
  widget_code: `
    <div style="font-family: -apple-system; padding: 20px; max-width: 500px;">
      <h1 style="color: var(--text-primary);">Color Palette Generator</h1>
      <div style="margin: 15px 0;">
        <label style="display: block; margin-bottom: 8px; color: var(--text-secondary);">Base Color</label>
        <input type="color" id="baseColor" value="#3498db" style="width: 100%; height: 50px; cursor: pointer; border: none; border-radius: var(--radius);">
      </div>
      <button onclick="generate()" style="width: 100%; padding: 12px; background: var(--primary); color: white; border: none; border-radius: var(--radius); cursor: pointer; font-weight: bold;">Generate Palette</button>
      <div id="palette" style="display: grid; grid-template-columns: repeat(auto-fit, minmax(120px, 1fr)); gap: 10px; margin-top: 20px;"></div>
      <script>
        function generate() {
          const base = document.getElementById('baseColor').value;
          const colors = [base];
          
          // Generate variations
          for (let i = 1; i <= 4; i++) {
            colors.push(shiftHue(base, i * 30));
            colors.push(shiftLuminance(base, i * 15));
          }
          
          const palette = document.getElementById('palette');
          palette.innerHTML = colors.map(color => \`
            <div style="text-align: center;">
              <div style="width: 100%; height: 100px; background: \${color}; border-radius: var(--radius); cursor: pointer; margin-bottom: 8px;" 
                onclick="navigator.clipboard.writeText('\${color}');">
              </div>
              <code style="font-size: 12px; color: var(--text-secondary);">\${color}</code>
            </div>
          \`).join('');
        }
        
        function shiftHue(color, degrees) {
          const rgb = parseInt(color.slice(1), 16);
          // Simplified HSL shift
          return '#' + ((rgb + degrees * 1000) % 0xFFFFFF).toString(16).padStart(6, '0');
        }
        
        function shiftLuminance(color, amount) {
          const rgb = parseInt(color.slice(1), 16);
          const r = Math.min(255, (rgb >> 16) + amount);
          const g = Math.min(255, ((rgb >> 8) & 0xFF) + amount);
          const b = Math.min(255, (rgb & 0xFF) + amount);
          return '#' + [r, g, b].map(x => Math.max(0, x).toString(16).padStart(2, '0')).join('');
        }
        
        generate();
      </script>
    </div>
  `
})
```

**Why This Works:**
- ✅ Algorithmic generation (no APIs)
- ✅ Interactive controls
- ✅ Copy-to-clipboard UX
- ✅ Real-time updates

---

## Pattern Selection Guide

| Pattern | Best For | Example |
|---------|----------|---------|
| **Generative** | Calculations, text transforms | Compound interest, markdown editor |
| **Static + Data** | Structured content, templates | Recipes, resumes |
| **CDN Libraries** | Visualization, charting | Stock dashboard, analytics |
| **Client State** | Persistence, user input | Todo lists, note taking |
| **Algorithmic** | Content generation, utilities | Color palette, QR code |

---

## Prompting Tips for Widgets

✅ **Be Specific:**
```
❌ "Create a calculator"
✅ "Create a compound interest calculator widget with inputs for principal, rate, years, and compounding frequency"
```

✅ **Reference CSS Variables:**
```
❌ Hardcode colors in CSS
✅ Use var(--primary), var(--text-primary), var(--surface-1)
```

✅ **Keep It Self-Contained:**
```
❌ "Fetch data from my API"
✅ "Accept data as a parameter, show a dashboard interface"
```

✅ **Define Clear Interactions:**
```
❌ "Make it interactive"
✅ "Allow users to adjust inputs and see results update in real-time"
```

---

## Common Mistakes to Avoid

❌ **Too large** (>10KB)
- Solution: Use CDN for libraries

❌ **API calls without error handling**
- Solution: Embed data or use fetch with try/catch

❌ **Hard-coded colors**
- Solution: Always use CSS variables

❌ **No responsive design**
- Solution: Use flexbox/grid with mobile-first media queries

❌ **Missing loading states**
- Solution: Use loading_messages parameter

