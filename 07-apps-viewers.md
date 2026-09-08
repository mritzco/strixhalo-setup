# 7. Calculator, image viewers, 360° photo viewer

**Date:** 2026-08-01

[← ComfyUI](06-comfyui-rocm.md) · [Setup index](README.md) · [Next: Resilience →](08-resilience.md)

---

## 9. Calculator + image viewers + 360° photo viewer — 2026-08-01

Before: no calculator at all, no image viewer except darktable, and `image/jpeg`
was opening in **Chromium**. darktable is a RAW darkroom (library/import
workflow) — wrong tool for "just look at this PNG". Keep it for editing only.

### Packages
```sh
sudo pacman -S --needed qalculate-qt loupe
# imv was tried and removed — loupe wins on a touchscreen convertible
# (pinch-zoom / swipe / rotate work in tablet mode; imv is keyboard-only)
```
`qalculate-qt` = Qt6 calculator, does units + currency (`60 km/h to mph`).
It pulls in `libqalculate`, which also gives you the **`qalc`** CLI in Alacritty.
Note Noctalia's launcher already has a built-in JS calculator
(`/etc/xdg/quickshell/noctalia-shell/Modules/Panels/Launcher/Providers/CalculatorProvider.qml`)
for plain arithmetic — qalculate is for units/conversion/algebra.

### Fix default image handler (was chromium.desktop)
```sh
for t in image/jpeg image/png image/webp image/gif image/tiff
    xdg-mime default org.gnome.Loupe.desktop $t
end
xdg-mime query default image/jpeg   # => org.gnome.Loupe.desktop
```

### 360° photos — `~/.local/bin/360view`
The Ricoh Theta shots in `~/Pictures/Theta` (401 files, 5376x2688, exact 2:1)
are **already-stitched equirectangular JPEGs** — no proprietary format, nothing
to stitch. The only missing piece was software that projects them onto a sphere.

`360view` is a self-contained Python script: it boots a throwaway localhost HTTP
server over the image's directory and opens a WebGL viewer in a dedicated
Chromium window (`--app`, own profile at `~/.cache/360view/profile`,
WM class `view360`). No CDN, no npm, no external deps — works offline.

```sh
360view                      # every image in the current directory
360view ~/Pictures/Theta     # that directory
360view R0011799.JPG         # start here, arrows walk the folder
```

Controls: drag to look · scroll/pinch to zoom · `←` `→` photo · `r` reset ·
`a` auto-spin · `f` fullscreen · `q` quit. Touch drag + two-finger pinch work,
so it's usable in tablet mode. Next/prev images are preloaded.

Two implementation details that matter if it ever needs editing:
- It renders a **full-screen triangle** and does the sphere lookup per-pixel in
  the fragment shader, rather than texturing a sphere mesh — no tessellation
  artefacts at the poles.
- The `u` texture coord wraps 0->1 directly behind you. Left alone, that
  discontinuity makes the mip selector pick a huge LOD and paints a bright
  vertical stripe down the seam. Fixed by hand-correcting the derivatives
  (`if (abs(dx.x) > 0.5) dx.x -= sign(dx.x);`) and sampling with `textureGrad`.

Also registered `~/.local/share/applications/360view.desktop` (StartupWMClass=
`view360`) so it appears under "Open with" — deliberately NOT the default
handler, since most JPEGs aren't 360.

### 360 *video* — the real problem was NOT metadata
First diagnosis was wrong and is corrected here. Yes, the Theta MP4s carry no
`sv3d`/`st3d` box and no Google `GSpherical` XML — but injecting that metadata
would NOT have fixed them. Extract a frame and look:

```sh
ffmpeg -ss 6 -i R0011835.MP4 -frames:v 1 frame.png   # => two circles, side by side
```

The videos are raw **dual fisheye** (3840x1920, two 1920x1920 circles), never
stitched. The photos *are* stitched equirectangular; the videos are not. No
player and no metadata could have shown these correctly — the pixels need
unwrapping first. Confirmed against ffmpeg:

```sh
# ground truth for one frame — roll=90 is what makes it upright
ffmpeg -ss 6 -i R0011835.MP4 -frames:v 1 \
  -vf "v360=dfisheye:e:ih_fov=190:iv_fov=190:roll=90" out.png
```

`360view` now does this unwrap **live in the fragment shader**, so nothing is
written to disk — which matters because most of the 360 media is on an external
HDD. Layout is auto-detected per file (`detect_source()`): dual fisheye
letterboxes all eight half-corners to black, equirectangular fills the frame,
and both are 2:1, so aspect ratio alone cannot tell them apart.

Calibrating the back lens was the fiddly part. Mapping each hemisphere onto its
lens axis with `(x,y,-z)` and `(-x,y,z)` looks symmetric but both are
*reflections* flipping *different* axes — so the halves got opposite handedness
and the back hemisphere came out mirrored. Settled it by rendering the sphere as
an equirect map (`?equi=1` debug flag) and scoring PSNR against the ffmpeg
output above, sweeping the unknowns:

```
flipB=-1 rotB=180  ->  27.7 dB   <-- correct
flipB=+1 rotB=180  ->  17.9 dB
flipB=+1 rotB=0    ->  15.8 dB
flipB=-1 rotB=0    ->  15.5 dB
(others 13-14 dB)
```

Packaged for publishing at `~/Projects/360view` (git repo, `main`, no remote
yet — add one once GitHub is sorted on this machine). `install.sh` there
reinstalls it into `~/.local/bin` on any machine.

Extra keys this added: `s` toggles fisheye/equirect if detection guesses wrong,
`[` / `]` nudge the base roll by 5° (Shift = 1°), `0` resets it. The roll is
applied *before* yaw/pitch, so dragging is always relative to the corrected
horizon, and the chosen value persists per source type in localStorage.

```

