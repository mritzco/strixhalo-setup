# 6. ComfyUI — image & video generation via ROCm

**Date:** 2026-07-17 · podman toolbox + prebuilt gfx1151 image.
Day-to-day usage: [reference/comfyui-usage.md](reference/comfyui-usage.md).

[← Local AI](05-local-ai.md) · [Setup index](README.md) · [Next: Apps & viewers →](07-apps-viewers.md)

---

## 7. Step 3 — ComfyUI / image-gen via ROCm — IN PROGRESS 2026-07-17

**Approach chosen: podman `toolbox` + kyuz0's prebuilt image** `docker.io/kyuz0/amd-strix-halo-comfyui:latest` (gfx1151 wheels + ROCm 7 / "TheRock" baked in). **Why not plain pip:** official ROCm PyTorch wheels DON'T support gfx1151 → "invalid device function" / silent CPU fallback; only special wheels (scottt's, or AMD gfx1151 nightlies, or this prebuilt image) work. Native venv+scottt-wheels was the alternative — rejected as more fragile (needs Python 3.11; we have 3.14) and prone to wheel/ROCm drift. Image is built for **toolbox/podman** (not plain docker); podman coexists with the existing docker.

**Install steps:**
```fish
sudo pacman -S podman toolbox
# if `toolbox create` errors on subuid/subgid:
#   sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 "$USER" && podman system migrate
toolbox create strix-halo-comfyui \
  --image docker.io/kyuz0/amd-strix-halo-comfyui:latest \
  -- --device /dev/dri --device /dev/kfd \
  --group-add video --group-add render --security-opt seccomp=unconfined
toolbox enter strix-halo-comfyui   # enters with bash (no fish in image — harmless)
# --- inside the toolbox ---
/opt/set_extra_paths.sh    # links ComfyUI model dirs -> ~/comfy-models (persists on host)
python /opt/model_manager.py   # download menu; OR curated scripts:
#   /opt/get_qwen_image.sh   (image; open-weight Qwen, has qwen-image-studio workflow)
#   /opt/get_wan22.sh /opt/get_hunyuan15.sh /opt/get_ltx2.sh   (video models)
start_comfy_ui             # launches ComfyUI (real cmd below)
```
- **UI:** http://127.0.0.1:**8000** (toolbox shares host net). Image = **ROCm nightly 7.14.0a / TheRock**, gfx1151.
- **Real launch (the `start_comfy_ui` alias):** `cd /opt/ComfyUI && python main.py --port 8000 --output-directory $HOME/comfy-outputs --disable-mmap --gpu-only --disable-smart-memory --cache-none --bf16-vae`. Outputs → `~/comfy-outputs`; models → `~/comfy-models`; prebuilt workflows in `/opt/comfy-workflows/`, `/opt/qwen-image-studio/`, `/opt/wan-video-studio/`.
- **`--gpu-only`** = errors instead of silently falling back to CPU → a render succeeding *proves* the gfx1151 GPU path works.
- **Strix Halo flags:** `--disable-mmap` (mmap >64 GB is very slow on gfx1151), `--bf16-vae` (avoid VAE OOM), `--cache-none`/`--disable-smart-memory` (aggressive unified-mem mgmt).
- **Perf expectation:** SDXL 1024² ~14–18 s, Flux Schnell ~10 s (~3–4× slower than an RTX 4090; fine for casual/experimental).
- **✅ VERIFIED WORKING 2026-07-17:** loaded `Qwen-Image-2512-BF16-4-Step-LoRA` from the Workflows sidebar and ran a render — `rocm-smi` showed **GPU% 100%**, 59 W, 73 °C (idle was 1% / 37 W). Downloaded: Qwen-Image 20B bf16 + Qwen2.5-VL-7B text encoder + qwen_image VAE + Lightning-4step LoRA. Outputs → `~/comfy-outputs`, models → `~/comfy-models`.
- **Usage cheat-sheets:** [reference/llm-usage.md](reference/llm-usage.md) (LLMs) and [reference/comfyui-usage.md](reference/comfyui-usage.md) (image/video) — light day-to-day references, kept current separately from this append-only chapter.
- Once this ROCm/GPU-container plumbing is proven, **DeepSeek-OCR** (vLLM + ROCm, gfx1151) is the natural follow-on (see §6 DeepSeek-OCR note).
- **Add/change models:** edit `config.yaml` (the model key = the id clients request), restart `llmswap`. For two models loaded at once, use llama-swap `groups` (mind RAM). Optional: run as a systemd user service for always-on.
- Model id the client must request = the server `--alias` of whatever `llm` is running (`qwen3-coder`, `glm-4.5-air`, …). Switching models means updating the client's model — the annoyance **llama-swap** would remove.
Later upgrade for auto-swapping from Open WebUI's model dropdown: **llama-swap** (single Go binary from GitHub releases, not in repos) — a proxy that starts/stops models on demand behind one endpoint.

### Model storage & pre-downloading
- llama.cpp `-hf` caches into the **standard HF cache**: `~/.cache/huggingface/hub/`. `/home` had 858 GB free (2026-07-15).
- **Pre-download without loading** (install `python-huggingface-hub` → `hf`):
  ```
  hf download <repo> --include "*<QUANT>*"
  ```
  e.g. `hf download unsloth/GLM-4.5-Air-GGUF --include "*UD-Q4_K_XL*"`. Same cache → `llm`/`-hf` reuse it, no re-download. Split models arrive as `…00001-of-0000N.gguf` parts.
- **Offline use:** `HF_HUB_OFFLINE=1` env, or launch by path with `-m <file.gguf>` to skip HF metadata checks entirely.

### DeepSeek-OCR (document OCR specialist) — deferred to Step 3
3B VLM with extreme optical-token compression → efficient bulk PDF/document OCR. **Not runnable on packaged `llama-cpp` (b9957):** DeepSeek-OCR-2 needs llama.cpp **PR #20975** (unmerged; OCR support is PR-only, not mainline). Two AMD-native (no-CUDA) paths:
1. **llama.cpp Vulkan, from PR source:** `git fetch origin pull/20975/head:ocr2 && git switch ocr2 && cmake -B build -DGGML_VULKAN=ON && cmake --build build -j`. GGUF `sabafallah/DeepSeek-OCR-2-GGUF` (Q8 ~3 GB) + its mmproj. Run: `llama-mtmd-cli -m deepseek-ocr-2-bf16.gguf --mmproj mmproj-deepseek-ocr-2-bf16.gguf --image <img> -p "..."` (or `llama-server ... --chat-template deepseek-ocr --no-jinja --flash-attn off`).
2. **ROCm + vLLM (PROVEN on identical Strix Halo):** ROCm 7.1.0 + vLLM 0.12.0 built for `gfx1151` (Docker `--build-arg ARG_PYTORCH_ROCM_ARCH=gfx1151`), env `TORCH_ROCM_AOTRITON_ENABLE_EXPERIMENTAL=1` for Flash Attn. Requires the GTT kernel config → **already done in Step 1** ✅. ~2 s/page. This is DeepSeek-OCR **v1**; OCR-2 lists vLLM support too. Ref: yjwong Medium "Running DeepSeek-OCR locally on AMD Strix Halo".

**Plan:** do this inside Step 3 (ROCm/vLLM/ComfyUI) — vLLM is set up there anyway. Use **Qwen3-VL** for OCR until then.

---

