# 2. Audio — internal speakers (TAS2783 SoundWire amps)

**Fixed:** 2026-07-12 · **Confirmed working:** 2026-07-13

[← Base system](01-base-system.md) · [Setup index](README.md) · [Next: Keyboard →](03-keyboard.md)

---

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
