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
  vision), `qwen3.8-flash-next` (strongest, thinking, `--reasoning-budget 2048`),
  `qwen3.8-flash-uncensored` (MTP fork on the custom Vulkan build, budget 2048 since 2026-09-24).
  The file is the truth: `~/.config/llama-swap/config.yaml` ([contents](config-files.md)).
- `/running` lists what's loaded · `POST /api/models/unload` frees the model(s) now · the
  config's `ttl: 1800` unloads them after 30 min idle anyway.
- Editing `config.yaml`: reload in place with `pkill -HUP -f 'llama-swap --config'` (llama-swap also
  takes `-watch-config`: polls the file every 2 s and reloads on change — `llmswap` does not pass
  it). ⚠️ Either way the reload **rebuilds the server and evicts the resident model**, so the next
  request pays the load again. `llmswap -d` against a live instance only says "already running".

#### ⚠️ Two traps we hit (2026-09-24)

**"The model stopped mid-thinking"** — that is a *client* output cap, not a crash. The Playground UI
(and most OpenAI clients) default to `max_tokens: 4096`; a thinking model with **no reasoning budget**
burns all 4096 inside the reasoning block and the stream ends before the answer starts. Proof from the
backend log: `eval time = 113967 ms / 4096 tokens`, `truncated = 0` — it stopped at exactly 4096, the
context was nowhere near full, and the process stayed up (it exited later on the TTL unload).

Do both fixes:

```yaml
# ~/.config/llama-swap/config.yaml — bound the thinking so the answer always has room
# (llama.cpp default is --reasoning-budget -1 = unlimited, which is the trap)
    cmd: ... --reasoning-effort medium --reasoning-preserve --reasoning-budget 2048 ...
```
```fish
pkill -HUP -f 'llama-swap --config'      # reload in place; the next request relaunches the backend
```
- **Also raise `max_tokens` in the client** (Playground UI setting; omp/python clients). 8192+ is
  comfortable with `-c 131072`. Capping thinking alone fixes it; doing both gives the model room to
  think *and* answer.

`qwen3.8-flash-next` already carried `--reasoning-budget 2048`, which is exactly why it "answered
fully" on the same conversation while `qwen3.8-flash-uncensored` (no budget) cut off mid-thought.

**A long generation can masquerade as a server error.** A client timeout shorter than the generation
(observed: a Python client gave up at 2m25s → `502` + `http: proxy error: context canceled`) is a
*client-side* abort. `dial tcp 127.0.0.1:5800: connection refused` is llama-swap proxying to a backend
that is unloaded/starting (TTL eviction) — noise, not a fault.

**Resident models add up.** There is no `groups:` block, so every requested model stays loaded until
its `ttl: 1800` expires — five models × 15–70 GB on a 128 GB box. Under real pressure the new
`earlyoom` policy kills the **largest RSS** process, which will be a resident model (see
[ch. 8](../08-resilience.md)). A `groups:` section can make swapping deterministic — check
llama-swap's docs for `exclusive` vs `swap` semantics before applying; they differ in whether models
*outside* the group are stopped too.


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

## Which local model for what (measured 2026-09-24)

Objective comparison — 20 hidden spec-compliance cases, a 4-bug hunt, a runnable `/proc` script —
plus sequential throughput. Harnesses: `~/model-eval/eval.py`, `speed.py`, `vision.py`; raw
responses in `~/model-eval/out/`.

| | `qwen3-coder` | `qwen3.8-flash-next` | `…flash-uncensored` | `qwen3.8-27b-vl` |
|---|---|---|---|---|
| spec compliance (20 cases) | 19/20 | **20/20** | **20/20** | not measured¹ |
| bug hunt (4 real bugs) | 3 | **4** | **4** | — |
| runnable `/proc` script | ❌ | ✅ | ✅ | — |
| throughput (400 tok, warm) | **77.0 t/s** | 22.5 t/s | 35.0 t/s | 7.4 t/s |
| cold-ish load | 7.5 s | 20.5 s | 17.0 s | (warm) |

¹ `27b-vl` is dense Q8 at 7.4 t/s — a long answer needs ~18 min, so a 420 s client timeout saw
nothing. It is a *vision* model, not a worker: all three VL-capable models read a generated test
image exactly (`vision.py`), `qwen3-vl` fastest at 9.9 s end-to-end.

- **Careful work → a flash model, not coder.** Coder lost all three tasks: `"1H30M"` raised
  `ValueError` (no case folding), and its `/proc` script whitespace-split `/proc/<pid>/stat`
  (a `comm` with a space shifts the field index — it printed an 11 TB RSS) *and* printed KB as MB.
  Both flash models read `VmRSS:` from `/proc/<pid>/status` and matched `ps` 3/3.
- **Mechanical churn → coder**: no thinking, 77 t/s, 3.4× the per-token speed.
- **A flash turn pays thinking time** (~2048 reasoning tokens, preserved into `content`), so a client
  `max_tokens` must sit well above the reasoning budget or the answer never arrives.
- **Never drive two local models at once**: llama-swap serves one model at a time, so parallel
  requests for *different* models only buy load/unload cycles (155–312 s per request, measured).
- **omp delegation**: `sonic` → `@worker` (flash-next) does real tool work — verified `write` →
  `bash` → `yield` with a matching sha256. `@smol` stays coder; `task`/`reviewer` stay on cloud.
  Trust workers' artifacts, not their prose: that same probe claimed a trailing `\n` the file
  does not have.

## Monitor / tips
- GPU load: `amdgpu_top`
- **Agents (omp/pi):** thinking models work *if* their thinking is bounded — cap them with
  `--reasoning-budget 2048` **and** raise the client's `max_tokens` well above that (see the traps
  under A). Measured 2026-09-24: the flash pair beats `qwen3-coder` on every task tried, at ~3.4×
  lower throughput — pick per task from the table above, and never run two models at once.
- Quant guide: `Q4_K_M`/`UD-Q4_K_XL` default; bump to `Q5`/`Q6` for models under ~60 GB (plenty of RAM).
- DeepSeek: use the cloud subscription, not local.
