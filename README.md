# GuyIDE

A modular terminal IDE framework: **tmux** as the window manager, **neovim** as the editor/debugger, and an **AI coding agent** — all working together through shared RPC and session continuity.

```
+---------------------------+--------------+
|                           |              |
|   nvim (editor/debugger)  |   opencode   |
|   NvGuy distro            |   (AI agent) |
|                           |              |
|---------------------------|              |
|   terminal                |              |
|   (build/run output)      |              |
+---------------------------+--------------+
```

One keybinding (`Ctrl-a e`) creates this layout. Close your laptop, reopen it — everything is exactly where you left off: editor session, AI conversation, terminal state.

---

## Breakthrough Features

These are things no other terminal IDE setup can do:

### AI-Controlled Debugging (debug-reach)

Your AI agent can **programmatically control the debugger**. It sets breakpoints, launches debug sessions, steps through code, and inspects variable state — all via neovim's RPC socket exposed through tmux. The AI can literally stop your program at a suspicious line, read the local variables, and tell you what's wrong.

```
Agent (OpenCode)                    Neovim (nvim-dap)
     |                                    |
     |-- set breakpoint file.py:42 ------>|
     |-- launch debug session ----------->|
     |                                    | (hits breakpoint)
     |<-- stopped at line 42, reason: bp -|
     |-- evaluate expression "x" -------->|
     |<-- x = {"corrupted": true} --------|
     |                                    |
     "Found it — x is corrupted at line 42"
```

No other terminal setup offers this. The AI doesn't just read your code — it runs your code under a debugger and finds the exact state that causes bugs.

### Session Continuity Across Restarts

Like Cursor or VSCode's session restore, but for the terminal:

- **tmux-resurrect** saves and restores the full pane layout
- **tmux-resurrect-opencode-sessions** resumes the exact AI conversation (not a fresh session — the same one, with full context)
- **possession.nvim** auto-restores your editor session per directory (open files, cursor positions, undo history)
- **tmux-continuum** auto-saves every 15 minutes

Your laptop dies. You reboot. `tmux` starts. Everything is back — the AI remembers what you were debugging, nvim has your files open, the terminal is in the right directory.

### AI Reads Your Terminal

The AI agent shares the tmux session. It can read output from the terminal pane (build errors, test results, server logs) and react. It sees what you see.

### Shared RPC Bridge

The `NVIM_IDE_SOCK` environment variable exposes neovim's RPC socket to all panes in the tmux session. Any process (including the AI agent) can programmatically control the editor — jump to files, set breakpoints, read buffer contents, trigger commands.

---

## Features

- **3-pane IDE layout** with one keybinding (`Ctrl-a e`)
- **VSCode-compatible** `.vscode/launch.json` for Run and Debug configurations
- **Full DAP debugger** with UI panels (scopes, watches, stack frames, console)
- **Menu bar in neovim** (`F10` or `<leader>m`) — File, Edit, View, Git, Run, Tools, Window, and more
- **Session auto-save/restore** per project directory
- **Vim-style pane navigation** (`h/j/k/l` between tmux panes)
- **Mouse support** enabled
- **Git integration** — Neogit (magit-style), Gitsigns, Telescope git pickers
- **LSP + completion** via Mason, nvim-lspconfig, nvim-cmp
- **Code minimap** sidebar
- **Tmux prefix**: `Ctrl-a` (screen-style)

---

## Architecture

GuyIDE is three independent layers. Each is replaceable:

| Layer | Default | Role | Alternatives |
|-------|---------|------|--------------|
| **Window Manager** | tmux | Holds the framework, manages panes/sessions | zellij, screen |
| **Editor** | NvGuy (neovim distro) | Code editing, debugging, LSP, menus | Emacs, Helix, vim |
| **AI Agent** | OpenCode | AI coding assistant with terminal access | Claude Code, Aider, Goose, Cursor (via terminal) |

The **workflow** is the product, not any single tool. The key insight: the AI agent lives in a tmux pane alongside your editor and terminal. It can:
- Read tmux pane content (see build output, test results)
- Send commands to the terminal pane
- Control the editor via RPC (set breakpoints, navigate to files)
- Persist its conversation across restarts

---

## Components

### 1. tmux Layer (Window Manager + Session Framework)

| Plugin | Purpose |
|--------|---------|
| [tmux-ide](https://github.com/guysoft/tmux-ide) | Creates the 3-pane IDE layout with `prefix + e` |
| [tmux-resurrect](https://github.com/tmux-plugins/tmux-resurrect) | Saves/restores tmux sessions (panes, layout, directories) |
| [tmux-continuum](https://github.com/tmux-plugins/tmux-continuum) | Auto-saves every 15 min, auto-restores on tmux start |
| [tmux-resurrect-opencode-sessions](https://github.com/guysoft/tmux-resurrect-opencode-sessions) | Preserves AI agent sessions across restarts |
| [vim-tmux-navigator](https://github.com/christoomey/vim-tmux-navigator) | Seamless `Ctrl-h/j/k/l` navigation between vim and tmux panes |
| [tpack](https://github.com/tmuxpack/tpack) | Plugin manager (modern TPM replacement) |

### 2. Editor Layer (NvGuy — Neovim Distribution)

[NvGuy](https://github.com/guysoft/NvGuy) is a neovim distribution built on NvChad that adds:

- **Menu bar** via vim-quickui — 13 menus covering all operations (`F10`)
- **Run/Debug** via [vscodium.nvim](https://github.com/guysoft/vscodium.nvim) — reads `.vscode/launch.json`, runs in tmux pane, debugs via nvim-dap
- **Full DAP setup** — mason-nvim-dap auto-installs debugpy, delve, etc.
- **Session management** — possession.nvim with auto-save per working directory
- **Git UI** — Neogit (magit-style) + Gitsigns
- **Code minimap** — mini.map sidebar
- **LSP, Treesitter, Telescope, nvim-tree** — all from NvChad base

### 3. AI Agent Layer (OpenCode)

[OpenCode](https://opencode.ai) is a terminal-based AI coding agent. In GuyIDE it:

- Runs in the right pane with full terminal access
- Continues sessions across restarts (via tmux-resurrect-opencode-sessions)
- Can control the debugger via the debug-reach skill
- Has access to project files and the terminal pane

---

## Installation

### Prerequisites

- Linux, macOS, or Windows (via WSL)
- git, curl
- Node.js >= 18 (for OpenCode)
- Neovim >= 0.9
- tmux >= 2.0
- sqlite3 (for session restoration)

### Step 1: Install tmux and Plugins

```bash
# Install tpack (tmux plugin manager)
brew install tmuxpack/tpack/tpack
# Or see: https://github.com/tmuxpack/tpack#installation

# Add to ~/.tmux.conf:
cat >> ~/.tmux.conf << 'EOF'
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
EOF

# Install plugins
tmux source ~/.tmux.conf
# Then in tmux: prefix + I
```

### Step 2: Install NvGuy (Neovim Distribution)

```bash
# Back up existing config
mv ~/.config/nvim ~/.config/nvim.bak

# Clone NvGuy
git clone https://github.com/guysoft/NvGuy.git ~/.config/nvim

# Launch nvim — plugins install automatically
nvim
```

### Step 3: Install OpenCode (AI Agent)

```bash
# Install OpenCode
curl -fsSL https://opencode.ai/install | bash

# Verify
opencode --version
```

### Step 4: Install the debug-reach Skill

The debug-reach skill lets OpenCode control nvim's debugger. It ships with vscodium.nvim:

```bash
# Copy the skill to your OpenCode skills directory
cp -r ~/.config/nvim/lua/nvim-launch/../../../.opencode/skills/debug-reach \
    ~/.opencode/skills/debug-reach

# Or if using vscodium.nvim standalone:
git clone https://github.com/guysoft/vscodium.nvim.git /tmp/vscodium-nvim
cp -r /tmp/vscodium-nvim/.opencode/skills/debug-reach ~/.opencode/skills/debug-reach
```

### Step 5: Create Your First IDE Session

```bash
# Start tmux
tmux

# Navigate to a project
cd ~/your-project

# Press Ctrl-a e to create the IDE layout
# Or from command line:
ide ~/your-project
```

---

## Key Bindings

### tmux (prefix: `Ctrl-a`)

| Binding | Action |
|---------|--------|
| `Ctrl-a e` | Create IDE layout (editor + agent + terminal) |
| `Ctrl-a h/j/k/l` | Navigate between panes (vim-style) |
| `Ctrl-a c` | New window (in current directory) |
| `Ctrl-a "` | Split horizontal (in current directory) |
| `Ctrl-a %` | Split vertical (in current directory) |
| `Ctrl-a Ctrl-s` | Save session (tmux-resurrect) |
| `Ctrl-a Ctrl-r` | Restore session (tmux-resurrect) |
| `Ctrl-a r` | Reload tmux config |

### Neovim (NvGuy)

| Binding | Action |
|---------|--------|
| `F10` / `<leader>m` | Open menu bar |
| `F5` | Run without debugging |
| `F6` | Start debugging |
| `F9` | Toggle breakpoint |
| `F11` | Step into |
| `Shift-F11` | Step out |
| `Shift-F5` | Stop debugger |
| `Ctrl-F5` | Run last configuration |
| `Ctrl-F6` | Debug last configuration |
| `<leader>du` | Toggle debug UI |

---

## Usage

### Basic Workflow

1. **Open tmux** and navigate to your project
2. **Press `Ctrl-a e`** — the IDE layout appears: nvim (top-left), terminal (bottom-left), OpenCode (right)
3. **Edit code** in nvim with full LSP, completion, and git integration
4. **Ask the AI** in the OpenCode pane — it can see your project, run commands, and control the debugger
5. **Run/Debug** via the Run menu (`F10` → Run) or keybindings (`F5`/`F6`)
6. **Close your laptop** — tmux-continuum saves state every 15 minutes
7. **Reopen** — everything is restored: layout, editor session, AI conversation

### AI-Driven Debugging Workflow

1. Tell the AI agent: "debug why `process_data()` returns None"
2. The AI sets a breakpoint at the suspicious line via RPC
3. The AI launches the debug session
4. The debugger hits the breakpoint — dap-ui opens showing variables
5. The AI inspects local variables, evaluates expressions
6. The AI reports: "Found it — `data` is None because the API returned 404 on line 38"

### Run Without Debugging

1. Create a `.vscode/launch.json` in your project (or let vscodium.nvim generate one)
2. Press `F5` or use the Run menu
3. Output appears in the terminal pane (bottom-left)

---

## Alternative Components

GuyIDE's power is the workflow, not any specific tool. Swap components as you prefer:

| Layer | Default | Swap for... |
|-------|---------|-------------|
| Window Manager | tmux | zellij, screen, i3/sway (tiling WM) |
| Editor | NvGuy (neovim) | Emacs (with dap-mode), Helix, Vim, Kakoune |
| AI Agent | OpenCode | Claude Code, Aider, Goose, Copilot CLI |
| Plugin Manager | tpack | TPM |
| Theme | tmux-oasis | catppuccin, dracula, gruvbox |

To swap the AI agent, set in `~/.tmux.conf`:

```bash
set -g @ide-agent "claude"  # or "aider", "goose", etc.
```

To swap the editor:

```bash
set -g @ide-editor "emacs"  # or "hx", "vim", etc.
```

---

## Screenshots

<!-- TODO: Add screenshots demonstrating each feature -->

### IDE Layout
*Screenshot: The 3-pane layout with nvim, OpenCode, and terminal*

![IDE Layout](screenshots/ide-layout.png)

### AI-Driven Debugging
*Screenshot: AI setting breakpoints and inspecting state via debug-reach*

![Debug Reach](screenshots/debug-reach.png)

### Menu Bar
*Screenshot: NvGuy's quickui menu bar with Run menu open*

![Menu Bar](screenshots/menu-bar.png)

### Session Restore
*Screenshot: Full IDE restored after tmux restart — same AI conversation, same editor state*

![Session Restore](screenshots/session-restore.png)

### Debug UI
*Screenshot: nvim-dap-ui panels showing scopes, watches, stack frames*

![Debug UI](screenshots/debug-ui.png)

---

## Skill Setup (debug-reach)

The **debug-reach** skill allows the AI agent to programmatically control nvim's DAP debugger. It requires all three components working together:

| Component | Repo | Role |
|-----------|------|------|
| tmux-ide | [guysoft/tmux-ide](https://github.com/guysoft/tmux-ide) | Exposes `NVIM_IDE_SOCK` so the agent discovers nvim's RPC socket |
| vscodium.nvim | [guysoft/vscodium.nvim](https://github.com/guysoft/vscodium.nvim) | Provides `debug-rpc.lua` module and the skill instructions |
| NvGuy | [guysoft/NvGuy](https://github.com/guysoft/NvGuy) | Wires up nvim-dap, dap-ui, mason-nvim-dap, and the Run menu |

### How it works

1. tmux-ide launches nvim with `--listen $NVIM_IDE_SOCK`
2. The AI agent discovers the socket via `tmux show-environment NVIM_IDE_SOCK`
3. The agent calls `debug-rpc.lua` functions via `nvim --server $SOCK --remote-expr`
4. Breakpoints are hit, dap-ui auto-opens, and the agent inspects state

### Installing the skill

```bash
# The skill lives in vscodium.nvim's repo
# If you installed NvGuy (which includes vscodium.nvim), it's already available

# For OpenCode, ensure the skill is in your skills directory:
mkdir -p ~/.opencode/skills
cp -r /path/to/vscodium.nvim/.opencode/skills/debug-reach ~/.opencode/skills/

# Verify it's loaded — in OpenCode, the debug-reach skill should appear
# when you ask about debugging
```

### Using the skill

In OpenCode, simply ask it to debug something:

```
"Set a breakpoint at main.py line 42 and run the debugger"
"Debug why test_login fails — step through it"
"Find what value `user_id` has when the error occurs"
```

The AI handles the rest — setting breakpoints, launching sessions, inspecting state, and reporting findings.

---

## Contributing

Contributions welcome! Each component has its own repo:

- **tmux-ide**: https://github.com/guysoft/tmux-ide
- **vscodium.nvim**: https://github.com/guysoft/vscodium.nvim
- **NvGuy**: https://github.com/guysoft/NvGuy
- **tmux-resurrect-opencode-sessions**: https://github.com/guysoft/tmux-resurrect-opencode-sessions

For GuyIDE meta-repo issues (documentation, integration, workflow ideas): open an issue here.

---

## License

GPL-3.0. See [LICENSE](LICENSE) for details.
