# CLAUDE.md — Bimma Retail FBA Ops Dashboard

## Architecture

**Single-file SPA.** Everything — CSS, HTML, JavaScript — lives in `index.html` (~345 KB). There is no build step, no bundler, no modules, no separate asset files. Editing is done directly in this one file.

Current version: **v2.9 · 12 Mar 26**

### Layout structure
```
body (flex row)
├── #sidebar (fixed, 240px wide, z-index 200)
│   ├── .sidebar-logo
│   ├── .sidebar-nav  (nav-item elements, each calls showPage())
│   └── .sidebar-footer (auth dot + Google connect button)
└── #main (margin-left: 240px)
    ├── .topbar (sticky, 56px, z-index 100)
    └── .page divs (only one has .active at a time)
```

### Page navigation
```js
showPage(name, el)
```
- Toggles `.active` on `.page` and `.nav-item` elements
- Updates `#topbarTitle` text
- Fires `document.dispatchEvent(new CustomEvent('bimma:pagechange', { detail: name }))`
- **Listen to `bimma:pagechange`** if you need to react to page switches

---

## Pages

| `data-page` / `id` suffix | Title | Notes |
|---|---|---|
| `dashboard` | Dashboard | Default active page; Google Sheets–driven |
| `refund` | Refund Monitor | CSV upload; return requests in localStorage |
| `deals` | Deal Analyser | **Deprecated** — redirects to `a2a` |
| `cogs` | COGS Calculator | CSV upload; date-range picker |
| `repricer` | Min Price Tool | Keepa BB history + purchase floor prices |
| `a2a` | A2A Analyser | 3-mode: EU / UK / ULN; CSV upload; Keepa enrichment |
| `scout` | Deal Scout | Keepa-based deal finder |
| `purchases` | Purchases | Google Sheets read/write |
| `listing` | Listing Uploader | Google Sheets append |
| `add` | Add Purchase | Form; writes to Sheets |
| `expenses` | Expenses | Card grid; values stored in localStorage |
| `stalker` | Storefront Stalker | Keepa seller monitoring; feed in localStorage |
| `leakage` | Leakage Tracker | CSV upload |

---

## CSS Custom Properties (`:root`)

### Backgrounds (dark → darker is NOT the order — surface layers up)
```css
--bg:        #060608   /* page background */
--surface:   #0d0d12   /* sidebar, cards */
--surface2:  #121219   /* inputs, modals */
--surface3:  #18181f   /* focused inputs, deeper panels */
--surface4:  #1d1d28   /* toggle track */
```

### Borders
```css
--border:   #1e1e2a   /* default card/table borders */
--border2:  #28283a   /* inputs, ghost buttons */
--border3:  #32324a   /* hover state borders */
```

### Brand colours
```css
--green:      #00e88a   /* primary accent — profit, active states */
--green-glow: rgba(0,229,132,0.18)
--green-dim:  #00b868
--amber:      #f0a500   /* warning, ULN mode */
--amber-glow: rgba(240,165,0,0.15)
--red:        #f03b50   /* negative, reject */
--red-glow:   rgba(240,59,80,0.12)
--purple:     #9b6dff   /* optional/info callouts */
--blue:       #4d8ef5   /* EU mode, Keepa links */
```

### Text (light → dim)
```css
--text:   #f0f0f8   /* headings, primary */
--text-2: #a0a0c0   /* body, secondary */
--text-3: #70708a   /* labels, meta */
--text-4: #22222e   /* near-black — very dim placeholders, empty states */
```

### Typography
```css
--sans: 'DM Sans', system-ui, sans-serif   /* body, nav, headings */
--mono: 'DM Mono', monospace               /* labels, numbers, buttons, code */
```

> **Rule:** Numbers and monospace labels always use `var(--mono)`. Section labels are `font-family: var(--mono); font-size: 9–10px; letter-spacing: 0.12–0.2em; text-transform: uppercase`.

### Layout
```css
--sidebar-w: 240px   /* collapses to 0px at ≤768px */
--topbar-h:  56px
```

### Border radii
```css
--r-xs: 5px   /* filter pills, small badges */
--r-sm: 8px   /* inputs, buttons, small cards */
--r:    14px  /* standard cards, table wrappers */
--r-lg: 18px  /* modals */
--r-xl: 22px  /* (available, less used) */
```

---

## localStorage Keys

| Key | Type | Used by |
|---|---|---|
| `bimma_goog_cid` | `string` | Google OAuth client ID (entered by user) |
| `bimma_prep` | `JSON object` | Prep page data |
| `bimma_expenses` | `JSON object` | Expenses totals per period |
| `bimma_exp_items` | `JSON array` | Expense card definitions (falls back to `EXP_DEFAULTS`) |
| `bimma_returnRequests` | `JSON object` | Refund Monitor — return requests keyed by ASIN |
| `bimma_customWindows` | `JSON object` | Refund Monitor — custom return windows keyed by ASIN |
| `bimma-a2a-veto` | `JSON array → Set` | A2A Analyser — vetoed ASINs |
| `bimma-a2a-viewed` | `JSON array → Set` | A2A Analyser — viewed ASINs |
| `bimma-a2a-rejected` | `JSON object → Map` | A2A Analyser — rejected ASINs with reason |
| `bimma_keepa_key` | `string` | Keepa API key (A2A, COGS, Min Price) |
| `bimma_stalker_key` | `string` | Keepa API key for Stalker (also used as fallback by other tools) |
| `bimma_scout_viewed` | `JSON array → Set` | Deal Scout — viewed ASINs |
| `bimma_stalker_sellers` | `JSON array` | Storefront Stalker — tracked sellers |
| `bimma_stalker_snaps` | `JSON object` | Storefront Stalker — ASIN snapshots |
| `bimma_stalker_feed` | `JSON array` | Storefront Stalker — activity feed (capped at 500 items) |

> **Note on Keepa key:** `bimma_keepa_key` and `bimma_stalker_key` both store Keepa keys. Most tools check `bimma_keepa_key` first then fall back to `bimma_stalker_key`.

---

## External Integrations

### Google Sheets
- **Sheet ID:** `17woLJlpqKBV6TEDROTVuNSTxvvCASlja_cCNmBSXtV0`
- **Tabs:** `'Purchasing sheet'`, `'Listing loader'`
- **Auth:** Google Identity Services (`google.accounts.oauth2`), loaded dynamically
- **Scope:** `https://www.googleapis.com/auth/spreadsheets`
- OAuth client ID stored in `bimma_goog_cid`; entered by user in sidebar footer

### Keepa API
- Base: `https://api.keepa.com`
- Endpoints used: `/product`, `/token`, `/seller`, `/query`
- Used by: A2A Analyser, Deal Scout, Min Price Tool, COGS Calculator, Storefront Stalker

---

## A2A Analyser Modes

| Mode | CSS accent | Trigger colour |
|---|---|---|
| `eu` | `--blue` (#4d8ef5) | Default active mode |
| `uk` | `--green` (#00e88a) | UK arbitrage |
| `uln` | `--amber` (#f0a500) | ULN / wholesale |

Mode buttons call `setMode(mode, el)`. EU mode requires two file slots loaded; UK/ULN require one.

---

## Z-index Layers

| Layer | Value |
|---|---|
| Noise texture overlay (`body::before`) | 9999 |
| Toast notification | 9998 |
| Modal overlay | 500 |
| Sidebar | 200 |
| Sidebar overlay (mobile) | 199 |
| Topbar | 100 |
| DA table sticky headers | 5 |

---

## Key Rules Before Editing

1. **Do not split the file.** Everything is intentionally in one `index.html`. Do not create separate CSS or JS files unless explicitly asked.

2. **No build step.** Changes take effect immediately — just save and reload.

3. **`window.onload`** calls `cogsSetLast30()` and auto-loads GIS if `bimma_goog_cid` is set. If you add new init code, hook into `window.onload` or `bimma:pagechange`.

4. **Text-4 is near-black, not near-white.** `--text-4: #22222e` — it's used for invisible/placeholder text on dark backgrounds, not for readable secondary text. Use `--text-3` for readable dim text.

5. **`body::before` is the noise overlay at z-index 9999.** Don't put UI elements above it without reason.

6. **Responsive breakpoint at 768px.** Sidebar collapses (`--sidebar-w: 0px`), grids restack. Test layout changes at both widths.

7. **Monospace everywhere for data.** All numbers, labels, ASIN codes, badges, and button text use `var(--mono)`. Prose and nav items use `var(--sans)`.

8. **Green = positive/active, Amber = warning/ULN, Red = negative/reject, Blue = EU/Keepa links, Purple = optional/info.** These are semantic — don't swap them.

9. **localStorage keys use `bimma_` prefix (underscore) or `bimma-` prefix (hyphen).** The A2A keys happen to use hyphens (`bimma-a2a-*`); all others use underscores. Don't mix them up when adding new keys.

10. **Page-specific JS functions are prefixed** by feature: `cogs*`, `scout*`, `stalker*`, `da*` (A2A), `rm*` (Refund Monitor), `exp*` (Expenses). Follow this convention.
