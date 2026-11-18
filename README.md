# nvim-herdr-navigator

Seamless navigation between Neovim splits and [herdr](https://herdr.dev) panes: one set of `ctrl+h/j/k/l` chords moves through Neovim windows and herdr panes as if they were a single layout. A port of the [vim-tmux-navigator](https://github.com/christoomey/vim-tmux-navigator) pattern to herdr.

This repository contains the Neovim plugin (`plugin/`, `lua/`, `doc/`) and the Herdr plugin (`herdr-plugin.toml`, `h-nav`, `lib/select-mru.jq`). Both halves must come from the same release or development checkout.

Full reference documentation lives in the vimdoc: `:h herdr-navigator`.


## Installation

### 1. Herdr

```sh
herdr plugin install kaar/nvim-herdr-navigator
```

Then bind the actions in `~/.config/herdr/config.toml`, using the qualified action ids:

```toml
[[keys.command]]
key = "ctrl+h"
type = "plugin_action"
command = "herdr-navigator.left"
description = "navigate left (nvim/herdr)"

[[keys.command]]
key = "ctrl+j"
type = "plugin_action"
command = "herdr-navigator.down"
description = "navigate down (nvim/herdr)"

[[keys.command]]
key = "ctrl+k"
type = "plugin_action"
command = "herdr-navigator.up"
description = "navigate up (nvim/herdr)"

[[keys.command]]
key = "ctrl+l"
type = "plugin_action"
command = "herdr-navigator.right"
description = "navigate right (nvim/herdr)"

# clear-screen parity: the direct ctrl+l chord is taken by navigation,
# so prefix+ctrl+l sends a literal ctrl+l to the pane (the analogue of
# vim-tmux-navigator's `bind C-l send-keys C-l`).
[[keys.command]]
key = "prefix+ctrl+l"
type = "shell"
command = "\"$HERDR_BIN_PATH\" pane send-keys \"$HERDR_ACTIVE_PANE_ID\" ctrl+l"
description = "send ctrl+l (clear screen) to pane"
```

**Important:** the direct `ctrl+h/j/k/l` chords must *not* also be bound to the `focus_pane_*` actions, or the action binding races the plugin action. Keep only prefix fallbacks there (`focus_pane_left = "prefix+h"`, …).

Reload and check for a clean diagnostics list:

```sh
herdr server reload-config
```

The manifest's startup hook clears the MRU history on server start. Linking or enabling the plugin does not run that hook, so reset existing history by hand in that case. See `:h herdr-navigator-reset`.

### 2. Neovim

With `vim.pack` (Neovim ≥ 0.12):

```lua
vim.pack.add({ "https://github.com/kaar/nvim-herdr-navigator" })
```

No `setup()` call is required. The plugin is a no-op unless nvim runs inside a herdr pane (`$HERDR_ENV == 1`).

[vim-tmux-navigator](https://github.com/christoomey/vim-tmux-navigator) can stay installed alongside: inside herdr this plugin owns the chords (it sets `g:tmux_navigator_no_mappings`, and its mappings win regardless of plugin load order); inside tmux it stays inactive and vim-tmux-navigator works as usual.


## Usage

| Focused pane        | `ctrl+h/j/k/l` does                                       |
| ------------------- | --------------------------------------------------------- |
| shell / Vim / other | moves herdr pane focus (MRU neighbor, else native focus)  |
| nvim                | moves Neovim windows; crosses out to herdr at the edge    |
| fzf                 | `ctrl+j/k` move the selection (chord forwarded)           |

Only Neovim and fzf receive forwarded keys. Vim counts as "other" and moves pane focus directly.

User commands (for custom mappings): `:HerdrNavigateLeft`, `:HerdrNavigateDown`, `:HerdrNavigateUp`, `:HerdrNavigateRight`. Disable the default mappings with `vim.g.herdr_navigator_no_mappings = 1`.

### MRU example

With a tall Pane 1 on the left and Panes 2 and 3 stacked on the right: focus Pane 3, navigate left to Pane 1, then navigate right. Pane 3 receives focus instead of Pane 2, because it is the most recent source of a plugin navigation request among the panes touching Pane 1's right edge.

History records sources of plugin requests only. Mouse clicks and native prefix navigation do not update it. Details: `:h herdr-navigator-mru`.


## Configuration

```lua
require("herdr-navigator").setup({
  herdr_bin = "herdr",   -- herdr binary used to call the action at the layout edge
  save_on_switch = nil,  -- nil | "update" | "wall": write buffers before leaving vim
})
```


## Requirements

- herdr ≥ 0.9.1 (macOS or Linux)
- `jq` on the Herdr plugin runtime's `PATH`. Every navigation action needs it, including key forwarding.
- Neovim ≥ 0.10 (`vim.system` and automatic kitty-keyboard-protocol negotiation, see [notes](#notes--trade-offs))
- Optional `nc` (netcat) with `-U -w 1` support, for MRU focus by ID. Without it, pane navigation uses native geometry.

See `:h herdr-navigator-requirements` for the netcat variants and timeout caveats.


## How it works

1. **Herdr owns the chords.** Each keypress runs the `h-nav` action with `invocation_source: "keybinding"`. For Neovim or fzf, `h-nav` forwards the chord with `herdr pane send-keys`. For other processes, including Vim, it moves pane focus.
2. **Neovim tries its windows first.** At the layout edge, the plugin asynchronously runs `herdr plugin action invoke herdr-navigator.<direction>`. That calls the same action with `invocation_source: "cli"`, which never forwards keys. This prevents a loop back into Neovim.
3. **Pane navigation prefers a remembered neighbor.** `h-nav` records the source pane and focuses the most recently used (MRU) neighbor that touches the source's edge. Without one, it runs native `herdr pane focus --direction`.

```text
Herdr keybinding → action → h-nav (keybinding)
├─ Neovim: forward the key
│  ├─ Window in that direction → focus that window
│  └─ At the edge → invoke the same action via CLI
│                   → h-nav (cli) → pane navigation
├─ fzf: forward the key for fzf to handle
└─ Other (including Vim) → pane navigation
```

```text
Pane navigation
  ↓
Record the source pane
  ↓
Find the most recently used neighbor in that direction
  ↓
Focus that neighbor via the socket (pane.focus)

If there is no neighbor or any step fails:
  → Use Herdr's native directional focus
```

Selection rules, history semantics, fallback conditions, and the socket transport are documented in `:h herdr-navigator-mru`.


## Notes & trade-offs

- **Shadowed shell keys.** Like the tmux original, shells lose direct `ctrl+l` (clear), `ctrl+k` (kill-to-EOL) and `ctrl+h` (backspace) to navigation. `prefix+ctrl+l` restores clear-screen; the others still work in Neovim's insert mode and in fzf.
- **`ctrl+h` vs Backspace.** `ctrl+h` and Backspace share byte `0x08` in legacy terminal encoding, so key forwarding only distinguishes them when the kitty keyboard protocol is active. Neovim ≥ 0.10 negotiates it automatically inside herdr panes, keeping `<C-h>` distinct from `<BS>`.
- **tmux is not supported.** Running tmux inside a herdr pane is unsupported: herdr owns the chords and does not pass them through to tmux. Use herdr panes directly instead.
- **Normal mode only** (for now). Terminal-mode mappings à la vim-tmux-navigator are a planned follow-up.
- **No wrap.** Neither MRU selection nor `herdr pane focus --direction` wraps at the screen edge — the behaviour vim-tmux-navigator ships `no_wrap` to get.
- **Latency and transport.** Every action runs `jq` once; keybindings also query process info; MRU adds one layout query and one socket request. See `:h herdr-navigator-mru` for the cost breakdown and socket timeout caveats.


## Health check

```vim
:checkhealth herdr-navigator
```

It reports whether the plugin is active, the herdr binary is executable, and the server is reachable. It does not check `jq`, `nc`, or the installed actions. Troubleshooting steps for failed edge navigation and missing MRU preference are in `:h herdr-navigator-health`.


## Development

```sh
herdr plugin link /path/to/nvim-herdr-navigator
herdr plugin action list --plugin herdr-navigator
herdr plugin action invoke herdr-navigator.left
herdr plugin log list --plugin herdr-navigator
```

The manual action call demonstrates pane-only CLI navigation, not key forwarding. Script and selector edits apply on the next invocation. After manifest edits, re-link the plugin.

Run the isolated shell tests from the checkout:

```sh
sh -n h-nav
bash -n tests/h-nav-test
shellcheck -s sh h-nav
shellcheck -s bash tests/h-nav-test
bash tests/h-nav-test
```

The harness uses fake Herdr and netcat commands and covers history, geometry, routing, socket responses, fallback, and reset under `sh` and, when available, `dash`. It does not move live panes.


## Credits

- [christoomey/vim-tmux-navigator](https://github.com/christoomey/vim-tmux-navigator) — the pattern, the `is_vim` process matching, and the ergonomics this plugin mirrors.
- [paulbkim-dev/vim-herdr-navigation](https://github.com/paulbkim-dev/vim-herdr-navigation) — prior art that validated the herdr-plugin packaging path.
