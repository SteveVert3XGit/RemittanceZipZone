# ZIP 33157 Retail Competitive Area Mapping
## Claude Code Handoff — Boost Mobile Prospect Engine

---

## Live URL
https://hipposteve02-sudo.github.io/BoostProspectEngine/zip_33157_store_map_v2.html

## GitHub Repo
https://github.com/hipposteve02-sudo/BoostProspectEngine

---

## Files
| File | Description |
|---|---|
| `zip_33157_store_map_v2.html` | Main application — fully self-contained HTML/CSS/JS |
| `all_stores_master_v3.csv` | Source store location data (Boost, Metro, competitors) |
| `README.md` | This file |

---

## Architecture

### Map
- Custom SVG viewBox: `0 0 700 520`
- Coordinate mapping (lat/lng → SVG x/y):
  - Lat range: 25.578 (y=510) to 25.632 (y=30)
  - Lng range: -80.402 (x=20) to -80.342 (x=680)
  - Formula: `x = (lng + 80.402) / 0.060 * 660 + 20`
  - Formula: `y = 510 - (lat - 25.578) / 0.054 * 480`

### Pin Categories & Colors
| Category | Color | Hex |
|---|---|---|
| Boost Mobile | Orange | #E07B20 |
| Other Prepaid (Metro by T-Mobile) | Purple | #7F77DD |
| Money Services | Green | #1D9E75 |
| Dollar Stores | Pink | #D4537E |

### Pin Sizes
- Boost Mobile + Metro by T-Mobile: `r="14"` (larger, prominent)
- Money Services + Dollar Stores: `r="8"` (smaller, context layer)

### Heat Map
- 5 manually positioned SVG ellipses with radialGradient fills
- Hidden by default, toggled via `toggleHeatmap()` JS function
- Zones based on co-location density analysis:
  1. S Dixie / Cutler Bay corridor (hottest)
  2. SW 184th St cluster
  3. SW 152nd St node
  4. SW 160th St node
  5. Caribbean Blvd / southern tip

---

## Store Data (hardcoded in HTML)

### Boost Mobile (1)
- 11340 Quail Roost Dr — ⭐ 4.0 (47 reviews)

### Metro by T-Mobile (5)
- 11346 SW 184th St — ⭐ 4.6 (54 reviews)
- 18851 S Dixie Hwy — ⭐ 4.2 (77 reviews)
- 18497 S Dixie Hwy — ⭐ 3.3 (48 reviews)
- 11271 SW 152nd St — ⭐ 3.8 (59 reviews)
- 9690 SW 160th St — ⭐ 4.6 (43 reviews)

### Money Services (10 standalone only — big-box excluded)
- Amscot 24hr — 18861 SW 117th Ave
- Check Cashing USA — 11348 SW 184th St
- The Check Cashing Store — 10853 Caribbean Blvd
- Fast Payday Loans — 19993 S Dixie Hwy
- Quick Cash Auto Loans — 18400 SW 97th Ave
- Western Union — 10945 SW 186th St
- MoneyGram — 10994 SW 184th St
- MoneyGram — 11221 SW 152nd St
- Ria Money Transfer — 10387 SW 186th St
- JN Money Agent — 16930 S Dixie Hwy

**Excluded (inside big-box retailers):**
- Western Union @ 18590 S Dixie (Walgreens)
- Western Union @ 19167 S Dixie (Winn-Dixie)
- Western Union @ 18485 S Dixie (Publix)
- MoneyGram @ 19198 S Dixie (CVS)
- MoneyGram @ 11398 Quail Roost Dr (CVS)

### Dollar Stores (6)
- Family Dollar — 9600 SW 160th St
- Family Dollar — 11520 Quail Roost Dr
- Family Dollar — 11269 SW 152nd St
- Dollar Tree — 19173 S Dixie Hwy
- Dollar General — 18713 S Dixie Hwy
- Dollar Tree — 9825 Hibiscus St, Palmetto Bay

---

## API Keys & Services
| Service | Key | Notes |
|---|---|---|
| Google Street View Static API | AIzaSyBeI7HLljOU3LjQe9RTOQVWx8gNOHBhAyc | Restrict to your domain in GCP console |
| Google Maps search links | None required | Uses `maps.google.com/maps/search` URL format |
| html2pdf.js | None | Loaded from cdnjs CDN |
| Aptos font | None | Loaded from Google Fonts |

---

## Recommended Refactors for Claude Code

### Priority 1 — Data-driven pins
Move all store data into a JS array:
```js
const STORES = [
  { name: 'Boost Mobile', addr: '11340 Quail Roost Dr', cat: 'boost',
    lat: 25.5926, lng: -80.3767, rating: 4.0, reviews: 47, hours: '9AM–7PM' },
  ...
];
```
Then render pins programmatically from the array rather than hardcoded SVG.

### Priority 2 — Switch to Leaflet.js
Replace custom SVG map with Leaflet for:
- Real zoom/pan
- Accurate pin placement via lat/lng
- Easy multi-ZIP expansion
- Built-in heat map via Leaflet.heat plugin

### Priority 3 — ZIP selector
Add a dropdown to switch between market ZIPs, loading different
store datasets dynamically.

### Priority 4 — Dynamic Google Places query
Query the Places API at runtime by ZIP code instead of
hardcoding stores, so data stays current.

### Priority 5 — Table view
Toggle between map and a sortable/filterable data table
showing all stores with ratings, hours, distance from Boost.

---

## Competitive Context (built into the HTML)
- Boost: 1 store | Metro: 5 stores — 5× retail gap
- S Dixie Hwy corridor is the highest-priority expansion zone
- Core demographic: cash-transacting, unbanked, value-sensitive
- Boost's $25/mo BYOP plan undercuts Metro's entry pricing
- 212,000 net subscribers added nationally in Q2 2025

---

*Built with Claude Sonnet 4.6 | Anthropic | May 2026*
