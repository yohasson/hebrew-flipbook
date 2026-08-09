# HANDOFF — rebuilding the הולך בדרכי flipbook from the final edited book

Written 8 Aug 2026, updated 9 Aug 2026 after the later flipbook rebuilds
(integrated cover/interior image set, clickable TOC, clean mode, paper-tone
picker, mobile reset, social preview, and show-through fixes), for whoever
picks this up next. This is everything I learned building and fixing the
current site, so you don't have to rediscover it.

**Read this whole document before touching anything.** The single biggest
time-saver in here is the pagination/source-state section — it explains
which artifacts currently match the live flipbook and which are only print
pipeline outputs, which will bite you if you skip straight to editing images.

---

## 1. What this actually is

A single-file HTML flipbook reader for a Hebrew memoir ("הולך בדרכי", Yaakov
Kabla), styled and behaviourally matched against a paid Heyzine flipbook the
family already had made of the same book
(`https://heyzine.com/flip-book/a711a2a3d2.html`). Yossi wanted a free,
self-hosted, ad-free equivalent that looks and feels the same.

- **Live site:** https://yohasson.github.io/kabala/
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
`C:\Users\yohasson\OneDrive\Documents\HOLECH-BEDARKI\5_BUILD\`.

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
images/regular/page_NNN.jpg  (90 files as of 9 Aug 2026)
images/large/page_NNN.jpg    (99 files as of 9 Aug 2026)
       |
       v
mksite4.py   -- generates the actual index.html from a template + these
                image counts + _sounds.json + tocmap_std/tocmap_large,
                writes to C:\Users\yohasson\.scout\hebrew-flipbook\index.html
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

## 3. Current pagination/source state — do not assume the PDFs and image folders are identical

This is the thing that will confuse you if you don't know it going in.

The old 68pp/84pp stale-image state is no longer current. After the later
rebuilds, the deployed image folders now contain:

| | Current print PDFs (`5_BUILD/rebuild/book_std.pdf`, `book_large.pdf`) | Deployed flipbook images (`images/regular`, `images/large`) |
|---|---|---|
| Page count | **96pp / 106pp** | **90pp / 99pp** |
| Source | `mkbook.py` output, current print-oriented PDFs in `rebuild\` | Current deployed flipbook image set in the GitHub Pages repo |

The current TOC maps in `rebuild\tocmap_std.json` and
`rebuild\tocmap_large.json` are **actively embedded by `mksite4.py`** for
the clickable TOC feature. As of the 9 Aug handoff, they contain 22 entries
each and max at page **89** (regular) / **98** (large), which lines up with
the 90/99 deployed image counts as "last chapter before the final cover/end
matter". The TOC UI currently displays **printed page numbers** using a
hardcoded `printedPage(page) = page - 12` offset while internally jumping to
the actual generated image page.

**Do not generalize those numbers.** The next PDF used for this flipbook may
be very different in page count (regular and large may differ from each
other, and both may differ wildly from today's 90/99 images). On any rebuild:
delete all old `page_*.jpg` files in the target image folder first, rasterize
from the actual new integrated PDF, let `mksite4.py` recount the folders, and
regenerate `tocmap_std.json` / `tocmap_large.json` from the actual new page
sequence. If the number of front-matter/cover/blank pages before printed page
1 changes, also update or remove the `printedPage(page) = page - 12` display
offset in `mksite4.py`; it is a property of the current integrated PDF, not a
universal rule.

**What this means for you:** the flipbook you see live right now is not an arbitrary old 68/84 build any
more, but you still must not assume `book_std.pdf` / `book_large.pdf` can be
blindly rasterized over the current site without checking page order. The
flipbook now includes cover/interior book-behaviour assumptions (front cover,
inside-cover blank, interior, inside-back-cover blank, back cover). If you
replace the source PDF, build the integrated PDF/page-image sequence
intentionally and regenerate the TOC maps in the same pass — see §15.

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
height at 1440×900. We originally got close with a very tight 6px stage
padding, but that later felt cramped against the browser edge. Current
state: desktop `#stage` uses `padding:26px 6px`, and `build()` now reads
the actual computed stage padding rather than a hardcoded `-12px` guess,
so future padding tweaks don't desync page sizing. Mobile still has its
own tighter padding via the media query.

**Shadow / page opacity** — Heyzine layers a second, spread-wide ambient
shadow (`rgb(204,204,204) 0 0 20px 0`, ≈ `rgba(0,0,0,.2)`) on top of the
per-page shadow. We added `#spreadfx` for this. Heyzine has **no page-edge
stack** effect at all — we had one (`.stack` elements), removed it entirely.
Later show-through bugs during flips led to an intentionally more solid
runtime setup: `drawShadow:false`, `maxShadowOpacity:0`, and additional
opaque/blank reverse-side handling in `index.html`/`mksite4.py`. Do not
re-enable transparent flip shadows casually; it can reintroduce Hebrew text
show-through during page turns.

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

**Flip timing** — measured roughly in the 760-1000ms range depending on
whether you're looking at hover/snap/settle behaviour; current StPageFlip
duration remains `flippingTime: 780`. StPageFlip only exposes a duration,
not Heyzine/turn.js's full custom easing curve, so this is the only safe
public lever available.

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
rotation). Current site uses StPageFlip's native `showPageCorners:true`
fold preview after patching the RTL coordinate bug; our old `.peel` divs
remain in the DOM as invisible click targets only (`opacity:0!important`),
not visible fold graphics.

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
2. **Heyzine and ours use different flip engines/physics.** Heyzine's
   reference uses turn.js-style 2D wrapper transforms; ours uses StPageFlip
   with real 3D page geometry. Both support drag-follow behaviour, but the
   exact physics/easing are not identical. Keep ours unless Yossi explicitly
   asks to trade the 3D curl for Heyzine's flatter turn.js motion.
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
cover, round-trips are lossless), but it is **only approximate mid-book**.
This approximation intentionally makes no assumption about the actual page
count; if a future PDF is 40 pages or 200 pages, the formula still maps to
the same relative position. There's a bug-history worth knowing: an earlier
off-by-one used `oldPage` instead of `(oldPage-1)` in the fraction, which
shifted the cover to page 2 on every switch — fixed, but a reminder to
actually test the cover/back-cover edge cases if you touch this formula,
not just the middle of the book.

**Clickable TOC (current state):** `mksite4.py` embeds
`rebuild\tocmap_std.json` and `rebuild\tocmap_large.json` as a `TOC` object.
The TOC button lives in the top-right title card next to the font-size
buttons, because the active font-size edition determines which TOC map is
used. Clicking an entry calls `flip.turnToPage(page-1)` and closes the
panel. The panel hides in clean mode, has a slim custom scrollbar, supports
`T` to toggle and `Esc` to close, and uses separate page targets for regular
and large editions. Important: the TOC maps must be regenerated from the
actual new PDFs on every rebuild; do not reuse today's page numbers if the
replacement PDFs have a different page count or different front matter.

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
     StPageFlip's own internal DOM (`stf__outerShadow` etc.). **Fix
     applied at the time:** `showPageCorners: false` in the
     `new St.PageFlip(...)` config, since we already had our own custom
     peel-hint UI and didn't (at the time) know the *real* root cause yet.
     Verified by checking StPageFlip's internal shadow/fold elements stay
     inert (0×0, no active class) at all four corners under slow,
     realistic mouse approaches, and confirmed fixed live by Yossi after
     deploy.

     **This was NOT the end of the story — see step 4 below and §12.**
     `showPageCorners: false` only hid the *symptom* (StPageFlip's own
     animated hover fold), it didn't fix the actual bug, which also
     mirrored the real drag-to-flip direction, not just the passive hover.
     Yossi later reported the real drag direction was ALSO crossed
     ("i try to lift the left page but actually the right page is being
     lifted") — that surfaced the true root cause and the real fix.
  4. **The TRUE root cause (found later, after re-opening this "fixed"
     bug):** StPageFlip's `getMousePos()` (in
     `vendor/page-flip.browser.js`) converts real `clientX/clientY` into
     local page coordinates with `x: t - i.left` — no awareness of the
     `scaleX(-1)` mirror at all. This single function feeds **every**
     real pointer interaction: the passive hover preview AND the actual
     drag/flip-direction decision. Patched in place: `x: r.width +
     2*r.left - (t - i.left)`. Confirmed via explicit A/B testing
     (temporarily reverting the patch reproduced the exact reported bug —
     dragging the bottom-left corner lifted the far-right page — then
     re-applying fixed it) across `flipNext()/flipPrev()` (unaffected,
     they use synthetic coordinates that bypass this function), real
     mouse drag-to-flip from either page, and keyboard arrows. With the
     coordinates now correct at the source, `showPageCorners` was safely
     switched back to `true` — StPageFlip's own animated hover-lift now
     works correctly. This immediately surfaced one more small issue: our
     own `.peel` div (kept around from step 1-3 as a hover substitute) now
     rendered *at the same time* as StPageFlip's real fold, clashing
     visually. Fixed by forcing `.peel{opacity:0!important}` — see §12
     for the full final-state writeup.

  **Lesson if you ever touch corner/hover behaviour again:** always
  check whether the library itself has an overlapping built-in feature
  before assuming a bug is in your own code — especially with any
  library wrapped in a CSS mirror/flip trick like this RTL setup. Grep
  the vendored source (`vendor/page-flip.browser.js` /
  `pageflow.js`) for the feature name, don't just trust the docs. And once
  you find the *real* shared root cause of what looked like separate
  bugs, go back and clean up any earlier workarounds rather than leaving
  both the workaround and the real fix active at once — that's exactly
  what caused the `.peel`-vs-native-fold clash here.
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
Invoke-WebRequest "https://yohasson.github.io/kabala/index.html?cb=$(Get-Random)" -OutFile "$env:TEMP\live.html" -UseBasicParsing
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
      tocmap_std.json        embedded by mksite4.py for regular clickable TOC (22 entries, max page 89)
      tocmap_large.json      embedded by mksite4.py for large clickable TOC (22 entries, max page 98)
      book_std.pdf/.docx     current print output, 96pp
      book_large.pdf/.docx   current print output, 106pp
    ebook_src\
      mksite4.py             generates index.html - THE FILE TO EDIT for site behaviour
      mksounds3.py            page-turn sound synthesis
      rasterize.py            regular-edition PDF -> JPEGs (hardcoded paths may still need cleanup, see §15)
      rasterize_large.py      large-edition PDF -> JPEGs (hardcoded paths may still need cleanup, see §15)
      pageflip.js              a saved copy of the vendored library (source of vendor/ in the site repo)

C:\Users\yohasson\.scout\hebrew-flipbook\      the deployed site's git repo
  index.html                 generated by mksite4.py - do not hand-edit, re-run the generator
  vendor\page-flip.browser.js  vendored StPageFlip - see §6, do not revert to CDN
  images\regular\, images\large\   deployed page images: 90pp/99pp as of 9 Aug 2026, see §3
  _sounds.json                generated by mksounds3.py
  README.md                   user-facing repo description
  social-cover-front-optionB.jpg  current social preview image
```

---

## 12. Final verification status (8 Aug 2026, updated after a second round of fixes)

Before handing this off, every UI bug reported by Yossi across this whole
session was re-tested live and confirmed fixed on the deployed site.
**Note:** the corner-peel/page-turn direction bug went through *three*
distinct fixes across the session, not one — §8 documents the full history,
but the short version if you only read this section:

- ✅ **Corner-peel/page-turn direction — the REAL final fix.** The
  `showPageCorners: false` workaround mentioned earlier in this doc (and in
  an earlier commit message) was **superseded** — it only papered over
  StPageFlip's own animated corner-hover, it never touched the actual bug,
  which also mirrored the real **drag-to-flip direction** (not just the
  passive hover hint). The real root cause: StPageFlip's own
  `getMousePos()` (in `vendor/page-flip.browser.js`) converts real
  `clientX/clientY` into local page coordinates with a naive
  `x - rect.left`, which has no idea `#book` is rendered with
  `transform:scaleX(-1)` for RTL. That one function feeds **both** the
  passive hover preview AND the real drag/flip-direction logic, so both
  were mirrored wrong. Patched in place (search for the comment above
  `getMousePos` in `vendor/page-flip.browser.js` and in the reference copy
  `ebook_src/pageflip.js`) to un-mirror real pointer coordinates:
  `x: r.width + 2*r.left - (t - i.left)` instead of `x: t - i.left`.
  Verified this is the correct fix, not another false lead, by explicit
  A/B testing (temporarily reverting it and confirming the bug reproduces
  exactly as reported, then re-applying and confirming it's gone) across
  four different interaction paths: `flipNext()/flipPrev()` (button clicks,
  unaffected either way since they build synthetic coordinates that never
  go through `getMousePos`), real mouse drag-to-flip from either page,
  and keyboard arrows. `showPageCorners` was then safely **re-enabled**
  (`true`) since the coordinate bug it was hiding is now actually fixed —
  StPageFlip's own animated hover-lift now works correctly and looks like
  a real page-corner lift.
- ✅ **Corner-hover visual clash (found immediately after re-enabling
  `showPageCorners`).** With StPageFlip's own animated fold now correctly
  enabled, our own custom `.peel` div (a small static 34px triangle+arrow
  icon, built earlier in the session as a *substitute* for the
  corner-hover hint before the real bug was understood) started rendering
  **at the same time and same spot**, visually clashing with StPageFlip's
  larger, dynamically-sized native fold — a small icon awkwardly sitting
  under/inside the real animated lift. Fixed by forcing `.peel{opacity:0
  !important}` — the div (and its onclick-to-flip handler) still exists in
  the DOM but never renders, so StPageFlip's own animation is now the
  single, clean hint. If you ever want the custom arrow icon back, it
  would need to be re-integrated *into* StPageFlip's own fold render rather
  than as an independent overlay — non-trivial, probably not worth it now
  that the native animation alone looks right. Note that the original
  StPageFlip hover zone was about ~200px for this book size, but the current
  vendored library has since been patched smaller (`isPointOnCorners` divisor
  12, `showCorner` offset 24, capped shadow width), so don't re-measure from
  this old historical number.
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
- ✅ Fullscreen button — works correctly in a real standalone browser
  window. It only appeared broken once, inside Microsoft Scout's own
  embedded preview pane, which silently blocks the Fullscreen API from
  taking visual effect even though the JS call succeeds without error —
  not a site bug, nothing to fix.
- ✅ Edition-switch (regular ↔ large font) keeps the reader on
  approximately the same paragraph when switching — see §7 for the exact
  (non-approximate) upgrade path if you want to do it properly during
  the rebuild.
- ✅ Clickable TOC — embedded from `tocmap_std.json` / `tocmap_large.json`,
  opened from the title-card controls next to the font-size switch, with
  edition-specific page targets and a custom slim scrollbar.
- ✅ Clean reading mode, paper-tone picker, mobile reset button, social
  preview image, scrub preview ordering/labels, and opaque/blank reverse
  side handling for page turns are all part of the current `mksite4.py` /
  `index.html` state and should be preserved unless intentionally replaced.

**A meta-lesson from this whole debugging arc, worth internalizing before
you touch this code again:** the same underlying bug (StPageFlip's
un-mirrored real pointer coordinates in RTL mode) manifested as what
looked like three *separate* problems reported at different times — a
crossed hover corner, then (after a partial fix) a crossed real
drag-flip direction, then (after fixing that for real) a cosmetic
double-render clash from an old workaround that was never cleaned up.
Each looked like a different bug from the outside. When you're deep in
an RTL-mirrored library integration like this one, always ask "is this
the same root cause wearing a different hat?" before assuming a fresh
investigation is needed from scratch — and clean up workarounds
(`showPageCorners: false`, the `.peel` substitute icon) once the real
underlying fix makes them unnecessary, rather than leaving both the
old workaround and the new real fix active at once.

Nothing is a known-open UI bug as of this handoff. The main caution is not
a UI bug: future replacement PDFs may have very different page counts and
front-matter structure, so every rebuild must regenerate image folders and
TOC maps from the actual new integrated PDFs instead of assuming today's
90/99 images, 96/106 print PDFs, or printed-page offset.

## 13. Zoom-in drag now pans, not flips (fixed later the same day)

Reported bug: with zoom above 100%, dragging over the enlarged page flipped
to the next/previous page instead of panning around the zoomed-in view —
not how any normal zoomed image/PDF viewer behaves.

**Root cause:** the pan-drag handler already existed (`#stage.zoomed` sets
`cursor:grab`, and a `pointerdown`/`pointermove`/`pointerup` triple on
`#stage` tracks `scrollLeft`/`scrollTop`), but it had
`if (zoom <= 1 || e.target.closest('#book')) return;` — and a drag almost
always *starts* on the visible page content, i.e. inside `#book`, so that
exclusion meant panning essentially never engaged in real use. Worse, even
without that check, StPageFlip attaches its own `mousedown`/`touchstart`
listeners **directly on `#book`**, so our `#stage`-level handler could never
reliably win an event-order race against the library's own page-turn drag
logic anyway.

**Fix:** `#stage.zoomed #book{pointer-events:none}`. While zoomed, the
browser's own hit-testing routes every `pointerdown` straight to
`#zoomwrap`/`#stage` — never to `#book` — so StPageFlip's drag listeners
simply never see the pointer events at all. No event-order tricks, no
`stopPropagation`, no library patching needed. Removed the now-obsolete
`e.target.closest('#book')` exclusion from the pan handler since it's no
longer reachable/needed.

**Verified via Playwright:** a drag over the book at 125% zoom changes
`stage.scrollLeft`/`scrollTop` and leaves the page counter unchanged; the
identical drag gesture at 100% zoom still triggers a normal page flip — no
regression to standard drag-to-flip.

## 14. Paper-tone picker (white / cream / aged sepia)

New feature, not a bug fix: lets the reader pick a warmer paper colour —
better suited to a family memoir than stark white as the only option.

**Why this needed an overlay, not a background-colour change:** every page
is a flat scanned/rasterized JPEG (`images/<edition>/page_NNN.jpg`), not
live-rendered text — there's no page "background" showing through to
recolour. So the tint has to sit visually **on top of** the existing image.

**Implementation (`mksite4.py`):**
- `.page::after` — a `pointer-events:none` overlay per page, gated by
  `mix-blend-mode:multiply` so a warm tint darkens the page toward
  cream/sepia while leaving the scan's own black ink strokes essentially
  untouched (multiply on black ≈ still black; multiply on white ≈ the tint
  colour). Two variants defined, `cream` and `aged`, each a soft radial
  vignette + flat tint colour; no attribute at all (or explicitly `white`)
  renders nothing.
- Gated by a `data-paper` **attribute on `<html>`**, deliberately *not* a
  class on `#book` — `build()` calls `el.innerHTML=''` on `#book` every
  edition switch / zoom-triggered resize / reload, which would wipe a class
  living there. `<html>` persists across all of that, so every freshly
  re-created `.page::after` just picks the current tone back up for free,
  no extra JS needed per rebuild.
- `#paperbtn` in the tools cluster cycles white → cream → aged on click.
  Reuses the exact same "swap which of several identical SVGs is visible"
  pattern as the dark-mode sun/moon toggle (`paintThemeIcon`) — three
  identically-shaped little page-swatch icons (`#paper-white`,
  `#paper-cream`, `#paper-aged`), only one shown at a time, so the icon
  itself previews the current tone instead of needing a text label. Note:
  the swatch fill colours are set via **ID selectors** (`#paper-white{fill:
  #fff}` etc.) specifically because they need to beat the generic
  `.ico svg{fill:none}` rule on specificity — a class-based override
  wouldn't have been enough without `!important`.
- Persisted via `localStorage['hb_paper']`, independent of the light/dark
  **chrome** theme (`toggleTheme()`/`.dark`) — paper tone is a property of
  the paper, not the app UI, so an aged-paper + dark-chrome combination (or
  any combination) works with no conflict.

**If you want to add a fourth tone (or change the two existing ones):** add
another entry to the `PAPERS` array in the JS, another `<svg id="paper-X">`
in the button markup, another `#paper-X{fill:...}` rule, and another
`html[data-paper="X"] .page::after{...}` CSS block. That's the entire
surface area — no other file needs touching.

## 15. How to replace the source PDF / rebuild from a new manuscript

This comes up whenever Yossi has a newly edited manuscript and wants the
flipbook to reflect it. **The flipbook itself never touches a PDF** — it
only ever consumes pre-rasterized JPEGs (`images/<edition>/page_NNN.jpg`).
Replacing "the PDF" really means re-running the rasterize step against a
new source PDF, for **both** editions, then regenerating `index.html`.

**Assume the replacement PDFs may have a very different number of pages.**
Do not write scripts or checks that expect today's 90/99 image counts, the
current 96/106 print PDF counts, or any small delta from either. Treat page
count as fully data-driven from the new integrated PDFs and the freshly
rasterized folders.

**Important — verified 8 Aug 2026, don't trust `rasterize.py`'s hardcoded
paths as-is, they're stale in three separate ways:**
- Both `rasterize.py` and `rasterize_large.py` hardcode
  `SRC = "...OneDrive - Microsoft\Documents\Microsoft Scout\Scratchpad\
  HOLECH-BEDARKI\5_BUILD\rebuild\out\C_RGB_trim.pdf"` (or `L_RGB_trim.pdf`)
  — that `OneDrive - Microsoft` root **no longer exists at all** (same
  library-rename already noted for `mksite4.py`'s own path elsewhere in
  this doc); the current root is plain `OneDrive` (no `- Microsoft`).
- Even under the current root, there is **no `rebuild\out\` subfolder** and
  **no `C_RGB_trim.pdf`/`L_RGB_trim.pdf`** — those were evidently an
  intermediate CMYK→RGB-trimmed export step from an earlier point in the
  project that no longer exists in the current pipeline.
- **The actual current PDFs**, confirmed present as of this handoff, live
  directly in `...\OneDrive\Documents\HOLECH-BEDARKI\5_BUILD\rebuild\`:
  - `book_std.pdf` (regular edition, 96pp — this is
    the one to rasterize for the `regular` web edition)
  - `book_large.pdf` (large-print edition, 106pp — rasterize for `large`)
  - `book_proof.pdf` (~3.9MB, an A4 proofreading copy — **not** a flipbook
    source)
  - `book_std_hires.pdf` / `book_large_hires.pdf` / `book_proof_hires.pdf`
    (~13-14MB each — these are print-target, likely CMYK/high-DPI exports
    for the physical print house, **not** what you want to rasterize for a
    web flipbook; use the corresponding non-`_hires` file instead)

**Steps, using a new final print-ready PDF pair (or the current
`book_std.pdf`/`book_large.pdf` if that's genuinely what you want to
re-rasterize):**

1. Fix `SRC` in `ebook_src/rasterize.py` and `ebook_src/rasterize_large.py`
   to point at your actual new/current PDF (e.g.
   `...\OneDrive\Documents\HOLECH-BEDARKI\5_BUILD\rebuild\book_std.pdf`
   for the regular edition), **not** the stale `out\..._RGB_trim.pdf` path
   currently hardcoded there.
2. Also fix `DST` in both scripts — they currently write to a staging
   location (`rebuild\flip\pages` / `rebuild\flipL\pages`) using a
   **different filename convention** (`pNNN.jpg`, e.g. `p001.jpg`) than
   what the deployed site actually expects
   (`images/<edition>/page_NNN.jpg`, e.g. `page_001.jpg`). Either change
   `DST` to point directly at
   `C:\Users\yohasson\.scout\hebrew-flipbook\images\regular` (and `\large`
   for the large script) and change the output filename pattern from
   `f"p{i+1:03d}.jpg"` to `f"page_{i+1:03d}.jpg"`, or rasterize to the
   staging folder as-is and then copy+rename into the deployed `images/`
   folders afterward — don't assume the site will pick up `pNNN.jpg` files,
   it won't (`mksite4.py` just counts *any* `.jpg` file via `os.listdir`,
   so stray `pNNN.jpg` files alongside real `page_NNN.jpg` ones would
   silently inflate the page count with duplicates/junk).
3. **Always delete old `page_NNN.jpg` files from the deployed
   `images/regular` and `images/large` folders first** before writing the
   new rasterized set, regardless of whether you think the new PDF is
   longer or shorter. `mksite4.py` just counts whatever `.jpg` files it
   finds, so stale leftovers from any previous book will silently show up
   as extra/wrong pages or duplicate pages.
4. Run both: `python rasterize.py` then `python rasterize_large.py`.
   Renders at 105 DPI, downsized to fit within 760×1100px, JPEG quality 76
   — reasonable defaults for a screen flipbook; adjust in the script if a
   sharper/smaller trade-off is wanted.
5. Rebuild and redeploy:
   ```powershell
   cd "...\HOLECH-BEDARKI\5_BUILD\ebook_src"
   python mksite4.py
   cd C:\Users\yohasson\.scout\hebrew-flipbook
   git add -A
   git commit -m "Replace source PDF / regenerate pages"
   git push origin main
   ```
   `mksite4.py` auto-detects the new page counts from the folders — no
   manual page-count edits needed anywhere, as long as the folders contain
   only the fresh `page_NNN.jpg` files for the new PDFs.
6. Verify per §10 (hash/byte-compare the live deployed `index.html` against
   local after ~75s, don't just trust the push).

**Always regenerate and verify `tocmap_std.json` / `tocmap_large.json` when
the PDFs are replaced**, even if the page count happens to look similar. The
clickable TOC is live product functionality now, not just documentation data.
Also sanity-check §7: edition-switch continuity is still a proportional
approximation, and its accuracy depends on the two editions' actual page
counts.

**If you're handing this task to a different AI agent/session** (e.g. one
that has the new PDF loaded but no memory of this project), give it: the
verified current PDF locations and names above, the exact
`rasterize.py`/`rasterize_large.py` paths (with the three staleness issues
called out explicitly so it doesn't propagate them), the deployed repo
path and `page_NNN.jpg` naming convention, the "delete stale leftover files
before writing the new image set" warning, and the full rebuild+deploy+verify
command sequence above.

---

If something here turns out to be wrong or incomplete once you're deeper
into the rebuild, please update this file rather than letting it rot —
the next person after you will thank you.
