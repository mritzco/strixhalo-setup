# Setup — the PX13 journey

How this laptop was configured, what broke, and how it was fixed. Append-only: each chapter
is dated, and history doesn't get rewritten. Current-state material lives in
[`reference/`](#reference), which *is* overwritten.

[← Handbook index](../README.md)

---

## The machine

| | |
|---|---|
| **Model** | ASUS ProArt PX13 (HN7306EA), convertible |
| **CPU/APU** | AMD Ryzen AI Max+ 395 "Strix Halo" · 32 threads |
| **iGPU** | Radeon 8060S (gfx1151), RDNA 3.5 |
| **Memory** | 128 GB unified · VRAM 512 MB + GTT ~110 GB |
| **Swap** | zram 124.9 GB (compressed RAM — see [ch. 8](08-resilience.md)) |
| **Storage** | 950 GB btrfs on LUKS |
| **OS** | CachyOS (Arch-based) · kernel `linux-cachyos-px13` |
| **Desktop** | niri (Wayland) · sddm · Noctalia shell |
| **Shell** | fish |
| **Keyboard** | Japanese physical layout, US keymap + kanata remap |

---

## Chapters

| # | Chapter | Date | Status |
|---|---|---|---|
| 1 | [Base system](01-base-system.md) — packages, configs, system changes | 2026-07-11 | ✅ |
| 2 | [Audio](02-audio.md) — TAS2783 speakers, px13 kernel | 2026-07-12 | ✅ |
| 3 | [Keyboard](03-keyboard.md) — JP remap via kanata, power & battery | 2026-07-11 | ✅ |
| 4 | [Display rotation](04-display-rotation.md) — convertible tablet/tent | 2026-07-19 | ✅ |
| 5 | [Local AI](05-local-ai.md) — GTT memory, Vulkan, llama.cpp, llama-swap | 2026-07-15 | ✅ |
| 6 | [ComfyUI](06-comfyui-rocm.md) — image & video via ROCm toolbox | 2026-07-17 | ✅ |
| 7 | [Apps & viewers](07-apps-viewers.md) — calculator, image, 360° | 2026-08-01 | ✅ |
| 8 | [Resilience](08-resilience.md) — OOM hardening | — | ⚠️ **proposed, not applied** |
| 9 | [DeepSeek-OCR](09-deepseek-ocr.md) — vLLM on ROCm/gfx1151 | 2026-08-07 | ✅ working, limits documented |

## Reference

Current state, safe to overwrite:

- [config-files.md](reference/config-files.md) — contents of every config file touched
- [llm-usage.md](reference/llm-usage.md) — running local LLMs day to day
- [comfyui-usage.md](reference/comfyui-usage.md) — image/video generation day to day
- [niri.md](reference/niri.md) — compositor how-to notes
- [nvim.md](reference/nvim.md) — editor keymaps, treesitter parsers, learning vim



---

## House style

Carried over from how this log was already written, and worth keeping:

- **Code over prose.** Format each item as `doing X => instructions`, name the files to
  modify, and include the actual file contents in fenced blocks. Keep narrative minimal.
- **Date everything.** `— 2026-07-15` in the heading, and mark verification separately from
  the change (`✅ VERIFIED WORKING 2026-07-17`).
- **Record the gotchas, with the symptom.** The most valuable lines in here are the ⚠️ ones:
  they describe a failure you'd otherwise spend an evening rediscovering. Always write the
  *observable symptom*, not just the fix.
- **Keep rollback instructions** for anything touching boot, kernel params, or root config.
- **Don't paste config contents into a chapter.** Put them in
  [reference/config-files.md](reference/config-files.md) and link. Duplicated config is how
  the original log ended up with two copies of `config.fish` disagreeing with each other.
