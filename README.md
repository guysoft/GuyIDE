# GuyIDE

A modular terminal IDE where your AI agent can **control the debugger**, your sessions **survive restarts**, and everything runs in the terminal.

```
+---------------------------+--------------+
|                           |              |
|   nvim (editor/debugger)  |   AI agent   |
|                           |              |
|                           |              |
|---------------------------|              |
|   terminal                |              |
|   (build/run output)      |              |
+---------------------------+--------------+
```

One keybinding. Full IDE. Close your laptop, reopen — everything is exactly where you left off.

---

## Why GuyIDE?

**Your AI can debug for you.** Tell it "find why `process_data()` returns None" and it will set breakpoints, run the debugger, inspect variables, and report back what's wrong. No other terminal setup can do this.

**Sessions survive anything.** Editor state, AI conversation, terminal working directory — all restored automatically after a crash or reboot. Like Cursor's session restore, but in the terminal.

**The AI sees what you see.** It shares your tmux session — it can read build output, test failures, and server logs from the terminal pane and react to them.

**Everything is swappable.** Don't like neovim? Use Emacs. Prefer Claude Code over OpenCode? One config line. The workflow is the product.

---

## Quick Start

Works on Linux, macOS, and Windows (WSL).

### 1. tmux + plugins

```bash
# Install tpack (plugin manager)
brew install tmuxpack/tpack/tpack

# Clone the IDE layout plugin
git clone https://github.com/guysoft/tmux-ide ~/.tmux/plugins/tmux-ide
~/.tmux/plugins/tmux-ide/install.sh
```

### 2. NvGuy (neovim distribution)

```bash
mv ~/.config/nvim ~/.config/nvim.bak  # backup existing config
git clone https://github.com/guysoft/NvGuy.git ~/.config/nvim
nvim  # plugins install automatically on first launch
```

### 3. OpenCode (AI agent)

```bash
curl -fsSL https://opencode.ai/install | bash
```

### 4. Launch

```bash
tmux
cd ~/your-project
# Press Ctrl-a e — IDE layout appears
```

---

## Features

### Breakthrough

| Feature | What it does |
|---------|-------------|
| **AI-controlled debugging** | The AI sets breakpoints, launches the debugger, inspects variable state, and reports findings — all programmatically via RPC |
| **Full session continuity** | Layout, editor session, AI conversation, and terminal state all survive restarts. Auto-saves every 15 minutes |
| **AI terminal observability** | The agent reads your terminal pane — it sees build errors, test output, and logs as they happen |
| **Shared editor bridge** | Any process in the tmux session can control neovim via the exposed RPC socket (`NVIM_IDE_SOCK`) |

### Standard

- 3-pane IDE layout with one keybinding (`Ctrl-a e`)
- VSCode-compatible `.vscode/launch.json` for Run and Debug configurations
- Full DAP debugger with UI panels (scopes, watches, stack frames, console)
- Menu bar in neovim (`F10` or `<leader>m`) — File, Edit, View, Git, Run, Tools, Window, and more
- Session auto-save/restore per project directory
- Vim-style pane navigation between tmux and nvim (`h/j/k/l`)
- Mouse support enabled
- Git integration — Neogit (magit-style UI), Gitsigns (inline blame, hunk ops), Telescope git pickers
- LSP + completion via Mason, nvim-lspconfig, nvim-cmp
- Treesitter syntax highlighting
- Code minimap sidebar
- File explorer (nvim-tree)
- Fuzzy finder for files, buffers, symbols, diagnostics (Telescope)
- Code formatting via conform.nvim
- Tmux prefix: `Ctrl-a` (screen-style)

---

## Screenshots

<!-- TODO: Add screenshots -->

| Feature | Screenshot |
|---------|-----------|
| IDE Layout | ![IDE Layout](screenshots/ide-layout.png) |
| AI Debugging | ![Debug Reach](screenshots/debug-reach.png) |
| Menu Bar | ![Menu Bar](screenshots/menu-bar.png) |
| Session Restore | ![Session Restore](screenshots/session-restore.png) |
| Debug UI | ![Debug UI](screenshots/debug-ui.png) |

---

## Key Bindings

### tmux (prefix: `Ctrl-a`)

| Binding | Action |
|---------|--------|
| `Ctrl-a e` | Create IDE layout |
| `Ctrl-a h/j/k/l` | Navigate panes |
| `Ctrl-a c` | New window |
| `Ctrl-a Ctrl-s` | Save session |
| `Ctrl-a Ctrl-r` | Restore session |

### Neovim

| Binding | Action |
|---------|--------|
| `F10` | Menu bar |
| `F5` | Run |
| `F6` | Debug |
| `F9` | Toggle breakpoint |
| `F11` / `Shift-F11` | Step into / out |
| `Shift-F5` | Stop debugger |

---

## Architecture

Three independent, replaceable layers:

| Layer | Default | Alternatives |
|-------|---------|--------------|
| **Window Manager** | tmux | zellij, screen |
| **Editor** | NvGuy (neovim) | Emacs, Helix, Vim |
| **AI Agent** | OpenCode | Claude Code, Aider, Goose |

Swap in `~/.tmux.conf`:

```bash
set -g @ide-agent "claude"   # swap AI agent
set -g @ide-editor "emacs"   # swap editor
```

---

## Components

### tmux Layer

| Plugin | Purpose |
|--------|---------|
| [tmux-ide](https://github.com/guysoft/tmux-ide) | 3-pane IDE layout (`prefix + e`) |
| [tmux-resurrect](https://github.com/tmux-plugins/tmux-resurrect) | Save/restore sessions |
| [tmux-continuum](https://github.com/tmux-plugins/tmux-continuum) | Auto-save every 15 min |
| [tmux-resurrect-opencode-sessions](https://github.com/guysoft/tmux-resurrect-opencode-sessions) | Preserve AI conversations across restarts |
| [vim-tmux-navigator](https://github.com/christoomey/vim-tmux-navigator) | Seamless pane navigation |

### Editor Layer — [NvGuy](https://github.com/guysoft/NvGuy)

Neovim distribution (NvChad-based) with:
- Menu bar via vim-quickui (13 menus)
- [vscodium.nvim](https://github.com/guysoft/vscodium.nvim) — VSCode-like Run/Debug, reads `launch.json`
- Full DAP debugger + UI panels
- Session management (auto-save per directory)
- Git UI (Neogit + Gitsigns)

### AI Agent Layer — [OpenCode](https://opencode.ai)

Terminal AI coding agent that:
- Lives in the right pane with full project access
- Persists conversations across restarts
- Controls the debugger via the **debug-reach** skill
- Reads terminal output from other panes

---

## AI Debugging Setup (debug-reach skill)

The debug-reach skill lets the AI agent control nvim's debugger. It ships with vscodium.nvim.

### Install the skill

```bash
# If you installed NvGuy, the skill is already bundled
# Otherwise, grab it from vscodium.nvim:
git clone https://github.com/guysoft/vscodium.nvim.git /tmp/vscodium-nvim
mkdir -p ~/.config/opencode/skills
cp -r /tmp/vscodium-nvim/.opencode/skills/debug-reach ~/.config/opencode/skills/
```

### Use it

Just ask the AI to debug something:

```
"Debug why test_login fails — step through it"
"Set a breakpoint at main.py:42 and check what x is"
"Find what value user_id has when the error occurs"
```

The AI handles breakpoints, launches the session, inspects state, and reports back.

### How it works (under the hood)

All three components collaborate:

| Component | Role |
|-----------|------|
| [tmux-ide](https://github.com/guysoft/tmux-ide) | Exposes `NVIM_IDE_SOCK` (nvim's RPC socket) to the tmux session |
| [vscodium.nvim](https://github.com/guysoft/vscodium.nvim) | Provides `debug-rpc.lua` — the RPC API the agent calls |
| [NvGuy](https://github.com/guysoft/NvGuy) | Wires up nvim-dap, dap-ui, mason-nvim-dap |

---

## Full Installation Details

<details>
<summary>Complete tmux.conf reference</summary>

```bash
set -g prefix C-a
unbind C-b
bind C-a send-prefix

setw -g mode-keys vi
bind-key h select-pane -L
bind-key j select-pane -D
bind-key k select-pane -U
bind-key l select-pane -R

set -g mouse on

# Plugins
set -g @plugin 'tmux-plugins/tmux-resurrect'
set -g @plugin 'tmux-plugins/tmux-continuum'
set -g @plugin 'guysoft/tmux-resurrect-opencode-sessions'
set -g @plugin 'christoomey/vim-tmux-navigator'
set -g @plugin 'guysoft/tmux-ide'

# Session restore
set -g @continuum-restore 'on'
set -g @resurrect-processes '~opencode'

# Pane creation in current directory
bind c new-window -c "#{pane_current_path}"
bind '"' split-window -v -c "#{pane_current_path}"
bind % split-window -h -c "#{pane_current_path}"

# Initialize tpack (keep at very bottom)
run 'tpack init'
```

</details>

<details>
<summary>Prerequisites</summary>

- git, curl
- tmux >= 2.0
- Neovim >= 0.9
- Node.js >= 18 (for OpenCode)
- sqlite3 (for session restoration)
- [tpack](https://github.com/tmuxpack/tpack) or [TPM](https://github.com/tmux-plugins/tpm) (plugin manager)

</details>

<details>
<summary>Step-by-step with TPM instead of tpack</summary>

If you prefer TPM over tpack, replace `run 'tpack init'` with:

```bash
set -g @plugin 'tmux-plugins/tpm'
run '~/.tmux/plugins/tpm/tpm'
```

Then press `prefix + I` to install plugins.

</details>

---

## Contributing

Each component has its own repo:

| Component | Repo |
|-----------|------|
| tmux-ide | https://github.com/guysoft/tmux-ide |
| vscodium.nvim | https://github.com/guysoft/vscodium.nvim |
| NvGuy | https://github.com/guysoft/NvGuy |
| tmux-resurrect-opencode-sessions | https://github.com/guysoft/tmux-resurrect-opencode-sessions |

For GuyIDE meta-repo issues (documentation, integration, workflow): open an issue here.

## License

GPL-3.0. See [LICENSE](LICENSE) for details.
