---
name: building-depth-heroes
description: "Use when building a hero section that needs a dramatic depth effect where landscape or foreground elements occlude the headline text. The technique layers a transparent cutout PNG above the headline so rocks, trees, buildings, or other foreground objects appear IN FRONT of the letters while the full scene sits behind them."
---

# Building Depth Heroes

A hero section where landscape elements (rocks, trees, rooftops, mountains) appear to crest up in front of the headline text, creating real photographic depth. The text feels embedded in the scene rather than floating on top of it.

## When to Use

- Hero needs a dramatic, immersive first impression
- You have a strong regional/landscape photo with clear foreground elements
- The headline is large enough (4rem+) for partial occlusion to look intentional
- Brand is place-based (local service company, resort, real estate, outdoor)

## The Depth Illusion

The effect comes from a **three-layer z-index sandwich**:

```
z-30  ┌─────────────────────────────────────┐  Eyebrow, CTAs, proof cards
      │ Sonoma & Marin · Family-Owned       │  (above everything)
      │                                      │
z-20  │    ▓▓▓▓▓▓▓   ▓▓▓▓▓                 │  Foreground cutout PNG
      │   ▓▓rocks▓▓ ▓▓cliff▓               │  (occludes headline)
      │  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓               │
      │                                      │
z-auto│     Seasons change.                  │  Headline text
      │     We don't.                        │  (NO z-index — default stacking)
      │                                      │
-z-10 │ ░░░░ gradient overlays ░░░░░░░░░░░░ │  Vignette + bottom fade
      │                                      │
-z-20 │ ▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒ │  Background photo (full scene)
      └─────────────────────────────────────┘
```

**The headline intentionally has NO z-index** — it sits at default stacking, sandwiched between the background (-z-20) and the foreground cutout (z-20). The rocks occlude the bottom of the letters. The UI controls (CTAs, subhead) sit at z-30, above everything.

## Image Preparation

You need **two images** from the same photo:

### 1. Background — Full Scene (`hero-bg.jpg`)
The complete photograph. Served as a standard `object-cover` background.

### 2. Foreground Cutout — Transparent PNG (`hero-foreground.png`)
The same photo with **sky, water, and distant scenery erased to alpha=0**. Only the foreground elements remain (rocks, cliffs, grass, trees, buildings, rooftops).

**How to create the cutout:**

| Method | Tool | Best For |
|--------|------|----------|
| Manual mask | Photoshop / GIMP | Complex edges, highest quality |
| AI removal | remove.bg / Photoshop AI | Quick, clean edges on distinct subjects |
| Luma key | After Effects | Video-based heroes |
| CSS clip-path | Browser | Simple geometric foregrounds only |

**Critical requirements:**
- **Same source photo** as the background — positioning must align exactly
- **Same dimensions** — both images must be identical pixel size
- **Both use `object-cover`** — browser scales them identically, so elements line up
- **Full opacity where foreground exists** — no semi-transparency, no gradients. The cutout must fully occlude the text where it overlaps.
- **`priority` loading** — both images are LCP candidates, load them eagerly

```tsx
{/* Background — full scene */}
<div className="absolute inset-0 -z-20" aria-hidden>
  <Image src={HERO_BG} alt="" fill priority sizes="100vw" className="object-cover" />
</div>

{/* Foreground cutout — rocks/landscape ONLY, sky = transparent */}
<div className="pointer-events-none absolute inset-0 z-20" aria-hidden>
  <Image src={HERO_FOREGROUND} alt="" fill priority sizes="100vw" className="object-cover" />
</div>
```

**Both use `absolute inset-0` + `object-cover`** — this guarantees pixel-perfect alignment at every viewport size. The browser crops and scales both images identically.

## Z-Index Stack

| Layer | z-index | Contains | Why |
|-------|---------|----------|-----|
| Background photo | `-z-20` | Full scene JPG | Behind everything |
| Overlay gradients | `-z-10` | Vignette, bottom fade | Darken edges for text legibility |
| **Headline** | **none** | `h1` text | **Deliberately unstyled — sits at default so foreground can occlude it** |
| Foreground cutout | `z-20` | Transparent PNG | Rocks/trees paint over headline letters |
| UI controls | `z-30` | Eyebrow, subhead, CTAs, badges | Must appear above the foreground |

**The headline must NOT have a z-index.** If you add `z-10` or `z-30` to the headline, it jumps above the foreground cutout and the depth illusion breaks.

## Overlay Gradients

Two overlays at `-z-10` handle text legibility without masking the depth effect:

### Top Vignette — Eyebrow Readability
```css
linear-gradient(180deg,
  rgba(8,12,9,0.55) 0%,    /* dark at top for eyebrow */
  rgba(8,12,9,0.15) 30%,   /* fade to near-transparent */
  rgba(8,12,9,0.05) 55%,   /* headline zone — minimal overlay */
  rgba(8,12,9,0.55) 100%)  /* dark at bottom for subhead */
```

The headline zone (30-55%) is nearly transparent so the photo shows through clearly behind the text.

### Bottom Fade — Section Transition
```css
linear-gradient(180deg,
  transparent 0%,
  rgba(8,12,9,0.9) 70%,
  var(--color-surface-dark) 100%)
```

Fades the bottom edge into the next section's background color. Applied to a `h-56` div anchored to the bottom.

### Optional: Brand Glow
```css
radial-gradient(70% 60% at 50% 100%,
  rgba(40,186,79,0.18) 0%,   /* brand color */
  transparent 60%)
```

Subtle colored glow from the bottom — adds warmth without competing with the photo.

## Layout Structure

Full-viewport hero using `min-h-screen` + `flex flex-col` with a spacer:

```tsx
<section className="relative isolate overflow-hidden text-white">
  {/* BG, overlays, foreground layers (absolute positioned) */}

  <div className="container-custom relative pt-24 pb-16 min-h-screen flex flex-col">
    {/* Eyebrow — z-30, above foreground */}
    <p className="relative z-30 text-center">...</p>

    {/* Headline — NO z-index, occluded by foreground */}
    <h1 className="text-center font-display font-extrabold"
        style={{ fontSize: "clamp(4.5rem, 14vw, 14rem)", lineHeight: 0.92 }}>
      Seasons change.<br />
      <span className="text-[color:var(--color-primary-light)]">We don't.</span>
    </h1>

    {/* Flex spacer — pushes lower content down so rocks crest
        through headline at ~50-65% of viewport height */}
    <div className="flex-1" />

    {/* Lower cluster — z-30, above foreground */}
    <div className="relative z-30 grid grid-cols-12">
      {/* Subhead + CTAs (left) */}
      {/* Glass proof card (right) */}
    </div>

    {/* Value bar + Trust strip — z-30 */}
  </div>
</section>
```

The `flex-1` spacer is what positions the foreground rocks to crest through the headline. Adjust padding/spacing to control where the occlusion happens vertically.

## Headline Text Shadow

The headline needs a tight, solid shadow — not a soft glow:

```css
text-shadow: 0 2px 4px rgba(0,0,0,0.55), 0 0 1px rgba(0,0,0,0.5);
```

**Why two shadows:**
- `0 2px 4px` — subtle drop shadow for depth
- `0 0 1px` — hairline outline that prevents letters from looking translucent where the photo shows through

Soft/large text shadows (`0 4px 20px`) make the text look ghostly. Keep it tight.

## GSAP Entrance Animation

Staggered reveal timeline (~1.5s total):

```typescript
const tl = gsap.timeline({ defaults: { ease: "power3.out" } });

tl.from(bgRef.current,       { opacity: 0, scale: 1.06, duration: 1.4, ease: "power2.out" }, 0)
  .from(eyebrowRef.current,  { opacity: 0, y: 12, duration: 0.5 }, 0.25)
  .from(headlineRef.current, { opacity: 0, y: 40, scale: 0.96, duration: 1.0 }, "-=0.25")
  .from(subheadRef.current,  { opacity: 0, y: 20, duration: 0.6 }, "-=0.55")
  .from(ctaRef.current,      { opacity: 0, y: 18, duration: 0.5 }, "-=0.4")
  .from(ratingRef.current,   { opacity: 0, y: 12, duration: 0.4 }, "-=0.35")
  .from(proofRef.current,    { opacity: 0, y: 16, duration: 0.6 }, "-=0.4");
```

**Key details:**
- Background scales from `1.06` → `1.0` (subtle zoom-in settle)
- Headline scales from `0.96` → `1.0` (punchy entrance)
- Negative offsets (`"-=0.25"`) overlap animations for fluid feel
- **Always check `prefers-reduced-motion`** — set all to final state if reduced

```typescript
const prefersReduced = window.matchMedia("(prefers-reduced-motion: reduce)").matches;
if (prefersReduced) {
  gsap.set([...allRefs], { opacity: 1, y: 0, scale: 1, clearProps: "all" });
  return;
}
```

## Glass Proof Card

Frosted glass card anchored bottom-right for social proof:

```tsx
<div className="rounded-2xl bg-black/45 backdrop-blur-md ring-1 ring-white/20 p-5">
  <p className="eyebrow text-[color:var(--color-primary-light)]">
    Diamond Certified · 12 yrs
  </p>
  <p className="text-white">
    Highest in Quality, rated by <strong>199 of your neighbors</strong>
  </p>
  <div className="mt-4 pt-4 border-t border-white/20 flex items-center gap-3">
    <Image src="/logos/badges/badge.jpg" alt="Badge" width={120} height={54} />
    <span className="text-xs text-neutral-300">Independently verified</span>
  </div>
</div>
```

Recipe: `bg-black/45` + `backdrop-blur-md` + `ring-1 ring-white/20`.

## Value Bar

Grid of key selling points, frosted glass:

```tsx
<div className="grid grid-cols-2 md:grid-cols-4 gap-px rounded-2xl overflow-hidden
                ring-1 ring-white/15 bg-white/5 backdrop-blur-md">
  {items.map(item => (
    <div className="bg-black/35 px-5 py-5">
      <p className="font-semibold text-white">{item.label}</p>
      <p className="text-xs text-neutral-300">{item.sub}</p>
    </div>
  ))}
</div>
```

The `gap-px` + `bg-white/5` creates subtle 1px divider lines between cells.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Headline has z-index | Remove it — headline MUST be at default stacking for foreground to occlude it |
| Foreground PNG has soft/gradient edges | Edges must be fully opaque or fully transparent — gradients create ghosting |
| Background and foreground are different crops | Must be same source photo, same dimensions, both `object-cover` |
| Foreground not loaded eagerly | Add `priority` to both `<Image>` components — LCP depends on it |
| Text shadow too soft/wide | Use tight shadow (`0 2px 4px`) — wide glows make text look transparent |
| Lower content hidden behind foreground | CTAs, subhead, badges all need `z-30` explicitly |
| No reduced-motion fallback | Must check `prefers-reduced-motion` and skip all GSAP animations |
| Bottom of hero doesn't blend into next section | Add bottom-fade gradient div transitioning to next section's bg color |

## Quick Reference

```
Two images:  hero-bg.jpg (full scene) + hero-foreground.png (cutout, transparent sky)
Both:        absolute inset-0, object-cover, priority, same source photo
Z-stack:     bg(-z-20) → overlays(-z-10) → headline(NO z) → cutout(z-20) → UI(z-30)
Headline:    clamp(4.5rem, 14vw, 14rem), lineHeight 0.92, tight textShadow
Layout:      min-h-screen flex-col, flex-1 spacer pushes CTAs below rock line
Entrance:    GSAP timeline ~1.5s, bg scale 1.06→1, headline scale 0.96→1
Glass card:  bg-black/45 backdrop-blur-md ring-1 ring-white/20
Value bar:   grid gap-px bg-white/5 backdrop-blur-md, cells bg-black/35
Motion:      always check prefers-reduced-motion, gsap.set() if reduced
```
