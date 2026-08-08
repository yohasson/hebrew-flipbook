# HANDOFF — rebuilding the הולך בדרכי flipbook from the final edited book

Written 8 Aug 2026, for whoever picks this up next after Yossi finishes the
final text edit of the book. This is everything I learned building and
fixing the current site, so you don't have to rediscover it.

**Read this whole document before touching anything.** The single biggest
time-saver in here is the "stale pagination" section — it explains a bug
that isn't visible from just looking at the site, and will bite you if you
skip straight to editing images.

---

## 1. What this actually is

A single-file HTML flipbook reader for a Hebrew memoir ("הולך בדרכי", Yaakov
Kabla), styled and behaviourally matched against a paid Heyzine flipbook the
family already had made of the same book
(`https://heyzine.com/flip-book/a711a2a3d2.html`). Yossi wanted a free,
self-hosted, ad-free equivalent that looks and feels the same.

- **Live site:** https://yohasson.github.io/hebrew-flipbook/
- **Repo:** `C:\Users\yohasson\.scout\hebrew-flipbook` — pushes to
  `https://github.com/yohasson/hebrew-flipbook.git`, deployed via GitHub
  Pages from `main`.
- **Engine:** [StPageFlip](https://github.com/Nodlik/StPageFlip) (MIT), a
  real 3D-curl page-flip library, **vendored locally** at
  `vendor/page-flip.browser.js` — do not change this back to loading from
  unpkg.com or a CDN (see §5).
- Two editions in one site: `regular` (11.5pt body text) and `large`
  (13pt), switchable live via the title-card toggle, each with its own set
  of page images.

## 2. The build pipeline — what generates what

All build scripts live in
`...\Documents\Microsoft Scout\Scratchpad\HOLECH-BEDARKI\5_BUILD\`.

```
paras.json (750 paragraphs, single source of text)
       |
       v
mkbook.py <variant>   ("std" or "large")   -- Word layout engine
       |                                       (margins/font size per variant,
       v                                        also drives the A5 print PDFs)
book_std.docx / book_large.docx
       |
       v  (Word -> PDF, via LibreOffice/soffice or similar)
book_std.pdf / book_large.pdf
       |
       v  rasterize.py / rasterize_large.py
       |  (these currently point at out/C_RGB_trim.pdf and out/L_RGB_trim.pdf —
       |   see §3, THESE INPUT FILES ARE MISSING)
       v
images/regular/page_NNN.jpg  (68 files)
images/large/page_NNN.jpg    (84 files)
       |
       v
mksite4.py   -- generates the actual index.html from a template + these
                image counts + _sounds.json, writes to
                C:\Users\yohasson\.scout\hebrew-flipbook\index.html
       |
       v
git add/commit/push  (from C:\Users\yohasson\.scout\hebrew-flipbook)
       |
       v
GitHub Pages serves it, ~60-90s after push
```

**To rebuild the site after any image or template change:**
```powershell
cd "...\HOLECH-BEDARKI\5_BUILD\ebook_src"
python mksite4.py
cd C:\Users\yohasson\.scout\hebrew-flipbook
git add -A
git commit -m "..."
git push origin main
```
`mksite4.py` reads image counts from `images/regular` and `images/large`
directly (`os.listdir`), so it auto-adjusts to however many pages actually
exist — you don't need to edit page counts by hand anywhere.

## 3. CRITICAL: the deployed page images are from a STALE, mismatched build

This is the thing that will confuse you if you don't know it going in.

I found, and did not fully resolve, a real inconsistency:

| | Current print PDFs (`5_BUILD/rebuild/book_std.pdf`, `book_large.pdf`) | Deployed flipbook images (`images/regular`, `images/large`) |
|---|---|---|
| Page count | **80pp / 87pp** | **68pp / 84pp** |
| Source | `mkbook.py` output, rebuilt 8 Aug 2026 | Rasterized at some earlier point from `out/C_RGB_trim.pdf` / `out/L_RGB_trim.pdf` — **these two input PDFs no longer exist anywhere on disk** |

`tocmap_std.json` / `tocmap_large.json` (chapter-name → page-number lookup,
in the same `rebuild` folder) correspond to the **current 80pp/87pp**
pagination, **not** the 68pp/84pp images actually on the live site. If you
use those tocmap files against the deployed flipbook you will get chapter
boundaries that are off by several pages (I measured this precisely —
see §7).

**What this means for you:** the flipbook you see live right now is
*already* out of sync with the current book text/layout, before you even
start incorporating Yossi's final edits. You are not starting from a
clean, consistent baseline.

**What I recommend:** don't try to patch the existing 68pp/84pp images.
Once Yossi's final edit lands and goes through `mkbook.py` (however many
pages that produces), run the **full pipeline end to end** — Word → PDF →
rasterize → `mksite4.py` — so the flipbook images and the print PDFs are
generated from the exact same source in the same pass and can never drift
apart again. While you're at it, see §7 for how to get *exact*
paragraph-level continuity for free as a side effect.

## 4. Everything measured and matched against Heyzine tonight

I ran three parallel research passes (geometry, flip-animation/physics,
chrome/UI) directly against the live Heyzine page, then applied fixes.
Exact values, so you don't have to re-measure from scratch if you're
re-doing this after a redesign:

**Background/backdrop** — Heyzine's SVG backdrop is drawn with
`background-size:cover`, so only a narrow middle band of it is ever
visible; reading the SVG's own gradient stops gives a value ~3x too
contrasty. The actual **rendered pixel values** at 1440×900 are corners
TL 247, TR 229, BL 232, BR 223 (out of 255) — a ~143° diagonal ramp. Ours:
`linear-gradient(143deg,#f7f7f7 0%,#efefef 35%,#e6e6e6 70%,#dedede 100%)`,
verified within 2.5 levels of Heyzine's actual rendered pixels.

**Spread sizing** — Heyzine fills ~82.1% of viewport width / 92.7% of
height at 1440×900. Achieved by trimming `#stage` padding to 6px and the
bottom bar to 64px.

**Shadow** — Heyzine layers a second, spread-wide ambient shadow
(`rgb(204,204,204) 0 0 20px 0`, ≈ `rgba(0,0,0,.2)`) on top of the per-page
shadow. We added `#spreadfx` for this. Heyzine has **no page-edge stack**
effect at all — we had one (`.stack` elements), removed it entirely.

**Chrome/panels** — Heyzine's floating panels (title card, tools cluster)
are flat translucent `rgba(255,255,255,.85)`, 5px radius, **no drop
shadow** — not the Material-style opaque-card-with-shadow look we started
with. Font is a generic sans-serif fallback, not worth chasing.

**Scrub thumbnail preview** — Heyzine shows a live two-page image preview
(204×150 box, two 92×130 thumbnails, `rgba(0,0,0,.2)` background, 4px
radius) while dragging the scrubber, only while dragging — not
persistent. We built the same thing (`#scrubprev`).

**Counter format** — Heyzine shows spreads as `"6 - 7"` (space-hyphen-
space); we use an isolated-Unicode `4–5` to prevent bidi reordering. Close
enough, not identical; not worth matching exactly.

**Flip timing** — measured ~760ms on Heyzine; we use `flippingTime: 780`
(StPageFlip only exposes a duration, not a custom easing curve, so this
is the only lever available).

**Edge-of-book arrows** — Heyzine's prev/next buttons sit fully **outside**
the page canvas with a consistent ~10-14px gap (not overlapping it), and
use a specific curved "reply-arrow" glyph (hooked head + sweeping tail),
**not** a plain chevron. I downloaded and cropped their actual sprite
(`https://cdnm.heyzine.com/flipbook/img/iconset2_6.png`) to confirm the
shape. Ours now matches both the gap and the icon (see the `.edgebtn`
CSS and the shared SVG path in `mksite4.py`).

**Corner-peel hover hint** — Heyzine shows a small triangular paper-fold
at page corners on hover (rotated-square-clipped-to-corner, confirmed via
its own `matrix(0.707107, 0.707107, -0.707107, 0.707107, ...)` = 45°
rotation), with the curved arrow icon layered on **only the corner nearest
the direction of travel** — the opposite corner shows the fold with no
arrow. We built four `.peel` divs (`peelNextB/T`, `peelPrevB/T`) to match
this exactly.

## 5. Deliberately NOT matched, and why — don't "fix" these

Three things Heyzine does that we do **not**, on purpose, because ours is
better and matching them would be a regression:

1. **Flip animation is a flat 2D hinge-fold on Heyzine, not a 3D curl.**
   Verified via live DOM inspection: their `<canvas>` elements are static
   bitmaps that never repaint during a flip; the actual animation is
   plain CSS `rotate(deg)` + `translate3d` on wrapper divs — no
   `rotateY`, no perspective anywhere. Ours uses genuine 3D `rotateY()`
   with real perspective (StPageFlip's actual rendering). Their easing is
   also unusually front-loaded (63% of rotation in the first 62ms) —
   StPageFlip doesn't expose a custom timing function, only duration, so
   this can be approximated by shortening duration but not reproduced
   exactly.
2. **No drag-to-peel on Heyzine** (best evidence: n=1 test, low
   confidence, but no intermediate transform ever appeared during a slow
   drag). Ours has real continuous drag-follow via StPageFlip. This is a
   capability *surplus*, not a gap.
3. **Heyzine has no real mobile layout** — at narrow viewports it's just
   a shrunk, side-cropped desktop canvas, not a responsive single-page
   view. Ours properly reflows to single-page with pruned controls. Do
   not "fix" this to match Heyzine; ours is objectively better here.

## 6. The vendoring fix — do not revert this

The site used to load StPageFlip from `https://unpkg.com/page-flip@2.0.7/...`
at runtime. **This silently broke the entire UI** on any network that
blocks that CDN (a corporate policy, an extension, Defender content
filtering — this is exactly what happened to Yossi on his own machine).
When the CDN is blocked, `St.PageFlip` never initializes, the boot
sequence throws partway through `build()`, and everything downstream
(chrome, bottom bar, centering) never runs — you're left with bare,
unstyled page `<div>`s.

Fixed by vendoring the library: `vendor/page-flip.browser.js` in the repo,
loaded via `<script src="vendor/page-flip.browser.js"></script>`. I
verified this works by literally blocking all `unpkg.com` requests in a
test page and confirming the reader still boots correctly.

**If you ever update StPageFlip, re-vendor it the same way** — download
the new version's `dist/js/page-flip.browser.js` into `vendor/`, don't
add back a CDN `<script src>`.

## 7. Edition-switch continuity — what's there now, and how to make it exact

Yossi asked that switching between the "regular" and "large-print"
editions keep the reader on the same paragraph (allowing the page number
to change, since the two editions paginate differently).

**What's implemented now (`setEdition()` in `mksite4.py`):** a
**proportional/global-ratio approximation**. It takes the reader's
fractional position through the book — `(oldPage-1)/(oldTotal-1)` — and
applies the identical fraction to the new edition's page count. This is
provably safe (cover always maps to cover, back cover always maps to back
cover, round-trips are lossless — I tested all three), but it is **only
approximate mid-book**. I validated it against the real chapter-boundary
data in `tocmap_std.json`/`tocmap_large.json` (which — reminder — is for a
*different* pagination than what's deployed, but the *relative* drift
pattern is still informative): the approximation is exact for the first
~6 of 22 chapters, then drifts up to **9 pages** off by the end of an
84-page book. Good enough to land in the right neighbourhood; not
paragraph-exact for later chapters. There's a bug-history worth knowing:
I initially had an off-by-one (used `oldPage` instead of `(oldPage-1)` in
the fraction), which shifted the cover to page 2 on every switch — fixed,
but a reminder to actually test the cover/back-cover edge cases if you
touch this formula, not just the middle of the book.

**How to make it exact, and why you can do this for free:** you'll be
building both editions from the same `paras.json`/paragraph source in one
pipeline run anyway. At the point where you know each edition's final
pagination (right after the Word→PDF step, or by parsing the generated
DOCX/PDF), emit **one JSON array per edition**: `pageOfParagraph[i] =
pageNumber` for each of the ~750 paragraphs. Embed both arrays in the
site (same pattern as `EDITIONS`/`SOUNDS` are embedded now via
`__EDITIONS__`/`__SOUNDS__` template placeholders in `mksite4.py`). Then
in `setEdition()`, instead of the proportional formula: find which
paragraph index is showing on the current page in the *old* edition's
map (first/any paragraph whose page equals the current page), look up
that same paragraph index in the *new* edition's map, and `turnToPage()`
there. This is exact, not approximate, and the infrastructure
(`paras.json`, the `EDITIONS` embedding pattern) already exists — it's a
build-step addition, not a redesign.

## 8. Bugs I found and fixed tonight via testing — test these yourself, don't just eyeball it

Every one of these was invisible from a quick look and only surfaced
under actual automated interaction testing. If you change anything in
this area, re-run equivalent tests, not just a visual check:

- **Corner-peel "crossed corner" bug — REAL root cause, read this before
  touching corner-hover code at all.** Yossi reported: cursor on the
  bottom-**left** page corner, but the fold/peel effect lifted on the
  bottom-**right** corner instead. This took three attempts to actually
  fix:
  1. First theory (wrong, but a real bug worth keeping fixed): the
     original hover implementation used `pointerenter`/`pointerleave` on
     small 34px `.peel` boxes. On fast mouse movement `leave` could fail
     to fire, leaving the *wrong* corner stuck visible. Fixed by
     switching to continuous proximity tracking on every `pointermove`
     — nearest-corner-within-90px, recomputed from scratch each time.
     Stress-tested with a 40-sample continuous sweep, no stuck states —
     but Yossi still saw the bug live after this shipped.
  2. Second theory (also a real bug, also not the actual cause): the
     `pointermove` listener was on `#stage`, but `#tools`/`#titlecard`
     are **siblings** of `#stage` (all direct children of `<body>`), not
     descendants — the event never bubbles through `#stage` when the
     cursor is resting over those panels, so proximity tracking silently
     stopped updating there. Fixed by moving the listener to `document`
     (using `pointerout` + `relatedTarget===null` to detect "left the
     viewport", since `document` doesn't fire `pointerleave` the way a
     contained element does). Still not the actual cause — a screen
     recording from Yossi after this fix still showed it crossed.
  3. **The actual cause:** StPageFlip — the library itself — has its own
     **built-in** corner-hover fold preview (`showCorner()`, gated by a
     config flag `showPageCorners` that **defaults to `true`**). It computes
     "nearest corner" against `#book`'s internal, un-mirrored coordinate
     space. But `#book` is rendered with `transform:scaleX(-1)` (that's
     how the LTR-only engine becomes an RTL book — see the comment above
     `#book` in `mksite4.py`). So StPageFlip's own corner decision is
     made correctly in pre-mirror space, then the whole `#book` gets
     visually flipped — landing its built-in fold on the mirror-image
     corner from what it computed. This was invisible to every prior
     test because all of that testing only inspected *our own* `.peel`
     divs (which were behaving correctly the whole time) — never
     StPageFlip's own internal DOM (`stf__outerShadow` etc.). **Fix:**
     `showPageCorners: false` in the `new St.PageFlip(...)` config,
     since we already have our own custom peel-hint UI and don't need
     StPageFlip's. Verified by checking StPageFlip's internal
     shadow/fold elements stay inert (0×0, no active class) at all four
     corners under slow, realistic mouse approaches, and confirmed fixed
     live by Yossi after deploy.

  **Lesson if you ever touch corner/hover behaviour again:** always
  check whether the library itself has an overlapping built-in feature
  before assuming a bug is in your own code — especially with any
  library wrapped in a CSS mirror/flip trick like this RTL setup. Grep
  the vendored source (`vendor/page-flip.browser.js` /
  `pageflow.js`) for the feature name, don't just trust the docs.
- **Edge arrows off-screen on mobile:** positioning arrows *outside* the
  page assumed 14px gap + 38px button width of margin always exists.
  On a 390px-wide phone viewport in portrait/single-page mode, there
  isn't — `edgeNext` computed to `x=-52`, completely unusable, no way to
  advance the book at all on a phone. Fixed with a clamp
  (`Math.max(4, ...)` / `Math.min(..., viewportWidth-42)`), falling back
  to slightly overlapping the page edge rather than disappearing.
- **Edge arrows off-screen again at zoom > 1:** the arrows live inside
  `#zoomwrap`, so their `left`/`top` are pre-scale-transform local
  coordinates; the viewport clamp above is only valid at zoom 1. At zoom
  1.5 the same (correctly clamped-at-zoom-1) value can still land
  off-screen after magnification. Rather than chase scale-compensated
  position math for a control whose entire purpose is reliable
  clickability, the arrows are now simply hidden whenever `zoom !== 1`
  — the bottom bar's prev/next (which live outside `#zoomwrap` and are
  zoom-immune) carry navigation while zoomed. Also discovered `setZoom()`
  never called `layout()` at all, so this toggle wouldn't have taken
  effect regardless — fixed that too.

**Testing method that found all three:** don't just take one screenshot
and eyeball it. Use Playwright's `mouse.move` with real coordinates
(derived from live `getBoundingClientRect()`, not guessed pixels),
sweep continuously across the full width at each viewport size you claim
to support (1440×900 desktop, 768×1024 tablet-portrait, 390×844 mobile,
and at least one zoomed state), and read back real computed state
(`classList.contains(...)`, `getBoundingClientRect()`) rather than
inferring from a screenshot.

## 9. Sound

Page-turn sounds are **synthesized from scratch** (`mksounds3.py`), not
hotlinked or re-hosted from Heyzine — their three MP3s
(`https://cdnm.heyzine.com/flipbook/snd/flip-ct-{sm,md,lg}.mp3`) are their
licensed CDN assets; redistributing them would be a licensing problem.
Only their *measured acoustic properties* are used as synthesis targets
(duration, spectral centroid, played RMS loudness, crest factor, attack
time — see the docstring in `mksounds3.py` for the full measured table).

**If you ever need to re-verify or re-tune this:** measure Heyzine's
targets at **44.1kHz**, not 32kHz — I found analyzing at 32kHz understates
the spectral centroid by 2.5-5%, which looks like a real mismatch but
isn't. Their files are still at the URL above as of tonight; I
re-downloaded and re-verified all five measured properties match exactly
using `imageio_ffmpeg` for decoding (no system ffmpeg needed) plus numpy.

## 10. How to verify a deployment actually went live

GitHub Pages takes roughly 60-90 seconds after a push. Don't trust the
push succeeding as proof it's live — fetch and hash-compare:

```powershell
git push origin main
Start-Sleep -Seconds 75
Invoke-WebRequest "https://yohasson.github.io/hebrew-flipbook/index.html?cb=$(Get-Random)" -OutFile "$env:TEMP\live.html" -UseBasicParsing
# compare $env:TEMP\live.html against your local index.html (normalize CRLF/LF - git
# checks out CRLF locally but serves LF, which will otherwise look like a false mismatch)
```

I used this pattern all night to confirm every fix actually reached
production, not just committed locally.

## 11. Quick reference — file map

```
...\HOLECH-BEDARKI\                          consolidated project folder
  5_BUILD\
    mkbook.py                current Word-layout engine (both print + ebook source)
    rebuild\
      paras.json             the 750-paragraph source of truth (also used for proofreading)
      tocmap_std.json        chapter->page for the CURRENT 80pp print PDF (NOT the live flipbook)
      tocmap_large.json      chapter->page for the CURRENT 87pp print PDF (NOT the live flipbook)
      book_std.pdf/.docx     current print output, 80pp
      book_large.pdf/.docx   current print output, 87pp
    ebook_src\
      mksite4.py             generates index.html - THE FILE TO EDIT for site behaviour
      mksounds3.py            page-turn sound synthesis
      rasterize.py            regular-edition PDF -> JPEGs (INPUT PDF MISSING, see §3)
      rasterize_large.py      large-edition PDF -> JPEGs (INPUT PDF MISSING, see §3)
      pageflip.js              a saved copy of the vendored library (source of vendor/ in the site repo)

C:\Users\yohasson\.scout\hebrew-flipbook\      the deployed site's git repo
  index.html                 generated by mksite4.py - do not hand-edit, re-run the generator
  vendor\page-flip.browser.js  vendored StPageFlip - see §6, do not revert to CDN
  images\regular\, images\large\   the STALE 68pp/84pp page images, see §3
  _sounds.json                generated by mksounds3.py
  README.md                   user-facing repo description
```

---

## 12. Final verification status (8 Aug 2026)

Before handing this off, every UI bug reported by Yossi across this whole
session was re-tested live and confirmed fixed on the deployed site:

- ✅ Corner-peel crossed-side bug — fixed (§8, `showPageCorners: false`),
  confirmed live by Yossi.
- ✅ Edge-of-book prev/next arrows — correct RTL side, correct icon,
  positioned outside the page, zoom/mobile-safe.
- ✅ Bottom bar/scrubber missing in windowed (non-fullscreen) mode — Yossi
  reported this once via a real desktop Edge window; extensive Playwright
  testing at 500-1200px+ heights never reproduced it, and one real-OS
  `PrintWindow` capture did briefly reproduce a blank bar before the page
  had fully painted. Yossi re-tested live afterward and confirmed **"the
  bottom bar looks good"** — treating this as resolved, most likely a
  one-off load-timing or cache artifact rather than a persistent layout
  bug. If it resurfaces: the bottom bar (`#bottom`) is pure CSS
  `position:fixed;bottom:0`, nothing in the JS ever hides or toggles it
  dynamically — so a recurrence points at something painting over it
  (z-index/stacking) or a slow/incomplete initial paint, not missing
  logic. Worth a hard-refresh + a few seconds' wait before concluding
  it's broken again.
- ✅ Edition-switch (regular ↔ large font) keeps the reader on
  approximately the same paragraph when switching — see §7 for the exact
  (non-approximate) upgrade path if you want to do it properly during
  the rebuild.

Nothing is a known-open UI bug as of this handoff. The only known,
deliberate gap is the stale-pagination mismatch in §3, which is a
content/pipeline issue, not a UI bug — it will resolve itself once you
run the full pipeline against Yossi's final edited text.

---

If something here turns out to be wrong or incomplete once you're deeper
into the rebuild, please update this file rather than letting it rot —
the next person after you will thank you.
