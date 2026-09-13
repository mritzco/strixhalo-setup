[← Setup index](../README.md)

# Local LLMs — Usage

Quick reference for running local text models on the PX13 (Strix Halo, Radeon 8060S, Vulkan).
Setup details live in [setup/05-local-ai.md](../05-local-ai.md). **Always use `127.0.0.1`, never `localhost`.**

## Three ways to run

### A. llama-swap — everyday driver (recommended)
One endpoint, all models, auto-loads/swaps on request. **It now starts with nothing loaded.**
```fish
llmswap                    # foreground: serves http://127.0.0.1:1234/v1 (Ctrl-C to stop)
llmswap -d                 # detached: own session, returns once it answers, logs to
                           # ~/.cache/llama-swap.log. Survives closing the terminal.
llmswap -p qwen3-coder     # ...and warm one model in the background (-p none = load nothing)
llmswap -d -p qwen3-vl     # combine freely
```
- Pick the model **in your client** — llama-swap loads it on demand and swaps as needed.
- **You usually don't start it at all:** `llmv` runs `llmswap -d` itself if 1234 is down
  (probes `/api/version`, which a plain `llama-server` doesn't serve).
- `-p/--preload` defaults to nothing — best on battery. The default is one line at the top of
  `~/.config/fish/functions/llmswap.fish` ([contents](config-files.md)); startup preload in
  `config.yaml` is deliberately `preload: []` so it can't override the flag.
- Built-in playground + model list: open `http://127.0.0.1:1234` in a browser.
- Models: `qwen3-coder` (coding/agents), `qwen3-vl` (fast vision), `qwen3.8-27b-vl` (dense
  vision), `qwen3.8-flash-next` (strongest, thinking), `qwen3.8-flash-uncensored` (MTP fork).
  The file is the truth: `~/.config/llama-swap/config.yaml` ([contents](config-files.md)).
- `/running` lists what's loaded · `POST /api/models/unload` frees the model(s) now · the
  config's `ttl: 1800` unloads them after 30 min idle anyway.
- Editing `config.yaml`: reload in place with `pkill -HUP -f 'llama-swap --config'` (llama-swap also
  takes `-watch-config`: polls the file every 2 s and reloads on change — `llmswap` does not pass
  it). ⚠️ Either way the reload **rebuilds the server and evicts the resident model**, so the next
  request pays the load again. `llmswap -d` against a live instance only says "already running".

### B. `llm` — manual single model (fallback, port 8080)
```fish
llm coder            # one model on http://127.0.0.1:8080
llm instruct 8081    # a second one on another port
llm                  # prints the menu
```
Keys: `coder`, `instruct`, `qwenvl|vl`, `flash`, `qwen-vl|qvl` (run `llm` for the menu + expected
t/s). Don't run `llm` and `llmswap` on the same port.

### C. `llmv` — one image + a question, no setup

```fish
llmv image.jpg "What is the total on this invoice?"
```

Starts llama-swap on demand (detached) if needed, then answers: **7.8 s** on a cold machine,
**2.1 s** warm. Model choice (`--model 3|3.8`), the resident-model knobs (`llmswap -p`), and the
two-residents gotcha: **[vision-usage.md](vision-usage.md)**.

## Clients
- **omp** → its **lm-studio** provider auto-finds llama-swap at `127.0.0.1:1234`. Models show under "lm-studio".
- **pi** → `/model` picker → the `local` provider (`~/.pi/agent/models.json` points at 1234).
- **Open WebUI / any OpenAI app** → base URL `http://127.0.0.1:1234/v1`, any API key.
- Endpoint speaks the **OpenAI API** (`/v1/chat/completions`, `/v1/models`).

## Getting models
Downloads cache to `~/.cache/huggingface/hub` (llama.cpp `-hf` reuses them).
```fish
hf download <user/repo> --include "*<QUANT>*"
# e.g. hf download unsloth/GLM-4.5-Air-GGUF --include "*UD-Q4_K_XL*"
```
To make a new model available in llama-swap: add a block to `~/.config/llama-swap/config.yaml`
(copy an existing one — keep `proxy: http://127.0.0.1:${PORT}`), then restart `llmswap`.

## Cleaning up models
```fish
hf cache ls                              # list cached repos + sizes
hf cache rm model/<user>/<repo>          # remove one
hf cache prune                           # remove half-finished downloads
```

## Monitor / tips
- GPU load: `amdgpu_top`
- **Agents (omp/pi):** use non-thinking models (`qwen3-coder`). Thinking models (GLM) burn the token budget in tool loops.
- Quant guide: `Q4_K_M`/`UD-Q4_K_XL` default; bump to `Q5`/`Q6` for models under ~60 GB (plenty of RAM).
- DeepSeek: use the cloud subscription, not local.
