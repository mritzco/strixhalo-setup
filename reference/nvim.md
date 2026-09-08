[← Setup index](../README.md)

# nvim usage

Day-to-day notes for the Neovim setup on this PX13. Config is a **single file**:
`~/.config/nvim/init.lua` (~200 lines, commented), plus `lazy-lock.json` pinning exact
plugin commits.

Running **NVIM v0.12.4** (Arch/CachyOS `/usr/bin/nvim`).

---

## What the setup is made of

| Plugin | Job |
|---|---|
| `lazy.nvim` | plugin manager — clones/updates the others |
| `nvim-treesitter` (**`main` branch**) | AST-aware syntax highlighting |
| `mason.nvim` + `mason-lspconfig` | downloads & auto-enables language servers |
| `nvim-lspconfig` | wires servers into nvim's built-in LSP client |
| `blink.cmp` | completion popup (prebuilt binary, no compile) |
| `telescope.nvim` | fuzzy finder (files, grep, buffers, symbols) |
| `oil.nvim` | edit directories as if they were buffers |
| `tokyonight.nvim` | colorscheme (`tokyonight-night`) |
| `which-key`, `gitsigns`, `lualine`, `nvim-autopairs` | quality of life |

Housekeeping commands: `:Lazy` (plugin status/update), `:Mason` (language servers),
`:checkhealth` (diagnose everything).

---

## Treesitter parsers — the `main` branch gotcha

**This is what broke the setup for weeks.** Worth understanding, because it will bite again.

The config pins `branch = "main"` (`init.lua:51`). Two different worlds:

| | old `master` branch | current `main` branch |
|---|---|---|
| How parsers are built | plugin calls `gcc` itself | plugin shells out to `tree-sitter build` |
| External dependency | none | **the `tree-sitter` CLI binary** |

lazy.nvim installs *Neovim plugins* (git repos). It has no idea a standalone Rust CLI is
needed, and never will. So the CLI is a **system package**, installed once:

```fish
sudo pacman -S tree-sitter-cli
```

Note `tree-sitter` (the C *library*) and `tree-sitter-cli` (the binary) are **separate
packages**. Having the first is not enough — the plugin runs the second.

Verify:
```fish
which tree-sitter   # => /usr/bin/tree-sitter
tree-sitter --version
```

### Symptom when it's missing

```
[nvim-treesitter/install/json] error: Error during "tree-sitter build":
vim/_core/system:324: ENOENT: no such file or directory (cmd): 'tree-sitter'
```

Repeated once per language, **on every single startup** — because nothing ever compiles,
nothing gets cached, so `install()` retries the whole list forever. It looks like a config
error. It isn't; it's one missing binary.

### Where parsers actually live

```fish
ls ~/.local/share/nvim/site/parser/    # the compiled .so files
ls ~/.local/share/nvim/site/queries/   # highlight rules per language
```

Currently built (13): `bash css html javascript json lua markdown markdown_inline python
rust tsx typescript yaml` — the list in `init.lua:55-59`.

### Managing parsers

```vim
:TSInstall go          " add a language
:TSUpdate              " rebuild all
:TSUninstall rust      " remove one
:TSLog                 " see why an install failed
:checkhealth vim.treesitter
```

To make a language install automatically on a fresh machine, add it to the `install({...})`
list in `init.lua`. Startup stays fast — `install()` skips parsers already on disk.

**If you see `curl: (56) Connection died`** — that's the network, not the config. It happens
when 13 grammars download at once. Just re-run `:TSInstall <lang>`.

---

## What's new in modern nvim

The big shift: **things that used to require plugins are now built in.** Coming from vim
tutorials written before ~2024, this is the main thing to un-learn.

### Built-in LSP keymaps (nvim 0.11+) — no plugin needed

These work in any buffer with a language server attached. All start with `g`:

| Key | Does |
|---|---|
| `grn` | rename symbol |
| `grr` | list references (usages) |
| `gri` | go to implementation |
| `gra` | code action (fixes, refactors) |
| `grt` | go to type definition |
| `grx` | run codelens |
| `gO` | document symbols (outline of current file) |
| `K` | hover docs |
| `CTRL-S` *(insert mode)* | signature help |

Mnemonic: **`gr`** = "go, refactor…", then `n`ame / `r`eferences / `i`mplementation /
`a`ction / `t`ype.

### Built-in bracket navigation (nvim 0.11+)

Tim Pope's `vim-unimpaired` used to be mandatory. Now built in:

| Key | Jumps between |
|---|---|
| `]d` / `[d` | diagnostics (errors/warnings) |
| `]q` / `[q` | quickfix entries |
| `]l` / `[l` | location-list entries |
| `]b` / `[b` | buffers |
| `]<Space>` / `[<Space>` | inserts a blank line below/above |

Capitalised versions (`]D`, `]Q`, …) jump to first/last.

### Other built-ins worth knowing

- `gc` — comment/uncomment a motion (`gcc` = current line, `gcap` = a paragraph). No comment plugin needed.
- `gx` — open the URL under the cursor in a browser.

### New in 0.12 specifically

- `ZR` — restart nvim (`:restart`), handy after config changes.
- `:uniq` — deduplicate lines in the buffer.
- `:iput` — like `:put` but fixes the indent.
- `K` in help buffers is now DWIM: puts the cursor on `vim.fn.expand` and hits `K` → jumps to that help topic.
- Markdown files get treesitter highlighting by default.
- `gO` in *any* help/doc buffer shows a table of contents.

---

## Keymaps from this config

Leader is **`<Space>`**. Pause after pressing it and **which-key** pops up the full list —
that popup is the real cheatsheet, this table is just a copy.

### Finding things (Telescope)

| Key | Does |
|---|---|
| `<C-p>` | find files by name |
| `<C-f>` | grep across the project |
| `<Space>b` | switch buffer |
| `<Space>d` | all diagnostics |
| `<Space>k` | **search all keymaps** — use this when you forget one |

### Files

| Key | Does |
|---|---|
| `-` | open the parent directory (oil) |

Oil edits directories **as text**: rename a file by editing its line, delete by `dd`, create
by adding a line, then `:w` to apply. `-` again goes further up.

### LSP (active when a server is attached)

| Key | Does |
|---|---|
| `gd` | go to definition |
| `gD` | go to declaration |
| `K` | hover docs |
| `gi` | go to implementation |
| `<F2>` | rename symbol |
| `ga` | code action |
| `]d` / `[d` | next / previous diagnostic |
| `<Space>e` | show the diagnostic under the cursor in a float |

### Git (gitsigns)

| Key | Does |
|---|---|
| `]c` / `[c` | next / previous changed hunk |
| `<Space>hp` | preview hunk |
| `<Space>hs` | stage hunk |
| `<Space>hr` | reset hunk |
| `<Space>hb` | blame this line |

---

## Why there is no `gr` map (fixed 2026-08-07)

The config used to map `gr` to `Telescope lsp_references`. It was removed, because nvim now
ships `grr`, `gra`, `gri`, `grn`, `grt`, `grx` globally. Verified on this machine:

```
gra gri grn grr grt grx   # all present as global normal-mode maps
timeoutlen = 1000
```

Because longer mappings starting with `gr` exist, pressing `gr` alone made nvim **wait
1000 ms** to see if another key was coming before firing. Tested precedence: typing `grr`
quickly fired the **built-in**, not the config's map. So `gr` was both laggy and a duplicate
of `grr`.

**Lesson, generally:** don't map a prefix of an existing mapping. `gr` is now a reserved
namespace in nvim — any bare `gr` map costs a `timeoutlen` stall. Use `grr` for references.

---

## Learning vim (the part that actually matters)

Plugins are cosmetic. The reason vim earns its reputation is one idea:

> **verb + motion** — you compose commands instead of memorising them.

| Verb | | Motion | |
|---|---|---|---|
| `d` | delete | `w` | word |
| `c` | change (delete + insert) | `}` | paragraph |
| `y` | yank (copy) | `$` | end of line |
| `v` | select | `ap` | a paragraph |
| `gc` | comment | `i(` | inside parentheses |

Any verb combines with any motion. `dw` deletes a word, `cap` changes a paragraph, `gci(`
comments inside parens, `yi"` copies inside quotes. You don't learn *N×M* commands, you
learn *N+M* pieces. `i` = "inner", `a` = "around" — `ci"` changes inside quotes,
`ca"` also eats the quotes.

Highest-leverage things for a beginner, roughly in order:

1. **`:Tutor`** — ships with nvim, ~30 minutes, hands-on. Start here.
2. Learn `ciw`, `ci"`, `ci(`, `dap` — text objects are where the speed comes from.
3. `.` repeats the last change. Combined with text objects it's most of the magic.
4. `/pattern` then `n`/`N` to search; `*` searches for the word under the cursor.
5. `<C-o>` / `<C-i>` — jump back / forward through where you've been. Pairs with `gd`.
6. `:h <topic>` — nvim's help is genuinely excellent, unlike most. `:h text-objects` is a good first read.

Don't bother with: memorising `hjkl` counts, arrow-key guilt, or anyone's 2000-line config.

---

## Troubleshooting

```vim
:checkhealth              " everything
:checkhealth vim.lsp      " why isn't my language server attaching
:checkhealth vim.treesitter
:Lazy                     " plugin state; `U` updates, `L` shows the log
:Mason                    " language server state
:messages                 " scroll back through errors that flashed past
:LspInfo                  " what's attached to this buffer
```

Startup errors, captured without opening the UI:

```fish
nvim --headless -c 'sleep 5' -c 'messages' -c 'qa!' 2>&1 | head -40
```

That one-liner is how the treesitter loop above was diagnosed — it shows the errors as plain
text instead of a wall of red that scrolls past on launch.
