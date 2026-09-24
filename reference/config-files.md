# Reference — config file contents

**Single source of truth** for the actual contents of every config file touched during setup.
This page is *overwritten* to stay current — unlike the numbered setup chapters, which are an
append-only log. If you change one of these files, update it here.

[← Setup index](../README.md)

---

### ~/.config/kanata/kanata.kbd

```sh
(defcfg
  ;; Intercept all unmapped keys so the rest of your keyboard typing passes through naturally
  process-unmapped-keys yes
)

;; Map raw evdev keycodes to names kanata can use
(deflocalkeys-linux
  yen 124
  ro  89
)

(defsrc
  ;; Identify the exact physical Japanese hardware layout keys you want to intercept
  kana     ;; Hiragana/Katakana thumb key (右側)
  muhenkan ;; Left thumb Muhenkan key (無変換)
  henkan   ;; Right thumb Henkan key (変換)
  yen      ;; Yen key
  ro       ;; RO key (ろ, near right-shift)
  bksl     ;; Backslash key
)

(deflayer default
  ;; Outputs under your English typing profile — position matches defsrc above
  ralt   ;; Katakana  → Right-Alt
  spc    ;; Muhenkan  → Space
  spc    ;; Henkan    → Space
  bspc   ;; Yen       → Backspace
  bksl   ;; RO        → Backslash (\)
  ret    ;; Backslash → Enter
)
```

### ~/.config/systemd/user/kanata.service
```sh
[Unit]
Description=Kanata keyboard remapper
Documentation=https://github.com/jtroo/kanata

[Service]
Type=simple
ExecStart=/usr/bin/kanata --cfg %h/.config/kanata/kanata.kbd
Restart=on-failure
RestartSec=1

[Install]
WantedBy=default.target
```


### ~/.config/fish/config.fish
```sh
source /usr/share/cachyos-fish-config/cachyos-config.fish

# overwrite greeting
# potentially disabling fastfetch
#function fish_greeting
#    # smth smth
#end
export PATH="$HOME/.local/bin:$PATH"

# Initialize fnm - node
fnm env --use-on-cd | source

# SSH menu with fzf
function gossh
    # Extract host nicknames from your ssh config (skipping wildcard defaults)
    set -l target (grep -E "^Host " ~/.ssh/config | awk '{print $2}' | grep -v "\*" | fzf --height 40% --layout=reverse --border --prompt="⚡ Select SSH Target: ")

    # If a target was selected, connect to it
    if test -n "$target"
        echo "Connecting to $target..."
        ssh $target
    end
end
```


### ~/.config/alacritty/alacritty.toml
Claude destroys the config to accept shift + enter, use this instead:
```sh
# Add this to block 
#[keyboard]
#bindings = [
#  ...
{ key = "Return", mods = "Shift", chars = "\u001B\r" }
#]
```

### ~/.local/share/applications/code.desktop
This is for the keyring to be used.

I think this actually failed, the next change is the good one, ignore this unless it doesn't work...

```sh
[Desktop Entry]
Name=Visual Studio Code
Comment=Code Editing. Redefined.
GenericName=Text Editor
Exec=/usr/bin/code --password-store="gnome-libsecret" %F
Icon=com.visualstudio.code
Type=Application
StartupNotify=true
StartupWMClass=Code
Categories=Utility;TextEditor;Development;IDE;
MimeType=text/plain;inode/directory;application/x-code-workspace;
Actions=new-empty-window;
Keywords=vscode;

[Desktop Action new-empty-window]
Name=New Empty Window
Exec=/usr/bin/code --new-window %F
Icon=com.visualstudio.code

```

### ~/.vscode/argv.json
```json
// This configuration file allows you to pass permanent command line arguments to VS Code.
// Only a subset of arguments is currently supported to reduce the likelihood of breaking
// the installation.
//
// PLEASE DO NOT CHANGE WITHOUT UNDERSTANDING THE IMPACT
//
// NOTE: Changing this file requires a restart of VS Code.
{
	// Use software rendering instead of hardware accelerated rendering.
	// This can help in cases where you see rendering issues in VS Code.
	// "disable-hardware-acceleration": true,

	// Allows to disable crash reporting.
	// Should restart the app if the value is changed.
	"enable-crash-reporter": false,

        "password-store": "gnome-libsecret"
}
```

### ~/.config/fish/functions/dspdf.fish
DeepSeek-OCR front door — `dspdf file.pdf [out.md]`. Usage:
[ocr-usage.md](ocr-usage.md); setup: [09-deepseek-ocr.md](../09-deepseek-ocr.md).

```fish
function dspdf --description 'OCR a PDF/image with DeepSeek-OCR (vLLM container) into one markdown file'
    argparse -n dspdf 'm/mode=' 'd/dpi=' 'f/force' -- $argv; or return 1

    set -l mode free
    set -q _flag_mode; and set mode $_flag_mode
    set -l dpi 200
    set -q _flag_dpi; and set dpi $_flag_dpi

    if not contains -- $mode free markdown grounding both
        echo "dspdf: bad mode '$mode' (free|markdown|grounding|both)" >&2
        return 2
    end
    if not string match -qr '^[0-9]+$' -- $dpi
        echo "dspdf: bad dpi '$dpi'" >&2
        return 2
    end

    if test (count $argv) -lt 1; or test (count $argv) -gt 2
        echo 'usage: dspdf [-m free|markdown|grounding|both] [-d DPI] [-f] INPUT.pdf [OUTPUT.md]' >&2
        return 2
    end

    # Scripts + model live on the host; the container bind-mounts /home/$USER.
    set -l ocr_dir $HOME/Projects/play/ocr-test
    set -l script $ocr_dir/ocr_batch.py

    if not podman container exists vllm
        echo 'dspdf: no podman container named vllm (see handbook/setup/09-deepseek-ocr.md)' >&2
        return 1
    end
    podman start vllm >/dev/null 2>&1
    if test (podman container inspect -f '{{.State.Running}}' vllm) != true
        echo 'dspdf: container vllm exists but is not running' >&2
        return 1
    end

    set -l input (path resolve -- (string replace -r '^~' $HOME -- $argv[1]))
    if not test -f $input
        echo "dspdf: not a file: $input" >&2
        return 1
    end
    switch (string lower -- (path extension -- $input))
        case .pdf .png .jpg .jpeg .webp .bmp .tif .tiff
        case '*'
            echo "dspdf: unsupported input type '$input'" >&2
            return 1
    end

    set -l out
    if test (count $argv) -eq 2
        set out (path resolve -- (string replace -r '^~' $HOME -- $argv[2]))
    else
        set out (path change-extension md -- $input)
    end

    if test -e $out; and not set -q _flag_force
        echo "dspdf: output exists: $out  (use -f to overwrite)" >&2
        return 1
    end

    set -l outdir (path dirname -- $out)
    mkdir -p -- $outdir

    # ocr_batch.py --combine names its output <input stem>.md, so run it against
    # the destination directory and rename afterwards if the names differ.
    set -l produced $outdir/(path change-extension md -- (path basename -- $input))

    set -l cmd (string join ' ' -- \
        (string escape -- $script) \
        (string escape -- $input) \
        --combine -o (string escape -- $outdir) \
        -m (string escape -- $mode) \
        --dpi $dpi)

    podman exec --user 1000:1000 -w $ocr_dir \
        -e HOME=$HOME \
        -e PYTHONPATH=$HOME/.local/ocr-libs \
        vllm bash -lc "python $cmd"
    set -l rc $status

    if test "$produced" != "$out"; and test -f $produced
        mv -f -- $produced $out
    end

    if test -f $out
        echo "[dspdf] $out"
        return 0
    end
    echo "[dspdf] no output produced (rc=$rc)" >&2
    return $rc
end
```

### ~/.config/llama-swap/config.yaml
The model lineup and per-model flags. `preload: []` is deliberate — warming moved into
`llmswap -p` so the default start is empty (see [vision-usage.md](vision-usage.md)).

Editing this file does not need a restart: `pkill -HUP -f 'llama-swap --config'` makes llama-swap
re-read it in place (or run llama-swap with `-watch-config`, which polls every 2 s). ⚠️ A reload
rebuilds the internal server and **stops every running model**, so the next request reloads it.

```yaml
# llama-swap — one OpenAI endpoint, all local models, auto-swap on request.
# Listen port set by the `llmswap` fish function (127.0.0.1:1234).
# 1234 is where omp's OpenAI-dialect "lm-studio" provider auto-discovers this endpoint.
# (omp's native "llama.cpp" provider speaks paths llama-swap does NOT serve — use lm-studio.)
#
# IMPORTANT: `proxy: http://127.0.0.1:${PORT}` forces IPv4. llama-server binds
# 127.0.0.1 (IPv4 only), but llama-swap otherwise dials localhost -> [::1] (IPv6)
# and fails with "connection refused" / "Server unavailable". Keep this on every model.

healthCheckTimeout: 600   # big models (GLM 68G) take a while to load from disk
logLevel: info

# Startup preload is driven by the `llmswap` function, not from here: it passes
# `-p <model>` after llama-swap is up, so plain `llmswap` loads NOTHING (best on
# battery) and `llmswap -p qwen3-coder` warms one model. `preload: []` is also the
# llama-swap default; preloading here would override the function either way.
hooks:
  on_startup:
    preload: []

models:
  "qwen3-coder":
    cmd: /usr/bin/llama-server -hf unsloth/Qwen3-Coder-30B-A3B-Instruct-GGUF:UD-Q4_K_XL -ngl 999 --jinja -c 131072 -fa on --port ${PORT}
    proxy: http://127.0.0.1:${PORT}
    ttl: 1800

  "qwen3-vl":
    cmd: /usr/bin/llama-server -hf unsloth/Qwen3-VL-30B-A3B-Instruct-GGUF:UD-Q4_K_XL -ngl 999 --jinja -c 131072 -fa on --port ${PORT}
    proxy: http://127.0.0.1:${PORT}
    ttl: 1800

  "qwen3.8-flash-next":
    cmd: /usr/bin/llama-server --model /home/itzco/models/Qwen3.8-Flash-Next-GGUF/UD-IQ4_XS/Qwen3.8-Flash-Next-UD-IQ4_XS-00001-of-00003.gguf --alias qwen3.8-flash-next -ngl 999 --jinja -fa on -c 131072 --cache-type-k q8_0 --cache-type-v q8_0 -b 8192 -ub 2048 -np 1 --reasoning-effort medium --reasoning-budget 2048 --port ${PORT}
    proxy: http://127.0.0.1:${PORT}
    ttl: 1800

  "qwen3.8-flash-uncensored":
    cmd: /home/itzco/src/myhacsint-llama.cpp/build-vulkan/bin/llama-server --model /home/itzco/models/Qwen3.8-Flash-Next-Uncensored-GGUF/IQ4_XS/Qwen3.8-Flash-Next-Uncensored.IQ4_XS.gguf --alias qwen3.8-flash-uncensored -md /home/itzco/models/Qwen3.8-Flash-Next-Uncensored-GGUF/MTP-unsloth/mtp-Qwen3.8-Flash-Next-shared-Q8_0.gguf --mmproj /home/itzco/models/Qwen3.8-Flash-Next-Uncensored-GGUF/mmproj/Qwen3.8-Flash-Next-Uncensored.mmproj-f16.gguf --image-min-tokens 1024 -ngl 999 -fa on --jinja -c 131072 --cache-type-k f16 --cache-type-v f16 -b 2048 -ub 1024 --n-cpu-moe 0 --load-mode mmap --tensor-read-lazy auto --no-repack --no-host --fit on --reasoning-effort medium --reasoning-preserve --reasoning-budget 2048 --temp 1.0 --top-k 20 --top-p 0.95 --spec-type draft-mtp --spec-draft-adaptive --spec-draft-n-min 0 --spec-draft-n-max 5 --spec-draft-p-min 0.75 --spec-draft-type-k f16 --spec-draft-type-v f16 --port ${PORT}
    proxy: http://127.0.0.1:${PORT}
    ttl: 1800

  "qwen3.8-27b-vl":
    cmd: /usr/bin/llama-server --model /home/itzco/models/Qwen3.8-27B-GGUF/Qwen3.8-27B-Q8_0.gguf --mmproj /home/itzco/models/Qwen3.8-27B-GGUF/mmproj-F16.gguf -ngl 999 --jinja -c 131072 -fa on --alias qwen3.8-27b-vl --port ${PORT}
    proxy: http://127.0.0.1:${PORT}
    ttl: 1800   
# DeepSeek-R1-70B removed 2026-07-16 — dense/slow locally; use DeepSeek via cloud subscription instead.
```

---

### ~/.config/niri/scripts/lock-before-sleep.sh

Added 2026-09-24 - see [ch. 11](../11-session-lock.md). Run by `swayidle` on `before-sleep`.

```sh
#!/bin/sh
# Lock the Noctalia lock screen before the machine suspends, and don't let the
# system sleep until the session is actually locked.
#
# Why: Noctalia's own `lockOnSuspend` only fires for suspends that Noctalia
# starts (idle timer, session menu, launcher). A lid close is handled by
# systemd-logind, which Noctalia does not listen to -> resumes unlocked.
#
# Used by: ~/.config/niri/cfg/autostart.kdl
#   swayidle -w before-sleep /home/itzco/.config/niri/scripts/lock-before-sleep.sh
# (swayidle listens for logind's PrepareForSleep and holds a delay inhibitor
#  until this script exits.)

qs -c noctalia-shell ipc call lockScreen lock

# Wait for ext-session-lock to be up (logind tracks it as LockedHint), max ~2 s.
i=0
while [ "$i" -lt 20 ]; do
	loginctl show-session "${XDG_SESSION_ID:-auto}" -p LockedHint 2>/dev/null | grep -q 'yes' && exit 0
	i=$((i + 1))
	sleep 0.1
done

# Fallback: give the shell a moment to raise the surface before we freeze.
sleep 0.5
exit 0
```

### ~/.config/niri/cfg/autostart.kdl

The third `spawn-sh-at-startup` line is the 2026-09-24 addition.

```kdl
// ────────────── Startup Applications ──────────────
// https://github.com/YaLTeR/niri/wiki/Configuration:-Miscellaneous#spawn-sh-at-startup

    spawn-sh-at-startup "qs -c noctalia-shell"
    spawn-sh-at-startup "kanata -c ~/.config/kanata/kanata.kbd"

    // Lock before any suspend (incl. lid close, which logind handles and Noctalia
    // does not see), so a resume lands on the lock screen instead of the desktop.
    // Added 2026-09-24; requires `swayidle`.
    spawn-sh-at-startup "swayidle -w before-sleep /home/itzco/.config/niri/scripts/lock-before-sleep.sh"
```

---

## OOM hardening (ch. 8, applied 2026-09-24)

### /etc/systemd/coredump.conf.d/10-cap.conf

Cap the 32G systemd default.
```ini
[Coredump]
# systemd's 64-bit default is 32G (coredump.conf(5)): a crashing app under memory
# pressure therefore tries to write a 32 GB core -- that is what turned an app crash
# into a session-killing OOM (handbook ch. 8). Cap it.
ProcessSizeMax=2G
ExternalSizeMax=2G
```

### /etc/systemd/system/systemd-coredump@.service.d/10-memcap.conf

Contain the core-writing worker.
```ini
[Service]
# Contain the process that writes the core. If the core is huge, *this* unit gets
# killed instead of the memory being taken from your session.
MemoryHigh=1G
MemoryMax=2G
```

### /etc/systemd/zram-generator.conf.d/10-size.conf

Overrides the stock `zram-size = ram` (124.9 GB). Drop-ins beat the single config file, per zram-generator.conf(5).
```ini
[zram0]
# Stock /usr/lib/systemd/zram-generator.conf says `zram-size = ram` -> 124.9 GB here.
# zram is *compressed RAM*, not capacity: at that size the kernel happily pushes
# ~30 GB of data into RAM and thrashes for minutes before anything fires (observed
# 2026-09-24: 27 page-allocation failures, 36 GB of RAM held by zram, 18 MB free in
# the normal zone). Upstream's own recommended ceiling for a big-RAM box is 32 GiB
# (zram-generator.conf(5)); this expression evaluates to exactly that here.
zram-size = min(ram / 2, 32 * 1024)
```

### /etc/default/earlyoom

Read by earlyoom.service. Tuned from two pressure tests — see ch. 8.
```ini
# /etc/default/earlyoom -- read by earlyoom.service (EnvironmentFile)
#
# Tuned 2026-09-24 from a deliberate pressure test (handbook ch. 8). Original config was
#   -m 5 -s 25 --prefer '^(chrome|chromium|electron|node|code)$'
# and the test showed exactly why that is wrong on this box:
#
#   * selection defaults to oom_score, and Chromium/VS Code set oom_score_adj=300 on their
#     renderers -> earlyoom killed SIX small helpers (57, 40, 14, 11, 103, 87 MiB) plus two
#     VS Code language servers before it got to the actual cause, a 42 GiB process
#   * `--prefer` made that worse: it biases the choice towards those very helpers
#   * node's main thread reports comm "MainThread", so `|node)` never matched the hog anyway
#
# Now: --sort-by-rss picks the *biggest* process (the thing that actually caused the pressure),
# no --prefer so nothing overrides that, and thresholds raised so it fires while there is still
# headroom to act. `both conditions must hold` keeps a pinned model (GTT is not swappable, swap
# stays untouched) from ever tripping it -- that half is verified.
#
# -m 10 : fire when free RAM < 10%  (~12 GB of 124 GB)
# -s 25 : ... AND free swap < 25%. Swap is zram here; 25% of 32 GiB ~= 8 GiB resident
#         compressed pages = "deep into swapping".
# -r 3600: hourly memory report (leave it, it is your only breadcrumb trail)
# --avoid: never kill these (niri/sddm/pipewire/ssh -- never the session, never the audio)
#
# Optional knobs, not enabled:
#   -n   desktop notification on each kill (needs `systembus-notify`)
#   -g   kill the whole process group of the victim (e.g. the entire browser instead of one tab)
EARLYOOM_ARGS="-r 3600 -m 10 -s 25 --sort-by-rss --avoid '^(niri|sddm|pipewire|pipewire-pulse|wireplumber|sshd|systemd|dbus-broker)$'"
```

