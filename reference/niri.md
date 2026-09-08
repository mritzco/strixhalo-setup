[← Setup index](../README.md)

# niri configurations

Notes on configuring the niri Wayland compositor on this PX13.

Config lives in `~/.config/niri/`:
- `config.kdl` — top-level, just `include`s the files under `cfg/`
- `cfg/display.kdl` — output/monitor layout
- `cfg/keybinds.kdl`, `cfg/input.kdl`, `cfg/layout.kdl`, etc.

niri **hot-reloads on save** — editing any config file takes effect instantly, no logout needed. KDL nodes prefixed with `/-` are disabled/commented out.

---

## Monitor position (swapping displays)

There is no GUI arrangement tool (unlike GNOME/KDE Settings). Output layout is defined in `cfg/display.kdl`.

### List connected outputs
```fish
niri msg outputs
```
Shows each output's name (e.g. `DP-8`, `DP-10`, `eDP-1`), current mode, and `Logical position`.

### Persistent layout — `cfg/display.kdl`
Pin explicit positions per output. `position x= y=` is the top-left corner in the global logical space; put a 1920-wide monitor at `x=1920` to place it immediately to the right of one at `x=0`.

```kdl
// Two Dell S2421HN (1920x1080). Swapped so DP-10 is on the left, DP-8 on the right.
output "DP-10" {
    mode "1920x1080@60.000"
    scale 1
    position x=0 y=0
}

output "DP-8" {
    mode "1920x1080@60.000"
    scale 1
    position x=1920 y=0
}
```

To swap sides again, just exchange the two `position` lines and save (hot-reload applies immediately). Useful when both monitors are identical models and DP-8/DP-10 map to the "wrong" physical panel.

### Quick runtime swap (non-persistent, lost on reload/reboot)
```fish
niri msg output DP-10 position set 0 0
niri msg output DP-8 position set 1920 0
```

### Current layout on this machine
- `DP-10` → left (`0,0`)
- `DP-8` → right (`1920,0`)
- `eDP-1` (laptop panel) → far right (`3840,0`)

---

## External keyboard — Ducky OK-M-65 (`DKOKM65 BT3`)

This is a 65% Bluetooth keyboard. kanata is set to remap **only** the internal
`AT Translated Set 2 keyboard`, so the Ducky passes through with its native
firmware layout (see kanata memory note). Its quirks are handled on the keyboard
itself, not in Linux config.

### Top row acting as F1–F12 instead of numbers
The top row has 3 firmware modes: **number keys → function keys (F1–F12) → multimedia**.
Cycle with:
```
Left Ctrl + Fn
```
Press until numbers return (may pass through multimedia mode on the way). Setting
is stored on the keyboard and persists.

### Esc / backtick
Number-mode top row spans "Esc to =", so the top-left key is **Esc**. Backtick/tilde
(`` ` ~ ``) is not printed on the top row — try **Fn + Esc** for it.

### Other combos (from the manual)
- `Fn + Backspace` — lighting effect/color switch; hold 3s to reset lighting
- `Win/Mac key + Fx` — multimedia functions
- `Fn + C + Delete` (hold 3s) — factory reset

Manual: https://manuals.plus/ducky/ok-m-65-mechanical-keyboard-manual

Fallback: if the hardware layer isn't enough, add `"DKOKM65 BT3 Keyboard"` to
kanata's `linux-dev-names-include` and define a corrective layer.
