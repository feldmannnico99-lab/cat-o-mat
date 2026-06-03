# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Single-file static web app — a cat breed personality quiz styled after the German Wahl-O-Mat. No build step, no package manager, no framework. Deployed on GitHub Pages via the `feldmannnico99-lab/cat-o-mat` repo.

## Running

```bash
open index.html
# or for fetch() / CORS:
python3 -m http.server 8080
```

## Git

Remote uses a per-repo SSH deploy key via the `github-cat-o-mat` host alias (`~/.ssh/config`). Push normally with `git push`.

---

## Architecture

Everything is inline in `index.html` — HTML, CSS, and JS. One external JS dependency: GSAP 3 (CDN, for BubbleMenu animation). One external font: Outfit (Google Fonts). One external API: countapi.xyz (visitor counter, fire-and-forget).

### Screens

Six screens share a single `showScreen(id)` toggle (`display: none` / `display: block` + `.active` class):

| ID | Purpose |
|----|---------|
| `screenStart` | Landing / quiz entry point |
| `screenQuestion` | One question at a time, dot navigation |
| `screenResults` | Match % ranking for all 60 breeds |
| `screenGallery` | "Find my Buddy" — animated 3-col photo grid |
| `screenImpressum` | Legal / Impressum |
| `screenDatenschutz` | Privacy policy + 7-day visitor counter |

### Quiz data flow

1. **`QUESTIONS`** — 20 items `{ label, text }`, sourced from the Excel cat-o-mat.xlsx. Index = question position.
2. **`BREEDS`** — 60 items `{ name, desc, note, ideal[] }`. `ideal` is a 20-element array where each value is `0` (stimme nicht zu), `1` (neutral), or `2` (stimme zu), mapping 1:1 to `QUESTIONS` by index.
3. **Global state**: `let currentQ = 0, answers = {}` — module-level vars. `answers` maps `{ [questionIndex]: 0|1|2|null }`. `null` = skipped (treated as `1` in scoring).
4. **`computeMatches()`** — for each breed: `pct = (1 - totalDist / (n*2)) * 100`, sorted descending.
5. Auto-advance 300 ms after answer selection. Last question → `showResults()`.
6. **`restartQuiz()`** — resets `currentQ = 0; answers = {}` and calls `showScreen('screenStart')`. Used by all Back/Zurück buttons and the menu's "Startseite" item.

### Gallery (ContainerScroll animation)

`screenGallery` ports the `ContainerScroll` / `GalleryContainer` pattern from 21st.dev to vanilla JS:

- **`.gal-outer`** — tall container (height set by JS after DOM render); drives scroll progress `p ∈ [0,1]`.
- **`.gal-sticky`** — `position: sticky; top: 0; height: 100vh; overflow: hidden` — the visible viewport.
- **`.gal-grid`** — 3-col CSS grid inside sticky. Animated via `rotateX(75→0deg) scale(0.85→1.0)` over `p: 0→0.45`.
- **Three `.gal-col`** elements — each column translates up via `translateY` with ratios `[1.0, 1.08, 1.0]` over `p: 0.3→1.0`, ensuring all 60 images fully scroll into view.
- On first user scroll: `runGalleryAnimation()` takes over for 2.6 s (non-interruptible via `wheel` + `touchmove` `preventDefault`).
- Outer container `minHeight` is calculated so every column is fully revealed at `p=1`: `(colH - winH) / (0.7 * ratio)` per column.
- Clicking a grid item calls `openBreedModal({ name, desc, note, img })` — a fullscreen fixed overlay. Closes via backdrop click or Escape. `closeBreedModal()` removes `.visible` then hides after 200 ms CSS transition.
- **`GALLERY_ITEMS`** (line ~891) is a dead-code constant — defined but never called. `buildGalleryGrid()` iterates `BREEDS` directly. Do not rely on `GALLERY_ITEMS`.
- The gallery header text reads "Alle 58 Rassen" but `BREEDS` contains 60 entries — stale copy, not a bug in logic.

### BubbleMenu

Fixed-position burger button (top-right, two lines, no circle). Opens a full-screen overlay with 5 pill items in a **CSS grid** (2 columns; 5th item `grid-column: 1/-1; max-width: 50%; margin: 0 auto`). GSAP `back.out(1.5)` stagger on open, fast scale-out on close. Non-passive `wheel` listener while opening to prevent scroll bleed-through.

### Assets

`assets/` holds **61 `.webp`** files — one unique B&W Higgsfield-generated illustration per breed plus `cat-start.webp` (generic). All rendered with `filter: invert(1) grayscale(1)` to guarantee true B&W on dark background regardless of image color profile.

`BREED_IMG` is a flat `{ breedName: 'assets/filename.webp' }` lookup; `breedImg(name)` falls back to `cat-start.webp`.

### Visitor counter

Fires `trackVisit()` on every page load (once per `sessionStorage` key per day). Calls `https://api.countapi.xyz/hit/catoamt-visits-v2/{YYYY-MM-DD}`. Stores the returned `value` in `todayCount` to avoid a race condition when `loadAndRender()` later reads today's bar. The counter widget is rendered inside `screenDatenschutz` via a `MutationObserver` on `.active`.

---

## Design constraints

- **Pure black/white** — `#000` background, `#fff` text/borders. No color anywhere. CSS variable `--r: 16px` for card border-radius.
- **Font**: Outfit (Google Fonts), weights 400–900.
- **Cards**: `border: 1.5px solid white; border-radius: 16px`. Speech-bubble tail = `.card-tail` (CSS triangle, absolute-positioned bottom-left).
- **Dot nav**: `.dot.done` (filled white), `.dot.active` (outlined circle), `.dot.upcoming` (dim).
- **Cat paw cursor** (desktop only, `@media (pointer: fine)`): SVG data-URL open paw; `html.paw-grab` class swaps to grabbing paw on `mousedown`.
- **All images**: `filter: invert(1) grayscale(1)` — mandatory for B&W consistency.
- **Important breed notes** (e.g., Bengalkatze genehmigungspflichtig, Perser brachycephaly): stored in `breed.note`, shown as a dashed `hinweis-box` in results and gallery modal.
