# 5. Local AI — unified memory, Vulkan, llama.cpp, llama-swap

**Date:** 2026-07-15 → 07-23 · the Strix Halo GTT config and the whole local LLM stack.
Day-to-day usage: [reference/llm-usage.md](reference/llm-usage.md).
Learning the internals: [../ai/](../ai/README.md).

[← Display rotation](04-display-rotation.md) · [Setup index](README.md) · [Next: ComfyUI →](06-comfyui-rocm.md)

---

## 6. Local AI / GPU unified-memory config (Strix Halo) — 2026-07-15

**Hardware (confirmed):** AMD **Ryzen AI MAX+ 395** (Strix Halo APU, GPU arch **gfx1151**) · iGPU **Radeon 8060S** · **128 GB** unified LPDDR5X (121 GiB usable) · single GPU, no discrete card · Kernel `7.0.11-3-cachyos-px13` · Bootloader **Limine**.

**Goal:** run local LLMs / image gen / AI on the iGPU via **Vulkan** (`vulkan-radeon` + `llama.cpp-vulkan`, later ROCm/pytorch/comfy), using the huge unified memory pool for large models.

### Memory model — the key concept
Two separate pools feed the GPU:
- **UMA / dedicated VRAM** = a *fixed carveout* reserved for the GPU at boot, set in **BIOS/Armoury Crate**. Whatever you give it is **permanently unavailable to the CPU/system**. → Set to the **minimum (512 MB)**.
- **GTT** = the GPU dynamically **borrows system RAM up to a ceiling**. It is a *cap, not a reservation*: when no model is loaded the GPU uses ~none of it and all RAM is free for apps (e.g. video editing); when a model loads it borrows up to the cap and frees it on unload.

So: tiny fixed VRAM + big dynamic GTT = maximum flexibility. Nothing is wasted when models aren't running.

**Defaults before change:** VRAM = 4 GB, GTT = 61 GB (≈ half RAM, the kernel default).
**Chosen target:** VRAM = 512 MB (BIOS), **GTT ceiling = 110 GB** via kernel params.

### Kernel params (added to Limine cmdline)
| Param | Value | Meaning |
|---|---|---|
| `amdgpu.gttsize` | `112640` | GTT size in **MiB** = 110 GiB. (Authoritative on recent kernels; may be ignored w/ warning on older — `ttm.pages_limit` is the real cap.) |
| `ttm.pages_limit` | `28835840` | Global TTM cap in **4 KiB pages** = 110 GiB. **This is the effective limit.** (110 × 1024³ ÷ 4096) |
| `ttm.page_pool_size` | `28835840` | TTM page pool = 110 GiB, so freed pages stay pooled for reuse. |

**IOMMU:** left **enabled** (decision 2026-07-15 — fewest changes). Known Strix-Halo failure mode: `amdgpu` page-fault crashes under sustained heavy GTT load. **If that happens, add `amd_iommu=off`** to the same cmdline line and re-run `limine-update` + reboot.

### How it was applied
CachyOS builds the kernel cmdline from `/etc/default/limine` (`KERNEL_CMDLINE[default]`). A **new `+=` line** was appended (the original root/LUKS line left untouched), then the boot config regenerated:

```fish
# 1. Back up first (disaster recovery)
sudo cp /etc/default/limine /etc/default/limine.bak-preGTT
# 2. Append GPU memory params (note the leading space inside the quotes)
echo 'KERNEL_CMDLINE[default]+=" amdgpu.gttsize=112640 ttm.pages_limit=28835840 ttm.page_pool_size=28835840"' | sudo tee -a /etc/default/limine
# 3. Regenerate /boot/limine.conf
sudo limine-update
# 4. Also set BIOS/Armoury UMA Frame Buffer -> 512 MB, then reboot
```

### Rollback (if boot breaks or you want defaults back)
```fish
sudo cp /etc/default/limine.bak-preGTT /etc/default/limine
sudo limine-update
```
At the Limine boot menu you can also pick the `*lts` or `*fallback` entry (see `BOOT_ORDER`) to boot a known-good kernel while fixing this.

### Verify after reboot
```fish
cat /proc/cmdline                                                   # should show the amdgpu/ttm params
cat /sys/class/drm/card*/device/mem_info_gtt_total | numfmt --to=iec # expect ~110G
cat /sys/class/drm/card*/device/mem_info_vram_total | numfmt --to=iec # expect ~512M (after BIOS change)
```
Monitoring tools to install: `amdgpu_top` (AUR), `rocm-smi`.

### Status
- [x] BIOS/Armoury UMA → 512 MB  *(done — `mem_info_vram_total` reads 512M)*
- [x] Append kernel params + `limine-update`  *(done)*
- [x] Reboot + verify  *(2026-07-15: `/proc/cmdline` has the params; GTT=110G, VRAM=512M confirmed)*
- [x] Install LLM stack (Vulkan): `vulkan-tools amdgpu_top ggml-vulkan llama-cpp`  *(done 2026-07-15)*
- [ ] Later: ROCm / PyTorch / ComfyUI (Step 3)

**✅ Step 1 (unified memory) COMPLETE 2026-07-15.** Verified GTT=110G / VRAM=512M.

### Step 2 — LLM inference via Vulkan — COMPLETE 2026-07-15
Installed from `cachyos-extra-znver4` (Zen4-optimized): `vulkan-tools`, `amdgpu_top`, `ggml-vulkan`, `llama-cpp` (b9957).
- **Packaging note:** the standalone `ggml-vulkan` pkg lagged at 0.15.3 while `llama-cpp` b9957 needs `ggml` 0.16.0, so pacman satisfied it with the **base `ggml` 0.16.0** (which `Provides: ggml-vulkan` and loads the Vulkan backend dynamically). At the provider prompt, chose **1) cachyos-extra-znver4**. Vulkan works because `vulkan-radeon` + `vulkan-icd-loader` are present — no ROCm needed, no `HSA_OVERRIDE`.
- **Backend:** Mesa **RADV** (`vulkan-radeon`), Vulkan 1.4, Mesa 26.1.4.
- **Verified:**
  - `vulkaninfo --summary` → `AMD Radeon 8060S Graphics (RADV STRIX_HALO)`, driver `radv`.
  - `llama-cli --list-devices` → `Vulkan0: AMD Radeon 8060S Graphics (RADV STRIX_HALO) (113152 MiB, 112190 MiB free)` — **~110 GB usable by the GPU for models.** End-to-end memory chain confirmed.
  - Inference smoke test ran successfully on GPU (`-ngl 999`).
- **Usage:** `llama-cli -hf <user/repo>:<QUANT> -ngl 999 -p "..."` (offload all layers). `llama-server` for an OpenAI-compatible API; `llama-bench` for tokens/sec. Monitor with `amdgpu_top`.
- **⚠️ Gotcha:** `llama-cli` drops into interactive mode when run non-interactively (no TTY) and spam-loops empty `>` prompts, ballooning any redirected log. For scripts/pipes use `-no-cnv` **and** feed stdin from `/dev/null` (or use `llama-cli -p ... -st`/`llama-bench`/`llama-server` instead).

### Serving via `llama-server` (OpenAI-compatible API)
```
llama-server -hf <repo>:<QUANT> -ngl 999 --port 8080 --jinja --alias <name>
```
- **Endpoint:** `http://127.0.0.1:8080/v1` · **API key:** ignored unless `--api-key` is set (use any string like `sk-local`).
- **Built-in chat UI:** `http://localhost:8080` (basic chat only — NO tools/web-search/RAG).
- `--alias <name>` gives the model a clean id in clients (else it reports the long GGUF path).
- **`--jinja` is REQUIRED for tool/function calling.** It applies the model's chat template and parses tool calls into structured `tool_calls`. Without it the model's tool-call syntax **leaks into the chat as raw text** (observed: `<function=web_search_exa>…` printed in the built-in UI when a query needed search).
- ⚠️ `llama-server` does **not execute tools** — it only formats/parses the protocol. The **client** must define AND run the tool (web search, file access, etc.). Built-in UI has none → use **Open WebUI** (enable its web-search feature) or an agent (aider/Cline/pi) for real tool use.
- **Run multiple models at once** on different ports (plenty of RAM): e.g. Coder on 8080, Instruct on 8081; both appear as selectable models in Open WebUI.

### Clients (all take an OpenAI base URL + model)
- **Open WebUI** (`open-webui`, self-hosted ChatGPT-style UI; has web-search feature) — point at `http://127.0.0.1:8080/v1`.
- **aider** (terminal coding agent), **Continue** / **Cline** (VS Code), **pi** (`@earendil-works/pi-coding-agent`).

### Recommended model shortlist (Strix Halo / Vulkan) — 2026-07
MoE models fly (bandwidth-bound box); dense models are slow. Default quant `Q4_K_M`/`UD-Q4_K_XL`; bump to `Q5_K_M`/`Q6_K` for models under ~60 GB (we have 110 GB). `-hf repo:QUANT` auto-downloads to `~/.cache/`.

| Model | Type | `-hf` repo | ~Size | ~Speed | Use |
|---|---|---|---|---|---|
| Qwen3-Coder-30B-A3B | MoE | `unsloth/Qwen3-Coder-30B-A3B-Instruct-GGUF` | ~18 GB | ~100 t/s | coding + agents (needs `--jinja`) |
| Qwen3-30B-A3B-Instruct-2507 | MoE | `unsloth/Qwen3-30B-A3B-Instruct-2507-GGUF` | ~17 GB | ~100 t/s | general chat/reasoning |
| Qwen3-VL-30B-A3B | MoE | `unsloth/Qwen3-VL-30B-A3B-Instruct-GGUF` | ~18 GB | fast | vision (image+text+tools) |
| GLM-4.5-Air (106B) | MoE | `unsloth/GLM-4.5-Air-GGUF` | ~65–70 GB | fast | strong coder (needs `--jinja`) |
| DeepSeek-R1-Distill-Llama-70B | **dense** | `unsloth/DeepSeek-R1-Distill-Llama-70B-GGUF` | ~42–58 GB | ~5–8 t/s 🐢 | reasoning (slow, dense) |

**GLM 5.2 does NOT fit** — 753B MoE, needs ≥256 GB even at 2-bit. GLM-4.5-Air is the fitting GLM.

**Coder vs Instruct:** both are chat-tuned ("Instruct" = not a base model). *Coder* = post-trained for code + **agentic** tool use (repo-scale, FIM, 256K–1M ctx) → use for aider/Cline/pi. Plain *Instruct* = general assistant.

### Easy launcher — fish `llm` function
`~/.config/fish/functions/llm.fish` — `llm <model> [port]` launches the right `llama-server` (all with `--jinja`, `-ngl 999`, `--alias`). Keys: `coder`, `instruct`, `qwenvl` (vision), `glm`. Default port 8080; run a second on 8081 (e.g. `llm instruct 8081`). `llm` with no args prints the menu. **This is the manual fallback — `llmswap` (llama-swap on 1234) is the everyday driver.**
**Preference (2026-07-15):** open-weight only, no OpenAI — `gpt-oss` removed (company dislike); `gemma` removed (dense → slow on this bandwidth-bound box, and its template breaks with `--jinja` + tools). Vision now via **Qwen3-VL** instead.

### Coding agents (omp / pi) → local server — WORKING 2026-07-16
Both speak the OpenAI API. Requirements on the server side: **`--jinja`** (tool-call parsing) + a **large `-c` context** — agents inject big system prompts + tool schemas + lint rules, so 32K overflowed omp into a compaction loop. Launcher now uses **`-c 131072 -fa on`**.
- **Server proven good via curl** (isolates client issues): `/v1/models`, non-stream, streaming SSE, and `tool_calls` (with `--jinja`) all return correctly.
- **omp** (`omp.sh`; config `~/.omp/agent/config.yml`, sqlite `~/.omp/agent/agent.db`, logs `~/.omp/logs/`): detects the endpoint as provider **`llama.cpp`** (not ollama — fine). Confirmed working with **`qwen3-coder`** (ran a web-search tool loop and answered). **GLM-4.5-Air failed in omp** — it's a *thinking* model; reasoning consumed the token budget → `contentBlocks:0, aborted, hasText:false`. **Rule: use non-thinking models (Qwen) for agents**, or disable thinking / raise token budget for GLM. omp's `snapcompact` wants a vision model — ignore for text, or use `qwenvl`.
- **pi** (`~/.pi/agent/`): fixes — (1) `auth.json` `openai.key` was empty `""` → pi rejected it, set to `sk-local`; (2) it defaults to DeepSeek cloud (`defaultProvider: deepseek`). **Final local setup: `~/.pi/agent/models.json`** with a single custom provider `local` → `http://127.0.0.1:1234/v1` (`api: "openai-completions"`, `apiKey: "sk-local"`) listing all llama-swap models (`qwen3-coder`, `qwen3-instruct`, `qwen3-vl`, `glm-4.5-air`). Pick via pi `/model`. (This superseded the earlier per-port-per-model providers once llama-swap fronts everything on 1234.) pi roles `--model`/`--smol`/`--slow`/`--plan` can each use a different model; pi discovers skills from `settings.json` `skills` paths (e.g. `~/.claude/skills`).
- **Multi-model reality:** llama.cpp = one model per process/port. Run several on different ports (RAM: GLM-Air ~68G + a Qwen ~18G fits; all three + 128K KV won't) or use **llama-swap** for one endpoint with auto-swap. GLM-4.5-Air works well for direct coding in pi/browser (fast MoE) even though it failed as an omp *agent* (thinking budget).
- **Skills:** omp supports skills natively (`--skills` discovery; installed agent-browser on first try, even pip/npm-installing the CLI itself). pi supports skills too (paths in `settings.json` `skills`, e.g. `~/.claude/skills`) but local models struggle to invoke them reliably — omp is the smoother skills experience on local models.

### llama-swap — unified endpoint (INSTALLED 2026-07-16)
One OpenAI endpoint that lists ALL local models and auto-loads/swaps them on request — removes the per-port juggling.
- **Binary:** `~/.local/bin/llama-swap` (v240, from GitHub release — not packaged; `curl` the `linux_amd64.tar.gz`, extract, `install -Dm755` to `~/.local/bin`). **Config:** `~/.config/llama-swap/config.yaml`. **Launch:** `llmswap` fish function → `llama-swap --config ~/.config/llama-swap/config.yaml --listen 127.0.0.1:1234`.
- **Endpoint `http://127.0.0.1:1234/v1`. Runs on 1234** (final, 2026-07-16) — the port omp probes for its OpenAI-dialect **lm-studio** provider. **KEY LESSON:** omp's built-in *llama.cpp* provider speaks llama.cpp-**native** paths (`GET /models`, native chat) that llama-swap does NOT serve (llama-swap is OpenAI-only, `/v1/...`) → every request 404s. 8080 failed for exactly this reason (omp only probes 8080 with that native provider). llama-swap must be reached through an **OpenAI** client: omp's lm-studio provider (port 1234), pi (openai-completions), or Open WebUI. Models: `qwen3-coder`, `qwen3-instruct`, `qwen3-vl`, `glm-4.5-air` (DeepSeek-R1-70B removed — cloud DeepSeek instead). `qwen3-coder` **preloaded on startup** via `hooks.on_startup.preload`. Default = one model at a time (swaps → RAM-safe); `ttl: 1800`; `healthCheckTimeout: 600`. Internal per-model ports auto-assigned via `${PORT}`.
- **Verified WORKING 2026-07-16:** omp (lm-studio provider), pi, and the playground all list every model and route/swap correctly on `127.0.0.1:1234`.
- **Clients:** pi → `~/.pi/agent/models.json` single `local` provider at `http://127.0.0.1:1234/v1` (done). omp → its **lm-studio** provider auto-discovers llama-swap at `127.0.0.1:1234/v1` (restart omp to discover; models appear under the "lm-studio" label). Open WebUI/browser → `http://127.0.0.1:1234`. All models are always listed, so no per-swap refresh needed.
- The manual `llm` function serves on **8080**, llama-swap on **1234** — no clash. `llmswap` is the everyday driver; `llm <model>` is a manual single-model fallback (and the only way omp's native llama.cpp provider works, since that needs a real llama-server).
- ⚠️ **IPv4/IPv6 gotcha (fixed 2026-07-16):** llama-server binds IPv4 `127.0.0.1` only, but llama-swap resolves the upstream `localhost`→`[::1]` (IPv6) → proxy fails with `dial tcp [::1]:PORT connection refused` and clients see "Server unavailable" even though `curl http://127.0.0.1:PORT/health` is OK. **Fix: add `proxy: http://127.0.0.1:${PORT}` to every model** in `config.yaml`. (Diagnosed via `ss -ltnp sport = :<childport>` showing 127.0.0.1 while the swap log showed `[::1]`.)
- ⚠️ **Dense 70B context:** DeepSeek-R1-70B at `-c 131072` used ~99 GB (dense KV is huge). Model later removed (use cloud DeepSeek); note kept for the KV-size lesson.
- ⚠️ **`localhost` vs `127.0.0.1`:** `llmswap` binds `127.0.0.1:1234` (IPv4 loopback). Use `http://127.0.0.1:1234` in clients/browser, NOT `localhost` — `curl` falls back to IPv4 but (a) some Node/Bun HTTP stacks resolve `localhost`→`::1` first and don't fall back, and (b) the browser on the `localhost` origin may serve a **stale service worker** cached by an old plain llama-server. pi uses `127.0.0.1`; omp probes `127.0.0.1`. `--listen :1234` (dual-stack) would make `localhost` bind directly but exposes llama-swap on the LAN.
- ⚠️ **omp 16→17 regression — broken `smol` role (fixed 2026-07-23):** omp updated `16.4.8 → 17.0.1` (config now `~/.omp/agent/config.yml`; providers cached in `~/.omp/agent/models.db` SQLite). CLI stopped working while Open WebUI was fine. Cause: `modelRoles.smol` was `llama.cpp/glm-4.5-air` → **"Model not found"** — wrong provider (`llama.cpp` = native, port 8080, only knows `qwen3-coder`; GLM lives under **lm-studio** on 1234). Since `mnemopi.llmMode = smol`, memory/lightweight tasks broke. **Fix: point both roles at the loaded model** — `smol` AND `default` = `lm-studio/qwen3-coder` (same model → zero swap thrash on this one-model-at-a-time setup). Removing `smol` also works (inherits) but memory relies on it, so set it explicitly. Session titles/memory now also use `omp tiny-models` (LFM2 700M), separate from the smol role. **Run on demand: `llmswap` must be running first** — no systemd service by choice; if `:1234` is empty the CLI has no endpoint (verify with `ss -ltnp | grep :1234`).
- 🚫 **GLM-4.5-Air is NOT usable as an omp *agent* — tool-calling breaks (confirmed 2026-07-23):** with omp's tools enabled (the default), `glm-4.5-air` **never returns** — llama-swap logs `error processing streaming response: no valid JSON data found in stream`. GLM-4.5's tool-call output via llama.cpp `--jinja` isn't parseable by omp, so it hangs indefinitely. **Same model with `--no-tools` answers fine**, and it's flawless/fast (70-80 tok/s) in Open WebUI (plain chat, no tools) — which is why "only the UI works." Diagnosis matrix: omp+tools → GLM ❌ / qwen3-coder ✅; omp+`--no-tools` → GLM ✅. **Use `qwen3-coder`/`qwen3-instruct` (or cloud deepseek) as omp's `default`; reserve GLM for the UI or `omp --model lm-studio/glm-4.5-air --no-tools` chat.** Note also: single `llama-server` slot (no `--parallel`) = one request at a time, so omp + UI block each other; and `defaultThinkingLevel: auto` picks 16-32k thinking tokens → multi-minute turns on slow models (both were red herrings vs the real tool-calling bug). Future fix for GLM-as-agent: newer llama.cpp + GLM-4.5-specific chat template/tool parser — not yet attempted.

---

# installing latest Qwen


 1. pacman -Ql ggml — see what .so files ship
 2. pacman -Qi ggml llama-cpp
 3. pacman -Q ggml-vulkan and other ggml backends
 4. vulkaninfo --summary — driver level
 5. ldd /usr/bin/llama-server | grep -i vulkan
 6. Check what backends ggml 0.20.0 provides

# Updating the system
```
 pacman -Syu
```

New versions of ggml do not include optional packages, so we need to install:

- Base ggml now ships only the core (libggml.so, libggml-base.so) and declares
   Provides: None.
 - Every compute backend is now a separate optional package: ggml-cpu,
   ggml-vulkan, ggml-blas, ggml-cuda, ggml-hip, ggml-openvino, ggml-sycl.
 - llama-cpp b10433 only Depends On: ggml — it pulls no backend.

```
sudo pacman -S ggml-vulkan ggml-cpu
```

Verify:

 ```fish
llama-cli --list-devices
 ```

 Expect:

 ```
Vulkan0: AMD Radeon 8060S Graphics (RADV STRIX_HALO) (~110000 MiB free)
CPU
 ```

# Downloading the models
```
hf download unsloth/Qwen3.8-27B-GGUF \
    --local-dir ~/models/Qwen3.8-27B-GGUF \
    --include "*Q8_0*"

# vision projector — separate file, needed for image input
hf download unsloth/Qwen3.8-27B-GGUF \
    --local-dir ~/models/Qwen3.8-27B-GGUF \
    --include "*mmproj*"
```

# Run it 
```
./llama-cli \
    --model ~/models/Qwen3.8-27B-GGUF/Qwen3.8-27B-Q8_0.gguf \
    --temp 1.0 --top-p 0.95 --top-k 20 --min-p 0.0 \
    --ctx-size 32768
```

With vision (use llama-mtmd-cli instead of llama-cli, and point --mmproj at the file you just downloaded):
```
./llama-mtmd-cli \
    --model ~/models/Qwen3.8-27B-GGUF/Qwen3.8-27B-Q8_0.gguf \
    --mmproj ~/models/Qwen3.8-27B-GGUF/mmproj-*.gguf \
    --temp 1.0 --top-p 0.95 --top-k 20 --min-p 0.0
```

For non-thinking/instruct mode, swap to --temp 0.7 --top-p 0.80 --presence-penalty 1.5 per their table.

One flag to watch: --ctx-size 32768 above is a reasonable starting point, not the full 262K native context — Q8_0 KV cache at 262K will eat a large chunk of your remaining memory. If you want to push context higher, add --cache-type-k q8_0 --cache-type-v q8_0 to quantize the KV cache itself, which lets you stretch further without much quality loss.


## Checks

1. Check devices
```fish
llama-cli --list-devices 2>&1 | grep -i -E "vulkan|backend|error"
```

2. The ggml -Qi output is ambiguous — ggml-vulkan: Vulkan backend could be either an installed sub-component or an optional dependency that isn't actually pulled in. Need the full context to tell which:

```fish
pacman -Qi ggml
```

Look specifically at the Optional Deps section — if it shows ggml-vulkan: Vulkan backend [installed], you're fine; if there's no [installed] tag, that's your answer — the Vulkan backend package exists but isn't on your system.

Also worth confirming the actual file is present regardless of what the metadata claims:

```fish
pacman -Ql ggml | grep -i vulkan
```

That should show something like /usr/lib/libggml-vulkan.so — if that line doesn't come back, the .so was dropped from ggml 0.20.0 in this package.

And finish the vulkaninfo check — your head -30 cut off before reaching the actual GPU/device list:

```fish
vulkaninfo --summary | grep -A3 "deviceName\|GPU id"
```



---

## 2026-09-07/08 — lineup changes, new tools, fork notes

The llama-swap lineup and launcher functions changed; this addendum is the
current source of truth where it conflicts with older sections above.

### Model lineup (llama-swap config, `~/.config/llama-swap/config.yaml`)

| key | model | size | notes |
|---|---|---|---|
| `qwen3-coder` | Qwen3-Coder-30B-A3B (UD-Q4_K_XL) | ~18G | daily agent driver, ~65 t/s |
| `qwen3-instruct` | Qwen3-30B-A3B-2507 | ~18G | general chat |
| `qwen3-vl` | Qwen3-VL-30B-A3B | ~18G | fast MoE vision |
| `qwen3.8-flash-next` | Qwen3.8-Flash-Next 125B-A6B (UD-IQ4_XS) | ~87G | strongest model; ~20 t/s decode on mainline |
| `qwen3.8-27b-vl` | Qwen3.8-27B dense Q8_0 + mmproj | ~29G | dense VL, kept for vision comparison |

**Removed 2026-09-07:** `glm-4.5-air` (omp tool-calling broken; weak coder,
3/8 on the registry's coding battery — chat/UI only) and `qwen3.8-27b`
dense (7 t/s, superseded by flash). Recipes for both stay in the registry
repo as knowledge.

Flash-Next is a **thinking model**: always run with
`--reasoning-effort medium --reasoning-budget 2048`, else it burns its
whole output budget on reasoning (`finish=length`). Cold load 87G takes
~1-2 min; llama-swap then keeps it 30 min (ttl).

### `llm.fish` / `llmswap.fish`

- `llm <model> [port]` now serves: `coder`, `instruct`, `qwenvl|vl`,
  `flash`, `qwen-vl|qvl` (run `llm` for the menu + expected t/s). No more
  `glm`/`qwen` keys.
- `llmswap` unchanged — restart it to pick up config edits
  (a running instance keeps the old model list).

### `llmv.fish` — single-image QA + model compare

```fish
llmv image.jpg "Is this image AI generated?"   # one-shot; default model = 3.8 (dense VL)
llmv --model 3 image.jpg "prompt"              # compare with qwen3-vl (fast MoE)
llmv                                          # no image -> interactive chat
```

- One-shot goes through llama-swap's OpenAI API (`127.0.0.1:1234/v1`)
  via `~/.config/fish/llmv_request.py`; prompt defaults to the
  AI-generated question.
- `--model 3` = `qwen3-vl` (fast), `--model 3.8` = `qwen3.8-27b-vl`
  (dense, better detail) — measured: 11 s vs 36 s incl. first load.
- Chat mode: 3.8 uses the local GGUF; 3 auto-detects the cached qwen3-vl
  mmproj.

**2026-09-13 — `llmv` no longer needs a hand-started `llmswap`.** It probes
`/api/version` on 1234 (llama-swap-specific; a plain `llama-server` doesn't
serve it) and, if nothing answers, runs the new `llmswap -d`: `setsid`+`nohup`
in its own session so the endpoint outlives the terminal, returning only once
llama-swap answers, logging to `~/.cache/llama-swap.log`. Foreground `llmswap`
is byte-for-byte the same as before.

Measured from cold (nothing running), same image, one question each:

| path | 1st call | later calls |
|---|---|---|
| `llmv --model 3` (qwen3-vl MoE) | 12.7 s | **2.0 s** |
| `llmv` (default 3.8, dense) | 29.3 s | 23.0 s |
| `llama-mtmd-cli` one-shot, no daemon | 10.0 s | ~10 s **every time** |

So the daemon wins from the second image on, at a ~3 s premium on the first.
Two things to know: the config's `on_startup` preload of `qwen3-coder` fires on
every autostart and is then swapped out by the first vision request (inside that
12.7 s), and the dense default `3.8` is slow *warm* as well — that is image
prefill, not loading. Use `--model 3` for quick questions.
`POST /api/models/unload` frees a model immediately; `ttl: 1800` does it after
30 min idle.

### Nathan fork + MTP (speed path, NOT in llama-swap)

- Fork built from source at `~/src/nathan-llama.cpp/build-vk/bin/llama-server`
  (branch `strix-halo-vulkan`, commit `5d8c07b44`, build 10659;
  upstream `github.com/Nathanw1014/llama.cpp`). Distro `/usr/bin/llama-server`
  untouched.
- **MTP head must use upstream (agentionai) naming** — unsloth `MTP/`
  heads (nested `blk.N.nextn.*` tensors) fail with
  `check_tensor_dims: tensor output_hc_norm.weight not found`.
  Working: `~/models/Qwen3.8-Flash-Next-GGUF/MTP-agentionai/Qwen3.8-Flash-Next-MTP-Q8_0.gguf`
  + `--spec-type draft-mtp --spec-draft-n-max 4` (fixed n4; adaptive loses
  on MTP chained drafts).
- Measured (IQ4_XS, same weights): mainline ~20 t/s decode → fork+MTP
  ~35 t/s mean (1.6-1.8×); cold PP 320 → 383 t/s. Fork patches are
  prefill-only; the decode win is the MTP sidecar. Mainline can't load the
  standalone MTP head (PR #28243 unmerged). Full A/B + per-flag WHY in the
  registry recipe `qwen3.8-flash-next v0.2.0`.

### Pitfalls learned

- **Fit abort**: `failed to fit params … n_gpu_layers already set by user
  to 999, abort` appears at large `-c` in some memory states; dropping ctx
  (e.g. 32k → 8k) or a clean restart works.
- **Wayland session can drop** while loading/unloading ~87G models (no
  kernel/amdgpu error seen). If it recurs under heavy GPU swapping,
  consider the handbook's `amd_iommu=off` hardening.
- Reasoning models + small `--max-tokens` = answers truncated into
  thinking; give ≥768 tokens or disable thinking for short answers.
- One GPU: a single llama-server at a time; two residents contend badly
  (observed TG 60 → 2 t/s).

All recipes, flags, results and the quality battery live in the registry
repo: `~/Projects/strixhalo` (published as
`github.com/mritzco/strixhalo-recipes`).

---

## 2026-09-13 — llama-swap starts empty; `-p` to warm one model

Day-to-day: [reference/llm-usage.md](reference/llm-usage.md),
[reference/vision-usage.md](reference/vision-usage.md).

**Change.** `hooks.on_startup.preload` in `~/.config/llama-swap/config.yaml` went
from `[qwen3-coder]` to `[]`, so **llama-swap now starts with nothing loaded** —
the right default when the plan is DeepSeek in the cloud on battery, and models
are configured when actually used. Warming moved into the launcher:

```fish
llmswap                    # nothing loaded (default)
llmswap -p qwen3-coder     # warm one model in the background
llmswap -d -p qwen3-vl     # detached + warm
llmswap -p none            # explicit nothing
```

- The default is one line at the top of `llmswap.fish`: `set -l default_preload ''`.
- Warming hits `/upstream/<model>/health`, which starts the upstream and runs
  **no inference**; it is fired detached (own session) because in the foreground
  case `llama-swap` owns the terminal and the function never regains control.
  A bad model id 404s and is reported — needs `curl -f`, since without it curl
  treats 404 as success and the failure is silent.
- Method note: `/upstream/<model>/health` looks like it *did* trigger a load the
  first time it was tested, but that was omp's own request racing the curl. The
  log line to check is always `Request ... "POST /v1/chat/completions"`.

**Measured consequence for `llmv`** (invoice scan, cold = nothing running):

| path | 1st call | later |
|---|---|---|
| `llmv --model 3` | **7.8 s** | **2.1 s** |
| `llmv` (default 3.8) | 29.3 s | 23.0 s |
| `llama-mtmd-cli`, no daemon | 10.0 s | ~10 s every time |

Dropping the startup preload made the first `llmv` call *faster* than the
no-daemon path (7.8 s vs 10.0 s): the coder preload used to load and then be
swapped straight out by the first vision request.

### ⚠️ Correction to the lineup table above

The 2026-09-07 table lists `qwen3-instruct`, which is **not in the config**
(and never was, as of today's file). Actual keys in
`~/.config/llama-swap/config.yaml`:

| key | model | notes |
|---|---|---|
| `qwen3-coder` | Qwen3-Coder-30B-A3B (UD-Q4_K_XL) | daily agent driver |
| `qwen3-vl` | Qwen3-VL-30B-A3B | fast MoE vision (~18 G) |
| `qwen3.8-flash-next` | Qwen3.8-Flash-Next 125B-A6B (UD-IQ4_XS) | strongest; thinking, needs `--reasoning-effort` |
| `qwen3.8-flash-uncensored` | Flash-Next Uncensored (IQ4_XS) + mmproj + MTP | runs the fork's llama-server |
| `qwen3.8-27b-vl` | Qwen3.8-27B dense Q8_0 + mmproj | dense vision (~29 G), slow prefill |

`instruct` still exists as a *separate* `llm` (manual, port 8080) key — it is not
a llama-swap model. Read the config file as the source of truth:

```fish
python3 -c "import yaml,pathlib;print(list(yaml.safe_load((pathlib.Path.home()/'.config/llama-swap/config.yaml').read_text())['models']))"
```

### ⚠️ Clients load models whether you preload or not

With the endpoint up, **omp requests `qwen3-coder` within seconds** — llama-swap
logs `POST /v1/chat/completions ... "omp/18.1.14"`, load 5.8 s. "Nothing
resident" therefore also means the client is not asking; llama-swap itself now
loads nothing on start, but that is not the whole story.
