# 10. Crash forensics — silent resets, and what to do about them

**Date:** 2026-09-24 · **Status:** ✅ capture stack installed and verified (no recurrence yet)

[← DeepSeek-OCR](09-deepseek-ocr.md) · [Setup index](README.md)

---

## The event this comes from — 2026-09-24 13:49:29

The machine went black and rebooted itself mid-work (ComfyUI generating + an OMP session). Nothing
in the journal explained it:

| Looked for | Found |
|---|---|
| shutdown sequence in the dead boot | **none** — that boot's journal ends mid-stream at `13:49:29.515`; every other boot ends with `systemd-shutdown` / `Unmounted` markers |
| panic / oops / BUG / hung task | none |
| OOM kill | none (`systemd-oomd` disabled, `earlyoom` not installed) |
| thermal trip, amdgpu reset, NVMe/AER errors | none |
| MCE / EDAC hardware error | none |
| `/sys/fs/pstore`, ACPI `BERT/ERST/HEST/GHES`, DMI type 15 (BIOS event log) | all **empty or not exposed** — the firmware recorded nothing |

Why it was invisible: `nowatchdog` was on the kernel command line (no NMI/hard-lockup detector),
`kernel.panic=0` + `panic_on_oops=0` (a panic hangs instead of rebooting), `RuntimeWatchdogUSec=0`
and no `/dev/watchdog` device. **Nothing in the OS could have rebooted this box**, so the reset
came from below the OS — EC/firmware power event, a platform-fatal machine check, or a triple
fault. The only pre-crash anomaly was severe memory pressure (below).

## What is installed now

| Piece | Path | Purpose |
|---|---|---|
| `crashwatch.service` | `/usr/local/bin/crashwatch.sh` | 5 s heartbeat into the journal: `mem_avail`, PSI, zram, `gpu_busy`/GTT, temps, fans, top-RSS process. The **last line before a reset is the state at death** |
| `crashwatch-bootcheck.service` | same script, `--bootcheck` | logs `UNCLEAN previous boot stop …` after any boot whose predecessor has no shutdown sequence |
| journald drop-in | `/etc/systemd/journald.conf.d/10-journal-durability.conf` | `SyncIntervalSec=5s` (the 5 min default is a five-minute blind tail after a hard reset) |
| lockup → panic | `/etc/sysctl.d/90-lockup-panic.conf` | `kernel.nmi_watchdog=1`, `kernel.hardlockup_panic=1` — **overrides `/usr/lib/sysctl.d/70-cachyos-settings.conf`, which disables the NMI watchdog for power reasons** |
| kernel cmdline | `/etc/default/limine` | `panic=10 oops=panic`, `nowatchdog` removed |
| forensics script | `~/crash-forensics/crash-forensics.sh` | root-only dump: clean/unclean verdict per boot, pstore, BERT/ERST, `dmidecode -t 15`, MCE, SMART, watchdog state |
| runbook | `~/crash-forensics/RESTART-CHECKLIST.md` | operations + rollback (incl. the audio ones) |

⚠️ **`hardlockup_panic=1` is not a kernel boot parameter** — the kernel reports it under
`Unknown kernel command line parameters … will be passed to user space`. That is why it lives in
`/etc/sysctl.d/` now. `panic=10` and `oops=panic` *are* valid boot parameters.

## Using it

After any unexplained freeze or reset:

```fish
sudo bash ~/crash-forensics/crash-forensics.sh | tee ~/crash-forensics/forensics-(date +%F-%H%M).txt
```

It picks the most recent boot without a shutdown sequence and dumps the firmware-level evidence.
Sanity-check the stack is alive:

```fish
systemctl is-active crashwatch
journalctl -t crashwatch -n 3 -o cat
sysctl kernel.hardlockup_panic kernel.nmi_watchdog kernel.panic    # 1 1 10
```

### Reading the next event

| Evidence | Conclusion |
|---|---|
| last `crashwatch` line, `UNCLEAN`, empty pstore/BERT, no MCE | firmware/EC power reset → hardware; ASUS service case |
| `Kernel panic` in the previous boot | kernel bug — `panic=10` reboots and the trace is now on disk |
| `Hard lockup` / hung-task trace | hang, captured instead of silent |
| `amdgpu` ring timeout / GPU reset | GPU hang; the driver recovered, no reset |

### Rollback

```fish
sudo rm -f /usr/lib/systemd/system-sleep/…             # n/a here
sudo systemctl disable --now crashwatch crashwatch-bootcheck
sudo rm -f /usr/local/bin/crashwatch.sh /etc/systemd/system/crashwatch*.service \
           /etc/sysctl.d/90-lockup-panic.conf /etc/systemd/journald.conf.d/10-journal-durability.conf
sudo rm -rf /home/itzco/crash-forensics
# and drop panic=10 oops=panic from /etc/default/limine, then: sudo limine-update
```

## ⚠️ Gotchas learned the hard way

- `nowatchdog` (a CachyOS default) with `panic=0` turns any panic or hard lockup into a **silent
  permanent freeze**: no log, no reboot, no clue. Fixing it needs both the cmdline change *and*
  the sysctl, because CachyOS re-disables the NMI watchdog at boot.
- A hard reset loses everything not yet fsynced — the journal's `SyncIntervalSec=5min` default is
  exactly why the last minutes of the dead boot were missing.
- `panic=10` is what makes a panic diagnosable: without it, the trace sits on a screen nobody sees.
- **`/tmp` is tmpfs.** Tooling staged there vanishes on reboot (cost one debugging round).
- The firmware on this laptop exposes **no** error channels (no ERST/BERT, empty pstore, no BIOS
  event log), so a power/EC-class reset will never be provable from software — only classified.

## Memory pressure — the one pre-crash anomaly

The dead boot had **27 `page allocation failure` events**, zram sized to `ram` (124.9 GB) holding
36 GB of RAM, ~18 MB free in the normal zone, plus `ttm.pages_limit=28835840` and
`amdgpu.gttsize=112640` (~110 GB each) on a 124 GB machine. Not proven to be the cause — but
[ch. 8](08-resilience.md)'s OOM hardening is still *proposed, not applied*, and capping
`zram-size` is the cheapest mitigation.

## See also

- `~/crash-forensics/RESTART-CHECKLIST.md` — the operational runbook
- [ch. 2](02-audio.md) — the audio migration done in the same session
- [ch. 8](08-resilience.md) — OOM hardening (**still not applied**)
