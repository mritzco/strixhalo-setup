# Cheat sheet — every command worth remembering

Copy-paste only. The chapters explain *why*; this page is the *what*. Overwritten freely, so it
stays current — if a command lives in a chapter, that chapter is authoritative and this page links
to it.

[← Setup index](../README.md)

---

## Status in one screen

```fish
uname -r                                                          # 7.2.3-1-cachyos expected
bash ~/px13-audio-fix/check-audio.sh                              # audio: must be all PASS
systemctl is-active crashwatch                                    # crash capture alive
sysctl kernel.hardlockup_panic kernel.nmi_watchdog kernel.panic    # 1 1 10
pgrep -a swayidle                                                 # lock-on-suspend armed
pactl list sinks short                                            # ...HiFi__Speaker__sink present
journalctl -t crashwatch -b 0 -o cat | grep -i bootcheck          # previous boot ended cleanly?
bash ~/crash-forensics/oom-hardening/check.sh                        # coredump caps / zram size / earlyoom
```

## Audio — [ch. 2](02-audio.md)

```fish
bash ~/px13-audio-fix/check-audio.sh                   # after EVERY kernel update
sudo /usr/local/lib/px13-soundwire-recover.sh          # no speakers after resume: full stack reload
amixer -D hw:1 cget name='tas2783-1 Channel Playback'  # values=1 Left  (amp 2 -> values=2 Right)
alsaucm -c 1 list _devices/HiFi | grep Speaker         # "Dummy Output"? if this prints nothing, UCM is broken
pactl get-sink-volume @DEFAULT_SINK@                   # silent though everything looks right -> 0%?
speaker-test -D pulse -c2 -l1 -t wav                   # left/right check
sudo tail -20 /var/log/px13-soundwire-resume.log       # what the resume hook did last time
```

```fish
# rollback to the old px13-kernel audio stack
cd ~/px13-audio-fix && bash uninstall-durable.sh
sudo bash ~/crash-forensics/restore-quirks-ucm.sh      # then reboot, pick the px13 entry
```

## Kernel & boot — [ch. 2](02-audio.md)

```fish
uname -r                                              # which kernel am I running
sudo limine-list                                      # boot entries in order (first = default)
grep -n BOOT_ORDER /etc/default/limine                # the order config
sudo limine-update                                    # apply it (never hand-edit /boot/limine.conf)

# revert the default-kernel flip
sudo cp /etc/default/limine.bak-pre-default-flip /etc/default/limine && sudo limine-update

# is IgnorePkg holding px13 + quirks back? (dry run, installs nothing)
sudo pacman -Sy && sudo pacman -Spu                   # those two must NOT appear in the plan

# kernel-update ritual
sudo pacman -Syu && bash ~/px13-audio-fix/check-audio.sh    # DKMS rebuilds; verify audio after
```

## Memory & OOM — [ch. 8](08-resilience.md)

```fish
bash ~/crash-forensics/oom-hardening/check.sh                    # all four items, one screen
sudo journalctl -u earlyoom -f                                   # watch it while loading a big model
free -g; swapon --show; zramctl                                  # raw view
sudo swapoff /dev/zram0 && sudo systemctl restart systemd-zram-setup@zram0.service   # apply a new zram size now
sudo systemctl restart earlyoom                                  # after a swap-total change
sudo bash ~/crash-forensics/oom-hardening/install.sh --revert    # undo the hardening
```

## Session lock — [ch. 11](11-session-lock.md)

```fish
qs -c noctalia-shell ipc call lockScreen lock          # lock now (also Mod+ALT+L)
pgrep -a swayidle                                      # is the suspend hook running?
niri msg action spawn-sh -- "swayidle -w before-sleep /home/itzco/.config/niri/scripts/lock-before-sleep.sh"
systemctl suspend                                      # test: waking must land on the lock screen
loginctl show-session $XDG_SESSION_ID -p LockedHint     # "yes" while locked

# notification audit: what talks to your desktop, and from which app
python3 -c "import json;d=json.load(open('/home/itzco/.cache/noctalia/notifications.json'));i=d if isinstance(d,list) else d.get('notifications',[]);[print(n.get('appName'),'|',n.get('summary')) for n in i[-20:]]"
```

## Crash forensics — [ch. 10](10-crash-forensics.md)

```fish
sudo bash ~/crash-forensics/crash-forensics.sh | tee ~/crash-forensics/forensics-(date +%F-%H%M).txt
journalctl --list-boots                                 # find the dead boot's index
journalctl -t crashwatch -b 0 -o cat | grep -i bootcheck # UNCLEAN = that stop logged no shutdown
journalctl -t crashwatch -n 5 -o cat                    # last known state before a reset
cat /proc/cmdline                                       # expect: panic=10 oops=panic
```

## Desktop / niri — [reference/niri.md](niri.md)

```fish
niri validate -c ~/.config/niri/config.kdl      # check the config after editing it
niri msg action spawn-sh -- "<command>"         # run something inside the session (survives the shell)
wpctl status                                    # PipeWire graph: devices, sinks, sources, streams
wpctl set-default <id>                          # pin the default sink (speaker id from the list above)
```

## Reinstall / repair (all idempotent)

```fish
cd ~/px13-audio-fix && bash install-durable.sh    # audio module + UCM (run as your user; local patch required)
sudo bash ~/crash-forensics/install.sh --apply-cmdline   # crashwatch + journald + sysctl + cmdline
sudo pacman -S --needed swayidle                          # lock-on-suspend dependency
```
