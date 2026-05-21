# WU Location Selector — Developer Reference

**Last updated:** 2026-05-20  
**Repo:** https://github.com/SteveVert3XGit/RemittanceZipZone  
**Main file:** `wu_location_selector.html`

---

## Architecture

Single self-contained HTML/CSS/JS file — no frameworks, no build tools, no backend.

- All store data hardcoded in the `ZIP_CONFIG` JS object at the top of the file
- Pins render programmatically from that data; nothing is hand-placed
- Hot zones computed at page load via the `computeZones()` function
- Local dev server: `npx serve -p 8891` (configured in `.claude/launch.json`)

---

## ZIP_CONFIG Structure

```js
const ZIP_CONFIG = {
  '33147': {
    label: 'ZIP 33147 — Liberty City / NW Miami',
    map: {
      latMin: 25.815, latMax: 25.855, latRange: 0.040,
      lngMin: -80.255, lngMax: -80.215, lngRange: 0.040,
      svgW: 700, svgH: 520
    },
    corridors: {
      // Named intersections used for zone labels
      // Key: "StreetA @ StreetB", Value: { lat, lng }
      'NW 7th Ave @ NW 62nd St': { lat: 25.837, lng: -80.221 }
    },
    streets: [
      // SVG line descriptors
      { type: 'ns', x: 350, weight: 2, label: 'NW 7th Ave' },
      { type: 'ew', y: 260, weight: 2, label: 'NW 62nd St', labelX: 10, labelY: 256 }
    ],
    demographics: {
      headline: 'High-density remittance corridor',
      context: 'Narrative paragraph shown below the map...'
    },
    stores: [ /* store objects */ ]
  }
};
```

---

## Store Object Shape

```js
{
  id: 'wu_001',               // unique per ZIP
  category: 'western_union',  // drives pin color + toggle
  name: 'Check City',
  address: '1234 NW 7th Ave, Miami FL 33147',
  lat: 25.8234,
  lng: -80.2156,
  rating: 4.2,
  reviews: 38,
  hours: 'Mon–Sat 9am–7pm',
  notes: 'Inside Check City',
  bigbox: true                // optional — see Big-Box Flag below
}
```

---

## Pin Categories & Colors

| Category | Color | Size | Role |
|---|---|---|---|
| `western_union` | Gold `#F5C518` | 14px | Primary |
| `moneygram` | Navy `#004B87` | 14px | Primary |
| `ria` | Red `#E31837` | 14px | Primary |
| `financial_services` | Green `#1D9E75` | 8px | Demand signal |
| `ethnic_grocery` | Purple `#9B59B6` | 8px | Demand signal |
| `dollar_store` | Pink `#E91E8C` | 8px | Demand signal |
| `laundromat` | Cyan `#00BCD4` | 8px | Demand signal |

---

## Zone Algorithm

### Tunable Constants (top of file)
```js
const CLUSTER_RADIUS_MI    = 0.35;   // radius to group demand signals
const HIGH_DEMAND_THRESHOLD = 5;     // min adjDemand to fire EXPAND or MONITOR
const LOW_THRESHOLD         = 4;     // max adjDemand before REDUCE triggers
```

### Demand Weights
```js
const DEMAND_WEIGHTS = {
  ethnic_grocery:    3,
  financial_services: 2,
  dollar_store:      1,
  laundromat:        1
};
const COMPETITOR_BONUS = 2; // per MG or Ria within 0.5mi of cluster
```

### Zone Classification
| Zone | Condition | Color |
|---|---|---|
| `zones_expand` | adjDemand ≥ 5, wuCount = 0 | Green |
| `zones_monitor` | adjDemand ≥ 5, wuCount ≥ 2 | Amber |
| `zones_reduce` | (wuCluster ≥ 3 OR adjDemand < LOW_THRESHOLD) AND wuCount ≥ 2 | Red |

---

## Big-Box Flag

Adding `bigbox: true` to a store object:
- Renders pin at 35% opacity (visually dimmed)
- Excludes store from `computeZones()` density counts
- Excludes from the STORE COUNT (Owned/Independent) scorecard

Use for: WU@Walmart, WU@Walgreens, MG@CVS, etc.

---

## SVG Coordinate Mapping

```js
svgX = (lng - lngMin) / lngRange * svgW
svgY = (1 - (lat - latMin) / latRange) * svgH
```

`latLngToSVG(lat, lng)` is the helper function — use it everywhere, don't inline the formula.

---

## Adding a New ZIP

1. Research stores via Google Maps (WU, MG, Ria, demand signals)
2. Look up ZIP boundary corners to set `latMin/Max`, `lngMin/Max`
3. Add `ZIP_CONFIG['XXXXX']` entry with map, corridors, streets, demographics, stores
4. Add `<option value="XXXXX">` to `#zip-select` dropdown (~line 96 of HTML file)
5. Reload page, switch to new ZIP in dropdown
6. Verify no out-of-bounds pins:
   ```js
   ZIP_CONFIG['XXXXX'].stores.map(s => {
     const cfg = ZIP_CONFIG['XXXXX'];
     const x = (s.lng - cfg.map.lngMin) / cfg.map.lngRange * cfg.map.svgW;
     const y = (1 - (s.lat - cfg.map.latMin) / cfg.map.latRange) * cfg.map.svgH;
     return { id: s.id, inBounds: x >= 0 && x <= cfg.map.svgW && y >= 0 && y <= cfg.map.svgH };
   })
   ```
7. Check zones: `computeZones().map(z => ({ type: z.type, label: z.label, adjDemand: z.adjDemand, wuCount: z.wuCount }))`

---

## Current ZIPs (as of 2026-05-20)

| ZIP | Label | WU | MG | Ria |
|---|---|---|---|---|
| 33147 | Liberty City / NW Miami | 8 | 3 | 3 |
| 33463 | Lake Worth / Greenacres | 5 | 4 | 4 |
| 33430 | Belle Glade | 3 | 2 | 1 |
| 33460 | Lake Worth Beach | 6 | 5 | 1 |
| 33023 | Hollywood / Pembroke Park | 7 | 5 | 4 |
| 33142 | Allapattah / Miami | 5 | 6 | 2 |
| 33144 | Westchester / Miami | 8 | 4 | 1 |
| 33180 | Aventura / NE Miami-Dade | 3 | 3 | 1 |
| 33139 | South Beach / Miami Beach | 8 | 8 | 0 |

---

## Key Functions

| Function | Purpose |
|---|---|
| `selectZip(zip)` | Switch active ZIP, re-renders everything |
| `computeZones()` | Returns zone array from current ZIP's store data; cached in `_zonesCache` |
| `renderPins()` | Draws all store pins on SVG |
| `renderZones()` | Draws zone ellipses on SVG |
| `renderScorecard()` | Updates right-panel zone list + store counts |
| `renderNarrative()` | Updates analysis text columns below map |
| `renderStreets()` | Draws street grid lines on SVG |
| `renderDemographics()` | Updates demographics section |
| `latLngToSVG(lat, lng)` | Converts coordinates to SVG pixel position |
| `nearestCorridor(lat, lng)` | Returns closest named intersection string for zone labels |
| `haversine(lat1, lng1, lat2, lng2)` | Returns distance in miles between two points |

---

## Local Dev

```bash
# Start server
npx serve -p 8891

# Open in browser
open http://localhost:8891/wu_location_selector.html
```

Or use the `.claude/launch.json` configuration in Claude Code (runs automatically with the "WU Location Selector" launch config).
