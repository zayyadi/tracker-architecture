# Accessibility & Low-Connectivity Test Report

Accurate as of: 2026-09-11
Subject: `promise_tracker_website` (public website)
Build analysed: `dist/` production bundle (built 2026-09-08) plus current source

## Executive summary

| Area | Verdict |
|---|---|
| **Accessibility** | **Partial** — strong foundations in the newest flows (pledge form, chatbot), systemic gaps in the older pages |
| **Low-connectivity readiness** | **Not ready** — the homepage can transfer roughly 3–5.5 MB uncompressed, dominated by one 2.7 MB image and ~1 MB of fonts |

The single highest-impact fix is compressing `VidCv.png` (2.7 MB, used as a CSS background). The second is converting six `.otf`/`.ttf` fonts (~988 KB) to subsetted WOFF2. Together these remove most of the low-connectivity problem without touching application logic.

## Scope and method

**What was tested (static, evidence-based):**

- Source audit of all 11 pages and key components: images, forms, buttons, headings, landmarks, ARIA usage, keyboard handlers, focus styles
- Built-bundle analysis: file inventory, byte sizes, font formats, code-splitting and lazy-loading presence
- First-load payload reconstruction for the homepage

**What was NOT tested (protocols provided below):**

- Full WCAG 2.1/2.2 audit
- Keyboard walkthrough and screen-reader testing (NVDA/VoiceOver)
- Colour-contrast measurement
- Real network throttling in a browser
- Production HTTP headers (compression, caching, HTTP version) — the local sandbox cannot bind a port, and the production host is unknown; verification commands are provided

## Accessibility findings

| ID | Finding | Severity | Evidence | Recommendation |
|---|---|---|---|---|
| A1 | **37 clickable `<div onClick>` vs 21 real `<button>`** — interactive elements are not keyboard-focusable and lack button semantics | High | `rg '<div[^>]*onClick' src` count = 37; `<button>` count = 21 | Convert every interactive div to `<button>` (or add `role`, `tabIndex`, and key handlers as a stopgap) |
| A2 | **8 of 11 pages have no `<h1>`** — assistive-tech users cannot identify the page topic | High | Only `PledgeToVote`, `PledgeThankYou`, `PrivacyPolicy` contain an `<h1>` | Add exactly one descriptive `<h1>` per page |
| A3 | **9 of 11 pages lack a `<main>` landmark** | Medium | Only the two pledge pages contain `<main>` | Wrap each page's primary content in `<main>`; add `<nav>` where appropriate |
| A4 | **Skip-link styles exist but the element is missing** | Medium | `.pledge-skip-link` is styled in `PledgeToVote.css` (lines 477, 545, 551) but no JSX element uses it | Add the skip-link element; the CSS is already written |
| A5 | **Heading level skip on the privacy page** (`h1` → `h3` before the first `h2`) | Low | `PrivacyPolicy.jsx:26` (h1), `:59` (h3), `:123` (h2) | Demote the early `h3` to `h2` or reorder |
| A6 | **Most imagery is CSS backgrounds / inline SVG with no text alternatives** — needs human verification | Medium | Only 3 `<img>` tags exist site-wide; hero and section visuals are backgrounds or SVG components | Audit each decorative vs informative visual; give informative ones text alternatives |
| A7 | Empty `og:image` / `twitter:image` metadata | Low | `index.html` | Fill in or remove |

### What is already working well

- All 3 `<img>` tags have `alt`; the decorative one correctly uses `alt="" aria-hidden="true"` (`PledgeCountCard.jsx:7`)
- The pledge form uses real `<label>`s, `aria-invalid`, `aria-describedby`, and `role="alert"` error messages (`PledgeToVote.jsx:120–227`)
- The chatbot has `role="dialog"`, `aria-label`, focus management, Enter-to-send, and `:focus-visible` styles (`ChatbotWidget.jsx:92–117`)
- `lang="en"`, viewport, and title are present in `index.html`
- Loading states exist on data-fetching pages (partial mitigation for slow APIs)

## Low-connectivity findings

### Payload inventory (measured)

| Asset | Size | Notes |
|---|---|---|
| `VidCv-dPmGwinN.png` | **2,797,827 B (2.7 MB)** | CSS background on the homepage (`VideoCon.css:126`) |
| `index-A79I9q1l.js` | 823,068 B (804 KB) | **Single bundle** — no route-level code splitting |
| `index-0CfGwslj.css` | 211,110 B (206 KB) | Single stylesheet |
| Fonts (6 files, `.otf`/`.ttf`) | **1,011,872 B (988 KB)** | Helvetica ×2, Inter, Barlow, Bodoni ×2 — no WOFF2, no subsetting |
| Homepage-imported images (9) | ~754 KB | Tinubu, school background, people, partners, positions |
| `blog-pic` (largest other image) | 820,747 B | Not homepage-critical |
| Google Analytics (gtag) | External, unmeasured | Loaded on every page |
| **dist total** | **8.1 MB** | |

### Findings

| ID | Finding | Severity | Evidence | Recommendation |
|---|---|---|---|---|
| L1 | **2.7 MB PNG used as a CSS background** | Critical | `VideoCon.css:126` → `VidCv.png` | Recompress to WebP/AVIF, target < 200 KB — saves ~2.5 MB |
| L2 | **Single 804 KB JS bundle; no route-level code splitting** | High | No `React.lazy` or dynamic `import()` found in `src` | Lazy-load per route (React Router makes this mechanical) |
| L3 | **988 KB of fonts in `.otf`/`.ttf`; ~26 duplicated `@font-face` blocks** | High | 6 font files; `@font-face` declared in 26 CSS files | Convert to subsetted WOFF2, declare once, drop unused families |
| L4 | 206 KB single CSS file | Medium | `index-0CfGwslj.css` | Purge unused rules; consider per-route CSS |
| L5 | **No `loading="lazy"` on any image** | Medium | No matches for `loading="lazy"` in `src` | Add lazy loading + `decoding="async"` to below-fold images |
| L6 | Google Analytics loads on every page over third-party DNS/TLS | Medium | `index.html` gtag snippet | Acceptable if compressed/cached; consider loading after interaction on slow connections |
| L7 | No offline/service-worker support; content pages depend on live API | Medium | No service worker; tracker pages fetch on mount | Acceptable for v1; loading states already exist — keep and standardise them |

### First-load estimate (homepage, uncompressed)

| Component | Estimate |
|---|---|
| HTML + JS + CSS | ~1.0 MB |
| Fonts actually used (2–4 faces) | ~0.3–0.9 MB |
| Images rendered on load (incl. `VidCv`) | ~3.5 MB |
| **Realistic total** | **~3–5.5 MB** |

| Connection profile | Approximate transfer time |
|---|---|
| Slow 3G (400 kbps ≈ 50 KB/s) | **60–110 seconds** |
| Fast 3G (1.6 Mbps ≈ 200 KB/s) | 15–28 seconds |

Production compression (gzip/brotli) would typically shrink the JS/CSS by 60–75%, but images and fonts — the dominant weight here — barely compress. Verify on production (below).

## Production verification checklist

Run against the real host and record results:

```bash
curl -sI https://promisetracker.ng/ | rg -i 'cache-control|content-encoding|age:'
curl -sI https://promisetracker.ng/assets/index-A79I9q1l.js | rg -i 'cache-control|content-encoding|age:'
curl -sI https://promisetracker.ng/assets/VidCv-dPmGwinN.png | rg -i 'cache-control|content-encoding|age:'
```

| Check | Expected |
|---|---|
| `content-encoding: br` or `gzip` on HTML/JS/CSS | Yes |
| `cache-control: public, max-age=31536000, immutable` on hashed assets | Yes |
| `cache-control: no-cache` (or short) on `index.html` | Yes |
| HTTP/2 or HTTP/3 | Yes |

## Manual test protocol (pending human verification)

### Keyboard
1. From the homepage, tab through nav, hero, and footer — every control reachable, visible focus, logical order
2. Complete the pledge form using only the keyboard, including the radio group and error states
3. Open, use, and close the chatbot with the keyboard only
4. Confirm the 37 clickable divs that remain are operable (this is expected to fail until A1 is fixed)

### Screen reader (NVDA on Windows or VoiceOver on macOS)
1. Navigate each page by landmarks and headings — every page should announce `<main>` and one `<h1>`
2. Submit the pledge form with errors and confirm the messages are announced (`role="alert"` is in place)
3. Use the chatbot and confirm the dialog is announced and replies are readable

### Contrast and throttling
1. DevTools → Lighthouse accessibility run; record contrast failures
2. DevTools → Network → "Slow 3G" with cache disabled; record load time and any failed requests
3. Repeat with "Fast 3G" and "Offline" after first load

## Prioritised recommendations

| # | Action | Impact | Effort |
|---|---|---|---|
| 1 | Compress `VidCv.png` to WebP/AVIF (< 200 KB) | Removes ~2.5 MB | Small |
| 2 | Convert fonts to subsetted WOFF2; declare each family once | Saves ~600–700 KB | Small |
| 3 | Add route-level code splitting (`React.lazy`) | Cuts initial JS substantially | Medium |
| 4 | Add `loading="lazy"` + `decoding="async"` to below-fold images | Defers ~700 KB | Small |
| 5 | Replace clickable divs with buttons (A1) | Unblocks keyboard users | Medium |
| 6 | Add `<h1>` and `<main>` to every page (A2, A3) | AT navigation | Small |
| 7 | Add the skip-link element (A4) | CSS already exists | Trivial |
| 8 | Verify production compression/caching headers | Confirms savings land | Small |
| 9 | Run the manual protocol above and record results | Closes this report's open items | Half a day |
