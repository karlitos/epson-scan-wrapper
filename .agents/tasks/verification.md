# Verification — scan-batch v2 PDF-page re-encode size fix

All verification below is **file-based** (no live scanner, no daemon). The
reviewer can read this without re-running anything.

- **Branch:** `v2` (not `main`).
- **Environment:** macOS, `node v24.19.0`. Tools present: `magick` (ImageMagick 7),
  `img2pdf`, `qpdf`, `ocrmypdf`, `tesseract` (deu+eng), `pdfimages`/`pdfinfo`/
  `pdftotext` (poppler), `jbig2` (jbig2enc), `pngquant`, `sips`, `caffeinate`.
- **Git identity (unchanged, repo-local):** `user.name=karlitos`,
  `user.email=karel.macha@karlitos.net`.

---

## Root cause (confirmed by reproduction)

The default `--quality 85` re-encode was bypassed for **PDF-format** scanner
pages. The epson2paperless daemon's default `SCAN_FORMAT` is `pdf`, so a real
panel scan deposits `scan_*.pdf` pages. In `buildMergedPdf`, the `allPdf` branch
called `concatPdfs()` (`qpdf --empty --pages … --`), which is **lossless** and
preserved each page's full-quality JPEG. `processImage` (the only place
`cfg.quality` was applied) was never reached on the PDF route. The old self-test
only exercised the JPEG route with tiny synthetic pages, so the bug slipped
through.

### Reproduction (the bug), 3 realistic PDF-format pages

Input page: `magick -size 2477x3500 xc:white ( plasma:fractal ) -compose blend
… -density 300 -units PixelsPerInch -quality 95` ≈ **1,443,050 bytes/page**
(~1.41 MB), wrapped as single-page PDFs via `img2pdf`.

Old PDF path (`qpdf` concat) → `pdfimages -list`:

```
page  width height color enc   x-ppi y-ppi size
  1   2477  3500  rgb   jpeg   300   300  1409K
  2   2477  3500  rgb   jpeg   300   300  1411K
  3   2477  3500  rgb   jpeg   300   300  1410K
total PDF bytes: 4,332,712   (~4.33 MB)
```

This matches the user's reported **4.7 MB / ~1.5–1.6 MB per page** symptom.

---

## The fix

`buildMergedPdf` PDF branch now:

- `--full-quality` → lossless `qpdf` concat (`fromPdf: true`, OCR `--skip-text`) —
  passthrough preserved.
- default (magick present) → **`processPdfPage()` rasterizes each PDF page at
  300 DPI and re-encodes at `cfg.quality`** (and `-colorspace Gray` when
  `--grayscale`), then `img2pdf` assembles; OCR runs normally (`fromPdf: false`,
  no `--skip-text`).
- no ImageMagick (sips/none) → warn + lossless `qpdf` concat fallback.

Rasterize command (300-DPI faithful, re-tagged 300 PPI on output):
`magick -density 300 <page>.pdf[0] [-colorspace Gray] -units PixelsPerInch -density 300 -quality <q> page_NNNN.jpg`

---

## `node --check scan-batch`

```
SYNTAX_OK   (exit 0)
```

## No hardcoded home path in shipped files

```
grep -rn '/Users/Karel' scan-batch README.md package.json
→ (no matches, grep exit 1)
```

---

## `./scan-batch --self-test` → exit 0, all 10 steps pass

Key new realistic-sized PDF-input assertions (drive the real `finishBatch()` on
2477×3500 300-DPI q95 ~1.4 MB pages):

```
[5/10] REALISTIC PDF-input default q85 recompresses pages (<700 KB/page)
  OK: each PDF page recompressed to <700 KB at q85 (input ~1409 KB/page; got 299K, 300K, 300K)
  OK: realistic PDF-input -> 3-page PDF
  OK: realistic PDF-input has a searchable text layer (OCR ran, no --skip-text)
  MEASURED (PDF-input q85): input ~1409 KB/page -> 299K, 300K, 300K embedded, total 907.3 KB

[6/10] REALISTIC PDF-input --full-quality preserves pages (>1000 KB/page)
  OK: --full-quality preserves each PDF page near scanner size (>1000 KB; got 1408K, 1406K, 1407K)
  MEASURED (PDF-input full-quality): 1408K, 1406K, 1407K embedded, total 4.13 MB

[8/10] ...
  OK: no caffeinate process spawned in self-test (detected on PATH: true)

=== SELF-TEST PASS ===
```

Full transcript: steps [1/10]–[10/10] all `OK`, 0 `FAIL`, final `=== SELF-TEST PASS ===`, exit 0.
(Steps: 1 JPEG→searchable PDF, 2 success cleanup, 3 interrupt preserve, 4 PDF→2-page,
5 realistic PDF q85 shrink, 6 realistic PDF full-quality preserve, 7 q80<q85<full,
8 no daemon/caffeinate orphan, 9 watcher order, 10 help flags.)

---

## Task-brief acceptance criteria — measured on real-sized pages

Input: 3 pages @ **1,443,050 bytes** (~1.41 MB) each, 2477×3500, 300 ppi, rgb jpeg.

### (a) default `--quality 85`: pages substantially smaller + text layer
- Embedded per page: **299K / 300K / 300K** (`pdfimages -list`) — well under the
  ~600 KB target and ~4.6× smaller than the ~1409 KB input.
- 3-page PDF total (post-OCR): **907.3 KB** (vs ~4.33 MB pre-fix). Under ~1 MB. ✓
- Searchable text layer present (`pdftotext` non-empty, OCR ran without
  `--skip-text`). ✓

### (b) `--full-quality`: NOT recompressed (≈ input)
- Embedded per page: **1408K / 1406K / 1407K** (≈ 1409K input). Total 4.13 MB. ✓

### (c) monotonic `--quality 60 < 85 < --full-quality` (same real pages, merged pre-OCR)
```
q60 : 82.9K / 83.0K / 83.1K per page   total 257,470 bytes
q85 : 303K  / 303K  / 304K  per page   total 934,184 bytes
full: 1409K / 1411K / 1410K per page   total 4,332,712 bytes
→ 83K  <  303K  <  1410K  (per page)   ✓ monotonic
```

### (d) `--optimize` shrinks further (same q85 PDF, post-OCR)
```
ocrmypdf --optimize 0 : 929,083 bytes
ocrmypdf --optimize 3 : 136,617 bytes   (jbig2enc + pngquant)
→ optimize 3 is ~6.8× smaller than optimize 0   ✓
```

---

## Secondary — Mac-sleep resilience (implemented, small & safe)

`startCaffeinate()` spawns `caffeinate -dimsu` at watch time on the real path
(both daemon and `--no-daemon`); `finalize()` calls `stopCaffeinate()` alongside
`killDaemon()`. No-op with a warning if `caffeinate` is absent. **Never started
in `--self-test`** — asserted by step [8/10]. README known-issues documents the
sleep behavior and recovery.

---

## Cleanup

All temp artifacts created during verification were under `/tmp` and self-test
`$TMPDIR` temp roots; the self-test removes its own temp root. No stray files
left in the repo.
