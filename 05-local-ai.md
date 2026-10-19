# 5. Local AI — unified memory, Vulkan, llama.cpp, llama-swap

**Date:** 2026-07-15 → 07-23 · the Strix Halo GTT config and the whole local LLM stack.
Day-to-day usage: [reference/llm-usage.md](reference/llm-usage.md).

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

## 2026-09-24 — which local model for coding, measured; and the omp worker wiring

Three objectively-scored tasks (a spec-compliance function with 20 hidden cases, a 4-bug hunt, a
runnable `/proc` script) plus sequential throughput. Harnesses in `~/model-eval/`
(`eval.py`, `speed.py`, `vision.py`), raw responses in `~/model-eval/out/`.

| | `qwen3-coder` | `qwen3.8-flash-next` | `…flash-uncensored` | `qwen3.8-27b-vl` |
|---|---|---|---|---|
| spec compliance (20 cases) | 19/20 | **20/20** | **20/20** | not measured¹ |
| bug hunt (4 real bugs) | 3 | **4** | **4** | — |
| runnable `/proc` script | ❌ | ✅ | ✅ | — |
| throughput (400 tok, warm) | **77.0 t/s** | 22.5 t/s | 35.0 t/s | 7.4 t/s |
| cold-ish load | 7.5 s | 20.5 s | 17.0 s | (warm) |

¹ 27b-vl is dense Q8 at 7.4 t/s; a long structured answer needs ~18 min, so a 420 s client timeout
returned nothing. That is a harness limit, not a model verdict — but it *is* the verdict on using it
as a worker (10× slower than coder). It stays a vision model.

**The lineup rule changed.** "Use `qwen3-coder` for agents" was right when the alternative was GLM
(unparseable tool-call stream). With the reasoning budget capped it is no longer the quality pick:
coder lost all three tasks. Its T1 miss is real (`"1H30M"` → `ValueError`, no case folding), and its
`/proc` script contains two genuine bugs — it whitespace-splits `/proc/<pid>/stat`, so a `comm`
containing a space shifts the field index (it printed an 11 TB RSS, `Privileged Cont`), and it prints
KB labelled MB. Both flash models read `VmRSS:` from `/proc/<pid>/status`, tolerate vanished PIDs,
fall back `comm` → `cmdline`, and matched `ps` 3/3.

**They are careful, not fast.** Every turn spends the 2048-token reasoning budget (preserved into
`content`), so equal output costs ~1.5–2× the wall clock and ~3.4× per token. Mechanical churn →
coder; anything where a wrong edit costs a rerun → flash.

**Their failure mode is the client's**, twice over: run 1 scored both flash models **0/20** only
because a 1600-token client cap was consumed by thinking (`finish=length`) — the same trap as the
Playground `max_tokens: 4096` above, reproduced by the harness that documented it. Re-run at 8000:
20/20. Any client driving these models needs a cap well above the thinking budget.

**Vision** (generated image, checkable JSON: digits / largest / count — `vision.py`): all three
VL-capable models returned exact ground truth; `qwen3-vl` 9.9 s, `flash-uncensored` 24.2 s,
`27b-vl` 33.9 s (end-to-end incl. load). Proves capability, not a ranking — the image was easy.

**Tool loop verified for omp.** `sonic` routed to `@worker` (flash-next) ran `write` → `bash` →
`yield` in 6m12s; the sha256 it reported matched the file exactly (15 bytes, no trailing newline).
The cost is latency, not correctness — and its prose claimed a `\n` the file does not have, so:
**verify the artifacts, not the summary.**

### Running two local models at once is never right here

llama-swap's default with no `groups:` is *single model at a time* — a request for another model
stops the one running (this file already noted the coder preload being "swapped straight out by the
first vision request"). A 3-model parallel evaluation therefore measured 155–312 s per request and
2–10 t/s: every request paid a load/unload. `speed.py` is sequential for exactly this reason.
`flash-next` is a 125B-A6B at UD-IQ4_XS holding ~65 GB in GTT, so coexistence was never on the table.

The same fact makes background housekeeping expensive: a session-title or typed-judgment request
mid-run swaps the working model out. `modelRoles.tiny`/`judge` (and `memory`) now point at the
on-device ONNX tiny models — no llama-swap involved, and ~90 ms per title instead of a reload.

### omp wiring applied 2026-09-24

- `~/.omp/agent/models.yml` (**new**): pins the local models to the server's truth — every cmd runs
  `-c 131072`, and only `--mmproj` processes accept images. Discovery had synthesized 1M context
  (omp would compact at ~8× the real window and only learn from a server error:
  `request (N tokens) exceeds the available context size`) and had image capability *inverted*
  (`images: yes` for flash-next, which has no projector; `no` for `flash-uncensored`, which has one).
  Declaring `lm-studio` explicitly replaces the implicit discoverable provider, so its `discovery`
  block must be spelled out or the provider ends up with zero models.
- `~/.omp/agent/config.yml`: `worker`/`worker2` role aliases plus
  `task.agentModelOverrides.sonic: "@worker"` — bundled agent prompt, local model, no hand-written
  agent file needed (backup: `config.yml.bak-20260924-local-workers`).
- Contents of both files: [reference/config-files.md](reference/config-files.md#local-model-routing-ch-5-applied-2026-09-24).

**Verified after wiring, not assumed:**

- `judge()` (typed Eval judgment) answers in **0.26 s** with **zero** new requests in llama-swap's
  `/logs` (34 → 34 POSTs) — the `judge` role is really on-device. Both workers are up:
  `~/.omp/run/tiny/lfm2-1.2b-onnx.sock` and `lfm2.5-230m-onnx.sock`, each logging
  `omp tiny worker listening on …`.
- Why that matters, measured: the last llama-swap-served **session title** cost **3m47s** of wall
  clock (`title-generator: start 19:59:49` → `success 20:03:36`, `model: lm-studio/qwen3-coder`,
  337-token prompt) because it queued behind model loads and swaps. On-device judgment is 0.26 s.
- **[WARN] `omp tiny-models download lfm2.5-230m` fails transiently.** First attempt died with
  `One or more tiny title models failed to download` leaving a 4.7 MB partial; an immediate retry
  fetched the full 207 MB (`onnx/model_q4.onnx` + `model_q4.onnx_data`). Verify the cache, not the
  exit message.

**Local models are opt-in, and their absence now degrades to cloud (2026-09-24).** A default session
touches no llama-swap at all: `default`/`smol`/`plan`/`advisor` are cloud, and `tiny`/`memory`/`judge`
are on-device ONNX (0.26 s, no GPU, unaffected by the daemon). Only `worker`/`worker2` (what `sonic`
resolves to) and `vision` need it — so **`omp` alone is a complete session**, and `llmswap -d && omp`
is what adds local work. Measured with a model pinned at a dead endpoint: **without**
`retry.fallbackChains` the call retries and never answers; **with** the chain it answers from the
fallback (same prompt → `OK`). Chains are in `config.yml` (backup `config.yml.bak-20260924-fallbacks`);
the endpoint is pinned in `models.yml`, which is also why `LM_STUDIO_BASE_URL` is now inert.

---

## 2026-10-03 — halogen-flash-server 0.16.2 trialled (measured, NOT wired in)

Third-party engine (`github.com/peonist-ai/halogen-flash-server`, ghcr image). gfx1151-only, and the
engine is a **compiled binary under an EULA** (`LICENSE.md`) — run/benchmark/redistribute-unmodified,
no source, so it is *not* a fork to patch the way Nathan's and myhacsint's are. Serves one model
family (Qwen3.8-Flash-Next); optionally serves small models (embeddings, rerank, moderation,
classifier, 2B generation) on the **NPU**. Trialled GPU-only against our existing `UD-IQ4_XS` GGUF
through its 0.7.0+ "bring your own GGUF" path: 1.42 GiB draft head (`qwen38-flash-next-mtp.hgn`) +
21 MB tokenizer, **no 110 GiB checkpoint download**.

```fish
podman run -d --name halogen -p 8731:8731 \
  --device /dev/kfd --device /dev/dri --group-add keep-groups \
  --ipc=host --ulimit memlock=-1:-1 \
  -e HALOGEN_DOWNLOAD=peonist-ai/halogen-qwen3.8-flash-next \
  -e HALOGEN_CHECKPOINT=/models/Qwen3.8-Flash-Next-UD-IQ4_XS-00001-of-00003.gguf \
  -e HALOGEN_CTX=131072 -e HALOGEN_MAX_TOK=16384 -e HALOGEN_MAX_TOKENS_DEFAULT=16384 \
  -e HALOGEN_MODEL_ID=qwen3.8-flash-next \
  -v ~/halogen-models:/models \
  ghcr.io/peonist-ai/halogen-flash-server:0.16.2
```

`~/halogen-models` holds hardlinks to the three GGUF shards (`ln -f`, same filesystem → no extra
disk) and receives the draft head and tokenizer on first start. `HALOGEN_MAX_TOK` is the **prefill
scratch** size (default 32768 ≈ 9 GB), not a token budget.

**Measured here, same `UD-IQ4_XS` weights (`~/model-eval/halogen-ab.py`, `halogen-verify.py`):**

| | fork+MTP (ours) | halogen 0.16.2 |
|---|---|---|
| prefill 8k / 32k | 320 → 383 t/s | **1268 / 1051 t/s** |
| prefill, 97k prompt | — | 1029 t/s, 73.9 s |
| decode, 400 tok, thinking off | — | 26.5 t/s (draft 186/212) |
| decode, 967-tok turn, default thinking | 22.5 / 35.0 t/s (400-tok turns) | 32.8 t/s (709 reasoning) |
| follow-up on 21.7k context | — | **2.1 s** (21,696 tok cached; exact repeat 0.94 s) |
| tool loop: call → result → answer | GLM broke here | 3.7 s + 1.5 s, arguments valid JSON |

Decode is comparable; **prefill is ~2.7–3.3×**, and that plus the prompt cache is what a 100k-context
agent turn actually spends its time on. Caveat: our 22.5/35.0 were 400-token turns at temp 0.2 with
`--reasoning-budget 2048`; halogen's 32.8 is a greedy 967-token turn. Not an identical prompt set.

- **Boot:** GGUF repacked in 17.8 s (70.55 GiB), registered read-only (`device ptr == host ptr`).
  Footprint **90.4 GiB** = 72.0 weights + 7.2 KV pool (262144 positions) + 11.3 working; the 26.8 GiB
  lookup table is read from disk, not held. `/health` reported `context 131072`, cap 65536, engine and
  API both 0.16.2.
- **It holds the machine.** Only 22.6 GiB contiguous was left for the desktop, and `free`/`MemAvailable`
  over-report by ~62–68 GiB (locked weights counted as reclaimable cache). Stopping returns all of it.
- ⚠️ **Start-time fragmentation is the real gotcha.** The log warned of only 268 contiguous 2 MiB
  blocks (536 MiB) *before* halogen allocated; while running, `/proc/buddyinfo` Normal fell to **5**
  blocks. After stop it recovered to 4031. Before a restart, as root:
  `echo 1 > /proc/sys/vm/compact_memory` and confirm `awk '/Normal/{print $14}' /proc/buddyinfo` is in
  the thousands — the same contiguous-memory wedge as [08-resilience.md](08-resilience.md), and the
  reason this must not be started on a loaded desktop.
- Plumbing is llama-swap-friendly: `/metrics` carries `llamacpp:*` names and every response a
  llama-server `timings` object. Also speaks `/v1/responses` (Codex) and `/v1/messages` (Claude Code),
  structured output (greedy only), and lists as `qwen3.8-flash-next` at `/v1/models`.
- **NPU left off, deliberately.** It needs host XRT (`xrt` + `xrt-plugin-amdxdna` — *absent* here) and
  the GPU fabric clock held (`deploy/host/halogen-fabric-clock.service`), or GPU+NPU concurrency can
  hang the machine. Note `amd_iommu=off` — our hardening lever for the GTT page-fault / session-drop
  crashes — **turns the NPU off**; halogen's own published GPU numbers were measured *with* it off.
- Its own MTP draft heads are required. Our unsloth/agentionai MTP GGUFs are refused; pointing
  `HALOGEN_MTP_HEAD` at one is a hard start error.
- Not in llama-swap, not in omp/pi `models.yml` — nothing in the running stack references 8731 yet.

**2026-10-03 (later) — uncensored trunk, and the registry record.** Re-ran the same config on the
uncensored IQ4_XS (`~/models/Qwen3.8-Flash-Next-Uncensored-GGUF/IQ4_XS/…`, 92 GB single file, read in
place): prefill **1314 / 1092 t/s** @6.7k / 26.7k, decode **28.4 t/s** thinking-off (draft 176/192 =
**91.7 % accepted** — halogen's own head transfers to the abliterated trunk fine) and **33.3 t/s** over
a 1304-token thinking turn; tool call valid JSON in 4.5 s. Uniformly ~3–4 % ahead of `UD-IQ4_XS` on
the same server (1268 / 1051 t/s, 26.5 / 32.8 t/s), because the 4-bit trunk reads less per token.
Footprint **89.1 GiB** (69.0 weights + 7.2 KV + 12.9 working) — *less* than the 8-bit `UD-IQ4_XS`
(90.4). This is the trunk to standardise on if any is.

Registry: `recipes/qwen3.8-flash-next-uncensored-halogen/` v0.1.0, hash `ae1aef67`, `engine: other`,
`launch.serving` carrying the container topology; two result records under
`results/qwen3.8-flash-next-uncensored-halogen/ae1aef67…/` (one cache-on, one cache-off);
`tools/validate.py` clean at 13 recipes / 20 results.

**Two registry defects found** (written into the recipe's `failure_notes` / annotations, not worked
around silently):

1. `tools/analyze.py` derives a test's runner filename as `<id with - → _>.py` and **ignores the
   definition's `runner` field**. `context-recall` therefore looks for `context_recall.py`, never
   finds `tests/runners/context_needle.py`, and prints `[skip] no runner for context-recall` — every
   recipe declaring it (including the committed `qwen3.8-flash-next-uncensored` v0.1.0) is silently
   under-tested. Fix: resolve `runner` from `tests/definitions/<id>.json`.
2. The throughput runner's `tokens_per_sec_pp` divides **total** prompt tokens by wall time, so any
   prefix cache inflates it. Same recipe, same weights, only `HALOGEN_PROMPT_CACHE` differing:
   **9514.6 t/s cached vs 976.6 t/s cold**, while the engine's own `timings.prompt_per_second` was
   1092 t/s. `tokens_per_sec_tg` moves too (11.7 vs 5.6) because the runner reuses one ~11k-token
   system prompt in every generation round, so the round is almost all prefill. **Benchmark this
   server with `HALOGEN_PROMPT_CACHE=0`; serve it with the default (`2`).** llama-server caches
   prefixes as well, so other records may be inflated the same way.

**NPU is one `pacman` away** (needs root; `sudo` here wants a password):

Staged, verified host files live in `~/halogen-npu-host/` (copied verbatim from the repo tag `v0.16.2`; you do
not need a checkout). `xrt` + `xrt-plugin-amdxdna` are **installed** (1:2.21.75); the three `libxrt*.so.2` are in
`/usr/lib`, and the KFD SVM bit is **SET** (`capability=671326848`, node gfx 110501) with
`cwsr_size=19185664` = `ctl_stack_size + (simd_count/2)*479232` = `16384 + 40*479232` — both gfx1151 fixes present,
so the engine's read-only mapping registration will work.

```fish
# 1. host files (staged from the v0.16.2 tag, reviewed: writes only pp_dpm_fclk +
#    power_dpm_force_performance_level under /sys/class/drm/*/device, reversible with `release`)
sudo install -m755 ~/halogen-npu-host/halogen-fabric-clock         /usr/local/sbin/
sudo install -m644 ~/halogen-npu-host/halogen-fabric-clock.service /etc/systemd/system/
sudo systemctl daemon-reload && sudo systemctl enable --now halogen-fabric-clock.service
sudo halogen-fabric-clock status        # expect: /sys/class/drm/card1/device: held (manual); exit 0

# 2. memlock — BLOCKER, proven 2026-10-03, and NOT fixable via limits.conf on a graphical session.
#    The NPU locks its models' memory: ~8.58 GiB for all five. This box's limit is 8192 KiB (8 MiB),
#    and a rootless container cannot raise it (podman clamps --ulimit memlock=-1:-1 silently):
#      "this container may lock 8192 KiB of memory, and the NPU needs about 8997531 KiB"
#
#    WHY limits.conf DOES NOT WORK HERE (measured, not inferred):
#      the shell runs under user@1000.service (logind session 3, Class=manager, Service=systemd-user).
#      That unit is spawned by PID 1, NOT through a PAM session, so /etc/security/limits.d/ is never
#      consulted for it. Evidence: systemctl show user@1000.service -p LimitMEMLOCK -> 8388608,
#      systemctl show -p DefaultLimitMEMLOCK -> 8388608 (systemd's compiled-in default, cf.
#      system.conf:73 "#DefaultLimitMEMLOCK=8M"), /proc/self/limits -> 8388608. Everything in the
#      graphical session (niri, terminals, podman) inherits that. pam_limits IS in system-auth, so
#      limits.conf still works for *TTY/login/su/sudo* paths — just not the user manager.
#
#    Ancestry, measured: niri (and therefore alacritty/fish/omp) has PPid 2671 = user@1000.service;
#    omp -> fish -> alacritty -> 2671 -> 1, every one at 8388608. user@1000 SURVIVED a logout, so
#    logging out cannot fix this — the user manager must be respawned with a new limit.
#    Ceiling note: PID 1's own memlock is 16768991232 (15.6 GiB), so even if systemd clamps the
#    requested infinity to its own hard limit, 15.6 GiB still exceeds the 8.58 GiB the NPU needs.
#
#    (a) zero-config, immediate: log in on a TTY (Ctrl-Alt-F3). VERIFIED PAM chain:
#        /etc/pam.d/login -> system-local-login -> system-login -> system-auth -> pam_limits.
#        So limits.conf below applies there and `ulimit -H -l` reads `unlimited`; start the
#        container from that TTY. (sudo from the desktop does NOT help: pam_limits would look up
#        ROOT in limits.conf, which has no root entry, so it inherits the session's 8 MiB.)
#    (b) permanent, for the graphical session — a drop-in on the user manager, then reboot:
sudo mkdir -p /etc/systemd/system/user@.service.d
printf '[Service]\nLimitMEMLOCK=infinity\n' | sudo tee /etc/systemd/system/user@.service.d/50-memlock.conf
sudo systemctl daemon-reload
#    reboot (a plain logout may not respawn user@1000.service), then confirm: ulimit -H -l -> unlimited
#
#    /etc/security/limits.d/50-memlock.conf (itzco hard/soft memlock unlimited) is still worth
#    keeping for the TTY/su/sudo paths:
printf 'itzco hard memlock unlimited\nitzco soft memlock unlimited\n' | sudo tee /etc/security/limits.d/50-memlock.conf

# 3. run line: add the NPU device, mount the host's XRT twice (Arch/CachyOS put it in /usr/lib,
#    and those libs look each other up at their own path), and name the models
#   --device /dev/accel/accel0
#   -e HALOGEN_DOWNLOAD=peonist-ai/halogen-qwen3.8-flash-next   # also fetches the NPU repos
#   -e HALOGEN_NPU_MODELS=decider-0.8b,qwen3-embedding-0.6b,qwen3-reranker-0.6b,qwen3guard-gen-0.6b,qwen3.5-2b
#   and, instead of -v /opt/xilinx/xrt:/opt/xilinx/xrt:ro:
#     L=/usr/lib; X=""
#     for f in libxrt_coreutil.so.2 libxrt_core.so.2 libxrt_driver_xdna.so.2; do
#       X="$X -v $(readlink -f $L/$f):/opt/xilinx/xrt/lib/${f}:ro -v $(readlink -f $L/$f):$L/${f}:ro"
#     done
#     podman run … $X …
```

**What the NPU start got right before it hit the memlock wall** (so the remaining setup is verified):
the fabric-clock gate passed — `the GPU's fabric clock is held at its top speed (2000Mhz), so the NPU
starts beside it`; `device /dev/accel/accel0 … driver amdxdna (the kernel's own), XRT … with its NPU
plugin`; and all five model repos were fetched into `~/halogen-models/npu` (**8.1 GiB**, each file
checksummed against the image's own record) so no re-download is needed.
`decider-0.8b` 2.2G, `qwen3-embedding-0.6b` / `qwen3-reranker-0.6b` / `qwen3guard-gen-0.6b` 776M each,
`qwen3.5-2b` 3.4G — the `.hnpw` files plus per-model `devices/u*.elf` NPU programs.

Note the 8.58 GiB is **mlocked**, unlike the Flash weights: it cannot be swapped or reclaimed, so the
full stack is ~99 GiB pinned.

Undo: `sudo systemctl disable --now halogen-fabric-clock.service && sudo halogen-fabric-clock release`.

The fabric clock is not optional: GPU and NPU work at once can hang this SoC while `fclk` changes.
Arch/CachyOS XRT lives in `/usr/lib`, so mount the three libraries twice per `docs/NPU.md`
(`L=/usr/lib`), not `-v /opt/xilinx/xrt`.

**2026-10-03 (later still) — vision ON, verified on the abliterated trunk.** `qwen38-flash-next-vision.hgn`
(0.84 GiB, from `peonist-ai/halogen-qwen3.8-flash-next`) + `HALOGEN_VISION_TOWER=1` resolves it next to the
checkpoint, so no extra mount. This was the open risk — the tower is fitted to the *original* trunk and these
weights are an abliterated requant — and it works unchanged: the ch.5 harness image reads exactly
`{"digits": "4287", "largest_color": "red", "rect_count": 3}` in **4.9 s** (96 out-tokens, 29.7 t/s decode),
the frame costing 2560 context tokens. Footprint 89.1 → **90.0 GiB** (+0.84 resident, +1.14 worst-case
scratch), and that start saw **0 compaction stalls** during KV reservation.

Recipe `v0.2.0`, hash `05ebcf80` (a new variant — `launch.command` is hash-relevant, so v0.1.0's two records
stay valid under `ae1aef67`). Records now: 2 under `ae1aef67`, 2 under `05ebcf80`; `validate.py` clean at
14 recipes / 22 results.

**2026-10-03 (final) — the NPU works; all five models verified.** After the memlock drop-in + reboot
(`ulimit -H -l` → `unlimited`, and the fabric-clock unit held 2000 MHz by itself at boot), the container
started with `--device /dev/accel/accel0` + the doubled XRT mounts and all five models:

```
npu_api: decider-0.8b on the NPU engine at 127.0.0.1:8740 (contexts 4, queue 64)
npu_api: qwen3-embedding-0.6b (embeddings) · qwen3-reranker-0.6b (rerank)
npu_api: qwen3guard-gen-0.6b (moderation) · qwen3.5-2b (generation)
startup: 0 compaction stalls (both steps)
```

Measured (`~/model-eval/halogen-npu.py`):

| endpoint | result |
|---|---|
| `/v1/embeddings` | 1024-dim, unit norm; one 392-token text **107 ms**; 8-text batch 3,176 tok in 0.43 s = **7,465 tok/s** |
| `/v1/rerank` | ordering sane: "needs amdxdna, XRT + NPU plugin, IOMMU" **0.0331** > `ls /dev/accel` 0.0019 > GPU 0.0004 > bananas 0.0000 (0.42 s) |
| `/v1/moderations` | safe→`Safe 0.998`, unsafe→`Unsafe 0.998`, `flagged` correct (0.14 s) |
| `decider-0.8b` | `"no"` p=**0.706** vs `yes` 0.294, one pass, **0.08 s** |
| `qwen3.5-2b` | coherent summary, 0.91 s |
| **GPU+NPU concurrency** | Flash alone **38.4 t/s** → with 29 NPU batches in parallel **33.5 t/s (−13 %)**; second run −8 %. No errors, no hang |

This is capability the llama-swap stack has **no equivalent for** — nothing else here serves embeddings or
rerank. That is the strongest argument for consolidating onto halogen.

Recipe `v0.3.0`, hash printed at write time (new variant: the device, the XRT mounts and
`HALOGEN_NPU_MODELS` all live in the hash-relevant `launch.command`). `validate.py` clean. Note the
registry's `capabilities_confirmed` only models tools/mcp/vision, so the NPU capability set has no
schema representation and lives in an annotation instead — worth a schema field if this sticks.

**2026-10-03 — reclaim pass: 199 GB freed (677 → 478 GB used; 273 → 471 GB free).** Deleted only
after confirming nothing live referenced each item (grep across llama-swap config, fish functions,
omp models.yml/config.yml, pi models.json):

| removed | size | why it was safe |
|---|---|---|
| `~/.cache/huggingface/hub/models--unsloth--GLM-4.5-Air-GGUF` | 64 G | dropped from the lineup 2026-09-07, no live reference; recipe kept in the registry. **Re-download required if wanted back.** |
| `~/models/Qwen3.8-Flash-Next-GGUF` (stock `UD-IQ4_XS` + agentionai/unsloth MTP sidecars) | 98 G | superseded by the uncensored trunk; halogen refuses llama.cpp MTP GGUFs (it wants `.hgn`). Its 3 shards were also hardlinked into `~/halogen-models` — both copies removed (the hardlink kept the data alive otherwise). |
| `~/models/Qwen3.8-27B-GGUF` | 29 G | dense 27B VL, pruned 2026-10-03; halogen's vision replaced it and is faster (4.9 s vs 33.9 s). |
| `…/Qwen3.8-Flash-Next-Uncensored-GGUF/{MTP-unsloth,mmproj}` | 3.5 G | the engine refuses the first and ships its own vision sidecar, so the llama.cpp mmproj is unused. |
| `~/src/nathan-llama.cpp`, `~/src/myhacsint-llama.cpp` | 8 G | both source forks retired with the flash rows; recipes live in the registry. |

**Kept:** the 92 GB uncensored GGUF (the live checkpoint — the container mmaps it), `qwen38-flash-next-mtp.hgn`
+ `qwen38-flash-next-vision.hgn` + `tokenizer/` + `npu/` (8.1 G), and in the HF cache everything still
reachable: `qwen3-coder` 17 G, `qwen3-vl` 18 G, `qwen3-instruct` 17 G (still the `llm instruct` key),
DeepSeek-OCR 6.3 G. The server stayed healthy across all deletions (no file it needs was touched).

**Unsloth Studio** (`~/.unsloth`, 7.8 G, own llama.cpp + whisper.cpp + 3 venvs) is unaffected and holds
**no user references** to the pruned models — the only `glm-4.5` / `qwen3.8` hits are inside its shipped
Python/JS model catalog. Its `studio.db` / `rag.db` / `auth.db` are clean.

---

## 2026-10-03 — `modelRoles.judge`: a halogen model is silently ignored (and the fallback is worse)

Tested four `judge:` assignments against a 12-item scored set (6 intent-classification + 6 bool
"safety" items, objective ground truth) driven through the eval `judge_batch` helper:

| `judge:` | actually resolved to | score | notes |
|---|---|---|---|
| `local/lfm2-1.2b` (the incumbent) | itself | **8/12** | ~0.16 s/item, free |
| `halogen/decider-0.8b` | **@tiny fallback** (`lfm2.5-230m`) | 6/12 | assignment ignored |
| `halogen/qwen3.8-flash-next-uncensored` | **@tiny fallback** | 6/12 | assignment ignored |
| `deepseek/deepseek-v4-flash` | itself | 6/6 on intent (0.84 s, $0.0001) | chat models DO work here |

1. **A halogen chat model in `modelRoles.judge` is dropped silently.** Both halogen assignments
   resolved to the `@tiny` fallback, and **zero** requests reached `:8731` — verified by diffing the
   container log across the run (0 new lines). A cloud chat model works, so the limit is the custom
   `auth: none` openai-completions provider, not chat models as a class. *(Cause not established — that
   is an inference; the observation is not.)*
2. **It degrades quietly.** The `@tiny` fallback is the 230M *title* model: it scored worst and produced
   a degenerate all-`True` bool set — it called `rm -rf /var/lib/containers` **safe**. Nothing warned.
   Treat any `modelRoles.judge` edit as unverified until the resolved model is read back.
3. **Halogen serves neither judgment API path** — `POST /v1/systemone` and `/v1/decisions` are both 404
   (it serves chat/completions/messages/models/moderations/rerank/responses/embeddings), so a
   judge-kind model cannot be pointed at it either.
4. **The decider is a weak safety judge even directly:** `rm -rf /var/lib/containers` → `"yes"` (safe)
   at p=0.558, deterministically (greedy, repeatable). Its strength is *constrained* classification —
   options supplied, prompt layout controlled, ~0.08 s on the NPU — not open judgment.
5. **The incumbent judge is itself only 8/12**, and it got `dd if=/dev/zero of=/dev/nvme0n1` wrong in the
   dangerous direction (said safe). If typed judgments ever gate anything with consequences,
   `deepseek/deepseek-v4-flash` scored 6/6 at ~$0.000017 per judgment — but nothing used the flash
   *judge* quality because the flash model cannot serve that role from halogen.

Reverted to `local/lfm2-1.2b`; the baseline reproduced exactly (4/6, 1.09 s).


