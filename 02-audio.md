# 2. Audio — internal speakers (TAS2783 SoundWire amps)

**Current fix:** 2026-09-24 (stock kernel ≥ 7.1 + `px13-audio-fix`) · **Previous fix:** 2026-07-12 (px13 kernel + quirks, kept below as fallback)

[← Base system](01-base-system.md) · [Setup index](README.md) · [Next: Keyboard →](03-keyboard.md)

---

## Solution A (current): stock kernel + `px13-audio-fix`

### TL;DR (the short path)

```fish
sudo pacman -S --needed dkms
git clone https://github.com/ftoleedo/px13-audio-fix.git ~/px13-audio-fix
cd ~/px13-audio-fix
# apply the local patch first - without it the installer exits 1 silently (see "Local patch")
bash install-durable.sh
sudo /usr/local/lib/px13-soundwire-recover.sh   # load the new module (or reboot)
bash ~/px13-audio-fix/check-audio.sh            # must be all PASS
```

TI's rewritten TAS2783 driver landed in mainline **7.1**, so the patched-kernel route is no
longer needed. What upstream still misses is userspace-shaped:

1. `alsa-ucm-conf` ships **no** tas2783 UCM config → on 7.2 the card's UCM cannot open at all
   (`failed to import hw:1 use case configuration`) → Dummy Output.
2. The driver leaves **both amps on the same DSP channel** → mono from one speaker (not fixed
   in 7.3-rc1).
3. s2idle kills the audio stack (SoundWire slaves drop off the bus, and the amp DSP loses its
   firmware → silence with every mixer looking correct).

`px13-audio-fix` supplies the missing UCM files + a small DKMS module with a
`tas2783-N Channel Playback` control, and an optional suspend-resume recovery hook.

| Piece | Where |
|---|---|
| DKMS module (stock 7.3 driver + `Channel Playback`) | `snd-soc-tas2783-sdw-px13/1.2` → `/lib/modules/<kver>/updates/dkms/snd-soc-tas2783-sdw.ko.zst` |
| UCM long-name override | `/usr/share/alsa/ucm2/conf.d/amd-soundwire/ASUSTeKCOMPUTERINC.-ProArtPX13HN7306EAC-1.0-HN7306EAC.conf` |
| UCM codec files | `/usr/share/alsa/ucm2/{sof-soundwire/tas2783.conf,codecs/tas2783/init.conf}` |
| Health check | `bash ~/px13-audio-fix/check-audio.sh` |
| Suspend recovery | `/usr/lib/systemd/system-sleep/50-px13-soundwire` → `/usr/local/lib/px13-soundwire-recover.sh`, log `/var/log/px13-soundwire-resume.log` |

### Install

```fish
sudo pacman -S --needed dkms          # required, else it is a manual build that breaks at the next kernel update
git clone https://github.com/ftoleedo/px13-audio-fix.git ~/px13-audio-fix
cd ~/px13-audio-fix
bash install-durable.sh               # run as YOUR user (it sudos itself); output was tee'd to
                                      # ~/crash-forensics/install-durable.log
bash install-resume-recovery.sh       # optional: s2idle recovery hook
```

### Local patch — reapply after every `git pull`

The installer as shipped dies **silently** on this machine (exit 1, nothing installed, no error
message) because the quirks package's UCM ctl-remap makes `amixer -c N info` exit 1, and
`COMPONENTS="$(px13_card_components "$CARD")"` is unguarded under `set -euo pipefail`.

| File | Change |
|---|---|
| `lib/px13-detect.sh` → `px13_card_components()` | `amixer -D hw:"$1" info` and end the pipeline with `|| true` |
| `lib/px13-detect.sh` → `px13_card_longname()` | `amixer -D hw:"$1" info` |
| `install-durable.sh` | `COMPONENTS="$(px13_card_components "$CARD" || true)"` |

Rule of thumb from the project itself: **always `amixer -D hw:N`, never `amixer -c N`** — the
plain `-c` view goes through the UCM ctl remap.

### Verify

```fish
bash ~/px13-audio-fix/check-audio.sh                       # all PASS
modinfo -k (uname -r) snd_soc_tas2783_sdw -F filename      # .../updates/dkms/... (NOT .../kernel/...)
alsaucm -c 1 list _devices/HiFi | grep Speaker             # prints "Speaker"
amixer -D hw:1 cget name='tas2783-1 Channel Playback'      # values=1  (Left)
amixer -D hw:1 cget name='tas2783-2 Channel Playback'      # values=2  (Right)
pactl list sinks short                                     # ...amd_sdw.HiFi__Speaker__sink
speaker-test -D pulse -c2 -l1 -t wav                       # voice from left, then right
```

**Right after installing, the new module is not loaded yet** — the card works, but there is no
channel assignment. `check-audio.sh` reports it:

```
WARN  module in memory (…) is not the one on disk (…) - a newer build is installed but not loaded
FAIL  stereo channel assignment
```

```fish
sudo /usr/local/lib/px13-soundwire-recover.sh   # ~30 s, or just reboot
```

### Gotchas

- Run `check-audio.sh` after **every** kernel update. If DKMS fails to rebuild, the stock module
  loads and you silently get mono.
- A driver swap under a live WirePlumber can restore the sink at **0 %** volume: check with
  `pactl get-sink-volume @DEFAULT_SINK@`.
- On kernels < 7.3 the suspend hook runs `PX13_RECOVER_POLICY=always`: full stack reload +
  PipeWire restart after every resume (~30 s, and it can take the whole PipeWire graph with it).
  **Leave it at `always` on 7.2.x.** `auto` (the no-op-on-healthy-resume policy) is only the
  default *and only safe* on ≥ 7.3, where the driver re-initialises the amps and re-applies the
  channel assignment by itself; below that, a PCM left open across s2idle is never re-prepared
  and you get silent speakers. Moving to ≥ 7.3 is what buys the fast resume.
- The three `/usr/share/alsa/ucm2/…` files are **also owned by `asus-proart-px13-quirks`**. A
  `pacman -Syu` of that package overwrites them and breaks stock-kernel audio. Only keep the
  quirks package while you still want the px13 kernel entry.
- `/etc/px13-audio-fix.conf` caches the ACP PCI address + long name for the recovery script.

### Maintenance runbook (the five things easy to forget) — 2026-09-24

| When | Do | Symptom if you skip it |
|---|---|---|
| After **every kernel update** | `bash ~/px13-audio-fix/check-audio.sh` | DKMS rebuild fails silently → stock module loads → **mono from one speaker** |
| Any `pacman -Syu` | nothing — `IgnorePkg = asus-proart-px13-quirks linux-cachyos-px13` in `/etc/pacman.conf` holds them back | a quirks update overwrites the three UCM files → **Dummy Output** on stock kernels; its `alsa-ucm-conf=1.2.16.1` pin also blocks other upgrades |
| After `git pull` in `~/px13-audio-fix` | re-apply the 3-line patch (see *Local patch* above) | installer prints `==> 0/8` and exits 1 having installed **nothing**, with no error message |
| After every suspend/resume | nothing — the sleep hook recovers in ~15–30 s | no speakers → `sudo /usr/local/lib/px13-soundwire-recover.sh` |
| When you drop the px13 fallback | delete the `IgnorePkg` line, `sudo pacman -R asus-proart-px13-quirks`, flip `BOOT_ORDER` | the `alsa-ucm-conf` pin keeps blocking upgrades forever |

Confirm the ignore list is really in effect — dry run, nothing gets installed:

```fish
sudo pacman -Sy      # refresh the databases only
sudo pacman -Spu     # print the upgrade plan; px13 + quirks must NOT be in it
```

`IgnorePkg` only affects `-Su`: an explicit `sudo pacman -S asus-proart-px13-quirks` still
installs it, if you ever deliberately want to refresh the old setup.

**Suspend verified 2026-09-24** — `systemctl suspend` → wake: `/var/log/px13-soundwire-resume.log`
shows the stack reloaded, all codecs `Attached`, PipeWire restarted and the Speaker sink back as
default (`recover: SUCESSO …`) in ~16 s. Keep browsers closed around a resume, or restart them —
they keep their old audio-service connection and will report "microphone not found"
(`brave://restart`).

### Rollback to the previous solution

```fish
cd ~/px13-audio-fix && bash uninstall-durable.sh       # module + UCM files + udev rule
sudo bash ~/crash-forensics/restore-quirks-ucm.sh      # restores the quirks UCM files
# reboot → pick the linux-cachyos-px13 entry
```
Backups + md5s of the pre-migration UCM files: `~/crash-forensics/audio-backup/` (`BEFORE.txt`).

### Default kernel - switched 2026-09-24

```fish
# /etc/default/limine
BOOT_ORDER="*linux-cachyos, *px13, *lts, *fallback, Snapshots"
```
```fish
sudo limine-update && sudo limine-list      # linux-cachyos first, px13 second
```

Pattern note: `BOOT_ORDER` is tried in order, **first match wins**, and the first entry becomes the
preselected one. `*linux-cachyos` cannot match `linux-cachyos-px13` or `-lts` (they do not end with
it), whereas a bare `*` would also match px13 and defeat the purpose.

Verified after a reboot: `uname -r` -> `7.2.3-1-cachyos`, audio all PASS, `crashwatch` active,
`linux-cachyos-px13` still selectable as the fallback entry.
Revert: `sudo cp /etc/default/limine.bak-pre-default-flip /etc/default/limine && sudo limine-update`.

---

## Solution B (previous): `linux-cachyos-px13` + `asus-proart-px13-quirks`

> Superseded 2026-09-24 by the section above. Kept because the px13 kernel is still installed
> and is the fallback entry — and because the quirks package is still what the rollback restores.


## The problem

Internal speakers dead on the stock `linux-cachyos` kernel. The PX13 (HN7306EA) has **dual
TAS2783 SoundWire smart amps** (one per speaker) plus an rt721 headset codec on the
`amdsoundwire` card. The mainline tas2783 driver enumerates the amp and exposes controls
(`Left/Right Spk Switch`, `tas2783-N Amp/Speaker Volume`) but **cannot drive it**.

Symptoms, in the order you'd notice them:
- Raw ALSA playback to `hw:1,2` is silent
- The HiFi UCM profile shows `available: no` — the codec never advertises a `spk:` component
- PipeWire therefore parks the card at profile `off` → Dummy Output

**This is not a mute or routing problem.** `alsamixer` will not save you.

## The fix

```fish
yay -S asus-proart-px13-quirks
```

That pulls in two things:

| Package | What it provides |
|---|---|
| `linux-cachyos-px13` | Kernel carrying ~19 audio patches incl. TAS2783 + jack-detect |
| `asus-proart-px13-quirks` | UCM profiles, firmware symlinks, PipeWire/WirePlumber drop-ins, MT7925 btusb autosuspend |

The px13 kernel installs **alongside** stock `linux-cachyos` / `-lts` — additive and
reversible, pick any entry at boot. Then boot the `cachyos-px13` entry in Limine.

**Confirmed 2026-07-13:** on the px13 kernel, speakers *and* mic work out of the box. No
activation step needed — fresh WirePlumber state on first boot picked the Speaker sink correctly.

### Make it the default kernel

Config-driven so it survives kernel updates. **Do not hand-edit `default_entry` in
`/boot/limine.conf`** — that file gets regenerated.

```fish
# /etc/default/limine
BOOT_ORDER="*px13, *, *lts, *fallback, Snapshots"
```
```fish
sudo limine-update
sudo limine-list        # verify
```

## Current state

```
kernel:  7.0.11-3-cachyos-px13
package: asus-proart-px13-quirks 1:7.0.11-4
```

## Recovery script

`~/px13-audio-activate.sh` — fallback for if audio breaks after an update. Checks you're on
the px13 kernel, resets `~/.local/state/wireplumber`, restarts pipewire, finds the Speaker
sink and sets it default + unmuted + 60%, then plays a test tone.

```fish
~/px13-audio-activate.sh
sudo alsactl store      # persist amp mixer settings once you hear sound
```

## Gotchas

- **`alsa-ucm-conf` is pinned to exactly `=1.2.16.1`** by the quirks package. A `pacman -Syu`
  that bumps it will **conflict — deliberately**. The quirks package's `conf.d` is a verbatim
  copy of that version's `sof-soundwire.conf`.
- **If speakers vanish after an update:** run `pactl list cards`. **No HiFi profile is the
  giveaway.** The card has quietly fallen back to a profile routing to the headphone PCM
  (`hw:1,0`) instead of the amp PCM (`hw:1,2`). Hardware is fine.
- **`paru` is broken** on this box (`libalpm.so.15` missing — pacman soname bumped, paru not
  rebuilt). Use `yay`, which shells out to pacman and is unaffected.

## When can I go back to a stock kernel?

The px13 kernel is a **bridge** until the ASUS PX13 machine-driver quirks land in mainline.
To detect that it has, boot a stock kernel and run:

```fish
amixer -c1 controls | grep -iE "Speaker Switch|Amp Playback Switch"
```

If those controls are present, mainline has caught up and you can drop the fork.

## See also

- [Daily-driving Fedora on a ProArt PX13](https://www.reddit.com/r/ProArt_PX13/comments/1u96idk/dailydriving_fedora_on_a_proart_px13_strix_halo/)
  by u/neuromacmd — someone else's writeup of the same hardware on Fedora; different
  distro, same amps, and it covers the post-silence crackle/pop fix (WirePlumber
  idle-suspend + SoundWire runtime PM).
- Upstream patches: <https://github.com/ftoleedo/px13-audio-fix>
