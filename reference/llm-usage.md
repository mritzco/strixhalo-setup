[← Setup index](../README.md)

# Local LLMs — Usage

Quick reference for running local text models on the PX13 (Strix Halo, Radeon 8060S, Vulkan).
Setup details live in [setup/05-local-ai.md](../05-local-ai.md). **Always use `127.0.0.1`, never `localhost`.**

## Two ways to run

### A. llama-swap — everyday driver (recommended)
One endpoint, all models, auto-loads/swaps on request.
```fish
llmswap        # serves http://127.0.0.1:1234/v1  (Ctrl-C to stop)
```
- Pick the model **in your client** — llama-swap loads it on demand and swaps as needed.
- Built-in playground + model list: open `http://127.0.0.1:1234` in a browser.
- Models: `qwen3-coder` (coding/agents), `qwen3-instruct` (general), `qwen3-vl` (vision), `glm-4.5-air` (coder).

### B. `llm` — manual single model (fallback, port 8080)
```fish
llm coder            # one model on http://127.0.0.1:8080
llm instruct 8081    # a second one on another port
llm                  # prints the menu
```
Keys: `coder`, `instruct`, `qwenvl`, `glm`. Don't run `llm` and `llmswap` on the same port.

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
- **Agents (omp/pi):** use non-thinking models (`qwen3-coder`/`instruct`). Thinking models (GLM) burn the token budget in tool loops.
- Quant guide: `Q4_K_M`/`UD-Q4_K_XL` default; bump to `Q5`/`Q6` for models under ~60 GB (plenty of RAM).
- DeepSeek: use the cloud subscription, not local.
