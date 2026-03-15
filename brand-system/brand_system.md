# opad.me — Brand System for Lovable
### Design Tokens & UI Rules · MVP · v1.1
### Target: **Lovable** · March 2026 · `CONFIDENTIAL`

---

## ⚡ TL;DR for Lovable

opad.me is a **calm observation system**, not a habit tracker.  
The visual identity must communicate: *"you are watching yourself, not being watched."*  
Dark teal surfaces. Desaturated graph lines. Generous space. No rewards. No pressure.

> Every design decision must serve one purpose: **make the trajectory visible without judgment.**

---

## 1. Product Mood & Aesthetic Direction

| Dimension | Direction |
|---|---|
| Tone | Calm · Neutral · Contemplative |
| Feel | Personal journal meets observatory dashboard |
| Visual texture | Dark, deep, slightly underwater — like observing something from a distance |
| NOT | Fitness app, SaaS dashboard, gamified tracker, productivity tool |

**One sentence to keep in mind when generating UI:**  
*"This interface is a quiet room where you can observe your own patterns."*

---

## 2. Color System

### 2.1 Global Tokens

```css
/* === BACKGROUNDS === */
--color-bg-primary:    #0F2F33;  /* App background — dominant, 90% of surfaces */
--color-bg-secondary:  #1F4A50;  /* Cards, graph containers, elevated surfaces */
--color-bg-neutral:    #F4F4F2;  /* Modals, light-on-dark overlays */

/* === TEXT === */
--color-text-primary:   #EAEAEA; /* All primary text on dark surfaces */
--color-text-secondary: #B9C0C1; /* Secondary labels, captions, axis ticks */

/* === ACCENT === */
--color-accent:         #E24A4A; /* Errors, destructive actions, validation only */
                                  /* ⚠️ NEVER use for missed days or negative trend */
```

### 2.2 Graph Color Tokens

```css
/* === GRAPH LINES — all intentionally desaturated === */
--color-graph-positive: #7DA3A0; /* Upward trajectory (positive 7-day slope) */
--color-graph-neutral:  #8C9496; /* Flat or stable trajectory */
--color-graph-decline:  #BFA37A; /* Declining trajectory — warm, never red */
```

### 2.3 Graph Color Logic

The line color is computed from the **7-day rolling slope** of cumulative score data:

```typescript
function getLineColor(slope: number): string {
  if (slope > 0.1)  return '#7DA3A0'; // graph-positive
  if (slope < -0.1) return '#BFA37A'; // graph-decline
  return '#8C9496';                   // graph-neutral
}
```

### 2.4 Tailwind Color Usage

```typescript
// Primary background
bg-[#0F2F33]

// Card / container
bg-[#1F4A50]

// Primary text
text-[#EAEAEA]

// Secondary text
text-[#B9C0C1]

// Positive graph / interactive elements
text-[#7DA3A0]   bg-[#7DA3A0]   border-[#7DA3A0]

// Accent (errors only)
text-[#E24A4A]
```

### 2.5 Color Rules

| Rule | Constraint |
|---|---|
| Main background coverage | `#0F2F33` on ≥ 85% of every screen |
| Graph colors | Always desaturated — no pure greens or reds |
| Negative trajectory | Always `#BFA37A` (warm amber) — never red |
| Accent `#E24A4A` | Errors and destructive actions only |
| White or light surfaces | Only for modals and overlapping sheets |

---

### 2.6 Wordmark Tokens

```css
/* === WORDMARK / LOGO === */
--color-wordmark-opad:  #FFFFFF;  /* "opad" portion of wordmark */
--color-wordmark-dotme: #B5453A;  /* ".me" portion — terracotta brick */
```

**Struttura del logo:**

```
opad  .me
────  ───
  ↑    ↑
  │    └── #B5453A (terracotta)
  └─────── #FFFFFF (bianco)
```

- **"opad"** → `#FFFFFF` (bianco)
- **"."** → `#B5453A` (terracotta)
- **"me"** → `#B5453A` (terracotta)

**Nota icona:** il logo finale prevede un'icona SVG (Uomo Vitruviano in cerchio) che sostituisce visivamente la "o" di "opad". L'asset SVG non è ancora disponibile — nel frattempo renderizzare il wordmark come testo `"opad.me"` con i colori sopra.

**Regole d'uso wordmark:**
- Il colore `#B5453A` è **esclusivamente del wordmark** — non usarlo in UI, pulsanti o grafici
- Su sfondo `#0F2F33` (nav/sidebar): usare il wordmark completo con i colori sopra
- Quando disponibile, l'icona SVG è sempre in bianco — mai colorata con i token di sistema

---

## 3. Typography

### 3.1 Font Stack

```css
font-family: 'Inter', 'SF Pro Display', 'Source Sans Pro', sans-serif;
```

> Lovable: import Inter from Google Fonts. Apply globally via `<html>` or Tailwind config.

### 3.2 Type Scale

```css
/* Title — page headers, area names */
font-size: 28px;  font-weight: 600;  color: #EAEAEA;
/* Tailwind: text-[28px] font-semibold text-[#EAEAEA] */

/* Section — section labels, card headers */
font-size: 18px;  font-weight: 500;  color: #EAEAEA;
/* Tailwind: text-[18px] font-medium text-[#EAEAEA] */

/* Body — default interface text */
font-size: 16px;  font-weight: 400;  color: #EAEAEA;
/* Tailwind: text-base font-normal text-[#EAEAEA] */

/* Secondary — supporting info, sub-labels */
font-size: 14px;  font-weight: 400;  color: #B9C0C1;
/* Tailwind: text-sm font-normal text-[#B9C0C1] */

/* Caption — axis ticks, helper text, pills */
font-size: 12px;  font-weight: 400;  color: #B9C0C1;
/* Tailwind: text-xs font-normal text-[#B9C0C1] */
```

### 3.3 Typography Rules

- No bold text for emphasis in body copy — use hierarchy instead
- Never use uppercase for labels (too aggressive for this tone)
- Line height: 1.5 for body, 1.2 for titles

---

## 4. Spacing System

Base unit: **8px**.

```
space-xs:  4px   → gap-1     p-1
space-sm:  8px   → gap-2     p-2
space-md:  16px  → gap-4     p-4
space-lg:  24px  → gap-6     p-6
space-xl:  32px  → gap-8     p-8
space-xxl: 48px  → gap-12    p-12
```

### Spacing Rules

- **Generous negative space is a product value** — tight layouts contradict the product philosophy
- Max 4 visible components per screen at any time
- Cards must breathe: always `p-4` (16px) internal padding minimum
- Vertical rhythm: `gap-4` between major sections minimum

---

## 5. Layout System

### 5.1 Layout Tokens

```
Graph height:         50–60vh   (graph is the dominant element)
Container padding:    16px      (px-4 in Tailwind)
Card border-radius:   12px      (rounded-xl in Tailwind)
Tap target minimum:   44×44px   (min-h-[44px] + min-w-[44px])
Max width (mobile):   428px
```

### 5.2 Mobile-First Structure

Primary layout order (top → bottom) for every main screen:

1. **Header** — minimal, 56px, title + optional icon
2. **Graph / Primary visual** — 50–60vh
3. **Context / Secondary info** — condensed
4. **Quick action** — single CTA, full-width

### 5.3 Bottom Navigation Bar

```
Height:      56px
Background:  bg-[#0F2F33]
Border top:  border-t border-[#1F4A50]
Icons:       Lucide, outline style, 24px
Active:      text-[#7DA3A0]
Inactive:    text-[#B9C0C1]
Tabs:        Home · Areas · Finance · Settings
Tap target:  flex-1, min-h-[44px]
```

---

## 6. Graph System

### 6.1 Graph Tokens

```
Library:            Recharts
Component:          <LineChart> with <Line type="monotone">
Line width:         strokeWidth={2}
Line curve:         type="monotone" (smooth, never angular)
Animation:          isAnimationActive={true}, animationDuration={300}
Grid:               <CartesianGrid> horizontal only, opacity-10, #EAEAEA
Y-axis:             hidden by default (hide={true})
X-axis ticks:       12px, #B9C0C1
Projection line:    strokeDasharray="4 4" (Finance screen only)
```

### 6.2 Graph Container

```
bg-[#1F4A50] rounded-xl p-4 h-[55vh]
```

### 6.3 Graph Principles

- Graphs must never look like analytics dashboards
- No data point dots unless tapped/focused
- Grid lines should barely be visible — context only
- X-axis shows date ticks only (no values on Y-axis by default)
- Smooth curves create the feeling of organic observation, not rigid tracking

### 6.4 Recharts Implementation Reference

```tsx
<LineChart data={data} margin={{ top: 8, right: 8, bottom: 0, left: 0 }}>
  <CartesianGrid
    strokeDasharray="0"
    horizontal={true}
    vertical={false}
    stroke="#EAEAEA"
    strokeOpacity={0.1}
  />
  <XAxis
    dataKey="date"
    tick={{ fill: '#B9C0C1', fontSize: 12 }}
    axisLine={false}
    tickLine={false}
  />
  <YAxis hide={true} />
  <Line
    type="monotone"
    dataKey="score"
    stroke={lineColor}   // computed from slope
    strokeWidth={2}
    dot={false}
    isAnimationActive={true}
    animationDuration={300}
    animationEasing="ease-in-out"
  />
</LineChart>
```

---

## 7. Motion System

### 7.1 Motion Tokens

```
motion-fast:   200ms ease-in-out
motion-normal: 300ms ease-in-out
motion-slow:   400ms ease-in-out
```

### 7.2 Framer Motion Usage

```tsx
// Screen transitions
<motion.div
  initial={{ opacity: 0, y: 12 }}
  animate={{ opacity: 1, y: 0 }}
  transition={{ duration: 0.3, ease: 'easeInOut' }}
>

// Check-in button state change
<motion.button
  animate={{ opacity: isCompleted ? 0.8 : 1 }}
  transition={{ duration: 0.2 }}
>

// Time range selector pill (shared layout)
<motion.div layoutId="activeRange" transition={{ duration: 0.2, ease: 'easeInOut' }} />
```

### 7.3 Motion Rules

| Allowed | Forbidden |
|---|---|
| Graph line update animation (300ms) | Confetti / particle effects |
| Button state transition | Bounce or spring on success |
| Screen fade-in | Celebration pop-ups |
| Skeleton shimmer (pulse) | Any reward animation |
| Layout pill transition | Flash or glow effects |

---

## 8. Component Library

### 8.1 `<TrajectoryCard>`

The primary UI unit. One per life area. Occupies ≈ 55vh.

```
Container:     bg-[#1F4A50] rounded-xl p-4
Area name:     top-left, text-[18px] font-medium text-[#EAEAEA]
Type badge:    top-right, <AreaTypePill>
Graph:         Recharts LineChart, h-full, line color from slope
Check-in CTA:  bottom, full-width, <CheckInButton>
```

### 8.2 `<CheckInButton>`

```
Idle:       border border-[#7DA3A0] text-[#EAEAEA] bg-transparent
            Label: "Log today"

Completed:  bg-[#7DA3A0]/20 text-[#7DA3A0] border border-[#7DA3A0]
            Label: "Observed"

Loading:    opacity-50 cursor-not-allowed + inline spinner

All states: min-h-[44px] w-full rounded-lg text-[16px] font-medium
```

> ✅ On completion: button transitions to "Observed" + graph animates.  
> ❌ No toast. No popup. No positive message. The graph is the only feedback.

### 8.3 `<TimeRangeSelector>`

```
Options:  30d · 90d · 365d
Active:   bg-[#1F4A50] text-[#EAEAEA] rounded-full px-4 py-1
Inactive: text-[#B9C0C1] px-4 py-1
Motion:   Framer Motion layoutId="activeRange" for sliding pill
```

### 8.4 `<AreaTypePill>`

```
health:   bg-[#7DA3A0]/20  text-[#7DA3A0]  border border-[#7DA3A0]/40
study:    bg-[#8C9496]/20  text-[#B9C0C1]  border border-[#8C9496]/40
reduce:   bg-[#BFA37A]/20  text-[#BFA37A]  border border-[#BFA37A]/40
finance:  bg-[#1F4A50]     text-[#EAEAEA]  border border-[#7DA3A0]/30

All:      rounded-full px-3 py-0.5 text-[12px] font-medium border
```

### 8.5 `<CalendarHeatmap>`

```
Layout:     Single horizontal row, overflow-x-auto, gap-0.5
Cell size:  w-7 h-7 rounded-sm

completed:  bg-[#7DA3A0]/60
missed:     bg-[#BFA37A]/40
future:     bg-[#1F4A50]/30
```

> ⚠️ No streak counter. No "best streak" label. No consecutive-day celebration.

### 8.6 Empty States

| Context | Icon | Copy | CTA |
|---|---|---|---|
| Dashboard (no areas) | `Eye` Lucide 48px `#7DA3A0` | "What do you want to observe?" | "Add your first area" |
| Area Detail (no data) | `TrendingUp` muted | "Keep observing. Your trajectory takes shape over days." | — |
| Finance (no data) | `BarChart2` muted | "Log your first finance check-in to see your trend." | — |

### 8.7 Loading States

```
Dashboard skeleton:  bg-[#1F4A50] animate-pulse rounded-xl h-[55vh]
Graph update:        Framer Motion 300ms transition on data change
Area creation:       Inline spinner in button only — no full-screen loader
```

---

## 9. Copy & Tone of Voice

### 9.1 Brand Voice

**Calm · Neutral · Observational**  
The app observes. It never evaluates.

### 9.2 Allowed Vocabulary

```
trend · trajectory · direction · pattern · over time
observe · notice · see
your data · your trajectory · over the past N days
```

### 9.3 Forbidden Vocabulary

```
success · achievement · failure · win
streak · chain · combo · record
great job · well done · keep it up · you're doing amazing
goal · target · challenge · level · reward
don't break the chain · you missed · you failed
let's go · start your journey · you're all set
```

### 9.4 Key UI Strings

| Location | String |
|---|---|
| App tagline | "Observe your direction." |
| Onboarding step 2 header | "What do you want to observe?" |
| Area frequency label | "How many days per week?" |
| Add area CTA | "Start observing" |
| Check-in button — idle | "Log today" |
| Check-in button — done | "Observed" |
| Dashboard empty state | "What do you want to observe?" |
| Finance projection label | "30-day estimate based on your current trend." |
| Score toggle | "Show trajectory score" |
| Delete account confirm | "This will permanently delete all your observation history. This cannot be undone." |

---

## 10. UI Anti-Patterns — NEVER implement

```
❌ Confetti or particle animations
❌ Badges, trophies, achievement popups
❌ Streak counters as primary UI
❌ "Great job!" / "Keep it up!" / "You're doing amazing!" messages
❌ Red color for missed days or negative trends
❌ Dense information layouts (> 4 components visible)
❌ Bright or saturated graph colors
❌ Score as primary display (default off)
❌ Stock imagery or illustration
❌ Generic SaaS white card layouts
❌ Push notification urgency ("Don't forget to log today!")
❌ Any visual that implies evaluation or performance
```

---

## 11. Accessibility

| Rule | Value |
|---|---|
| Minimum contrast ratio | 4.5:1 |
| All tap targets | ≥ 44 × 44px |
| Font scaling | Respect system settings (`rem`-based sizing) |
| Trend communication | slope + position + color (never color alone) |
| Focus indicators | Visible ring on all interactive elements |

---

## 12. Brand System Summary Card

> Quick reference for Lovable generation

```
Primary background:  #0F2F33
Card surface:        #1F4A50
Primary text:        #EAEAEA
Secondary text:      #B9C0C1
Accent (errors):     #E24A4A
Graph positive:      #7DA3A0
Graph neutral:       #8C9496
Graph decline:       #BFA37A
Wordmark "opad":     #FFFFFF  ← wordmark only
Wordmark ".me":      #B5453A  ← wordmark only, never in UI

Font:        Inter, 600/500/400
Radius:      12px (rounded-xl)
Motion:      300ms ease-in-out
Graph:       Recharts monotone, 2px line
Tap target:  min 44px

Mood:        Calm. Neutral. Observational.
NOT:         Gamified. Pressuring. Evaluative.
```

---

*opad.me Brand System v1.2 — Ready for Lovable generation*
