# scan-batch

Group an **Epson ET-3700** flatbed scanner's one-file-per-press scans into a
single, OCR'd, **searchable PDF** — driven entirely from the terminal.

The ET-3700 is flatbed-only. Each panel press of **Scan → Computer → Paperless**
produces exactly one file; the device has no concept of a multi-page "session".
`scan-batch` adds that session grouping on the host: it starts the stock
[`epson2paperless`](https://github.com/mtheuma/epson2paperless) daemon, watches
its output folder as you scan page after page, and when you press **ENTER** it
merges the pages (in scan order) into one PDF and runs OCR so the text is
selectable and searchable.

## Relationship to epson2paperless (and why this is separate)

This wrapper is a **thin host-side layer that is intentionally SEPARATE from and
does not modify** the `epson2paperless` repo. It only uses that project's public
surface: the `PRINTER_IP` / `OUTPUT_DIR` environment variables, `npm run dev`,
and its stdout log line `epson2paperless ready …`. `epson2paperless` deliberately
keeps OCR and page-merging logic out of its scope, so that work lives here. This
layer becomes **optional** once your document stack (e.g. Paperless-ngx) performs
OCR server-side — at that point you can drop the OCR/merge step and just let the
daemon deposit pages.

## What it does, step by step

1. **Startup** — quits the Epson helper apps that would otherwise hold the scan
   port (see caveat below), starts the daemon pointed at a hidden staging folder,
   and waits for it to report ready.
2. **During the batch** — watches the staging folder. Each time a new page lands
   it prints `Page N captured (<file>)`. Keep scanning as many pages as you like.
3. **Finish** — press **ENTER**. You're prompted for a document name (ENTER for a
   timestamp default). The pages are assembled into one multi-page PDF and OCR'd
   with `ocrmypdf -l deu+eng`. The finished PDF lands in your output folder.
4. **Cleanup** — always runs: the daemon is stopped (no orphan bound to the scan
   port) and the raw per-page scans are deleted. Finished PDFs are never touched.

## Prerequisites

- **epson2paperless, run from source.** Default location
  `~/Dev/epson2paperless` (override with `--e2p`). It is
  launched via `npm run dev` (needs Node + its `node_modules` present). The
  wrapper sets `PRINTER_IP` and `OUTPUT_DIR` for it; it does not modify the repo.
- **ocrmypdf** (`brew install ocrmypdf`) — mandatory. Makes the PDF searchable.
- **tesseract with the `deu` and `eng` language packs**
  (`brew install tesseract tesseract-lang`). The script fails fast if a requested
  language is missing.
- **An image→PDF assembler**: `img2pdf` (preferred — lossless, no re-encode) or
  **ImageMagick** (`magick`) as a fallback. Install the preferred one with
  `brew install img2pdf`. The script auto-detects which is present.
- Optional: `pdfinfo` / `pdftotext` (from `poppler`) are used by the self-test to
  verify output; `sips` is a last-resort test-image generator.

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
# scan page 1 as JPEG from the panel (Scan → Computer → Paperless → Save as JPEG)
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
| `--self-test` | Run the offline self-test (no daemon, no scanner) and exit | — |
| `--no-daemon` | Skip helper-app quit + daemon startup (file-based mode) | — |
| `-h`, `--help` | Show help and exit | — |

`PRINTER_IP`, `E2P_DIR`, `OUT_DIR`, and `OCR_LANGS` may also be set via the
environment; flags take precedence. A leading `~` in a path argument is expanded
to your home directory.

## How grouping and finish work

- Grouping is **host-side**. The scanner has no sessions: one panel press = one
  page = one file in the staging folder.
- `epson2paperless` writes `scan_YYYY-MM-DD_HHMMSS.jpg`. The filename timestamp
  order equals the scan order equals the merge order, so pages always end up in
  the order you scanned them. The watcher only accepts a page once its file size
  is stable across two polls, so a half-written file is never grabbed.
- **ENTER** ends the batch. Zero pages scanned → the script says so and exits
  without creating a PDF.
- The document name is sanitized (path separators stripped; empty-after-trim
  falls back to the timestamp default; `.pdf` appended if missing). If the name
  already exists in the output folder it is **not overwritten** — a numeric
  suffix (`_1`, `_2`, …) is appended instead.

## Cleanup behavior

- Raw per-page scans live in a **hidden** `.work/` staging folder inside the
  output folder (configurable via `--staging`).
- On **success**: staging raw pages, the merged temp PDF, and `daemon.log` are
  deleted; the empty staging folder is removed. The finished PDF is kept.
- On **failure** (e.g. OCR error): staging is **kept** so your raw pages are
  recoverable, and `daemon.log` is kept for diagnosis. The script prints exactly
  where the pages are.
- The script **never** deletes anything in the output-folder root except its own
  temp files, and **never** touches finished PDFs.
- Cleanup runs on normal exit, on **Ctrl-C** (abort mid-batch → daemon stopped,
  staging cleaned, non-zero exit), and on TERM.

## Known failure modes and fixes

- **Port 2968 already in use / daemon won't start.** Usually an Epson helper app
  or a stale daemon. The script detects this before starting anything and prints:
  ```
  Find it:   lsof -nP -iTCP:2968 -sTCP:LISTEN
  Kill it:   kill $(lsof -nP -tiTCP:2968 -sTCP:LISTEN)
  ```
  Quit ScanSmart / Scanner Monitor / Event Manager, or kill the stale process,
  then re-run.
- **Daemon never reports ready** (timeout ~30s). Check the printer IP
  (`--printer-ip`) and the tail of `<staging>/daemon.log` that the script prints.
  Confirm the printer is on and reachable.
- **ocrmypdf fails.** The run fails loudly, the raw pages are **kept** in staging
  (path printed), and `daemon.log` is kept. Fix the cause (often a corrupt page
  or a Ghostscript hiccup) and you can re-assemble manually from the kept pages.
- **"Communication problem — ensure a computer is attached" on the printer.**
  The daemon isn't running/ready when you pressed Scan. Start `scan-batch` first
  and wait for the Ready prompt; on the panel use *Scan → Computer → Paperless*
  (the "Preview on Computer" entry is not used by this flow).

## Compatibility note

Written for macOS **system bash 3.2** (`/bin/bash`). It avoids bash-4-only
features (no associative arrays, no `wait -n`, no `${var,,}`, no `mapfile`). In
particular, note that under bash 3.2 a `read -t` timeout and EOF both return
status 1, so the watch loop treats any non-zero read as "keep watching" and only
finishes on a successful read (ENTER).

## Test run (self-test)

The script ships with an offline, scanner-free self-test:
`./scan-batch --self-test`. It skips the daemon entirely, generates three test
JPEGs with real text, drives the merge → OCR → cleanup path, and asserts the
results. It also exercises the success-cleanup, the OCR-failure path (which must
retain raw pages), and the port detector.

Run on this machine (macOS, bash 3.2.57, ocrmypdf 16.x, tesseract 5.x with
deu+eng, img2pdf 0.6.x):

```
=== scan-batch self-test ===
  OK: page count == 3
  OK: text layer present (searchable)
  OK: staging cleaned, daemon.log removed
  OK: finished PDF untouched by cleanup
  OK: failure path left raw page in staging (<tmp>/out/.work)
note: port 2968 free (detector works).
  OK: self-test started no daemon (no orphan possible).
=== SELF-TEST PASS ===
```

Exit code `0`. In addition to the self-test, the following were verified manually
in `--no-daemon` mode (file-based, no scanner):

- **Multi-page merge + searchable OCR**: two text JPEGs → one 2-page PDF;
  `pdfinfo` reports `Pages: 2`; `pdftotext` recovers the page text (`PageOne`,
  `PageTwo`) → searchable layer confirmed.
- **Name sanitize**: input `My/Weird:Doc` → output file `MyWeird:Doc.pdf` (path
  separator stripped).
- **No-overwrite collision**: scanning again with an existing name produced
  `watchdoc_1.pdf` rather than overwriting `watchdoc.pdf`.
- **Zero pages**: pressing ENTER with no pages prints "No pages scanned — nothing
  to do.", creates no PDF, and removes the staging folder.
- **Port-in-use**: with TCP 2968 occupied, the script printed the helpful
  find/kill guidance and exited non-zero (code 2) with no daemon started.
- **Cleanup**: staging folder removed on success; no leftover `scan-batch`
  process; nothing bound to port 2968 afterwards.

### shellcheck

`shellcheck` is **not installed** on this machine (`/opt/homebrew/bin/shellcheck`
absent), so it was skipped per plan (not installed). Static checking was done with
`bash -n scan-batch` (syntax OK) plus the behavioral self-test and manual runs
above. If you install shellcheck (`brew install shellcheck`), run
`shellcheck scan-batch` and address any warnings.
