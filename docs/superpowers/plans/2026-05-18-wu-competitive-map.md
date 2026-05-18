# WU Competitive Intelligence Map Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single self-contained HTML file that maps Western Union, MoneyGram, and Ria locations in ZIP 33147 (NW Miami) against demand-signal businesses, computing expansion and saturation zones algorithmically from the data.

**Architecture:** All store data lives in a top-of-file JS array. Pins render programmatically from that array onto a hand-drawn SVG street grid for ZIP 33147. Hot zones (expand / monitor / reduce) are computed at page load via a demand-scoring algorithm — no ellipses are hand-placed.

**Tech Stack:** Pure HTML/CSS/JS, no frameworks. html2pdf.js via CDN for PDF export. Google Fonts (Aptos). Google Maps search links for pin click-through.

---

## File Structure

```
zip_33147_wu_competitive_map.html   ← single deliverable, all code inline
```

Reference file (read-only, patterns to borrow):
```
zip_33157_store_map_v2.html         ← existing tool: toggle JS, pin CSS, PDF wiring
```

---

## Coordinate System Reference

ZIP 33147 bounds and SVG mapping formula used throughout all tasks:

```js
const MAP = {
  latMin: 25.813, latMax: 25.851, latRange: 0.038,
  lngMin: -80.251, lngMax: -80.213, lngRange: 0.038,
  svgW: 700, svgH: 520
};

function latLngToSVG(lat, lng) {
  return {
    x: ((lng - MAP.lngMin) / MAP.lngRange) * MAP.svgW,
    y: (1 - (lat - MAP.latMin) / MAP.latRange) * MAP.svgH
  };
}
```

Key street SVG coordinates (pre-computed for hand-drawing):
| Street | Direction | SVG position |
|---|---|---|
| NW 79th St | E/W | y ≈ 27 |
| NW 62nd St | E/W | y ≈ 205 |
| NW 54th St | E/W | y ≈ 287 |
| NW 36th St | E/W | y ≈ 520 (bottom edge) |
| NW 27th Ave | N/S | x ≈ 37 |
| NW 17th Ave | N/S | x ≈ 258 |
| NW 7th Ave  | N/S | x ≈ 479 |

---

## Category Config Reference

Used in Tasks 3, 4, 6, 7 — defined once in Task 1:

```js
const CATEGORIES = {
  western_union:      { label: 'Western Union',    color: '#F5C518', role: 'primary',       r: 14 },
  moneygram:          { label: 'MoneyGram',         color: '#004B87', role: 'primary',       r: 14 },
  ria:                { label: 'Ria',               color: '#E31837', role: 'primary',       r: 14 },
  financial_services: { label: 'Financial Svcs',   color: '#1D9E75', role: 'demand_signal', r: 8  },
  ethnic_grocery:     { label: 'Ethnic Grocery',   color: '#9B59B6', role: 'demand_signal', r: 8  },
  dollar_store:       { label: 'Dollar Store',     color: '#E91E8C', role: 'demand_signal', r: 8  },
  laundromat:         { label: 'Laundromat',       color: '#00BCD4', role: 'demand_signal', r: 8  }
};
```

Demand score weights (used in Task 6):
```js
const DEMAND_WEIGHTS = {
  ethnic_grocery:     3,
  financial_services: 2,
  dollar_store:       1,
  laundromat:         1
};
const COMPETITOR_BONUS = 2; // per MoneyGram or Ria within 0.5mi of centroid
```

Tunable thresholds (defined as constants at top of file):
```js
const CLUSTER_RADIUS_MI    = 0.3;  // demand signal clustering radius
const COMPETITOR_RADIUS_MI = 0.5;  // competitor validation radius
const WU_RADIUS_MI         = 0.5;  // WU supply density radius
const HIGH_DEMAND_THRESHOLD = 5;   // adjDemand ≥ this = high demand
const LOW_DEMAND_THRESHOLD  = 4;   // adjDemand < this = low demand
```

---

## Task 1: Store Data Research + HTML Shell

**Files:**
- Create: `zip_33147_wu_competitive_map.html`

### Step 1.1 — Research real store locations for ZIP 33147

Search Google Maps / Google Places for each of the following queries. For each result, record: name, full address, lat, lng, rating, review count, hours. Aim for completeness — capture all results within or immediately adjacent to ZIP 33147.

Search queries:
- "Western Union Miami FL 33147"
- "MoneyGram Miami FL 33147"
- "Ria Money Transfer Miami FL 33147"
- "check cashing Miami FL 33147"
- "payday loan Miami FL 33147"
- "ethnic grocery store Miami FL 33147"
- "Latin supermarket Miami FL 33147"
- "Caribbean grocery Miami FL 33147"
- "dollar store Miami FL 33147"
- "laundromat Miami FL 33147"

Expected: 5–15 results per category. Expect WU/MG/Ria to appear inside grocery stores, pharmacies, and check-cashing shops — capture the host business name in the `notes` field.

- [ ] **Step 1.2 — Create the HTML file with the stores array**

Create `zip_33147_wu_competitive_map.html` with this exact structure. Fill in all `// ADD REAL DATA` entries with research from Step 1.1:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>WU Competitive Intel — ZIP 33147</title>
  <link href="https://fonts.googleapis.com/css2?family=Aptos:wght@400;600;700&display=swap" rel="stylesheet">
  <script src="https://cdnjs.cloudflare.com/ajax/libs/html2pdf.js/0.10.1/html2pdf.bundle.min.js"></script>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: 'Aptos', sans-serif; background: #0f1117; color: #e6edf3; min-height: 100vh; }
  </style>
</head>
<body>
<script>
// ─── TUNABLE CONSTANTS ────────────────────────────────────────────────────────
const CLUSTER_RADIUS_MI     = 0.3;
const COMPETITOR_RADIUS_MI  = 0.5;
const WU_RADIUS_MI          = 0.5;
const HIGH_DEMAND_THRESHOLD = 5;
const LOW_DEMAND_THRESHOLD  = 4;

// ─── MAP BOUNDS ───────────────────────────────────────────────────────────────
const MAP = {
  latMin: 25.813, latMax: 25.851, latRange: 0.038,
  lngMin: -80.251, lngMax: -80.213, lngRange: 0.038,
  svgW: 700, svgH: 520
};

// ─── CATEGORY CONFIG ──────────────────────────────────────────────────────────
const CATEGORIES = {
  western_union:      { label: 'Western Union',  color: '#F5C518', role: 'primary',       r: 14 },
  moneygram:          { label: 'MoneyGram',       color: '#004B87', role: 'primary',       r: 14 },
  ria:                { label: 'Ria',             color: '#E31837', role: 'primary',       r: 14 },
  financial_services: { label: 'Financial Svcs', color: '#1D9E75', role: 'demand_signal', r: 8  },
  ethnic_grocery:     { label: 'Ethnic Grocery', color: '#9B59B6', role: 'demand_signal', r: 8  },
  dollar_store:       { label: 'Dollar Store',   color: '#E91E8C', role: 'demand_signal', r: 8  },
  laundromat:         { label: 'Laundromat',     color: '#00BCD4', role: 'demand_signal', r: 8  }
};

const DEMAND_WEIGHTS = {
  ethnic_grocery: 3, financial_services: 2, dollar_store: 1, laundromat: 1
};
const COMPETITOR_BONUS = 2;

// ─── STORE DATA ───────────────────────────────────────────────────────────────
// Each store: { id, category, name, address, lat, lng, rating, reviews, hours, notes }
const stores = [
  // ADD REAL DATA FROM STEP 1.1 HERE
  // Example shape:
  // { id:'wu_001', category:'western_union', name:'Western Union at Navarro', address:'1901 NW 7th Ave, Miami FL 33147', lat:25.8234, lng:-80.2256, rating:3.8, reviews:24, hours:'Mon–Sat 8am–9pm', notes:'Inside Navarro Discount Pharmacy' },
];
</script>
</body>
</html>
```

- [ ] **Step 1.3 — Verify file opens in browser**

Open `zip_33147_wu_competitive_map.html` directly in a browser (no server needed — `file://` works).

Expected: Blank dark page, no JS errors in console (F12 → Console).

- [ ] **Step 1.4 — Commit**

```bash
git add zip_33147_wu_competitive_map.html
git commit -m "feat: scaffold WU map with store data array for ZIP 33147"
```

---

## Task 2: SVG Street Grid

**Files:**
- Modify: `zip_33147_wu_competitive_map.html`

- [ ] **Step 2.1 — Add page layout HTML and SVG street grid**

Add inside `<body>`, before `<script>`:

```html
<div id="app">
  <header id="top-bar">
    <div id="title-block">
      <span id="logo">⬡</span>
      <span id="title-text">WU Competitive Intel <span id="zip-label">ZIP 33147 — NW Miami</span></span>
    </div>
    <button id="pdf-btn" onclick="exportPDF()">⬇ Export PDF</button>
  </header>

  <div id="toggle-bar"><!-- populated by JS in Task 4 --></div>

  <div id="main-content">
    <div id="map-wrap">
      <svg id="map-svg" viewBox="0 0 700 520" xmlns="http://www.w3.org/2000/svg">
        <!-- Street grid -->
        <g id="streets">
          <!-- E/W streets -->
          <line x1="0"   y1="27"  x2="700" y2="27"  stroke="#2a3340" stroke-width="3"/>
          <line x1="0"   y1="205" x2="700" y2="205" stroke="#2a3340" stroke-width="3"/>
          <line x1="0"   y1="287" x2="700" y2="287" stroke="#2a3340" stroke-width="2"/>
          <line x1="0"   y1="520" x2="700" y2="520" stroke="#2a3340" stroke-width="2"/>
          <!-- N/S streets -->
          <line x1="37"  y1="0"   x2="37"  y2="520" stroke="#2a3340" stroke-width="3"/>
          <line x1="258" y1="0"   x2="258" y2="520" stroke="#2a3340" stroke-width="2"/>
          <line x1="479" y1="0"   x2="479" y2="520" stroke="#2a3340" stroke-width="3"/>
          <!-- Street labels -->
          <text x="42"  y="23"  fill="#3a4a5a" font-size="9" font-family="monospace">NW 79th St</text>
          <text x="42"  y="201" fill="#3a4a5a" font-size="9" font-family="monospace">NW 62nd St</text>
          <text x="42"  y="283" fill="#3a4a5a" font-size="9" font-family="monospace">NW 54th St</text>
          <text x="42"  y="516" fill="#3a4a5a" font-size="9" font-family="monospace">NW 36th St</text>
          <text x="0"   y="16"  fill="#3a4a5a" font-size="9" font-family="monospace" transform="rotate(-90 39 120)" text-anchor="middle">NW 27th Ave</text>
          <text x="0"   y="16"  fill="#3a4a5a" font-size="9" font-family="monospace" transform="rotate(-90 260 120)" text-anchor="middle">NW 17th Ave</text>
          <text x="0"   y="16"  fill="#3a4a5a" font-size="9" font-family="monospace" transform="rotate(-90 481 120)" text-anchor="middle">NW 7th Ave</text>
        </g>
        <!-- Zone ellipses rendered here by Task 7 -->
        <g id="zones-layer" display="none"></g>
        <!-- Pins rendered here by Task 3 -->
        <g id="pins-layer"></g>
        <!-- Compass -->
        <g transform="translate(660,480)">
          <circle r="14" fill="#1a1f2e" stroke="#2a3040"/>
          <text x="0" y="-4" text-anchor="middle" fill="#79c0ff" font-size="10" font-weight="bold">N</text>
          <line x1="0" y1="-12" x2="0" y2="12" stroke="#444" stroke-width="1"/>
          <line x1="-12" y1="0" x2="12" y2="0" stroke="#444" stroke-width="1"/>
        </g>
      </svg>
    </div>

    <div id="right-panel">
      <div id="pin-detail">
        <p id="pin-placeholder">Click any pin to see details</p>
      </div>
      <div id="scorecard"><!-- populated by Task 8 --></div>
    </div>
  </div>

  <div id="narrative"><!-- populated by Task 9 --></div>
</div>
```

- [ ] **Step 2.2 — Add layout CSS**

Add inside `<style>`:

```css
#app { display: flex; flex-direction: column; min-height: 100vh; padding: 12px; gap: 10px; max-width: 1100px; margin: 0 auto; }
#top-bar { display: flex; justify-content: space-between; align-items: center; padding: 10px 14px; background: #1a1f2e; border-radius: 8px; }
#logo { color: #F5C518; font-size: 18px; margin-right: 8px; }
#title-text { font-weight: 700; font-size: 15px; }
#zip-label { color: #666; font-weight: 400; font-size: 13px; margin-left: 8px; }
#pdf-btn { background: #2a2a2a; color: #ccc; border: 1px solid #444; padding: 5px 12px; border-radius: 5px; cursor: pointer; font-size: 12px; }
#pdf-btn:hover { background: #333; }
#toggle-bar { display: flex; flex-wrap: wrap; gap: 6px; }
#main-content { display: grid; grid-template-columns: 1fr 280px; gap: 10px; }
#map-wrap { background: #111927; border-radius: 10px; overflow: hidden; }
#map-svg { width: 100%; height: auto; display: block; }
#right-panel { display: flex; flex-direction: column; gap: 8px; }
#pin-detail { background: #1a1f2e; border-radius: 8px; padding: 14px; min-height: 120px; border: 1px solid #2a3040; }
#pin-placeholder { color: #555; font-size: 13px; }
#scorecard { background: #1a1f2e; border-radius: 8px; padding: 14px; flex: 1; border: 1px solid #2a3040; }
#narrative { background: #1a1f2e; border-radius: 8px; padding: 16px; border: 1px solid #2a3040; }
```

- [ ] **Step 2.3 — Verify street grid renders**

Open/refresh the file in browser.

Expected: Dark page with a visible grey street grid showing 4 horizontal streets and 3 vertical streets. Labels visible. Compass in bottom-right of map. No JS errors.

- [ ] **Step 2.4 — Commit**

```bash
git add zip_33147_wu_competitive_map.html
git commit -m "feat: add SVG street grid for ZIP 33147"
```

---

## Task 3: Programmatic Pin Rendering

**Files:**
- Modify: `zip_33147_wu_competitive_map.html`

- [ ] **Step 3.1 — Add coordinate helper and pin renderer to JS**

Add after the `stores` array in `<script>`:

```js
// ─── COORDINATE HELPER ────────────────────────────────────────────────────────
function latLngToSVG(lat, lng) {
  return {
    x: ((lng - MAP.lngMin) / MAP.lngRange) * MAP.svgW,
    y: (1 - (lat - MAP.latMin) / MAP.latRange) * MAP.svgH
  };
}

// ─── PIN RENDERING ────────────────────────────────────────────────────────────
function renderPins() {
  const layer = document.getElementById('pins-layer');
  layer.innerHTML = '';
  stores.forEach(s => {
    const cfg = CATEGORIES[s.category];
    if (!cfg) return;
    const { x, y } = latLngToSVG(s.lat, s.lng);
    const r = cfg.r;
    const g = document.createElementNS('http://www.w3.org/2000/svg', 'g');
    g.setAttribute('data-id', s.id);
    g.setAttribute('data-cat', s.category);
    g.setAttribute('class', 'pin');
    g.style.cursor = 'pointer';
    // Drop shape (triangle below circle)
    const drop = document.createElementNS('http://www.w3.org/2000/svg', 'polygon');
    const tipY = y + r + 8;
    drop.setAttribute('points', `${x - r * 0.5},${y + r * 0.6} ${x + r * 0.5},${y + r * 0.6} ${x},${tipY}`);
    drop.setAttribute('fill', cfg.color);
    // Circle
    const circle = document.createElementNS('http://www.w3.org/2000/svg', 'circle');
    circle.setAttribute('cx', x);
    circle.setAttribute('cy', y);
    circle.setAttribute('r', r);
    circle.setAttribute('fill', cfg.color);
    circle.setAttribute('stroke', '#0f1117');
    circle.setAttribute('stroke-width', '1.5');
    g.appendChild(drop);
    g.appendChild(circle);
    g.addEventListener('mouseenter', () => g.style.filter = 'brightness(1.25)');
    g.addEventListener('mouseleave', () => g.style.filter = '');
    g.addEventListener('click', () => showPin(s));
    layer.appendChild(g);
  });
}
```

- [ ] **Step 3.2 — Call renderPins on page load**

Add at the bottom of `<script>` (after all function definitions):

```js
// ─── INIT ─────────────────────────────────────────────────────────────────────
document.addEventListener('DOMContentLoaded', () => {
  renderPins();
});
```

- [ ] **Step 3.3 — Verify pins render**

Open/refresh in browser.

Expected: Colored circle+drop pins appear at correct street intersections for each store. Pins respond to hover (brighten). Clicking a pin does nothing yet (Task 5). No JS errors.

If you have only a few test stores in the array at this point, verify those render at geographically sensible SVG positions using `latLngToSVG()` manually: e.g., a store at lat 25.836, lng -80.225 should appear at approximately x=479, y=205 (NW 7th Ave @ NW 62nd St).

- [ ] **Step 3.4 — Commit**

```bash
git add zip_33147_wu_competitive_map.html
git commit -m "feat: render pins programmatically from stores array"
```

---

## Task 4: Toggle Buttons

**Files:**
- Modify: `zip_33147_wu_competitive_map.html`

- [ ] **Step 4.1 — Add toggle button rendering to JS**

Add after `renderPins()` definition:

```js
// ─── TOGGLE BUTTONS ───────────────────────────────────────────────────────────
const activeToggles = new Set(Object.keys(CATEGORIES));
activeToggles.add('zones_expand');
activeToggles.add('zones_reduce');

function renderToggles() {
  const bar = document.getElementById('toggle-bar');
  bar.innerHTML = '';
  // Category toggles
  Object.entries(CATEGORIES).forEach(([cat, cfg]) => {
    const btn = document.createElement('button');
    btn.className = 'toggle-btn';
    btn.dataset.cat = cat;
    btn.textContent = '● ' + cfg.label;
    btn.style.background = cfg.color;
    btn.style.color = cfg.role === 'primary' ? '#000' : '#fff';
    btn.addEventListener('click', () => toggleCategory(cat, btn));
    bar.appendChild(btn);
  });
  // Zone toggles
  [
    { key: 'zones_expand', label: '◎ Expansion Zones', border: '#3fb950', color: '#3fb950' },
    { key: 'zones_reduce', label: '◎ Saturation Zones', border: '#f85149', color: '#f85149' }
  ].forEach(({ key, label, border, color }) => {
    const btn = document.createElement('button');
    btn.className = 'toggle-btn zone-toggle';
    btn.dataset.cat = key;
    btn.textContent = label;
    btn.style.border = `1px solid ${border}`;
    btn.style.color = color;
    btn.addEventListener('click', () => toggleZones(key, btn));
    bar.appendChild(btn);
  });
}

function toggleCategory(cat, btn) {
  if (activeToggles.has(cat)) {
    activeToggles.delete(cat);
    btn.style.opacity = '0.35';
  } else {
    activeToggles.add(cat);
    btn.style.opacity = '1';
  }
  document.querySelectorAll(`.pin[data-cat="${cat}"]`).forEach(el => {
    el.style.display = activeToggles.has(cat) ? '' : 'none';
  });
}

function toggleZones(key, btn) {
  if (activeToggles.has(key)) {
    activeToggles.delete(key);
    btn.style.opacity = '0.35';
  } else {
    activeToggles.add(key);
    btn.style.opacity = '1';
  }
  // Zones layer visibility handled in Task 7
  document.querySelectorAll(`.zone-ellipse[data-type="${key}"]`).forEach(el => {
    el.style.display = activeToggles.has(key) ? '' : 'none';
  });
}
```

- [ ] **Step 4.2 — Add toggle button CSS**

Add inside `<style>`:

```css
.toggle-btn { border: none; padding: 5px 11px; border-radius: 5px; font-size: 11px; font-weight: 600; cursor: pointer; background: #1a1f2e; transition: opacity 0.15s; }
.zone-toggle { background: #1a1f2e !important; }
.toggle-btn:hover { filter: brightness(1.1); }
```

- [ ] **Step 4.3 — Call renderToggles in DOMContentLoaded**

Update the init block:

```js
document.addEventListener('DOMContentLoaded', () => {
  renderToggles();
  renderPins();
});
```

- [ ] **Step 4.4 — Verify toggles work**

Open/refresh in browser.

Expected: 9 buttons appear in toggle bar (7 colored category buttons + 2 outlined zone buttons). Clicking any category button dims it and hides that category's pins on the map. Clicking again restores. No JS errors.

- [ ] **Step 4.5 — Commit**

```bash
git add zip_33147_wu_competitive_map.html
git commit -m "feat: add category and zone toggle buttons"
```

---

## Task 5: Pin Click → Detail Panel

**Files:**
- Modify: `zip_33147_wu_competitive_map.html`

- [ ] **Step 5.1 — Add showPin function to JS**

Add after `renderToggles()` definition:

```js
// ─── PIN DETAIL PANEL ─────────────────────────────────────────────────────────
function showPin(store) {
  const cfg = CATEGORIES[store.category];
  const mapsUrl = `https://www.google.com/maps/search/?api=1&query=${encodeURIComponent(store.name + ' ' + store.address)}`;
  const stars = store.rating ? `⭐ ${store.rating} · ${store.reviews} reviews` : '';
  const hours = store.hours ? `🕐 ${store.hours}` : '';
  const notes = store.notes ? `<div class="pin-notes">${store.notes}</div>` : '';
  document.getElementById('pin-detail').innerHTML = `
    <div class="pin-category-badge" style="background:${cfg.color};color:${cfg.role==='primary'?'#000':'#fff'}">${cfg.label}</div>
    <div class="pin-name">${store.name}</div>
    <div class="pin-address">${store.address}</div>
    ${stars ? `<div class="pin-meta">${stars}</div>` : ''}
    ${hours ? `<div class="pin-meta">${hours}</div>` : ''}
    ${notes}
    <a class="pin-maps-link" href="${mapsUrl}" target="_blank" rel="noopener">↗ Open in Google Maps</a>
  `;
}
```

- [ ] **Step 5.2 — Add pin detail CSS**

Add inside `<style>`:

```css
.pin-category-badge { display: inline-block; padding: 2px 8px; border-radius: 4px; font-size: 10px; font-weight: 700; letter-spacing: 0.5px; text-transform: uppercase; margin-bottom: 8px; }
.pin-name { font-weight: 700; font-size: 14px; margin-bottom: 4px; }
.pin-address { font-size: 11px; color: #8b949e; margin-bottom: 6px; }
.pin-meta { font-size: 12px; color: #ccc; margin-bottom: 3px; }
.pin-notes { font-size: 11px; color: #79c0ff; margin: 6px 0; font-style: italic; }
.pin-maps-link { display: inline-block; margin-top: 10px; font-size: 12px; color: #79c0ff; text-decoration: none; }
.pin-maps-link:hover { text-decoration: underline; }
```

- [ ] **Step 5.3 — Verify pin click panel**

Open/refresh in browser. Click a Western Union pin.

Expected: Right panel top section updates with category badge (gold background, black text), store name, address, rating, hours, notes, and a Google Maps link. Clicking the link opens a new tab with Google Maps search. No JS errors.

- [ ] **Step 5.4 — Commit**

```bash
git add zip_33147_wu_competitive_map.html
git commit -m "feat: pin click populates detail panel with store info and Maps link"
```

---

## Task 6: Haversine + Cluster Detection + Zone Classification

**Files:**
- Modify: `zip_33147_wu_competitive_map.html`

- [ ] **Step 6.1 — Add haversine distance function**

Add to JS (before zone computation functions):

```js
// ─── SPATIAL HELPERS ──────────────────────────────────────────────────────────
function haversine(lat1, lng1, lat2, lng2) {
  const R = 3958.8; // Earth radius in miles
  const dLat = (lat2 - lat1) * Math.PI / 180;
  const dLng = (lng2 - lng1) * Math.PI / 180;
  const a = Math.sin(dLat / 2) ** 2 +
            Math.cos(lat1 * Math.PI / 180) * Math.cos(lat2 * Math.PI / 180) *
            Math.sin(dLng / 2) ** 2;
  return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
}
```

- [ ] **Step 6.2 — Add cluster detection + demand scoring**

Add after haversine:

```js
// ─── ZONE COMPUTATION ─────────────────────────────────────────────────────────
function computeZones() {
  const demandStores = stores.filter(s => DEMAND_WEIGHTS[s.category] !== undefined);
  const competitors  = stores.filter(s => s.category === 'moneygram' || s.category === 'ria');
  const wuStores     = stores.filter(s => s.category === 'western_union');

  // Build clusters: for each demand store, collect all demand stores within CLUSTER_RADIUS_MI
  const visited = new Set();
  const clusters = [];
  demandStores.forEach(seed => {
    if (visited.has(seed.id)) return;
    const members = demandStores.filter(s =>
      haversine(seed.lat, seed.lng, s.lat, s.lng) <= CLUSTER_RADIUS_MI
    );
    members.forEach(m => visited.add(m.id));
    // Centroid = geographic mean
    const centroid = {
      lat: members.reduce((s, m) => s + m.lat, 0) / members.length,
      lng: members.reduce((s, m) => s + m.lng, 0) / members.length
    };
    // Base demand score from signal weights
    const baseScore = members.reduce((s, m) => s + (DEMAND_WEIGHTS[m.category] || 0), 0);
    // Competitor validation bonus
    const compBonus = competitors.filter(c =>
      haversine(centroid.lat, centroid.lng, c.lat, c.lng) <= COMPETITOR_RADIUS_MI
    ).length * COMPETITOR_BONUS;
    const adjDemand = baseScore + compBonus;
    // WU supply count
    const wuCount = wuStores.filter(w =>
      haversine(centroid.lat, centroid.lng, w.lat, w.lng) <= WU_RADIUS_MI
    ).length;
    // Classify
    let type = null;
    if (adjDemand >= HIGH_DEMAND_THRESHOLD && wuCount === 0) type = 'zones_expand';
    else if (adjDemand >= HIGH_DEMAND_THRESHOLD && wuCount >= 2) type = 'zones_monitor';
    else if (adjDemand < LOW_DEMAND_THRESHOLD  && wuCount >= 2) type = 'zones_reduce';
    clusters.push({ centroid, members, adjDemand, wuCount, type, compBonus });
  });

  return clusters.filter(c => c.type !== null);
}
```

- [ ] **Step 6.3 — Verify zone computation in console**

Add a temporary console.log in DOMContentLoaded, refresh, check the console:

```js
// Temporary — remove after verification
const zones = computeZones();
console.log('Computed zones:', zones.map(z => ({ type: z.type, adjDemand: z.adjDemand, wuCount: z.wuCount, lat: z.centroid.lat.toFixed(4), lng: z.centroid.lng.toFixed(4) })));
```

Expected: Array of zone objects logged with correct type classifications. A cluster near ethnic groceries with no WU nearby should show `zones_expand`. A cluster where 2+ WU are close but demand is low should show `zones_reduce`. If stores array is still sparse, at least verify no JS errors and the haversine function returns sensible distances (e.g., `haversine(25.83, -80.23, 25.83, -80.23)` → `0`, `haversine(25.83, -80.23, 25.84, -80.23)` → `~0.69mi`).

Remove the temporary console.log after verification.

- [ ] **Step 6.4 — Commit**

```bash
git add zip_33147_wu_competitive_map.html
git commit -m "feat: haversine distance, cluster detection, and zone classification algorithm"
```

---

## Task 7: Zone Ellipse Rendering

**Files:**
- Modify: `zip_33147_wu_competitive_map.html`

- [ ] **Step 7.1 — Add zone color config and renderZones function**

Add after `computeZones()` definition:

```js
const ZONE_CONFIG = {
  zones_expand:  { fill: 'rgba(63,185,80,0.18)',  stroke: 'rgba(63,185,80,0.6)',  label: 'Underserved — Open Here' },
  zones_monitor: { fill: 'rgba(240,192,64,0.18)', stroke: 'rgba(240,192,64,0.6)', label: 'High demand — Monitor' },
  zones_reduce:  { fill: 'rgba(248,81,73,0.18)',  stroke: 'rgba(248,81,73,0.6)',  label: 'WU Oversaturated — Review' }
};

function renderZones() {
  const layer = document.getElementById('zones-layer');
  layer.innerHTML = '';
  const defs = document.getElementById('zone-defs') || (() => {
    const d = document.createElementNS('http://www.w3.org/2000/svg', 'defs');
    d.id = 'zone-defs';
    document.getElementById('map-svg').insertBefore(d, layer);
    return d;
  })();
  defs.innerHTML = '';

  const zones = computeZones();
  zones.forEach((zone, i) => {
    const cfg = ZONE_CONFIG[zone.type];
    if (!cfg) return;
    const { x, y } = latLngToSVG(zone.centroid.lat, zone.centroid.lng);
    // Size ellipse by number of members (min 40px, +8px per member)
    const rx = Math.max(40, 40 + zone.members.length * 8);
    const ry = rx * 0.65;

    // Radial gradient
    const gradId = `zone-grad-${i}`;
    const grad = document.createElementNS('http://www.w3.org/2000/svg', 'radialGradient');
    grad.setAttribute('id', gradId);
    grad.innerHTML = `
      <stop offset="0%"   stop-color="${cfg.stroke}" stop-opacity="0.35"/>
      <stop offset="100%" stop-color="${cfg.stroke}" stop-opacity="0"/>
    `;
    defs.appendChild(grad);

    // Ellipse
    const ellipse = document.createElementNS('http://www.w3.org/2000/svg', 'ellipse');
    ellipse.setAttribute('cx', x);
    ellipse.setAttribute('cy', y);
    ellipse.setAttribute('rx', rx);
    ellipse.setAttribute('ry', ry);
    ellipse.setAttribute('fill', `url(#${gradId})`);
    ellipse.setAttribute('stroke', cfg.stroke);
    ellipse.setAttribute('stroke-width', '1');
    ellipse.setAttribute('class', 'zone-ellipse');
    ellipse.setAttribute('data-type', zone.type);
    // Apply current toggle visibility
    if (!activeToggles.has(zone.type)) ellipse.style.display = 'none';
    layer.appendChild(ellipse);

    // Zone label
    const text = document.createElementNS('http://www.w3.org/2000/svg', 'text');
    text.setAttribute('x', x);
    text.setAttribute('y', y - ry - 5);
    text.setAttribute('text-anchor', 'middle');
    text.setAttribute('fill', cfg.stroke);
    text.setAttribute('font-size', '9');
    text.setAttribute('font-family', 'monospace');
    text.setAttribute('class', 'zone-ellipse');
    text.setAttribute('data-type', zone.type);
    text.textContent = cfg.label;
    if (!activeToggles.has(zone.type)) text.style.display = 'none';
    layer.appendChild(text);
  });
}
```

- [ ] **Step 7.2 — Call renderZones in DOMContentLoaded**

Update the init block:

```js
document.addEventListener('DOMContentLoaded', () => {
  renderToggles();
  renderPins();
  renderZones();
});
```

- [ ] **Step 7.3 — Verify zone ellipses render correctly**

Open/refresh in browser. Click the "◎ Expansion Zones" toggle button to make zones visible.

Expected: Semi-transparent gradient ellipses appear on the map at cluster centroids. Green ellipses = expansion, amber = monitor, red = reduce. Labels appear above each ellipse. Toggle buttons correctly show/hide each zone type. No JS errors.

- [ ] **Step 7.4 — Commit**

```bash
git add zip_33147_wu_competitive_map.html
git commit -m "feat: render algorithmic zone ellipses sized by cluster density"
```

---

## Task 8: Zone Analysis Scorecard (Right Panel)

**Files:**
- Modify: `zip_33147_wu_competitive_map.html`

- [ ] **Step 8.1 — Add renderScorecard function**

Add after `renderZones()` definition:

```js
// ─── SCORECARD ────────────────────────────────────────────────────────────────
function renderScorecard() {
  const zones = computeZones();
  const expand  = zones.filter(z => z.type === 'zones_expand');
  const monitor = zones.filter(z => z.type === 'zones_monitor');
  const reduce  = zones.filter(z => z.type === 'zones_reduce');

  const wuCount = stores.filter(s => s.category === 'western_union').length;
  const mgCount = stores.filter(s => s.category === 'moneygram').length;
  const riaCount = stores.filter(s => s.category === 'ria').length;

  const zoneRow = (zone, idx) => {
    const { x, y } = latLngToSVG(zone.centroid.lat, zone.centroid.lng);
    return `<div class="scorecard-row">
      <strong>${idx + 1}.</strong> ${zone.centroid.lat.toFixed(4)}, ${zone.centroid.lng.toFixed(4)}
      <div class="scorecard-sub">Demand: ${zone.adjDemand} · WU nearby: ${zone.wuCount} · ${zone.members.length} signals</div>
    </div>`;
  };

  document.getElementById('scorecard').innerHTML = `
    <div class="scorecard-label">ZONE ANALYSIS</div>
    ${expand.length ? `
      <div class="scorecard-section expand">
        <div class="scorecard-title">▲ EXPAND (${expand.length})</div>
        ${expand.map(zoneRow).join('')}
      </div>` : ''}
    ${monitor.length ? `
      <div class="scorecard-section monitor">
        <div class="scorecard-title">⚠ MONITOR (${monitor.length})</div>
        ${monitor.map(zoneRow).join('')}
      </div>` : ''}
    ${reduce.length ? `
      <div class="scorecard-section reduce">
        <div class="scorecard-title">▼ REDUCE (${reduce.length})</div>
        ${reduce.map(zoneRow).join('')}
      </div>` : ''}
    <div class="scorecard-counts">
      <div class="scorecard-label" style="margin-bottom:6px">PROVIDER COUNT</div>
      <div class="count-row"><span style="color:#F5C518">●</span> Western Union <strong>${wuCount}</strong></div>
      <div class="count-row"><span style="color:#004B87">●</span> MoneyGram <strong>${mgCount}</strong></div>
      <div class="count-row"><span style="color:#E31837">●</span> Ria <strong>${riaCount}</strong></div>
    </div>
  `;
}
```

- [ ] **Step 8.2 — Add scorecard CSS**

Add inside `<style>`:

```css
.scorecard-label { font-size: 10px; color: #666; letter-spacing: 1px; text-transform: uppercase; margin-bottom: 8px; }
.scorecard-section { margin-bottom: 10px; }
.scorecard-title { font-size: 11px; font-weight: 700; margin-bottom: 4px; }
.scorecard-section.expand .scorecard-title { color: #3fb950; }
.scorecard-section.monitor .scorecard-title { color: #f0c040; }
.scorecard-section.reduce  .scorecard-title { color: #f85149; }
.scorecard-row { font-size: 11px; color: #ccc; padding: 2px 0 2px 8px; }
.scorecard-sub { font-size: 10px; color: #666; }
.scorecard-counts { border-top: 1px solid #2a3040; padding-top: 10px; margin-top: 8px; }
.count-row { display: flex; justify-content: space-between; font-size: 12px; color: #ccc; padding: 2px 0; }
```

- [ ] **Step 8.3 — Call renderScorecard in DOMContentLoaded**

Update the init block:

```js
document.addEventListener('DOMContentLoaded', () => {
  renderToggles();
  renderPins();
  renderZones();
  renderScorecard();
});
```

- [ ] **Step 8.4 — Verify scorecard**

Open/refresh in browser.

Expected: Right panel bottom section shows zone counts by type with lat/lng coordinates and demand scores. Provider count section shows correct WU/MG/Ria totals from the stores array. No JS errors.

- [ ] **Step 8.5 — Commit**

```bash
git add zip_33147_wu_competitive_map.html
git commit -m "feat: zone analysis scorecard with provider counts in right panel"
```

---

## Task 9: Analysis Narrative

**Files:**
- Modify: `zip_33147_wu_competitive_map.html`

- [ ] **Step 9.1 — Add renderNarrative function**

Add after `renderScorecard()` definition:

```js
// ─── NARRATIVE ────────────────────────────────────────────────────────────────
function renderNarrative() {
  const zones = computeZones();
  const expand  = zones.filter(z => z.type === 'zones_expand');
  const monitor = zones.filter(z => z.type === 'zones_monitor');
  const reduce  = zones.filter(z => z.type === 'zones_reduce');

  const wuCount  = stores.filter(s => s.category === 'western_union').length;
  const mgCount  = stores.filter(s => s.category === 'moneygram').length;
  const riaCount = stores.filter(s => s.category === 'ria').length;
  const totalSignals = stores.filter(s => DEMAND_WEIGHTS[s.category]).length;

  const expandRows = expand.map((z, i) => `
    <div class="narrative-item">
      <div class="narrative-rank">${i + 1}</div>
      <div>
        <div class="narrative-item-title">Cluster near (${z.centroid.lat.toFixed(3)}, ${z.centroid.lng.toFixed(3)})</div>
        <div class="narrative-item-desc">Adjusted demand score: ${z.adjDemand} · ${z.members.length} demand signals · ${z.compBonus > 0 ? `+${z.compBonus} competitor validation bonus` : 'No competitor validation'} · 0 WU locations within ${WU_RADIUS_MI}mi</div>
      </div>
    </div>`).join('') || '<div class="narrative-empty">No expansion zones identified with current data.</div>';

  const reduceRows = reduce.map((z, i) => `
    <div class="narrative-item">
      <div class="narrative-rank">${i + 1}</div>
      <div>
        <div class="narrative-item-title">Cluster near (${z.centroid.lat.toFixed(3)}, ${z.centroid.lng.toFixed(3)})</div>
        <div class="narrative-item-desc">Adjusted demand score: ${z.adjDemand} (below threshold) · ${z.wuCount} WU locations within ${WU_RADIUS_MI}mi · Low market justification for WU density</div>
      </div>
    </div>`).join('') || '<div class="narrative-empty">No saturation concerns identified with current data.</div>';

  document.getElementById('narrative').innerHTML = `
    <div class="narrative-header">COMPETITIVE INTELLIGENCE SUMMARY — ZIP 33147</div>
    <div class="narrative-grid">
      <div class="narrative-col expand">
        <div class="narrative-col-title">▲ OPEN NEW LOCATIONS</div>
        ${expandRows}
      </div>
      <div class="narrative-col reduce">
        <div class="narrative-col-title">▼ REDUCE FOOTPRINT</div>
        ${reduceRows}
      </div>
    </div>
    <div class="narrative-context">
      ZIP 33147 has ${wuCount + mgCount + riaCount} remittance locations across 3 providers within the mapped area.
      Western Union: ${wuCount} · MoneyGram: ${mgCount} · Ria: ${riaCount}.
      ${totalSignals} demand-signal businesses (ethnic groceries, financial services, dollar stores, laundromats) were mapped.
      ${expand.length > 0 ? `${expand.length} expansion zone${expand.length > 1 ? 's' : ''} identified where demand is strong but WU is absent.` : 'No clear expansion zones identified.'}
      ${reduce.length > 0 ? `${reduce.length} saturation zone${reduce.length > 1 ? 's' : ''} flagged where WU density exceeds local demand justification.` : 'No saturation concerns identified.'}
    </div>
  `;
}
```

- [ ] **Step 9.2 — Add narrative CSS**

Add inside `<style>`:

```css
.narrative-header { font-size: 11px; color: #666; letter-spacing: 1px; text-transform: uppercase; margin-bottom: 14px; }
.narrative-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; margin-bottom: 14px; }
.narrative-col-title { font-size: 12px; font-weight: 700; margin-bottom: 10px; }
.narrative-col.expand .narrative-col-title { color: #3fb950; }
.narrative-col.reduce  .narrative-col-title { color: #f85149; }
.narrative-item { display: flex; gap: 10px; margin-bottom: 10px; }
.narrative-rank { background: #2a3040; color: #ccc; border-radius: 50%; width: 20px; height: 20px; display: flex; align-items: center; justify-content: center; font-size: 11px; font-weight: 700; flex-shrink: 0; }
.narrative-item-title { font-size: 12px; font-weight: 600; color: #e6edf3; margin-bottom: 2px; }
.narrative-item-desc { font-size: 11px; color: #8b949e; line-height: 1.5; }
.narrative-empty { font-size: 12px; color: #555; font-style: italic; }
.narrative-context { font-size: 12px; color: #8b949e; line-height: 1.7; border-top: 1px solid #2a3040; padding-top: 12px; }
```

- [ ] **Step 9.3 — Call renderNarrative in DOMContentLoaded**

Update the init block:

```js
document.addEventListener('DOMContentLoaded', () => {
  renderToggles();
  renderPins();
  renderZones();
  renderScorecard();
  renderNarrative();
});
```

- [ ] **Step 9.4 — Verify narrative section**

Open/refresh in browser.

Expected: Below the map, a two-column grid appears showing ranked expansion recommendations (left, green) and reduce footprint concerns (right, red). Market context paragraph auto-generated with correct store counts. No JS errors.

- [ ] **Step 9.5 — Commit**

```bash
git add zip_33147_wu_competitive_map.html
git commit -m "feat: auto-generated analysis narrative with ranked recommendations"
```

---

## Task 10: PDF Export + Final Polish

**Files:**
- Modify: `zip_33147_wu_competitive_map.html`

- [ ] **Step 10.1 — Add exportPDF function**

Add to JS:

```js
// ─── PDF EXPORT ───────────────────────────────────────────────────────────────
function exportPDF() {
  const btn = document.getElementById('pdf-btn');
  btn.textContent = 'Generating...';
  btn.disabled = true;
  const opt = {
    margin:      [8, 8],
    filename:    'wu_competitive_intel_33147.pdf',
    image:       { type: 'jpeg', quality: 0.95 },
    html2canvas: { scale: 2, useCORS: true },
    jsPDF:       { unit: 'mm', format: 'a4', orientation: 'landscape' }
  };
  html2pdf().set(opt).from(document.getElementById('app')).save()
    .then(() => { btn.textContent = '⬇ Export PDF'; btn.disabled = false; });
}
```

- [ ] **Step 10.2 — Add final polish CSS**

Add inside `<style>`:

```css
/* Scrollbar */
::-webkit-scrollbar { width: 6px; } ::-webkit-scrollbar-track { background: #0f1117; } ::-webkit-scrollbar-thumb { background: #2a3040; border-radius: 3px; }
/* Map pin cursor */
.pin { cursor: pointer; }
/* Responsive fallback */
@media (max-width: 800px) { #main-content { grid-template-columns: 1fr; } #right-panel { display: none; } }
/* Print / PDF hide toggle bar */
@media print { #toggle-bar { display: none; } #pdf-btn { display: none; } }
```

- [ ] **Step 10.3 — Full end-to-end verification**

Open `zip_33147_wu_competitive_map.html` in browser and run through this checklist:

1. Page loads with dark theme, header, toggle bar, map, right panel, narrative — no JS errors (F12)
2. All 9 toggle buttons visible and functional: each hides/shows correct pins or zone layers
3. Click a Western Union pin → gold badge, name, address, rating, hours, Google Maps link appear
4. Click a demand signal pin (small) → correct badge color and store info appear
5. Toggle "◎ Expansion Zones" → green ellipses appear at high-demand / no-WU clusters
6. Toggle "◎ Saturation Zones" → red ellipses appear at WU-oversaturated clusters
7. MONITOR zones (amber) visible if data produces any `adjDemand ≥ 5 AND wuCount ≥ 2` clusters
8. Right panel scorecard shows correct zone counts and WU/MG/Ria totals
9. Narrative two-column section shows ranked recommendations auto-generated from zones
10. Market context paragraph counts match actual stores in the JS array
11. Click "⬇ Export PDF" → PDF downloads as `wu_competitive_intel_33147.pdf` in landscape format

- [ ] **Step 10.4 — Final commit**

```bash
git add zip_33147_wu_competitive_map.html
git commit -m "feat: PDF export, final polish, complete WU competitive map for ZIP 33147"
```

---

## Spec Coverage Verification

| Spec Requirement | Task |
|---|---|
| Single self-contained HTML file | Task 1 |
| All store data in JS array | Task 1 |
| 7 pin categories with correct colors/sizes | Task 3 |
| Programmatic pin rendering | Task 3 |
| 9 toggle buttons (7 categories + 2 zone types) | Task 4 |
| Pin click → detail panel with Maps link | Task 5 |
| Haversine distance function | Task 6 |
| Demand signal scoring with weights | Task 6 |
| Competitor presence as +2 validation bonus | Task 6 |
| Zone classification: EXPAND / MONITOR / REDUCE | Task 6 |
| Algorithmic zone ellipses sized by cluster density | Task 7 |
| Expansion and saturation toggles independent | Task 4 + 7 |
| Zone analysis scorecard (right panel) | Task 8 |
| Auto-generated analysis narrative | Task 9 |
| PDF export | Task 10 |
| SVG street grid for ZIP 33147 | Task 2 |
| Coordinate mapping formula | Task 1 + 3 |
| Tunable threshold constants at top of file | Task 1 |
