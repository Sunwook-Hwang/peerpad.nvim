# peerpad.nvim

Live collaboration between independent Neovim processes, with no plugin dependencies
and no external server executable. Edit together across nodes that share a file over
NFS, or join directly by host, port and token.

The source lives on disk; unsaved edits and peer cursors travel over TCP. Sharing
an NFS path alone does not provide a network connection between the editors.

## Example: pair debugging across servers

Two developers open the same NFS file on different servers. One runs `:Peerpad`;
the other accepts the join prompt. They edit together and see each other's unsaved
changes and cursors. Only the owner saves the shared result to the original file.

## Demo

[![Peerpad demo: edits synchronize between two Neovim processes, with keypresses shown](assets/peerpad-demo.gif)](https://github.com/Sunwook-Hwang/peerpad.nvim/raw/refs/heads/main/assets/peerpad-demo.mp4)

Start sharing, join, edit from either side, save on the owner, and disconnect.
The overlay shows the keys used; this demo uses Space as the leader.
[Watch or download the MP4](https://github.com/Sunwook-Hwang/peerpad.nvim/raw/refs/heads/main/assets/peerpad-demo.mp4).

Recorded from native Neovim UI output on macOS with two independent processes
connected over local TCP. This illustrates the editing workflow; it is not a
recording of separate servers or an NFS deployment. Captions and a keypress overlay
were added. Session credentials are hidden.

## Requirements

- Neovim 0.12 or newer on Linux or macOS.
- A reachable TCP port between participants.
- Git is used only to exclude session credentials when advertising inside a Git repository.
  Without safe exclusion, manual sharing still works.

## Installation

Choose one installation method below.
`<leader>` refers to your configured leader key; set `vim.g.mapleader` before loading plugins.

### Native `vim.pack` (Neovim 0.12+)

In `init.lua`:

```lua
vim.pack.add({
    { src = "https://github.com/Sunwook-Hwang/peerpad.nvim" },
})
require("peerpad").setup({ keymaps = true })
```

### lazy.nvim

Add to your plugin specifications:

```lua
{
    "Sunwook-Hwang/peerpad.nvim",
    lazy = false, -- Register discovery before source files are read.
    main = "peerpad",
    opts = { keymaps = true },
}
```

Only the small setup module loads at startup. Transport and edit algorithms remain
lazy-loaded until starting, joining or discovering an advertised session. Loading only on commands would miss automatic
join discovery for files opened before the plugin loads.

### packer.nvim

For existing packer configurations, inside `require("packer").startup(function(use)`:

```lua
use({
    "Sunwook-Hwang/peerpad.nvim",
    config = function()
        require("peerpad").setup({ keymaps = true })
    end,
})
```

Run `:PackerSync`. Packer is no longer maintained; this example supports existing users.

### vim-plug

Inside your `plug#begin()` / `plug#end()` block:

```vim
Plug 'Sunwook-Hwang/peerpad.nvim'
```

After `call plug#end()`:

```vim
lua require('peerpad').setup({ keymaps = true })
```

Run `:PlugInstall`.

### Local checkout / disconnected machine

Copy the entire package directory and add its absolute path to `init.lua`:

```lua
vim.opt.runtimepath:prepend(vim.fn.expand("~/src/peerpad.nvim"))
require("peerpad").setup({
    discovery = true,
    max_peers = 8, -- Includes the owner; 2–64.
    keymaps = true,
})
```

Change the path to your checkout. There is no dependency on FLASH, dotfiles, a language
server or Treesitter. Commands also register automatically when installed as a native
`pack/*/start` plugin. Discovery performs one filesystem check per source-file read.

The examples follow the official [vim.pack](https://neovim.io/doc/user/pack.html),
[lazy.nvim](https://lazy.folke.io/spec), [packer.nvim](https://github.com/wbthomason/packer.nvim)
and [vim-plug](https://github.com/junegunn/vim-plug) interfaces.

## Keymaps

Enable the suggested normal-mode mappings with `setup({ keymaps = true })`.
They are off by default and existing mappings are never overwritten.
`P` is uppercase (Shift+p); `<leader>Ps` means your leader key, Shift+p, then s.
Descriptions appear in keymap listings and which-key if you already use it; no
which-key dependency is required.

| Key | Command | Purpose |
| --- | --- | --- |
| `<leader>Ps` | `:Peerpad` | Start sharing the current source (s: start) |
| `<leader>Pj` | `:PeerpadJoin` | Join the current file's advertised session (j: join) |
| `<leader>Pq` | `:PeerpadStop` | Disconnect / stop hosting (q: quit) |
| `<leader>Pi` | `:PeerpadStatus` | Show session information (i: info) |

For manual host/port/token entry, use `:PeerpadJoin host port token`.
For manual mappings, replace `keymaps = true` in your installation example with
the setup below. This reproduces the default keys explicitly; change the
left-hand keys if desired. Choose automatic or manual registration, not both:

```lua
require("peerpad").setup({ keymaps = false })

vim.keymap.set("n", "<leader>Ps", "<Cmd>Peerpad<CR>", {
    desc = "Peerpad: start sharing",
})
vim.keymap.set("n", "<leader>Pj", "<Cmd>PeerpadJoin<CR>", {
    desc = "Peerpad: join current file",
})
vim.keymap.set("n", "<leader>Pq", "<Cmd>PeerpadStop<CR>", {
    desc = "Peerpad: disconnect",
})
vim.keymap.set("n", "<leader>Pi", "<Cmd>PeerpadStatus<CR>", {
    desc = "Peerpad: session information",
})
```

## Usage

On the owner, open a named, editable UTF-8 source file, save it and run:

```vim
:Peerpad
```

This opens a dedicated shared buffer and advertises the session beside the source.
Another participant opening that source in their active editor receives a join prompt.
For files that were already open, use `:PeerpadJoin` without arguments.

Manual joining works without shared storage:

```vim
:PeerpadJoin <owner-host> <port> <token>
```

The owner can find that command in `:messages`. The default bind address is `0.0.0.0`
and the port is selected automatically. For local-only collaboration:

```vim
:Peerpad 0 127.0.0.1
```

| Command / key | Action |
| --- | --- |
| `:Peerpad [port] [bind-address]` | Share the current source; at most one session per Neovim process |
| `:PeerpadJoin [host port token]` | Join the current source's advertised session or an explicit session |
| `:PeerpadStatus` | Show role, revision, pending edits and peer cursor positions |
| `:PeerpadStop` | Disconnect; the owner also stops the server |
| `u` / `Ctrl+r` | Undo / redo your own edits in the shared buffer |
| `:w` | Owner saves synchronized text through the original source buffer |

Edit the shared buffer rather than the original source. Guests cannot save the original
through this plugin. Source-buffer changes or external disk changes prevent the owner
from overwriting them. Disconnecting shows a notification. If the shared text
matches the loaded source and no edits are pending, its windows return to the
source without changing splits, and the redundant shared buffer is removed.
Otherwise a `[disconnected]` snapshot remains; copy its text into a normal buffer
to save it. Disconnect never overwrites or reloads the source.

Public Lua entry points mirror the commands:

```lua
require("peerpad").start({ "0", "127.0.0.1" })
require("peerpad").join({ "host", "12345", "token" })
require("peerpad").status()
require("peerpad").stop()
```

## Neovim 0.13 compatibility

On Neovim 0.13, shared buffers and disconnected snapshots are excluded from
native session saves, including `:restart`. A TCP collaboration session cannot
be restored from a `peerpad://` filename. Before restarting, the owner should
save synchronized changes with `:w`; guests should copy any retained snapshot
into a normal file. Reconnect after restarting.

The filter runs only when a session is written, restores buffer options afterward,
and serializes the original source view in place of visible shared buffers. If that
source was removed, an empty view is used. Native autoread, `Q` and `.` behavior is unchanged.
Neovim 0.12 keeps its existing behavior.

## Safety and limits

- The bearer token authorizes access. TCP is not encrypted; use a trusted network or an SSH tunnel.
  Do not publish the token or forward it to people who should not edit the document.
- Sidecars retain the `.<name>.flash-share` filename and protocol used by the
  Peerpad implementation bundled with FLASH, so both can discover and join each other.
- Sidecars and their random temporary files are excluded through local Git `info/exclude`.
  Existing advertised sessions prevent a second owner. If safe advertisement fails,
  the manual command remains available.
- Sidecar readers follow source write-permission classes. This is not a general
  access-control system: ACLs and network security must be configured separately.
- Discovery never prompts for a background preview. If the target window or buffer
  changes during connection, the shared buffer remains available through `:buffer`.
- Concurrent UTF-8 edits use byte-based operational transformation. Final newline is
  retained, and documents are limited to 1 MiB. Undo history, queues and lag are bounded.
- Shared `acwrite` buffers have their own in-memory undo. LSP attachment, formatting and
  other editor features are left to the user's configuration; they are not supplied here.
- Sidecars are removed on clean owner exit. Same-host stale sessions are cleaned up;
  stale files from another host may need manual removal.

Peer colors use `LiveSharePeer1` through `LiveSharePeer6`, linked to the active theme's
native diagnostic/Visual groups. No icon font is required.

[한국어 사용 안내](README.ko.md)
