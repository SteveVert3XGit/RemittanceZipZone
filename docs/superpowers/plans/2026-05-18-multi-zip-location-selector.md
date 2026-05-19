# Multi-ZIP Location Selector — Phase 2 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add ZIP 33463 (Lake Worth / Greenacres, Palm Beach County) to the Western Union Location Selector, with a dropdown in the header that switches between ZIPs and re-renders all content — map, pins, zones, scorecard, narrative, and demographics — without a page reload.

**Architecture:** All per-ZIP data (stores, map bounds, street grid, corridor labels, demographics) is extracted from globals and hardcoded HTML into a single `ZIP_CONFIG` object keyed by ZIP string. An `activeZip` state variable drives all rendering. A `selectZip(zip)` function invalidates the zone cache and triggers a full re-render. CATEGORIES, DEMAND_WEIGHTS, zone thresholds, and algorithm constants remain global (same for every ZIP).

**Tech Stack:** Pure HTML/CSS/JS — no frameworks, no build tools, same as Phase 1.

---

## Files

| File | Action |
|---|---|
| `zip_33147_wu_competitive_map.html` | Rename → `wu_location_selector.html`, refactor contents |

---

## Task 1: Rename file and extract 33147 data into ZIP_CONFIG

**File:** `zip_33147_wu_competitive_map.html` → `wu_location_selector.html`

This task restructures per-ZIP globals and hardcoded HTML into a data object. No visual change — the tool should look and behave identically after this task. Commit at end to lock in the rename before further edits.

- [ ] **Step 1: Rename the file**

```bash
git mv zip_33147_wu_competitive_map.html wu_location_selector.html
```

- [ ] **Step 2: Replace `const MAP = {...}` and `const stores = [...]` with `ZIP_CONFIG`**

In the JS block, **replace** the `const MAP = {...}` block and `const stores = [...]` declaration with this structure (the `stores` array content stays identical — only its location in the data structure changes):

```js
// ─── PER-ZIP CONFIGURATION ────────────────────────────────────────────────────
const ZIP_CONFIG = {
  '33147': {
    label: 'ZIP 33147 — Liberty City / NW Miami',
    map: {
      latMin: 25.813, latMax: 25.853, latRange: 0.040,
      lngMin: -80.251, lngMax: -80.213, lngRange: 0.038,
      svgW: 700, svgH: 520
    },
    corridors: {
      ns: [
        { name: 'NW 27th Ave', lng: -80.249 },
        { name: 'NW 17th Ave', lng: -80.234 },
        { name: 'NW 7th Ave',  lng: -80.222 }
      ],
      ew: [
        { name: 'NW 79th St', lat: 25.851 },
        { name: 'NW 62nd St', lat: 25.837 },
        { name: 'NW 54th St', lat: 25.831 },
        { name: 'NW 36th St', lat: 25.813 }
      ]
    },
    streets: [
      { type: 'ew', y: 27,  weight: 3, label: 'NW 79th St', labelX: 42,  labelY: 23  },
      { type: 'ew', y: 205, weight: 3, label: 'NW 62nd St', labelX: 42,  labelY: 201 },
      { type: 'ew', y: 287, weight: 2, label: 'NW 54th St', labelX: 42,  labelY: 283 },
      { type: 'ew', y: 520, weight: 2, label: 'NW 36th St', labelX: 42,  labelY: 516 },
      { type: 'ns', x: 37,  weight: 3, label: 'NW 27th Ave' },
      { type: 'ns', x: 313, weight: 2, label: 'NW 17th Ave' },
      { type: 'ns', x: 534, weight: 3, label: 'NW 7th Ave'  }
    ],
    demographics: {
      areaName: 'Liberty City / NW Miami',
      population: '~35,200',
      foreignBorn: { pct: '38%', count: '~13,400 residents' },
      unbanked:    { pct: '~35%', note: 'vs. 5.9% national avg' },
      poverty:     { pct: '33%',  medianHHI: '$28,500' },
      diaspora: [
        { name: 'Haitian',              pct: '~22%' },
        { name: 'Black / African American', pct: '~68%' },
        { name: 'Hispanic / Latino',    pct: '~27%' },
        { name: 'Cuban',                pct: '~8%'  },
        { name: 'Caribbean (other)',    pct: '~6%'  },
        { name: 'Per Capita Income',    pct: '$11,800' }
      ],
      context: 'ZIP 33147 is one of Miami\'s highest-density remittance corridors. The Haitian diaspora (~22%) is the dominant transnational sender group, with strong ties to Port-au-Prince and Croix-des-Bouquets. A high unbanked rate (~35%) means cash-based remittance services — not digital apps — remain the primary channel. Low median household income ($28,500) and high poverty (33%) reinforce reliance on informal and agent-based transfer networks.'
    },
    stores: [
      /* ── PASTE THE ENTIRE EXISTING stores ARRAY CONTENT HERE, unchanged ── */
    ]
  }
};

let activeZip = '33147';
function getZip() { return ZIP_CONFIG[activeZip]; }
```

- [ ] **Step 3: Remove the `<g id="streets">` hardcoded SVG content from HTML**

In the HTML `<svg>` block, replace the entire contents of `<g id="streets">...</g>` with just the empty element:

```html
<g id="streets"></g>
```

The street lines and labels will now be drawn by `renderStreets()` (added in Task 2).

- [ ] **Step 4: Remove the hardcoded `<div id="demographics">` content from HTML**

Replace the entire contents of `<div id="demographics">` with an empty element:

```html
<div id="demographics"></div>
```

The demographics content will now be rendered by `renderDemographics()` (added in Task 2).

- [ ] **Step 5: Update `<span id="zip-label">` to be empty — it will be set by JS**

```html
<span id="zip-label"></span>
```

- [ ] **Step 6: Verify the page still loads without JS errors**

Open `wu_location_selector.html` in browser. The map SVG canvas, toggle bar, and header should render. Streets and demographics will be blank until Task 2 wires up the render functions. No console errors expected.

- [ ] **Step 7: Commit**

```bash
git add wu_location_selector.html
git commit -m "refactor: extract 33147 data into ZIP_CONFIG, rename to wu_location_selector"
```

---

## Task 2: Dynamic street rendering, demographics rendering, and ZIP-aware functions

Wire all rendering functions to use `getZip()` instead of the old `MAP` and `stores` globals. Add `renderStreets()` and `renderDemographics()` functions. Update INIT to call them.

- [ ] **Step 1: Update `latLngToSVG` to use `getZip().map`**

```js
function latLngToSVG(lat, lng) {
  const m = getZip().map;
  return {
    x: Math.min(m.svgW, Math.max(0, ((lng - m.lngMin) / m.lngRange) * m.svgW)),
    y: Math.min(m.svgH, Math.max(0, (1 - (lat - m.latMin) / m.latRange) * m.svgH))
  };
}
```

- [ ] **Step 2: Update `renderPins` to use `getZip().stores`**

Change the `.forEach` source from `stores.forEach` to `getZip().stores.forEach`:

```js
function renderPins() {
  const layer = document.getElementById('pins-layer');
  layer.innerHTML = '';
  getZip().stores.forEach(s => {
    // ... rest of function unchanged
  });
}
```

- [ ] **Step 3: Update `computeZones` to use `getZip().stores`**

Change the three filter lines at the top of `computeZones`:

```js
function computeZones() {
  if (_zonesCache) return _zonesCache;
  const allStores   = getZip().stores;
  const demandStores = allStores.filter(s => DEMAND_WEIGHTS[s.category] !== undefined && !s.bigbox);
  const competitors  = allStores.filter(s => (s.category === 'moneygram' || s.category === 'ria') && !s.bigbox);
  const wuStores     = allStores.filter(s => s.category === 'western_union' && !s.bigbox);
  // ... rest of function unchanged
```

- [ ] **Step 4: Update `nearestCorridor` to use `getZip().corridors`**

```js
function nearestCorridor(lat, lng) {
  const { ns, ew } = getZip().corridors;
  const nsStreet = ns.reduce((a, b) => Math.abs(lng - a.lng) < Math.abs(lng - b.lng) ? a : b);
  const ewStreet = ew.reduce((a, b) => Math.abs(lat - a.lat) < Math.abs(lat - b.lat) ? a : b);
  return `${nsStreet.name} @ ${ewStreet.name}`;
}
```

- [ ] **Step 5: Update `renderScorecard` provider counts to use `getZip().stores`**

```js
const allStores = getZip().stores;
const wuCount  = allStores.filter(s => s.category === 'western_union').length;
const mgCount  = allStores.filter(s => s.category === 'moneygram').length;
const riaCount = allStores.filter(s => s.category === 'ria').length;
```

- [ ] **Step 6: Update `renderNarrative` total signal count to use `getZip().stores`**

```js
const totalSignals = getZip().stores.filter(s => DEMAND_WEIGHTS[s.category] !== undefined).length;
```

- [ ] **Step 7: Add `renderStreets()` function (add before `renderPins`)**

```js
// ─── STREET GRID ──────────────────────────────────────────────────────────────
function renderStreets() {
  const layer = document.getElementById('streets');
  layer.innerHTML = '';
  const m = getZip().map;
  const SVG_NS = 'http://www.w3.org/2000/svg';

  getZip().streets.forEach(s => {
    const line = document.createElementNS(SVG_NS, 'line');
    if (s.type === 'ew') {
      line.setAttribute('x1', 0);   line.setAttribute('y1', s.y);
      line.setAttribute('x2', m.svgW); line.setAttribute('y2', s.y);
    } else {
      line.setAttribute('x1', s.x); line.setAttribute('y1', 0);
      line.setAttribute('x2', s.x); line.setAttribute('y2', m.svgH);
    }
    line.setAttribute('stroke', '#2a3340');
    line.setAttribute('stroke-width', s.weight);
    layer.appendChild(line);

    const text = document.createElementNS(SVG_NS, 'text');
    text.setAttribute('fill', '#3a4a5a');
    text.setAttribute('font-size', '9');
    text.setAttribute('font-family', 'monospace');
    if (s.type === 'ew') {
      text.setAttribute('x', s.labelX);
      text.setAttribute('y', s.labelY);
    } else {
      const midY = m.svgH / 2;
      text.setAttribute('x', s.x);
      text.setAttribute('y', midY);
      text.setAttribute('transform', `rotate(-90 ${s.x} ${midY})`);
      text.setAttribute('text-anchor', 'middle');
    }
    text.textContent = s.label;
    layer.appendChild(text);
  });
}
```

- [ ] **Step 8: Add `renderDemographics()` function (add after `renderNarrative`)**

```js
// ─── DEMOGRAPHICS ─────────────────────────────────────────────────────────────
function renderDemographics() {
  const d = getZip().demographics;
  document.getElementById('demographics').innerHTML = `
    <div class="demo-header">ZIP ${activeZip} · Population Profile · Source: U.S. Census ACS 2019–2023</div>
    <div class="demo-grid">
      <div class="demo-card">
        <div class="demo-card-label">Total Population</div>
        <div class="demo-card-value">${d.population}</div>
        <div class="demo-card-sub">${d.areaName}</div>
      </div>
      <div class="demo-card highlight">
        <div class="demo-card-label">Foreign-Born</div>
        <div class="demo-card-value">${d.foreignBorn.pct}</div>
        <div class="demo-card-sub">${d.foreignBorn.count}</div>
      </div>
      <div class="demo-card highlight">
        <div class="demo-card-label">Est. Unbanked Rate</div>
        <div class="demo-card-value">${d.unbanked.pct}</div>
        <div class="demo-card-sub">${d.unbanked.note}</div>
      </div>
      <div class="demo-card">
        <div class="demo-card-label">Poverty Rate</div>
        <div class="demo-card-value">${d.poverty.pct}</div>
        <div class="demo-card-sub">Median HHI ${d.poverty.medianHHI}</div>
      </div>
    </div>
    <div class="demo-header" style="margin-top:4px">Diaspora Concentration</div>
    <div class="demo-diaspora-grid">
      ${d.diaspora.map(item => `
        <div class="demo-diaspora-item">
          <span class="demo-diaspora-name">${item.name}</span>
          <span class="demo-diaspora-pct">${item.pct}</span>
        </div>`).join('')}
    </div>
    <div class="demo-context">${d.context}</div>
  `;
}
```

- [ ] **Step 9: Update INIT block to call `renderStreets` and `renderDemographics`, and set the zip label**

```js
document.addEventListener('DOMContentLoaded', () => {
  document.getElementById('zip-label').textContent = getZip().label;
  renderToggles();
  renderStreets();
  renderPins();
  renderZones();
  renderScorecard();
  renderNarrative();
  renderDemographics();
});
```

- [ ] **Step 10: Verify in browser**

Open `wu_location_selector.html`. Streets should render identically to before. Demographics section should appear. All pins, zones, toggles, scorecard, and narrative should work as before. No console errors.

- [ ] **Step 11: Commit**

```bash
git add wu_location_selector.html
git commit -m "refactor: dynamic street/demographics rendering, all functions ZIP-aware via getZip()"
```

---

## Task 3: ZIP selector dropdown UI and selectZip function

Add the dropdown to the header and wire it to a `selectZip(zip)` function that invalidates the zone cache and triggers a full re-render of all layers.

- [ ] **Step 1: Add CSS for the ZIP selector (add to `<style>` block)**

```css
#zip-select { background: #1a1f2e; color: #e6edf3; border: 1px solid #2a3040; border-radius: 6px; padding: 4px 10px; font-size: 13px; font-family: inherit; cursor: pointer; }
#zip-select:focus { outline: none; border-color: #F5C518; }
```

- [ ] **Step 2: Replace the static `<span id="zip-label">` in the header with a `<select>` element**

```html
<header id="top-bar">
  <div id="title-block">
    <span id="logo">⬡</span>
    <span id="title-text">Western Union Location Selector</span>
  </div>
  <div style="display:flex;align-items:center;gap:10px">
    <select id="zip-select" onchange="selectZip(this.value)">
      <option value="33147">ZIP 33147 — Liberty City / NW Miami</option>
      <option value="33463">ZIP 33463 — Lake Worth / Greenacres</option>
    </select>
    <button id="pdf-btn" onclick="exportPDF()">⬇ Export PDF</button>
  </div>
</header>
```

- [ ] **Step 3: Add `selectZip(zip)` function (add near top of script, after `getZip` definition)**

```js
function selectZip(zip) {
  activeZip = zip;
  _zonesCache = null;
  activeToggles.clear();
  Object.keys(CATEGORIES).forEach(k => activeToggles.add(k));
  activeToggles.add('zones_expand');
  activeToggles.add('zones_monitor');
  activeToggles.add('zones_reduce');
  renderToggles();
  renderStreets();
  renderPins();
  renderZones();
  renderScorecard();
  renderNarrative();
  renderDemographics();
}
```

- [ ] **Step 4: Update INIT — remove `zip-label` textContent line (no longer needed)**

```js
document.addEventListener('DOMContentLoaded', () => {
  renderToggles();
  renderStreets();
  renderPins();
  renderZones();
  renderScorecard();
  renderNarrative();
  renderDemographics();
});
```

- [ ] **Step 5: Verify in browser**

Open `wu_location_selector.html`. Header shows dropdown with "ZIP 33147 — Liberty City / NW Miami" selected. Switching to "ZIP 33463" re-renders the map (will show empty — no 33463 data yet, added in Task 5). Switching back to 33147 restores all content correctly. No console errors on either selection.

- [ ] **Step 6: Commit**

```bash
git add wu_location_selector.html
git commit -m "feat: ZIP selector dropdown with selectZip re-render"
```

---

## Task 4: Research ZIP 33463 store data and geography

Research all store locations, coordinate bounds, street corridors, and demographics for ZIP 33463 (Lake Worth / Greenacres, Palm Beach County, FL).

**Do this research using WebSearch and WebFetch.**

- [ ] **Step 1: Determine ZIP 33463 lat/lng bounds**

Search: `"ZIP code 33463" bounds latitude longitude site:census.gov OR site:unitedstateszipcodes.org`

Expected: The ZIP covers roughly lat 26.575–26.630, lng -80.165–-80.095. Confirm the four boundary values and record them. Calculate:
- `latRange = latMax - latMin`
- `lngRange = lngMax - lngMin`

These drive the SVG coordinate mapping:
```
svgX = (lng - lngMin) / lngRange × 700
svgY = (1 - (lat - latMin) / latRange) × 520
```

- [ ] **Step 2: Identify the 4–5 major streets in 33463 for the street grid**

Search: `major roads ZIP 33463 Lake Worth Greenacres Florida`

Expected corridors:
- E/W: Hypoluxo Rd (~lat 26.626), Lake Worth Rd (~26.612), 10th Ave N (~26.597), Lantana Rd (~26.584)
- N/S: Congress Ave (~lng -80.128), Military Trail (~-80.113), Jog Rd (~-80.098)

For each street, record the actual lat or lng, then calculate its SVG coordinate using the bounds from Step 1.

- [ ] **Step 3: Find Western Union locations in ZIP 33463**

Search: `Western Union locations "Lake Worth" OR "Greenacres" FL 33463`
Also check: Google Maps search for "Western Union 33463"

Record each location as:
```js
{
  id: 'wu_001',  // sequential
  category: 'western_union',
  name: 'Western Union @ [Host Store]',
  address: '[full address]',
  lat: [decimal],
  lng: [decimal],
  rating: [google rating or null],
  reviews: [review count or null],
  hours: '[hours string or null]',
  notes: '[brief note about host store]',
  bigbox: true  // add if inside Walmart, Walgreens, CVS, etc.
}
```

Target: 3–6 WU locations. If fewer exist, that's fine — it reflects real market data.

- [ ] **Step 4: Find MoneyGram locations in ZIP 33463**

Search: `MoneyGram locations "Lake Worth" OR "Greenacres" FL 33463`

Same schema as Step 3, `category: 'moneygram'`. Target: 3–6 locations.

- [ ] **Step 5: Find Ria Money Transfer locations in ZIP 33463**

Search: `Ria Money Transfer "Lake Worth" OR "Greenacres" FL 33463`

Same schema, `category: 'ria'`. Target: 2–4 locations.

- [ ] **Step 6: Find demand signal stores in ZIP 33463**

Search each category separately:
- `ethnic grocery store "Lake Worth" OR "Greenacres" FL 33463` → `category: 'ethnic_grocery'`, target 5–8
- `check cashing payday loans "Lake Worth" OR "Greenacres" FL 33463` → `category: 'financial_services'`, target 4–6
- `"Family Dollar" OR "Dollar Tree" OR "Dollar General" "Lake Worth" FL 33463` → `category: 'dollar_store'`, target 3–5
- `laundromat "Lake Worth" OR "Greenacres" FL 33463` → `category: 'laundromat'`, target 2–4

- [ ] **Step 7: Look up ZIP 33463 demographics (ACS 2019–2023)**

Search: `ZIP 33463 demographics census foreign born income poverty`
Also: `site:census.gov "33463"` or use https://censusreporter.org/profiles/86000US33463-33463/

Record:
- Total population
- % foreign-born (and approx count)
- Racial/ethnic composition (Hispanic %, Black/AA %, etc.)
- Key diaspora groups (Guatemalan, Haitian, Puerto Rican, etc. — varies from 33147)
- Median household income
- Poverty rate
- Per capita income
- Estimated unbanked rate (use https://www.fdic.gov/analysis/household-survey if needed)

- [ ] **Step 8: No commit — data collected, used in Task 5**

---

## Task 5: Add ZIP 33463 data to ZIP_CONFIG

Add the researched data as a `'33463'` entry in `ZIP_CONFIG`.

- [ ] **Step 1: Calculate SVG coordinates for 33463 streets**

Using the bounds from Task 4 Step 1 and the formula:
```
svgX = (lng - lngMin) / lngRange × 700   // for N/S streets
svgY = (1 - (lat - latMin) / latRange) × 520  // for E/W streets
```

For each street found in Task 4 Step 2, calculate its `x` or `y` SVG value. Also calculate `labelX/labelY` for E/W streets (labelX = 42, labelY = svgY - 4) and leave N/S streets with just `x` (the renderStreets function will center the label vertically).

- [ ] **Step 2: Add `'33463'` entry to `ZIP_CONFIG`**

Add after the `'33147'` entry, inside `ZIP_CONFIG`:

```js
'33463': {
  label: 'ZIP 33463 — Lake Worth / Greenacres',
  map: {
    latMin: [from Task 4 Step 1],
    latMax: [from Task 4 Step 1],
    latRange: [latMax - latMin],
    lngMin: [from Task 4 Step 1],
    lngMax: [from Task 4 Step 1],
    lngRange: [lngMax - lngMin],
    svgW: 700, svgH: 520
  },
  corridors: {
    ns: [
      { name: '[Street Name]', lng: [decimal] },
      // ... one entry per N/S street found
    ],
    ew: [
      { name: '[Street Name]', lat: [decimal] },
      // ... one entry per E/W street found
    ]
  },
  streets: [
    // E/W streets: { type: 'ew', y: [svgY], weight: 2, label: '[name]', labelX: 42, labelY: [svgY - 4] }
    // N/S streets: { type: 'ns', x: [svgX], weight: 2, label: '[name]' }
    // Use weight: 3 for the most prominent arteries (Lake Worth Rd, Congress Ave), weight: 2 for others
  ],
  demographics: {
    areaName: 'Lake Worth / Greenacres',
    population: '[from Task 4 Step 7]',
    foreignBorn: { pct: '[pct]%', count: '[approx count] residents' },
    unbanked:    { pct: '~[pct]%', note: 'vs. 5.9% national avg' },
    poverty:     { pct: '[pct]%', medianHHI: '$[amount]' },
    diaspora: [
      // { name: '[group]', pct: '[pct]%' } for each major group found
    ],
    context: '[2-3 sentence market context for 33463 — note key diaspora groups, unbanked rate, remittance sending patterns]'
  },
  stores: [
    // All stores from Task 4 Steps 3–6, following exact same schema as 33147 stores
  ]
}
```

- [ ] **Step 3: Verify 33463 in browser**

Open `wu_location_selector.html`, switch dropdown to "ZIP 33463 — Lake Worth / Greenacres". Verify:
- Street grid renders correct corridors for 33463
- Pins appear within the map canvas (none clipped to edges)
- Toggle all 7 categories — pins show/hide correctly
- Click a pin — detail panel shows name, address, maps link
- Toggle Expansion/Saturation/Monitor zones — ellipses appear in reasonable locations
- Demographics section shows 33463 data
- Scorecard shows correct provider counts for 33463
- Switch back to 33147 — all 33147 content restores correctly

If any pins render off-canvas: recalculate lat/lng bounds. The formula `svgY = (1 - (lat - latMin)/latRange) × 520` should clamp between 0–520.

- [ ] **Step 4: Commit**

```bash
git add wu_location_selector.html
git commit -m "feat: add ZIP 33463 Lake Worth/Greenacres data — stores, street grid, demographics"
```

---

## Task 6: Push to GitHub and verify

- [ ] **Step 1: Push**

```bash
git push origin main
```

- [ ] **Step 2: Final end-to-end verification checklist**

Open `wu_location_selector.html` in browser and confirm all of the following:

| Check | 33147 | 33463 |
|---|---|---|
| Street grid renders | ✓ | ✓ |
| All 7 pin categories visible | ✓ | ✓ |
| Bigbox pins dimmed | ✓ | ✓ |
| Expansion zones fire (if data supports) | ✓ | ✓ |
| Saturation zones fire (if data supports) | ✓ | ✓ |
| Zone labels show intersections (not lat/lng) | ✓ | ✓ |
| Scorecard shows correct provider counts | ✓ | ✓ |
| Narrative shows correct zone summaries | ✓ | ✓ |
| Demographics shows ZIP-specific data | ✓ | ✓ |
| Switching ZIPs resets toggles and zones | ✓ | — |
| PDF export works | ✓ | ✓ |

---

## Critical Files

- **Modify:** `zip_33147_wu_competitive_map.html` → `wu_location_selector.html`
- **Reference (existing 33147 data):** Current stores array at line ~267 in `zip_33147_wu_competitive_map.html`
- **Research reference:** https://censusreporter.org/profiles/86000US33463-33463/ for 33463 demographics
