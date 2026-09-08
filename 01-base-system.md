# 1. Base system — packages, configs, system changes

**Audit date:** 2026-07-11 · reconstructed from fish history, `/var/log/pacman.log`, and on-disk config.
Point-in-time review of what was installed and what should stay. Config file *contents* live in
[reference/config-files.md](reference/config-files.md).

[← Setup index](README.md) · [Next: Audio →](02-audio.md)

---

## 1. Packages installed

### From official repos
| Package | Purpose | Keep? |
|---|---|---|
| `asusctl` | ASUS control (battery charge limit, profiles) | ✅ Keep |
| `firefox-developer-edition` | Browser | ✅ Keep |
| `openssh` | SSH client **and server** (`sshd`) | ⚠️ See §4 |
| `rsync` | File sync | ✅ Keep |
| `fnm` | Node version manager (active — see `config.fish`) | ✅ Keep |
| `fzf` | Fuzzy finder (used by your `gossh` function) | ✅ Keep |
| `gnome-keyring` | Secret storage (VS Code password-store) | ✅ Keep |
| `neovim` | Editor | ✅ Keep (your main editor) |
| `vim` | Editor | 🟡 Redundant with neovim — optional removal |
| `wev` | Wayland event viewer (used to debug the yen key) | 🟡 Debug tool — safe to remove |
| `base-devel`, `git` | Build toolchain (needed for AUR/paru) | ✅ Keep |

### From AUR (`pacman -Qm`)
| Package | Purpose | Keep? |
|---|---|---|
| `paru-bin` | AUR helper | ✅ Keep |
| `kanata` | Keyboard remapper (yen→Backspace, thumb keys) | ✅ Keep |
| `visual-studio-code-bin` | VS Code (owns `/usr/bin/code`) | ✅ Keep |

### Removed already (clean)
- `nvm` — replaced by `fnm`. `~/.nvm` is gone, no references in `config.fish`. ✅ Fully cleaned.
- `rust`, `zlib`, `lib32-zlib`, `limine-entry-tool` — removed as part of normal dependency/system churn.

### Attempted but NOT installed
- `fish-bass` — the install command ran but the package is not present (wrong name / failed). Nothing to clean.

---

## 2. Config files edited

| File | What changed |
|---|---|
| `~/.config/kanata/kanata.kbd` | Remaps via `deflocalkeys-linux` + layers: Yen(124)→Backspace, RO(89)→Backslash `\`, Backslash(43)→Enter, Katakana→RAlt, Muhenkan/Henkan→Space. `process-unmapped-keys yes`. |
| `~/.config/systemd/user/kanata.service` | User service to autostart kanata (points at `kanata.kbd`, `Restart=on-failure`, `RestartSec=1`). |
| `~/.config/fish/config.fish` | `$HOME/.local/bin` on PATH; `fnm env --use-on-cd`; `gossh` fzf-based SSH picker. |
| `~/.config/niri/config.kdl`, `~/.config/niri/cfg/input.kdl` | niri compositor / input config. |
| `~/.config/alacritty/alacritty.toml` | Terminal config. |
| `~/.local/share/applications/code.desktop` | VS Code launcher forced to `--password-store="gnome-libsecret"` (keyring integration). |
| `~/.vscode/argv.json` | Also sets `password-store: gnome-libsecret`; crash reporter disabled. |

---

## 3. System-level changes (root)

| Change | Location | Notes |
|---|---|---|
| uinput udev rule | `/etc/udev/rules.d/99-kanata.rules` | Grants `uinput` group access to `/dev/uinput` (kanata needs it). ✅ Keep. |
| uinput autoload | `/etc/modules-load.d/uinput.conf` | Loads `uinput` module at boot. ✅ Keep. |
| `uinput` group | `groupadd --system uinput` | You are a member (`input`, `uinput`). ✅ Keep. |
| Battery charge limit | `asusctl battery limit 80` | Persisted by `asusd` (D-Bus activated). ✅ Keep. |

### Runtime-only (NOT persisted — nothing to clean)
- `echo 1 > /sys/module/atkbd/parameters/softraw` — a debugging experiment, resets on reboot.
- `modprobe uinput`, `udevadm` reloads — one-off; the config files above make them permanent.

---

## 4. ⚠️ Recommendations — decisions to make

### SSH server (`sshd`) — KEEP (decided)
`sshd.service` is enabled and listening on `0.0.0.0:22` / `[::]:22`. **Intentionally kept** — user rsyncs and SSHes *into* this laptop from other machines. No action.
- Optional hardening (not required): in `/etc/ssh/sshd_config.d/*.conf` set `PasswordAuthentication no` (key-only) if all your clients use SSH keys.

### LOW — Redundant / leftover
| Item | Action | Command |
|---|---|---|
| `vim` vs `neovim` | **Keep both for now** — user's primary is vim, trialing nvim; remove nvim later once comfortable, or vim if switching. | — |
| `wev` (debug tool, done its job) | Optional remove | `sudo pacman -Rns wev` |

### Housekeeping
- Find other orphaned dependencies: `pacman -Qdtq` (lists them; remove with `sudo pacman -Rns $(pacman -Qdtq)` — review first).
- `bluetooth.service` is enabled — normal, keep unless you never use Bluetooth.

---

## 5. What's healthy (no action)
- **Power management:** `power-profiles-daemon` + `amd_pstate` — correct stack; TLP correctly avoided.
- **Node:** `fnm` only; `nvm` fully removed.
- **Keyboard:** kanata autostarts via user service; uinput permissions persistent.
- **Battery:** charge limit 80% via asusctl.


---

