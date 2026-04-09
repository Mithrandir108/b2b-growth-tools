# b2b-growth-tools

A collection of lightweight, dependency-minimal browser tools for B2B revenue and operations contexts. Built as single-file HTML applications — no build step, no framework, no backend required.

---

## Tools

### 1. ROI Calculator (`tools/roi-calculator.html`)

**What it does:** Multi-step wizard that models a fractional engagement's revenue impact. Takes pipeline volume, current conversion rate, and sales cycle as inputs; outputs ROI multiple, Year 1 gain, and payback period. Renders a 12-month revenue trajectory comparing baseline vs. uplifted scenario.

**Technical choices:**
- **Chart.js 4.4** via CDN — chosen for its small footprint and native canvas rendering. No React or D3 overhead for a single chart.
- **Multi-step form state** managed in vanilla JS — no state library needed; three `<div>` visibility toggles are sufficient.
- **Trajectory model** is a simple compound growth function with a configurable ramp-up period (`rampUpMonths`). Not a financial model — deliberately simplified to produce directional estimates, not projections.
- **CONFIG block at top of file** externalises all business logic parameters (investment amount, uplift rate, growth assumptions, CTA URL, currency symbol). Swap values without touching UI code.

**Dependencies:** Chart.js 4.4.0 (CDN)

---

### 2. COO Gap Visualizer (`tools/coo-gap-visualizer.html`)

**What it does:** Interactive radar chart across five operational dimensions (Processes, Tech Stack, Scalability, Human Capital, Governance & Data). Users adjust sliders; chart updates in real time. Exports a PDF snapshot via html2canvas + jsPDF.

**Technical choices:**
- **Chart.js radar type** — standard polar grid is the right structure for multi-axis comparison. The benchmark dataset (dashed line at value 9) is a static overlay, not computed from user input.
- **Dynamic slider generation** — sliders are built from the `CONFIG.dimensions` array, making the tool axis-agnostic. Change labels or add axes by editing the config only.
- **PDF export via html2canvas + jsPDF** — captures the DOM as a canvas, converts to PNG, embeds in an A4 landscape PDF. Simpler than SVG serialisation for this use case; trade-off is pixel-based output (not vector).
- **`hexToRgba` utility** — converts hex accent colour to rgba for Chart.js `backgroundColor` (which requires rgba, not hex).
- **CONFIG block** externalises dimensions, benchmark value, accent colour, PDF filename, and footer URL.

**Dependencies:** Chart.js (CDN), html2canvas 1.4.1 (CDN), jsPDF 2.5.1 (CDN)

---

### 3. RevOps Roadmap Simulator (`tools/revops-roadmap-simulator.html`)

**What it does:** Template-driven roadmap builder. Users select from three engagement templates (Quick Win / Core Build / Full Stack), toggle individual building blocks across six categories, and see a real-time Q1–Q3+ deployment timeline. Exports a styled PDF roadmap; optionally captures email via a configurable webhook before export.

**Technical choices:**
- **Data-driven UI** — both the template buttons and the category/chip grid are generated from `CONFIG.CATEGORIES` and `CONFIG.TEMPLATES` arrays, not hardcoded HTML. Adding a new block or category requires only a CONFIG edit.
- **State management** — `selectedBlocks` is a `Set` of block IDs. Toggle logic is O(1) add/delete. Timeline re-renders on every state change (lightweight enough to skip debouncing).
- **Timeline rendering** — blocks are bucketed into three quarter columns by their `quarter` property (1, 2, or ≥3). No date arithmetic — intentionally abstract for a planning tool.
- **PDF generation (jsPDF)** — rendered programmatically (not DOM capture), giving precise control over layout, typography, and colour. Each block is drawn as a `roundedRect` with the category's RGB colour.
- **Webhook pattern** — `CONFIG.webhookUrl` is checked before opening the modal. If empty, the tool skips the email gate and downloads the PDF directly. This makes it usable as a pure local tool or as a lead-capture tool depending on deployment context.
- **CONFIG block** externalises titles, templates, categories, quarters, PDF filename, footer text, and webhook URL.

**Dependencies:** jsPDF 2.5.1 (CDN)

---

## Architecture pattern

All three tools follow the same pattern:

```
CONFIG block (top of <script>)
    ↓
DOMContentLoaded → inject CONFIG into DOM elements
    ↓
Event listeners → update state → re-render
    ↓
Export function (PDF or chart) → reads from current state
```

The CONFIG block is intentionally the only place that needs to be edited for deployment customisation. UI logic and CONFIG are separated by convention — CONFIG comes first, logic follows.

---

## Config generator (Python)

All three tools can be configured from a single YAML file using the included Python script. This avoids editing HTML directly when deploying to a new context.

```bash
# Install dependency
pip install -r scripts/requirements.txt

# Inject config into all tools
python scripts/generate_config.py

# Single tool
python scripts/generate_config.py --tool roi-calculator

# Preview without writing files
python scripts/generate_config.py --dry-run

# Validate config structure only
python scripts/generate_config.py --validate
```

Edit `scripts/configs/default.yaml` — the script serialises the relevant section as a `const CONFIG = {...}` block and injects it into the target HTML file, replacing the existing CONFIG delimiter.

The YAML config is the single source of truth for all business logic parameters. HTML files contain only UI and rendering code.

---

## Customisation

Each tool has a `CONFIG` object at the top of the `<script>` tag. Fields are documented inline with comments. Common changes:

| What to change | Where |
|---|---|
| Currency symbol | `CONFIG.currency` (ROI Calculator) |
| Investment amount assumption | `CONFIG.investment` (ROI Calculator) |
| Radar dimensions/labels | `CONFIG.dimensions` (COO Gap) |
| PDF filename | `CONFIG.pdfFilename` (COO Gap, Roadmap) |
| Roadmap blocks or categories | `CONFIG.CATEGORIES` (Roadmap) |
| Webhook endpoint for lead capture | `CONFIG.webhookUrl` (Roadmap — leave `""` to skip) |
| CTA button URL | `CONFIG.ctaUrl` (ROI Calculator) |

---

## Running locally

No build step. Open any file directly in a browser:

```bash
open tools/roi-calculator.html
open tools/coo-gap-visualizer.html
open tools/revops-roadmap-simulator.html
```

Or serve with any static server:

```bash
npx serve .
# or
python3 -m http.server 8080
```

---

## Design system

Tools use a consistent design token set:

| Token | Value | Usage |
|---|---|---|
| `--bg` | `#242322` | Page background |
| `--yellow` | `#fdda24` | Primary accent, CTAs, key metrics |
| `--white` | `#ffffff` | Text on dark backgrounds |
| `--gray` | `#9ca3af` | Secondary text |
| `--card` | `rgba(255,255,255,0.04)` | Card backgrounds |

Typography: Inter (UI), Lora (headings) — both loaded from Google Fonts.

---

## Deployment

Any static host works. Tested on Netlify (drag-and-drop or CLI):

```bash
netlify deploy --dir . --prod
```

No environment variables required unless using the Roadmap Simulator webhook (set `CONFIG.webhookUrl` to your function endpoint).

---

## Dependencies summary

| Library | Version | Used in | Why |
|---|---|---|---|
| Chart.js | 4.4.0 | ROI Calculator, COO Gap | Canvas-based charting |
| html2canvas | 1.4.1 | COO Gap | DOM-to-canvas for PDF capture |
| jsPDF | 2.5.1 | COO Gap, Roadmap Simulator | PDF generation |

All loaded via CDN (jsDelivr / cdnjs). No npm install required.
