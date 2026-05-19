---
name: building-svg-service-maps
description: "Use when building a service area map for an HVAC, plumbing, roofing, or local-service client site. Covers GeoJSON-to-SVG projection, county outline extraction, city markers with hover tooltips, drive time estimates, and dual-scale map composition."
---

# Building SVG Service Area Maps

Build interactive, geographically accurate SVG maps showing a local service company's coverage area. Two-scale composition: state overview + zoomed county detail with city markers, highways, and hover-driven drive time tooltips.

## When to Use

- Client site needs a "Where we serve" or "Service Areas" section
- You want geographic accuracy (real county outlines) not hand-drawn shapes
- The business serves a multi-city region from a single HQ location
- You need hover interactivity (drive times, city info)

## Architecture

```
┌─────────────────────────────────────────────────┐
│  ServiceAreas section                           │
│  ┌──────────┐  ┌──────────────┐  ┌───────────┐ │
│  │ State    │  │ County Detail │  │ Legend +  │ │
│  │ Overview │──│ (clip-path   │  │ Address   │ │
│  │ (small)  │  │  to outline) │  │ Card      │ │
│  └──────────┘  └──────────────┘  └───────────┘ │
│  Disclaimer · City Lists · CTA                  │
└─────────────────────────────────────────────────┘
```

## Core Pipeline

### 1. Get County Boundaries (GeoJSON)

Source accurate county outlines from public GeoJSON:

```
codeforamerica/click_that_hood  → state-level county boundaries
johan/world.geo.json            → individual county files
US Census TIGER/Line             → highest resolution
```

Fetch the GeoJSON coordinates array for each county in the service area.

### 2. Project to SVG Coordinates

Convert `[lon, lat]` GeoJSON pairs to SVG `[x, y]`:

```typescript
// Constants — adjust per region
const LAT_CENTER = 38.4;          // center latitude of service area
const COS_LAT = Math.cos(LAT_CENTER * Math.PI / 180); // ~0.784 for NorCal
const SCALE = 319;                // px per degree — tune to fit viewBox
const LON_OFFSET = -123.5;       // shift origin (westmost lon)
const LAT_MAX = 39.0;             // northmost lat (SVG y=0)

function project(lon: number, lat: number): [number, number] {
  const x = (lon - LON_OFFSET) * SCALE * COS_LAT;
  const y = (LAT_MAX - lat) * SCALE;
  return [x, y];
}
```

**Critical:** Apply `cos(latitude)` correction or east-west distances will be stretched ~27% at typical US latitudes.

### 3. Simplify for Visual Angularity

Raw GeoJSON has 150-300+ points per county. At SVG rendering scale, 2-3px segments get anti-aliased into curves — looks rounded, not cartographic.

Apply **Douglas-Peucker (RDP) simplification** selectively:

| Edge Type | Epsilon | Result |
|-----------|---------|--------|
| Inland borders (straight survey lines) | 0.8–1.2 | Keep detail — already angular |
| Coastlines (organic curves) | 2.5–4.0 | Force 10-60px segments — visibly angular |
| State outline (overview map) | 3.0–5.0 | Simplified for small rendering |

```typescript
function rdpSimplify(points: [number, number][], epsilon: number): [number, number][] {
  if (points.length <= 2) return points;
  let maxDist = 0, maxIdx = 0;
  const [start, end] = [points[0], points[points.length - 1]];
  for (let i = 1; i < points.length - 1; i++) {
    const d = perpendicularDist(points[i], start, end);
    if (d > maxDist) { maxDist = d; maxIdx = i; }
  }
  if (maxDist > epsilon) {
    const left = rdpSimplify(points.slice(0, maxIdx + 1), epsilon);
    const right = rdpSimplify(points.slice(maxIdx), epsilon);
    return [...left.slice(0, -1), ...right];
  }
  return [start, end];
}
```

### 4. Build SVG Path Strings

Convert projected + simplified points to SVG `d` attribute:

```typescript
const outline = points.map((p, i) =>
  `${i === 0 ? 'M' : 'L'} ${p[0].toFixed(1)} ${p[1].toFixed(1)}`
).join(' ') + ' Z';
```

## Component Structure

### MapCity Type

```typescript
type MapCity = {
  name: string;
  x: number;          // projected SVG x
  y: number;          // projected SVG y
  isHQ?: boolean;     // headquarters marker
  driveTime: string;  // e.g. "15–20 min"
};
```

### SVG Layering Order (inside clip-path)

1. **Land fill** — neutral-50 per county path
2. **County tints** — primary color at 4-7% opacity per county
3. **Water features** — ellipses for bays, paths for inlets
4. **County boundary** — dashed line on shared border
5. **Highway** — double-stroke (wide neutral + narrow white)
6. **Route shield** — small rect with route number text
7. **County/water labels** — small caps, low opacity
8. **City markers** — circles with hover groups (see below)

Outside clip-path:
9. **County outline strokes** — crisp, unclipped
10. **Hover tooltip** — rendered last (on top)
11. **Scale indicator** — line with distance label

### Hover Tooltip with Drive Time Estimate

The full interaction: user hovers a city dot → dot grows, text bolds, and a tooltip card appears above the dot showing the city name and estimated drive time from HQ.

```tsx
const [hoveredCity, setHoveredCity] = useState<MapCity | null>(null);
const isHovered = (city: MapCity) => hoveredCity?.name === city.name;

{/* Each city wrapped in a hover group */}
<g
  onMouseEnter={() => setHoveredCity(city)}
  onMouseLeave={() => setHoveredCity(null)}
  className="cursor-pointer"
>
  {/* Invisible larger hit area — 3px dots are impossible to hover */}
  <circle cx={city.x} cy={city.y} r={14} fill="transparent" />

  {/* Visible dot — grows on hover (3→4.5), opacity increases */}
  <circle
    cx={city.x}
    cy={city.y}
    r={isHovered(city) ? 4.5 : 3}
    fill="var(--color-primary)"
    fillOpacity={isHovered(city) ? 1 : 0.65}
    style={{ transition: "r 0.15s ease, fill-opacity 0.15s ease" }}
  />

  {/* City name — bolds and darkens on hover */}
  <text
    x={city.x + 8}
    y={city.y + 3.5}
    fontSize={8}
    fontWeight={isHovered(city) ? 600 : 400}
    fill={isHovered(city)
      ? "var(--color-neutral-900)"
      : "var(--color-neutral-600)"}
    style={{ transition: "fill 0.15s ease" }}
  >
    {city.name}
  </text>
</g>

{/* Drive time tooltip — rendered LAST in SVG so it paints on top.
    Auto-positions above the dot, flips below if near top edge,
    clamps horizontally to stay inside the viewBox. */}
{hoveredCity && !hoveredCity.isHQ && (() => {
  const tipW = 100, tipH = 34;
  let tx = hoveredCity.x - tipW / 2;
  let ty = hoveredCity.y - tipH - 12;
  // Clamp horizontal — keep inside viewBox
  if (tx < 4) tx = 4;
  if (tx + tipW > viewBoxWidth - 4) tx = viewBoxWidth - tipW - 4;
  // Flip below if too close to top
  if (ty < 4) ty = hoveredCity.y + 16;

  return (
    <g style={{ pointerEvents: "none" }}>
      {/* Drop shadow */}
      <rect x={tx + 1} y={ty + 1} width={tipW} height={tipH}
            rx={6} fill="rgba(0,0,0,0.08)" />
      {/* Card background */}
      <rect x={tx} y={ty} width={tipW} height={tipH} rx={6}
            fill="white" stroke="var(--color-neutral-200)" strokeWidth={0.8} />
      {/* Clock icon area */}
      <circle cx={tx + 13} cy={ty + tipH / 2} r={5}
              fill="var(--color-primary)" fillOpacity={0.12} />
      <text x={tx + 13} y={ty + tipH / 2 + 3} textAnchor="middle"
            fontSize={7} fill="var(--color-primary)">
        ⏱
      </text>
      {/* City name */}
      <text x={tx + 24} y={ty + 13} fontSize={7} fontWeight={600}
            fill="var(--color-neutral-900)">
        {hoveredCity.name}
      </text>
      {/* Drive time estimate */}
      <text x={tx + 24} y={ty + 25} fontSize={8} fontWeight={700}
            fill="var(--color-primary-dark)">
        ~{hoveredCity.driveTime}
      </text>
    </g>
  );
})()}
```

### HQ Marker Pattern

```tsx
{/* Larger dot + animated pulse ring */}
<circle cx={x} cy={y} r={5} fill="var(--color-accent)" />
<circle cx={x} cy={y} r={9} fill="none" stroke="var(--color-accent)"
        strokeWidth={1.2} opacity={0.4}>
  <animate attributeName="r" values="9;14;9" dur="2.5s"
           repeatCount="indefinite" />
  <animate attributeName="opacity" values="0.4;0;0.4" dur="2.5s"
           repeatCount="indefinite" />
</circle>
```

### Hover Hint in Sidebar

Tell users the map is interactive — they won't discover it otherwise:

```tsx
<div className="mt-6 rounded-lg bg-[color:var(--color-primary)] bg-opacity-[0.06] p-3.5">
  <div className="flex items-start gap-2.5">
    <Clock className="w-4 h-4 text-[color:var(--color-primary)] shrink-0 mt-0.5" />
    <p className="text-xs text-[color:var(--color-neutral-600)] leading-relaxed">
      <span className="font-semibold text-[color:var(--color-neutral-800)]">
        Hover a city
      </span>{" "}
      on the map to see estimated drive time from our {hqCity} headquarters.
    </p>
  </div>
</div>
```

## Drive Time Estimation

Estimate from HQ to each city based on:
- **Highway distance** via Google Maps (straight-line × 1.3 for road factor)
- **Traffic factor**: +20-30% for cities requiring freeway merges
- **Present as ranges**: "15–20 min" not "17 min" — acknowledges variability

| Distance from HQ | Typical Range |
|-------------------|---------------|
| Adjacent city (< 5 mi) | 5 min |
| Same county, on highway (5-15 mi) | 10–20 min |
| Same county, off highway (10-20 mi) | 15–25 min |
| Adjacent county, near border (15-25 mi) | 20–30 min |
| Adjacent county, far side (25-40 mi) | 35–55 min |

**Always include disclaimer:**
> *Drive times are rough estimates from our [City] headquarters and may vary based on traffic, time of day, and road conditions.*

## Dual-Scale Composition

### State Overview (small, left column)

- Simplified state outline (~40-60 points)
- Service region highlight polygon (rough, covers counties)
- HQ marker dot with pulse
- Dashed zoom-indicator rectangle around service area
- Connector dashed line to detail map

### County Detail (large, center column)

- `<clipPath>` with separate `<path>` per county (allows multi-county clipping)
- Full cartographic detail inside clip
- County outlines stroked OUTSIDE clip for crisp edges
- Scale indicator at bottom

### Legend + Address (right column)

- Address card with MapPin icon
- Color-coded legend entries (HQ dot, service city dot, highway, county line)
- "Hover a city" hint with Clock icon
- Quick stats (communities served, counties)

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Forgetting `cos(lat)` correction | East-west stretches ~27%. Always correct. |
| Using raw GeoJSON (200+ pts) | Looks smooth/rounded at render scale. Simplify coastlines. |
| Clipping outline strokes | Stroke gets cut at county edges. Render outlines OUTSIDE `<clipPath>`. |
| Tooltip cut off by viewBox | Clamp tooltip position to viewBox bounds; flip below if near top. |
| Single `<path>` for multi-county clip | Use separate `<path>` per county inside one `<clipPath>`. |
| Hard-coding drive times as exact | Use ranges. Include disclaimer. Traffic varies. |
| No hit area on markers | 3px dots are hard to hover. Add transparent r=14 circle. |

## Quick Reference

```
GeoJSON source → project(lon,lat) → RDP simplify → SVG path string
                   ↓ cos(lat) fix     ↓ ε=1.2 inland
                   ↓ scale+offset     ↓ ε=3.5 coast

Component: "use client" + useState for hover
SVG viewBox: ~320×420 for county detail, ~420×490 for state
clipPath: separate <path> per county
Tooltip: rendered last in SVG, pointerEvents:none
HQ marker: dot + <animate> pulse ring
Disclaimer: always included below map
```
