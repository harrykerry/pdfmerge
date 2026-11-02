# pdfmerge

Merge PDFs in natural numeric order and compress the result enough to email
directly — no cloud upload, no Drive link.

```console
$ pdfmerge ~/Documents/report -x draft
Merging 5 file(s) in this order:

  1.pdf                                99 pages   -> 1-99
  3.pdf                               113 pages   -> 100-212
  4.pdf                                 1 pages   -> 213
  5.pdf                                 2 pages   -> 214-215
  6.pdf                                 1 pages   -> 216

  total                               216 pages   3.57 MB

Merging and compressing (quality: ebook)...

Wrote /home/you/Documents/report/merged.pdf
  pages:   216 (verified)
  size:    3.57 MB -> 1.91 MB  (47% smaller)
  quality: ebook
```

## Why

Most merge tools either sort `10.pdf` before `2.pdf`, or hand you a file too
big to attach. `pdfmerge` does both halves of the job in one pass and checks
its own work: if a single page goes missing, the output is thrown away rather
than handed to you.

- **Numeric ordering** — `2.pdf` before `10.pdf`, as a human would.
- **Email-ready output** — typically 40–50% smaller; `--max-size` targets a cap.
- **Verified** — page counts are compared before and after; mismatch aborts.
- **No pip install** — Python 3 standard library plus Ghostscript. Nothing else.
- **Non-destructive** — inputs are never modified, content is never re-flowed.

## Requirements

| | |
|---|---|
| Python | 3.8 or newer (standard library only) |
| Ghostscript | 9.50 or newer (`gs` on PATH) |

Works on Linux, macOS and WSL. On Windows the script looks for `gswin64c`, but
that path is only lightly tested.

## Install

```bash
# Debian / Ubuntu
sudo apt install -y ghostscript python3

# Fedora / RHEL / Rocky
sudo dnf install -y ghostscript python3

# Alpine
apk add --no-cache ghostscript python3

# macOS
brew install ghostscript
```

Then drop the script somewhere on your PATH:

```bash
git clone https://github.com/haroldkerry/pdfmerge.git
cd pdfmerge
install -Dm755 pdfmerge ~/.local/bin/pdfmerge     # or /usr/local/bin for everyone
pdfmerge --help
```

If `~/.local/bin` isn't on your PATH:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && . ~/.bashrc
```

There is nothing to build and no dependency to vendor — the script is a single
self-contained file you can also just copy into a project.

## Usage

```
pdfmerge [options] [files or directory]
```

With no arguments it merges every PDF in the current directory. Given a
directory it merges that one, writing `merged.pdf` **into** it. Given explicit
files it uses exactly that order.

| Option | Effect |
|---|---|
| `-n`, `--dry-run` | Print the merge order and page ranges, write nothing |
| `-o FILE` | Output path (default `merged.pdf` in the merged directory) |
| `-x NAME` | Skip a file — `-x 2` or `-x draft.pdf`; repeatable |
| `-q PRESET` | `screen`, `ebook` (default), `printer`, `prepress` |
| `--max-size MB` | Try presets best→worst, stop at the first that fits |
| `--no-compress` | Merge only, skip re-encoding |
| `-f`, `--force` | Overwrite an existing output file |
| `--include-outputs` | Don't skip previous pdfmerge outputs when scanning |
| `-h`, `--help` | Full help |

### Quality presets

These map to Ghostscript's `-dPDFSETTINGS`. They affect **images only** — text
and vectors stay sharp at every level.

| Preset | Images | Use for |
|---|---|---|
| `screen` | 72 dpi | Mostly text; smallest output |
| `ebook` | 150 dpi | **Default.** Scans stay readable |
| `printer` | 300 dpi | Intended for printing |
| `prepress` | 300 dpi, colour preserved | Largest; commercial print |

### Examples

```bash
pdfmerge -n                           # always worth doing first
pdfmerge                              # current dir -> merged.pdf
pdfmerge ~/scans/batch7               # -> ~/scans/batch7/merged.pdf
pdfmerge -x 2 -o thesis.pdf           # skip 2.pdf, name the output
pdfmerge --max-size 10                # auto-fit under 10 MB
pdfmerge -q printer -o print.pdf      # higher image quality
pdfmerge 6.pdf 5.pdf 1.pdf -o cover.pdf
pdfmerge --no-compress -o exact.pdf   # no re-encoding
```

## How it works

Ghostscript's `pdfwrite` device reads every input in sequence and writes one
document, re-encoding embedded images to the chosen resolution and subsetting
fonts along the way. The script wraps that with the parts Ghostscript doesn't
do for you: ordering, page accounting, size targeting and safety checks.

Output is written to a temporary directory and moved into place only after the
page count is verified, so an interrupted or failed run never leaves a
truncated file behind.

## Behaviour worth knowing

**Run `-n` first.** Scanning a directory picks up *every* PDF in it. The dry
run shows precisely what will be merged, in what order, and where it will go.

**Ordering.** Names are split into text and number runs and compared
piecewise, so `Scan_1, scan_02, scan_3, scan_10` sorts in that order
regardless of leading zeros or case. Pass files explicitly to override.

**Outputs are tagged.** Every output carries a `pdfmerge-output` marker in its
XMP metadata, so re-running in the same folder won't fold a previous merge back
into the new one and silently double the document. Skipped files are reported.
Use `--include-outputs` to merge them anyway.

**Compression is lossy for images.** Text and vector content are unaffected,
but a photo or scan is resampled. Use `--no-compress`, or `-q prepress`, when
fidelity matters more than size.

**Two Ghostscript side effects**, both from the re-encode:

- Pages carrying a `/Rotate` flag have the rotation baked into the page itself.
  They display identically; only the internal representation changes.
- A ligature such as `fi` can occasionally lose its Unicode mapping. The glyph
  still renders correctly, but a text search for that one word may not match.

`--no-compress` avoids both.

**Not handled:** encrypted or password-protected PDFs (decrypt them first),
bookmarks and form fields (Ghostscript drops some), and digital signatures,
which any re-encode necessarily invalidates.

## Exit codes

| Code | Meaning |
|---|---|
| `0` | Success |
| `1` | Ghostscript missing, no PDFs found, output exists without `-f`, or page-count mismatch |
| `130` | Interrupted |

Suitable for scripting:

```bash
pdfmerge /srv/incoming -o /srv/out/batch.pdf --max-size 20 -f \
  || { echo "merge failed" >&2; exit 1; }
```

## Server use

### Docker

```dockerfile
FROM python:3.12-slim
RUN apt-get update \
 && apt-get install -y --no-install-recommends ghostscript \
 && rm -rf /var/lib/apt/lists/*
COPY pdfmerge /usr/local/bin/pdfmerge
RUN chmod +x /usr/local/bin/pdfmerge
```

```bash
docker run --rm -v "$PWD:/work" -w /work myimage pdfmerge -o out.pdf
```

### Batch and cron

```bash
# Each subdirectory becomes one merged PDF
for d in /srv/incoming/*/; do
    pdfmerge "$d" -o "/srv/out/$(basename "$d").pdf" --max-size 20 -f \
      || echo "FAILED: $d" >> /var/log/pdfmerge.err
done
```

Cron's PATH is minimal, so use an absolute path:

```cron
0 2 * * * /usr/local/bin/pdfmerge /srv/incoming -o /srv/out/nightly.pdf -f >> /var/log/pdfmerge.log 2>&1
```

### Security

The script invokes Ghostscript with `-dSAFER`, which blocks file and command
access from inside the PDF.

**If you process untrusted uploads, do not rely on `-dSAFER` alone.**
Ghostscript has a history of sandbox-escape CVEs. Keep it patched and add real
isolation — a dedicated unprivileged user, no network, a container, and
resource caps, since a malformed PDF can make Ghostscript spin:

```bash
ulimit -t 120 -v 2000000        # 120s CPU, ~2 GB address space
timeout 300 pdfmerge /srv/incoming -o /srv/out.pdf
```

Ghostscript is CPU-bound and effectively single-threaded per job; parallelise
across jobs rather than within one.

## Attachment size limits

| Service | Cap |
|---|---|
| Gmail | 25 MB |
| Outlook / Microsoft 365 | 20 MB (often 10 MB on corporate tenants) |
| Yahoo Mail | 25 MB |

Base64 encoding adds roughly 33%, so `--max-size 15` is a safe target for
Gmail. The script warns when output exceeds 25 MB.

## Contributing

Issues and pull requests are welcome. The project is deliberately a single
dependency-free file — please keep it that way, and keep `pdfmerge --help`
accurate for any flag you add.

## Licence

MIT — see [LICENSE](LICENSE).

Ghostscript is a separate program under its own licence (AGPL, or a commercial
licence from Artifex). `pdfmerge` only invokes it as a subprocess and does not
bundle or link against it.
