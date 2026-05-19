---
name: building-mountain-card-stacks
description: "Use when building a services section, feature list, or stacked card layout that needs a distinctive visual treatment. Creates layered mountain-silhouette cards with hover-to-expand accordion, color gradient, and grain texture."
---

# Building Mountain Card Stacks

Layered cards clipped to mountain silhouette shapes, stacked with negative overlap so peaks from lower cards rise through valleys of upper cards — like a real ridgeline. Hover activates a card: silhouette fills solid, expansion panel drops down with description and CTA.

## When to Use

- Services section needs a distinctive, non-generic card layout
- You have 4-10 items that form a natural vertical stack
- The brand leans outdoorsy, natural, or regional (HVAC, landscaping, roofing)
- You want hover interactivity without JavaScript animation libraries

## Visual Anatomy

```
  ┌─────────────────────────────────────┐
  │          sky gradient bg            │
  │   ╱╲    ╱╲                          │
  │ ╱    ╲╱    ╲  ← silhouette peaks    │  Layer 1 (lightest)
  │ ████████████████████████████████████│
  │ █ Marker · Title                   █│  ← flat text zone (left 30%)
  │─╱╲──╱╲────────────────────────────── │
  │╱    ╲  ╲╱╲   ← next layer's peaks  │  Layer 2 (darker)
  │ ████████████████████████████████████│
  │ █ Marker · Title                   █│
  │──────────╱╲──╱╲──────────────────── │
  │         ╱    ╲  ╲                   │  Layer 3 (darkest)
  │ ████████████████████████████████████│
  │ █ Marker · Title                   █│
  │█████████████████████████████████████│
  │ ▓▓▓▓▓▓▓ dark base strip ▓▓▓▓▓▓▓▓▓ │
  └─────────────────────────────────────┘
```

## Core Mechanics

### 1. Silhouette Polygons

Each card has a CSS `clip-path: polygon(...)` defining its mountain shape. The polygon has two zones:

- **Left 30% (x: 0–30%)** — flat at the baseline `y`. This is the text zone — no peaks intrude, so labels are always readable.
- **Right 70% (x: 30–100%)** — peaks and valleys at varying heights above baseline.
- **Bottom edge** — always flat at `100%` (card bottom).

```typescript
const SILHOUETTES = [
  // Layer 1 — gentle rolling peaks
  "0% 100%, 0% 40%, 30% 40%, 42% 28%, 56% 40%, 68% 24%, 80% 40%, 90% 32%, 100% 40%, 100% 100%",
  // Layer 2 — twin peaks
  "0% 100%, 0% 40%, 30% 40%, 40% 22%, 50% 40%, 62% 18%, 74% 40%, 86% 30%, 100% 40%, 100% 100%",
  // Layer 3 — sharp Matterhorn peak
  "0% 100%, 0% 40%, 30% 40%, 44% 30%, 54% 12%, 64% 28%, 76% 40%, 88% 34%, 100% 40%, 100% 100%",
  // Add more for variety — cycle with index % length
];
```

**Key constant: baseline at `y=40%`.** The solid body of each card fills 40–100% of its height. This determines the overlap math.

### 2. Overlap & Layering Math

Cards stack with negative top margin so each card's solid body covers the valleys of the card above:

```
Card height:  200px
Baseline:     40% = 80px from top
Solid body:   120px (from baseline to bottom)
Overlap:      -80px (= baseline height — covers ALL valleys above)
Visible band: 120px per card (the solid body)
```

```typescript
const mountainHeight = 200;
const overlap = index === 0 ? 0 : -80; // first card has no overlap

// z-index: active card on top, otherwise natural stack order
const zIndex = isActive ? total + 2 : index + 1;
```

**Why -80px overlap works:** The previous card's peaks max out at `y≈8-12%` (about 16-24px from top). The next card's solid body starts at `y=40%` (80px). With -80px overlap, the next card's body starts exactly at the previous card's baseline — covering all its valleys while its own peaks rise through.

### 3. Color Gradient (oklch)

Each layer gets progressively darker, simulating atmospheric perspective — distant mountains are lighter, closer ones darker:

```typescript
const t = total > 1 ? index / (total - 1) : 0;
const lightness = 0.86 - t * 0.56; // 0.86 (sage) → 0.30 (forest)
const chroma = 0.06 + t * 0.06;    // 0.06 → 0.12 (richer color deeper)
const bg = `oklch(${lightness} ${chroma} 145)`; // 145 = green hue
```

Adapt the hue for different brands:
| Brand Feel | Hue | Range |
|------------|-----|-------|
| Nature / HVAC | 145 (green) | sage → forest |
| Ocean / Pool | 220 (blue) | sky → navy |
| Desert / Roofing | 55 (amber) | sand → brown |
| Neutral / Tech | 260 (gray) | silver → charcoal |

### 4. Auto-Contrast Text

Text color flips light/dark based on background lightness:

```typescript
const textIsLight = lightness < 0.55;
const labelColor = textIsLight
  ? "rgba(255,255,255,0.92)"
  : "rgba(20,40,25,0.95)";
const mutedColor = textIsLight
  ? "rgba(255,255,255,0.65)"
  : "rgba(20,40,25,0.65)";
```

Top cards (light bg) get dark text. Bottom cards (dark bg) get light text. The threshold at `0.55` ensures WCAG-compliant contrast automatically.

### 5. Hover Activation

Single accordion — only one card active at a time. Hover, click, or focus activates:

```tsx
const [activeIndex, setActiveIndex] = useState<number | null>(null);

// On the stack container — collapse all on mouse leave
<div onMouseLeave={() => setActiveIndex(null)}>
  {services.map((service, i) => (
    <MountainCard
      isActive={activeIndex === i}
      onActivate={() => setActiveIndex(i)}
    />
  ))}
</div>
```

On each card: `onMouseEnter`, `onClick`, and `onFocus` all call `onActivate`.

### 6. Active State Transition

When a card activates, two things animate simultaneously:

**a) Silhouette fills to solid rectangle:**
```typescript
const solidRect = "0% 0%, 100% 0%, 100% 100%, 0% 100%";
const activeClip = isActive ? solidRect : silhouette;
```
Applied via `transition-[clip-path] duration-500 ease-out` — the mountain shape smoothly morphs into a full rectangle.

**b) Expansion panel drops down:**
```tsx
<div
  className="absolute left-0 right-0 overflow-hidden
             transition-[max-height,opacity] duration-500 ease-out"
  style={{
    top: mountainHeight,       // drops below the card
    maxHeight: isActive ? 200 : 0,
    opacity: isActive ? 1 : 0,
    background: bg,            // same color = seamless
  }}
>
  <p>{service.description}</p>
  <Link href={service.href}>Learn more <ArrowRight /></Link>
</div>
```

The expansion panel uses the **same `bg` color** as the card, so the card and panel look like one continuous block.

### 7. Grain Texture

Adds a vintage/tactile feel — generated inline as an SVG data URI (no image file needed):

```tsx
<div
  className="absolute inset-0 pointer-events-none opacity-15
             mix-blend-overlay transition-[clip-path] duration-500 ease-out"
  style={{
    clipPath: `polygon(${activeClip})`,
    backgroundImage:
      'url("data:image/svg+xml;utf8,<svg viewBox=\'0 0 80 80\' xmlns=\'http://www.w3.org/2000/svg\'><filter id=\'n\'><feTurbulence baseFrequency=\'0.9\' numOctaves=\'2\' stitchTiles=\'stitch\'/><feColorMatrix values=\'0 0 0 0 0  0 0 0 0 0  0 0 0 0 0  0 0 0 1 0\'/></filter><rect width=\'100%\' height=\'100%\' filter=\'url(%23n)\' opacity=\'0.5\'/></svg>")',
  }}
  aria-hidden
/>
```

**Important:** The grain div must use the same `clipPath` as the card shape and transition in sync.

## Section Layout

Two-column grid: mountain stack on left (7 cols), sticky copy + CTAs on right (5 cols):

```
┌──────────────────────────────────────────────┐
│ [Mountain Stack - 7 cols]  [Copy - 5 cols]   │
│                            Eyebrow           │
│   ╱╲  ╱╲  Layer 1         Headline           │
│   █████████████            Body text          │
│  ╱╲╱╲  Layer 2             [CTA] [CTA]       │
│  ██████████████                               │
│  ╱╲ Layer 3                ── Hover a peak    │
│  ██████████████              to expand        │
│  ▓▓ base ▓▓▓▓▓                               │
└──────────────────────────────────────────────┘
```

The right column is `lg:sticky lg:top-28` so the copy stays visible while the user explores the stack.

**Sky background** sits behind the stack with its own rounded corners and shadow — the stack itself is NOT clipped by its container so expansion panels can overflow freely:

```tsx
<div className="absolute inset-0 rounded-2xl overflow-hidden ring-1 ring-black/5 shadow-xl"
  style={{
    background: "linear-gradient(180deg, oklch(0.95 0.03 HUE) 0%, oklch(0.85 0.05 HUE) 100%)",
  }}
/>
```

## Data Shape

```typescript
type Service = {
  icon: LucideIcon;    // icon component for optional use
  title: string;       // card headline
  description: string; // expansion panel body
  href: string;        // "Learn more" link
  marker: string;      // small uppercase tag (e.g. "Same-day", "Efficiency")
};
```

## Accessibility

- Each card: `role="button"`, `tabIndex={0}`, `aria-expanded={isActive}`
- `aria-controls` links card to its expansion panel `id`
- Keyboard: focus triggers activation (via `onFocus`)
- `onMouseLeave` on the container collapses all — no trapped states
- "Hover a peak to expand" hint text for discoverability

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Peaks in the text zone (left 30%) | Keep left edge flat at baseline — text is unreadable behind peaks |
| Overlap too shallow | Must equal baseline height (40% × card height) or valleys show sky gaps |
| Expansion panel clipped by container | Stack container must NOT have `overflow: hidden` — only the sky bg div does |
| Active card behind lower cards | Active card needs `zIndex: total + 2` to sit above all others |
| Grain doesn't follow clip-path | Grain div needs the SAME `clipPath` and transition as the shape div |
| Expansion panel color mismatch | Must use exact same `bg` variable — any difference breaks the seamless look |

## Quick Reference

```
Silhouette: polygon() with flat left (0-30%) + peaks right (30-100%)
Baseline:   y=40% — everything below is solid body
Height:     200px per card, -80px overlap (40% × 200)
Color:      oklch(L C H) — L: 0.86→0.30, C: 0.06→0.12
Text flip:  lightness < 0.55 → white text, else dark text
Active:     clip-path → solid rect, expansion max-height 0→200
z-index:    active = total+2, else = index+1
Grain:      inline SVG feTurbulence, opacity-15, mix-blend-overlay
Layout:     7/5 grid, right col sticky
```
