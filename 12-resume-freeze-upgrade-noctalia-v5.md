# 12. Resume freeze → full upgrade → Noctalia v5

**Date:** 2026-10-19 · **Status:** ✅ desktop back on Noctalia v5, audio PASS on 7.2.10 · ⏳ freeze not reproduced yet

[← Session lock](11-session-lock.md) · [Setup index](README.md)

---

One morning, three problems in a row:

1. the machine froze while waking from an overnight sleep;
2. the system was ~570 packages behind, and the upgrade that followed broke the shell;
3. CachyOS moved `noctalia-shell` to **v5**, so the v4 (Quickshell) desktop had to be migrated.

## 1. The resume freeze — 2026-10-19 10:15

**Symptom:** woke the laptop, the screen went a dark red never seen before, the password was
accepted, then the machine froze completely. REISUB did nothing. Held power for 15 s. The next boot
dropped to **emergency mode** and froze too. The boot after that came up normally.

**Timeline** (from `journalctl -b -1`, crashwatch from [ch. 10](10-crash-forensics.md)):

| Time | Event |
|---|---|
| 10-18 23:00:34 | lid closed → `PM: suspend entry (s2idle)` · last heartbeat healthy (121 GB available, 53 °C) |
| 10-19 10:15:23 | lid opened **+ power key** → resume … `suspend requested from client PID 3287 ('niri')` → straight back to sleep |
| 10:15:34 | power key → resume → niri suspends **again** |
| 10:15:48 | third resume stays up: GPU, USB hub, Wi-Fi all come back |
| **10:15:53** | **journal ends.** No panic, no amdgpu error, no lockup trace |
| 10:28:43 | good boot: `crashwatch-bootcheck: UNCLEAN previous boot stop`, fsck clears the ESP dirty bit |

Every resume also logged `amd_pmc: Last suspend didn't reach deepest state`.

**Reading it** (see the evidence table in ch. 10):

- The lockup detector **was** armed (`nmi_watchdog=1 hardlockup_panic=1 panic=10`). The machine
  did not reboot on its own, so this wasn't a kernel panic or a CPU hard lockup. It was a hang
  below the kernel, or one the kernel doesn't panic on: display, GPU or firmware stuck after resume.
- `crash-forensics.sh` turned up nothing: no pstore, BERT, MCE or SMART errors (NVMe `Unsafe Shutdowns: 2`).
- The emergency-mode boot left **no journal** (the journal lives on the encrypted root and was never written), so its cause is unknown.
- This differs from the 2026-09-24 reset: that one was a spontaneous reboot under memory pressure. This was a freeze right after resume with memory idle.

**Likely trigger:** three suspend/resume cycles in 25 s, because the power key used to wake the laptop
is also niri's *suspend* key.

### Fix: stop niri from handling the power key

`~/.config/niri/cfg/input.kdl`, inside `input { }`:

```kdl
    disable-power-key-handling // Let logind handle the power key (niri's suspend-on-press caused a resume loop + freeze, 2026-10-19)
```

```fish
niri validate      # niri hot-reloads the config
```

⚠️ **With this line, a short power-key press goes to logind, whose default is `poweroff`.**
`/etc/systemd/logind.conf` has no override. **Wake the laptop with the lid or a key**, or add a drop-in:

```ini
# /etc/systemd/logind.conf.d/10-power-key.conf   (not applied yet)
[Login]
HandlePowerKey=suspend
HandlePowerKeyLongPress=poweroff
```

### Still open

- `amd_s2idle.py` (AMD's script) to find what blocks the deepest sleep state
- `kernel.softlockup_panic=1` in `/etc/sysctl.d/90-lockup-panic.conf` (catches more hangs)
- BIOS: running `HN7306EAC.305` (2026-01-17). Check ASUS for something newer
- `rasdaemon`: install after the upgrade (`yay -S rasdaemon && sudo systemctl enable --now rasdaemon`) so MCEs survive a reset

## 2. The upgrade (~570 packages) — 2026-10-19

### Running `crash-forensics.sh` from Claude Code

```fish
# `! cmd` in Claude Code runs zsh with no TTY: sudo can't prompt, and fish's (date) syntax fails.
# Run it in a real terminal instead:
sudo bash ~/crash-forensics/crash-forensics.sh > ~/crash-forensics/forensics-latest.txt 2>&1
```

### paru broken, yay fine

```
paru: error while loading shared libraries: libalpm.so.15: cannot open shared object file
```

`paru-bin` (prebuilt, AUR) was linked against `libalpm.so.15`, and pacman moved to `.so.16`. yay was built against 16.
Fix: replace it with CachyOS's own build:

```fish
sudo pacman -S cachyos/paru     # answer yes to removing paru-bin
```

### Upgrade, not a single install

Running bare `yay` did `pacman -Sy` (refreshed the package lists, upgraded nothing). That is a **partial-upgrade
state**: don't install single packages from there. Do the full upgrade:

```fish
sudo pacman -Syu        # snap-pac takes a pre/post snapper snapshot -> bootable from the Limine menu
yay -Sua                # AUR packages
```

Notable bumps: `linux-cachyos` 7.2.3 → 7.2.10, `linux-firmware(-amdgpu)` / `amd-ucode` 20260810 → 20260916,
`mesa` 26.2.2 → 26.2.4, `systemd` 261 → 262, `qt6-base` 6.11 → 6.12, `noctalia-shell` 4.7.7 → **5.3.0**.

⚠️ **DKMS error on the LTS kernel is expected:**

```
Error! Bad return status for module build on kernel: 6.18.55-2-cachyos-lts
tas2783-sdw.c:1339:31: error: implicit declaration of function 'sdca_regmap_write_init'
```

The audio module ([ch. 2](02-audio.md)) is the 7.x driver and can't build on 6.18. It built fine for
`7.2.10-1-cachyos`. To silence it on future LTS updates, add `BUILD_EXCLUSIVE_KERNEL_MIN="7.1"` to
`/usr/src/snd-soc-tas2783-sdw-px13-1.2/dkms.conf` (and to the copy in `~/px13-audio-fix`). Not applied yet.

### After the reboot

```fish
uname -r                                  # 7.2.10-1-cachyos
dkms status | grep px13                   # installed for 7.2.10
bash ~/px13-audio-fix/check-audio.sh      # all PASS
```

⚠️ Right after login `check-audio.sh` said `FAIL speaker sink audible -> PipeWire shows no SoundWire speaker sink`.
A minute later it passed. WirePlumber simply hadn't finished starting. **Re-run before debugging.**

## 3. Black desktop → Noctalia v5

**Symptom:** after the reboot, all screens black: no bar, wallpaper or launcher. niri keybinds worked
(terminal opened), Noctalia ones didn't.

```fish
pgrep -a qs                     # nothing
qs -c noctalia-shell
#  WARN: noctalia-qs was built against Qt 6.11.2 but the system has updated to Qt 6.12.0 ...
#  Could not find "noctalia-shell" config directory in any valid config path.
ls /etc/xdg/quickshell/noctalia-shell/     # meson.build src/ flake.nix ... no QML
```

CachyOS's `noctalia-shell` 5.3.0 package no longer contains the v4 QML shell. It ships the **v5 source tree**.
v5 is a native C++ shell launched as `noctalia` (package `noctalia`, already installed). Upstream FAQ:
*"v5 is a fresh install … v4 settings are not migrated"*.

### Stopgap: back to v4 (worked, but degraded)

```fish
sudo pacman -U /var/cache/pacman/pkg/noctalia-shell-4.7.7-3-any.pkg.tar.zst
# /etc/pacman.conf, [options]:   IgnorePkg = noctalia-shell
niri msg action spawn -- qs -c noctalia-shell
```

⚠️ v4 then rendered **gray, with black squares where icons go (visible only on hover)**. That's the Qt
6.11 → 6.12 mismatch in `noctalia-qs`. That's why we migrated.

### Migration — doing it => steps

| v4 | v5 |
|---|---|
| `~/.config/noctalia/settings.json` (JSON) | `~/.config/noctalia/config.toml` (yours) + `~/.local/state/noctalia/settings.toml` (GUI overrides, win) |
| `qs -c noctalia-shell` | `noctalia` |
| `qs -c noctalia-shell ipc call …` | `noctalia msg …` |
| QML plugins in `~/.config/noctalia/plugins/` | Luau plugins, `[plugins] enabled`, exported to `~/.local/state/noctalia/plugins/materialized/` |

**Config:** write `~/.config/noctalia/config.toml`, translated from `settings.json`. Contents are in
[reference/config-files.md](reference/config-files.md#confignoctaliaconfigtoml). Then:

```fish
noctalia config validate        # ✓ Config is valid  (hot-reloaded on save)
```

The full default config with every key: `example.toml` inside the 5.3.0 package. The docs ship there too:

```fish
mkdir -p /tmp/v5 && bsdtar -xf /var/cache/pacman/pkg/noctalia-shell-5.3.0-1-any.pkg.tar.zst -C /tmp/v5 \
  etc/xdg/quickshell/noctalia-shell/docs etc/xdg/quickshell/noctalia-shell/example.toml
ls /tmp/v5/etc/xdg/quickshell/noctalia-shell/docs/user/      # ipc/ bar/widgets/ plugins/ services/ ...
```

**niri:** back up first (`cp -a ~/.config/niri/cfg ~/.config/niri/cfg.bak-v4-2026-10-19`), then:

```fish
cd ~/.config/niri/cfg
sed -i 's|spawn-sh-at-startup "qs -c noctalia-shell"|spawn-at-startup "noctalia"|' autostart.kdl
sed -i \
 -e 's|qs -c noctalia-shell ipc call launcher toggle|noctalia msg panel-toggle launcher|' \
 -e 's|qs -c noctalia-shell ipc call lockScreen lock|noctalia msg session lock|' \
 -e 's|qs -c noctalia-shell ipc call sessionMenu toggle|noctalia msg panel-toggle session|' \
 -e 's|qs -c noctalia-shell ipc call volume increase|noctalia msg volume-up|' \
 -e 's|qs -c noctalia-shell ipc call volume decrease|noctalia msg volume-down|' \
 -e 's|qs -c noctalia-shell ipc call volume muteOutput|noctalia msg volume-mute|' \
 -e 's|qs -c noctalia-shell ipc call volume muteInput|noctalia msg mic-mute|' \
 -e 's|qs -c noctalia-shell ipc call media next|noctalia msg media next|' \
 -e 's|qs -c noctalia-shell ipc call media previous|noctalia msg media previous|' \
 -e 's|qs -c noctalia-shell ipc call media playPause|noctalia msg media toggle|' \
 -e 's|qs -c noctalia-shell ipc call brightness increase|noctalia msg brightness-up|' \
 -e 's|qs -c noctalia-shell ipc call brightness decrease|noctalia msg brightness-down|' \
 keybinds.kdl
sed -i 's|^qs -c noctalia-shell ipc call lockScreen lock$|noctalia msg session lock|' ../scripts/lock-before-sleep.sh
grep -rn 'qs ' . ../scripts      # must be empty
```

Append to `rules.kdl` (from `/etc/skel/.config/niri/cfg/rules.kdl`):

```kdl
window-rule {
    match app-id="dev.noctalia.Noctalia"
    open-floating true
    default-column-width { fixed 1080; }
    default-window-height { fixed 920; }
}
```

**Swap the running shell:**

```fish
niri validate && pkill -x qs; niri msg action spawn -- noctalia
noctalia msg status             # {"barVisible": true, ...}
```

**Plugins**: v4 plugin → v5 equivalent:

| v4 | v5 | bar widget `type` |
|---|---|---|
| todo | `nightwatch75/todo` | `nightwatch75/todo:todo` |
| sticky-notes | `ahmedhossamdev/sticky-notes` | `…:bar` |
| pomodoro | `thepunkoff/pomodoro` | `…:widget` |
| mini-docker | `8bury/mini-docker` | `…:mini-docker` |
| keybind-cheatsheet | `kenn/keybind-cheatsheet` | `…:keybinds` |
| screen-recorder | `noctalia/screen_recorder` (official) | `…:recorder` |
| battery-and-power-management | `piero-93/battery-power-management` | `…:widget` |
| obsidian-provider | `davemhammer/obsidian` | `…:status` (not on bar) |
| ssh-sessions | `cleboost/ssh-launcher` | none (launcher provider) |
| polkit-agent | built in: `[shell] polkit_agent = true` | |
| noctalia-calculator | built in: launcher `calculator` provider | |
| vscode-provider, osk-toggle, show-keys, niri-screensaver widget | **no v5 equivalent** (screensaver still runs via `[idle.behavior.screensaver]`) | |

```fish
noctalia msg plugins list | grep enabled
noctalia msg plugins enable nightwatch75/todo      # ... one per plugin; also listed in config.toml [plugins]
# entry ids for the bar: [[widget]] blocks in
#   ~/.local/state/noctalia/plugins/materialized/<source>/<plugin>/plugin.toml
```

**Todos:** the v5 todo plugin uses a different file: `~/Documents/Todo/todo.json` (`todo_folder` setting).
v4's `~/.config/noctalia/plugins/todo/settings.json` is now frozen. Converted once:

```fish
mkdir -p ~/Documents/Todo; python3 -I -c '
import json, pathlib
src = json.load(open(pathlib.Path.home()/".config/noctalia/plugins/todo/settings.json"))
tasks = [{"id": i+1, "text": t["text"], "priority": t["priority"], "done": t["completed"]} for i, t in enumerate(src["todos"])]
(pathlib.Path.home()/"Documents/Todo/todo.json").write_text(json.dumps({"tasks": tasks, "sort": "priority", "hideCompleted": False, "showCount": True}, indent=2))'
```

✅ VERIFIED WORKING 2026-10-19: bar and plugin widgets render. ⏳ Not yet confirmed: media/volume/brightness keys, lock over a real (undocked) suspend.

## ⚠️ Gotchas

- **`noctalia msg …` → `error: read() failed: Resource temporarily unavailable`**, shell process idle at 0 % CPU.
  Happened right after enabling 9 plugins at once (they download in the background). `kill` + respawn fixed it:
  `pkill -x noctalia; niri msg action spawn -- noctalia`.
- **First v5 start opens a setup wizard** (`activePanelId: "setup-wizard"`). Close it; `config.toml` already sets everything.
- **Bar has wide side margins** → v5 default `margin_ends = 180`. Set `margin_ends = 4` in `[bar.main]`.
- **Lid close with the external monitor connected doesn't lock.** That's correct: logind treats the
  laptop as *docked* (`HandleLidSwitchDocked=ignore`), so nothing suspends and `lock-before-sleep.sh` never
  runs. The journal shows only `Lid closed.` / `Lid opened.`. To test the lock, unplug the monitor or run `systemctl suspend`.
- The **v5 docs disappear** when you downgrade to 4.7.7 (they ship only in the 5.3.0 package). Extract them from the cache (above).
- `IgnorePkg = noctalia-shell` stays in `/etc/pacman.conf` while v4 is kept as a fallback. Once you're done with v4:
  `sudo pacman -Rns noctalia-shell noctalia-qs`, then delete the `IgnorePkg` line.

## Rollback

```fish
# back to v4 (needs the pinned 4.7.7 + a noctalia-qs rebuilt for the current Qt, or it renders gray):
cp -a ~/.config/niri/cfg.bak-v4-2026-10-19/. ~/.config/niri/cfg/
cp ~/.config/niri/scripts/lock-before-sleep.sh.bak-v4 ~/.config/niri/scripts/lock-before-sleep.sh
pkill -x noctalia; niri msg action spawn -- qs -c noctalia-shell
# v4 settings.json / plugins/ were never touched.

# undo the power-key change: delete the disable-power-key-handling line from input.kdl

# whole-system: pick the pre-upgrade snapper snapshot in the Limine boot menu
```

## See also

- [ch. 10](10-crash-forensics.md): crashwatch + forensics script that produced the timeline
- [ch. 11](11-session-lock.md): lock-before-sleep (now calls `noctalia msg session lock`)
- [ch. 2](02-audio.md): DKMS audio module
- [reference/config-files.md](reference/config-files.md): `config.toml`, lock script
