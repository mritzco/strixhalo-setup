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

