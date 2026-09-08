# 3. Keyboard — Japanese layout remap, power & battery

**Date:** 2026-07-11 · updated 2026-07-17 (multi-keyboard filter)

[← Audio](02-audio.md) · [Setup index](README.md) · [Next: Display rotation →](04-display-rotation.md)

---

This laptop has a **Japanese physical keyboard** with the system layout set to **US**. The
extra JP keys emit keycodes the US layout has no keysym for, so they do nothing. Fixed with
**kanata** (v1.11.0, `/usr/bin/kanata`).

Config contents: [reference/config-files.md](reference/config-files.md).

## The yen-key problem

The physical ¥ key emits evdev **keycode 124 = `KEY_YEN`** (confirmed with `evemu-record`:
`EV_KEY / KEY_YEN`, `0x7c`). The US layout has no keysym for 124, so `wev` showed
`NoSymbol` — the key worked at the hardware level and produced nothing.

Kanata's built-in `yen` name was unreliable. **Bind the raw keycode instead:**

```lisp
(defcfg
  process-unmapped-keys yes
)
(deflocalkeys-linux
  yen 124
)
(defsrc
  yen
)
(deflayer base
  bspc
)
```

⚠️ **Every key in `defsrc` must have an entry at the same position in every `deflayer`.**
The counts must match or kanata fails to load. A partial hand-edit that added a `defsrc` key
without its `deflayer` entry caused exactly this failure.

## Current mapping

| Physical key | Keycode | Output | Needs `deflocalkeys`? |
|---|---|---|---|
| Yen ¥ | 124 | `bspc` (Backspace) | ✅ yes |
| RO ろ | 89 | `bksl` (Backslash) | ✅ yes |
| Backslash | 43 | `ret` (Enter) | ❌ standard name |
| Katakana | — | `ralt` | ❌ |
| Muhenkan 無変換 | — | `spc` | ❌ |
| Henkan 変換 | — | `spc` | ❌ |

Only **non-standard** keys (yen, ro) need `deflocalkeys-linux` entries.

**To capture a new dead key:** `systemctl --user stop kanata` first — kanata `EVIOCGRAB`s the
device, so `evemu-record` fails while it's running. Record, then start again.

## Internal keyboard only

The JP remaps must apply **only** to the built-in keyboard; an external US Bluetooth keyboard
must keep its native layout. Without a filter, kanata grabs *all* keyboards.

```lisp
(defcfg
  linux-dev-names-include (
    "AT Translated Set 2 keyboard"
  )
)
```

Internal keyboard = **`AT Translated Set 2 keyboard`** (`/dev/input/event2`). Verified: kanata
now registers only event2 and ignores everything else, including "Asus WMI hotkeys" (event6)
and any Bluetooth keyboard.

Find a device's exact name via `/proc/bus/input/devices` or kanata's startup log
(`journalctl --user -u kanata`).

## Persistence

- Config at `~/.config/kanata/kanata.kbd` — **`kanata.kbd`, not `config.kbd`**. A path
  mismatch caused an early `start-limit-hit` failure.
- User systemd service `~/.config/systemd/user/kanata.service`, enabled with
  `systemctl --user enable --now kanata.service`
- Permissions: `uinput` group created via `groupadd --system uinput`; user in `input` +
  `uinput`. udev rule `/etc/udev/rules.d/99-kanata.rules` for `/dev/uinput`, and
  `/etc/modules-load.d/uinput.conf` autoloads the module.
- Reload after editing: `systemctl --user restart kanata`
- If it isn't active at the login screen or after resume, consider
  `loginctl enable-linger $USER` — user services only run while logged in.

## Power & battery

**Do NOT install TLP.** This system uses **power-profiles-daemon**, driving `amd_pstate` +
`platform_profile` — the correct stack for this AMD chip. TLP would conflict with PPD, and its
charge-threshold feature is flaky on ASUS.

```fish
powerprofilesctl set balanced|power-saver|performance
```

**Charge limit via asusctl** (v6.3.8), set to **80%**:

```fish
asusctl battery limit 80      # subcommand-based in this version, NOT the old `-c 80`
```

- `asusd` is **D-Bus activated** — do **not** `systemctl enable` it. That errors, and it's expected.
- The raw sysfs node `/sys/class/power_supply/BAT0/charge_control_end_threshold` reads
  "No data available" (write-only quirk). asusctl is the reliable interface.
- Kernel exposes `asus_wmi` plus the newer `asus_armoury` firmware-attributes module.

## Known quirk

**`XF86Launch2` (keycode 157) auto-spams continuously** — an `asus-nb-wmi` misfire. Do **not**
bind it. Worth silencing later via an hwdb/keyd remap of the scancode to reserved.
