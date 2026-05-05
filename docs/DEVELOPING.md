# Developing GuyIDE

> **Audience:** humans contributing to GuyIDE, and AI coding agents working
> on any of its repos. If you're an LLM and you've been pointed at the
> umbrella repo, **read this file first** — it tells you where every other
> file lives and what state it's in.

GuyIDE is an umbrella project. Almost no code lives in this repo; we are
the front door, the documentation hub, and the integration spec. The
working code is spread across several sibling repos under
[github.com/guysoft](https://github.com/guysoft) plus a few upstream
dependencies we ship as-is.

---

## Repo map

The canonical layout when developing locally is `~/workspace/<repo>`. AI
agents should assume these paths exist and resolve to the active
checkout; if a path is missing, prompt the user before cloning.

| Role | Repo | Local path | Purpose |
|---|---|---|---|
| **Umbrella / front door** | [GuyIDE](https://github.com/guysoft/GuyIDE) | `~/workspace/GuyIDE` | This repo. README, docs, screenshots, install pointer, support matrix narrative. No code. |
| **Installer + agent bridge** | [guyide-cli](https://github.com/guysoft/guyide-cli) | `~/workspace/guyide-cli` | The `guyide` Go binary. Installs/updates/uninstalls the stack, runs `doctor`, exposes the AI ↔ nvim ↔ tmux ↔ DAP bridge. |
| **Editor distribution** | [NvGuy](https://github.com/guysoft/NvGuy) | `~/workspace/vibe/nvim/nvguy` | Neovim distribution (NvChad-based). Bundles the debug stack, menu bar, sessions. |
| **Debugger glue** | [vscodium.nvim](https://github.com/guysoft/vscodium.nvim) | `~/workspace/vscodium.nvim` | Reads `.vscode/launch.json`, exposes `debug-rpc.lua`, ships the `debug-reach` OpenCode skill. |
| **tmux IDE layout** | [tmux-ide](https://github.com/guysoft/tmux-ide) | _(submodule of tmux plugins)_ | 3-pane layout helper bound to `prefix + e`. Exports `NVIM_IDE_SOCK`. |
| **Session restore for AI** | [tmux-resurrect-opencode-sessions](https://github.com/guysoft/tmux-resurrect-opencode-sessions) | _(submodule of tmux plugins)_ | Persists OpenCode conversations across reboots. |

### Component versions (current as of writing)

| Component | Tag | Notes |
|---|---|---|
| guyide-cli | `v0.1.0` | First public tag; M1 of installer landed on `feature/installer`. |
| NvGuy | `v0.1.0` | Bundles `lua/plugins/debug.lua` (single source for the debug stack). |
| vscodium.nvim | `v0.1.0` | `list_configs` returns `{success, configs}` envelope (post-v0.1.0 breaking change consumers must adapt). |

The authoritative compat matrix is baked into the `guyide` binary at
`embed/compat.json` in guyide-cli. If you change a component's API,
update that file and the matching entry under
`embed/support_matrix.yaml`.

---

## Architectural cheatsheet

GuyIDE has three pluggable slots. v0.2 ships exactly one fully-supported
triple: **(nvim, tmux, opencode)**.

```
+-------------------------+    +---------------------+    +----------------+
|        EDITOR           |    |    MULTIPLEXER      |    |     AGENT      |
|                         |    |                     |    |                |
|  driver: nvim           |<-->|  driver: tmux       |<-->|  driver:       |
|  package: NvGuy         |    |  config: ~/.tmux.   |    |    opencode    |
|  plugin: vscodium.nvim  |    |    conf (managed)   |    |  (claude-code  |
|  glue:    nvim-launch   |    |  layout: tmux-ide   |    |    is a stub)  |
+-------------------------+    +---------------------+    +----------------+
            ^                            ^                        ^
            |                            |                        |
            +----------------------------+------------------------+
                                         |
                              +-------------------+
                              |   guyide CLI      |
                              |   ~/.guyide/bin/  |
                              |   guyide          |
                              +-------------------+
```

Data flow during a debug session:

1. The user (or AI agent) presses `Ctrl-a e` → tmux-ide spawns the IDE
   layout and exports `NVIM_IDE_SOCK` to every pane.
2. Agent calls `guyide debug breakpoint set <file>:<line>` → guyide-cli
   dials the socket, talks msgpack-rpc to nvim, calls
   `debug-rpc.lua → set_breakpoint`.
3. Agent calls `guyide debug launch <config>` → vscodium.nvim parses
   `.vscode/launch.json`, nvim-dap starts debugpy/delve/etc., DAP
   protocol flows through nvim.
4. On stop, `guyide debug state --json` returns frames + variables in
   the schema-stable `guyide/v1` envelope.

---

## Where the docs live

| Doc | Path | When to read |
|---|---|---|
| User-facing intro + Quick Start | `~/workspace/GuyIDE/README.md` | Anyone trying to *use* GuyIDE. |
| **This file** | `~/workspace/GuyIDE/docs/DEVELOPING.md` | Anyone hacking on GuyIDE; AI agents at session start. |
| Support matrix (human view) | `~/workspace/guyide-cli/SUPPORT_MATRIX.md` | Before adding a new driver or triple. |
| Support matrix (machine) | `~/workspace/guyide-cli/embed/support_matrix.yaml` | Edit alongside the human view. Baked into the binary. |
| Compat pins (machine) | `~/workspace/guyide-cli/embed/compat.json` | Bump when tagging a new guyide-cli release that pins different component versions. |
| Welcome cheatsheet | `~/workspace/guyide-cli/embed/welcome/WELCOME.md` | The five-key intro shown on first launch. Edit if onboarding flow changes. |
| Output schema | `~/workspace/guyide-cli/pkg/schema/schema.go` | Before changing any `--json` output. Schema string is `guyide/v1`. |
| Manifest schema | `~/workspace/guyide-cli/pkg/schema/manifest.go` | Before changing `~/.guyide/manifest.json` shape. |
| User config schema | `~/workspace/guyide-cli/pkg/schema/userconfig.go` | Before changing `~/.guyide/config.yaml` shape. |
| E2E tests | `~/workspace/guyide-cli/testdata/e2e/` | The breakpoint-flow harness. Keep green. |
| CI workflow | `~/workspace/guyide-cli/.github/workflows/e2e.yml` | E2E pipeline; tweak when test deps change. |

When in doubt: search the umbrella repo first, then `guyide-cli`,
*then* the per-component repos.

---

## Install layout (target machine)

What the `guyide install` command produces under `$HOME`:

```
~/.guyide/
├── bin/guyide              # the binary (symlinked from ~/.local/bin/guyide)
├── config.yaml             # user-editable, drives driver selection
├── manifest.json           # what guyide installed; source of truth
├── channel                 # one line: stable | dev
├── env                     # sourced by tmux profile + agent shell
├── WELCOME.md              # first-launch cheatsheet (rendered from embed/)
├── components/
│   ├── nvim/               # NvGuy clone; ~/.config/nvim symlinked here
│   ├── tmux/               # canonical ~/.tmux.conf source
│   ├── vscodium.nvim/      # debugger glue
│   └── opencode/           # agent (or claude-code/ when wired)
├── backups/<RFC3339>/...   # tarballs; kept forever
└── logs/
```

Anything under `~/.guyide/` is owned by guyide and may be replaced on
update. Anything else is treated as user-owned; the manifest's
`OwnedFiles` list is the deciding factor for what we may overwrite.

---

## Local dev workflow

### Building the CLI from source

guyide-cli ships an **untracked** `build.sh` (gitignored) that builds
the binary against your local component checkouts so you can iterate
without going through CI.

```bash
cd ~/workspace/guyide-cli
./build.sh                       # → ./guyide
./build.sh install               # → symlink to ~/.local/bin/guyide
./build.sh run -- doctor --json  # build + run with args
```

Environment overrides honoured by future install/update flows:

| Env var | Default | Effect |
|---|---|---|
| `GUYIDE_DEV_NVGUY` | `~/workspace/vibe/nvim/nvguy` | Use this dir as the NvGuy source instead of cloning. |
| `GUYIDE_DEV_VSCODIUM` | `~/workspace/vscodium.nvim` | Same for vscodium.nvim. |
| `GUYIDE_HOME` | `~/.guyide` | Test-only: point the install root somewhere else. |

### Running the e2e harness

```bash
cd ~/workspace/guyide-cli
rm -rf /tmp/pytest-of-guy
GUYIDE_E2E_NVGUY_LOCAL=$HOME/workspace/vibe/nvim/nvguy \
GUYIDE_E2E_VSCODIUM_LOCAL=$HOME/workspace/vscodium.nvim \
~/vpy/bin/pytest -xvs testdata/e2e/
```

Expected runtime: ~15s on a warm cache. The harness runs an actual nvim
headless session, sets a breakpoint at `sample.py:14`, and verifies the
process pauses there.

### Tagging order

Components have a strict dependency direction. **Always tag in this
order:**

1. `vscodium.nvim` (no internal deps)
2. `NvGuy` (depends on a tagged vscodium.nvim)
3. `guyide-cli` (depends on tagged refs of both above; bake into
   `embed/compat.json`)

Skipping the order makes `guyide doctor` complain about pinned-but-
missing tags on stable channel.

---

## Channels

| Channel | Source | Compat gating |
|---|---|---|
| `stable` | latest tag of each component | yes — `embed/compat.json` keyed by guyide-cli version |
| `dev` | `main` HEAD of each component | no gating; you're on your own |

Switch with `guyide channel set <stable|dev>`. The default for fresh
installs is `stable`.

---

## Conventions

- **Output**: every command writes through the `internal/output.Writer`
  interface. Never touch `os.Stdout` directly. Three modes: human-styled
  (Charmtone palette via lipgloss), human-plain (no ANSI), machine
  (ndjson tagged with `schema: "guyide/v1"`).
- **Style spec**: `~/workspace/sniplets/CLI_STYLE.md` is the source of
  truth for colours, badges, and whitespace. Do not invent new styles.
- **Schema stability**: `guyide/v1` envelopes are stable. Add fields
  freely; never remove or rename. Bump to `v2` only when truly
  necessary.
- **Filesystem ownership**: guyide may overwrite anything tracked in
  the manifest's `OwnedFiles`. Anything else is user-owned; back up
  before touching, and require `--force` to clobber drift.
- **Backups**: kept forever, RFC3339-stamped tarballs under
  `~/.guyide/backups/`. Restore via `guyide uninstall --backup <ts>`.
- **Tests**: `internal/...` should hover ≥75% coverage. E2E harness
  must stay green before any tag.
- **Commits**: never commit unless the human asks. AGENTS.md applies.
- **Branches**: `feature/<topic>` for in-flight work; `main` is the
  default branch on every repo. PRs are squash-merged.

---

## Troubleshooting orienteering for AI agents

If a user reports a problem, this is the rough decision tree:

1. **"`guyide` command not found"** — `~/.local/bin` likely not on
   `$PATH`. Confirm `~/.guyide/bin/guyide` exists, then ask the user
   to run `export PATH="$HOME/.local/bin:$PATH"` and add it to their
   shell rc.
2. **"plugins missing in nvim"** — Lazy didn't drain. Run `nvim
   --headless "+Lazy! sync" "+qa"` or rerun `guyide install` (idempotent).
3. **"breakpoint doesn't hit"** — start at `guyide doctor`. Then
   verify `.vscode/launch.json` exists and `program` resolves. Then
   check `NVIM_IDE_SOCK` is exported in the pane.
4. **"AI can't drive the debugger"** — confirm
   `~/.config/opencode/skills/debug-reach/SKILL.md` exists; the
   installer copies it from `vscodium.nvim/.opencode/skills/`.
5. **"installer broke my tmux"** — every install creates a backup tarball
   under `~/.guyide/backups/<ts>/`. Restore is `guyide uninstall
   --backup <ts>` or manually `tar xzf` the backup over `$HOME`.

---

## Open work

These are tracked informally; promote to GitHub issues when concrete:

- **M2 (in flight on guyide-cli `feature/installer`)** — `Component`
  interface, nvim driver, cobra `install` subcommand wired to
  `WriteWelcome` + `PromptYesNo`.
- **M3** — tmux driver (full `~/.tmux.conf` ownership, drift detection),
  opencode + claude-code agent drivers (claude-code = stub returning
  `ErrNotImplemented`).
- **M4** — `update`, `uninstall`, `channel`, `config` subcommands;
  doctor `--fix`; auto-rollback on failure.
- **M5** — goreleaser, GitHub Actions release matrix
  (linux/{amd64,arm64} + darwin/{amd64,arm64}), tag guyide-cli v0.2.0.
- **v0.3+** — real claude-code driver, delve/js-debug DAP backends,
  recursive variable walking, OpenCode skill manifest exposed by the
  installer, ndjson streaming for long-running ops, cosign release
  signing.

---

## Contact + license

Each component has its own issue tracker (see the table at the top).
For umbrella concerns — docs, integration, install workflow — open an
issue here at [GuyIDE](https://github.com/guysoft/GuyIDE).

GPL-3.0 across the stack.
