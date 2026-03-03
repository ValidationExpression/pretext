## Text Metrics

DOM-free text measurement library using canvas `measureText()` + `Intl.Segmenter`.

Two-phase API: `prepare()` once per text, `layout()` is pure arithmetic on resize. Full i18n: CJK, Thai, Arabic, Hebrew, bidi, emoji. ~0.1ms to reflow 500 comments.

### Commands

- `bun run serve` — serve demo pages at http://localhost:3000
- `bun run check` — typecheck + lint
- `bun test` — headless tests (HarfBuzz, 1472-point i18n sweep, 100% accuracy)

### Architecture

- `src/layout.ts` — the library. prepare() segments + measures words via canvas. layout() walks cached widths.
- `src/measure-harfbuzz.ts` — HarfBuzz measurement backend for headless tests (not used in browser).
- `src/test-data.ts` — shared test texts/params for browser and headless tests.
- `src/layout.test.ts` — bun test suite: consistency + accuracy (word-sum vs full-line).
- `pages/` — browser demo pages: accuracy sweep, benchmark, interleaving demo, visual demo.

### Key decisions

- Canvas measureText over DOM: avoids read/write interleaving entirely. No DOM reads ever.
- Intl.Segmenter over split(' '): handles CJK (no spaces between words), Thai, etc. Replaces Sebastian's linebreak npm dependency.
- Punctuation merged with preceding word before measuring: "better." measured as one unit, not "better" + ".". Reduces accumulation error (up to 2.6px at 28px without merging).
- Trailing whitespace hangs past line edge (CSS behavior): spaces that overflow maxWidth don't trigger line breaks.
- Word-level cache keyed on (segment, font): survives across resize since font doesn't change. Common words shared across texts.
- HarfBuzz with explicit LTR direction for headless tests: guessSegmentProperties() mis-assigns RTL to isolated Arabic words, changing their advance widths. LTR gives consistent results matching browser canvas.

### Known limitations

- Emoji: canvas measures 4px wider than DOM at font sizes <24px on macOS (Apple Color Emoji pipeline difference). Converges at >=24px. Untested on Windows.
- system-ui font: canvas and DOM resolve to different optical variants at certain sizes on macOS. Use named fonts.
- Server-side: needs canvas (browser) or @napi-rs/canvas (with registered fonts). HarfBuzz works for headless testing but metrics differ from browser.

### Things we tried and rejected

- Full-line measureText in layout phase: pixel-perfect but 250-1000x slower due to string concatenation. Actually less accurate in practice.
- DOM-based measurement in prepare(): accurate but reintroduces the interleaving problem.
- SVG getComputedTextLength(): still a DOM read, no auto-wrap.
- @napi-rs/canvas server-side: wrong CJK/emoji metrics without explicit font registration.

### Based on

Fork of Sebastian Markbage's [text-layout](https://github.com/reactjs/text-layout) research prototype. We added: two-phase caching, Intl.Segmenter, punctuation merging, CJK grapheme splitting, overflow-wrap support, trailing whitespace handling.
