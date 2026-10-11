# scan-batch (v2.1, Node)

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
- **`ghostscript`** (`gs`, `brew install ghostscript`) — used by `--mrc` to
  downsample the display image after full-resolution OCR. Without it `--mrc`
  still works but keeps the full-resolution (larger) image.
- Optional for **`--mrc`** (even smaller many-page PDFs): `archive-pdf-tools`
  (`recode_pdf`) in a Python 3.13 venv, plus `jbig2enc` and `openjpeg` — see
  *MRC compression* below for the exact install commands. Without it, `--mrc`
  automatically uses the full-resolution-OCR + Ghostscript downsample fallback.

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
# -> OCRs full-res, then assembles the default MRC color 1.5x (email-sized) PDF
# -> "Done. Searchable PDF (N page(s)) written to: <path>"
```

By default this produces an **email-sized, searchable, full-color** PDF (MRC
color 1.5×). Need full resolution instead? Add `--no-mrc` (or `--quality <n>` /
`--full-quality`). See *Quality ladder — which should I use?* below.

### CLI flags

| Flag | Meaning | Default |
|------|---------|---------|
| `--out <dir>` (or a positional first arg) | Output folder for the finished PDF | `~/Documents/Scans` |
| `--staging <dir>` | Hidden work folder for raw per-page scans | `<OUTPUT_DIR>/.work` |
| `--keep-staging` | On a **successful** finish, keep the raw `scan_*` pages instead of deleting them (inspection / reuse) | off |
| `--printer-ip <ip>` | Printer IP passed to the daemon | `192.168.178.3` |
| `--e2p <path>` | Path to the epson2paperless checkout | `~/Dev/epson2paperless` |
| `--langs <langs>` | OCR languages for `ocrmypdf -l` | `deu+eng` |
| `--no-mrc` | Opt out of the default MRC; use the full-resolution (q70) path | off (MRC on) |
| `--quality <1-100>` | Full-resolution JPEG re-encode at this quality (implies `--no-mrc`) | `70` |
| `--full-quality` | Full-resolution, no re-encode; raw scanner JPEG (implies `--no-mrc`) | off |
| `--optimize <0-3>` | Pass to `ocrmypdf --optimize N` (on the `--no-mrc` path) | `0` |
| `--mrc` | MRC compression — the **default**. This flag re-enables it after a `--quality`/`--no-mrc` on the same command | **on** |
| `--mrc-bg-downsample <n>` | Display-image downsample factor (≥1, fractional OK); `1` = full-res image, `2` = ~150 dpi | `1.5` |
| `--mrc-fallback` | Force the full-res-OCR + Ghostscript downsample MRC path (implies `--mrc`) | off |
| `--dpi <n>` | Scan resolution (`SCAN_RESOLUTION`) for the daemon | scanner default |
| `--grayscale` / `--color-mode gray` | Convert pages to grayscale before assembly | off (color) |
| `--self-test` | Run the offline self-test (no daemon, no scanner) and exit | — |
| `--no-daemon` | Skip helper-app quit + daemon startup (file-based mode) | — |
| `--version` | Print tool name, version, git commit, and Node version; exit | — |
| `-h`, `--help` | Show help and exit | — |

`PRINTER_IP`, `E2P_DIR`, `OUT_DIR`, and `OCR_LANGS` may also be set via the
environment; **flags take precedence over env, env over the built-in defaults.**
A leading `~` in a path argument is expanded to your home directory.

### PDF size controls — the default and why

**The default is MRC color 1.5× — email-sized and searchable out of the box.**
The whole point of the tool is emailing multipage scans under the ~4–5 MB
attachment ceiling that authorities enforce, so the common case is small by
default. With no flags, `scan-batch` OCRs the full-resolution page (so small, and
small *colored*, text lands in the searchable text layer) and then downsamples
only the *displayed* image to ~1651×2333 (~200 dpi). The result is a sharp,
full-color, searchable PDF of ~250 KB/page — about **3.1 MB for 13 pages**,
comfortably under the ceiling. See *MRC compression* below for how it works and
*Real-page acceptance* for the measured numbers.

**Why the ET-3700 pages are so big to begin with.** The scanner emits a
near-max-quality (~q90) JPEG at ~1.8–2.0 MB/page (2477×3500, 300 DPI, RGB).
Ten of those straight through would be ~18–20 MB — far past any mail limit. The
default MRC profile is what brings that down while staying searchable.

**Opting out to full resolution.** When every pixel matters (tiny or colored
micro-print you must capture verbatim, photos, maps), opt out of MRC:

- **`--no-mrc`** — full-resolution path at `--quality 70` (crisp, larger: ~2.3 MB
  for 3 pages). Raise fidelity with `--quality <n>`.
- **`--quality <1-100>`** — full-resolution JPEG re-encode at your quality
  (implies `--no-mrc`); higher = sharper + bigger.
- **`--full-quality`** — full-resolution, no re-encode, raw scanner JPEG (implies
  `--no-mrc`); maximum fidelity, biggest files.
- Or stay on MRC and just make it sharper with **`--mrc-bg-downsample 1`**
  (full-resolution image, email size trade-off).

**On the `--no-mrc` path, the re-encode applies to BOTH scanner output formats.**
The `epson2paperless` daemon can deposit each page as a JPEG *or* as a single-page
PDF (its default `SCAN_FORMAT` is `pdf`). JPEG pages are re-encoded with
ImageMagick/`sips`; PDF pages are **rasterized at 300 DPI and re-encoded at
`--quality`** with ImageMagick *before* assembly. Earlier builds only re-encoded
JPEG pages and concatenated PDF pages losslessly, so a real PDF-format scan stayed
at full quality (~1.4 MB/page, ~4.3 MB for 3 pages). That is now fixed: a
`--no-mrc` PDF-format scan is recompressed exactly like a JPEG one.
`--full-quality` still preserves PDF pages via lossless `qpdf` concatenation.
(The default MRC path handles PDF-format pages too — it rasterizes them full-res
for OCR, then downsamples the display image like any other page.)

> **Note:** PDF-page size reduction needs **ImageMagick** (`magick`) — `sips`
> cannot rasterize a PDF. On a machine with only `sips` (or no re-encoder), PDF
> pages fall back to lossless `qpdf` concatenation and the tool warns that
> `brew install imagemagick` is required to shrink them. JPEG pages are
> unaffected and still re-encode with `sips`.

- `--quality 1-100` — lower = smaller. ~70 (the default) is the sweet spot for
  text docs; push to ~80–85 if you want a little more headroom on fidelity.
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

**On the `--no-mrc` path, measured sizes** (see *Verification* at the bottom)
confirm the quality lever is monotonic — `--quality 60 < --quality 70 (default)
< --full-quality` on the same real-sized 300-DPI pages. That path is for when you
deliberately want full resolution; for everyday emailing the default MRC profile
is already small (see *Real-page acceptance*).

## MRC compression (the default)

**Why this is the default.** The whole point of the tool is sending multi-page
PDFs to authorities by email, and those mailboxes routinely **reject attachments
over ~4–5 MB**. A 10–13 page color document at full-resolution q70 (~770 KB/page)
blows past that ceiling. MRC keeps a long color document both **searchable and
small enough to actually send**, so it is the no-flag default; opt out to full
resolution with `--no-mrc`, `--quality <n>`, or `--full-quality`.

**What MRC does here — and why the pipeline order matters.** The goal is a page
that is both *visually small* and *fully searchable*, including small print. The
shipped default achieves that by **OCR-ing the full-resolution page first, then
downsampling only the displayed image**:

1. assemble the pages at full scanner resolution (2477×3500);
2. run OCR on that crisp full-res page, so the text layer captures small — and
   small **colored** — print completely;
3. downsample **only the visible image** (Ghostscript) to ~1.5× smaller
   (~1651×2333, ~200 dpi) for an email-sized file. The text layer is untouched.

This order is the whole point: an earlier version downsampled *before* OCR, which
fed Tesseract a low-res page and dropped small text from the text layer. OCR-ing
full-res first keeps the complete text layer while the visible image still shrinks
for size. (Measured: the color 1.5× default captures the same OCR token count as
a full-resolution scan — see *Real-page acceptance* below.)

**It stays searchable.** The result always passes a `pdftotext` check before the
run is reported as done. If the optional `recode_pdf` engine is installed it is
used for even better compression (a bitonal JBIG2 text mask over a downsampled
background); otherwise the full-res-OCR + Ghostscript path above is used
automatically.

### Using it (and tuning it)

```sh
./scan-batch                             # DEFAULT: MRC color 1.5x, email-sized, searchable
./scan-batch --mrc-bg-downsample 1       # sharper (full-res image) when tiny/colored print must be caught
./scan-batch --mrc-bg-downsample 2       # smaller (~150 dpi image)
./scan-batch --grayscale                 # smallest; black-text-only (see caveat)
./scan-batch --no-mrc                    # opt out: full-resolution q70, no MRC
```

MRC is **on by default**; `--no-mrc`, `--quality <n>`, or `--full-quality` opt out
to the full-resolution path. On the MRC path you don't choose an engine — it uses
`recode_pdf` when installed (best size) and otherwise the guaranteed-installable
fallback (below) automatically.

> **Grayscale caveat.** `--grayscale` makes the file smaller but collapses colored
> small print (red stamps/notices) toward mid-gray, which can **hurt OCR and
> legibility** of that text. On the real test pages, grayscale dropped OCR capture
> from 592 → 546 tokens and lost a line of small red print. Use `--grayscale`
> only for black-text-only documents; color is the safer default.

### Installing the best engine (`recode_pdf`)

For the smallest output, install `archive-pdf-tools` (which provides
`recode_pdf`) into a dedicated Python 3.13 virtualenv. `scan-batch` looks for it
automatically. Copy-paste:

```sh
python3.13 -m venv ~/.local/share/scan-batch-mrc/venv
~/.local/share/scan-batch-mrc/venv/bin/pip install --upgrade pip wheel
~/.local/share/scan-batch-mrc/venv/bin/pip install archive-pdf-tools
# recode_pdf then lives at ~/.local/share/scan-batch-mrc/venv/bin/recode_pdf
```

The external binaries MRC needs are typically already present on a macOS box set
up for this tool: `jbig2` (jbig2enc), `opj_compress`/`opj_decompress` (openjpeg),
plus `tesseract` (+`deu`,`eng`), `ocrmypdf`, `img2pdf`, `qpdf`, ImageMagick,
`ghostscript`, and `poppler` (`pdfimages`/`pdfinfo`/`pdftotext`). Only
`archive-pdf-tools`' tiny Cython extension compiles from source (Apple `clang` is
enough). *Optional:* `brew install jpegoptim` — only needed if you switch the MRC
background to the JPEG (rather than JPEG2000) codec.

`scan-batch` finds `recode_pdf` in this order and uses the first hit:

1. the `SCAN_BATCH_RECODE_PDF` environment variable, if set;
2. `~/.local/share/scan-batch-mrc/venv/bin/recode_pdf` (the path above);
3. `recode_pdf` on your `PATH`.

### Automatic fallback (no `recode_pdf` needed) — the shipped default path

If `recode_pdf` is **not installed**, or a `recode_pdf` run fails / produces no
text layer, `scan-batch` does **not** error out. It automatically uses the
**full-resolution-OCR + Ghostscript downsample** path described above: assemble
full-res → `ocrmypdf` OCRs the crisp full-res page → Ghostscript downsamples only
the display image to the `--mrc-bg-downsample` target. This needs no `recode_pdf`
— only `ocrmypdf` (already required) and `ghostscript` (`gs`). The result is a
**searchable color PDF**; if `gs` is missing it stays full-resolution (still
searchable) with a warning. Force this path explicitly with **`--mrc-fallback`**
even when `recode_pdf` is installed. This is the path that produced the approved
real-page numbers below.

### Flag precedence and composition

- **MRC is the default.** `--quality <n>`, `--full-quality`, and `--no-mrc` each
  turn it off and select the full-resolution path. `--mrc` re-enables it (useful
  if it appears after one of those on the same command line).
- **Under MRC, `--quality` does not apply.** The display downsample controls the
  size, not the JPEG re-encode quality.
- **`--mrc` composes with `--grayscale`.** Grayscale yields a smaller file, but
  see the caveat above — it can hurt OCR of colored small print, so it is opt-in.
- **`--mrc` ignores `--optimize`.** The MRC path manages its own compression.
- **`--mrc` conflicts with `--full-quality`.** Giving both explicitly is rejected
  with a clear error (they are contradictory goals).
- **`--mrc-bg-downsample <n>`** (default `1.5`, fractional allowed) sets how much
  the **displayed image** is downsampled; OCR is always run on the full page
  first. `1` = full resolution (sharpest, largest); `2` = ~150 dpi (smaller).

## Quality ladder — which should I use?

`scan-batch` gives you a ladder from maximum fidelity to minimum size. OCR text
is always captured from the full-resolution page, so **every** option below stays
searchable; the choice is purely image fidelity vs file size.

| Option | What it does | Fidelity / size |
|--------|--------------|-----------------|
| **_(no flag — DEFAULT)_** | **MRC color 1.5×: full-res OCR, display image ~200 dpi** | **~3.1 MB / 13 pages, sharp, color, email-sized** |
| `--mrc-bg-downsample 1` | Full-resolution MRC image | Sharper MRC (bigger) — when tiny/colored print MUST be captured |
| `--mrc-bg-downsample 2` | More aggressive display downsample | Smaller MRC (~1.8 MB / 13 pages) |
| `--grayscale` | Grayscale MRC | Smallest (~2.9 MB / 13 pages); black-text-only (red/colored small-print caveat) |
| `--no-mrc` *(or `--quality <n>`)* | Full-resolution q70 (no MRC) | Crisp, larger (~10 MB / 13 pages) |
| `--full-quality` | Full-resolution, no re-encode; raw page | Max fidelity, biggest |

OCR text is always captured from the full-resolution page, so **every** option
above stays searchable; the choice is purely image fidelity vs file size.

**Pick by document type:**

- **Everyday / anything you'll email (the common case):** no flag. The default
  MRC color 1.5× is sharp, full-color, searchable, and email-sized out of the box.
- **A document with critical tiny or red print:** `--mrc-bg-downsample 1` (sharper
  MRC, still email-ish) or `--full-quality` / `--no-mrc` (full resolution) so the
  micro-print is preserved for OCR.
- **Long black-text-only document you want as small as possible:** `--grayscale` —
  but only when there is no colored small print to preserve (see caveat).
- **Photos or maps where every pixel matters:** `--full-quality` (raw) or
  `--no-mrc --quality 90`.

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
   ones (recompressed at the default q70 instead of staying ~1.4 MB/page). With **`--full-quality`**
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

## Verification

Verification has two tiers. The **SMOKE** tier (offline self-test, synthetic
pages) runs on every build and proves the code paths work. The **real-page**
tier is the acceptance gate for the size claims: synthetic images are not
representative of scanner output, so every size number used as a *claim* comes
from genuine ET-3700 pages, not the self-test.

### SMOKE — offline self-test (synthetic pages, directional only)

All of the following were run on this machine — **no live scanner** was involved
— and are reproducible with `./scan-batch --self-test`. The KB figures in this
block come from **synthetic** pages and are **SMOKE-only** (directional: they
prove the levers move the right way, not absolute real-scan sizes).

- **Environment:** macOS, `node v24.19.0`; `ocrmypdf` 17.x, `tesseract` with
  `deu`+`eng`, `img2pdf`, ImageMagick 7.x (`magick`), `qpdf`, `jbig2` (jbig2enc),
  `pngquant`, `pdfinfo`/`pdftotext` all present.
- **`node --check scan-batch`** → syntax OK.
- **`eslint`** → **not installed** on this machine, so it was skipped. If you
  install it (`npm i -g eslint` or `brew install eslint`), run `eslint scan-batch`.
- **`./scan-batch --self-test`** → exit `0`, all assertions pass. The self-test
  drives the real `finishBatch()` across 10 phases plus two MRC phases: realistic
  PDF-input cases ([5/10], [6/10]) on full-page 2477×3500 300-DPI pages assert the
  embedded images via `pdfimages -list`; `[MRC]` exercises the `recode_pdf` path
  (or SKIPs if it is not installed); `[MRC-FB]` always exercises the full-res-OCR
  + Ghostscript downsample fallback. The internal preset labels in the transcript
  (e.g. `q85`, `q95`) are self-test fixtures and do not reflect the shipped q70
  default. Abbreviated transcript:

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
  OK: --keep-staging preserves staged scan_* page
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
  OK: q80 (37.1 KB) < full-quality (47.1 KB) and default q85 (40.9 KB) < full-quality
[8/10] No daemon started in self-test (no orphan possible)
  OK: no daemon child spawned
[9/10] Watcher detects + orders two injected files
  OK: watcher saw 2 ordered files: scan_2026-01-01_000001.jpg, scan_2026-01-01_000002.jpg
[10/10] --help lists new size-control flags
  OK: help text includes --quality/--full-quality/--optimize/--dpi/--grayscale
[MRC] MRC smoke (recode_pdf path end-to-end, or SKIP if not installed)
  OK: SKIP MRC smoke (recode_pdf not installed; run --mrc for install commands)
[MRC-FB] MRC fallback smoke (full-res OCR + gs downsample; ALWAYS runs, no recode_pdf)
  OK: MRC fallback: output has 3 pages (pdfinfo)
  OK: MRC fallback: searchable text layer present (pdftotext status 0)
  OK: MRC fallback smaller than --full-quality (240.4 KB < 3.70 MB)
=== SELF-TEST PASS ===
```

> On this machine `recode_pdf` is **not installed**, so `[MRC]` SKIPs and the
> live `--mrc` path uses the full-res-OCR + Ghostscript downsample fallback (its
> deps — `ocrmypdf`, `img2pdf`, `gs`, `pdftotext` — are all present). Install
> `archive-pdf-tools` (commands above) for the even-smaller `recode_pdf` path.

- **Real interrupt codes** (spawned the actual tool in `--no-daemon` mode with a
  staged page, then signalled it): **SIGINT → exit 130**, **SIGTERM → exit 143**,
  and in both cases the captured page was **preserved** in staging (fix #3/#4).

> The self-test's embedded KB numbers above are **synthetic** (plasma pages with
> an OCR-able text band) and are kept only to show the levers are directionally
> correct. Do **not** read them as real-scan sizes — see the real-page tier next.

### Real-page acceptance (genuine ET-3700 color pages)

These are **real measurements** on 3 genuine ET-3700 color pages (native q90,
2477×3500, ~1.9 MB/page avg), each profile run through the real finish path and
measured with `pdfinfo` (page count), `file`/`ls` (size), `pdfimages -list`
(embedded image dimensions) and `pdftotext` (searchable-text token capture — a
proxy for how much small print the OCR layer actually caught; higher = more
small text captured). 13-page totals are per-page × 13.

| Profile | Command | 3-page size | Image dim | OCR tokens | 13-page (extrapolated) |
|---------|---------|-------------|-----------|-----------:|------------------------|
| **MRC color 1.5× (DEFAULT)** | *(no flag)* | **723 KB** | 1651×2333 | **592** | **~3.1 MB** |
| MRC grayscale 1.5× (opt-in) | `--grayscale` | 670 KB | 1651×2333 | 546 | ~2.9 MB |
| q70 full-res (opt-out) | `--no-mrc` | 2.33 MB | 2477×3500 | 592 | ~10.1 MB |

- **MRC color 1.5× is the approved default.** It keeps full color and a
  full-resolution-derived text layer (592 tokens — equal to the full-res q70
  reference, because OCR runs on the crisp full-res page *before* the display
  image is downsampled), while landing ~3.2× smaller than q70 and comfortably
  under the ~4–5 MB email ceiling for 10–13 pages. The visible image is sharp
  enough to pass as a document submitted to authorities, and small text is
  searchable.
- **Grayscale caveat (empirical):** switching the same pages to `--grayscale`
  dropped OCR capture from **592 → 546 tokens** and lost a line of small **red**
  print entirely (red collapses toward mid-gray and loses contrast on white).
  So grayscale is smaller (~2.9 MB/13pg) but is **not** safe for documents with
  colored small print — hence color is the default and `--grayscale` is opt-in.
- **Known limit:** on genuinely tiny red micro-print, OCR still drops the odd
  word (e.g. one word of a small red fee notice) even in the color default. That
  is the OCR limit on that text size; use `--mrc --mrc-bg-downsample 1` (sharper,
  larger) or `--full-quality` when such micro-print must be captured verbatim.

All profiles above produced a **searchable** PDF (`pdftotext` non-empty with
recognizable German/English words). The MRC path OCRs the full-resolution page
first, then downsamples only the displayed image with Ghostscript, so the text
layer is never degraded by the size reduction.
