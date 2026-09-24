# 9. DeepSeek-OCR via vLLM (ROCm / gfx1151)

**Date:** 2026-08-07 · **Status:** ✅ WORKING, with characterised limitations

[← Resilience](08-resilience.md) · [Setup index](README.md) · [Next: Crash forensics →](10-crash-forensics.md)

---

Unblocks the thread deferred since July in [05-local-ai.md](05-local-ai.md): DeepSeek-OCR
can't run on stock llama.cpp (needs unmerged PR #20975). The working route is **vLLM on
ROCm**, via a prebuilt gfx1151 image.

## Usage — start here (2026-09-13)

One command per document. `dspdf` is a fish function (`~/.config/fish/functions/dspdf.fish`,
contents in [reference/config-files.md](reference/config-files.md)) that wraps `ocr_batch.py
--combine` end to end: it starts the `vllm` container if it is stopped, OCRs the file inside
it, and writes a single markdown file. Full day-to-day notes:
[reference/ocr-usage.md](reference/ocr-usage.md).

```fish
dspdf book.pdf                 # -> book.md, next to the input
dspdf book.pdf out/notes.md    # explicit destination
dspdf -m both -f book.pdf      # both prompt modes (they fail in complementary places), overwrite
```

What changed on this date, all measured:

- **Scripts moved: `~/ocr-test` → `~/Projects/play/ocr-test`.** Commands in this chapter that
  passed the old workdir (`-w $HOME/ocr-test`) were stale and failed; fixed inline.
- **First real book — a 58-page scanned book, image-only, `-m free --combine`: 127.1 s
  total** (56.6 s startup, 1.2 s/page, chunk 16) → one 92 KB / 16.6k-word `.md`, 58/58 pages.
  See **Performance** below — startup varies run to run (56.6 s here vs 80.7 s for the 26-page
  manual).
- **That PDF has no usable text layer: 0 of 58 pages, scanned exhaustively rather than sampled.**
  So `--text-layer` has nothing to skip on scanned books; it only pays off on born-digital PDFs.

## Install

```fish
# 1. Pull the image (35.2 GB)
podman pull docker.io/kyuz0/vllm-therock-gfx1151:latest
# If this fails with "unauthorized: incorrect username or password" on a PUBLIC image,
# you have stale Docker Hub tokens — see gotchas. Cleared here on 2026-08-07.

# 2. Create the toolbox — keep-groups, NOT named groups (differs from the ComfyUI toolbox)
toolbox create vllm \
  --image docker.io/kyuz0/vllm-therock-gfx1151:latest \
  -- --device /dev/dri --device /dev/kfd \
  --group-add keep-groups --security-opt seccomp=unconfined

# 3. Model (6.3 GB) — run as YOUR uid so it lands in the shared HF cache
podman start vllm
podman exec --user 1000:1000 -e HOME=$HOME vllm \
  bash -lc 'hf download deepseek-ai/DeepSeek-OCR'
```

Verified inside the container: vLLM `0.22.1rc1`, torch `2.13.0a0+rocm7.14.0a`,
`rocminfo` → gfx1151, `torch.cuda.is_available()` → True.

## Run

Scripts live in `~/Projects/play/ocr-test/`: `ocr_batch.py` (the batch engine every path below
ends up in, and what `dspdf` wraps) plus the test rig `make_test_image.py`, `run_ocr.py`,
`ground_truth.txt`.

```fish
podman exec --user 1000:1000 -w $HOME/Projects/play/ocr-test -e HOME=$HOME \
  -e OCR_MODE=free vllm bash -lc 'python run_ocr.py path/to/page.png'
```

Non-negotiable per vLLM's own recipe — both are quality-critical, not optional:

```python
llm = LLM(model="deepseek-ai/DeepSeek-OCR",
          enable_prefix_caching=False,          # OFF
          mm_processor_cache_gb=0,
          logits_processors=[NGramPerReqLogitsProcessor])   # custom n-gram processor
```

## Batch processing — the engine underneath

`dspdf` wraps exactly this, once per document. Drive `ocr_batch.py` directly when you have a
tree of files: it takes files, directories, or both, and `-r` recurses. One engine, many pages.

```fish
podman exec --user 1000:1000 -w $HOME/Projects/play/ocr-test \
  -e HOME=$HOME -e PYTHONPATH=$HOME/.local/ocr-libs \
  vllm bash -lc 'python ocr_batch.py ~/scans -o ~/scans-text -m free'
```

| Flag | Default | Notes |
|---|---|---|
| `-m/--mode` | `free` | `free`, `markdown`, `grounding`, or `both` (concatenates — see accuracy) |
| `-o/--out` | `ocr-out` | writes `<stem>.md` per input |
| `-r/--recursive` | off | descend into subdirectories |
| `--chunk` | 16 | images per `generate()` call |
| `--gpu-util` | **0.15** | tuned down from 0.55, see below |
| `-f/--force` | off | otherwise **existing outputs are skipped** — runs are resumable |
| `--dpi` | 200 | PDF rasterisation DPI |
| `--combine` | off | one `.md` per PDF instead of one per page |
| `--text-layer` | `off` | `off` / `prefer` / `append` — see below |

### PDFs

Requires `pypdfium2`, installed **into the host home** so it survives container recreation:

```fish
podman exec --user 1000:1000 -e HOME=$HOME vllm \
  bash -lc 'pip install --target $HOME/.local/ocr-libs pypdfium2'
```

Then add `-e PYTHONPATH=$HOME/.local/ocr-libs` to every `podman exec`. Verified: the
venv at `/opt/venv` honours `PYTHONPATH`, so the host copy resolves normally
(pypdfium2 5.12.1, 8.5 MB).

### What survives what

| Operation | Image redownloaded? | Container-local installs | Host-home files |
|---|---|---|---|
| `podman stop` / `start` | no | **kept** | kept |
| `podman rm` + `toolbox create` | **no** — 35.2 GB image stays cached | **lost** | kept |
| `podman rmi` / pull a new tag | yes | lost | kept |

Recreating the toolbox does **not** re-download anything: the image is a cached, read-only
35.2 GB layer, and the container is just a thin writable layer on top of it. Only changes
made *inside* the container (a `pip install` into `/opt/venv`) are discarded.

Always safe, because `$HOME` is bind-mounted: the model in `~/.cache/huggingface`
(6.3 GB), the scripts in `~/Projects/play/ocr-test/`, and `~/.local/ocr-libs`.

PDFs are rasterised page by page, one chunk at a time, so a 500-page document never sits in
RAM at once. Output is `<stem>-p001.md`, `<stem>-p002.md`, … or a single file with `--combine`.

Measured on a real 26-page manual: **138 s total (80.7 s startup, 2.2 s/page)**.

## ⚠️ Rasterise+OCR vs. extracting the PDF text layer

Not obvious, and the intuitive answer is wrong often enough to matter.

| PDF type | Best approach | Why |
|---|---|---|
| Born-digital, simple layout | extract text | exact characters, ~10× faster, zero OCR error |
| Scanned / image-only | rasterise + OCR | no text layer exists |
| Born-digital, complex layout | **OCR, or both** | text layer scrambles reading order |

**Measured counter-example — Ducky OK-M-65 manual, page 6.** The PDF *has* a text layer, so
extraction looked like the obvious win. It wasn't:

| Section | Text layer | OCR | Correct |
|---|---|---|---|
| "2.4 GHz mode" | BT pairing steps (Fn+E/R/T, passkey) | switch to 2.4G, connect dongle | **OCR** |
| "BT mode" | switch to 2.4G, connect dongle | switch to BT, Fn+E/R/T, passkey | **OCR** |

The two sections' bullet lists came out **swapped**, because a PDF's text layer is in *draw*
order, not reading order. Extraction gave perfect characters in the wrong structure; OCR read
the page the way a person would.

The reverse tradeoff also showed up: OCR rendered a mode-selector icon as `■■-`, where
extraction silently dropped it.

**So:** `--text-layer prefer` is a real 10× speedup (26 pages in 14 s, no GPU at all) but
inherits the reading-order bug. Use `--text-layer append` when correctness matters — you get
the OCR reading plus the raw text layer in a fenced block, and can cross-check.

## Performance (measured, this box)

| Phase | Time |
|---|---|
| Weight load (6.21 GiB, btrfs) | ~3.9 s |
| Engine startup, total | **~81 s** |
| Generation — **single image** | ~13 s |
| Generation — **batched** | **2.9 s/image** |
| 58-page PDF, chunk 16, `--combine` (2026-09-13) | **127.1 s total** — 56.6 s startup, **1.2 s/page** |

**Batching is a 4.5× win per image** — startup is paid once and vLLM overlaps the work.
Measured: 6 images in 98.2 s (80.8 s startup + 17.4 s generation).

Per-page cost keeps falling as the chunk grows: 1.2 s/page at chunk 16 (the book) vs 2.9 s/image
for a 6-image run. **Startup is not fixed either** — 56.6 s for the book run against 80.7–80.8 s
for the two runs above, same box, same image. Budget ~1–1.5 min before the first page appears.

Extrapolated to 100 images: `81 + 100 × 2.9 ≈` **6.5 minutes**. One invocation per file
would be ~2.8 hours; the whole difference is amortised startup.

## Tuning — KV cache (applied 2026-08-07)

`gpu_memory_utilization` **0.55 → 0.15**:

| | 0.55 | 0.15 |
|---|---|---|
| KV cache | 51.11 GiB | **7.11 GiB** |
| KV tokens | 893,280 | 124,336 |
| Max concurrency @ 8192 | 109.04x | 15.18x |

**~44 GiB reclaimed, no measurable cost** — 15x concurrency is far beyond what single-user
document OCR uses. The model itself needs only 6.23 GiB.

## ⚠️ Output is not bit-reproducible

Six copies of an *identical* image in one batch gave five byte-identical outputs and one that
differed — by a single dash in a markdown table separator, plus a trailing newline. **No data
differed**: every figure, word and structure matched.

`temperature=0.0` does not guarantee identical bytes in a batched engine; batch position
changes reduction order. Don't build a pipeline that byte-diffs OCR output to detect change —
compare parsed content instead.

## ⚠️ Accuracy — no single prompt mode captured the whole document

Tested on a synthetic invoice with known ground truth (`~/Projects/play/ocr-test/ground_truth.txt`):

| Content | `Free OCR.` | `<\|grounding\|>Convert the document to markdown.` |
|---|---|---|
| `INVOICE #2026-0817` (first line) | ❌ | ❌ |
| Company name | ❌ | ✅ |
| Street address | ❌ | ✅ |
| Table rows, all 9 figures | ✅ as markdown tables | ✅ as text + bboxes |
| Subtotal / VAT / TOTAL | ✅ | ✅ |
| Footer + `Ref: PX13-STRIX-2026` | ✅ | ❌ |

- **Numeric accuracy was perfect in both modes** — every figure, including `1,001.52` with
  its comma and `65.52` as 7% VAT. That's the hard part, and it works.
- **`Free OCR.` reconstructs table structure** from plain ASCII columns — real layout
  understanding, not just character recognition.
- **Punctuation degrades:** `Payment terms:` → `Payment terms`, `Ref:` → `Ref.`
- **The first line is missed by BOTH modes.**

**Tested and ruled out — top margin.** First hypothesis was that the title sat too close to
the edge (28 px). Regenerated at 90 px margin, re-ran: **still missing.** Not a margin issue.

**Untested next lever: resolution mode.** DeepSeek-OCR has Tiny/Small/Base/Large/Gundam
modes (`base_size`, `image_size`, `crop_mode`). The default may downsample a 1000px-wide
page enough to lose a single sparse title line. Try a larger mode before concluding it's a
model limitation.

**Practical takeaway:** for a document where you need everything, run **both modes and
merge** — they fail in complementary places. For tables and numbers alone, `Free OCR.` is
sufficient and excellent.

## Tuning

- **KV cache is wildly oversized by default.** At `gpu_memory_utilization=0.55` vLLM
  allocated **51.11 GiB / 893,280 tokens** ("max concurrency 109x") for single-image OCR.
  Drop to ~0.15 and reclaim ~40 GiB. `run_ocr.py` reads `OCR_GPU_UTIL`.
- Two ROCm perf fallbacks fire, both benign but costing speed:
  - `Cannot use ROCm custom paged attention kernel, falling back to Triton`
  - `Using default MoE config. Performance might be sub-optimal!` — the config filename is
    built from the device name, which arrives as a **`<MagicMock ...amdsmi_get_gpu_asic_info...>`**
    because amdsmi is mocked in this image. So no tuned MoE config can ever be found.
    DeepSeek-OCR is itself an MoE (E=64, N=896).
- Triton JIT-compiles some kernels *during* inference (`_fwd_kernel`, `fused_moe_kernel`),
  causing latency spikes on first use of a shape.

## ⚠️ Gotchas

- **Public pulls fail with `unauthorized: incorrect username or password`.** Cause:
  `~/.docker/config.json` holds **stale Docker Hub tokens**; podman sends them and Hub
  rejects the auth instead of falling back to anonymous. Workaround: `--authfile` pointing
  at a file containing `{}`. Permanent fix: `podman logout docker.io`, or re-`docker login`.
- **`podman exec` runs as root with `HOME=/root`.** The host home *is* bind-mounted at its
  real path, but a plain `exec` downloads models into container storage instead of the
  shared cache, and any files it writes to your home are root-owned. Always pass
  `--user 1000:1000 -e HOME=$HOME`.
- **Default workdir is `/opt`, which uid 1000 cannot write.** Always pass `-w`.
- **Harmless noise at exit:** `ValueError: too many values to unpack` and `UnicodeDecodeError`
  from `torch/library.py:_clear_torch_ops_cache`. These are teardown-only, fire *after* results
  are printed, and do not affect output.
- `--group-add keep-groups` is required here — named groups (`--group-add render`) do not
  work, unlike the ComfyUI toolbox in [06-comfyui-rocm.md](06-comfyui-rocm.md).
- **The container is handed the host's `$HOME`, `/tmp` and `/mnt` — and little else.** A path
  outside those (`/var/tmp`, `/srv`, `/run`) resolves to the container's *own* filesystem, not
  the host's: the run reports success, and the file is nowhere on the host. Keep inputs and
  outputs under `$HOME`. Related: a file `/tmp` written by a plain `podman exec` is
  **root-owned** on the host (`rm: Permission denied`) — remove it with `podman exec vllm rm -f`.
