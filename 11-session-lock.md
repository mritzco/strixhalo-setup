# 11. Session lock — lock on suspend

**Date:** 2026-09-24 · **Status:** ✅ working (verified over a real suspend/resume)

[← Crash forensics](10-crash-forensics.md) · [Setup index](README.md)

---

## The problem

Closing the lid suspended the machine (systemd-logind `HandleLidSwitch=suspend` — `/etc/systemd/logind.conf`
has no overrides) but waking it up landed straight on the **unlocked desktop**.

Noctalia *has* a lock screen and `lockOnSuspend` **is** enabled
(`~/.config/noctalia/settings.json` → `"lockOnSuspend": true`). The catch is where that setting is
read — `Modules/Power/IdleService.qml`:

```
stage === "suspend"  ->  CompositorService.lockAndSuspend()   // only for suspends Noctalia starts
```

The same call backs Noctalia's session menu and launcher. There is **no logind listener**
(`grep -r 'login1\|PrepareForSleep\|LockedHint' /etc/xdg/quickshell/noctalia-shell/` → nothing), so a
lid-close suspend — which logind performs, not Noctalia — never triggers the lock.

Two other things that look like lockers but aren't: the installed `niri-screensaver` package is a
TerminalTextEffects screensaver, and niri itself has no built-in lock. Something external has to call
the lock.

## The fix

`swayidle` subscribes to logind's `PrepareForSleep` signal and holds a delay inhibitor while its
`before-sleep` command runs — so the machine cannot finish suspending until the lock is up.

```fish
sudo pacman -S --needed swayidle
```

One line in `~/.config/niri/cfg/autostart.kdl`:

```kdl
spawn-sh-at-startup "swayidle -w before-sleep /home/itzco/.config/niri/scripts/lock-before-sleep.sh"
```

That script (full contents in [reference/config-files.md](reference/config-files.md)) locks and then
waits until logind agrees the session is locked:

```sh
qs -c noctalia-shell ipc call lockScreen lock
# then poll up to 2 s:  loginctl show-session "$XDG_SESSION_ID" -p LockedHint   ->  "yes"
```

Start it in the **current** session as well — `spawn-at-startup` only applies at the next niri start:

```fish
niri msg action spawn-sh -- "swayidle -w before-sleep /home/itzco/.config/niri/scripts/lock-before-sleep.sh"
pgrep -a swayidle
```

## Verify

```fish
qs -c noctalia-shell ipc call lockScreen lock   # locks immediately; password gets you back in
systemctl suspend                               # wake -> you should land on the lock screen
```

## ⚠️ Gotchas

- **Wakes unlocked** → the hook isn't running. Check `pgrep -a swayidle`; the autostart line takes
  effect only from the next niri start (or use the manual `spawn-sh` above).
- Manual locking already existed and is independent of all this: `Mod+ALT+L` →
  `qs -c noctalia-shell ipc call lockScreen lock` (`~/.config/niri/cfg/keybinds.kdl`). Volume,
  brightness and media keys carry `allow-when-locked=true`, so they still work on the lock screen.
- **Notifications appear over the lock screen** — anything your apps send shows up there (see below).
- The lock screen background is the wallpaper; with a missing `wallpaper.directory` and
  `noctaliaPerformance.disableWallpaper = true` you get the solid colour instead.

## The "ads" on the lock screen (not Linux, not Noctalia)

Promotional notifications on the lock screen turned out to be **browser-extension push
notifications**. Audit from 2026-09-24:

```
~/.config/chromium/Default/Extensions/ophjlpahpchlmihnnnihgmmeilfjmjjc/3.7.2_0
  name: LINE 3.7.2      permissions: [… 'notifications' …]      host_permissions: ['*://*/*']
```

and Noctalia's notification cache showed the actual payloads:

```
[Chromium] LINE SHOPPING | รับส่วนลดพิเศษจาก LINE Premium แล้วไปช้อปกัน ⭐️ สมัครสมาชิก
```

The tell: Chromium's **site** notification permission list was *empty* (0 entries in
`Preferences`/`Secure Preferences`), so the pushes came from the extension, not from a site you
allowed. Extensions can't have their `notifications` permission toggled per-extension in modern
Chrome, so the fix is to remove/disable the extension:

```fish
# what has been notifying you, and from which app — your notification audit log
python3 -c "import json;d=json.load(open('/home/itzco/.cache/noctalia/notifications.json'));i=d if isinstance(d,list) else d.get('notifications',[]);[print(n.get('appName'),'|',n.get('summary')) for n in i[-20:]]"
```

- `chrome://extensions` → **LINE** → Remove (or disable). The extension is legacy; `chat.line.me`
  in a normal tab works without it.
- Or keep it and turn off promo notifications inside LINE (Settings → Notifications).
- Nothing in CachyOS or Noctalia serves ads: Noctalia's only endpoints are `api.open-meteo.com`
  (weather), `api.noctalia.dev/{geolocate,geocode,supporters,stars,upgradelog}` and `api.github.com`;
  its supporters/contributors lists render only inside Settings → About.

## Rollback

```fish
# delete the swayidle line from ~/.config/niri/cfg/autostart.kdl, then:
sudo pacman -Rns swayidle
rm ~/.config/niri/scripts/lock-before-sleep.sh
```
Manual locking (`Mod+ALT+L`) keeps working — it needs neither swayidle nor the script.

## See also

- [reference/niri.md](reference/niri.md) — niri config how-to (incl. the lock wiring)
- [reference/config-files.md](reference/config-files.md) — lock script + autostart contents
- [ch. 2](02-audio.md) — audio: its resume hook restarts PipeWire, which costs browsers their mic
  until they are restarted
