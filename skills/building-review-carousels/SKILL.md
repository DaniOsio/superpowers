---
name: building-review-carousels
description: "Use when building a testimonials or reviews section with continuous scrolling carousels. Creates dual counter-rotating marquee rows driven by pure CSS animation, with click-to-expand modal, hover pause, and edge fade masks."
---

# Building Review Carousels

Two rows of review cards scrolling in opposite directions — top row left, bottom row right. Pure CSS `@keyframes` animation (no JS scroll logic). Click a card to open a centered modal with the full review; both carousels pause. Click off or press Escape to close and resume.

## When to Use

- Testimonials section with 8+ reviews to showcase
- You want continuous, ambient motion that draws the eye
- Reviews should feel abundant (not just 3 static cards)
- The section needs interactivity beyond static text

## Architecture

```
┌──────────────────────────────────────────────┐
│  Header (container-custom)                   │
│  Eyebrow · Headline · Body copy              │
├──────────────────────────────────────────────┤
│ ◁◁◁ Row 1 — scrolls LEFT ◁◁◁◁◁◁◁◁◁◁◁◁◁◁◁│  full-bleed
│ ▷▷▷ Row 2 — scrolls RIGHT ▷▷▷▷▷▷▷▷▷▷▷▷▷▷▷│  full-bleed
├──────────────────────────────────────────────┤
│  Aggregate row (container-custom)            │
│  Google 4.9★ · Diamond · HomeAdvisor · BBB   │
│  Badge CTA                                   │
└──────────────────────────────────────────────┘

  Click card → centered modal overlay (z-50)
  Both rows pause while modal is open
```

## Core Technique: CSS Marquee with Triple Rendering

### Why Triple, Not Double

Each review set is rendered **three times** in a flex row. The CSS animation translates the track by exactly `-33.333%` (one set width). When the animation loops, the position resets seamlessly because sets 1 and 3 are identical — the viewer never sees a seam.

```
Track:  [Set 0] [Set 1] [Set 2]    ← identical content × 3
         ────────────────────────→
Animation: translateX(0) → translateX(-33.333%)
           When it resets to 0, Set 1 is where Set 0 was — seamless
```

Double rendering (2 sets, -50%) can show visible gaps during fast scrolling or on wide viewports. Triple eliminates this entirely.

### Keyframes

Injected once via `<style dangerouslySetInnerHTML>`:

```css
@keyframes marquee-left {
  0%   { transform: translateX(0); }
  100% { transform: translateX(-33.333%); }
}
@keyframes marquee-right {
  0%   { transform: translateX(-33.333%); }
  100% { transform: translateX(0); }
}
@keyframes fadeIn {
  from { opacity: 0; }
  to   { opacity: 1; }
}
@keyframes scaleIn {
  from { opacity: 0; transform: scale(0.92); }
  to   { opacity: 1; transform: scale(1); }
}
```

`marquee-right` is the reverse: starts at `-33.333%` and animates to `0`.

### Speed Calculation

Measure one card set's width, then derive duration from a constant speed:

```typescript
const CARD_GAP = 16;
const SPEED_PX_PER_SEC = 60; // tune this — 35 = leisurely, 60 = brisk

// Measure first card set
useEffect(() => {
  const firstSet = trackRef.current?.querySelector("[data-set='0']") as HTMLElement;
  if (firstSet) setTrackWidth(firstSet.scrollWidth + CARD_GAP);
}, [reviews]);

const duration = trackWidth > 0 ? trackWidth / SPEED_PX_PER_SEC : 30;
```

This keeps scroll speed consistent regardless of card count or viewport width.

## Component Structure

### Data Shape

```typescript
type Review = {
  quote: string;
  reviewer: string;
  city: string;
  service: string;
  rating: number;
  source: "Google" | "Diamond Certified" | "Yelp";
};
```

Split reviews into two arrays — 5-8 per row works well.

### MarqueeRow

```tsx
function MarqueeRow({
  reviews,
  direction,   // "left" | "right"
  modalOpen,   // pause when ANY modal is open
  onSelectReview,
}: {
  reviews: Review[];
  direction: "left" | "right";
  modalOpen: boolean;
  onSelectReview: (review: Review) => void;
}) {
  const trackRef = useRef<HTMLDivElement>(null);
  const [trackWidth, setTrackWidth] = useState(0);
  const [isHovered, setIsHovered] = useState(false);

  // Pause on hover OR when modal is open
  const paused = isHovered || modalOpen;

  const animationStyle: React.CSSProperties = trackWidth > 0
    ? {
        animationName: direction === "left" ? "marquee-left" : "marquee-right",
        animationDuration: `${trackWidth / SPEED_PX_PER_SEC}s`,
        animationTimingFunction: "linear",
        animationIterationCount: "infinite",
        animationPlayState: paused ? "paused" : "running",
      }
    : {};

  return (
    <div className="relative overflow-hidden"
         onMouseEnter={() => setIsHovered(true)}
         onMouseLeave={() => setIsHovered(false)}>

      {/* Edge fade masks */}
      <div className="pointer-events-none absolute inset-y-0 left-0 w-16 z-10"
           style={{ background: "linear-gradient(90deg, var(--background) 0%, transparent 100%)" }} />
      <div className="pointer-events-none absolute inset-y-0 right-0 w-16 z-10"
           style={{ background: "linear-gradient(270deg, var(--background) 0%, transparent 100%)" }} />

      {/* Scrolling track — triple rendered */}
      <div ref={trackRef} className="flex py-2"
           style={{ gap: CARD_GAP, ...animationStyle }}>
        {[0, 1, 2].map((setIdx) => (
          <div key={setIdx} data-set={setIdx} className="flex shrink-0"
               style={{ gap: CARD_GAP }}>
            {reviews.map((review, i) => (
              <CarouselCard key={`${setIdx}-${i}`} review={review}
                            onSelect={() => onSelectReview(review)} />
            ))}
          </div>
        ))}
      </div>
    </div>
  );
}
```

**Edge fade masks** — `w-16` gradient divs on left and right edges that fade cards into the background color. Uses `var(--background)` so it matches the section automatically.

### CarouselCard

Compact card, always truncated (`line-clamp-3`). Click fires `onSelect`:

```tsx
function CarouselCard({ review, onSelect }: { review: Review; onSelect: () => void }) {
  return (
    <article role="button" tabIndex={0} onClick={onSelect}
             onKeyDown={(e) => { if (e.key === "Enter" || e.key === " ") onSelect(); }}
             className="shrink-0 w-[280px] sm:w-[320px] cursor-pointer rounded-2xl
                        border border-neutral-200 bg-white hover:border-primary
                        hover:shadow-md transition-all">
      <div className="p-5 sm:p-6">
        {/* Stars + source badge */}
        {/* Quote — line-clamp-3 */}
        {/* Footer: avatar initials · name · service · city */}
      </div>
    </article>
  );
}
```

**Card width is fixed** (`w-[280px] sm:w-[320px]`) — not responsive. This ensures consistent spacing and animation speed across viewports.

### Initials Avatar

Generate from reviewer name — no photos needed:

```tsx
<div className="w-8 h-8 rounded-full flex items-center justify-center text-xs font-semibold"
     style={{ background: "color-mix(in oklab, var(--color-primary) 14%, white)" }}>
  {review.reviewer.split(" ").map(p => p[0]).join("")}
</div>
```

## Click-to-Expand Modal

### State Management

Track the selected `Review` object (not just an ID) so the modal can render immediately:

```typescript
const [expandedReview, setExpandedReview] = useState<Review | null>(null);
const modalOpen = expandedReview !== null;
```

Pass `modalOpen` to both `MarqueeRow` components — when true, both rows pause via `animationPlayState: "paused"`.

### ExpandedReview Component

Centered overlay with backdrop blur:

```tsx
function ExpandedReview({ review, onClose }: { review: Review; onClose: () => void }) {
  // Close on Escape
  useEffect(() => {
    const handler = (e: KeyboardEvent) => { if (e.key === "Escape") onClose(); };
    document.addEventListener("keydown", handler);
    return () => document.removeEventListener("keydown", handler);
  }, [onClose]);

  return (
    <div className="fixed inset-0 z-50 flex items-center justify-center p-4"
         onClick={onClose}>
      {/* Backdrop */}
      <div className="absolute inset-0 bg-black/40 backdrop-blur-sm
                      animate-[fadeIn_200ms_ease-out]" aria-hidden />

      {/* Card — stops click propagation */}
      <article className="relative w-full max-w-lg rounded-2xl bg-white shadow-2xl
                          animate-[scaleIn_250ms_ease-out] p-7 sm:p-9"
               onClick={(e) => e.stopPropagation()} role="dialog">
        {/* Close X button (top-right) */}
        {/* Quote icon */}
        {/* Stars (larger — w-5 h-5) */}
        {/* Full quote (no line-clamp) */}
        {/* Reviewer footer: larger avatar (w-12 h-12) · name · service · source */}
      </article>
    </div>
  );
}
```

**Close triggers:** backdrop click, X button, Escape key.

**Animations:** Backdrop fades in (200ms), card scales in from 0.92 (250ms) — defined in the `@keyframes` injected with the marquee styles.

## Pause Behavior

Three pause triggers, all via `animationPlayState`:

| Trigger | Scope | Implementation |
|---------|-------|----------------|
| **Hover** on a row | That row only | `isHovered` state via `onMouseEnter`/`onMouseLeave` |
| **Modal open** | Both rows | `modalOpen` prop passed from parent |
| **Combined** | `paused = isHovered \|\| modalOpen` | Single boolean drives `animationPlayState` |

CSS `animation-play-state: paused` freezes the animation in place — no jump or reset when it resumes.

## Aggregate Row & Badge CTA

Below the carousels, add a stats grid and badge call-to-action:

### Aggregate Row

```tsx
<div className="grid grid-cols-2 lg:grid-cols-4 gap-px rounded-2xl overflow-hidden
                border border-neutral-200 bg-neutral-200">
  {aggregates.map(a => (
    <div className="bg-white p-6">
      <p className="eyebrow text-neutral-500">{a.label}</p>
      <p className="font-display font-bold text-2xl">{a.value}</p>
      <p className="text-xs text-neutral-500">{a.detail}</p>
    </div>
  ))}
</div>
```

The `gap-px bg-neutral-200` trick creates 1px divider lines between cells.

### Badge CTA

```tsx
<div className="flex items-center justify-between rounded-2xl bg-neutral-50 p-6 lg:p-8">
  <div className="flex items-center gap-5">
    <Image src="/logos/badges/badge.jpg" alt="Badge" width={120} height={54} />
    <div>
      <p className="font-semibold">Highest in Quality, 12 consecutive years</p>
      <p className="text-sm text-neutral-600">Independently rated by 199 neighbors.</p>
    </div>
  </div>
  <Link href="/reviews" className="rounded-full bg-neutral-900 text-white px-6 py-3">
    Read all reviews <ArrowRight />
  </Link>
</div>
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Double-rendering cards (2 sets) | Use triple (3 sets) — double shows gaps on wide viewports |
| JS-driven scroll (requestAnimationFrame) | Use CSS `@keyframes` — smoother, zero JS per frame, GPU-accelerated |
| Fixed animation duration for all viewports | Calculate duration from measured `trackWidth / speed` — keeps speed consistent |
| Tracking expanded state by ID string | Track the full `Review` object — modal can render without a lookup |
| Only pausing the clicked row's carousel | Pass `modalOpen` to ALL rows — both should freeze when modal is up |
| No edge fade masks | Cards abruptly appear/vanish at edges — add gradient masks matching section bg |
| Cards have responsive width | Keep width fixed (`w-[280px] sm:w-[320px]`) — responsive widths break animation math |
| Missing Escape handler | Always add `keydown` listener for Escape on the modal |
| Modal `onClick` closes when clicking card text | Add `e.stopPropagation()` on the modal card element |

## Quick Reference

```
Animation:     CSS @keyframes, translateX(0 → -33.333%), linear, infinite
Triple render: [Set 0][Set 1][Set 2] — seamless loop at 33.333%
Speed:         trackWidth / SPEED_PX_PER_SEC = duration in seconds
Pause:         animationPlayState: paused (hover OR modal open)
Edge masks:    w-16 gradient divs, var(--background) → transparent
Card width:    fixed 280px / 320px (not responsive)
Modal:         fixed inset-0 z-50, backdrop bg-black/40 backdrop-blur-sm
Animations:    fadeIn 200ms (backdrop), scaleIn 250ms (card)
Close:         backdrop click + X button + Escape key
State:         expandedReview: Review | null (not string ID)
Layout:        carousels full-bleed, header + aggregates in container
```
