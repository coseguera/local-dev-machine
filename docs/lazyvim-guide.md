# LazyVim guide (Pocket Dev edition)

A short cheat-sheet for the LazyVim setup this repo ships. It is **not** a LazyVim manual -
for the full picture see the upstream docs:

- LazyVim: <https://www.lazyvim.org/>
- Keymaps: <https://www.lazyvim.org/keymaps>
- Neovim: `:help` and <https://neovim.io/doc/>

LazyVim's leader key is **Space** (`<leader>` below = Space). Press `<leader>` and pause to
let **which-key** show available follow-ups - that menu is the real cheat-sheet.

## What's shipped (and customised)

- **LazyVim starter**, cloned idempotently to `~/.config/nvim`.
- **Tokyo Night** colorscheme (LazyVim default) - matched by the `foot` palette on the
  local console.
- **Nerd Font glyphs** - JetBrainsMono Nerd Font (installed Pi-side for the console;
  installed on the Mac for the SSH path).
- **snacks.nvim terminal** override: the `<C-/>` toggle is a **large centered float**
  (90% x 90%, rounded border) so the Copilot CLI has room. See
  `~/.config/nvim/lua/plugins/snacks.lua` (written by cloud-init).
- **Terminal-agnostic toggle fallbacks** (because macOS Terminal.app can't send `<C-/>`):
  `Ctrl+t` and `<leader>tt`.

## The terminal toggle (where Copilot CLI lives)

| Key | Where it works | Action |
|---|---|---|
| `<C-/>` | foot console, iTerm2, most terminals | Toggle the snacks terminal float |
| `Ctrl+t` | **any** terminal (incl. Terminal.app) | Toggle the float (normal **and** terminal mode) |
| `<leader>tt` | any terminal | Toggle the float |

Open the float, then run `copilot` inside it. Toggle again to hide it without killing the
session.

## Everyday keymaps (LazyVim defaults)

| Key | Action |
|---|---|
| `<leader>` (then wait) | which-key menu of everything |
| `<leader><space>` | Find files (Telescope) |
| `<leader>/` | Live grep (search in project) |
| `<leader>e` | File explorer (Neo-tree) |
| `<leader>gg` | lazygit |
| `<leader>bd` | Close buffer |
| `<leader>w` | Window commands |
| `<S-h>` / `<S-l>` | Previous / next buffer |
| `<leader>qq` | Quit all |
| `gd` / `gr` | LSP go-to-definition / references |
| `K` | Hover docs |
| `<leader>cf` | Format buffer |

## Tools wired in

- **lazygit** (`<leader>gg`) - full-screen git UI, with **delta** for diffs.
- **Telescope** - fuzzy find files/grep/buffers/symbols (`<leader>` + `f`/`s`).
- **Neo-tree** (`<leader>e`) - file tree.
- **Treesitter / LSP** - syntax + language servers via Mason (`:Mason`).
- **ripgrep / fd / fzf** - back the search pickers.

## Plugins & updates

- Plugins live in `~/.config/nvim/lua/plugins/` (LazyVim convention). Our only addition is
  `snacks.lua`.
- `:Lazy` - plugin manager UI (install/update/sync). cloud-init pre-runs `:Lazy! sync`
  headlessly so first launch is fast.
- `:LazyExtras` - enable LazyVim language/feature extras.
- `:checkhealth` - diagnose missing tools.

## Glyphs look wrong (boxes / `?`)

The glyphs render in whatever terminal is **displaying** nvim, not on the Pi:

- **Local console:** the Pi-side Nerd Font is already configured in `foot`.
- **Over SSH:** install JetBrainsMono Nerd Font on the client and select it - see the
  client-setup section in [`usb-gadget.md`](usb-gadget.md).
