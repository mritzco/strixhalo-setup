# 8. Resilience — OOM hardening for a shared-memory box

> ## ✅ STATUS: APPLIED 2026-09-24
> Applied by `~/crash-forensics/oom-hardening/install.sh` (idempotent; `install.sh --revert` undoes
> it). Everything below the **Applied** section is the original 2026-08-07 proposal, kept as the
> reasoning that led here.

[← Apps & viewers](07-apps-viewers.md) · [Setup index](README.md) · [Next: DeepSeek-OCR →](09-deepseek-ocr.md)

---

## Applied — 2026-09-24

| # | File | Value |
|---|---|---|
| 1 | `/etc/systemd/coredump.conf.d/10-cap.conf` | `ProcessSizeMax=2G`, `ExternalSizeMax=2G` (systemd's 64-bit default is **32G**) |
| 2 | `/etc/systemd/system/systemd-coredump@.service.d/10-memcap.conf` | `MemoryHigh=1G`, `MemoryMax=2G` |
| 3 | `/etc/systemd/zram-generator.conf.d/10-size.conf` | `zram-size = min(ram / 2, 32 * 1024)` → **32 GiB** (stock: `zram-size = ram` = 124.9 GB). Live device re-created 2026-09-24 18:41: `swapon --show` / `zramctl` → 32G |
| 4 | `pacman -S earlyoom` + `/etc/default/earlyoom` | args below; tuned twice, both times because of a test |

Everything is installed by `~/crash-forensics/oom-hardening/install.sh` (idempotent, also restarts
earlyoom) and reverted by `install.sh --revert`.

### How the earlyoom arguments ended up where they are

**1. The gates.** The proposal said `-m 6 -s 6`. Both conditions must hold, and earlyoom measures them
against the totals it reads at startup:

```
mem total: 127937 MiB, user mem total: 124415 MiB, swap total: 32767 MiB
sending SIGTERM when mem avail <= 10.00% and swap free <= 25.00%,
```

With zram sized to RAM, "swap free <= 6 %" meant **93 GB of zram in use** — unreachable, so earlyoom
would never have fired. The zram cap (item 3) is therefore not cosmetic: it is what makes a swap
threshold mean anything. The RAM gate was raised to 10 % so it acts with ~12 GB still available.

**2. The victim.** `--prefer '^(chrome|chromium|electron|node|code)$'` looked sensible and is wrong on
this box, proven by the first pressure test: earlyoom selects by `oom_score` by default, Chromium and
VS Code set `oom_score_adj=300` on their renderers, and so it killed **six small helpers (57, 40, 14,
11, 103, 87 MiB) plus two VS Code language servers before reaching the actual cause — a 42 GiB
process**. (`node`'s main thread additionally reports `comm=MainThread`, so `|node)` never matched the
hog at all.) Replaced with `--sort-by-rss` and no `--prefer`: pick the biggest process, which *is* the
cause. Final:

```
EARLYOOM_ARGS="-r 3600 -m 10 -s 25 --sort-by-rss --avoid '^(niri|sddm|pipewire|pipewire-pulse|wireplumber|sshd|systemd|dbus-broker)$'"
```

### Pressure-tested 2026-09-24 — both directions

**Must NOT fire** (ordinary heavy load): a model pinning ~70 GB in GTT →

```
Mem:  total 124Gi  used 78Gi  free 0.7Gi  buff/cache 46Gi  available 46Gi
Swap: total 124Gi  used 0
```

RAM went low, swap stayed **untouched**, earlyoom stayed quiet. GTT pages are not swappable, so a
pinned model alone must never trigger a kill — that is exactly what the `-m` **and** `-s` conjunction
is for.

**Must fire, on time, at the right victim.** Deliberate, disposable hog:

```fish
node -e 'const c=[];let n=0;setInterval(()=>{c.push(Buffer.alloc(1024*1024*1024,0x41));console.log(++n+" GiB")},1000)'
```

Final run, in full:

```
mem avail:   319 of 52796 MiB ( 0.61%), swap free: 7748 of 32767 MiB (23.65%)
low memory! at or below SIGTERM limits: mem 10.00%, swap 25.00%
sending SIGTERM to process 278368 uid 1000 "MainThread": VmRSS 50766 MiB,
  cmdline "node -e const c=[];…"
process 278368 exited after 1.804 seconds
```

**One kill, the right one (VmRSS 50.7 GiB), zero collateral** — niri, PipeWire, the audio sink and the
IDEs all survived. Note *which* gate bound: the swap gate (23.65 % free), not the RAM gate — a runaway
allocator gets its pages swapped into zram long before RAM reaches 10 % of the reference total.

**Policy choice:** the victim is "the largest RSS process not in the avoid list". With a model loaded
that is likely the model server — deliberate, since it frees the most memory fastest, and it is cheap
to restart. Put the model's process names in `--avoid` if you would rather lose browsers instead.

One-screen status, any time:

```fish
bash ~/crash-forensics/oom-hardening/check.sh
```

### ⚠️ Gotchas

- **`--prefer` is a trap on this machine.** Browser/IDE renderers carry `oom_score_adj=300`, so
  score-based selection kills a dozen tiny helpers before the real hog. Use `--sort-by-rss`.
- **`node`'s main thread is `comm=MainThread`** — a `--prefer …|node)` regex never matches it.
- **Changing `/etc/default/earlyoom` needs `systemctl restart earlyoom`** (`install.sh` does it).
- **Restart earlyoom only *after* the new swap device is up.** It reads `MemTotal`/`SwapTotal` once at
  startup. Restarting during the zram re-creation window captures `swap total: 0 MiB`, which makes
  "swap free <= 25 %" permanently true and silently degrades it to a RAM-only guard (seen 18:43).
  Order: `swapon --show` → `sudo systemctl restart earlyoom` → `journalctl -u earlyoom -n 6`.
- The zram device only changes size when it is **re-created** (zram-generator runs at boot). Either
  reboot, or `sudo swapoff /dev/zram0 && sudo systemctl restart systemd-zram-setup@zram0.service`.
- `swapoff /dev/zram0` can print `swapoff failed: Invalid argument` **and still have done its job** —
  read `swapon --show`, not the exit code.
- Cores larger than 2G are now **not processed** (no stack trace). That is the trade for not letting a
  core dump eat the machine; `ProcessSizeMax=0` would disable coredumps entirely instead.
- One killer only: `systemd-oomd` stays **disabled** (it already was). Don't enable both.

### Rollback

```fish
sudo bash ~/crash-forensics/oom-hardening/install.sh --revert
sudo pacman -Rns earlyoom        # optional; the revert leaves the package installed
```

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

You run exactly this workload. GLM-4.5-Air is 64 GB, and the engine lessons have you
building and running things alongside it.

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

Adapted from [Daily-driving Fedora on a ProArt PX13](https://www.reddit.com/r/ProArt_PX13/comments/1u96idk/dailydriving_fedora_on_a_proart_px13_strix_halo/)
by **u/neuromacmd** — a third-party writeup of the *same hardware* on **Fedora 44 + GNOME 50**.
The memory analysis transfers; the distro specifics (dnf, DKMS, `gnome-shell`/`gdm` process
names) do not, and have been adapted above.
