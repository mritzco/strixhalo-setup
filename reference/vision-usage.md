[← Setup index](../README.md)

# Vision (image → answer) — Usage

Ask questions about an image on the iGPU. Text-only models: [llm-usage.md](llm-usage.md).
Pulling the *text* out of a page or PDF is a different job: [ocr-usage.md](ocr-usage.md).
Setup, model lineage and the fork notes: [05-local-ai.md](../05-local-ai.md).

## One image, one question

```fish
llmv image.jpg                                       # default question: "Is this image AI generated?"
llmv image.jpg "What is the total on this invoice?"
llmv --model 3 image.jpg "Is the signature real?"    # qwen3-vl (fast MoE) instead of the default 3.8
```

Nothing needs to be running first: if port 1234 is down, `llmv` starts llama-swap itself,
detached. The answer goes to stdout, prefixed with the model id.

| Flag | Default | Model | Notes |
|---|---|---|---|
| `--model 3` | | `qwen3-vl` | 30B-A3B MoE, ~18 G — fast, the right default for questions |
| `--model 3.8` | ✅ | `qwen3.8-27b-vl` | dense 27B Q8, ~29 G — finer detail, ~10× slower per image |

With **no image argument** `llmv` drops into interactive chat (`llama-mtmd-cli`).

## Why the second image is fast

The model lives in a `llama-server` process that llama-swap keeps warm for its `ttl` (30 min).
Measured on a 1-page invoice scan, "cold" meaning nothing was running at all:

| Path | 1st call | 2nd and later |
|---|---|---|
| `llmv --model 3` (MoE ~18 G) | **7.8 s** (including starting llama-swap) | **2.1 s** |
| `llmv` (default 3.8, dense ~29 G) | 29.3 s | 23.0 s |
| `llama-mtmd-cli` one-shot, no daemon at all | 10.0 s | ~10 s **every time** |

- **The daemon costs nothing now.** With no startup preload, the first `llmv` call is *faster*
  than the no-daemon path (7.8 s vs 10.0 s), and every later one is ~4× faster.
- **The default `3.8` stays slow even when warm (23 s).** That is dense-model image prefill, not
  loading, so llama-swap cannot fix it. Use `--model 3` for quick questions; keep `3.8` for when
  you want the better reading and can wait.

## Which models are resident

```fish
llmswap -d                 # start the endpoint, load NOTHING (the default)
llmswap -d -p qwen3-coder  # start and warm one model in the background
llmswap -p none            # explicit "nothing" (same as the default)
llmswap                    # foreground; -p works here too
```

The default is one line at the top of `~/.config/fish/functions/llmswap.fish`:
`set -l default_preload ''` — set it to a model id to always warm that one (contents in
[config-files.md](config-files.md)). Startup preload in
[`~/.config/llama-swap/config.yaml`](config-files.md) is `preload: []` on purpose: it applies to
every start and would override the flag.

Warming uses `/upstream/<model>/health`, which starts the model and runs **no inference** (a bad
model id is reported instead of silently doing nothing).

Watching and freeing:

```fish
curl -s http://127.0.0.1:1234/running                  # what's loaded, and its state
curl -X POST http://127.0.0.1:1234/api/models/unload   # free every model now
pkill -f 'llama-swap --config'                         # stop the proxy as well
```

## ⚠️ Your own clients pull models in

`preload: []` makes llama-swap start empty, but it does not stop clients from loading things: with
the endpoint up, **omp requests `qwen3-coder` within seconds** — llama-swap logs
`POST /v1/chat/completions ... "omp/18.1.14"` and the load takes 5.8 s. On battery, wanting
"nothing loaded" therefore means quitting the client (or pointing it at DeepSeek), not just
tidying llama-swap.

## ⚠️ Chat mode loads a second copy

`llmv` with no image argument runs `llama-mtmd-cli` directly, which loads its **own** copy of the
model. If llama-swap already holds that model you get two residents on one GPU and both crawl —
measured before at TG 60 → 2 t/s. Unload first:
`curl -X POST http://127.0.0.1:1234/api/models/unload`.

## When it is really a document

`llmv` answers questions about a page. To get the *text* of a document, transcribe instead:
DeepSeek-OCR is better at that and writes a `.md` — [ocr-usage.md](ocr-usage.md).
