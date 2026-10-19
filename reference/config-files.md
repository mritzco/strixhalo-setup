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

  ;; Only grab the laptop's built-in keyboard, so the JP remaps apply there only.
  ;; External keyboards (e.g. the US Bluetooth one) are left untouched by kanata
  ;; and use their own native layout.
  linux-dev-names-include (
    "AT Translated Set 2 keyboard"
  )
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

# Auto-manage Python virtual environments on cd
__auto_python_venv

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
[general]
working_directory = "None"
live_config_reload = true

[env]
TERM = "xterm-256color"
WINIT_X11_SCALE_FACTOR = "1.0"

[window]
dimensions = { columns = 100, lines = 30 }
dynamic_padding = true
decorations = "Full"
opacity = 0.95
title = "Alacritty@CachyOS"
class = { instance = "Alacritty", general = "Alacritty" }
decorations_theme_variant = "Dark"

[scrolling]
history = 10000
multiplier = 3

[font]
normal = { family = "MesloLGS Nerd Font Mono", style = "Regular" }
bold = { family = "MesloLGS Nerd Font Mono", style = "Bold" }
italic = { family = "MesloLGS Nerd Font Mono", style = "Italic" }
bold_italic = { family = "MesloLGS Nerd Font Mono", style = "Bold Italic" }
size = 12.0

[colors]
draw_bold_text_with_bright_colors = true

[colors.primary]
background = "0x2E3440"
foreground = "0xD8DEE9"

[colors.normal]
black = "0x3B4252"
red = "0xBF616A"
green = "0xA3BE8C"
yellow = "0xEBCB8B"
blue = "0x81A1C1"
magenta = "0xB48EAD"
cyan = "0x88C0D0"
white = "0xE5E9F0"

[colors.bright]
black = "0x4C566A"
red = "0xBF616A"
green = "0xA3BE8C"
yellow = "0xEBCB8B"
blue = "0x81A1C1"
magenta = "0xB48EAD"
cyan = "0x8FBCBB"
white = "0xECEFF4"

[selection]
semantic_escape_chars = ",│`|:\"' ()[]{}<>\t"
save_to_clipboard = true

[cursor]
style = { shape = "Underline", blinking = "Off" }
unfocused_hollow = true
thickness = 0.15

[mouse]
hide_when_typing = true
bindings = [
{ mouse = "Middle", mods = "None", action = "PasteSelection" },
]

[keyboard]
bindings = [
{ key = "Paste", mods = "None", action = "Paste" },
{ key = "Copy", mods = "None", action = "Copy" },
{ key = "L", mods = "Control", action = "ClearLogNotice" },
{ key = "L", mods = "Control", mode = "~Vi", chars = "\f" },
{ key = "PageUp", mods = "Shift", mode = "~Alt", action = "ScrollPageUp" },
{ key = "PageDown", mods = "Shift", mode = "~Alt", action = "ScrollPageDown" },
{ key = "Home", mods = "Shift", mode = "~Alt", action = "ScrollToTop" },
{ key = "End", mods = "Shift", mode = "~Alt", action = "ScrollToBottom" },
{ key = "V", mods = "Control|Shift", action = "Paste" },
{ key = "C", mods = "Control|Shift", action = "Copy" },
{ key = "F", mods = "Control|Shift", action = "SearchForward" },
{ key = "B", mods = "Control|Shift", action = "SearchBackward" },
{ key = "C", mods = "Control|Shift", mode = "Vi", action = "ClearSelection" },
{ key = "Key0", mods = "Control", action = "ResetFontSize" },
{ key = "Return", mods = "Shift", chars = "\u001B\r" }
]

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

	// Unique id used for correlating crash reports sent from this instance.
	// Do not edit this value.
	"crash-reporter-id": "2c3be373-ef90-4f3a-8d6b-64d0d8d4d63b",
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
The on-demand llama-swap lineup and per-model flags. Pruned 2026-10-03 to two models —
`qwen3-coder` (77 t/s coding) and `qwen3-vl`. The Flash stack (`qwen3.8-flash-next`,
`qwen3.8-flash-uncensored`, `qwen3.8-27b-vl`) moved to the resident halogen container — see the
`halogen.fish` section below; their recipes stay in `~/Projects/strixhalo`. What remains here is
what halogen cannot do: load on demand, idle at zero cost. `preload: []` is deliberate — warming
moved into `llmswap -p` so the default start is empty (see [vision-usage.md](vision-usage.md)).

Editing this file does not need a restart: `pkill -HUP -f 'llama-swap --config'` makes llama-swap
re-read it in place (or run llama-swap with `-watch-config`, which polls every 2 s). ⚠️ A reload
rebuilds the internal server and **stops every running model**, so the next request reloads it.

```yaml
# llama-swap — the light, on-demand pair (coding churn + quick image description) at :1234.
# Listen port set by the `llmswap` fish function (127.0.0.1:1234).
# 1234 is where omp's OpenAI-dialect "lm-studio" provider auto-discovers this endpoint.
# (omp's native "llama.cpp" provider speaks paths llama-swap does NOT serve — use lm-studio.)
#
# PRUNED 2026-10-03. Removed: qwen3.8-flash-next, qwen3.8-flash-uncensored, qwen3.8-27b-vl.
# Those three moved to halogen-flash-server (`halogen`, 127.0.0.1:8731): one resident stack that
# serves the Flash model ~3x faster on prefill, plus vision and the NPU models. Dropping the two
# flash rows also retired both source forks — only qwen3.8-flash-uncensored used the myhacsint
# build, and the Nathan fork was already superseded. Recipes for all of them stay in
# ~/Projects/strixhalo. What remains here is what halogen cannot do: load on demand, and a
# 77 t/s coding model.
#
# IMPORTANT: `proxy: http://127.0.0.1:${PORT}` forces IPv4. llama-server binds
# 127.0.0.1 (IPv4 only), but llama-swap otherwise dials localhost -> [::1] (IPv6)
# and fails with "connection refused" / "Server unavailable". Keep this on every model.

healthCheckTimeout: 600   # a model loading from disk can take a while
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
# DeepSeek-R1-70B removed 2026-07-16 — dense/slow locally; use DeepSeek via cloud subscription instead.
```

### ~/.config/fish/functions/halogen.fish

Added 2026-10-03. Launcher for the `halogen-flash-server` container (Flash-Next uncensored + vision + 5 NPU models) on `http://127.0.0.1:8731/v1`. `halogen` starts it detached and waits for `/health`; `-f` runs it in the foreground, `-n` skips the NPU models, `--fetch` pulls the image first; `stop`, `status`, `logs` and `restart` manage a running instance.

```fish
function halogen --description 'halogen-flash-server: Flash-Next + vision + NPU models on one endpoint at :8731'
    # The container holds ~90 GiB (Flash weights + KV pool) and, with the NPU on, locks another
    # ~8.6 GiB. It is a "give it the machine" server: `halogen stop` before heavy ComfyUI/video work.
    # A bare `halogen` starts it if needed and always prints the crib sheet below.

    set -l models   $HOME/halogen-models
    set -l ckpt     /models/Qwen3.8-Flash-Next-Uncensored.IQ4_XS.gguf
    set -l npu      decider-0.8b,qwen3-embedding-0.6b,qwen3-reranker-0.6b,qwen3guard-gen-0.6b,qwen3.5-2b
    set -l image    ghcr.io/peonist-ai/halogen-flash-server:0.16.2
    set -l log      $HOME/.cache/halogen.log
    set -l url      http://127.0.0.1:8731

    set -l help '
halogen — the resident Flash stack (container, API only, port 8731)
  model     qwen3.8-flash-next-uncensored · 131k ctx · prompt cache · OpenAI /v1
            also /v1/responses (Codex) and /v1/messages (Claude Code)
  vision    ON — send an image content part (data: URL or bare base64); ~2k tokens per 1440p frame
  NPU       decider-0.8b (classify/router)   qwen3-embedding-0.6b  -> /v1/embeddings
            qwen3-reranker-0.6b -> /v1/rerank   qwen3guard-gen-0.6b -> /v1/moderations
            qwen3.5-2b (short generation)
  clients   omp -> halogen/qwen3.8-flash-next-uncensored   pi -> halogen
            any OpenAI client -> base http://127.0.0.1:8731/v1 with any api key
  no UI     API only, there is no chat page here. For a UI point Open WebUI at
            http://127.0.0.1:8731/v1 (llama-swap :1234 is the separate on-demand pair)
  verbs     halogen          start detached if needed (prints this)
            halogen -f       foreground, log in this terminal, Ctrl-C stops
            halogen -n       no NPU models: Flash + vision only, ~90 GiB, no mlock
            halogen --fetch  also set HALOGEN_DOWNLOAD to fetch/verify weights
            halogen status   one screen: container, endpoint, NPU list, clock, memory
            halogen logs -f  follow the container log
            halogen restart  bounce it
            halogen stop     stop it and give the ~90 GiB back
            halogen help     this text
  host      needs the GPU fabric clock held + memlock unlimited; both checked on start
'

    argparse -n halogen 'f/foreground' 'n/no-npu' 'fetch' -- $argv; or return 1
    set -l verb $argv[1]

    switch $verb
        case help
            echo $help
            return 0

        case stop
            podman stop -t 30 halogen >/dev/null 2>&1
            podman rm -f halogen >/dev/null 2>&1   # podman stop returns 1 on SIGTERM exit; ignore
            echo "[halogen] stopped — memory returned (check: free -g)"
            return 0

        case logs
            podman logs $argv[2..-1] halogen
            return $status

        case status
            set -l st (podman inspect -f '{{.State.Status}}' halogen 2>/dev/null)
            echo "[halogen] container: "(string length -q -- "$st"; and echo $st; or echo 'not created')
            if curl -sf -m 3 $url/health -o /tmp/.halogen-health.json 2>/dev/null
                python3 -c "
import json; h=json.load(open('/tmp/.halogen-health.json'))
print('  endpoint : $url/v1  model=%s  ctx=%s' % (h['model'], h['context']))
print('  vision   : %s' % h['vision']['enabled'])
n = h.get('npu') or {}
print('  npu      : %s' % (', '.join(n['models']) if n.get('models') else 'off'))
"
            else
                echo '  endpoint : NOT answering'
            end
            if test -x /usr/local/sbin/halogen-fabric-clock
                echo "[halogen] fabric clock: "(/usr/local/sbin/halogen-fabric-clock status 2>&1 | head -1)
            end
            echo "[halogen] memory: "(free -g | awk 'NR==2 {print $3" GiB used, "$7" available"}')
            return 0

        case restart
            halogen stop
            sleep 2
    end

    # ---- start -------------------------------------------------------------
    if curl -sf -m 2 $url/health >/dev/null 2>&1
        if test "$verb" != restart
            echo "[halogen] already answering on $url/v1 (log: $log)"
            echo $help
            return 0
        end
        podman rm -f halogen >/dev/null 2>&1
    end

    set -l use_npu 1
    set -q _flag_no_npu; and set use_npu 0

    if test $use_npu = 1
        # The NPU locks its models' memory (~8.58 GiB); a rootless container cannot exceed the
        # user manager's hard limit, and /etc/security/limits.d does NOT reach it (systemd spawns
        # user@.service outside PAM). Fail fast with the fix instead of a 3 GiB download + refusal.
        if test (ulimit -H -l) != unlimited
            echo "[halogen] memlock hard limit is "(ulimit -H -l)" KiB, but the NPU needs ~8.58 GiB locked." >&2
            echo "          sudo mkdir -p /etc/systemd/system/user@.service.d" >&2
            echo "          printf '[Service]\\nLimitMEMLOCK=infinity\\n' | sudo tee /etc/systemd/system/user@.service.d/50-memlock.conf" >&2
            echo "          sudo systemctl daemon-reload && reboot     # then: ulimit -H -l -> unlimited" >&2
            echo "          (or run 'halogen -n' for the GPU-only stack, which needs no mlock)" >&2
            return 1
        end
        if not /usr/local/sbin/halogen-fabric-clock status >/dev/null 2>&1
            echo "[halogen] the GPU fabric clock is not held — GPU+NPU together can hang this SoC." >&2
            echo "          sudo systemctl enable --now halogen-fabric-clock.service" >&2
            return 1
        end
    end

    # Host XRT lives in /usr/lib on Arch/CachyOS, and those libraries look each other up at their
    # own path, so each one is mounted twice (see docs/NPU.md). Not needed for the GPU-only stack.
    set -l X
    if test $use_npu = 1
        for f in libxrt_coreutil.so.2 libxrt_core.so.2 libxrt_driver_xdna.so.2
            set -l p (readlink -f /usr/lib/$f)
            set -a X -v $p:/opt/xilinx/xrt/lib/$f:ro -v $p:/usr/lib/$f:ro
        end
    end

    set -l envs \
        -e HALOGEN_CHECKPOINT=$ckpt \
        -e HALOGEN_VISION_TOWER=1 \
        -e HALOGEN_CTX=131072 \
        -e HALOGEN_MAX_TOK=16384 \
        -e HALOGEN_MAX_TOKENS_DEFAULT=16384 \
        -e HALOGEN_MODEL_ID=qwen3.8-flash-next-uncensored
    set -q _flag_fetch; and set -a envs -e HALOGEN_DOWNLOAD=peonist-ai/halogen-qwen3.8-flash-next
    test $use_npu = 1; and set -a envs -e HALOGEN_NPU_MODELS=$npu

    set -l devs --device /dev/kfd --device /dev/dri
    test $use_npu = 1; and set -a devs --device /dev/accel/accel0

    podman rm -f halogen >/dev/null 2>&1
    echo "[halogen] starting (NPU "(test $use_npu = 1; and echo on; or echo off)")… first start repacks the GGUF (~18 s)"

    if set -q _flag_foreground
        podman run --rm --name halogen -p 8731:8731 $devs --group-add keep-groups \
            --ipc=host --ulimit memlock=-1:-1 $X $envs -v $models:/models $image
        return $status
    end

    podman run -d --name halogen -p 8731:8731 $devs --group-add keep-groups \
        --ipc=host --ulimit memlock=-1:-1 $X $envs -v $models:/models $image >/dev/null
    or begin; echo "[halogen] podman run failed" >&2; return 1; end

    for i in (seq 1 90)
        if curl -sf -m 3 $url/health >/dev/null 2>&1
            echo "[halogen] ready at $url/v1  (log: $log)"
            echo $help
            return 0
        end
        if test (podman inspect -f '{{.State.Running}}' halogen 2>/dev/null) != true
            echo "[halogen] container exited — last lines:" >&2
            podman logs --tail 15 halogen >&2
            return 1
        end
        sleep 2
    end
    echo "[halogen] not answering after 180 s — see 'halogen logs --tail 40'" >&2
    return 1
end
```

### ~/.pi/agent/models.json

Added 2026-10-03. pi's two local providers: `local` (:1234, llama-swap: `qwen3-coder`, `qwen3-vl`) and `halogen` (:8731: `qwen3.8-flash-next-uncensored`).

```json
{
  "providers": {
    "local": {
      "baseUrl": "http://127.0.0.1:1234/v1",
      "api": "openai-completions",
      "apiKey": "sk-local",
      "models": [
        { "id": "qwen3-coder", "input": ["text"] },
        { "id": "qwen3-vl", "input": ["text", "image"] }
      ]
    },
    "halogen": {
      "baseUrl": "http://127.0.0.1:8731/v1",
      "api": "openai-completions",
      "apiKey": "sk-local",
      "models": [
        { "id": "qwen3.8-flash-next-uncensored", "input": ["text", "image"] }
      ]
    }
  }
}
```

---

### ~/.config/niri/scripts/lock-before-sleep.sh

Added 2026-09-24 - see [ch. 11](../11-session-lock.md). Run by `swayidle` on `before-sleep`. Lock command switched to Noctalia v5 on 2026-10-19 ([ch. 12](../12-resume-freeze-upgrade-noctalia-v5.md)).

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

noctalia msg session lock

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

## Local model routing (ch. 5, applied 2026-09-24)

Two omp files decide which model does what. Both were changed after the measured comparison in
[ch. 5](../05-local-ai.md) — `qwen3-coder` is no longer the quality pick, and llama-swap serves one
model at a time.

### ~/.omp/agent/models.yml
New file (2026-09-24; the `halogen` provider added 2026-10-03, switched to an explicit model list the
same day). Pins the local models to the truth of their server: `lm-studio` is discovery-based, while
`halogen` declares its rows explicitly with per-model context/output limits. Discovery alone got
things wrong — it claimed a 1M context (omp would compact at ~8× the real window and only learn from
`request (N tokens) exceeds the available context size`) and inverted image capability (it has no way
to see `--mmproj`).

Providers:
- `lm-studio` (:1234, llama-swap, `openai-models-list` discovery): `qwen3-coder` (text) and `qwen3-vl`
  (text + image), both 131072.
- `halogen` (:8731, halogen-flash-server): explicit `models:` allowlist. halogen advertises six rows at
  `/v1/models` but only three are chat models, and omp's discovery cannot tell the runners apart
  (`qwen3-embedding-0.6b` → `/v1/embeddings`, `qwen3-reranker-0.6b` → `/v1/rerank`,
  `qwen3guard-gen-0.6b` → `/v1/moderations`), so all six appeared as selectable chat models that would
  400. Declared chat rows: `qwen3.8-flash-next-uncensored` (text + image, 131072/16384), `qwen3.5-2b`
  (text, 16384/2048) and `decider-0.8b` (text, 4096/32, a classifier/router).

```yaml
# ~/.omp/agent/models.yml — truth-pinning for the two local endpoints.
#
#   lm-studio -> llama-swap on :1234   (qwen3-coder, qwen3-vl): loads on demand, so these are
#                the always-available quick pair and cost nothing while idle.
#   halogen   -> halogen-flash-server on :8731 (the Flash stack: text, tools, vision, and the
#                NPU's embeddings/rerank/moderation). One trunk, resident; started with `halogen`.
#                Replaces lm-studio/qwen3.8-flash-next and .../flash-uncensored (pruned 2026-10-03).
#
# Why an explicit provider entry at all: declaring `lm-studio` replaces the implicit
# discoverable provider, so the discovery block below has to be spelled out or the
# provider ends up with zero models.
#
# Why halogen lists its models EXPLICITLY instead of discovering them (2026-10-03):
# halogen advertises six models at /v1/models, but only three are chat models. The other
# three are runners for other routes — qwen3-embedding-0.6b serves /v1/embeddings,
# qwen3-reranker-0.6b serves /v1/rerank, qwen3guard-gen-0.6b serves /v1/moderations.
# omp's openai-models-list discovery only reclassifies a row as a non-chat runner when the
# endpoint advertises embedding-only output, which halogen does not, so all six showed up
# as selectable chat models and picking one of those would 400. An explicit list is the
# allowlist: three chat-shaped entries, exact context/output limits, no noise.
providers:
  lm-studio:
    baseUrl: http://127.0.0.1:1234/v1
    auth: none
    api: openai-completions
    discovery:
      type: openai-models-list
    modelOverrides:
      qwen3-coder:
        contextWindow: 131072
        input: [text]
      qwen3-vl:
        contextWindow: 131072
        input: [text, image]
  halogen:
    baseUrl: http://127.0.0.1:8731/v1
    auth: none
    api: openai-completions
    models:
      # The Flash stack. Thinking model: halogen closes the think block itself (answer room)
      # rather than returning empty content, so a normal agent max_tokens is enough.
      - id: qwen3.8-flash-next-uncensored
        name: Flash-Next Uncensored (halogen, GPU)
        reasoning: true
        input: [text, image]
        contextWindow: 131072
        maxTokens: 16384
      # NPU. Short jobs beside the big model: summaries, rewrites. No tools, no thinking.
      - id: qwen3.5-2b
        name: Qwen3.5-2B (halogen, NPU)
        reasoning: false
        input: [text]
        contextWindow: 16384
        maxTokens: 2048
      # NPU. One pass returns a probability per option, so it is a classifier/router, not a
      # chat model: give it a question and 2-10 options and read the chosen label (or logprobs).
      - id: decider-0.8b
        name: Decider-0.8B (halogen, NPU classifier)
        reasoning: false
        input: [text]
        contextWindow: 4096
        maxTokens: 32
```

Verify with `omp models find qwen` — the lm-studio table must show `131K` everywhere and
`images: yes` only for `qwen3-vl`; halogen must show `qwen3.8-flash-next-uncensored` at `131K` with images.

### ~/.omp/agent/config.yml

`worker`/`worker2` exist so the local worker model is one line to retarget; both point at the
resident `halogen/qwen3.8-flash-next-uncensored`, where before the 2026-10-03 prune they were
`lm-studio/qwen3.8-flash-next` and `lm-studio/qwen3.8-flash-uncensored`. `sonic` (bundled "mechanical
updates / data collection" agent) is routed to `worker`; `smol` now follows `@default`.

`default` was reassigned from the cloud `deepseek/deepseek-v4-flash` to the same halogen Flash model
(2026-10-03 23:31) — the cloud model is now the **fallback**, not the default.
`retry.fallbackChains.default` still covers the case where the halogen container is not running: if
:8731 is down, a default-model request walks to `deepseek/deepseek-v4-flash` instead of retrying into
a failure.

`tiny`/`memory`/`judge` stay on the on-device ONNX models so background titles and typed judgments
stop swapping a llama-swap model out from under a running agent.

*Note: the live file has no trailing newline, so the block below is exactly one byte larger than the
file on disk — a markdown artifact, not drift.*

```yaml
setupVersion: 2
modelRoles:
  smol: "@default"
  worker: halogen/qwen3.8-flash-next-uncensored
  worker2: halogen/qwen3.8-flash-next-uncensored
  tiny: local/lfm2.5-230m
  memory: local/lfm2-1.2b
  judge: local/lfm2-1.2b
  plan: deepseek/deepseek-v4-flash
  advisor: deepseek/deepseek-v4-flash
  vision: lm-studio/qwen3-vl
  default: halogen/qwen3.8-flash-next-uncensored
defaultThinkingLevel: auto
providers:
  streamFirstEventTimeoutSeconds: 180
retry:
  fallbackChains:
    worker:
      - deepseek/deepseek-v4-flash
    worker2:
      - deepseek/deepseek-v4-flash
    default:
      - deepseek/deepseek-v4-flash
task:
  agentModelOverrides:
    sonic: "@worker"
composer:
  shape: claude
dev:
  autoqaConsent: denied
```


### ~/.config/noctalia/config.toml

Added 2026-10-19: Noctalia v5, translated from v4 `settings.json`. See [ch. 12](../12-resume-freeze-upgrade-noctalia-v5.md).
Validate with `noctalia config validate`. GUI changes go to `~/.local/state/noctalia/settings.toml` and override this file.

```toml
# Noctalia v5 config, translated from the v4 settings.json on 2026-10-19.
# v4 files (settings.json, plugins.json, plugins/) are left untouched as a fallback.
# GUI changes land in ~/.local/state/noctalia/settings.toml and override this file.

[shell]
font_family   = "Bitstream Vera Sans Mono"
time_format   = "{:%H:%M}"
polkit_agent  = true            # replaces the v4 polkit-agent plugin
telemetry_enabled = false
avatar_path   = "~/.face"

[shell.animation]
enabled = true
speed   = 1.0

[shell.shadow]
direction = "down_right"

[shell.launcher]
sort_by_usage = true
pinned = ["Alacritty", "code", "obsidian", "chrome-ompifgpmddkgmclendfeacglnodjjndh-Default", "io.dbeaver.DBeaver"]

[shell.launcher.providers.calculator]
global = true                   # replaces the v4 noctalia-calculator plugin in the launcher

[theme]
mode    = "dark"
source  = "builtin"
builtin = "Tokyo-Night"

[wallpaper]
enabled   = true
fill_mode = "crop"
transition = ["fade", "disc", "stripes", "wipe", "honeycomb"]
transition_duration = 1500

[notification]
background_opacity = 1.0

[osd]
position      = "top_right"
hide_delay_ms = 2000

[lockscreen]
enabled = true

[weather]
enabled = false                 # v4 had weather on but no location set; set [location] first

[nightlight]
enabled           = false
temperature_day   = 6500
temperature_night = 4000

# ── Idle (v4: screensaver 300s, screen-off 1800s, lock 1820s, suspend 1800s) ──
[idle]
pre_action_fade_seconds = 10.0

[idle.behavior.screensaver]
timeout        = 300
action         = "command"
command        = "niri-screensaver-launch launch"
resume_command = "niri-screensaver-launch kill"

[idle.behavior.screen-off]
timeout = 1800
action  = "screen_off"
enabled = true

[idle.behavior.lock]
timeout = 1820
action  = "lock"
enabled = true

[idle.behavior.suspend]
timeout = 1800
action  = "lock_and_suspend"
enabled = true

[hooks]
session_locked   = "niri-screensaver-launch kill"
session_unlocked = "niri-screensaver-launch kill"

# ── Bar ───────────────────────────────────────────────────────────────────────
[bar.main]
position           = "bottom"
background_opacity = 0.93
margin_edge        = 4
margin_ends        = 4
widget_spacing     = 9
capsule            = true
capsule_opacity    = 0.82

start  = ["launcher", "clock", "cpu", "ram", "temp", "active-window", "media"]
center = ["todo", "sticky-notes", "workspaces", "mini-docker", "pomodoro"]
end    = ["keybinds", "recorder", "brightness", "volume", "power", "tray", "notifications", "control-center"]

# Plugin bar widgets (v4 plugin -> v5 community/official equivalent)
[widget.todo]
type = "nightwatch75/todo:todo"

[widget.sticky-notes]
type = "ahmedhossamdev/sticky-notes:bar"

[widget.mini-docker]
type = "8bury/mini-docker:mini-docker"

[widget.pomodoro]
type = "thepunkoff/pomodoro:widget"

[widget.keybinds]
type = "kenn/keybind-cheatsheet:keybinds"

[widget.recorder]
type = "noctalia/screen_recorder:recorder"

[widget.power]
type = "piero-93/battery-power-management:widget"

[widget.clock]
format = "{:%H}\n{:%M}"

[widget.cpu]
type = "sysmon"
stat = "cpu_usage"

[widget.ram]
type = "sysmon"
stat = "ram_pct"

[widget.temp]
type = "sysmon"
stat = "cpu_temp"

# ── Dock ──────────────────────────────────────────────────────────────────────
[dock]
enabled   = true
position  = "bottom"
auto_hide = true
show_dots = false

# ── Control center shortcuts (v4: network, bluetooth, wallpaper, notifications, power profile, keep-awake, night light)
[[control_center.shortcuts]]
type = "wifi"
[[control_center.shortcuts]]
type = "bluetooth"
[[control_center.shortcuts]]
type = "wallpaper"
[[control_center.shortcuts]]
type = "notification"
[[control_center.shortcuts]]
type = "nightlight"
[[control_center.shortcuts]]
type = "session"

# ── Plugins ───────────────────────────────────────────────────────────────────
# v4 polkit-agent -> [shell] polkit_agent; v4 noctalia-calculator -> built-in launcher calculator;
# v4 ssh-sessions -> cleboost/ssh-launcher (launcher provider, no bar widget);
# v4 vscode-provider / osk-toggle / show-keys / niri-screensaver widget: no v5 equivalent yet.
[plugins]
enabled = [
  "noctalia/screen_recorder",
  "nightwatch75/todo",
  "ahmedhossamdev/sticky-notes",
  "thepunkoff/pomodoro",
  "8bury/mini-docker",
  "kenn/keybind-cheatsheet",
  "cleboost/ssh-launcher",
  "davemhammer/obsidian",
  "piero-93/battery-power-management",
]
```
