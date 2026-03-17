# alphaNode · development stack

> *"Engineering systems that make failure mathematically expensive, bounded, and observable."*

A researched, production-grade development environment for **CachyOS · x86-64-v3 · i9-14900HX · RTX 5070 Mobile**.
Audited and rebuilt for March 2026. Every tool chosen on current evidence, not habit.

![ISA](https://img.shields.io/badge/ISA-x86--64--v3-informational?style=flat-square)
![uv](https://img.shields.io/badge/uv-0.10.x-blueviolet?style=flat-square)
![Ollama](https://img.shields.io/badge/Ollama-0.14%2B-black?style=flat-square)
![Shell](https://img.shields.io/badge/shell-zsh%20%2B%20starship-orange?style=flat-square)
![AUR](https://img.shields.io/badge/AUR%20helper-paru-blue?style=flat-square)

---

## Stack Map

| # | Layer | Tools |
|:--|:------|:------|
| 01 | Shell environment | zsh · starship · zsh-autosuggestions · zsh-syntax-highlighting |
| 02 | Version control | git · github-cli · lazygit · delta |
| 03 | Modern CLI primitives | eza · bat · ripgrep · fd · btop · zoxide |
| 04 | Session management | tmux · fzf · atuin |
| 05 | Python runtime | uv (replaces pip · venv · pyenv · pip-tools) |
| 06 | Containerisation | docker · docker-compose |
| 07 | Agentic AI toolchain | ollama · claude-code · opencode · open-webui |
| 08 | IDE | visual-studio-code-bin |
| 09 | Language runtimes | rust · node.js (via nvm) |
| 10 | Academic typesetting | texlive-basic · texlive-latexextra |

---

## Installation Checklist

> Run blocks in order. Items marked `[AUR]` require `paru`. Reboot after Docker group step (§06).

---

### 01 · Shell environment

The zsh shell is already present on CachyOS. This step adds the prompt, autocompletion, and syntax highlighting layer.

- [ ] `starship` — cross-shell prompt written in Rust; shows git state, Python venv, command timing ([starship.rs](https://starship.rs))
- [ ] `zsh-autosuggestions` — fish-style inline command suggestions from history
- [ ] `zsh-syntax-highlighting` — real-time syntax colouring before execution

```bash
sudo pacman -S starship zsh-autosuggestions zsh-syntax-highlighting
```

Add to `~/.zshrc`:

```zsh
# Plugins
source /usr/share/zsh/plugins/zsh-autosuggestions/zsh-autosuggestions.zsh
source /usr/share/zsh/plugins/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh

# Starship prompt
eval "$(starship init zsh)"
```

Post-install verification:

```bash
starship --version   # starship 1.x.x
```

**References:** [starship.rs](https://starship.rs) · [zsh-autosuggestions](https://github.com/zsh-users/zsh-autosuggestions) · [zsh-syntax-highlighting](https://github.com/zsh-users/zsh-syntax-highlighting)

---

### 02 · Version control

- [ ] `git` — version control engine
- [ ] `github-cli` — PR / issue / repo management from the terminal ([cli.github.com](https://cli.github.com))
- [ ] `lazygit` — terminal UI for Git; visually review diffs, stage, commit, branch ([github.com/jesseduffield/lazygit](https://github.com/jesseduffield/lazygit))
- [ ] `git-delta` — syntax-highlighted, side-by-side diff viewer; replaces the default `git diff` output ([github.com/dandavison/delta](https://github.com/dandavison/delta))

```bash
sudo pacman -S git github-cli lazygit git-delta
gh auth login
```

Configure delta as the default git pager (`~/.gitconfig`):

```ini
[core]
    pager = delta
[delta]
    navigate = true
    side-by-side = true
[interactive]
    diffFilter = delta --color-only
```

Post-install verification:

```bash
git --version
gh auth status
lazygit --version
delta --version
```

> **Why lazygit?** When Claude Code or any agentic tool makes autonomous changes to your codebase, you need a fast way to review what it just did before committing. `lazygit` gives you a full visual diff without leaving the terminal. The average `branch + stage + commit + push` cycle drops from ~48s to ~18s vs raw git commands.

---

### 03 · Modern CLI primitives

These are direct, superior replacements for standard Unix tools. They are faster, have better defaults, and output syntax-highlighted, human-readable results. Drop them in as aliases.

- [ ] `eza` — replaces `ls` · tree views, git status, icons ([github.com/eza-community/eza](https://github.com/eza-community/eza))
- [ ] `bat` — replaces `cat` · syntax highlighting, line numbers, git diff integration ([github.com/sharkdp/bat](https://github.com/sharkdp/bat))
- [ ] `ripgrep` — replaces `grep` · 5-10× faster, respects `.gitignore` ([github.com/BurntSushi/ripgrep](https://github.com/BurntSushi/ripgrep))
- [ ] `fd` — replaces `find` · intuitive syntax, fast, respects `.gitignore` ([github.com/sharkdp/fd](https://github.com/sharkdp/fd))
- [ ] `btop` — replaces `htop` · beautiful real-time CPU / GPU / RAM / disk / network monitor ([github.com/aristocratos/btop](https://github.com/aristocratos/btop))
- [ ] `zoxide` — replaces `cd` · learns your most-visited directories; type `z proj` instead of `cd ~/dev/projects/...` ([github.com/ajeetdsouza/zoxide](https://github.com/ajeetdsouza/zoxide))

```bash
sudo pacman -S eza bat ripgrep fd btop zoxide
```

Add aliases to `~/.zshrc`:

```zsh
# Modern CLI replacements
alias ls="eza --icons"
alias ll="eza -la --icons --git"
alias lt="eza --tree --level=2 --icons"
alias cat="bat"
alias grep="rg"
alias find="fd"

# Zoxide (smart cd) — must come after PATH setup
eval "$(zoxide init zsh)"
```

Post-install verification:

```bash
eza --version && bat --version && rg --version && fd --version && btop --version && zoxide --version
```

---

### 04 · Session management

- [ ] `tmux` — terminal multiplexer; persistent sessions for long-running inference loops and background research tasks
- [ ] `fzf` — fuzzy finder; powers `Ctrl+R` history search and `Ctrl+T` file picker
- [ ] `atuin` — replaces shell history with a searchable SQLite database; `Ctrl+R` becomes a structured, filterable log of every command ever run ([github.com/atuinsh/atuin](https://github.com/atuinsh/atuin))

```bash
sudo pacman -S tmux fzf atuin
```

Minimal `~/.tmux.conf`:

```bash
set -g default-terminal "tmux-256color"
set -ag terminal-overrides ",xterm-256color:RGB"
set -g mouse on
set -g history-limit 100000
set -g base-index 1
setw -g pane-base-index 1
bind r source-file ~/.tmux.conf \; display "Config reloaded"
bind | split-window -h -c "#{pane_current_path}"
bind - split-window -v -c "#{pane_current_path}"
```

Add to `~/.zshrc`:

```zsh
# fzf key bindings
source /usr/share/fzf/key-bindings.zsh
source /usr/share/fzf/completion.zsh

# Atuin shell history
eval "$(atuin init zsh)"
```

Post-install verification:

```bash
tmux -V && fzf --version && atuin --version
```

---

### 05 · Python runtime & package management

`uv` is the current (2026) standard for Python environment and package management. It replaces `pip`, `venv`, `pyenv`, `pip-tools`, `pipx`, and `virtualenv` with a single Rust binary that operates 10–100× faster.

Current version: **uv 0.10.x** (March 2026 · [pypi.org/project/uv](https://pypi.org/project/uv/))

- [ ] `uv` installed and on `PATH`
- [ ] Python 3.12 and 3.13 managed via uv (no system Python dependency)

```bash
# Install uv (Rust binary, no Python required)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Or via pacman (may lag behind latest release)
sudo pacman -S uv
```

Core workflow:

```bash
# Install Python versions
uv python install 3.12 3.13

# New project (creates pyproject.toml + uv.lock)
uv init my-project
cd my-project
uv add numpy pandas torch

# Sync from lockfile (reproducible)
uv sync

# Run a script without activating a venv
uv run python main.py

# Install a CLI tool globally (replaces pipx)
uv tool install ruff
uvx black .
```

> **GPU packages note:** `uv` handles pure Python packages. For CUDA-dependent packages (PyTorch with CUDA, cuDNN), install via `uv add torch --index-url https://download.pytorch.org/whl/cu124`. `uv` does not manage CUDA itself — the NVIDIA driver handles that.

Post-install verification:

```bash
uv --version        # uv 0.10.x
uv python list      # shows installed Python versions
```

**References:** [docs.astral.sh/uv](https://docs.astral.sh/uv) · [pypi.org/project/uv](https://pypi.org/project/uv/)

---

### 06 · Containerisation

- [ ] `docker`
- [ ] `docker-compose`
- [ ] Docker daemon enabled as a systemd service
- [ ] Current user added to the `docker` group

```bash
sudo pacman -S docker docker-compose
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```

> **Restart your session** after `usermod` — group membership is not active in the current shell.

Post-install verification:

```bash
docker run --rm hello-world
docker compose version
```

---

### 07 · Agentic AI toolchain

This is the layer most significantly changed from earlier versions of this document. The 2026 local AI stack is now mature enough to use as a daily driver for agentic coding workflows.

#### 07A · Ollama (local model runtime)

- [ ] `ollama` installed and running as a systemd service
- [ ] At least one coding model pulled (`qwen2.5-coder:14b` is the v3-node sweet spot)

```bash
# Install ollama
curl -fsSL https://ollama.com/install.sh | sh

# Or via AUR
paru -S ollama-cuda   # NVIDIA GPU variant
```

```bash
# Enable as a service
sudo systemctl enable --now ollama

# Pull recommended models for the RTX 5070 (8 GB VRAM)
ollama pull qwen2.5-coder:14b        # ~8.5 GB — coding, fits with Q4
ollama pull qwen2.5-coder:7b         # ~4.5 GB — fast autocomplete
ollama pull llama3.1:8b              # ~5.0 GB — general reasoning
```

> **VRAM budget for the RTX 5070 (8 GB GDDR7):** The 14B model at Q4 quantization sits at ~7.5 GB. Running anything larger requires CPU offloading, which slows inference significantly. The 7B model is the better choice for low-latency agentic loops (the agent makes many sequential calls — response speed matters more than raw capability per call).

Post-install verification:

```bash
ollama list          # shows pulled models
ollama run qwen2.5-coder:7b "write a python hello world"
```

**References:** [ollama.com](https://ollama.com) · [Ollama + Claude Code integration](https://docs.ollama.com/integrations/claude-code)

---

#### 07B · Claude Code (primary agentic CLI)

Anthropic's agentic coding tool. Reads files, runs commands, edits code autonomously. As of Ollama v0.14+, it can route to local Ollama models via the Anthropic-compatible API — no cloud required for local inference sessions.

- [ ] `claude` CLI installed
- [ ] `.zshrc` aliases configured for each API pathway

```bash
# Install via npm
npm install -g @anthropic-ai/claude-code

# Or via the official installer
curl -fsSL https://claude.ai/install.sh | sh
```

`.zshrc` mesh aliases (from `alphaNode` agentic mesh config):

```zsh
# Claude Code — Anthropic native
alias claude-native="claude"

# Claude Code — OpenRouter (DeepSeek R1)
alias cr1="claude-or --model deepseek/deepseek-r1"

# Claude Code — Ollama local (RTX 5070)
alias cl="ANTHROPIC_BASE_URL=http://localhost:11434 ANTHROPIC_AUTH_TOKEN=ollama ANTHROPIC_API_KEY='' claude --model qwen2.5-coder:14b"

# Claude Code — Ollama cloud tunnel
alias cc="claude --model kimi-k2.5:cloud"
```

**References:** [claude.ai/code](https://claude.ai/code) · [Ollama Claude Code docs](https://docs.ollama.com/integrations/claude-code)

---

#### 07C · OpenCode (multi-provider TUI)

Terminal UI coding agent supporting 75+ providers including Ollama, OpenRouter, Anthropic, and GitHub Copilot. Used as the multi-agent research swarm layer in the alphaNode mesh.

- [ ] `opencode` installed

```bash
# Install via npm
npm install -g opencode-ai

# Launch with provider
opencode --model openrouter/deepseek/deepseek-r1   # OpenRouter swarm
opencode --model ollama/qwen2.5-coder:7b           # Sovereign local mode
```

**References:** [opencode.ai](https://opencode.ai) · [OpenCode GitHub](https://github.com/sst/opencode)

---

#### 07D · Open-WebUI (local inference web frontend)

Browser-based interface for local Ollama inference. Runs at `localhost:3000`.

- [ ] `open-webui` container running via Docker

```bash
docker run -d \
  --name open-webui \
  --gpus all \
  -p 3000:8080 \
  -v open-webui:/app/backend/data \
  --restart unless-stopped \
  ghcr.io/open-webui/open-webui:cuda
```

Access at `http://localhost:3000`. Connects to `http://host.docker.internal:11434` for Ollama automatically.

**References:** [github.com/open-webui/open-webui](https://github.com/open-webui/open-webui)

---

### 08 · IDE

- [ ] `visual-studio-code-bin` — official Microsoft binary `[AUR]`

```bash
paru -S visual-studio-code-bin
```

Recommended `settings.json`:

```jsonc
{
  "telemetry.telemetryLevel": "off",
  "editor.gpuAcceleration": "on",
  "terminal.integrated.gpuAcceleration": "on",
  "window.titleBarStyle": "custom",
  "editor.fontFamily": "'JetBrainsMono Nerd Font', monospace",
  "editor.fontSize": 14,
  "editor.lineHeight": 1.6,
  "files.autoSave": "onFocusChange"
}
```

Install a Nerd Font for icon support in `eza`, `starship`, and `lazygit`:

```bash
paru -S ttf-jetbrains-mono-nerd
```

**References:** [code.visualstudio.com](https://code.visualstudio.com) · [Nerd Fonts](https://www.nerdfonts.com)

---

### 09 · Language runtimes

#### Rust

Required by many tools in this stack (`uv`, `starship`, `zoxide`, `ripgrep`, `bat`, `fd`, `delta` are all written in Rust). Install the toolchain for building and contributing to Rust projects.

- [ ] `rustup` installed with the `stable` toolchain

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup default stable
```

Post-install verification:

```bash
cargo --version && rustc --version
```

#### Node.js (via nvm)

Required for `claude-code`, `opencode`, and any JavaScript/TypeScript tooling.

- [ ] `nvm` installed
- [ ] Node.js LTS pinned

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
# Restart shell, then:
nvm install --lts
nvm use --lts
```

Post-install verification:

```bash
node --version && npm --version
```

**References:** [rustup.rs](https://rustup.rs) · [github.com/nvm-sh/nvm](https://github.com/nvm-sh/nvm)

---

### 10 · Academic typesetting

Minimal LaTeX distribution for research deliverables and FAiR documentation.

- [ ] `texlive-basic` — core TeX engine
- [ ] `texlive-latexextra` — floats, listings, bibliography, algorithm environments
- [ ] `texlive-science` — physics, math, and technical notation packages

```bash
sudo pacman -S texlive-basic texlive-latexextra texlive-science
```

Post-install verification:

```bash
pdflatex --version
```

---

## Full install — single block

```bash
# ── 01 Shell environment
sudo pacman -S starship zsh-autosuggestions zsh-syntax-highlighting

# ── 02 Version control
sudo pacman -S git github-cli lazygit git-delta
gh auth login

# ── 03 Modern CLI primitives
sudo pacman -S eza bat ripgrep fd btop zoxide

# ── 04 Session management
sudo pacman -S tmux fzf atuin

# ── 05 Python runtime
curl -LsSf https://astral.sh/uv/install.sh | sh
uv python install 3.12 3.13

# ── 06 Containerisation
sudo pacman -S docker docker-compose
sudo systemctl enable --now docker
sudo usermod -aG docker $USER

# ── 07 Agentic AI
curl -fsSL https://ollama.com/install.sh | sh
sudo systemctl enable --now ollama
ollama pull qwen2.5-coder:14b
ollama pull qwen2.5-coder:7b
npm install -g @anthropic-ai/claude-code opencode-ai

# ── 08 IDE (AUR)
paru -S visual-studio-code-bin ttf-jetbrains-mono-nerd

# ── 09 Language runtimes
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash

# ── 10 LaTeX
sudo pacman -S texlive-basic texlive-latexextra texlive-science
```

> **After the full install:** restart your session for Docker group membership, then run the verification table below.

---

## Post-install verification table

| Layer | Tool | Command | Expected |
|:--|:--|:--|:--|
| 01 | starship | `starship --version` | `starship 1.x.x` |
| 02 | git | `git --version` | `git version 2.x.x` |
| 02 | gh | `gh auth status` | `Logged in to github.com` |
| 02 | lazygit | `lazygit --version` | version string |
| 02 | delta | `delta --version` | `delta 0.x.x` |
| 03 | eza | `eza --version` | `eza v0.x.x` |
| 03 | bat | `bat --version` | `bat 0.x.x` |
| 03 | ripgrep | `rg --version` | `ripgrep 1x.x.x` |
| 03 | fd | `fd --version` | `fd 10.x.x` |
| 03 | btop | `btop --version` | `btop version 1.x.x` |
| 03 | zoxide | `zoxide --version` | `zoxide 0.x.x` |
| 04 | tmux | `tmux -V` | `tmux 3.x` |
| 04 | fzf | `fzf --version` | `0.x.x` |
| 04 | atuin | `atuin --version` | `atuin 18.x.x` |
| 05 | uv | `uv --version` | `uv 0.10.x` |
| 05 | python | `uv run python --version` | `Python 3.12.x` |
| 06 | docker | `docker run --rm hello-world` | `Hello from Docker!` |
| 06 | compose | `docker compose version` | `Docker Compose v2.x.x` |
| 07 | ollama | `ollama list` | models listed |
| 07 | claude | `claude --version` | version string |
| 08 | code | `code --version` | VSCode version |
| 09 | cargo | `cargo --version` | `cargo 1.x.x` |
| 09 | node | `node --version` | `v22.x.x` |
| 10 | pdflatex | `pdflatex --version` | `pdfTeX 3.x` |

---

## `.zshrc` reference — complete aliases block

```zsh
# ── Modern CLI replacements
alias ls="eza --icons"
alias ll="eza -la --icons --git"
alias lt="eza --tree --level=2 --icons"
alias cat="bat"
alias grep="rg"
alias find="fd"

# ── Navigation
eval "$(zoxide init zsh)"
alias cd="z"

# ── Claude Code mesh aliases
alias cr1="claude-or --model deepseek/deepseek-r1"
alias cc="claude --model kimi-k2.5:cloud"
alias cl="ANTHROPIC_BASE_URL=http://localhost:11434 ANTHROPIC_AUTH_TOKEN=ollama ANTHROPIC_API_KEY='' claude --model qwen2.5-coder:14b"

# ── Git shortcuts
alias lg="lazygit"
alias gs="git status"
alias gd="git diff"

# ── System
alias top="btop"
alias df="df -h"
alias free="free -h"

# ── Docker
alias dc="docker compose"
alias dps="docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'"

# ── fzf + atuin
source /usr/share/fzf/key-bindings.zsh
source /usr/share/fzf/completion.zsh
eval "$(atuin init zsh)"

# ── Starship prompt (must be last)
eval "$(starship init zsh)"
```

---

## What changed from the original document

| Category | Original | 2026 upgrade | Reason |
|:--|:--|:--|:--|
| ISA target | `x86-64-v4` | `x86-64-v3` | Raptor Lake-HX has no AVX-512 |
| Shell prompt | none | `starship` | Context-aware, fast, shell-agnostic |
| `ls` | bare `ls` | `eza` | Icons, git status, tree views |
| `cat` | bare `cat` | `bat` | Syntax highlighting, line numbers |
| `grep` | bare `grep` | `ripgrep` | 5–10× faster, `.gitignore`-aware |
| `find` | bare `find` | `fd` | Intuitive syntax, faster |
| `htop` | not included | `btop` | GPU, disk, network visibility |
| `cd` | bare `cd` | `zoxide` | Directory jump intelligence |
| Git UI | none | `lazygit` | Critical for reviewing agentic code changes |
| Git diffs | bare `git diff` | `delta` | Side-by-side syntax-highlighted diffs |
| Shell history | bare history | `atuin` | Structured, searchable, filterable |
| AI toolchain | not included | Ollama + Claude Code + OpenCode | Core 2026 agentic workflow layer |
| Python | `uv` (correct) | `uv 0.10.x` | Version confirmed current |
| Node.js | not included | `nvm` + LTS | Required for `claude-code`, `opencode` |
| Rust | not included | `rustup` | Required by most Rust CLI tools |
| LaTeX | `texlive-basic` + `latexextra` | + `texlive-science` | Physics / math notation support |

---

*Maintainer: David Grace (@davidgracemann) · alphaNode · March 2026*
