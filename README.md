# PDFlipbook

Turn any PDF into an interactive page-flip book. One script, no build step,
no dependencies to install — [pdf.js](https://mozilla.github.io/pdf.js/) (v4)
is loaded automatically from a CDN the first time a book is created.

- Animated page turns with realistic shading and a 3D perspective
- Grab a page near its edge and **drag** to flip — release past halfway
  (or flick) to complete the turn, otherwise the page falls back
- Grabbing near a **top or bottom corner** folds the paper over along a
  real moving crease — the page stays locked at the spine, the folded
  piece shows the next page's content, and the lifted area reveals the
  page beneath. Mid-edge grabs and the arrows stay a straight 3D flip
- **Tap/click** near an edge to turn a page outright
- Previous / next **arrow buttons** and ← → **keyboard** support
- Responsive — re-lays-out live as its container resizes or the device
  rotates, re-rendering pages crisply at the new size (Hi-DPI aware)
- **Single-page mode** for portrait screens: picked automatically when one
  page can be drawn much larger than half a spread (override with the
  toggle button or `displayMode`). Each page is displayed entirely on its
  own, and every turn is a page peel over the left edge — forward peels
  the current page away (its underside a faint thin-paper ghost) to reveal
  the next beneath; backward sweeps the previous page in on top
- **Fullscreen** button (native Fullscreen API, with a fill-the-viewport
  fallback on iPhone where the API isn't available); in fullscreen the book
  gets extra breathing room and, by default, a soft **drop shadow** so it
  reads as floating — the shadow tracks the visible footprint (one page at
  the covers or in single-page view, the full spread otherwise) and can be
  enabled in normal view too via the `shadow` option
- **Zoom** in steps via the +/− buttons or pinch (continuous), with
  drag-to-pan while zoomed; pages re-render sharper at higher zoom
- Lazy rendering: only pages near the open spread are kept in memory,
  so large PDFs stay light
- Respects `prefers-reduced-motion`

## Quick start

Give a `div` a height, point it at your PDF, include the script:

```html
<div data-pdflipbook="brochure.pdf" style="height:600px"></div>
<script src="pdflipbook.js"></script>
```

That's it. Optional data attributes: `data-start-page="3"`, `data-arrows="false"`,
`data-page-numbers="false"`, `data-display-mode="single"`,
`data-controls="false"`, `data-shadow="always"`.

> The page must be served over http(s) — opening the HTML file directly
> from disk (`file://`) blocks the PDF fetch in most browsers. For local
> testing run `npx serve` or `python3 -m http.server` in the folder.

## JS API

```js
const book = PDFlipbook.create(document.querySelector('#book'), {
  url: 'brochure.pdf',   // or data: Uint8Array (e.g. from a file input)
  startPage: 1,
  duration: 520,         // ms for a full page turn
  edgeSize: 0.14,        // grab-zone width as a fraction of the spread
  cornerFold: true,      // corner drags fold the paper along a crease (false = always straight flip)
  displayMode: 'auto',   // 'auto' | 'double' | 'single'
  controls: true,        // fullscreen / page-view / zoom buttons
  zoomSteps: [1, 1.5, 2, 3],
  shadow: 'fullscreen',  // drop shadow under the book: 'none' | 'normal' |
                         // 'fullscreen' | 'always' (true/false also accepted)
  fullscreenPadding: null, // px around the book in fullscreen (null = ~7% of screen)
  arrows: true,
  pageNumbers: true,
  maxScale: 2,           // devicePixelRatio cap for rendering
  padding: 16,           // breathing room inside the container (px)
  pdfjsSrc: null,        // override the pdf.js module URL (self-hosting)
  pdfWorkerSrc: null     // override the pdf.js worker URL (self-hosting)
});

book.next();
book.prev();
book.goTo(5);
book.currentPage();        // page in view (first of the spread in double mode)
book.zoomIn();             // step through zoomSteps
book.zoomOut();
book.setZoom(2);
book.setDisplayMode('single');   // 'single' | 'double' | 'auto'
book.toggleFullscreen();
book.destroy();
```

Loading a user-supplied file:

```js
input.addEventListener('change', async (e) => {
  const buf = await e.target.files[0].arrayBuffer();
  PDFlipbook.create(el, { data: new Uint8Array(buf) });
});
```

## Events

Dispatched on the container element (they bubble):

| Event                 | `detail`             | When                          |
|-----------------------|----------------------|-------------------------------|
| `flipbook:ready`      | `{ pages }`          | PDF loaded, book built        |
| `flipbook:pagechange` | `{ page, pages }`    | A flip or page slide completes |
| `flipbook:modechange` | `{ mode }`           | Single/double view changes    |
| `flipbook:zoomchange` | `{ zoom }`           | Zoom level settles            |
| `flipbook:error`      | `{ error }`          | The PDF could not be loaded   |

```js
el.addEventListener('flipbook:pagechange', (e) => {
  console.log('now on page', e.detail.page);
});
```

## Theming

Set CSS variables on the container:

```css
#book {
  --fb-bg: transparent;                 /* behind the book        */
  --fb-paper: #fff;                     /* blank page colour      */
  --fb-control-bg: rgba(20,20,24,.72);  /* arrows & counter pill  */
  --fb-control-fg: #fff;                /* arrow icon colour      */
  --fb-counter-fg: rgba(255,255,255,.85);
  --fb-shadow: 0 26px 64px rgba(0,0,0,.5), 0 10px 22px rgba(0,0,0,.35);
  --fb-fold-shadow: drop-shadow(0 10px 12px rgba(0,0,0,.3)); /* lifted-page silhouette */
}
```

## Self-hosting pdf.js

pdf.js 4.x ships as an ES module and is loaded with a dynamic `import()`.
For offline use, CSP restrictions, or to avoid trusting a third-party CDN
(the recommended posture for production), self-host the two files and point
the book at them:

```js
PDFlipbook.create(el, {
  url: 'brochure.pdf',
  pdfjsSrc:      '/vendor/pdf.min.mjs',
  pdfWorkerSrc:  '/vendor/pdf.worker.min.mjs'
});
```

(`npm i pdfjs-dist@^4.10` and copy `pdf.min.mjs` + `pdf.worker.min.mjs`
from `build/`.) These two URLs apply to whichever book is created first;
later books reuse the already-loaded module.

Alternatively, pre-load pdf.js yourself and assign it to `window.pdfjsLib`
before any book is created — pdflipbook.js will detect and reuse it:

```html
<script type="module">
  import * as pdfjsLib from '/vendor/pdf.min.mjs';
  pdfjsLib.GlobalWorkerOptions.workerSrc = '/vendor/pdf.worker.min.mjs';
  window.pdfjsLib = pdfjsLib;
</script>
<script src="pdflipbook.js"></script>
```

## Security

- **Keep pdf.js current.** Versions **before 4.2.67** are vulnerable to
  [CVE-2024-4367](https://github.com/advisories/GHSA-wgrm-67xf-hhpq)
  (arbitrary JavaScript execution from a crafted PDF). pdflipbook.js pins a
  4.x build and additionally passes `isEvalSupported: false` to pdf.js as
  defense-in-depth. Do not downgrade the pinned version.
- **Untrusted PDFs / supply chain.** If you render PDFs you don't control,
  prefer self-hosting pdf.js (above) so a CDN compromise can't inject code,
  and serve under a Content-Security-Policy. Self-hosted assets can be
  pinned with [Subresource Integrity](https://developer.mozilla.org/docs/Web/Security/Subresource_Integrity).
- A bad PDF (or a failed load) surfaces as a `flipbook:error` event and an
  inline message rather than throwing.

## Notes & limitations

- Keyboard: ← → turn pages, + / − zoom, F toggles fullscreen.
- A manual view toggle overrides automatic single/double selection until
  changed again; pass `displayMode: 'auto'` to `setDisplayMode` to restore.
- Pages of mixed sizes are each fitted and centred within the page box.
- Cross-origin PDF URLs need CORS headers, as with any `fetch`.

## Demo

A live demo runs at **https://symplenz.github.io/PDFlipbook/**.

`index.html` shows the component with a bundled sample booklet
(`sample.pdf`) and lets you drag-and-drop your own PDF onto the page.

`pdflipbook-demo-standalone.html` is the same demo as a single self-contained
file with the library and a sample PDF inlined — open it and it just works
(it still fetches pdf.js from the CDN, so it needs a network connection).

Released under the [MIT License](LICENSE).
