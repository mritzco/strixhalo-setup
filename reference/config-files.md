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
#
# Paths in `cmd` are literal and machine-specific (shown here as /path/to/...).
# llama-swap expands only its own macros (${PORT}, ${MODEL_ID}): `${HOME}` is
# rejected at startup with `unknown macro`, and neither `$HOME` nor `~` is ever
# seen by a shell, so the real absolute path must be written out.

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
    cmd: /usr/bin/llama-server --model /path/to/models/Qwen3.8-Flash-Next-GGUF/UD-IQ4_XS/Qwen3.8-Flash-Next-UD-IQ4_XS-00001-of-00003.gguf --alias qwen3.8-flash-next -ngl 999 --jinja -fa on -c 131072 --cache-type-k q8_0 --cache-type-v q8_0 -b 8192 -ub 2048 -np 1 --reasoning-effort medium --reasoning-budget 2048 --port ${PORT}
    proxy: http://127.0.0.1:${PORT}
    ttl: 1800

  "qwen3.8-flash-uncensored":
    cmd: /path/to/src/myhacsint-llama.cpp/build-vulkan/bin/llama-server --model /path/to/models/Qwen3.8-Flash-Next-Uncensored-GGUF/IQ4_XS/Qwen3.8-Flash-Next-Uncensored.IQ4_XS.gguf --alias qwen3.8-flash-uncensored -md /path/to/models/Qwen3.8-Flash-Next-Uncensored-GGUF/MTP-unsloth/mtp-Qwen3.8-Flash-Next-shared-Q8_0.gguf --mmproj /path/to/models/Qwen3.8-Flash-Next-Uncensored-GGUF/mmproj/Qwen3.8-Flash-Next-Uncensored.mmproj-f16.gguf --image-min-tokens 1024 -ngl 999 -fa on --jinja -c 131072 --cache-type-k f16 --cache-type-v f16 -b 2048 -ub 1024 --n-cpu-moe 0 --load-mode mmap --tensor-read-lazy auto --no-repack --no-host --fit on --reasoning-effort medium --reasoning-preserve --temp 1.0 --top-k 20 --top-p 0.95 --spec-type draft-mtp --spec-draft-adaptive --spec-draft-n-min 0 --spec-draft-n-max 5 --spec-draft-p-min 0.75 --spec-draft-type-k f16 --spec-draft-type-v f16 --port ${PORT}
    proxy: http://127.0.0.1:${PORT}
    ttl: 1800

  "qwen3.8-27b-vl":
    cmd: /usr/bin/llama-server --model /path/to/models/Qwen3.8-27B-GGUF/Qwen3.8-27B-Q8_0.gguf --mmproj /path/to/models/Qwen3.8-27B-GGUF/mmproj-F16.gguf -ngl 999 --jinja -c 131072 -fa on --alias qwen3.8-27b-vl --port ${PORT}
    proxy: http://127.0.0.1:${PORT}
    ttl: 1800   
# DeepSeek-R1-70B removed 2026-07-16 — dense/slow locally; use DeepSeek via cloud subscription instead.
```

