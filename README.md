# PangoSvelte

**Synthetic manuscript page generator for OCR / HTR training data** — Pango-quality typography, configurable multi-frame layouts, and matching ground truth (ALTO + PAGE-XML).

> **Status: work in progress.** This repository currently contains only this README. The full source tree will be published when the project is feature-complete.

**GitHub:** [johnlockejrr/PangoSvelte](https://github.com/johnlockejrr/PangoSvelte)

---

## What it is

PangoSvelte combines:

1. **A Python render pipeline** built on [Pango](https://docs.gtk.org/Pango/) + Cairo — real font shaping, RTL/LTR, justify, and world-script coverage via fontconfig.
2. **A SvelteKit web editor** for designing page templates, previewing pages, and saving layouts as JSON.
3. **Batch CLI** for unattended generation of image + ground-truth pairs from a template, text corpus, and background pool.

Output is training data: **PNG page images** plus **PAGE-XML** (and optionally ALTO) with line polygons, baselines, and Unicode text.

### How it differs from browser-based synth tools

PangoSvelte uses **Pango** as the single layout engine — the same stack used for serious multilingual typesetting. Text flows through **independent frames** (columns, headers, footers) defined in JSON templates. This is **not** CSS multi-column flow and **not** a Chromium screenshot pipeline.

For a complementary Chromium/CSS approach (grapheme-aware DOM extraction, CSS column flow), see the separate **[SvelteSynth](https://github.com/johnlockejrr/SvelteSynth)** project.

### Heritage

PangoSvelte extends **[PangoLine](https://github.com/mittagessen/pangoline)** (Benjamin Kiessling et al.) with a SvelteKit editor, PAGE-XML export, batch generation, corpus preview, baseline/bbox overlays, extended template `render` settings, and deployment tooling.

---

## Features (planned release)

### Layout & typography

- **Multi-frame JSON templates** — single column, 2/3 columns, header + columns, bilingual parallel columns, Talmud-style presets
- **Sequential frame flow** — each frame fills independently; overflow moves to the next frame (not row-synchronized multi-column)
- **RTL support** — Hebrew, Arabic, Syriac, etc.; frame order reversed for RTL languages
- **Per-frame alignment** — `left`, `center`, `right`, `justify`
- **Typography controls** — line spacing, baseline placement within GT line boxes, bbox padding
- **Pango Markup** and optional random word styling (CLI)
- **Custom fonts** — drop `.ttf` / `.otf` in `fonts/` at repo root; fontconfig + repo `fonts.conf`

### Web editor (`web/`)

- Form-based template editor (not drag-and-resize WYSIWYG)
- Live preview via Pango render → rasterize (same path as CLI)
- Preset templates + save/load user templates to `templates/`
- **Sample text** or **corpus upload** with prev/next page pagination
- Overlay toggles: **baselines**, **line boxes**, **random background**
- Font family dropdown from fontconfig (`pangoline fonts_cli`)
- CLI command snippets mirroring current settings

### CLI

```bash
# Single document
pangoline render --template templates/two_columns.json corpus/doc.txt
pangoline rasterize --gt-format pagexml doc.txt.0.xml

# Batch (template + corpus folder + backgrounds)
pangoline batch \
  --template templates/two_columns.json \
  --texts ./corpus \
  --backgrounds ./backgrounds \
  --count 500 \
  --dpi 300 \
  --out-dir outputs \
  --gt-format pagexml
```

Outputs: `outputs/images/page_NNNN.png` + `outputs/pagexml/page_NNNN.xml`

### Ground truth

- **ALTO XML** (native PangoLine format, mm coordinates)
- **PAGE-XML** (pixel coordinates after rasterization; compatible with eScriptorium / PaddleOCR-style pipelines)
- Baseline and bounding-box controls for GT annotation without moving rendered ink

---

## Tech stack

| Layer | Stack |
|-------|--------|
| Render engine | Python 3.9+, PyGObject, Pango, Cairo, pypdfium2 |
| Ground truth | lxml, custom ALTO + PAGE-XML serializers |
| Web UI | SvelteKit 2, Svelte 5, TypeScript, Vite |
| Web server | `@sveltejs/adapter-node` |
| Packaging | hatchling (`pyproject.toml`), npm, release tarball scripts |

---

## Repository layout (when published)

```
PangoSvelte/
├── pangoline/              Python package (render, rasterize, batch, layout, preview_cli)
│   └── templates/          Built-in layout presets (*.json)
├── web/                    SvelteKit editor
│   └── src/routes/         Editor UI + /api/preview, /api/templates, /api/fonts
├── scripts/                make-release.sh, install-target.sh, start-web.sh
├── fonts/                  User-supplied font files
├── backgrounds/            Paper/texture images for compositing
├── templates/              User-saved layout JSON
├── outputs/                Generated images + PAGE-XML
├── corpus/                 UTF-8 .txt files for batch
├── tests/
├── INSTALL.md              System deps (Pango, gi, fontconfig)
├── WEB.md                  Web editor guide
├── DEPLOY.md               Release bundle deployment
├── TEMPLATES.md            Template format reference
└── pyproject.toml
```

---

## Template format (summary)

Templates are JSON with **frames** (required) and an optional **`render`** block for defaults saved from the editor.

```json
{
  "frames": [
    { "x": 20, "y": 20, "width": 80, "height": 257, "alignment": "justify" },
    { "x": 110, "y": 20, "width": 80, "height": 257, "alignment": "justify" }
  ],
  "render": {
    "paper_size": [210, 297],
    "margins": [25, 30, 25, 25],
    "font_family": "Ezra SIL SR",
    "font_weight": "Normal",
    "font_size_pt": 12,
    "language": "he-IL",
    "dpi": 150,
    "line_spacing": 2,
    "baseline_factor": 0.8,
    "baseline_position": 1.0,
    "padding_all": 1.0
  }
}
```

Legacy **frame-only** templates remain valid. CLI flags override individual `render` fields.

---

## Prerequisites (overview)

**System (Linux recommended):**

- Pango + Cairo + GObject Introspection (`python3-gi` on Debian/Ubuntu — do not pip-install PyGObject on top of distro `gi` without care)
- fontconfig
- Node.js 20+ (web editor only)

**Quick Debian/Ubuntu system packages:**

```bash
sudo apt install \
  python3-gi python3-gi-cairo \
  gir1.2-gtk-3.0 gir1.2-pango-1.0 \
  libcairo2-dev pkg-config python3-dev \
  fontconfig
```

Full install instructions will ship in `INSTALL.md`.

---

## Quick start (when source is available)

```bash
# Python (from repo root)
python3 -m venv .venv --system-site-packages
source .venv/bin/activate
pip install -e ".[dev]" --no-deps

# Web editor
cd web && npm ci && npm run dev -- --host 0.0.0.0
# → http://localhost:5180
```

Production deploy: `./scripts/make-release.sh` → unpack on target → `./install-target.sh` → `./start-web.sh`

---

## Built-in template presets

| File | Layout |
|------|--------|
| `two_columns.json` | Classic two-column |
| `three_columns.json` | Three columns |
| `header_two_columns.json` | Centered header + two columns |
| `bilingual_parallel.json` | Parallel bilingual columns |
| `hebrew_syriac_parallel.json` | Hebrew + Syriac parallel |
| `talmud.json` | Talmud-style multi-region |

---

## Roadmap / before v1.0

- [ ] Publish full source tree to GitHub
- [ ] Finalize documentation (`INSTALL.md`, `WEB.md`, `TEMPLATES.md`)
- [ ] End-to-end batch smoke test on clean Debian install
- [ ] PAGE-XML validation against existing training pipeline fixtures
- [ ] Editor polish pass (error surfacing, empty states)

---

## Limitations

- **No CSS-style continuous column flow** — frames are independent; text does not balance across columns row-by-row.
- **No image degradation** in-tool — backgrounds are static user images; augmentation happens downstream in training.
- **No multi-page text carry** in the editor preview beyond corpus pagination; batch mode paginates sequentially.
- **Pango layout ceiling** — very long single layouts hit Pango's 32-bit baseline offset limit (~3000 pages depending on settings).

---

## License

Apache-2.0 (inherited from PangoLine).

## Authors

- **John Locke Jrr** — PangoSvelte editor, PAGE-XML, batch, deployment ([johnlockejrr@gmail.com](mailto:johnlockejrr@gmail.com))
- **Benjamin Kiessling** and contributors — [PangoLine](https://github.com/mittagessen/pangoline) core

## Funding

PangoLine (upstream) was co-financed by the European Union (ERC, MiDRASH, project number 101071829).

---
