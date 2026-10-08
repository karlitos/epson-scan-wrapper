# scan-batch (v2, Node)

Group an **Epson ET-3700** flatbed scanner's one-file-per-press scans into a
single, OCR'd, **searchable PDF** — driven entirely from the terminal.

The ET-3700 is flatbed-only. Each panel press of **Scan → Computer → Paperless**
produces exactly one file; the device has no concept of a multi-page "session".
`scan-batch` adds that session grouping on the host: it starts the stock
[`epson2paperless`](https://github.com/mtheuma/epson2paperless) daemon, watches
its output folder as you scan page after page, and when you press **ENTER** it
merges the pages (in scan order) into one PDF and runs OCR so the text is
selectable and searchable. It also gives you **PDF size controls** so a routine
multi-page document doesn't balloon to several megabytes.

## This is a Node rewrite of the bash v1 — and why

v1 was a macOS `bash` 3.2 script. `epson2paperless` itself needs Node to run, so
**Node is already guaranteed present** on any machine that can use this tool —
which removes the only reason the wrapper was ever bash. Rewriting in Node (one
file, zero third-party runtime dependencies, Node standard library only) buys:

- **Proper event-driven concurrency.** v1 polled stdin with a 1-second `read -t`
  loop because bash can't easily watch a folder and wait for ENTER at the same
  time. v2 uses `fs.watch` (plus a ~1 s `readdir` fallback sweep, because
  `fs.watch` is flaky on macOS) racing a `readline` ENTER — no polling hack.
- **Robust daemon process management.** v2 spawns `npm run dev` *detached* (its
  own process group), reads the child's **stdout** directly for the ready line,
  and kills the whole process group (`SIGTERM`, then `SIGKILL`) so nothing is
  ever orphaned on TCP 2968.
- **Cleaner arg handling and real size controls** (see below).

The command name, flags, env vars, and day-to-day behavior are unchanged from
v1; v1 remains on the `main` branch as a fallback. OCR still shells out to
`ocrmypdf`; assembly shells to `img2pdf`/ImageMagick; size re-encode uses
ImageMagick/`sips`; PDF-mode concatenation uses `qpdf`.

## Relationship to epson2paperless (and why this is separate)

This wrapper is a **thin host-side layer that is intentionally SEPARATE from and
does not modify** the `epson2paperless` repo. It only uses that project's public
surface: the `PRINTER_IP` / `OUTPUT_DIR` / `SCAN_RESOLUTION` environment
variables, `npm run dev`, and its stdout log line `epson2paperless ready …`.
`epson2paperless` deliberately keeps OCR and page-merging logic out of its scope,
so that work lives here. This layer becomes **optional** once your document stack
(e.g. Paperless-ngx) performs OCR server-side — at that point you can drop the
OCR/merge step and just let the daemon deposit pages.

## What it does, step by step

1. **Startup** — quits the Epson helper apps that would otherwise hold the scan
   port (see caveat below), starts the daemon pointed at a hidden staging folder,
   and waits for it to report ready (reading the daemon's stdout).
2. **During the batch** — watches the staging folder. Each time a new page lands
   it prints `Page N captured (<file>)`. Keep scanning as many pages as you like.
3. **Finish** — press **ENTER**. You're prompted for a document name (ENTER for a
   timestamp default). The pages are assembled into one multi-page PDF (applying
   your size controls) and OCR'd with `ocrmypdf -l deu+eng`. The finished PDF
   lands in your output folder.
4. **Cleanup** — on a **successful** finish the daemon is stopped (no orphan on
   the scan port) and the raw per-page scans are deleted. **On interrupt the raw
   pages are kept** (see the four fixed behaviors). Finished PDFs are never
   touched.

## Prerequisites

- **Node ≥ 24.** (`node --version`.) No `npm install` is needed for this tool —
  it has **zero runtime dependencies**.
- **epson2paperless, run from source.** Default location `~/Dev/epson2paperless`
  (override with `--e2p`). Launched via `npm run dev` (needs its own
  `node_modules` present). The wrapper sets `PRINTER_IP`, `OUTPUT_DIR`, and
  (when `--dpi` is given) `SCAN_RESOLUTION` for it; it does not modify the repo.
- **ocrmypdf** (`brew install ocrmypdf`) — mandatory. Makes the PDF searchable.
- **tesseract with the `deu` and `eng` language packs**
  (`brew install tesseract tesseract-lang`). The tool fails fast if a requested
  language is missing.
- **An image→PDF assembler**: `img2pdf` (preferred — lossless, no re-encode) or
  **ImageMagick** (`magick`) as a fallback (`brew install img2pdf`). Auto-detected.
- **ImageMagick or `sips`** for the size controls (`--quality`, `--grayscale`).
  ImageMagick is preferred; `sips` is the fallback. If neither is present the
  tool ships the raw JPEG and warns that size controls were skipped.
- Optional: **`qpdf`** (`brew install qpdf`) to handle PDF-mode scans;
  **`jbig2enc`** (binary name `jbig2`) and **`pngquant`** for `--optimize 2/3`
  (`brew install jbig2enc pngquant`). All optional tools are detected at runtime
  and the tool degrades gracefully (warns and falls back) when they are absent.
- Optional for the self-test: `pdfinfo` / `pdftotext` (from `poppler`).

### Epson helper-app caveat (important)

The macOS Epson helper apps — **Epson ScanSmart**, **Epson Scanner Monitor**, and
**Epson Event Manager** — grab the scanner's discovery/scan port (TCP **2968**).
If any of them is running, the daemon can't bind the port. `scan-batch` quits
them automatically on startup and reports what it quit. If you still hit a
port-in-use error, see *Known failure modes* below.

## Installation

```sh
chmod +x ~/Dev/epson-scan-wrapper/scan-batch
# optional: put it on your PATH
ln -s ~/Dev/epson-scan-wrapper/scan-batch /usr/local/bin/scan-batch
```

## Usage

```sh
./scan-batch [OUTPUT_DIR] [options]
```

Day-to-day:

```sh
./scan-batch
# -> helper apps quit, daemon starts
# -> "Ready — scan your pages from the printer panel ..."
# scan page 1 as JPEG from the panel (Scan -> Computer -> Paperless -> Save as JPEG)
#    -> "Page 1 captured (scan_...jpg)."
# scan page 2, 3, ... each prints a "Page N captured" line
# press ENTER
# -> "Name for this document (ENTER for timestamp default):"  type a name or ENTER
# -> assembles + OCRs
# -> "Done. Searchable PDF (N page(s)) written to: <path>"
```

### CLI flags

| Flag | Meaning | Default |
|------|---------|---------|
| `--out <dir>` (or a positional first arg) | Output folder for the finished PDF | `~/Documents/Scans` |
| `--staging <dir>` | Hidden work folder for raw per-page scans | `<OUTPUT_DIR>/.work` |
| `--printer-ip <ip>` | Printer IP passed to the daemon | `192.168.178.3` |
| `--e2p <path>` | Path to the epson2paperless checkout | `~/Dev/epson2paperless` |
| `--langs <langs>` | OCR languages for `ocrmypdf -l` | `deu+eng` |
| `--quality <1-100>` | Re-encode each page JPEG at this quality during assembly | `85` |
| `--full-quality` | Skip re-encode; keep the raw scanner JPEG (photos/maps) | off |
| `--optimize <0-3>` | Pass to `ocrmypdf --optimize N` | `0` |
| `--dpi <n>` | Scan resolution (`SCAN_RESOLUTION`) for the daemon | scanner default |
| `--grayscale` / `--color-mode gray` | Convert pages to grayscale before assembly | off (color) |
| `--self-test` | Run the offline self-test (no daemon, no scanner) and exit | — |
| `--no-daemon` | Skip helper-app quit + daemon startup (file-based mode) | — |
| `-h`, `--help` | Show help and exit | — |

`PRINTER_IP`, `E2P_DIR`, `OUT_DIR`, and `OCR_LANGS` may also be set via the
environment; **flags take precedence over env, env over the built-in defaults.**
A leading `~` in a path argument is expanded to your home directory.

### PDF size controls — the default and why

**The dominant lever on file size is JPEG re-encode quality.** Measured fact
behind this: at the *same* 300 DPI / full-color RGB, Epson ScanSmart produces
~290 KB/page while a straight passthrough of the daemon's near-max-quality JPEG
is ~1.8 MB/page — roughly **6× larger** for visually identical text documents.
The only real difference is the JPEG quality the page is encoded at.

So v2 **re-encodes each page at `--quality 85` by default.** On text documents
this lands near ScanSmart parity with no visible loss and fixes the "2.6 MB /
3.8 MB is too big" problem out of the box. Opt out with **`--full-quality`** when
you're scanning photographs or maps where re-encode artifacts matter — that
preserves the raw scanner JPEG byte-for-byte.

**The re-encode applies to BOTH scanner output formats.** The `epson2paperless`
daemon can deposit each page as a JPEG *or* as a single-page PDF (its default
`SCAN_FORMAT` is `pdf`). JPEG pages are re-encoded with ImageMagick/`sips`; PDF
pages are **rasterized at 300 DPI and re-encoded at `--quality`** with
ImageMagick *before* assembly. Earlier builds only re-encoded JPEG pages and
concatenated PDF pages losslessly, so a real PDF-format scan stayed at full
quality (~1.4 MB/page, ~4.3 MB for 3 pages). That is now fixed: a default-quality
3-page PDF-format scan lands at **~300 KB/page / ~0.9 MB total**. `--full-quality`
still preserves PDF pages via lossless `qpdf` concatenation.

> **Note:** PDF-page size reduction needs **ImageMagick** (`magick`) — `sips`
> cannot rasterize a PDF. On a machine with only `sips` (or no re-encoder), PDF
> pages fall back to lossless `qpdf` concatenation and the tool warns that
> `brew install imagemagick` is required to shrink them. JPEG pages are
> unaffected and still re-encode with `sips`.

- `--quality 1-100` — lower = smaller. ~80–85 is the sweet spot for text docs.
- `--grayscale` — drop color before assembly; helps on color scans of B/W docs.
- `--optimize 0-3` — hands off to `ocrmypdf --optimize N`. `0` (default) leaves
  it to the quality/grayscale levers. `2`/`3` add lossless image optimization and
  need `jbig2enc`/`pngquant`; if those are missing the tool **warns and falls
  back to the highest level it can run** rather than failing.
- `--dpi n` — forwarded to the daemon as `SCAN_RESOLUTION`. The ET-3700
  advertises **75/150/300/600**; other values warn and pass through, and whether
  the scanner honors a given DPI is decided by the device at session time, so
  this flag cannot *guarantee* an effect. Lower DPI is the biggest size lever of
  all when you don't need 300 DPI detail.

**Measured sizes** (see *Verification* at the bottom) confirm the lever is
monotonic — `--quality 60 < --quality 85 (default) < --full-quality` on the same
real-sized 300-DPI pages — and that `--optimize 3` shrinks the result further
still. On a realistic full-page 300-DPI scan the default q85 re-encode produces
~300 KB/page versus ~1.4 MB/page at full quality (a ~4.6× reduction). Real
documents (with whitespace) compress even more, so ScanSmart's real-world
~290 KB/page is a realistic target at these settings; lowering `--dpi` or using
`--grayscale` on B/W documents reduces it further.

## How grouping and finish work

- Grouping is **host-side**. The scanner has no sessions: one panel press = one
  page = one file in the staging folder.
- `epson2paperless` writes `scan_YYYY-MM-DD_HHMMSS.<ext>` (and a `_1`/`_2`
  collision suffix if needed). The filename timestamp order equals the scan order
  equals the merge order, so pages always end up in the order you scanned them.
  The watcher accepts `scan_*.{jpg,jpeg,pdf}` and only once a file's size is
  stable across two checks, so a half-written file is never grabbed.
- **ENTER** ends the batch. Zero pages scanned → the tool says so and exits
  without creating a PDF.
- The document name is sanitized (path separators stripped; empty-after-trim
  falls back to the timestamp default; `.pdf` appended if missing). If the name
  already exists in the output folder it is **not overwritten** — a numeric
  suffix (`_1`, `_2`, …) is appended instead.

## The four fixed behaviors (vs v1)

1. **Self-test drives the REAL finish path.** `--self-test` injects simulated
   page files into staging and calls the exact `finishBatch()` function the real
   run uses — not a parallel copy — then asserts page count, a real text layer,
   cleanup, interrupt preservation, PDF-input handling, and the size lever.
2. **PDF-mode input is handled — and size-reduced.** If pages arrive as
   single-page **PDFs** (the daemon's default `SCAN_FORMAT=pdf`, or the panel's
   "Save as PDF" choice), the default path now **rasterizes each page at 300 DPI
   and re-encodes it at `--quality`** (ImageMagick) before a lossless `img2pdf`
   assembly, then OCRs normally — so PDF-format scans shrink exactly like JPEG
   ones (~300 KB/page at q85 instead of ~1.4 MB/page). With **`--full-quality`**
   the pages are instead concatenated losslessly with `qpdf --empty --pages … --`
   and OCR'd with `--skip-text` (so a page that already carries text doesn't abort
   the run). Without ImageMagick, PDF pages fall back to the lossless `qpdf`
   concat with a warning. A mixed JPEG+PDF batch is rejected with a clear
   "re-scan the whole batch in one format" message rather than silently dropping
   pages.
3. **Ctrl-C preserves captured pages.** Interrupting mid-batch does **not** delete
   staging. The tool prints exactly where your captured pages are and the command
   to resume/assemble them:
   ```
   scan-batch --no-daemon --staging '<staging dir>' --out '<out dir>'
   ```
   then press ENTER. (A normal successful finish still cleans up.)
4. **Meaningful exit codes.** `130` on SIGINT (Ctrl-C), `143` on SIGTERM, so
   automation can tell an interrupt from a real error. Other codes: `0` success,
   `2` port busy, `3` daemon not ready, `4` assemble failed, `5` OCR failed.

## Cleanup behavior

- Raw per-page scans live in a **hidden** `.work/` staging folder inside the
  output folder (configurable via `--staging`).
- On **success**: staging raw pages, intermediate processed/merged temp files,
  and `daemon.log` are deleted; the empty staging folder is removed. The finished
  PDF is kept.
- On **failure** (e.g. OCR error): staging is **kept** so your raw pages are
  recoverable, and `daemon.log` is kept for diagnosis. The tool prints exactly
  where the pages are.
- On **interrupt** (Ctrl-C / TERM): staging is **kept** with the recovery command
  printed (fix #3); the daemon is still stopped so nothing orphans on the port.
- The tool **never** deletes anything in the output-folder root except its own
  temp files, and **never** touches finished PDFs.

## Known failure modes and fixes

- **Port 2968 already in use / daemon won't start.** Usually an Epson helper app
  or a stale daemon. The tool detects this before starting anything and prints:
  ```
  Find it:   lsof -nP -iTCP:2968 -sTCP:LISTEN
  Kill it:   kill $(lsof -nP -tiTCP:2968 -sTCP:LISTEN)
  ```
  Quit ScanSmart / Scanner Monitor / Event Manager, or kill the stale process,
  then re-run. (Exit code `2`.)
- **Daemon never reports ready** (timeout ~30 s). Check the printer IP
  (`--printer-ip`) and the tail of `<staging>/daemon.log` that the tool prints.
  Confirm the printer is on and reachable. (Exit code `3`.)
- **ocrmypdf fails.** The run fails loudly, the raw pages are **kept** in staging
  (path printed), and `daemon.log` is kept. Fix the cause (often a corrupt page)
  and re-assemble from the kept pages. (Exit code `5`.)
- **`--optimize 2/3` warns about jbig2enc/pngquant.** Those optional tools aren't
  installed; install with `brew install jbig2enc pngquant`, or accept the
  automatic fall-back to a lower optimize level.
- **PDF-mode scan rejected / "scan as JPEG".** You mixed JPEG and PDF pages in one
  batch, or a PDF couldn't be concatenated. Re-scan the batch as JPEG.
- **"Communication problem — ensure a computer is attached" on the printer.** The
  daemon isn't running/ready when you pressed Scan. Start `scan-batch` first and
  wait for the Ready prompt; on the panel use *Scan → Computer → Paperless*.
- **Mac sleeps mid-batch.** If the display/system sleeps between pages, the
  in-flight scan to the daemon can fail. `scan-batch` holds the Mac awake for the
  duration of a batch with macOS `caffeinate -dimsu` (started at watch time,
  stopped on cleanup); if `caffeinate` isn't available it warns and continues. In
  practice a scan interrupted by sleep can be recovered by waking the Mac and
  re-selecting *Paperless* on the panel to continue the batch.

## Verification (what was actually run)

All of the following were run on this machine — **no live scanner** was involved
— and are reproducible with `./scan-batch --self-test`.

- **Environment:** macOS, `node v24.19.0`; `ocrmypdf` 17.x, `tesseract` with
  `deu`+`eng`, `img2pdf`, ImageMagick 7.x (`magick`), `qpdf`, `jbig2` (jbig2enc),
  `pngquant`, `pdfinfo`/`pdftotext` all present.
- **`node --check scan-batch`** → syntax OK.
- **`eslint`** → **not installed** on this machine, so it was skipped. If you
  install it (`npm i -g eslint` or `brew install eslint`), run `eslint scan-batch`.
- **`./scan-batch --self-test`** → exit `0`, all assertions pass. The self-test
  now includes two **realistic-sized** PDF-input cases ([5/10] and [6/10]) that
  drive the real `finishBatch()` on full-page 2477×3500 300-DPI q95 pages
  (~1.4 MB each) — the input size needed to expose the re-encode bug — and assert
  the embedded images via `pdfimages -list`:

```
=== scan-batch self-test ===
Node: v24.19.0
Assembler: img2pdf; re-encoder: magick; jbig2=true pngquant=true
[1/10] Positive: 3 JPEGs -> OCR searchable PDF (real finishBatch)
  OK: page count == 3 (pdfinfo)
  OK: real text layer present (pdftotext)
[2/10] Success cleanup removes staging, keeps finished PDF
  OK: staging cleaned (no scan_* / daemon.log)
  OK: finished PDF untouched by cleanup
[3/10] Simulated interrupt preserves captured pages (fix #3)
  OK: raw page preserved under interrupted status
  OK: interrupt maps to preserve (exit-code intent 130)
[4/10] PDF-mode input -> 2-page searchable PDF (real finishBatch)
  OK: PDF-input batch -> 2-page searchable PDF
[5/10] REALISTIC PDF-input default q85 recompresses pages (<700 KB/page)
  OK: each PDF page recompressed to <700 KB at q85 (input ~1409 KB/page; got 299K, 300K, 300K)
  OK: realistic PDF-input -> 3-page PDF
  OK: realistic PDF-input has a searchable text layer (OCR ran, no --skip-text)
  MEASURED (PDF-input q85): input ~1409 KB/page -> 299K, 300K, 300K embedded, total 907.3 KB
[6/10] REALISTIC PDF-input --full-quality preserves pages (>1000 KB/page)
  OK: --full-quality preserves each PDF page near scanner size (>1000 KB; got 1408K, 1406K, 1407K)
  MEASURED (PDF-input full-quality): 1408K, 1406K, 1407K embedded, total 4.13 MB
[7/10] Size controls reduce size (q80 < full-quality)
  OK: q80 (37.1 KB) < full-quality (47.2 KB) and default q85 (40.9 KB) < full-quality
[8/10] No daemon started in self-test (no orphan possible)
  OK: no daemon child spawned
[9/10] Watcher detects + orders two injected files
  OK: watcher saw 2 ordered files: scan_2026-01-01_000001.jpg, scan_2026-01-01_000002.jpg
[10/10] --help lists new size-control flags
  OK: help text includes --quality/--full-quality/--optimize/--dpi/--grayscale
=== SELF-TEST PASS ===
```

- **Real interrupt codes** (spawned the actual tool in `--no-daemon` mode with a
  staged page, then signalled it): **SIGINT → exit 130**, **SIGTERM → exit 143**,
  and in both cases the captured page was **preserved** in staging (fix #3/#4).

- **Measured output sizes** on **realistic full-page scanner input** — three
  2477×3500 (300-DPI) RGB JPEG pages re-encoded at q95 (~1.41 MB/page, mimicking
  the daemon's near-max-quality output), each wrapped as a single-page PDF and
  run through the real finish/assembly path (no scanner). Per-page figures are
  the embedded-image sizes from `pdfimages -list`:

  *PDF-format input (the daemon's default `SCAN_FORMAT=pdf`), 3 pages:*

  | Mode | Per page (embedded) | 3-page total (merged) |
  |------|---------------------|-----------------------|
  | input (scanner original) | ~1409 KB | ~4.33 MB |
  | `--full-quality` (lossless `qpdf` concat) | ~1407 KB | ~4.13 MB |
  | default (`--quality 85`) | ~303 KB | ~0.93 MB |
  | `--quality 60` | ~83 KB | ~0.26 MB |

  The quality lever is monotonic: **`--quality 60` (83 KB/pg) < `--quality 85`
  (303 KB/pg) < `--full-quality` (1407 KB/pg)** on the same pages. `--optimize`
  shrinks the result further still — on the same q85 PDF, `ocrmypdf --optimize 3`
  took the finished file from ~929 KB down to **~137 KB** (jbig2enc + pngquant
  installed). Before this fix, a 3-page PDF-format scan stayed at ~4.3 MB because
  the pages were concatenated losslessly and never recompressed; the default now
  lands near ScanSmart's real-world **~290 KB/page** target. Lowering `--dpi` or
  using `--grayscale` on B/W documents reduces it further.
