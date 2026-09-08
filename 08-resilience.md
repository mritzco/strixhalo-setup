# 8. Resilience — OOM hardening for a shared-memory box

> ## ⚠️ STATUS: NOT APPLIED
> Verified on **2026-08-07**: no `/etc/systemd/coredump.conf.d/`, no coredump service
> drop-in, no `/etc/default/earlyoom`, `earlyoom` not installed or enabled.
> This chapter is a **proposal**, not a record. Everything below is untested on this machine.

[← Apps & viewers](07-apps-viewers.md) · [Setup index](README.md) · [Next: DeepSeek-OCR →](09-deepseek-ocr.md)

---

## Why this matters here specifically

On a normal laptop 128 GB is enormous headroom. On **this** machine the iGPU shares that same
pool, and a large local model can pin 90–120 GB through GTT. What's left for everything else
is small — and the failure mode is not a friendly "out of memory" dialog.

The reported chain (see [source](#source) below) is:

```
big model pins ~110 GB via GTT
        │
        ▼
some app crashes  ──►  systemd-coredump starts writing a core
        │                       │
        │              default ProcessSizeMax / ExternalSizeMax = 32G
        │                       │
        ▼                       ▼
             memory exhausted → OOM killer fires
                        │
                        ▼
            session process killed  ("I got logged out")
                        │
                        ▼
        log back in → fresh compositor under pressure → thrash → hard lock
```

The tell in the journal is a cluster of **SIGBUS** crashes — mmap-backed pages that couldn't
be faulted in once memory was gone.

You run exactly this workload. GLM-4.5-Air is 64 GB, and lesson 02 in [`../ai/`](../ai/README.md)
has you building and running things alongside it.

## The zram wrinkle — read before copying the source config

This machine's swap is **zram**, not a disk partition:

```
NAME       TYPE        SIZE
/dev/zram0 partition 124,9G
```

zram is *compressed RAM*, not extra memory. Two consequences the source article (which
assumed conventional swap) doesn't cover:

1. **"Free swap" is not spare capacity.** Filling zram consumes the very RAM you're short of.
2. **GTT pages aren't swappable anyway**, so zram cannot relieve pressure from a loaded model —
   it only absorbs pressure from ordinary processes, while burning CPU on compression.

So an earlyoom rule that waits for *both* RAM and swap to be low will fire **later** here than
on a disk-swap box, not earlier. Consider tuning the swap threshold up, or relying primarily
on the RAM threshold.

## Proposed changes

### 1. Cap coredump size

```ini
# /etc/systemd/coredump.conf.d/10-cap.conf
[Coredump]
ProcessSizeMax=2G
ExternalSizeMax=2G
```

### 2. Confine the coredump worker to its own cgroup

So a huge core gets the worker killed, not your session.

```ini
# /etc/systemd/system/systemd-coredump@.service.d/10-memcap.conf
[Service]
MemoryHigh=1G
MemoryMax=2G
```

### 3. earlyoom — kill a runaway app instead of the session

```fish
sudo pacman -S earlyoom
```

```sh
# /etc/default/earlyoom
# NOTE: --avoid list ADAPTED for this box — niri + sddm, NOT gnome-shell/gdm as in the source.
EARLYOOM_ARGS="-r 3600 -m 6 -s 6 \
  --avoid '^(niri|sddm|pipewire|pipewire-pulse|wireplumber|sshd|systemd|dbus-broker)$' \
  --prefer '^(chrome|chromium|electron|node|code)$'"
```

```fish
sudo systemctl enable --now earlyoom
```

`-m 6 -s 6` = fire when **both** free RAM and free swap drop below 6%. The intent is that a
loaded model alone never trips it. **Given the zram note above, validate this threshold rather
than trusting it** — see verification.

## Verification plan

Do this deliberately, not by waiting for a crash:

```fish
# 1. baseline
free -g; swapon --show; systemd-analyze cat-config systemd/coredump.conf

# 2. confirm the coredump cap took
systemctl show systemd-coredump@0 -p MemoryMax 2>/dev/null
coredumpctl list | tail -5

# 3. watch earlyoom's own view of memory
sudo journalctl -u earlyoom -f

# 4. load GLM-4.5-Air (64 GB) and observe headroom under real load
llmswap    # then request glm-4.5-air
watch -n2 'free -g; echo; cat /sys/class/drm/card*/device/mem_info_gtt_used'
```

If earlyoom reports very little margin while just a model is loaded, raise the thresholds
before you're in a real squeeze.

## Rollback

```fish
sudo rm /etc/systemd/coredump.conf.d/10-cap.conf
sudo rm -r /etc/systemd/system/systemd-coredump@.service.d/
sudo systemctl disable --now earlyoom
sudo systemctl daemon-reload
```

## Source

Adapted from [external/px13-fedora-strix-halo.md](../external/px13-fedora-strix-halo.md) —
a third-party writeup of the *same hardware* on **Fedora 44 + GNOME 50**. The memory analysis
transfers; the distro specifics (dnf, DKMS, `gnome-shell`/`gdm` process names) do not, and have
been adapted above.
