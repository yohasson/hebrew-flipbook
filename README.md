# הולך בדרכי — ספר לדפדוף

סיפור חייו של יעקב קבלה. ספר דיגיטלי לדפדוף, בעברית, מימין לשמאל.

**לקריאה: https://yohasson.github.io/hebrew-flipbook/**

---

## Holech Bedarki — RTL Hebrew Flipbook

A 3D page-turning web edition of a Hebrew family memoir.

- Right-to-left reading order — the book opens from the right
- Two-page spread on desktop, single-page swipe on mobile
- Two editions: standard (68 pp) and large print (84 pp, 14 pt)
- Hebrew UI: next / previous page, page counter, fullscreen
- Keyboard: `→` back, `←` forward, `F` fullscreen

### Structure

```
index.html            the flipbook
images/regular/       standard edition, 68 pages
images/large/         large-print edition, 84 pages
```

Page images are served at screen resolution so the site loads quickly on
mobile data. The 300 DPI print masters are kept outside this repository.

Built with [StPageFlip](https://github.com/Nodlik/StPageFlip) (MIT),
vendored locally in `vendor/` — see `HANDOFF.md` before changing that.

**Rebuilding this site from a new edit of the book?** Read `HANDOFF.md`
first — it covers the full build pipeline, a real pagination mismatch
between the current print PDFs and these deployed images, everything
measured against the Heyzine reference this was styled on, and several
non-obvious bugs (and their fixes) that only surface under real
interaction testing, not a quick look.

See `NOTICE.txt` — the memoir text and photographs are not licensed for reuse.
