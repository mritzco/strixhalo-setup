[← Setup index](../README.md)

# ComfyUI (image & video) — Usage

Local image/video generation on the PX13 (Strix Halo, gfx1151) via kyuz0's podman toolbox
(ROCm/TheRock, gfx1151 wheels baked in). Setup details: [setup/06-comfyui-rocm.md](../06-comfyui-rocm.md).

## Start it
```fish
toolbox enter strix-halo-comfyui     # enter the container (drops into bash)
start_comfy_ui                       # launches ComfyUI
```
Then open **http://127.0.0.1:8000** in a browser.
Stop: `Ctrl-C` in the toolbox, then `exit`. (Models/outputs persist on the host.)

> First time in a freshly (re)created toolbox only: run `/opt/set_extra_paths.sh` once.

## Get models
Inside the toolbox:
```bash
python /opt/model_manager.py     # interactive menu
# or direct scripts:
/opt/get_qwen_image.sh           # Qwen-Image (text-to-image)  ← recommended
/opt/get_wan22.sh                # Wan 2.2  (video)
/opt/get_hunyuan15.sh            # Hunyuan 1.5 (video)
/opt/get_ltx2.sh                 # LTX-2 (video)
```
Models land in `~/comfy-models/` (host). Pick BF16 (quality) or FP8 (faster) + the Lightning 4-step LoRA.

## Run a generation
1. **Workflows** sidebar (left rail) → choose a workflow. **Not** the "Getting Started" templates.
   - `Qwen-Image-2512-BF16-4-Step-LoRA` — fast (4-step Lightning) ← start here
   - `Qwen-Image-2512-BF16-20-Steps` — higher quality, slower
   - `Qwen-Image-Edit-…` — edit an existing image
   - `Wan2.2-…`, `LTX2-…`, `Hunyuan-…` — video
2. Edit the prompt in the **CLIP Text Encode (Positive Prompt)** node.
3. Click **Run**.
4. If you just downloaded a model and it's not listed: press **R** (or reload the tab) to rescan.

## Wan 2.2 image-to-video — two working paths
The built-in **Workflow → Browse Templates → Video → "Wan 2.2 Image to Video"** is a
**subgraph** (one node hides the real graph). **Double-click it to open/edit inside**;
the breadcrumb at top takes you back out. Model/LoRA dropdowns may be promoted onto the
outer node — but to change a loader's *type* (e.g. safetensors → GGUF) you must go inside.

Text encoder + VAE are shared by both paths (already installed):
`text_encoders/umt5_xxl_fp8_e4m3fn_scaled.safetensors` · `vae/wan_2.1_vae.safetensors`

**Path A — Base Wan 2.2 (matches the template, fastest ~5–7 min first run):**
- Diffusion high/low → `wan2.2_i2v_high_noise_14B_fp8_scaled` + `..._low_noise_...`
- LoRA high/low → `loras/Wan2.2-I2V-A14B-4steps-lora-rank64-Seko-V1/{high,low}_noise_model.safetensors`
  (template lists `lightx2v` LoRAs; the Seko ones do the same 4-step job — just pick them)
- **Steps 4 · CFG 1 · Euler**

**Path B — Wan-2.2-Remix GGUF (no LoRA needed; accel is baked in):**
- Open the subgraph, replace both **Load Diffusion Model** → **Unet Loader (GGUF)**
- Pick `wan22RemixT2VI2V_i2vHighV30-Q6_K.gguf` (high) + `...LowV30-Q6_K.gguf` (low)
- **Mute both LoRA nodes** (select → `Ctrl+B`) — stacking Lightning LoRAs here causes artifacts
- **Steps 8 · CFG 5 · Euler** (Remix's native settings)

Notes:
- Don't cross the paths: Lightning LoRAs want CFG 1; the Remix wants CFG 5.
- "High-noise / Low-noise" = a required **pair** (early vs late denoise steps), *not* a quality tier.
- A "models missing" warning that still renders = an unused/muted node referencing an absent
  file (usually the template's original `lightx2v` LoRA slots). Fine to ignore if it produces video.
- Mid-render the progress bar can park ~50% while it swaps the high-noise model out for the
  low-noise one (~12–14 GB reload). Normal. GGUF loads via the `ComfyUI-GGUF` custom node.
- GGUF quants live under `diffusion_models/` (scanned fine); pick one High+Low pair only —
  grab specific files with `hf download <repo> <exact/path/file.gguf> --local-dir …`
  (NOT `git clone`, which leaves LFS pointers; NOT `--include` globs, which the CLI ignores).

## Where files go
- **Generated images/videos →** `~/comfy-outputs/`
- **Models →** `~/comfy-models/` (`diffusion_models/`, `text_encoders/`, `vae/`, `loras/`)

## Check it's on the GPU
- Built-in **AMD GPU Monitor** panel in the ComfyUI UI (GPU %, GTT usage).
- Or a 2nd toolbox shell: `toolbox enter strix-halo-comfyui` → `rocm-smi` during a render
  (expect **GPU% ~100%**, GTT climbing to ~40 GB for BF16 20B).

## Speed notes (Strix Halo)
- Diffusion is ~3–4× slower than a big Nvidia dGPU — normal. First render is slowest (model load + compile); later ones are quick.
- **Use 4-step Lightning workflows** for speed; **FP8** models render faster than BF16 (bandwidth-bound GPU).
- Launch flags (already baked into `start_comfy_ui`): `--gpu-only --disable-mmap --bf16-vae --cache-none`.

## Model format on this chip (RDNA 3.5 / gfx1151) — pick FP8 or GGUF, not FP16
- **No native FP8.** ComfyUI logs `emulated ops: float8_e4m3fn` + `manual cast: torch.float16`
  — FP8 (and GGUF quants) are **upcast to FP16 for compute** regardless. You always compute in FP16.
- This GPU is **memory-bandwidth-bound**, so smaller weights = less bandwidth/step = **faster**:
  - **FP8 / GGUF Q5–Q6** = smaller, lighter on RAM, *faster* here. Default to these. ✅
  - **FP16/BF16** = biggest, most bandwidth, *slower* — no compute benefit on this chip. Only for max-quality finals.
  - Quality: `bf16 > fp8 ≈ Q8 > Q6 > Q5 > Q4`; speed is the reverse. **fp8 or Q6_K = sweet spot.**
- Takeaway: tune render time via **resolution + frame count**, NOT by chasing FP8 "acceleration" (there is none).

## Video clip length (Wan) — stay near 81 frames
- Wan 2.2 is trained for **~81 frames** (≈5 s @ 16 fps). Frame counts must be **4n+1** (81, 121, 161…); other values get silently adjusted.
- Cost scales **super-linearly** with frames (attention is O(n²) in sequence length, and there's no
  fast-attention kernel on gfx1151 — logs show `Using split attention`). Real example: **200 frames ≈ 33 min/step, ~4 h total.**
- For longer video: DON'T raise frame count. Instead generate 81 frames and **interpolate fps** (RIFE/FILM),
  or **stitch** clips (last frame of A → input image of B). Lower resolution to iterate faster.
