<img width="1000" height="500" alt="applied-&-foundational-ai banner" src="https://media3.giphy.com/media/v1.Y2lkPTc5MGI3NjExdmk5OWYzZmJsZWNsdmE5NGNleXVhY281b2JjYmNyd2FjcjIwa2czOCZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/FqpCVcIHXJuuCfNa4b/giphy.gif"/>

# applied-&-foundational-ai · resourceHub

> Personal research & engineering infrastructure — **@davidgracemann**
> 
> A distributed, multi-node AI compute mesh with tiered API fallback, sovereign local inference, and a unified agentic toolchain spanning CLI, TUI, IDE, and web interfaces.

![OS](https://img.shields.io/badge/OS-CachyOS-blue?style=flat-square)
![Kernel](https://img.shields.io/badge/Kernel-Linux%206.19-informational?style=flat-square)
![GPU](https://img.shields.io/badge/GPU-RTX%205070%20%2B%20GTX%201650-green?style=flat-square)
![Mesh](https://img.shields.io/badge/Mesh-4--Tool%20%C3%97%204--Provider-orange?style=flat-square)
![Updated](https://img.shields.io/badge/Updated-March%202026-lightgrey?style=flat-square)

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Distributed Computing Network](#2-distributed-computing-network-via-ssh)
   - [alphaNode — Primary Research Machine](#21-alphanode--primary-research-machine)
   - [betaNode — Distributed Support Node](#22-betanode--distributed-support-node)
3. [Node Capability Matrix](#3-node-capability-matrix)
4. [Agentic Mesh — Fallback Table](#4-agentic-mesh--fallback-table)
5. [Mesh Status Legend](#5-mesh-status-legend)
6. [alphaNode Execution Guide](#6-alphanode-execution-guide-triggering-the-mesh)
   - [A · Claude Code (CLI)](#a-claude-code-cli)
   - [B · OpenCode (TUI)](#b-opencode-tui)
   - [C · Agentic VS Code (Roo Code)](#c-agentic-vs-code-roo-code)
   - [D · proChats (Web UI)](#d-prochats-web-interfaces)
7. [Environment & Prerequisites](#7-environment--prerequisites)
8. [Metadata](#8-metadata)

---

## 1. Architecture Overview

This repository documents a **two-node distributed AI research mesh** connected via SSH. The mesh supports four agentic tooling layers — CLI, TUI, IDE, and web — each configurable across four independent API pathways: Anthropic native, OpenRouter, Ollama-Cloud, and Ollama-Local.

The design philosophy is **graceful degradation**: if the primary Anthropic pathway is unavailable or cost-prohibitive, inference automatically routes through OpenRouter (DeepSeek R1/V3), then to cloud-tunnelled Ollama, and finally to fully sovereign local GPU inference on the RTX 5070.

```
┌─────────────────────────────────────────────────────────┐
│                    AGENTIC TOOLCHAIN                    │
│   [A] Claude Code  [B] OpenCode  [C] RooCode  [D] Web  │
└───────────────────────┬─────────────────────────────────┘
                        │ API Pathway Selection
          ┌─────────────┼────────────────┐
          ▼             ▼                ▼
   [1] Anthropic  [2] OpenRouter  [3] Ollama-Cloud
        Pro          R1 / V3        Kimi / Minimax
                                        │
                                        ▼
                               [4] Ollama-Local
                               RTX 5070 · alphaNode
                               GTX 1650 · betaNode
```

```
SSH Fabric
──────────────────────────────────────────
  192.168.1.13  ←→  alphaNode  (primary)
        │
        └──── betaNode  (headless / mirror)
```

---

## 2. Distributed Computing Network (via SSH)

### 2.1 alphaNode · Primary Research Machine

**Role:** Master node for heavy model orchestration, AI research, and high-performance computing.

| Component | Specification | Operational Notes |
|:---|:---|:---|
| **Device** | Lenovo Legion Pro 5 (83NN003CIN) | Optimized for high-concurrency research |
| **OS** | CachyOS (Linux Kernel — Optimized) | Low-latency kernel for real-time task scheduling |
| **CPU** | Intel Core i9-14900HX (24C / 32T) | Base: 2.2 GHz · Boost: 5.8 GHz |
| **RAM** | 32 GB DDR5-5600 | High-bandwidth for local RAG and agent logic |
| **GPU** | NVIDIA RTX 5070 Mobile (8 GB GDDR7 VRAM) | Max fully-offloaded model size: ≈ 7.5 GB |
| **Storage** | 1 TB NVMe Gen4 SSD | High-speed I/O for large dataset processing |
| **Network** | Realtek 8922AE Wi-Fi 7 | Primary link for high-speed node communication |
| **Display** | 2560 × 1600 @ 240 Hz | — |
| **Local IP** | 192.168.1.13 | Primary SSH target for mesh orchestration |

<details>
<summary><code>fastfetch</code> — live system snapshot</summary>

```zsh
❯ fastfetch
           .-------------------------:                    davidgracemann@alphaNode
          .+=========================.                    ------------------------
         :++===++==================-       :++-           OS: CachyOS x86_64
        :*++====+++++=============-        .==:           Host: 83NN (Legion Pro 5 16IRX10)
       -*+++=====+***++==========:                        Kernel: Linux 6.19.7-1-cachyos
      =*++++========------------:                         Uptime: 38 mins
      =*+++++=====-                     ...                Packages: 1265 (pacman), 8 (flatpak)
   .+*+++++=-===:                    .=+++=:              Shell: zsh 5.9
  :++++=====-==:                     -*****+              Display: 2560x1600 @ 240 Hz
  :++========-=.                      .=+**+.              DE: COSMIC 1.0.0
.+==========-.                          .                 WM: cosmic-comp (Wayland)
 :+++++++====-                                .--==-.     Cursor: Adwaita
  :++==========.                             :+++++++:    Terminal: cosmic-term
   .-===========.                            =*****+*+    Terminal Font: Noto Sans Mono
    .-===========:                           .+*****+:    CPU: Intel i9-14900HX (32) @ 5.80 GHz
      -=======++++:::::::::::::::::::::::::-:  .---:      GPU 1: NVIDIA RTX 5070 Mobile
       :======++++====+++******************=.             GPU 2: Intel Raptor Lake-S UHD
        :=====+++==========++++++++++++++*-               Memory: 3.63 GiB / 31.06 GiB
         .====++==============++++++++++*-                Swap: 0 B / 31.06 GiB
          .===+==================+++++++:                 Disk (/): 34.50 GiB / 583.94 GiB
           .-=======================+++:                  Local IP: 192.168.1.13/24
             ..........................                   Battery: 100% [AC Connected]
```

</details>

---

### 2.2 betaNode · Distributed Support Node

**Role:** Repurposed legacy workstation for headless background tasks and task mirroring.

| Component | Specification | Operational Notes |
|:---|:---|:---|
| **Device** | Lenovo IdeaPad Gaming 3 (15IMH05) | Secondary compute tasks |
| **OS** | CachyOS (Linux Kernel — Optimized) | Unified SSH orchestration environment |
| **CPU** | Intel Core i7-10750H (6C / 12T) | Base: 2.6 GHz · Boost: 5.0 GHz |
| **RAM** | 8 GB DDR4-2933 | Optimized for minimal idle overhead |
| **GPU** | NVIDIA GTX 1650 (4 GB GDDR6) | Small-quantization models (≤ 3.5 GB) |
| **Storage** | SATA SSD | Logs and build artifacts |
| **Network** | Intel Wi-Fi 6 AX201 | 50 Mbps Spectra WLAN link |

---

## 3. Node Capability Matrix

A quick reference for which model classes each node can fully offload to GPU VRAM.

| | alphaNode · RTX 5070 (8 GB GDDR7) | betaNode · GTX 1650 (4 GB GDDR6) |
|:---|:---:|:---:|
| **Max fully-offloaded model** | ≈ 7.5 GB | ≈ 3.5 GB |
| **Typical quant tier** | Q4–Q6 at 7B–14B params | Q4 at 3B–7B params |
| **Example models** | `qwen2.5-coder:14b`, `llama3.1:8b` | `phi3:mini`, `llama3.2:3b` |
| **Recommended role** | Primary inference + orchestration | Background tasks, log processing, build artifacts |
| **Headless operation** | Optional | Default |

---

## 4. Agentic Mesh · Fallback Table

The mesh is a **4 × 4 matrix** of tooling layers vs. API pathways. Each cell represents an independently configurable inference route with its own activation state and primary use case. Fallback priority runs left → right (columns 1 → 4).

| Tool ↓ / API Pathway → | 1 · Anthropic Pro | 2 · OpenRouter | 3 · Ollama-Cloud | 4 · Ollama-Local |
|:---|:---|:---|:---|:---|
| **A · Claude Code (CLI)** | `[Active]` Autonomous engineering | `[Active]` DeepSeek R1/V3 pipe | `[Active]` Kimi/Minimax tunnel | `[Configuring]` Llama 3.1 / Qwen local |
| **B · OpenCode (TUI)** | `[Standby]` Reasoning via shim | `[Active]` Multi-agent research swarm | `[Active]` Rapid prototyping | `[Active]` Zero-latency private |
| **C · Agentic VS Code** | `[Pending Sleep]` IDE refactors | `[Pending Sleep]` "God Mode" IDE | `[Pending Sleep]` Infinite autocompletes | `[Configuring]` Local unit testing |
| **D · proChats (Web UI)** | `[Processing]` Architectural math | `[Active]` Model discovery & R1 | `[Active]` Cloud brainstorming | `[Configuring]` Desktop inference |

---

## 5. Mesh Status Legend

| Status | Meaning |
|:---|:---|
| `[Active]` | Fully operational; in regular use |
| `[Standby]` | Configured and ready; not in active rotation |
| `[Processing]` | Currently in use for an ongoing task |
| `[Pending Sleep]` | Configured but temporarily deprioritized |
| `[Configuring]` | In setup; not yet production-ready |

---

## 6. alphaNode Execution Guide: Triggering the Mesh

### A · Claude Code (CLI)

The primary autonomous engineering tool, optimized via `.zshrc` aliases.

**1 · Anthropic Native**

```zsh
claude
```

**2 · OpenRouter (DeepSeek R1) — via Command Sniper alias**

```zsh
# Alias shortcut
cr1

# Manual invocation
claude-or --model deepseek/deepseek-r1
```

**3 · Ollama-Cloud (Kimi tunnel — zero-cost cloud reasoning)**

```zsh
cc --model kimi-k2.5:cloud
```

**4 · Ollama-Local (RTX 5070 direct GPU inference)**

```zsh
cl --model qwen2.5-coder:14b
```

---

### B · OpenCode (TUI)

The multi-agent research hub. Controlled via the internal TUI provider menu.

**1 · Anthropic Native**

Set `ANTHROPIC_API_KEY` in settings and select `claude-3-5-sonnet`.

**2 · OpenRouter — Swarm Mode**

```zsh
opencode --model openrouter/deepseek/deepseek-r1
```

**3 · Ollama-Cloud**

Point the TUI to `localhost:11434` tunnel using the cloud-mapped model ID.

**4 · Ollama-Local — Sovereign Mode**

```zsh
opencode --model ollama/llama3.1:8b
```

---

### C · Agentic VS Code (Roo Code)

IDE-level autonomy. Configure via the **Roo Code extension settings** `[Gear Icon]`.

**1 · Roo-Claude (Anthropic)**

```
Provider  →  Anthropic
Model     →  claude-3-5-sonnet-20241022
```

**2 · Roo-OR (OpenRouter — complex refactors)**

```
Provider  →  OpenRouter
Model     →  deepseek/deepseek-r1
```

**3 · Roo-Local (Ollama — low-latency autocomplete)**

```
Provider  →  Ollama
Base URL  →  http://localhost:11434
Model     →  qwen2.5-coder:7b
```

---

### D · proChats (Web Interfaces)

High-level architectural drafting and research.

| Interface | Access | Primary Use |
|:---|:---|:---|
| **Anthropic Web UI** | [claude.ai](https://claude.ai) | Architectural math, long-form reasoning |
| **OpenRouter Chat** | [openrouter.ai/chat](https://openrouter.ai/chat) | Model discovery, DeepSeek R1 |
| **Local Desktop (Open-WebUI)** | `localhost:3000` | RTX 5070 local inference |

---

## 7. Environment & Prerequisites

The execution aliases in Section 6 (`cr1`, `cc`, `cl`, etc.) are defined in `.zshrc` on alphaNode. The following components must be present for the full mesh to operate:

| Dependency | Purpose | Pathway |
|:---|:---|:---|
| `claude` CLI | Claude Code autonomous engineering | A1 — Anthropic native |
| `ANTHROPIC_API_KEY` | Authenticates Claude Code + OpenCode Anthropic routes | A1, B1 |
| `opencode` | Multi-agent TUI hub | B — all pathways |
| Roo Code extension | Agentic VS Code integration | C — all pathways |
| `ollama` daemon | Local model serving on RTX 5070 | A4, B4, C3, D3 |
| Open-WebUI | Local web inference frontend | D4 — `localhost:3000` |
| OpenRouter API key | R1/V3 and swarm-mode routing | A2, B2, C2, D2 |
| SSH config | alphaNode ↔ betaNode fabric | Both nodes |
| CachyOS + optimized kernel | Low-latency scheduling across both nodes | System-level |

> **Note:** Ollama-Cloud tunnels (`cc --model kimi-k2.5:cloud`) require a running tunnel process on `localhost:11434` mapped to the cloud provider endpoint. Configuration is node-local.

---

## 8. Metadata

| Field | Value |
|:---|:---|
| **Maintainer** | David Grace (`@davidgracemann`) |
| **Hardware** | Gracemann365 |
| **Primary Node OS** | CachyOS · Linux 6.19.7-1-cachyos |
| **Last Updated** | March 2026 |
| **Mesh Topology** | 2-node SSH fabric · 4-tool × 4-provider matrix |
| **Inference Stack** | Anthropic → OpenRouter → Ollama-Cloud → Ollama-Local |

---

*Personal research infrastructure · Not affiliated with Anthropic, OpenRouter, or any referenced model provider.*
