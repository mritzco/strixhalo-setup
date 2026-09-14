[← Setup index](../README.md)

# DeepSeek-OCR — Usage

Turn PDFs and page images into markdown on the iGPU. Setup, tuning and the accuracy
characterisation live in [setup/09-deepseek-ocr.md](../09-deepseek-ocr.md); this page is only
how to drive it day to day. Every number below is measured on this machine.

## The one command

```fish
dspdf book.pdf                 # -> book.md, next to the input
dspdf book.pdf out/notes.md    # explicit destination; directories are created
dspdf -m both -f book.pdf      # both prompt modes, overwrite the existing output
```

`dspdf` is a fish function (`~/.config/fish/functions/dspdf.fish`, contents in
[config-files.md](config-files.md)). Per input it starts the `vllm` toolbox container if it is
stopped, rasterises and OCRs every page inside it, and writes **one markdown file** with pages
joined by `---` under `<!-- page N -->` markers. No `podman` flags to remember.

| Flag | Default | Notes |
|---|---|---|
| `-m/--mode` | `free` | `free` / `markdown` / `grounding` / `both` |
| `-d/--dpi` | `200` | PDF rasterisation DPI; higher helps small print, costs vision tokens |
| `-f/--force` | off | **without it an existing output is refused**, never overwritten |

Exit codes: `0` wrote the file · `1` missing input, output exists, or no output produced ·
`2` bad usage or bad `-m`.

## What it costs

A 58-page scanned book, `-m free --combine`, chunk 16: **127.1 s**
total, of which **56.6 s** is startup before the first page, then **1.2 s/page**. Startup has
been as slow as 80.7 s on this same box, so budget 1–1.5 min before anything appears. `-m both`
roughly doubles the per-page half. Accepted inputs: `.pdf`, plus
`.png .jpg .jpeg .webp .bmp .tif .tiff`.

## Choosing a mode

The two useful modes fail in *complementary* places (full table under **Accuracy** in
[09-deepseek-ocr.md](../09-deepseek-ocr.md)):
`free` gets tables, figures and footers but can lose the first line; `markdown`
(`<|grounding|>Convert the document to markdown`) gets the header but loses the footer.

- **Prose and books → `free`.** What the book above used.
- **When a dropped line is expensive → `-m both`**, which concatenates both readings per page
  under `<!-- mode: X -->` comments so you can compare them by eye.

## ⚠️ Where files can live

The container is handed your **home directory, `/tmp` and `/mnt`** — and little else. A path
outside those resolves to the container's *own* filesystem rather than the host's: `/var/tmp`,
for instance, is the container's empty directory. The run still reports success, so you only
notice when the file is missing on the host. Keep inputs and outputs under `$HOME` and
this never comes up.

## Check the text layer before you OCR a PDF

`--text-layer prefer` skips OCR on pages that already carry embedded text — 10× faster, no GPU —
but PDF text is stored in *draw* order, which is not always reading order, so it is a
cross-check and not ground truth. `dspdf` deliberately does not expose it. Measure first:

```fish
podman exec --user 1000:1000 -e HOME=$HOME vllm bash -lc "python - <<'EOF'
import os, pypdfium2 as pdfium
d = pdfium.PdfDocument(os.path.expanduser('~/Downloads/books/book.pdf'))
n = len(d)
print(sum(1 for i in range(n) if d[i].get_textpage().get_text_range().strip()), 'of', n, 'pages have text')
EOF"
```

The book above: **0 of 58 pages** — scanned, so rasterise + OCR is the only route and
`--text-layer` has nothing to skip. On a born-digital PDF, go around `dspdf` and run
`ocr_batch.py --text-layer prefer` (fast) or `append` (OCR plus the raw text, to cross-check).

## Going around the function

Use `ocr_batch.py` directly for a tree of files — `dspdf` is a thin wrapper over exactly this:

```fish
podman exec --user 1000:1000 -w $HOME/Projects/play/ocr-test \
  -e HOME=$HOME -e PYTHONPATH=$HOME/.local/ocr-libs \
  vllm bash -lc 'python ocr_batch.py ~/Downloads/books -o ~/books-text -m free -r'
```

- `-w` is required: the default workdir `/opt` is not writable by uid 1000.
- `--user 1000:1000 -e HOME=$HOME` keeps the HF cache and any written files owned by you
  instead of root.
- `-r` recurses. Per-page outputs that already exist are skipped unless `-f`, so tree runs are
  resumable; `dspdf` checks its single combined file instead.

Full flag list: **Batch processing** in [09-deepseek-ocr.md](../09-deepseek-ocr.md).

## Output shape

```
<!-- page 1 -->
…text…

---

<!-- page 2 -->
```

A genuinely blank page comes back as `The image is a blank white background with no text or
graphics.` — the model describing what it saw. The last page of the book above is exactly that.

## Two separate runs were byte-identical

The same 58 pages, run twice (once plain, once `-f` into another directory), produced
**byte-identical** output — `cmp` clean, 92,837 B each. The non-reproducibility in
[09-deepseek-ocr.md](../09-deepseek-ocr.md) is a *within-batch* effect (six copies of one image,
one differing by a table dash), not a cross-run one. Still: compare parsed content, not bytes,
if you build anything automatic on top.

## Troubleshooting

| Symptom | What it is |
|---|---|
| `dspdf: output exists: … (use -f to overwrite)` | by design — the function never clobbers |
| `dspdf: no output produced (rc=…)` | the engine died before writing; the real error is further up the log |
| `can only create exec sessions on running containers` | container is stopped and you ran `podman exec` by hand — `podman start vllm`, or just use `dspdf` |
| Tracebacks after the results (`_clear_torch_ops_cache`) | teardown-only noise, fires after the output is written; exit code is still `0` |
| Nothing happens for ~1 min | startup: weights + engine init, 56–81 s |
| `Permission denied` deleting something under `/tmp` | a root-owned file a plain `podman exec` wrote — remove it via `podman exec vllm rm -f …` |
