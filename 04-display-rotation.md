# 4. Screen rotation — convertible tablet / tent mode

**Date:** 2026-07-19 · niri + accelerometer. Scripts in `~/.config/niri/scripts/`.

[← Keyboard](03-keyboard.md) · [Setup index](README.md) · [Next: Local AI →](05-local-ai.md)

---

## 8. Screen rotation (convertible: tablet / tent) — niri — 2026-07-19

**Goal:** auto-rotate in tablet/tent folds + manual rotate + on-screen keyboard.
**Key fact:** PX13 exposes **no `SW_TABLET_MODE` switch** on Linux (only Lid + Asus WMI), and the keyboard is unreachable when folded → accelerometer auto-rotation is the only real path. niri rotates display **and** touch/stylus (ELAN9008) together natively — no `map-to-output` needed.

### Install => sensor feed + on-screen keyboard

```sh
sudo pacman -S --needed iio-sensor-proxy   # D-Bus activated, no enable needed
yay -S wvkbd-git                            # provides the wvkbd-deskintl binary (NOT mobintl)
monitor-sensor                              # sanity: tilt laptop, prints "orientation changed: ..."; Ctrl-C
```

### Scripts => create in `~/.config/niri/scripts/` (chmod +x all)

`niri-autorotate.sh` — accelerometer daemon (orientation → niri transform):
```sh
#!/usr/bin/env bash
OUTPUT="eDP-1"
declare -A MAP=(
  [normal]="normal"
  [bottom-up]="180"    # tent fold
  [left-up]="90"       # swap 90<->270 here if a portrait side is upside down
  [right-up]="270"
)
stdbuf -oL monitor-sensor 2>/dev/null | while IFS= read -r line; do
  case "$line" in
    *"orientation changed: "*)
      o="${line##*orientation changed: }"; t="${MAP[$o]:-}"
      [[ -n "$t" ]] && niri msg output "$OUTPUT" transform "$t" ;;
  esac
done
```

`niri-rotate.sh` — manual cycle normal→90→180→270:
```sh
#!/usr/bin/env bash
OUTPUT="eDP-1"; STATE="${XDG_RUNTIME_DIR:-/tmp}/niri-rotate.state"
seq=(normal 90 180 270)
cur=$(cat "$STATE" 2>/dev/null || echo 0); next=$(( (cur + 1) % 4 ))
echo "$next" > "$STATE"; niri msg output "$OUTPUT" transform "${seq[$next]}"
```

`niri-autorotate-toggle.sh` — start/stop the daemon (turn ON before folding):
```sh
#!/usr/bin/env bash
if pkill -f niri-autorotate.sh; then notify-send "Auto-rotate OFF" 2>/dev/null
else setsid -f "$HOME/.config/niri/scripts/niri-autorotate.sh" >/dev/null 2>&1; notify-send "Auto-rotate ON" 2>/dev/null; fi
```

`niri-osk-toggle.sh` — show/hide on-screen keyboard:
```sh
#!/usr/bin/env bash
OSK="wvkbd-deskintl"
if pkill -x "$OSK"; then :; else setsid -f "$OSK" -L 300 >/dev/null 2>&1; fi
```

### Keybinds => append to `~/.config/niri/cfg/keybinds.kdl` (inside `binds {}`)

```kdl
Mod+Shift+R  hotkey-overlay-title="Rotate screen (cycle)"                { spawn-sh "~/.config/niri/scripts/niri-rotate.sh"; }
Mod+Shift+A  hotkey-overlay-title="Toggle auto-rotate (accelerometer)"   { spawn-sh "~/.config/niri/scripts/niri-autorotate-toggle.sh"; }
Mod+Shift+K  hotkey-overlay-title="Toggle on-screen keyboard"            { spawn-sh "~/.config/niri/scripts/niri-osk-toggle.sh"; }
```

### Optional => always-on auto-rotate: append to `~/.config/niri/cfg/autostart.kdl`

```kdl
spawn-sh-at-startup "~/.config/niri/scripts/niri-autorotate.sh"
```

### Notes
- **Calibration done 2026-07-19:** left-up/right-up were inverted with the initial guess → corrected to `left-up=90 / right-up=270`.
- **Quirk — do NOT bind:** `XF86Launch2` (keycode 157) auto-spams continuously (asus-nb-wmi misfire). Silence later via hwdb/keyd remap of scancode → reserved.

---
