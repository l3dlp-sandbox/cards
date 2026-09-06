# Performance audit

This document records a mobile Lighthouse audit of [notes.jordanscales.com](https://notes.jordanscales.com) performed on September 6, 2026. It is a dated baseline rather than a permanent performance guarantee: the site content, third-party embeds, browser, network, and Lighthouse scoring model can all change.

## Method

- Lighthouse 13.0.1 with Chrome 152.
- Mobile simulation and cold page loads.
- Three runs per page unless noted otherwise.
- Scores and timings below are medians; ranges are called out when variance was material.
- Transfer sizes include page resources observed by Lighthouse, including third-party embeds.

## Baseline

| Page | Performance | FCP | LCP | Transfer / requests | Notes |
| --- | ---: | ---: | ---: | ---: | --- |
| [Home](https://notes.jordanscales.com/) | 100 | 1.25 s | 1.57 s | 199 KiB / 34 | TBT 0 ms and CLS 0. The page requested 29 unique emoji PNGs. |
| [Rogue agents](https://notes.jordanscales.com/rogue) | 100 | 1.21 s | 1.36 s | 27 KiB / 8 | Small, static post with no material bottleneck. |
| [Estimates](https://notes.jordanscales.com/estimates) | 100 | 1.21 s | 1.36 s | 27 KiB / 8 | Small, static post with no material bottleneck. |
| [Game of Life Tower](https://notes.jordanscales.com/gol-tower) | 100 median | — | — | 260 KiB / 26 | One run scored 76 after 1,211 ms of TBT in the Shader/Stack framework bundle. |
| [Reactivity](https://notes.jordanscales.com/reactivity) | 99 | 1.67 s | 1.67 s | 84 KiB / 19 | KaTeX CSS requested fonts below `/fonts/`, but deployment flattened them into the site root. |
| [French House](https://notes.jordanscales.com/french-house) | 97 median | — | — | 1.38 MiB / 93–96 | Scores ranged from 57 to 98. Five eager YouTube iframes dominated payload and variance. |
| [When code is data](https://notes.jordanscales.com/when-code-is-data) | 100 | — | — | 518 KiB / 10 | One run. A 493,254-byte BMP accounted for nearly the entire payload; Lighthouse estimated 467 KiB of image savings. |
| [Large numbers](https://notes.jordanscales.com/large-numbers) | 97 | — | — | — | 10,473 DOM nodes, 394 KiB of uncompressed HTML, and 679 ms of main-thread work. |
| [What comes next](https://notes.jordanscales.com/what-comes-next) | 96 | — | — | 121 KiB / 20 | No single large payload; the score reflects cumulative rendering and scripting cost. |

## Findings and status

### 1. Replace eager YouTube embeds with a facade

**Status: pending.** French House is the clearest remaining network bottleneck. Loading five complete YouTube players on initial navigation produces roughly 1.38 MiB across more than 90 requests and causes highly variable scores.

Render a lightweight poster and play button first, then create the YouTube iframe after interaction. Preserve the video title for accessible labeling and provide a normal YouTube link as a fallback.

### 2. Serve KaTeX fonts at the paths used by KaTeX CSS

**Status: complete.** Commit `c0c347d` preserves the `fonts/` directory when copying KaTeX assets and removes the production and local-server path-flattening workarounds. The deployed URL `/fonts/KaTeX_Main-Regular.woff2` now responds successfully.

### 3. Reduce the front-page emoji payload

**Status: complete.** The home page previously loaded 29 unique 64-pixel Apple emoji PNGs totaling about 181 KiB of source data. A local experiment reduced the set to about 53 KiB as same-size AVIF, or about 37 KiB as two-times-display-size AVIF.

Commit `fc05f79` converts emoji assets to WebP during the build and emits the correct favicon MIME type. This keeps the existing Apple emoji presentation while reducing transfer size without adding requests.

### 4. Convert BMP images to browser-friendly formats

**Status: complete.** A local conversion of the 493,254-byte image on When code is data produced approximately 14 KiB AVIF, 21 KiB WebP, and 41 KiB JPEG versions.

Commit `b7cbac2` adds 24- and 32-bit BMP decoding to the image pipeline. BMP sources now receive intrinsic dimensions and responsive AVIF/WebP variants; the fallback image also uses WebP instead of the original BMP. The deployed post currently advertises both AVIF and WebP sources.

### 5. Reduce large DOM output

**Status: pending.** Large numbers renders more than 10,000 DOM nodes and nearly 400 KiB of HTML. Consider representing repeated visual data with canvas, SVG, CSS backgrounds, or a progressively enhanced compact data model instead of emitting every element into the initial document.

Any rewrite should preserve readable fallback content and avoid moving a large amount of equivalent work into blocking JavaScript.

### 6. Investigate Shader/Stack startup variance

**Status: pending.** Game of Life Tower was fast in most runs but occasionally spent more than a second in total blocking time inside the embedded Shader/Stack framework bundle.

Profile the iframe independently, reduce its initial JavaScript, and defer compilation or rendering until the embed approaches the viewport. Continue honoring `prefers-reduced-motion` and pause rendering while the document is hidden.

## Implementation checks

The three completed optimizations were kept as individual commits and deployed together. Before deployment:

- 130 focused renderer, download, and snapshot tests passed.
- Lint and TypeScript checks passed.
- The full test run passed 134 tests; four local server tests could not bind `127.0.0.1` in the sandbox and timed out after `listen EPERM`.

After deployment:

- The home page referenced WebP emoji assets.
- A representative WebP emoji returned HTTP 200.
- `/fonts/KaTeX_Main-Regular.woff2` returned HTTP 200.
- When code is data emitted responsive AVIF and WebP sources and used an optimized WebP fallback.

## Re-running the audit

Use at least three cold mobile Lighthouse runs per page and report the median. Keep the browser and Lighthouse versions with the results, and record score ranges for pages with third-party or interactive content. Re-test the pages above after changes to the renderer, image pipeline, embed behavior, or deployment headers.

For renderer changes, run:

```sh
npm run lint
npm run typecheck
npm test
```

Use `npm run build:compare-fresh` when a refactor should not change generated pages. Use `npm run build:cached-status` when intentionally inspecting deployment-output changes.
