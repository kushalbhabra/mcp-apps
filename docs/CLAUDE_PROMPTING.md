# Claude Prompting Guide for MCP Widgets

Effective prompts generate better widget code. This guide teaches the patterns that work best.

## Core Principle

Claude works best when you:
1. **Describe the goal** (what should it do)
2. **Explain the structure** (how should it look)
3. **Specify constraints** (size, dependencies, styling)
4. **Show examples** (optional, but powerful)

---

## Template: The Perfect Widget Prompt

```
Create an [WIDGET TYPE] widget that:
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

---

## Real Examples That Work

### ✅ GOOD: Specific, Constrained, Clear

```
Create a mortgage calculator widget that:
- Takes home price, down payment %, interest rate, and loan years
- Calculates monthly payment using the formula: M = P[r(1+r)^n]/[(1+r)^n-1]
- Shows final amount paid and total interest in prominent display

Layout:
- 4 input fields stacked vertically
- Large number display below inputs
- Update results in real-time as user types

Styling:
- Use CSS variables for colors
- Desktop width ~400px, mobile responsive
- No external libraries

Don't include:
- API calls
- Complex animations
- Extra features
```

**Result**: Claude generates exactly the widget you need, ~1.5KB code

---

### ❌ BAD: Vague, Unlimited Scope

```
Create a financial calculator
```

**Result**: Claude might generate too much (amortization schedule, refinance comparison, etc.) or miss what you wanted

---

## Pattern-Specific Prompts

### Pattern 1: Generative Components (Calculation/Transformation)

Use this when Claude generates the full widget code:

```
Create a [TOOL TYPE] widget that transforms [INPUT] to [OUTPUT].

The widget should:
- Accept [input fields with types]
- Compute [algorithm/formula]
- Display [result format]

Make it interactive so users can:
- Modify inputs and see results update instantly
- Copy/export the result
- [Optional: show intermediate steps]

Keep it under 1KB of code. Use only vanilla JavaScript.
Use CSS variables for theming: --primary, --text-primary, --surface-1.
```

**Examples:**
- "Create a unit converter widget that transforms temperature between Celsius/Fahrenheit/Kelvin"
- "Create a markdown-to-HTML preview widget"
- "Create a regex pattern tester widget"

---

### Pattern 2: Static Components with Data (Templates)

Use this when you provide data, Claude builds the template:

```
Create a widget template to display [DATA TYPE] with this structure:
- Title: [from data.field]
- Sections: [describe layout]
- Interactive elements: [scaling, filtering, sorting]

The widget receives data in this JSON format:
{
  "field1": "type and description",
  "field2": [{"nested": "structure"}]
}

Features:
- [interactivity requirement]
- Responsive grid/flex layout
- Use CSS variables

Show this data:
[example JSON]
```

**Examples:**
- "Create a recipe display widget with scalable ingredients"
- "Create a product listing widget with filtering"
- "Create a resume/CV display widget"

---

### Pattern 3: Algorithmic Generation (Output-Focused)

Use this when Claude generates content:

```
Create a [CONTENT TYPE] generator widget that:
- Accepts [input parameters]
- Generates [output type] based on [algorithm]
- Displays/allows exporting the result

Implementation:
- Use vanilla JavaScript (no libraries)
- All processing happens in-browser
- Cache results in widget state

Example:
- Input: [example input]
- Output: [expected output]

Styling: Use CSS variables, fit in 300x400px container
```

**Examples:**
- "Create a color palette generator from a base color"
- "Create a QR code generator from text input"
- "Create a random name generator for [type]"

---

## Effective Constraint Patterns

### 1. Size Constraint (Most Important)

```
Keep the widget code under 2KB total.
```

**Why**: Smaller widgets load faster, fit in tokens better

**How Claude handles it**:
- ✅ Removes comments, extra whitespace
- ✅ Uses efficient selectors
- ✅ Skips unused features
- ❌ Will refuse if impossible (size is hard limit)

### 2. Technology Constraint

```
Use only vanilla JavaScript - no libraries.
Use CSS from the host theme: var(--primary), var(--text-primary), etc.
No external APIs or fetch calls.
```

**Why**: 
- Faster (no CDN download)
- Portable (works offline)
- Safe (no external dependencies)

### 3. Interaction Constraint

```
The widget should update instantly as users type - no buttons or delays.
All state persists in the browser - no server calls.
```

**Why**:
- UX feels responsive
- Doesn't require backend
- Works if disconnected

### 4. Styling Constraint

```
Make it work on mobile (viewport < 400px width).
Use a single column on mobile, 2+ columns on desktop.
```

**Why**:
- Claude generates responsive by default
- Ensures usability everywhere

---

## The Ask-Show-Verify Pattern

### 1. Ask (What you want)
```
I need a widget that calculates loan payments
```

### 2. Show (What good looks like)
```
Show me a version that:
- Displays monthly payment prominently
- Updates as I type
- Explains what each field means

Here's a similar widget you made before:
[copy/paste example or describe structure]
```

### 3. Verify (Test the output)
```
Does this work? [Try the widget]
Let me adjust: [iterate on features]
```

---

## Common Issues & Fixes

### Issue: "Widget is too big (>3KB)"

**Fix your prompt:**
```
❌ Original: "Create a finance app with charts, calculators, and reports"
✅ Better: "Create a simple mortgage calculator under 2KB"
```

**Claude's adjustment:**
- Removes chart library
- Removes secondary features
- Keeps core calculation

---

### Issue: "Colors look wrong in light/dark mode"

**Fix your prompt:**
```
❌ Hardcoded colors: "Use blue (#3498db) for buttons"
✅ Theme-aware: "Use CSS variables: var(--primary) for buttons"
```

**Claude will use:**
```css
button { background: var(--primary); }  /* ✅ Theme-aware */
button { background: #3498db; }         /* ❌ Breaks dark mode */
```

---

### Issue: "Widget doesn't work without internet"

**Fix your prompt:**
```
❌ "Fetch weather data from an API"
✅ "Accept city name as input, show a weather widget UI"
```

**Claude will:**
- Remove API call
- Focus on interface
- Use static data for demo

---

### Issue: "Claude generated too many features"

**Fix your prompt:**
```
❌ "Create a notes app"
✅ "Create a simple note-taking widget with:
     - Text input area
     - Save/load from localStorage
     - No editing history or sharing
     - Under 1.5KB"
```

---

## Power Patterns

### Pattern A: Specification by Example

```
I want a widget like this:
[Show the widget you want]

Features to include:
- [List specific features]

Don't include:
- [Explicitly exclude features]
```

**Why it works**: Claude sees exact expectations

---

### Pattern B: Iterative Refinement

```
First request: "Create a simple calculator"
Claude generates: Basic 4-operation calculator

Follow-up: "Now add percentage and square root buttons"
Claude updates: Adds two new operations, keeps size under limit

Follow-up: "Change the button layout from grid to single row"
Claude refactors: Layout changes, same functionality
```

**Why it works**: Claude maintains context across messages

---

### Pattern C: Constraint Negotiation

```
You: "I want a real-time chart that updates every second"
Claude: "That'll be hard in <2KB. Here are options:
  1. Static chart with preset data
  2. Canvas-based simple line chart
  3. Larger widget (3-4KB) with full chart.js"
You: "Let's do option 2"
Claude: Generates efficient canvas-based chart
```

**Why it works**: Claude explains tradeoffs upfront

---

## Token Cost Analysis

| Widget Type | Generate | Per Update | Total/10 uses |
|-----------|----------|-----------|-------|
| Simple calculation | 1,500-2,000 | ~0 | 1,500-2,000 |
| Data display (no data) | 1,200-1,800 | ~500 (data) | 6,200-7,800 |
| Complex widget | 3,000-4,000 | ~0 | 3,000-4,000 |
| Widget + iterate 3x | 2,000 + (500×3) | ~0 | 3,500 |

**Pro tip**: Generate once, iterate on the same widget (cheaper than regenerating)

---

## Advanced: Chain-of-Thought Prompting

When Claude makes mistakes, ask it to explain:

```
That widget doesn't work. Let me describe what's happening:
[Describe the bug]

I think the issue is [your hypothesis]. Can you fix it by:
- [Specific code change or approach]
```

Claude will:
1. Understand your reasoning
2. Apply the fix correctly
3. Avoid the same mistake in future widgets

---

## Checklist: Before You Ask

- [ ] **Defined goal**: "Create a [widget] that does [what]"
- [ ] **Specified structure**: Input fields, layout, output
- [ ] **Set constraints**: Size <2KB, no libraries, responsive
- [ ] **Given example**: Showed similar widget or expected UX
- [ ] **Excluded features**: What to NOT include
- [ ] **Chosen interaction**: Instant updates, button-driven, etc.

✅ If all checked → Claude generates working widget first try
❌ If some unchecked → Might need iteration or refinement

---

## Example: Complete Request

```
Create a compound interest calculator widget with these requirements:

What it does:
- User enters: principal amount, annual interest rate, number of years, compounding frequency
- Widget calculates: final amount, total interest earned
- Shows: Both values prominently, updates instantly

Layout:
- 4 stacked input fields: principal, rate, years, frequency dropdown
- Large display showing final amount (emphasized)
- Smaller display showing interest earned
- All within 300px width (mobile) to 500px (desktop)

Styling:
- Use CSS variables: --primary (for numbers), --text-primary (text), --surface-1 (background), --radius (borders)
- Light card-based design
- Numbers in large, bold font

Constraints:
- Vanilla JavaScript only, no libraries
- Keep under 2KB total code
- No API calls or external resources
- All calculation happens in browser

Example:
- User enters: $10,000 principal, 5% rate, 10 years, monthly compound
- Widget shows: Final amount: $16,470.09, Interest earned: $6,470.09

Don't include:
- Charts or visualization
- Amortization schedule
- Comparison features
- Extra calculations beyond what I specified
```

**Result**: Claude generates exactly the widget you described, works first try.

